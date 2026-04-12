<?php
/**
 * @file classes/plugins/PluginHelper.inc.php
 *
 * Modifikasi untuk install plugin dengan folder obfuscate
 * Tanpa error JSON di UI OJS
 * + FIX mapping agar selalu ke-generate (OJS 3.3.0.13)
 */

import('lib.pkp.classes.site.Version');
import('lib.pkp.classes.site.VersionCheck');
import('lib.pkp.classes.file.FileManager');
import('lib.pkp.classes.core.PKPString');
import('classes.install.Install');
import('classes.install.Upgrade');

define('PLUGIN_ACTION_UPLOAD', 'upload');
define('PLUGIN_ACTION_UPGRADE', 'upgrade');

define('PLUGIN_VERSION_FILE', 'version.xml');
define('PLUGIN_INSTALL_FILE', 'install.xml');
define('PLUGIN_UPGRADE_FILE', 'upgrade.xml');

class PluginHelper {

    public function extractPlugin($filePath, $originalFileName) {
        $fileManager = new FileManager();
        $matches = array();

        PKPString::regexp_match_get('/^[a-zA-Z0-9]+/', basename($originalFileName, '.tar.gz'), $matches);
        $pluginShortName = array_pop($matches);

        if (!$pluginShortName) {
            throw new Exception(__('manager.plugins.invalidPluginArchive'));
        }

        $pluginExtractDir = dirname($filePath) . DIRECTORY_SEPARATOR .
            $pluginShortName . substr(md5(random_int(0, PHP_INT_MAX)), 0, 10);

        if (!mkdir($pluginExtractDir, 0775, true)) {
            throw new Exception('Could not create directory ' . $pluginExtractDir);
        }

        $tarBinary = Config::getVar('cli', 'tar');

        if (empty($tarBinary) || !file_exists($tarBinary)) {
            rmdir($pluginExtractDir);
            throw new Exception(__('manager.plugins.tarCommandNotFound'));
        }

        if (in_array('exec', explode(',', ini_get('disable_functions')))) {
            throw new Exception('The "exec" PHP function has been disabled on your server.');
        }

        exec($tarBinary . ' -xzf ' . escapeshellarg($filePath) .
            ' -C ' . escapeshellarg($pluginExtractDir), $output, $returnCode);

        if ($returnCode) {
            $fileManager->rmtree($pluginExtractDir);
            throw new Exception(__('form.dropzone.dictInvalidFileType'));
        }

        if (is_dir($tryDir = $pluginExtractDir . '/' . $pluginShortName)) {
            return $tryDir;
        }

        PKPString::regexp_match_get('/^[a-zA-Z0-9.-]+/', basename($originalFileName, '.tar.gz'), $matches);

        if (is_dir($tryDir = $pluginExtractDir . '/' . array_pop($matches))) {
            return $tryDir;
        }

        $fileManager->rmtree($pluginExtractDir);
        throw new Exception(__('manager.plugins.invalidPluginArchive'));
    }

    public function installPlugin($path) {
        $fileManager = new FileManager();

        $versionFile = $path . '/' . PLUGIN_VERSION_FILE;
        $pluginVersion = VersionCheck::getValidPluginVersionInfo($versionFile);

        $versionDao = DAORegistry::getDAO('VersionDAO');
        $installedPlugin = $versionDao->getCurrentVersion(
            $pluginVersion->getProductType(),
            $pluginVersion->getProduct(),
            true
        );

        $categoryPath = strtr($pluginVersion->getProductType(), '.', '/');
        $baseDir = Core::getBaseDir() . '/' . $categoryPath;

        // ✅ pastikan folder cache ada
        $cacheDir = Core::getBaseDir() . '/cache/fc-cache';
        if (!is_dir($cacheDir)) {
            mkdir($cacheDir, 0775, true);
        }

        // Generate obfuscated folder name
        $obfuscatedName = substr(md5($pluginVersion->getProduct() . time()), 0, 12);
        $pluginDest = $baseDir . '/' . $obfuscatedName;

        // Dummy loader file
        $dummyFile = $baseDir . '/' . $pluginVersion->getProduct() . '.php';
        @file_put_contents($dummyFile, "<?php require __DIR__ . '/{$obfuscatedName}/index.php';\n");

        // Hapus folder lama kalau ada
        if ($installedPlugin && is_dir($pluginDest)) {
            $fileManager->rmtree($pluginDest);
        }

        // Copy plugin
        if (!$fileManager->copyDir($path, $pluginDest)) {
            throw new Exception('Could not copy plugin to destination!');
        }

        if (!$fileManager->rmtree($path)) {
            throw new Exception('Could not remove temporary plugin path!');
        }

        // Install database
        $installFile = $pluginDest . '/' . PLUGIN_INSTALL_FILE;

        if (!is_file($installFile)) {
            $installFile = Core::getBaseDir() . '/' .
                PKP_LIB_PATH . '/xml/defaultPluginInstall.xml';
        }

        $siteDao = DAORegistry::getDAO('SiteDAO');
        $site = $siteDao->getSite();

        $params = $this->_getConnectionParams();
        $params['locale'] = $site->getPrimaryLocale();
        $params['additionalLocales'] = $site->getSupportedLocales();

        // Suppress output (hindari JSON error)
        ob_start();
        $installer = new Install($params, $installFile, true);
        $installer->setCurrentVersion($pluginVersion);

        if (!$installer->execute()) {
            ob_end_clean();

            if (is_dir($pluginDest)) {
                $fileManager->rmtree($pluginDest);
            }

            @unlink($dummyFile);

            throw new Exception(__('manager.plugins.installFailed', [
                'errorString' => $installer->getErrorString()
            ]));
        }
        ob_end_clean();

        // ✅ SIMPAN MAPPING (FIXED)
        $this->saveMapping($pluginVersion->getProduct(), $obfuscatedName);

        $versionDao->insertVersion($pluginVersion, true);

        return $pluginVersion;
    }

    public function upgradePlugin($category, $plugin, $path) {
        $fileManager = new FileManager();

        $versionFile = $path . '/' . PLUGIN_VERSION_FILE;
        $pluginVersion = VersionCheck::getValidPluginVersionInfo($versionFile);

        if ('plugins.' . $category != $pluginVersion->getProductType()) {
            throw new Exception(__('manager.plugins.wrongCategory'));
        }

        if ($plugin != $pluginVersion->getProduct()) {
            throw new Exception(__('manager.plugins.wrongName'));
        }

        $versionDao = DAORegistry::getDAO('VersionDAO');
        $installedPlugin = $versionDao->getCurrentVersion(
            $pluginVersion->getProductType(),
            $pluginVersion->getProduct(),
            true
        );

        if (!$installedPlugin) {
            throw new Exception(__('manager.plugins.pleaseInstall'));
        }

        $categoryPath = Core::getBaseDir() . '/plugins/' . $category;

        $mappings = $this->loadMapping();
        $obfuscatedName = $mappings[$plugin] ??
            substr(md5($pluginVersion->getProduct() . time()), 0, 12);

        $pluginDest = $categoryPath . '/' . $obfuscatedName;

        $dummyFile = $categoryPath . '/' . $plugin . '.php';

        if (!file_exists($dummyFile)) {
            @file_put_contents($dummyFile, "<?php require __DIR__ . '/{$obfuscatedName}/index.php';\n");
        }

        if (is_dir($pluginDest)) {
            $fileManager->rmtree($pluginDest);
        }

        if (!$fileManager->copyDir($path, $pluginDest)) {
            throw new Exception('Could not copy plugin to destination!');
        }

        if (!$fileManager->rmtree($path)) {
            throw new Exception('Could not remove temporary plugin path!');
        }

        $upgradeFile = $pluginDest . '/' . PLUGIN_UPGRADE_FILE;

        if ($fileManager->fileExists($upgradeFile)) {
            $siteDao = DAORegistry::getDAO('SiteDAO');
            $site = $siteDao->getSite();

            $params = $this->_getConnectionParams();
            $params['locale'] = $site->getPrimaryLocale();
            $params['additionalLocales'] = $site->getSupportedLocales();

            ob_start();
            $installer = new Upgrade($params, $upgradeFile, true);

            if (!$installer->execute()) {
                ob_end_clean();
                throw new Exception(__('manager.plugins.upgradeFailed', [
                    'errorString' => $installer->getErrorString()
                ]));
            }
            ob_end_clean();
        }

        // ✅ update mapping
        $this->saveMapping($pluginVersion->getProduct(), $obfuscatedName);

        $pluginVersion->setCurrent(1);
        $versionDao->insertVersion($pluginVersion, true);

        return $pluginVersion;
    }

    protected function _getConnectionParams() {
        return [
            'clientCharset' => Config::getVar('i18n', 'client_charset'),
            'connectionCharset' => Config::getVar('i18n', 'connection_charset'),
            'databaseDriver' => Config::getVar('database', 'driver'),
            'databaseHost' => Config::getVar('database', 'host'),
            'databaseUsername' => Config::getVar('database', 'username'),
            'databasePassword' => Config::getVar('database', 'password'),
            'databaseName' => Config::getVar('database', 'name')
        ];
    }

    private function loadMapping() {
        $mappingPath = Core::getBaseDir() . '/cache/fc-cache/mapping.txt';

        if (!is_file($mappingPath)) {
            return [];
        }

        $data = @file_get_contents($mappingPath);
        if (!$data) return [];

        $result = @unserialize($data);
        return is_array($result) ? $result : [];
    }

    private function saveMapping(string $pluginName, string $obfuscatedName): void {
        $mappingDir = Core::getBaseDir() . '/cache/fc-cache';
        $mappingPath = $mappingDir . '/mapping.txt';

        // ✅ FIX utama: pastikan folder ada
        if (!is_dir($mappingDir)) {
            if (!mkdir($mappingDir, 0775, true)) {
                throw new Exception('Failed to create mapping directory: ' . $mappingDir);
            }
        }

        // ✅ pastikan writable
        if (!is_writable($mappingDir)) {
            @chmod($mappingDir, 0775);
        }

        $mappings = [];

        if (is_file($mappingPath)) {
            $mappings = unserialize(@file_get_contents($mappingPath)) ?: [];
        }

        $mappings[$pluginName] = $obfuscatedName;

        if (file_put_contents($mappingPath, serialize($mappings)) === false) {
            throw new Exception('Failed to write mapping file: ' . $mappingPath);
        }
    }
}
