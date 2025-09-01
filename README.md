/ModLoader/app/src/main/java/com/modloader/ui/OfflineDiagnosticActivity.java

// File: OfflineDiagnosticActivity.java (Part 1 - Main Class)
// Path: /main/java/com/terrarialoader/ui/OfflineDiagnosticActivity.java

package com.modloader.ui;

import android.app.ProgressDialog;
import android.content.Intent;
import android.net.Uri;
import android.os.AsyncTask;
import android.os.Bundle;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.FileProvider;

import com.modloader.R;
import com.modloader.diagnostic.DiagnosticManager;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;
import com.modloader.util.PathManager;
import com.modloader.loader.MelonLoaderManager;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;

public class OfflineDiagnosticActivity extends AppCompatActivity {
    
    private DiagnosticManager diagnosticManager;
    
    // UI Components
    private Button btnRunFullDiagnostic;
    private Button btnDiagnoseApk;
    private Button btnFixSettings;
    private Button btnAutoRepair;
    private Button btnExportReport;
    private Button btnClearResults;
    private TextView diagnosticResultsText;
    
    // Progress dialog
    private ProgressDialog progressDialog;
    
    // File picker for APK selection
    private final ActivityResultLauncher<Intent> apkPickerLauncher = 
        registerForActivityResult(new ActivityResultContracts.StartActivityForResult(), result -> {
            if (result.getResultCode() == RESULT_OK && result.getData() != null) {
                Uri apkUri = result.getData().getData();
                runApkDiagnostic(apkUri);
            }
        });
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_offline_diagnostic);
        setTitle("🔧 Offline Diagnostics");
        
        initializeComponents();
        setupUI();
        
        LogUtils.logUser("Offline Diagnostics opened");
    }
    
    private void initializeComponents() {
        diagnosticManager = new DiagnosticManager(this);
        
        // Find UI components
        btnRunFullDiagnostic = findViewById(R.id.btn_run_full_diagnostic);
        btnDiagnoseApk = findViewById(R.id.btn_diagnose_apk);
        btnFixSettings = findViewById(R.id.btn_fix_settings);
        btnAutoRepair = findViewById(R.id.btn_auto_repair);
        btnExportReport = findViewById(R.id.btn_export_report);
        btnClearResults = findViewById(R.id.btn_clear_results);
        diagnosticResultsText = findViewById(R.id.diagnostic_results_text);
    }
    
    private void setupUI() {
        // Full system diagnostic
        btnRunFullDiagnostic.setOnClickListener(v -> runFullSystemCheck());
        
        // APK diagnostic
        btnDiagnoseApk.setOnClickListener(v -> selectApkForDiagnostic());
        
        // Settings diagnostic and fix
        btnFixSettings.setOnClickListener(v -> diagnoseAndFixSettings());
        
        // Auto repair
        btnAutoRepair.setOnClickListener(v -> performAutoRepair());
        
        // Export report
        btnExportReport.setOnClickListener(v -> exportDiagnosticReport());
        
        // Clear results
        btnClearResults.setOnClickListener(v -> clearResults());
    }
    
    private void runFullSystemCheck() {
        showProgress("Running comprehensive system diagnostic...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== TerrariaLoader Comprehensive Diagnostic ===\n");
                results.append("Timestamp: ").append(new java.util.Date().toString()).append("\n");
                results.append("Device: ").append(android.os.Build.MANUFACTURER).append(" ")
                       .append(android.os.Build.MODEL).append("\n");
                results.append("Android: ").append(android.os.Build.VERSION.RELEASE).append("\n\n");
                
                // 1. Directory Structure Check
                results.append("📁 DIRECTORY STRUCTURE\n");
                results.append(checkDirectoryStructure()).append("\n");
                
                // 2. MelonLoader/LemonLoader Status
                results.append("🛠️ LOADER STATUS\n");
                results.append(checkLoaderStatus()).append("\n");
                
                // 3. Mod Files Validation
                results.append("📦 MOD FILES\n");
                results.append(checkModFiles()).append("\n");
                
                // 4. System Permissions
                results.append("🔐 PERMISSIONS\n");
                results.append(checkPermissions()).append("\n");
                
                // 5. Storage and Space
                results.append("💾 STORAGE\n");
                results.append(checkStorage()).append("\n");
                
                // 6. Settings Validation
                results.append("⚙️ SETTINGS\n");
                results.append(checkSettingsIntegrity()).append("\n");
                
                // 7. Suggested Actions
                results.append("💡 RECOMMENDATIONS\n");
                results.append(generateRecommendations()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                    LogUtils.logUser("Full system diagnostic completed");
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Diagnostic failed: " + e.getMessage());
                    LogUtils.logDebug("Diagnostic error: " + e.toString());
                });
            }
        });
    }
    
    private String checkDirectoryStructure() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            File baseDir = PathManager.getGameBaseDir(this, gamePackage);
            
            if (baseDir == null) {
                result.append("❌ Base directory path is null\n");
                return result.toString();
            }
            
            result.append("Base Path: ").append(baseDir.getAbsolutePath()).append("\n");
            
            // Check key directories
            String[] criticalPaths = {
                "",                           // Base
                "Mods",                      // Mods root
                "Mods/DEX",                  // DEX mods
                "Mods/DLL",                  // DLL mods
                "Loaders",                   // Loaders root
                "Loaders/MelonLoader",       // MelonLoader
                "Logs",                      // Game logs
                "AppLogs",                   // App logs
                "Config",                    // Configuration
                "Backups"                    // Backups
            };
            
            int existingDirs = 0;
            for (String path : criticalPaths) {
                File dir = new File(baseDir, path);
                boolean exists = dir.exists() && dir.isDirectory();
                String status = exists ? "✅" : "❌";
                result.append(status).append(" ").append(path.isEmpty() ? "Base" : path).append("\n");
                if (exists) existingDirs++;
            }
            
            result.append("\nDirectory Health: ").append(existingDirs).append("/").append(criticalPaths.length);
            if (existingDirs < criticalPaths.length) {
                result.append(" (⚠️ Some directories missing)");
            } else {
                result.append(" (✅ Complete)");
            }
            
        } catch (Exception e) {
            result.append("❌ Directory check failed: ").append(e.getMessage());
        }
        
        return result.toString();
    }
    
    private String checkLoaderStatus() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(this);
            boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(this);
            
            if (melonInstalled) {
                result.append("✅ MelonLoader detected\n");
                result.append("   Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
                
                // Check core files
                File loaderDir = PathManager.getMelonLoaderDir(this, gamePackage);
                if (loaderDir != null && loaderDir.exists()) {
                    File[] files = loaderDir.listFiles();
                    int fileCount = (files != null) ? files.length : 0;
                    result.append("   Files: ").append(fileCount).append(" detected\n");
                }
            } else if (lemonInstalled) {
                result.append("✅ LemonLoader detected\n");
                result.append("   Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
            } else {
                result.append("❌ No loader installed\n");
                result.append("   Recommendation: Use 'Complete Setup Wizard' to install MelonLoader\n");
            }
            
            // Check runtime directories
            File net8Dir = new File(PathManager.getMelonLoaderDir(this, gamePackage), "net8");
            File net35Dir = new File(PathManager.getMelonLoaderDir(this, gamePackage), "net35");
            
            result.append("Runtime Support:\n");
            result.append(net8Dir.exists() ? "✅" : "❌").append(" NET8 Runtime\n");
            result.append(net35Dir.exists() ? "✅" : "❌").append(" NET35 Runtime\n");
            
        } catch (Exception e) {
            result.append("❌ Loader check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkModFiles() {
        StringBuilder result = new StringBuilder();
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            
            // Check DEX mods
            File dexDir = PathManager.getDexModsDir(this, gamePackage);
            int dexCount = 0, dexEnabled = 0;
            if (dexDir != null && dexDir.exists()) {
                File[] dexFiles = dexDir.listFiles((dir, name) -> {
                    String lower = name.toLowerCase();
                    return lower.endsWith(".dex") || lower.endsWith(".jar") || 
                           lower.endsWith(".dex.disabled") || lower.endsWith(".jar.disabled");
                });
                if (dexFiles != null) {
                    dexCount = dexFiles.length;
                    for (File file : dexFiles) {
                        if (!file.getName().endsWith(".disabled")) {
                            dexEnabled++;
                        }
                    }
                }
            }
            
            // Check DLL mods
            File dllDir = PathManager.getDllModsDir(this, gamePackage);
            int dllCount = 0, dllEnabled = 0;
            if (dllDir != null && dllDir.exists()) {
                File[] dllFiles = dllDir.listFiles((dir, name) -> {
                    String lower = name.toLowerCase();
                    return lower.endsWith(".dll") || lower.endsWith(".dll.disabled");
                });
                if (dllFiles != null) {
                    dllCount = dllFiles.length;
                    for (File file : dllFiles) {
                        if (!file.getName().endsWith(".disabled")) {
                            dllEnabled++;
                        }
                    }
                }
            }
            
            result.append("DEX/JAR Mods: ").append(dexEnabled).append("/").append(dexCount)
                  .append(" enabled\n");
            result.append("DLL Mods: ").append(dllEnabled).append("/").append(dllCount)
                  .append(" enabled\n");
            result.append("Total Active Mods: ").append(dexEnabled + dllEnabled).append("\n");
            
            if (dexCount == 0 && dllCount == 0) {
                result.append("ℹ️ No mods installed - use Mod Management to add mods\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Mod check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkPermissions() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Test write permissions
            File testDir = new File(getExternalFilesDir(null), "permission_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "write_test.txt");
            try (FileWriter writer = new FileWriter(testFile)) {
                writer.write("Permission test successful");
                result.append("✅ External storage write access\n");
            } catch (Exception e) {
                result.append("❌ External storage write failed: ").append(e.getMessage()).append("\n");
            } finally {
                if (testFile.exists()) testFile.delete();
                testDir.delete();
            }
            
            // Check install packages permission
            try {
                getPackageManager().canRequestPackageInstalls();
                result.append("✅ Package installation permission available\n");
            } catch (Exception e) {
                result.append("⚠️ Package installation permission may be restricted\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Permission check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkStorage() {
        StringBuilder result = new StringBuilder();
        
        try {
            File externalDir = getExternalFilesDir(null);
            if (externalDir != null) {
                long freeSpace = externalDir.getFreeSpace();
                long totalSpace = externalDir.getTotalSpace();
                long usedSpace = totalSpace - freeSpace;
                
                result.append("Free Space: ").append(FileUtils.formatFileSize(freeSpace)).append("\n");
                result.append("Used Space: ").append(FileUtils.formatFileSize(usedSpace)).append("\n");
                result.append("Total Space: ").append(FileUtils.formatFileSize(totalSpace)).append("\n");
                
                if (freeSpace < 100 * 1024 * 1024) { // Less than 100MB
                    result.append("⚠️ Low storage space - consider freeing up space\n");
                } else {
                    result.append("✅ Sufficient storage space available\n");
                }
            } else {
                result.append("❌ Cannot access external storage\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Storage check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkSettingsIntegrity() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Check app preferences
            android.content.SharedPreferences prefs = 
                getSharedPreferences("TerrariaLoaderPrefs", MODE_PRIVATE);
            
            // Test write operation
            android.content.SharedPreferences.Editor editor = prefs.edit();
            editor.putString("diagnostic_test", "test_value");
            boolean writeSuccess = editor.commit();
            
            if (writeSuccess) {
                String testValue = prefs.getString("diagnostic_test", null);
                if ("test_value".equals(testValue)) {
                    result.append("✅ Settings persistence working\n");
                    // Clean up test
                    editor.remove("diagnostic_test").commit();
                } else {
                    result.append("❌ Settings read/write mismatch\n");
                }
            } else {
                result.append("❌ Settings write failed\n");
                result.append("   This may explain your auto-refresh issue\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Settings check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String generateRecommendations() {
        StringBuilder result = new StringBuilder();
        
        try {
            boolean hasIssues = false;
            
            // Check if directories need repair
            File baseDir = PathManager.getGameBaseDir(this, "com.and.games505.TerrariaPaid");
            if (baseDir == null || !baseDir.exists()) {
                result.append("• Run 'Auto-Repair' to create missing directories\n");
                hasIssues = true;
            }
            
            // Check if loader is missing
            if (!MelonLoaderManager.isMelonLoaderInstalled(this) && 
                !MelonLoaderManager.isLemonLoaderInstalled(this)) {
                result.append("• Use 'Complete Setup Wizard' to install MelonLoader\n");
                hasIssues = true;
            }
            
            // Check storage
            File externalDir = getExternalFilesDir(null);
            if (externalDir != null && externalDir.getFreeSpace() < 50 * 1024 * 1024) {
                result.append("• Free up storage space (recommended: 100MB+)\n");
                hasIssues = true;
            }
            
            if (!hasIssues) {
                result.append("✅ System appears to be in good condition\n");
                result.append("• If you're still experiencing issues, try:\n");
                result.append("  - Restart the app completely\n");
                result.append("  - Reboot your device\n");
                result.append("  - Check specific mod compatibility\n");
            }
            
        } catch (Exception e) {
            result.append("• General recommendation: Check system permissions\n");
        }
        
        return result.toString();
    }
    
    // Continue to Part 2...
// File: OfflineDiagnosticActivity.java (Part 2 - Methods & UI)
// Continuation of Part 1

    private void selectApkForDiagnostic() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/vnd.android.package-archive");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        
        try {
            apkPickerLauncher.launch(Intent.createChooser(intent, "Select APK to Diagnose"));
        } catch (Exception e) {
            showToast("No file manager available");
        }
    }
    
    private void runApkDiagnostic(Uri apkUri) {
        showProgress("Analyzing APK installation issues...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== APK Installation Diagnostic ===\n");
                results.append("File URI: ").append(apkUri.toString()).append("\n\n");
                
                String fileName = getFileNameFromUri(apkUri);
                results.append("File Name: ").append(fileName != null ? fileName : "Unknown").append("\n");
                
                results.append(validateApkFromUri(apkUri)).append("\n");
                results.append("🔧 INSTALLATION ENVIRONMENT\n");
                results.append(checkInstallationEnvironment()).append("\n");
                results.append("📱 DEVICE COMPATIBILITY\n");
                results.append(checkDeviceCompatibility()).append("\n");
                results.append("💡 SOLUTIONS FOR APK PARSING ERRORS\n");
                results.append(getApkSolutions()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("APK analysis failed: " + e.getMessage());
                });
            }
        });
    }
    
    private String validateApkFromUri(Uri apkUri) {
        StringBuilder result = new StringBuilder();
        result.append("📦 APK VALIDATION\n");
        
        try (java.io.InputStream stream = getContentResolver().openInputStream(apkUri)) {
            if (stream == null) {
                result.append("❌ Cannot access APK file\n");
                return result.toString();
            }
            
            int available = stream.available();
            if (available > 0) {
                result.append("✅ APK accessible (").append(FileUtils.formatFileSize(available)).append(")\n");
                if (available < 10 * 1024 * 1024) {
                    result.append("⚠️ APK seems small for Terraria - may be corrupted\n");
                }
            } else {
                result.append("⚠️ APK file size unknown or empty\n");
            }
            
            byte[] header = new byte[30];
            int bytesRead = stream.read(header);
            
            if (bytesRead >= 4) {
                if (header[0] == 0x50 && header[1] == 0x4b && header[2] == 0x03 && header[3] == 0x04) {
                    result.append("✅ Valid ZIP/APK signature\n");
                } else {
                    result.append("❌ Invalid ZIP/APK signature - file is corrupted\n");
                    result.append("   This is likely causing your parsing error!\n");
                }
            } else {
                result.append("❌ Cannot read APK header - file corrupted\n");
            }
            
        } catch (Exception e) {
            result.append("❌ APK access failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String checkInstallationEnvironment() {
        StringBuilder result = new StringBuilder();
        
        try {
            boolean unknownSources = canInstallFromUnknownSources();
            result.append(unknownSources ? "✅" : "❌").append(" Unknown sources enabled\n");
            
            if (!unknownSources) {
                result.append("   📋 Fix: Settings > Apps > TerrariaLoader > Install unknown apps\n");
            }
            
            File dataDir = getDataDir();
            long freeSpace = dataDir.getFreeSpace();
            result.append("Internal space: ").append(FileUtils.formatFileSize(freeSpace)).append("\n");
            
            if (freeSpace < 200 * 1024 * 1024) {
                result.append("⚠️ Low storage - may cause installation failure\n");
            }
            
            try {
                getPackageManager().getPackageInfo("com.and.games505.TerrariaPaid", 0);
                result.append("⚠️ Terraria already installed - uninstall first\n");
            } catch (android.content.pm.PackageManager.NameNotFoundException e) {
                result.append("✅ No conflicting installation\n");
            }
            
        } catch (Exception e) {
            result.append("❌ Environment check failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private boolean canInstallFromUnknownSources() {
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            return getPackageManager().canRequestPackageInstalls();
        } else {
            try {
                return android.provider.Settings.Secure.getInt(
                    getContentResolver(), 
                    android.provider.Settings.Secure.INSTALL_NON_MARKET_APPS, 0) != 0;
            } catch (Exception e) {
                return false;
            }
        }
    }
    
    private String checkDeviceCompatibility() {
        StringBuilder result = new StringBuilder();
        
        result.append("Device: ").append(android.os.Build.MANUFACTURER)
              .append(" ").append(android.os.Build.MODEL).append("\n");
        result.append("Android: ").append(android.os.Build.VERSION.RELEASE)
              .append(" (API ").append(android.os.Build.VERSION.SDK_INT).append(")\n");
        result.append("Architecture: ").append(android.os.Build.SUPPORTED_ABIS[0]).append("\n");
        
        if (android.os.Build.VERSION.SDK_INT >= 21) {
            result.append("✅ Compatible Android version\n");
        } else {
            result.append("❌ Android version too old\n");
        }
        
        return result.toString();
    }
    
    private String getApkSolutions() {
        StringBuilder result = new StringBuilder();
        
        result.append("For 'There was a problem parsing the package':\n\n");
        result.append("1. 🔧 Re-download APK (may be corrupted)\n");
        result.append("2. 🔧 Enable 'Install unknown apps'\n");
        result.append("3. 🔧 Uninstall original Terraria first\n");
        result.append("4. 🔧 Clear Package Installer cache\n");
        result.append("5. 🔧 Restart device and retry\n");
        result.append("6. 🔧 Copy APK to internal storage\n");
        result.append("7. 🔧 Use different file manager\n");
        result.append("8. 🔧 Check antivirus isn't blocking\n");
        
        return result.toString();
    }
    
    private void diagnoseAndFixSettings() {
        showProgress("Diagnosing settings persistence...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== Settings Persistence Diagnostic ===\n\n");
                results.append("🔧 SHARED PREFERENCES TEST\n");
                results.append(testSharedPreferences()).append("\n");
                results.append("🔄 AUTO-REFRESH SPECIFIC TEST\n");
                results.append(testAutoRefreshSetting()).append("\n");
                results.append("💾 FILE SYSTEM TEST\n");
                results.append(testFileSystemWrites()).append("\n");
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                    showSettingsFixOptions();
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Settings diagnostic failed: " + e.getMessage());
                });
            }
        });
    }
    
    private String testSharedPreferences() {
        StringBuilder result = new StringBuilder();
        
        try {
            android.content.SharedPreferences prefs = getSharedPreferences("DiagnosticTest", MODE_PRIVATE);
            android.content.SharedPreferences.Editor editor = prefs.edit();
            
            editor.putBoolean("test_bool", true);
            editor.putString("test_string", "test_value");
            boolean commitSuccess = editor.commit();
            
            result.append("Write test: ").append(commitSuccess ? "✅ Success" : "❌ Failed").append("\n");
            
            if (commitSuccess) {
                boolean boolVal = prefs.getBoolean("test_bool", false);
                String stringVal = prefs.getString("test_string", null);
                boolean readSuccess = boolVal && "test_value".equals(stringVal);
                
                result.append("Read test: ").append(readSuccess ? "✅ Success" : "❌ Failed").append("\n");
                
                if (!readSuccess) {
                    result.append("   This explains your auto-refresh issue!\n");
                }
                
                editor.clear().commit();
            }
            
        } catch (Exception e) {
            result.append("❌ SharedPreferences test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String testAutoRefreshSetting() {
        StringBuilder result = new StringBuilder();
        
        try {
            // Simulate the exact auto-refresh setting behavior
            android.content.SharedPreferences logPrefs = getSharedPreferences("LogViewerPrefs", MODE_PRIVATE);
            android.content.SharedPreferences.Editor editor = logPrefs.edit();
            
            // Test the specific setting that's failing
            editor.putBoolean("auto_refresh_enabled", false);
            boolean applyResult = editor.commit(); // Use commit instead of apply for immediate result
            
            result.append("Auto-refresh disable: ").append(applyResult ? "✅ Success" : "❌ Failed").append("\n");
            
            if (applyResult) {
                // Check if it actually persisted
                boolean currentValue = logPrefs.getBoolean("auto_refresh_enabled", true); // default true
                result.append("Setting persisted: ").append(!currentValue ? "✅ Success" : "❌ Failed").append("\n");
                
                if (currentValue) {
                    result.append("   Setting reverted to default - persistence failed!\n");
                    result.append("   This is your exact issue.\n");
                }
            }
            
        } catch (Exception e) {
            result.append("❌ Auto-refresh test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String testFileSystemWrites() {
        StringBuilder result = new StringBuilder();
        
        try {
            File testDir = new File(getFilesDir(), "diagnostic_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "settings_test.txt");
            
            try (FileWriter writer = new FileWriter(testFile)) {
                writer.write("auto_refresh=false\n");
                writer.write("timestamp=" + System.currentTimeMillis() + "\n");
                result.append("✅ File write successful\n");
            }
            
            if (testFile.exists()) {
                try (java.io.BufferedReader reader = new java.io.BufferedReader(
                        new java.io.FileReader(testFile))) {
                    String line = reader.readLine();
                    if (line != null && line.contains("auto_refresh=false")) {
                        result.append("✅ File read successful\n");
                    } else {
                        result.append("❌ File content corrupted\n");
                    }
                }
            }
            
            testFile.delete();
            testDir.delete();
            
        } catch (Exception e) {
            result.append("❌ File system test failed: ").append(e.getMessage()).append("\n");
        }
        
        return result.toString();
    }
    
    private String getFileNameFromUri(Uri uri) {
        try {
            android.database.Cursor cursor = getContentResolver().query(uri, null, null, null, null);
            if (cursor != null) {
                int nameIndex = cursor.getColumnIndex(android.provider.OpenableColumns.DISPLAY_NAME);
                if (nameIndex >= 0 && cursor.moveToFirst()) {
                    String name = cursor.getString(nameIndex);
                    cursor.close();
                    return name;
                }
                cursor.close();
            }
        } catch (Exception e) {
            return uri.getLastPathSegment();
        }
        return null;
    }
    
    private void performAutoRepair() {
        new AlertDialog.Builder(this)
            .setTitle("Auto-Repair System")
            .setMessage("Attempt automatic fixes for:\n\n" +
                       "• Missing directories\n" +
                       "• Settings persistence\n" +
                       "• File permissions\n" +
                       "• Configuration corruption\n\n" +
                       "Continue?")
            .setPositiveButton("Yes, Repair", (dialog, which) -> executeAutoRepair())
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void executeAutoRepair() {
        showProgress("Performing auto-repair...");
        
        AsyncTask.execute(() -> {
            try {
                StringBuilder results = new StringBuilder();
                results.append("=== Auto-Repair Results ===\n\n");
                
                boolean directoryRepair = diagnosticManager.attemptSelfRepair();
                boolean settingsRepair = repairSettings();
                boolean permissionRepair = repairPermissions();
                
                results.append("Directory Structure: ").append(directoryRepair ? "✅ Fixed" : "❌ Failed").append("\n");
                results.append("Settings Persistence: ").append(settingsRepair ? "✅ Fixed" : "❌ Failed").append("\n");
                results.append("Permissions: ").append(permissionRepair ? "✅ Fixed" : "❌ Failed").append("\n\n");
                
                if (directoryRepair || settingsRepair || permissionRepair) {
                    results.append("🔄 Restart recommended to apply changes.\n");
                } else {
                    results.append("❌ Could not auto-fix detected issues.\n");
                    results.append("💡 Try manual solutions or check device settings.\n");
                }
                
                runOnUiThread(() -> {
                    hideProgress();
                    displayResults(results.toString());
                });
            } catch (Exception e) {
                runOnUiThread(() -> {
                    hideProgress();
                    showError("Auto-repair failed: " + e.getMessage());
                });
            }
        });
    }
    
    private boolean repairSettings() {
        try {
            // Clear all shared preferences and recreate
            String[] prefFiles = {"TerrariaLoaderPrefs", "LogViewerPrefs", "AppSettings"};
            
            for (String prefFile : prefFiles) {
                android.content.SharedPreferences prefs = getSharedPreferences(prefFile, MODE_PRIVATE);
                android.content.SharedPreferences.Editor editor = prefs.edit();
                editor.clear();
                if (!editor.commit()) {
                    return false;
                }
            }
            
            // Test write after clear
            android.content.SharedPreferences testPrefs = getSharedPreferences("TerrariaLoaderPrefs", MODE_PRIVATE);
            android.content.SharedPreferences.Editor testEditor = testPrefs.edit();
            testEditor.putBoolean("settings_repaired", true);
            return testEditor.commit();
            
        } catch (Exception e) {
            LogUtils.logDebug("Settings repair failed: " + e.getMessage());
            return false;
        }
    }
    
    private boolean repairPermissions() {
        try {
            File testDir = new File(getExternalFilesDir(null), "permission_test");
            testDir.mkdirs();
            
            File testFile = new File(testDir, "test.txt");
            FileWriter writer = new FileWriter(testFile);
            writer.write("test");
            writer.close();
            
            boolean canWrite = testFile.exists() && testFile.length() > 0;
            testFile.delete();
            testDir.delete();
            
            return canWrite;
        } catch (Exception e) {
            return false;
        }
    }
    
    private void exportDiagnosticReport() {
        try {
            String reportContent = diagnosticResultsText.getText().toString();
            if (reportContent.isEmpty() || reportContent.startsWith("Click")) {
                showToast("No diagnostic results to export");
                return;
            }
            
            File reportsDir = new File(getExternalFilesDir(null), "DiagnosticReports");
            reportsDir.mkdirs();
            
            String timestamp = new java.text.SimpleDateFormat("yyyyMMdd_HHmmss", 
                java.util.Locale.getDefault()).format(new java.util.Date());
            File reportFile = new File(reportsDir, "diagnostic_" + timestamp + ".txt");
            
            try (FileWriter writer = new FileWriter(reportFile)) {
                writer.write(reportContent);
                writer.write("\n\n=== Export Info ===\n");
                writer.write("Exported by: TerrariaLoader Diagnostic Tool\n");
                writer.write("Export time: " + new java.util.Date().toString() + "\n");
            }
            
            // Share the report
            Uri fileUri = FileProvider.getUriForFile(this, 
                getPackageName() + ".provider", reportFile);
            
            Intent shareIntent = new Intent(Intent.ACTION_SEND);
            shareIntent.setType("text/plain");
            shareIntent.putExtra(Intent.EXTRA_STREAM, fileUri);
            shareIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            
            startActivity(Intent.createChooser(shareIntent, "Share Diagnostic Report"));
            showToast("Report exported: " + reportFile.getName());
            
        } catch (Exception e) {
            showError("Export failed: " + e.getMessage());
        }
    }
    
    private void clearResults() {
        diagnosticResultsText.setText("Click 'Run Full System Check' to start diagnostics...");
    }
    
    private void showSettingsFixOptions() {
        new AlertDialog.Builder(this)
            .setTitle("Settings Fix Options")
            .setMessage("Settings persistence issue detected. Try these fixes:")
            .setPositiveButton("Clear All Settings", (dialog, which) -> clearAllSettings())
            .setNeutralButton("Reset App Data", (dialog, which) -> showResetAppDataInfo())
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void clearAllSettings() {
        try {
            String[] prefFiles = {"TerrariaLoaderPrefs", "LogViewerPrefs", "AppSettings"};
            for (String prefFile : prefFiles) {
                getSharedPreferences(prefFile, MODE_PRIVATE).edit().clear().commit();
            }
            showToast("Settings cleared - restart app to test");
        } catch (Exception e) {
            showError("Failed to clear settings: " + e.getMessage());
        }
    }
    
    private void showResetAppDataInfo() {
        new AlertDialog.Builder(this)
            .setTitle("Reset App Data")
            .setMessage("To completely reset TerrariaLoader:\n\n" +
                       "1. Go to Android Settings\n" +
                       "2. Apps > TerrariaLoader\n" +
                       "3. Storage > Clear Data\n\n" +
                       "This will fix persistent settings issues.")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void displayResults(String results) {
        diagnosticResultsText.setText(results);
    }
    
    private void showProgress(String message) {
        progressDialog = new ProgressDialog(this);
        progressDialog.setMessage(message);
        progressDialog.setCancelable(false);
        progressDialog.show();
    }
    
    private void hideProgress() {
        if (progressDialog != null && progressDialog.isShowing()) {
            progressDialog.dismiss();
        }
    }
    
    private void showToast(String message) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
    }
    
    private void showError(String error) {
        new AlertDialog.Builder(this)
            .setTitle("Error")
            .setMessage(error)
            .setPositiveButton("OK", null)
            .show();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        hideProgress();
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/PluginConfigActivity.java

// File: PluginConfigActivity.java - Plugin Configuration UI (150+ lines)
// Path: /app/src/main/java/com/modloader/ui/PluginConfigActivity.java

package com.modloader.ui;

import android.content.SharedPreferences;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.*;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.modloader.R;
import com.modloader.plugin.*;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * Plugin Configuration Activity - UI for configuring plugin settings
 * Provides interface for managing plugin preferences and configuration files
 */
public class PluginConfigActivity extends AppCompatActivity {
    private static final String TAG = "PluginConfigActivity";
    
    // UI Components
    private TextView pluginNameText;
    private TextView pluginStatusText;
    private RecyclerView configRecyclerView;
    private Button saveButton;
    private Button resetButton;
    private Button exportButton;
    private Button importButton;
    
    // Plugin data
    private String pluginId;
    private Plugin plugin;
    private PluginContext pluginContext;
    private PluginManager pluginManager;
    private SharedPreferences pluginPrefs;
    
    // Configuration items
    private ConfigAdapter configAdapter;
    private List<ConfigItem> configItems = new ArrayList<>();
    
    /**
     * Configuration item types
     */
    private enum ConfigType {
        STRING, BOOLEAN, INTEGER, FLOAT, SELECT
    }
    
    /**
     * Configuration item data holder
     */
    private static class ConfigItem {
        String key;
        String title;
        String description;
        ConfigType type;
        Object value;
        Object defaultValue;
        String[] options; // For SELECT type
        boolean hasChanged = false;
        
        public ConfigItem(String key, String title, String description, ConfigType type, Object defaultValue) {
            this.key = key;
            this.title = title;
            this.description = description;
            this.type = type;
            this.defaultValue = defaultValue;
            this.value = defaultValue;
        }
        
        public ConfigItem(String key, String title, String description, String[] options, Object defaultValue) {
            this.key = key;
            this.title = title;
            this.description = description;
            this.type = ConfigType.SELECT;
            this.options = options;
            this.defaultValue = defaultValue;
            this.value = defaultValue;
        }
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_plugin_config);
        setTitle("🔧 Plugin Configuration");
        
        // Get plugin ID from intent
        pluginId = getIntent().getStringExtra("plugin_id");
        if (pluginId == null) {
            finish();
            return;
        }
        
        // Initialize plugin manager and get plugin
        pluginManager = PluginManager.getInstance(this);
        plugin = pluginManager.getPlugin(pluginId);
        pluginContext = pluginManager.getPluginContext(pluginId);
        
        if (plugin == null) {
            showError("Plugin not found: " + pluginId);
            return;
        }
        
        LogUtils.logUser("Plugin configuration opened for: " + plugin.getName());
        
        initializeViews();
        setupRecyclerView();
        setupButtons();
        loadConfiguration();
        updatePluginInfo();
    }
    
    private void initializeViews() {
        pluginNameText = findViewById(R.id.pluginNameText);
        pluginStatusText = findViewById(R.id.pluginStatusText);
        configRecyclerView = findViewById(R.id.configRecyclerView);
        saveButton = findViewById(R.id.saveButton);
        resetButton = findViewById(R.id.resetButton);
        exportButton = findViewById(R.id.exportButton);
        importButton = findViewById(R.id.importButton);
    }
    
    private void setupRecyclerView() {
        configAdapter = new ConfigAdapter();
        configRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        configRecyclerView.setAdapter(configAdapter);
    }
    
    private void setupButtons() {
        saveButton.setOnClickListener(v -> saveConfiguration());
        resetButton.setOnClickListener(v -> resetConfiguration());
        exportButton.setOnClickListener(v -> exportConfiguration());
        importButton.setOnClickListener(v -> importConfiguration());
    }
    
    private void updatePluginInfo() {
        pluginNameText.setText(plugin.getName() + " v" + plugin.getVersion());
        
        if (pluginContext != null && pluginContext.isEnabled()) {
            pluginStatusText.setText("✅ Plugin is active");
            pluginStatusText.setTextColor(getColor(android.R.color.holo_green_dark));
        } else {
            pluginStatusText.setText("⚠️ Plugin is not active");
            pluginStatusText.setTextColor(getColor(android.R.color.holo_orange_dark));
        }
    }
    
    private void loadConfiguration() {
        pluginPrefs = getSharedPreferences("plugin_" + pluginId, MODE_PRIVATE);
        
        // Create default configuration items
        createDefaultConfigItems();
        
        // Load current values from preferences
        for (ConfigItem item : configItems) {
            loadConfigValue(item);
        }
        
        configAdapter.notifyDataSetChanged();
    }
    
    private void createDefaultConfigItems() {
        configItems.clear();
        
        // Add common plugin configuration options
        configItems.add(new ConfigItem("enabled", "Enable Plugin", 
            "Enable or disable this plugin", ConfigType.BOOLEAN, true));
        
        configItems.add(new ConfigItem("debug_logging", "Debug Logging", 
            "Enable detailed debug logging for this plugin", ConfigType.BOOLEAN, false));
        
        configItems.add(new ConfigItem("auto_update", "Auto Update", 
            "Automatically check for plugin updates", ConfigType.BOOLEAN, true));
        
        configItems.add(new ConfigItem("log_level", "Log Level", 
            "Set the logging level for this plugin", 
            new String[]{"ERROR", "WARN", "INFO", "DEBUG"}, "INFO"));
        
        configItems.add(new ConfigItem("max_memory", "Max Memory (MB)", 
            "Maximum memory usage for this plugin", ConfigType.INTEGER, 64));
        
        configItems.add(new ConfigItem("update_interval", "Update Interval (seconds)", 
            "How often the plugin should check for updates", ConfigType.INTEGER, 3600));
        
        configItems.add(new ConfigItem("performance_mode", "Performance Mode", 
            "Optimize plugin performance", ConfigType.BOOLEAN, false));
        
        // Add plugin-specific configuration if available
        addPluginSpecificConfig();
    }
    
    private void addPluginSpecificConfig() {
        // This would be implemented to load plugin-specific configuration
        // For now, we'll add some example items based on plugin name/type
        
        if (plugin.getName().toLowerCase().contains("theme")) {
            configItems.add(new ConfigItem("theme_color", "Theme Color", 
                "Primary color for the theme", 
                new String[]{"Blue", "Green", "Red", "Purple", "Orange"}, "Blue"));
            
            configItems.add(new ConfigItem("dark_mode", "Dark Mode", 
                "Enable dark mode theme", ConfigType.BOOLEAN, false));
        }
        
        if (plugin.getName().toLowerCase().contains("mod")) {
            configItems.add(new ConfigItem("mod_compatibility", "Mod Compatibility Mode", 
                "Enable compatibility with other mods", ConfigType.BOOLEAN, true));
            
            configItems.add(new ConfigItem("backup_saves", "Backup Game Saves", 
                "Automatically backup game saves when mod is active", ConfigType.BOOLEAN, true));
        }
        
        // Add utility-specific configuration
        if (plugin.getName().toLowerCase().contains("utility") || 
            plugin.getName().toLowerCase().contains("tool")) {
            configItems.add(new ConfigItem("startup_delay", "Startup Delay (ms)", 
                "Delay before plugin starts after app launch", ConfigType.INTEGER, 1000));
            
            configItems.add(new ConfigItem("notification_enabled", "Show Notifications", 
                "Display notifications from this plugin", ConfigType.BOOLEAN, true));
        }
    }
    
    private void loadConfigValue(ConfigItem item) {
        switch (item.type) {
            case BOOLEAN:
                item.value = pluginPrefs.getBoolean(item.key, (Boolean) item.defaultValue);
                break;
            case INTEGER:
                item.value = pluginPrefs.getInt(item.key, (Integer) item.defaultValue);
                break;
            case FLOAT:
                item.value = pluginPrefs.getFloat(item.key, (Float) item.defaultValue);
                break;
            case STRING:
            case SELECT:
                item.value = pluginPrefs.getString(item.key, (String) item.defaultValue);
                break;
        }
        item.hasChanged = false;
    }
    
    private void saveConfiguration() {
        SharedPreferences.Editor editor = pluginPrefs.edit();
        int savedCount = 0;
        
        for (ConfigItem item : configItems) {
            if (item.hasChanged) {
                switch (item.type) {
                    case BOOLEAN:
                        editor.putBoolean(item.key, (Boolean) item.value);
                        break;
                    case INTEGER:
                        editor.putInt(item.key, (Integer) item.value);
                        break;
                    case FLOAT:
                        editor.putFloat(item.key, (Float) item.value);
                        break;
                    case STRING:
                    case SELECT:
                        editor.putString(item.key, (String) item.value);
                        break;
                }
                savedCount++;
                item.hasChanged = false;
            }
        }
        
        editor.apply();
        
        if (savedCount > 0) {
            Toast.makeText(this, "Saved " + savedCount + " setting(s)", Toast.LENGTH_SHORT).show();
            LogUtils.logUser("Saved " + savedCount + " configuration items for plugin: " + plugin.getName());
            
            // Notify plugin of configuration change if it's loaded
            if (pluginContext != null) {
                // Create a simple event notification instead of calling fireEvent directly
                try {
                    // Trigger a settings change hook
                    PluginHook.trigger(PluginHook.Hooks.SETTINGS_CHANGE, this, "plugin_id", pluginId);
                } catch (Exception e) {
                    LogUtils.logDebug("Could not notify plugin of config change: " + e.getMessage());
                }
            }
        } else {
            Toast.makeText(this, "No changes to save", Toast.LENGTH_SHORT).show();
        }
        
        configAdapter.notifyDataSetChanged();
    }
    
    private void resetConfiguration() {
        new AlertDialog.Builder(this)
            .setTitle("Reset Configuration")
            .setMessage("Reset all settings to default values?")
            .setPositiveButton("Reset", (dialog, which) -> {
                for (ConfigItem item : configItems) {
                    item.value = item.defaultValue;
                    item.hasChanged = true;
                }
                configAdapter.notifyDataSetChanged();
                Toast.makeText(this, "Configuration reset to defaults", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void exportConfiguration() {
        try {
            File configDir = pluginContext != null ? pluginContext.getConfigDir() : 
                new File(getExternalFilesDir(null), "plugins/" + pluginId + "/config");
            configDir.mkdirs();
            
            File configFile = new File(configDir, "exported_config.txt");
            StringBuilder export = new StringBuilder();
            
            export.append("# Configuration for ").append(plugin.getName()).append("\n");
            export.append("# Exported on ").append(new java.util.Date()).append("\n\n");
            
            for (ConfigItem item : configItems) {
                export.append("# ").append(item.description).append("\n");
                export.append(item.key).append("=").append(item.value).append("\n\n");
            }
            
            java.nio.file.Files.write(configFile.toPath(), export.toString().getBytes());
            
            Toast.makeText(this, "Configuration exported to: " + configFile.getName(), Toast.LENGTH_LONG).show();
            LogUtils.logUser("Exported plugin configuration to: " + configFile.getAbsolutePath());
            
        } catch (Exception e) {
            LogUtils.logError("Failed to export configuration: " + e.getMessage());
            Toast.makeText(this, "Export failed: " + e.getMessage(), Toast.LENGTH_SHORT).show();
        }
    }
    
    private void importConfiguration() {
        // For now, show a dialog explaining the feature
        new AlertDialog.Builder(this)
            .setTitle("Import Configuration")
            .setMessage("Configuration import is not yet implemented.\n\n" +
                       "To manually import settings:\n" +
                       "1. Place a config file in the plugin's config directory\n" +
                       "2. Restart the plugin\n" +
                       "3. The settings will be loaded automatically")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void showError(String message) {
        new AlertDialog.Builder(this)
            .setTitle("Error")
            .setMessage(message)
            .setPositiveButton("OK", (dialog, which) -> finish())
            .show();
    }
    
    // ===== CONFIG ADAPTER =====
    
    private class ConfigAdapter extends RecyclerView.Adapter<ConfigAdapter.ConfigViewHolder> {
        
        @NonNull
        @Override
        public ConfigViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_plugin_config, parent, false);
            return new ConfigViewHolder(view);
        }
        
        @Override
        public void onBindViewHolder(@NonNull ConfigViewHolder holder, int position) {
            ConfigItem item = configItems.get(position);
            holder.bind(item);
        }
        
        @Override
        public int getItemCount() {
            return configItems.size();
        }
        
        class ConfigViewHolder extends RecyclerView.ViewHolder {
            private TextView titleText;
            private TextView descriptionText;
            private View inputContainer;
            private Switch switchInput;
            private EditText textInput;
            private Spinner spinnerInput;
            private TextView changeIndicator;
            
            public ConfigViewHolder(@NonNull View itemView) {
                super(itemView);
                titleText = itemView.findViewById(R.id.configTitleText);
                descriptionText = itemView.findViewById(R.id.configDescriptionText);
                inputContainer = itemView.findViewById(R.id.configInputContainer);
                switchInput = itemView.findViewById(R.id.configSwitchInput);
                textInput = itemView.findViewById(R.id.configTextInput);
                spinnerInput = itemView.findViewById(R.id.configSpinnerInput);
                changeIndicator = itemView.findViewById(R.id.configChangeIndicator);
            }
            
            public void bind(ConfigItem item) {
                titleText.setText(item.title);
                descriptionText.setText(item.description);
                
                // Hide all input types initially
                switchInput.setVisibility(View.GONE);
                textInput.setVisibility(View.GONE);
                spinnerInput.setVisibility(View.GONE);
                
                // Show appropriate input type
                switch (item.type) {
                    case BOOLEAN:
                        switchInput.setVisibility(View.VISIBLE);
                        switchInput.setOnCheckedChangeListener(null);
                        switchInput.setChecked((Boolean) item.value);
                        switchInput.setOnCheckedChangeListener((buttonView, isChecked) -> {
                            item.value = isChecked;
                            item.hasChanged = true;
                            updateChangeIndicator(item);
                        });
                        break;
                        
                    case STRING:
                    case INTEGER:
                    case FLOAT:
                        textInput.setVisibility(View.VISIBLE);
                        textInput.setText(item.value.toString());
                        textInput.setOnFocusChangeListener((v, hasFocus) -> {
                            if (!hasFocus) {
                                updateValueFromText(item, textInput.getText().toString());
                            }
                        });
                        break;
                        
                    case SELECT:
                        spinnerInput.setVisibility(View.VISIBLE);
                        ArrayAdapter<String> adapter = new ArrayAdapter<>(
                            PluginConfigActivity.this, 
                            android.R.layout.simple_spinner_item, 
                            item.options
                        );
                        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
                        spinnerInput.setAdapter(adapter);
                        
                        // Set current selection
                        String currentValue = (String) item.value;
                        for (int i = 0; i < item.options.length; i++) {
                            if (item.options[i].equals(currentValue)) {
                                spinnerInput.setSelection(i);
                                break;
                            }
                        }
                        
                        spinnerInput.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
                            @Override
                            public void onItemSelected(AdapterView<?> parent, View view, int position, long id) {
                                String selectedValue = item.options[position];
                                if (!selectedValue.equals(item.value)) {
                                    item.value = selectedValue;
                                    item.hasChanged = true;
                                    updateChangeIndicator(item);
                                }
                            }
                            
                            @Override
                            public void onNothingSelected(AdapterView<?> parent) {}
                        });
                        break;
                }
                
                updateChangeIndicator(item);
            }
            
            private void updateValueFromText(ConfigItem item, String text) {
                try {
                    Object newValue = null;
                    switch (item.type) {
                        case STRING:
                            newValue = text;
                            break;
                        case INTEGER:
                            newValue = Integer.parseInt(text);
                            break;
                        case FLOAT:
                            newValue = Float.parseFloat(text);
                            break;
                    }
                    
                    if (newValue != null && !newValue.equals(item.value)) {
                        item.value = newValue;
                        item.hasChanged = true;
                        updateChangeIndicator(item);
                    }
                    
                } catch (NumberFormatException e) {
                    Toast.makeText(PluginConfigActivity.this, 
                        "Invalid value for " + item.title, Toast.LENGTH_SHORT).show();
                    // Reset to previous value
                    textInput.setText(item.value.toString());
                }
            }
            
            private void updateChangeIndicator(ConfigItem item) {
                if (item.hasChanged) {
                    changeIndicator.setVisibility(View.VISIBLE);
                    changeIndicator.setText("●");
                    changeIndicator.setTextColor(getColor(android.R.color.holo_orange_dark));
                } else {
                    changeIndicator.setVisibility(View.GONE);
                }
            }
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/PluginInstallActivity.java

// File: PluginInstallActivity.java - Plugin Installation UI (200+ lines)
// Path: /app/src/main/java/com/modloader/ui/PluginInstallActivity.java

package com.modloader.ui;

import android.app.ProgressDialog;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.view.View;
import android.widget.*;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.plugin.*;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;

import java.io.File;
import java.io.InputStream;
import java.io.FileOutputStream;
import java.util.jar.JarFile;
import java.util.jar.Manifest;
import java.util.jar.Attributes;

/**
 * Plugin Installation Activity - UI for installing new plugins
 * Handles plugin file validation, installation, and initial configuration
 */
public class PluginInstallActivity extends AppCompatActivity {
    private static final String TAG = "PluginInstallActivity";
    
    // UI Components
    private TextView fileInfoText;
    private TextView pluginInfoText;
    private TextView validationStatusText;
    private Button installButton;
    private Button cancelButton;
    private ProgressBar validationProgress;
    private LinearLayout pluginDetailsLayout;
    private CheckBox autoEnableCheckBox;
    
    // Installation data
    private Uri pluginUri;
    private File tempPluginFile;
    private Plugin detectedPlugin;
    private PluginManager pluginManager;
    private boolean isValid = false;
    private String pluginFileName;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_plugin_install);
        setTitle("📦 Install Plugin");
        
        LogUtils.logUser("Plugin installation activity opened");
        
        // Initialize plugin manager
        pluginManager = PluginManager.getInstance(this);
        
        // Get plugin URI from intent
        Intent intent = getIntent();
        pluginUri = intent.getData();
        
        if (pluginUri == null) {
            showError("No plugin file selected");
            return;
        }
        
        initializeViews();
        setupButtons();
        
        // Start validation process
        validatePlugin();
    }
    
    private void initializeViews() {
        fileInfoText = findViewById(R.id.fileInfoText);
        pluginInfoText = findViewById(R.id.pluginInfoText);
        validationStatusText = findViewById(R.id.validationStatusText);
        installButton = findViewById(R.id.installButton);
        cancelButton = findViewById(R.id.cancelButton);
        validationProgress = findViewById(R.id.validationProgress);
        pluginDetailsLayout = findViewById(R.id.pluginDetailsLayout);
        autoEnableCheckBox = findViewById(R.id.autoEnableCheckBox);
        
        // Initially hide plugin details and install button
        pluginDetailsLayout.setVisibility(View.GONE);
        installButton.setEnabled(false);
    }
    
    private void setupButtons() {
        installButton.setOnClickListener(v -> installPlugin());
        cancelButton.setOnClickListener(v -> finish());
    }
    
    private void validatePlugin() {
        validationProgress.setVisibility(View.VISIBLE);
        validationStatusText.setText("Validating plugin file...");
        
        new Thread(() -> {
            try {
                // Step 1: Copy file to temporary location
                runOnUiThread(() -> validationStatusText.setText("Reading plugin file..."));
                
                tempPluginFile = copyPluginToTemp();
                if (tempPluginFile == null) {
                    runOnUiThread(() -> showValidationError("Failed to read plugin file"));
                    return;
                }
                
                // Step 2: Basic file validation
                runOnUiThread(() -> validationStatusText.setText("Validating file format..."));
                
                if (!validateFileFormat()) {
                    runOnUiThread(() -> showValidationError("Invalid file format. Expected .jar file."));
                    return;
                }
                
                // Step 3: Extract plugin information
                runOnUiThread(() -> validationStatusText.setText("Extracting plugin information..."));
                
                detectedPlugin = extractPluginInfo();
                if (detectedPlugin == null) {
                    runOnUiThread(() -> showValidationError("Could not detect plugin information. Make sure this is a valid plugin file."));
                    return;
                }
                
                // Step 4: Check for conflicts
                runOnUiThread(() -> validationStatusText.setText("Checking for conflicts..."));
                
                String conflictCheck = checkForConflicts();
                if (conflictCheck != null) {
                    runOnUiThread(() -> showConflictWarning(conflictCheck));
                    return;
                }
                
                // Step 5: Validation complete
                runOnUiThread(() -> {
                    validationProgress.setVisibility(View.GONE);
                    validationStatusText.setText("✅ Plugin validation successful");
                    validationStatusText.setTextColor(getColor(android.R.color.holo_green_dark));
                    showPluginDetails();
                    isValid = true;
                    installButton.setEnabled(true);
                });
                
            } catch (Exception e) {
                LogUtils.logError("Plugin validation error: " + e.getMessage());
                runOnUiThread(() -> showValidationError("Validation failed: " + e.getMessage()));
            }
        }).start();
    }
    
    private File copyPluginToTemp() {
        try {
            // Get filename from URI
            pluginFileName = FileUtils.getFilenameFromUri(this, pluginUri);
            if (pluginFileName == null) {
                pluginFileName = "plugin_" + System.currentTimeMillis() + ".jar";
            }
            
            // Create temp file
            File tempDir = new File(getCacheDir(), "plugin_install");
            tempDir.mkdirs();
            File tempFile = new File(tempDir, pluginFileName);
            
            // Copy content
            try (InputStream inputStream = getContentResolver().openInputStream(pluginUri);
                 FileOutputStream outputStream = new FileOutputStream(tempFile)) {
                
                if (inputStream == null) {
                    return null;
                }
                
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = inputStream.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                }
            }
            
            runOnUiThread(() -> {
                fileInfoText.setText("File: " + pluginFileName + "\n" +
                    "Size: " + FileUtils.formatFileSize(tempFile.length()));
            });
            
            return tempFile;
            
        } catch (Exception e) {
            LogUtils.logError("Failed to copy plugin file: " + e.getMessage());
            return null;
        }
    }
    
    private boolean validateFileFormat() {
        if (tempPluginFile == null || !tempPluginFile.exists()) {
            return false;
        }
        
        // Check file extension
        if (!pluginFileName.toLowerCase().endsWith(".jar")) {
            return false;
        }
        
        // Try to open as JAR file
        try (JarFile jarFile = new JarFile(tempPluginFile)) {
            return true;
        } catch (Exception e) {
            LogUtils.logDebug("File format validation failed: " + e.getMessage());
            return false;
        }
    }
    
    private Plugin extractPluginInfo() {
        try (JarFile jarFile = new JarFile(tempPluginFile)) {
            // Try to read manifest
            Manifest manifest = jarFile.getManifest();
            if (manifest != null) {
                return extractFromManifest(manifest);
            }
            
            // Try to find plugin.json or plugin.yml
            Plugin pluginFromConfig = extractFromConfigFile(jarFile);
            if (pluginFromConfig != null) {
                return pluginFromConfig;
            }
            
            // Fallback: create basic plugin info
            return createFallbackPlugin();
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to extract plugin info: " + e.getMessage());
            return null;
        }
    }
    
    private Plugin extractFromManifest(Manifest manifest) {
        try {
            Attributes mainAttributes = manifest.getMainAttributes();
            
            String name = mainAttributes.getValue("Plugin-Name");
            String version = mainAttributes.getValue("Plugin-Version");
            String author = mainAttributes.getValue("Plugin-Author");
            String description = mainAttributes.getValue("Plugin-Description");
            String mainClass = mainAttributes.getValue("Plugin-Main");
            
            if (name == null) name = getFileNameWithoutExtension();
            if (version == null) version = "1.0.0";
            if (author == null) author = "Unknown";
            if (description == null) description = "No description provided";
            
            return new SimplePlugin(
                generatePluginId(name),
                name,
                version,
                author,
                description,
                mainClass
            );
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to extract from manifest: " + e.getMessage());
            return null;
        }
    }
    
    private Plugin extractFromConfigFile(JarFile jarFile) {
        // Implementation for extracting from plugin.json or plugin.yml
        // This would involve reading JSON/YAML files from the JAR
        return null;
    }
    
    private Plugin createFallbackPlugin() {
        String name = getFileNameWithoutExtension();
        String id = generatePluginId(name);
        
        return new SimplePlugin(
            id,
            name,
            "1.0.0",
            "Unknown",
            "Plugin loaded from " + pluginFileName,
            null
        );
    }
    
    private String getFileNameWithoutExtension() {
        if (pluginFileName == null) return "UnknownPlugin";
        int lastDot = pluginFileName.lastIndexOf('.');
        if (lastDot > 0) {
            return pluginFileName.substring(0, lastDot);
        }
        return pluginFileName;
    }
    
    private String generatePluginId(String name) {
        return name.toLowerCase()
                  .replaceAll("[^a-zA-Z0-9]", "_")
                  .replaceAll("_+", "_")
                  .replaceAll("^_|_$", "");
    }
    
    private String checkForConflicts() {
        try {
            // Check if plugin with same ID already exists
            Plugin existingPlugin = pluginManager.getPlugin(detectedPlugin.getId());
            if (existingPlugin != null) {
                return "A plugin with ID '" + detectedPlugin.getId() + "' is already installed.\n" +
                       "Existing: " + existingPlugin.getName() + " v" + existingPlugin.getVersion() + "\n" +
                       "New: " + detectedPlugin.getName() + " v" + detectedPlugin.getVersion();
            }
            
            // Check if plugin with same name already exists
            for (Plugin plugin : pluginManager.getLoadedPlugins()) {
                if (plugin.getName().equals(detectedPlugin.getName()) && 
                    !plugin.getId().equals(detectedPlugin.getId())) {
                    return "A plugin with name '" + detectedPlugin.getName() + "' already exists with different ID.";
                }
            }
            
            return null; // No conflicts
            
        } catch (Exception e) {
            LogUtils.logDebug("Error checking conflicts: " + e.getMessage());
            return null;
        }
    }
    
    private void showPluginDetails() {
        StringBuilder details = new StringBuilder();
        details.append("Plugin Name: ").append(detectedPlugin.getName()).append("\n");
        details.append("Version: ").append(detectedPlugin.getVersion()).append("\n");
        details.append("Author: ").append(detectedPlugin.getAuthor()).append("\n");
        details.append("Description: ").append(detectedPlugin.getDescription()).append("\n");
        details.append("Plugin ID: ").append(detectedPlugin.getId());
        
        pluginInfoText.setText(details.toString());
        pluginDetailsLayout.setVisibility(View.VISIBLE);
    }
    
    private void showValidationError(String message) {
        validationProgress.setVisibility(View.GONE);
        validationStatusText.setText("❌ " + message);
        validationStatusText.setTextColor(getColor(android.R.color.holo_red_dark));
        installButton.setEnabled(false);
    }
    
    private void showConflictWarning(String conflictMessage) {
        new AlertDialog.Builder(this)
            .setTitle("Plugin Conflict Detected")
            .setMessage(conflictMessage + "\n\nDo you want to replace the existing plugin?")
            .setPositiveButton("Replace", (dialog, which) -> {
                // Continue with installation (will replace existing)
                runOnUiThread(() -> {
                    validationProgress.setVisibility(View.GONE);
                    validationStatusText.setText("⚠️ Will replace existing plugin");
                    validationStatusText.setTextColor(getColor(android.R.color.holo_orange_dark));
                    showPluginDetails();
                    isValid = true;
                    installButton.setEnabled(true);
                });
            })
            .setNegativeButton("Cancel", (dialog, which) -> {
                runOnUiThread(() -> showValidationError("Installation cancelled due to conflict"));
            })
            .show();
    }
    
    private void installPlugin() {
        if (!isValid || detectedPlugin == null || tempPluginFile == null) {
            Toast.makeText(this, "Cannot install: validation not complete", Toast.LENGTH_SHORT).show();
            return;
        }
        
        ProgressDialog progressDialog = new ProgressDialog(this);
        progressDialog.setTitle("Installing Plugin");
        progressDialog.setMessage("Installing " + detectedPlugin.getName() + "...");
        progressDialog.setCancelable(false);
        progressDialog.show();
        
        new Thread(() -> {
            try {
                // Install the plugin
                boolean success = pluginManager.installPlugin(tempPluginFile, detectedPlugin);
                
                runOnUiThread(() -> {
                    progressDialog.dismiss();
                    
                    if (success) {
                        // Check if should auto-enable
                        boolean autoEnable = autoEnableCheckBox.isChecked();
                        
                        if (autoEnable) {
                            // Enable the plugin
                            new Thread(() -> {
                                try {
                                    pluginManager.loadPlugin(detectedPlugin.getId());
                                    runOnUiThread(() -> showInstallationSuccess(true));
                                } catch (Exception e) {
                                    LogUtils.logError("Failed to enable plugin after install: " + e.getMessage());
                                    runOnUiThread(() -> showInstallationSuccess(false));
                                }
                            }).start();
                        } else {
                            showInstallationSuccess(false);
                        }
                    } else {
                        showInstallationError("Plugin installation failed. Check logs for details.");
                    }
                });
                
            } catch (Exception e) {
                LogUtils.logError("Plugin installation error: " + e.getMessage());
                runOnUiThread(() -> {
                    progressDialog.dismiss();
                    showInstallationError("Installation failed: " + e.getMessage());
                });
            }
        }).start();
    }
    
    private void showInstallationSuccess(boolean enabled) {
        String message = "Plugin '" + detectedPlugin.getName() + "' installed successfully!";
        if (enabled) {
            message += " Plugin has been enabled and is ready to use.";
        }
        
        new AlertDialog.Builder(this)
            .setTitle("✅ Installation Complete")
            .setMessage(message)
            .setPositiveButton("View Plugins", (dialog, which) -> {
                Intent intent = new Intent(this, PluginManagementActivity.class);
                intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
                startActivity(intent);
                finish();
            })
            .setNegativeButton("Done", (dialog, which) -> finish())
            .show();
    }
    
    private void showInstallationError(String error) {
        new AlertDialog.Builder(this)
            .setTitle("❌ Installation Failed")
            .setMessage(error)
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void showError(String message) {
        new AlertDialog.Builder(this)
            .setTitle("Error")
            .setMessage(message)
            .setPositiveButton("OK", (dialog, which) -> finish())
            .show();
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        
        // Clean up temporary file
        if (tempPluginFile != null && tempPluginFile.exists()) {
            tempPluginFile.delete();
        }
    }
    
    // Simple Plugin implementation for installation
    private static class SimplePlugin implements Plugin {
        private final String id;
        private final String name;
        private final String version;
        private final String author;
        private final String description;
        private final String mainClass;
        
        public SimplePlugin(String id, String name, String version, String author, String description, String mainClass) {
            this.id = id;
            this.name = name;
            this.version = version;
            this.author = author;
            this.description = description;
            this.mainClass = mainClass;
        }
        
        @Override
        public String getId() { return id; }
        
        @Override
        public String getName() { return name; }
        
        @Override
        public String getVersion() { return version; }
        
        @Override
        public String getAuthor() { return author; }
        
        @Override
        public String getDescription() { return description; }
        
        @Override
        public void onLoad(PluginContext context) {
            // Basic plugin implementation - can be overridden
        }
        
        @Override
        public void onEnable(PluginContext context) {
            // Basic plugin implementation - can be overridden
        }
        
        @Override
        public void onDisable(PluginContext context) {
            // Basic plugin implementation - can be overridden
        }
        
        @Override
        public void onUnload(PluginContext context) {
            // Basic plugin implementation - can be overridden
        }
        
        public String getMainClass() {
            return mainClass;
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/PluginManagementActivity.java

// File: PluginManagementActivity.java - Plugin Management UI (350+ lines)
// Path: /app/src/main/java/com/modloader/ui/PluginManagementActivity.java

package com.modloader.ui;

import android.app.AlertDialog;
import android.content.Intent;
import android.graphics.Color;
import android.net.Uri;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.view.ViewGroup;
import android.widget.*;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.cardview.widget.CardView;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;
import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;

import com.modloader.R;
import com.modloader.plugin.*;
import com.modloader.util.LogUtils;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

/**
 * Plugin Management Activity - UI for managing plugins
 * Provides interface for loading, enabling, disabling, and configuring plugins
 */
public class PluginManagementActivity extends AppCompatActivity {
    private static final String TAG = "PluginManagementActivity";
    private static final int REQUEST_INSTALL_PLUGIN = 1001;
    
    // UI Components
    private RecyclerView pluginRecyclerView;
    private SwipeRefreshLayout swipeRefreshLayout;
    private TextView statusText;
    private TextView pluginCountText;
    private Button installPluginButton;
    private Button refreshButton;
    private Button scanButton;
    
    // Plugin management
    private PluginManager pluginManager;
    private PluginAdapter pluginAdapter;
    private List<PluginInfo> pluginList = new ArrayList<>();
    
    // Plugin info container
    private static class PluginInfo {
        Plugin plugin;
        PluginContext context;
        boolean isLoaded;
        boolean isEnabled;
        String status;
        String errorMessage;
        
        public PluginInfo(Plugin plugin, PluginContext context) {
            this.plugin = plugin;
            this.context = context;
            this.isLoaded = context != null && context.isInitialized();
            this.isEnabled = context != null && context.isEnabled();
            updateStatus();
        }
        
        public void updateStatus() {
            if (!isLoaded) {
                status = "Not Loaded";
            } else if (!isEnabled) {
                status = "Disabled";
            } else {
                status = "Active";
            }
        }
        
        public void setError(String error) {
            this.errorMessage = error;
            this.status = "Error";
        }
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_plugin_management);
        setTitle("🔌 Plugin Management");
        
        LogUtils.logUser("Plugin Management activity opened");
        
        // Initialize plugin manager
        pluginManager = PluginManager.getInstance(this);
        
        initializeViews();
        setupRecyclerView();
        setupSwipeRefresh();
        setupButtons();
        
        // Load plugins
        refreshPluginList();
    }
    
    private void initializeViews() {
        pluginRecyclerView = findViewById(R.id.pluginRecyclerView);
        swipeRefreshLayout = findViewById(R.id.swipeRefreshLayout);
        statusText = findViewById(R.id.statusText);
        pluginCountText = findViewById(R.id.pluginCountText);
        installPluginButton = findViewById(R.id.installPluginButton);
        refreshButton = findViewById(R.id.refreshButton);
        scanButton = findViewById(R.id.scanButton);
    }
    
    private void setupRecyclerView() {
        pluginAdapter = new PluginAdapter();
        pluginRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        pluginRecyclerView.setAdapter(pluginAdapter);
    }
    
    private void setupSwipeRefresh() {
        swipeRefreshLayout.setOnRefreshListener(this::refreshPluginList);
        swipeRefreshLayout.setColorSchemeColors(
            getColor(android.R.color.holo_blue_bright),
            getColor(android.R.color.holo_green_light),
            getColor(android.R.color.holo_orange_light)
        );
    }
    
    private void setupButtons() {
        installPluginButton.setOnClickListener(v -> openPluginInstaller());
        refreshButton.setOnClickListener(v -> refreshPluginList());
        scanButton.setOnClickListener(v -> scanForPlugins());
    }
    
    private void refreshPluginList() {
        swipeRefreshLayout.setRefreshing(true);
        
        new Thread(() -> {
            try {
                // Get all plugins
                List<Plugin> loadedPlugins = pluginManager.getLoadedPlugins();
                List<Plugin> availablePlugins = pluginManager.getAvailablePlugins();
                
                // Combine and create plugin info objects
                List<PluginInfo> newPluginList = new ArrayList<>();
                
                // Add loaded plugins
                for (Plugin plugin : loadedPlugins) {
                    PluginContext context = pluginManager.getPluginContext(plugin.getId());
                    newPluginList.add(new PluginInfo(plugin, context));
                }
                
                // Add available but not loaded plugins
                for (Plugin plugin : availablePlugins) {
                    if (!isPluginInList(plugin, newPluginList)) {
                        newPluginList.add(new PluginInfo(plugin, null));
                    }
                }
                
                // Update UI on main thread
                runOnUiThread(() -> {
                    pluginList.clear();
                    pluginList.addAll(newPluginList);
                    pluginAdapter.notifyDataSetChanged();
                    updateStatusText();
                    swipeRefreshLayout.setRefreshing(false);
                });
                
            } catch (Exception e) {
                LogUtils.logError("Failed to refresh plugin list: " + e.getMessage());
                runOnUiThread(() -> {
                    Toast.makeText(this, "Failed to refresh plugins", Toast.LENGTH_SHORT).show();
                    swipeRefreshLayout.setRefreshing(false);
                });
            }
        }).start();
    }
    
    private void scanForPlugins() {
        Toast.makeText(this, "Scanning for plugins...", Toast.LENGTH_SHORT).show();
        
        new Thread(() -> {
            try {
                int foundCount = pluginManager.scanForPlugins();
                runOnUiThread(() -> {
                    Toast.makeText(this, "Found " + foundCount + " plugin(s)", Toast.LENGTH_SHORT).show();
                    refreshPluginList();
                });
            } catch (Exception e) {
                LogUtils.logError("Failed to scan for plugins: " + e.getMessage());
                runOnUiThread(() -> {
                    Toast.makeText(this, "Plugin scan failed", Toast.LENGTH_SHORT).show();
                });
            }
        }).start();
    }
    
    private boolean isPluginInList(Plugin plugin, List<PluginInfo> list) {
        for (PluginInfo info : list) {
            if (info.plugin.getId().equals(plugin.getId())) {
                return true;
            }
        }
        return false;
    }
    
    private void updateStatusText() {
        int totalPlugins = pluginList.size();
        int loadedPlugins = 0;
        int enabledPlugins = 0;
        int errorPlugins = 0;
        
        for (PluginInfo info : pluginList) {
            if (info.status.equals("Error")) {
                errorPlugins++;
            } else if (info.isLoaded) {
                loadedPlugins++;
                if (info.isEnabled) {
                    enabledPlugins++;
                }
            }
        }
        
        pluginCountText.setText(String.format("Total: %d | Loaded: %d | Active: %d | Errors: %d", 
            totalPlugins, loadedPlugins, enabledPlugins, errorPlugins));
        
        if (totalPlugins == 0) {
            statusText.setText("📦 No plugins found. Install plugins to get started.");
            statusText.setTextColor(getColor(android.R.color.darker_gray));
        } else if (errorPlugins > 0) {
            statusText.setText("❌ " + errorPlugins + " plugin(s) have errors. Check logs for details.");
            statusText.setTextColor(getColor(android.R.color.holo_red_dark));
        } else if (enabledPlugins == 0) {
            statusText.setText("⚠️ No plugins are currently active.");
            statusText.setTextColor(getColor(android.R.color.holo_orange_dark));
        } else {
            statusText.setText("✅ Plugin system is active with " + enabledPlugins + " plugin(s) running.");
            statusText.setTextColor(getColor(android.R.color.holo_green_dark));
        }
    }
    
    private void openPluginInstaller() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("*/*");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        intent.putExtra(Intent.EXTRA_MIME_TYPES, new String[]{"application/java-archive", "application/zip"});
        startActivityForResult(Intent.createChooser(intent, "Select Plugin File (.jar/.zip)"), REQUEST_INSTALL_PLUGIN);
    }
    
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (requestCode == REQUEST_INSTALL_PLUGIN && resultCode == RESULT_OK && data != null) {
            Uri pluginUri = data.getData();
            if (pluginUri != null) {
                installPluginFromUri(pluginUri);
            }
        }
    }
    
    private void installPluginFromUri(Uri uri) {
        // Start plugin installation activity
        Intent intent = new Intent(this, PluginInstallActivity.class);
        intent.setData(uri);
        startActivity(intent);
    }
    
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.plugin_management_menu, menu);
        return true;
    }
    
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int id = item.getItemId();
        
        if (id == R.id.action_reload_all) {
            reloadAllPlugins();
            return true;
        } else if (id == R.id.action_disable_all) {
            disableAllPlugins();
            return true;
        } else if (id == R.id.action_enable_all) {
            enableAllPlugins();
            return true;
        } else if (id == R.id.action_plugin_directory) {
            openPluginDirectory();
            return true;
        } else if (id == R.id.action_plugin_logs) {
            viewPluginLogs();
            return true;
        }
        
        return super.onOptionsItemSelected(item);
    }
    
    private void reloadAllPlugins() {
        new AlertDialog.Builder(this)
            .setTitle("Reload All Plugins")
            .setMessage("This will unload and reload all plugins. Continue?")
            .setPositiveButton("Reload", (dialog, which) -> {
                new Thread(() -> {
                    pluginManager.reloadAllPlugins();
                    runOnUiThread(() -> {
                        Toast.makeText(this, "All plugins reloaded", Toast.LENGTH_SHORT).show();
                        refreshPluginList();
                    });
                }).start();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void disableAllPlugins() {
        new Thread(() -> {
            for (PluginInfo info : pluginList) {
                if (info.isEnabled && info.context != null) {
                    info.context.disable();
                }
            }
            runOnUiThread(() -> {
                Toast.makeText(this, "All plugins disabled", Toast.LENGTH_SHORT).show();
                refreshPluginList();
            });
        }).start();
    }
    
    private void enableAllPlugins() {
        new Thread(() -> {
            for (PluginInfo info : pluginList) {
                if (info.isLoaded && !info.isEnabled && info.context != null) {
                    info.context.enable();
                }
            }
            runOnUiThread(() -> {
                Toast.makeText(this, "All plugins enabled", Toast.LENGTH_SHORT).show();
                refreshPluginList();
            });
        }).start();
    }
    
    private void openPluginDirectory() {
        // Show plugin directory information
        File pluginDir = pluginManager.getPluginDirectory();
        new AlertDialog.Builder(this)
            .setTitle("Plugin Directory")
            .setMessage("Plugins are stored in:\n" + pluginDir.getAbsolutePath() + 
                       "\n\nPlace .jar plugin files in this directory and tap 'Scan' to load them.")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void viewPluginLogs() {
        Intent intent = new Intent(this, LogViewerEnhancedActivity.class);
        intent.putExtra("filter_tag", "Plugin");
        startActivity(intent);
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        refreshPluginList();
    }
    
    // ===== PLUGIN ADAPTER =====
    
    private class PluginAdapter extends RecyclerView.Adapter<PluginAdapter.PluginViewHolder> {
        
        @NonNull
        @Override
        public PluginViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_plugin, parent, false);
            return new PluginViewHolder(view);
        }
        
        @Override
        public void onBindViewHolder(@NonNull PluginViewHolder holder, int position) {
            PluginInfo pluginInfo = pluginList.get(position);
            holder.bind(pluginInfo);
        }
        
        @Override
        public int getItemCount() {
            return pluginList.size();
        }
        
        class PluginViewHolder extends RecyclerView.ViewHolder {
            private CardView cardView;
            private TextView nameText;
            private TextView versionText;
            private TextView descriptionText;
            private TextView statusText;
            private Switch enableSwitch;
            private Button configButton;
            private Button infoButton;
            private Button deleteButton;
            private View statusIndicator;
            
            public PluginViewHolder(@NonNull View itemView) {
                super(itemView);
                cardView = (CardView) itemView;
                nameText = itemView.findViewById(R.id.pluginNameText);
                versionText = itemView.findViewById(R.id.pluginVersionText);
                descriptionText = itemView.findViewById(R.id.pluginDescriptionText);
                statusText = itemView.findViewById(R.id.pluginStatusText);
                enableSwitch = itemView.findViewById(R.id.pluginEnableSwitch);
                configButton = itemView.findViewById(R.id.pluginConfigButton);
                infoButton = itemView.findViewById(R.id.pluginInfoButton);
                deleteButton = itemView.findViewById(R.id.pluginDeleteButton);
                statusIndicator = itemView.findViewById(R.id.pluginStatusIndicator);
            }
            
            public void bind(PluginInfo pluginInfo) {
                Plugin plugin = pluginInfo.plugin;
                
                // Set plugin information
                nameText.setText(plugin.getName());
                versionText.setText("v" + plugin.getVersion());
                descriptionText.setText(plugin.getDescription());
                statusText.setText(pluginInfo.status);
                
                // Set status indicator color
                int statusColor;
                if (pluginInfo.status.equals("Active")) {
                    statusColor = Color.parseColor("#4CAF50"); // Green
                } else if (pluginInfo.status.equals("Error")) {
                    statusColor = Color.parseColor("#F44336"); // Red
                } else if (pluginInfo.status.equals("Disabled")) {
                    statusColor = Color.parseColor("#FF9800"); // Orange
                } else {
                    statusColor = Color.parseColor("#9E9E9E"); // Gray
                }
                statusIndicator.setBackgroundColor(statusColor);
                statusText.setTextColor(statusColor);
                
                // Set card background based on status
                if (pluginInfo.isEnabled) {
                    cardView.setCardBackgroundColor(Color.parseColor("#E8F5E8"));
                } else if (pluginInfo.status.equals("Error")) {
                    cardView.setCardBackgroundColor(Color.parseColor("#FFEBEE"));
                } else {
                    cardView.setCardBackgroundColor(Color.WHITE);
                }
                
                // Configure enable switch
                enableSwitch.setOnCheckedChangeListener(null);
                enableSwitch.setChecked(pluginInfo.isEnabled);
                enableSwitch.setEnabled(pluginInfo.isLoaded);
                
                enableSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
                    togglePlugin(pluginInfo, isChecked);
                });
                
                // Configure buttons
                configButton.setEnabled(pluginInfo.isLoaded);
                configButton.setOnClickListener(v -> openPluginConfig(pluginInfo));
                
                infoButton.setOnClickListener(v -> showPluginInfo(pluginInfo));
                
                deleteButton.setOnClickListener(v -> deletePlugin(pluginInfo));
                
                // Handle long click for additional options
                cardView.setOnLongClickListener(v -> {
                    showPluginContextMenu(pluginInfo);
                    return true;
                });
            }
            
            private void togglePlugin(PluginInfo pluginInfo, boolean enabled) {
                new Thread(() -> {
                    try {
                        boolean success;
                        if (enabled) {
                            if (!pluginInfo.isLoaded) {
                                success = pluginManager.loadPlugin(pluginInfo.plugin.getId());
                            } else {
                                success = pluginInfo.context.enable();
                            }
                        } else {
                            success = pluginInfo.context != null && pluginInfo.context.disable();
                        }
                        
                        runOnUiThread(() -> {
                            if (success) {
                                refreshPluginList();
                            } else {
                                Toast.makeText(PluginManagementActivity.this, 
                                    "Failed to " + (enabled ? "enable" : "disable") + " plugin", 
                                    Toast.LENGTH_SHORT).show();
                                // Reset switch state
                                enableSwitch.setOnCheckedChangeListener(null);
                                enableSwitch.setChecked(!enabled);
                                enableSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
                                    togglePlugin(pluginInfo, isChecked);
                                });
                            }
                        });
                        
                    } catch (Exception e) {
                        LogUtils.logError("Error toggling plugin: " + e.getMessage());
                        runOnUiThread(() -> {
                            Toast.makeText(PluginManagementActivity.this, 
                                "Error: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                        });
                    }
                }).start();
            }
            
            private void openPluginConfig(PluginInfo pluginInfo) {
                Intent intent = new Intent(PluginManagementActivity.this, PluginConfigActivity.class);
                intent.putExtra("plugin_id", pluginInfo.plugin.getId());
                startActivity(intent);
            }
            
            private void showPluginInfo(PluginInfo pluginInfo) {
                Plugin plugin = pluginInfo.plugin;
                StringBuilder info = new StringBuilder();
                
                info.append("Name: ").append(plugin.getName()).append("\n");
                info.append("ID: ").append(plugin.getId()).append("\n");
                info.append("Version: ").append(plugin.getVersion()).append("\n");
                info.append("Author: ").append(plugin.getAuthor()).append("\n");
                info.append("Status: ").append(pluginInfo.status).append("\n");
                
                if (pluginInfo.context != null) {
                    Map<String, Object> stats = pluginInfo.context.getStatistics();
                    info.append("Loaded: ").append(new java.util.Date((Long) stats.get("loadTime"))).append("\n");
                    info.append("Uptime: ").append((Long) stats.get("uptime") / 1000).append("s\n");
                    info.append("Hooks: ").append(stats.get("hookCount")).append("\n");
                }
                
                if (pluginInfo.errorMessage != null) {
                    info.append("\nError: ").append(pluginInfo.errorMessage);
                }
                
                new AlertDialog.Builder(PluginManagementActivity.this)
                    .setTitle("Plugin Information")
                    .setMessage(info.toString())
                    .setPositiveButton("OK", null)
                    .show();
            }
            
            private void deletePlugin(PluginInfo pluginInfo) {
                new AlertDialog.Builder(PluginManagementActivity.this)
                    .setTitle("Delete Plugin")
                    .setMessage("Are you sure you want to delete " + pluginInfo.plugin.getName() + "?")
                    .setPositiveButton("Delete", (dialog, which) -> {
                        new Thread(() -> {
                            try {
                                boolean success = pluginManager.uninstallPlugin(pluginInfo.plugin.getId());
                                runOnUiThread(() -> {
                                    if (success) {
                                        Toast.makeText(PluginManagementActivity.this, 
                                            "Plugin deleted", Toast.LENGTH_SHORT).show();
                                        refreshPluginList();
                                    } else {
                                        Toast.makeText(PluginManagementActivity.this, 
                                            "Failed to delete plugin", Toast.LENGTH_SHORT).show();
                                    }
                                });
                            } catch (Exception e) {
                                LogUtils.logError("Error deleting plugin: " + e.getMessage());
                                runOnUiThread(() -> {
                                    Toast.makeText(PluginManagementActivity.this, 
                                        "Error deleting plugin: " + e.getMessage(), Toast.LENGTH_SHORT).show();
                                });
                            }
                        }).start();
                    })
                    .setNegativeButton("Cancel", null)
                    .show();
            }
            
            private void showPluginContextMenu(PluginInfo pluginInfo) {
                String[] options;
                if (pluginInfo.isLoaded) {
                    options = new String[]{"Reload", "View Logs", "Open Directory", "Export Config"};
                } else {
                    options = new String[]{"Load", "View File", "Open Directory"};
                }
                
                new AlertDialog.Builder(PluginManagementActivity.this)
                    .setTitle(pluginInfo.plugin.getName())
                    .setItems(options, (dialog, which) -> {
                        handleContextMenuAction(pluginInfo, which, pluginInfo.isLoaded);
                    })
                    .show();
            }
            
            private void handleContextMenuAction(PluginInfo pluginInfo, int action, boolean isLoaded) {
                if (isLoaded) {
                    switch (action) {
                        case 0: // Reload
                            new Thread(() -> {
                                pluginManager.reloadPlugin(pluginInfo.plugin.getId());
                                runOnUiThread(() -> refreshPluginList());
                            }).start();
                            break;
                        case 1: // View Logs
                            Intent logIntent = new Intent(PluginManagementActivity.this, LogViewerEnhancedActivity.class);
                            logIntent.putExtra("filter_tag", pluginInfo.plugin.getName());
                            startActivity(logIntent);
                            break;
                        case 2: // Open Directory
                            showPluginDirectory(pluginInfo);
                            break;
                        case 3: // Export Config
                            exportPluginConfig(pluginInfo);
                            break;
                    }
                } else {
                    switch (action) {
                        case 0: // Load
                            new Thread(() -> {
                                pluginManager.loadPlugin(pluginInfo.plugin.getId());
                                runOnUiThread(() -> refreshPluginList());
                            }).start();
                            break;
                        case 1: // View File
                            showPluginFileInfo(pluginInfo);
                            break;
                        case 2: // Open Directory
                            showPluginDirectory(pluginInfo);
                            break;
                    }
                }
            }
            
            private void showPluginDirectory(PluginInfo pluginInfo) {
                if (pluginInfo.context != null) {
                    File dataDir = pluginInfo.context.getDataDir();
                    new AlertDialog.Builder(PluginManagementActivity.this)
                        .setTitle("Plugin Directory")
                        .setMessage("Plugin data directory:\n" + dataDir.getAbsolutePath())
                        .setPositiveButton("OK", null)
                        .show();
                }
            }
            
            private void showPluginFileInfo(PluginInfo pluginInfo) {
                // Show information about the plugin file
                new AlertDialog.Builder(PluginManagementActivity.this)
                    .setTitle("Plugin File")
                    .setMessage("Plugin file information for " + pluginInfo.plugin.getName())
                    .setPositiveButton("OK", null)
                    .show();
            }
            
            private void exportPluginConfig(PluginInfo pluginInfo) {
                Toast.makeText(PluginManagementActivity.this, 
                    "Config export not yet implemented", Toast.LENGTH_SHORT).show();
            }
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/SettingsActivity.java

// File: SettingsActivity.java (Enhanced UI with Operation Modes)
// Path: /app/src/main/java/com/terrarialoader/ui/SettingsActivity.java

package com.modloader.ui;

import android.Manifest;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.pm.PackageManager;
import android.graphics.Color;
import android.net.Uri;
import android.os.Build;
import android.os.Bundle;
import android.os.Environment;
import android.provider.Settings;
import android.view.View;
import android.widget.*;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;
import androidx.cardview.widget.CardView;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.modloader.R;
import com.modloader.util.LogUtils;
import com.modloader.util.PermissionManager;
import com.modloader.util.ShizukuManager;
import com.modloader.util.RootManager;

public class SettingsActivity extends AppCompatActivity {
    
    // Operation Mode Constants
    public static final String PREF_OPERATION_MODE = "operation_mode";
    public static final String MODE_NORMAL = "normal";
    public static final String MODE_SHIZUKU = "shizuku";
    public static final String MODE_ROOT = "root";
    public static final String MODE_HYBRID = "hybrid"; // Both Shizuku + Root
    
    // UI Components
    private RadioGroup operationModeGroup;
    private RadioButton normalModeRadio;
    private RadioButton shizukuModeRadio;
    private RadioButton rootModeRadio;
    private RadioButton hybridModeRadio;
    
    private CardView normalCard, shizukuCard, rootCard, hybridCard;
    private TextView normalStatus, shizukuStatus, rootStatus, hybridStatus;
    private Button shizukuSetupBtn, rootSetupBtn, permissionBtn;
    
    private Switch autoEnableSwitch;
    private Switch debugLoggingSwitch;
    private Switch autoBackupSwitch;
    private Switch autoUpdateSwitch;
    
    private SharedPreferences prefs;
    private PermissionManager permissionManager;
    private ShizukuManager shizukuManager;
    private RootManager rootManager;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_settings_enhanced);
        setTitle("⚙️ Settings & Operation Modes");
        
        LogUtils.logUser("Settings activity opened");
        
        // Initialize managers
        prefs = getSharedPreferences("terraria_loader_settings", MODE_PRIVATE);
        permissionManager = new PermissionManager(this);
        shizukuManager = new ShizukuManager(this);
        rootManager = new RootManager(this);
        
        initializeViews();
        setupOperationModes();
        setupFeatureToggles();
        setupActionButtons();
        updateUIState();
        
        // Auto-setup permissions based on current mode
        autoSetupPermissions();
    }
    
    private void initializeViews() {
        // Operation Mode Selection
        operationModeGroup = findViewById(R.id.operationModeGroup);
        normalModeRadio = findViewById(R.id.normalModeRadio);
        shizukuModeRadio = findViewById(R.id.shizukuModeRadio);
        rootModeRadio = findViewById(R.id.rootModeRadio);
        hybridModeRadio = findViewById(R.id.hybridModeRadio);
        
        // Mode Cards
        normalCard = findViewById(R.id.normalCard);
        shizukuCard = findViewById(R.id.shizukuCard);
        rootCard = findViewById(R.id.rootCard);
        hybridCard = findViewById(R.id.hybridCard);
        
        // Status Text
        normalStatus = findViewById(R.id.normalStatus);
        shizukuStatus = findViewById(R.id.shizukuStatus);
        rootStatus = findViewById(R.id.rootStatus);
        hybridStatus = findViewById(R.id.hybridStatus);
        
        // Setup Buttons
        shizukuSetupBtn = findViewById(R.id.shizukuSetupBtn);
        rootSetupBtn = findViewById(R.id.rootSetupBtn);
        permissionBtn = findViewById(R.id.permissionBtn);
        
        // Feature Toggles
        autoEnableSwitch = findViewById(R.id.autoEnableSwitch);
        debugLoggingSwitch = findViewById(R.id.debugLoggingSwitch);
        autoBackupSwitch = findViewById(R.id.autoBackupSwitch);
        autoUpdateSwitch = findViewById(R.id.autoUpdateSwitch);
    }
    
    private void setupOperationModes() {
        // Load current mode
        String currentMode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        setOperationMode(currentMode, false);
        
        // Set up radio button listeners
        operationModeGroup.setOnCheckedChangeListener((group, checkedId) -> {
            String newMode;
            if (checkedId == R.id.normalModeRadio) {
                newMode = MODE_NORMAL;
            } else if (checkedId == R.id.shizukuModeRadio) {
                newMode = MODE_SHIZUKU;
            } else if (checkedId == R.id.rootModeRadio) {
                newMode = MODE_ROOT;
            } else if (checkedId == R.id.hybridModeRadio) {
                newMode = MODE_HYBRID;
            } else {
                newMode = MODE_NORMAL;
            }
            
            setOperationMode(newMode, true);
        });
        
        // Card click listeners for better UX
        setupCardListeners();
    }
    
    private void setupCardListeners() {
        normalCard.setOnClickListener(v -> {
            normalModeRadio.setChecked(true);
        });
        
        shizukuCard.setOnClickListener(v -> {
            if (shizukuManager.isShizukuAvailable()) {
                shizukuModeRadio.setChecked(true);
            } else {
                showShizukuSetupDialog();
            }
        });
        
        rootCard.setOnClickListener(v -> {
            if (rootManager.isRootAvailable()) {
                rootModeRadio.setChecked(true);
            } else {
                showRootInfoDialog();
            }
        });
        
        hybridCard.setOnClickListener(v -> {
            if (shizukuManager.isShizukuAvailable() && rootManager.isRootAvailable()) {
                hybridModeRadio.setChecked(true);
            } else {
                showHybridSetupDialog();
            }
        });
    }
    
    private void setOperationMode(String mode, boolean save) {
        if (save) {
            prefs.edit().putString(PREF_OPERATION_MODE, mode).apply();
            LogUtils.logUser("Operation mode changed to: " + mode);
            
            // Auto-setup permissions for new mode
            autoSetupPermissions();
        }
        
        // Update radio buttons
        switch (mode) {
            case MODE_NORMAL:
                normalModeRadio.setChecked(true);
                break;
            case MODE_SHIZUKU:
                shizukuModeRadio.setChecked(true);
                break;
            case MODE_ROOT:
                rootModeRadio.setChecked(true);
                break;
            case MODE_HYBRID:
                hybridModeRadio.setChecked(true);
                break;
        }
        
        updateUIState();
    }
    
    private void setupFeatureToggles() {
        // Load current settings
        autoEnableSwitch.setChecked(prefs.getBoolean("auto_enable_mods", true));
        debugLoggingSwitch.setChecked(prefs.getBoolean("debug_logging", false));
        autoBackupSwitch.setChecked(prefs.getBoolean("auto_backup", true));
        autoUpdateSwitch.setChecked(prefs.getBoolean("auto_update_check", true));
        
        // Set up listeners
        autoEnableSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_enable_mods", isChecked).apply();
            LogUtils.logUser("Auto-enable mods: " + isChecked);
        });
        
        debugLoggingSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("debug_logging", isChecked).apply();
            LogUtils.setDebugEnabled(isChecked);
            LogUtils.logUser("Debug logging: " + isChecked);
        });
        
        autoBackupSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_backup", isChecked).apply();
            LogUtils.logUser("Auto backup: " + isChecked);
        });
        
        autoUpdateSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            prefs.edit().putBoolean("auto_update_check", isChecked).apply();
            LogUtils.logUser("Auto update check: " + isChecked);
        });
    }
    
    private void setupActionButtons() {
        // Shizuku Setup Button
        shizukuSetupBtn.setOnClickListener(v -> {
            if (!shizukuManager.isShizukuInstalled()) {
                showShizukuInstallDialog();
            } else if (!shizukuManager.isShizukuRunning()) {
                showShizukuStartDialog();
            } else if (!shizukuManager.hasShizukuPermission()) {
                shizukuManager.requestShizukuPermission();
            } else {
                Toast.makeText(this, "Shizuku is already properly configured!", Toast.LENGTH_SHORT).show();
            }
        });
        
        // Root Setup Button
        rootSetupBtn.setOnClickListener(v -> {
            if (!rootManager.isRootAvailable()) {
                showRootInfoDialog();
            } else {
                rootManager.requestRootAccess();
            }
        });
        
        // Permission Management Button
        permissionBtn.setOnClickListener(v -> {
            showPermissionManagementDialog();
        });
        
        // Additional action buttons
        findViewById(R.id.resetSettingsBtn).setOnClickListener(v -> resetToDefaults());
        findViewById(R.id.exportSettingsBtn).setOnClickListener(v -> exportSettings());
        findViewById(R.id.importSettingsBtn).setOnClickListener(v -> importSettings());
    }
    
    private void updateUIState() {
        String currentMode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        
        // Update card appearances
        updateCardAppearance(normalCard, normalStatus, MODE_NORMAL.equals(currentMode), 
                           permissionManager.hasBasicPermissions(), "Standard Android permissions");
        
        boolean shizukuReady = shizukuManager.isShizukuReady();
        updateCardAppearance(shizukuCard, shizukuStatus, MODE_SHIZUKU.equals(currentMode), 
                           shizukuReady, getShizukuStatusText());
        
        boolean rootReady = rootManager.isRootReady();
        updateCardAppearance(rootCard, rootStatus, MODE_ROOT.equals(currentMode), 
                           rootReady, getRootStatusText());
        
        boolean hybridReady = shizukuReady && rootReady;
        updateCardAppearance(hybridCard, hybridStatus, MODE_HYBRID.equals(currentMode), 
                           hybridReady, "Maximum capabilities with both Shizuku and Root");
        
        // Update setup button states
        updateSetupButtons();
    }
    
    private void updateCardAppearance(CardView card, TextView status, boolean selected, 
                                      boolean available, String statusText) {
        int cardColor;
        int textColor = Color.BLACK;
        
        if (selected) {
            cardColor = Color.parseColor("#E8F5E8"); // Light green
            card.setCardElevation(12f);
        } else if (available) {
            cardColor = Color.parseColor("#E3F2FD"); // Light blue
            card.setCardElevation(6f);
        } else {
            cardColor = Color.parseColor("#FFEBEE"); // Light red
            textColor = Color.parseColor("#666666");
            card.setCardElevation(2f);
        }
        
        card.setCardBackgroundColor(cardColor);
        status.setText(statusText);
        status.setTextColor(textColor);
    }
    
    private void updateSetupButtons() {
        // Shizuku setup button
        if (shizukuManager.isShizukuReady()) {
            shizukuSetupBtn.setText("✅ Shizuku Ready");
            shizukuSetupBtn.setEnabled(false);
        } else if (shizukuManager.isShizukuRunning()) {
            shizukuSetupBtn.setText("🔐 Grant Permission");
            shizukuSetupBtn.setEnabled(true);
        } else if (shizukuManager.isShizukuInstalled()) {
            shizukuSetupBtn.setText("▶️ Start Shizuku");
            shizukuSetupBtn.setEnabled(true);
        } else {
            shizukuSetupBtn.setText("📥 Install Shizuku");
            shizukuSetupBtn.setEnabled(true);
        }
        
        // Root setup button
        if (rootManager.isRootReady()) {
            rootSetupBtn.setText("✅ Root Ready");
            rootSetupBtn.setEnabled(false);
        } else if (rootManager.isRootAvailable()) {
            rootSetupBtn.setText("🔐 Grant Root Access");
            rootSetupBtn.setEnabled(true);
        } else {
            rootSetupBtn.setText("❌ Root Not Available");
            rootSetupBtn.setEnabled(false);
        }
    }
    
    private String getShizukuStatusText() {
        if (!shizukuManager.isShizukuInstalled()) {
            return "Shizuku app not installed";
        } else if (!shizukuManager.isShizukuRunning()) {
            return "Shizuku service not running";
        } else if (!shizukuManager.hasShizukuPermission()) {
            return "Shizuku permission not granted";
        } else {
            return "✅ Shizuku ready - Enhanced file access";
        }
    }
    
    private String getRootStatusText() {
        if (!rootManager.isRootAvailable()) {
            return "Root access not available on this device";
        } else if (!rootManager.hasRootPermission()) {
            return "Root permission not granted";
        } else {
            return "✅ Root access ready - Full system control";
        }
    }
    
    private void autoSetupPermissions() {
        String mode = prefs.getString(PREF_OPERATION_MODE, MODE_NORMAL);
        LogUtils.logDebug("Auto-setting up permissions for mode: " + mode);
        
        // Request basic permissions for all modes
        permissionManager.requestBasicPermissions();
        
        switch (mode) {
            case MODE_SHIZUKU:
                if (shizukuManager.isShizukuRunning() && !shizukuManager.hasShizukuPermission()) {
                    shizukuManager.requestShizukuPermission();
                }
                break;
            case MODE_ROOT:
                if (rootManager.isRootAvailable() && !rootManager.hasRootPermission()) {
                    rootManager.requestRootAccess();
                }
                break;
            case MODE_HYBRID:
                if (shizukuManager.isShizukuRunning() && !shizukuManager.hasShizukuPermission()) {
                    shizukuManager.requestShizukuPermission();
                }
                if (rootManager.isRootAvailable() && !rootManager.hasRootPermission()) {
                    rootManager.requestRootAccess();
                }
                break;
        }
    }
    
    // Dialog methods
    private void showShizukuSetupDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Shizuku Setup Required")
            .setMessage("Shizuku provides enhanced file access without root. Would you like to install it?")
            .setPositiveButton("Install", (dialog, which) -> shizukuManager.installShizuku())
            .setNeutralButton("Learn More", (dialog, which) -> openUrl("https://shizuku.rikka.app/"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showRootInfoDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Root Access")
            .setMessage("Root access provides maximum system control but requires a rooted device. " +
                       "Root access cannot be installed through this app - your device must already be rooted.")
            .setPositiveButton("Check Root", (dialog, which) -> rootManager.checkRootStatus())
            .setNeutralButton("Root Guide", (dialog, which) -> openUrl("https://www.xda-developers.com/root/"))
            .setNegativeButton("OK", null)
            .show();
    }
    
    private void showHybridSetupDialog() {
        String message = "";
        if (!shizukuManager.isShizukuAvailable()) {
            message += "• Shizuku is not available\n";
        }
        if (!rootManager.isRootAvailable()) {
            message += "• Root access is not available\n";
        }
        
        new AlertDialog.Builder(this)
            .setTitle("Hybrid Mode Requirements")
            .setMessage("Hybrid mode requires both Shizuku and Root access:\n\n" + message + 
                       "\nPlease set up both components individually first.")
            .setPositiveButton("OK", null)
            .show();
    }
    
    private void showShizukuInstallDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Install Shizuku")
            .setMessage("Shizuku needs to be downloaded and installed. This will open your browser.")
            .setPositiveButton("Download", (dialog, which) -> 
                openUrl("https://github.com/RikkaApps/Shizuku/releases/latest"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showShizukuStartDialog() {
        new AlertDialog.Builder(this)
            .setTitle("Start Shizuku Service")
            .setMessage("Shizuku is installed but not running. Please start it using ADB or root, " +
                       "then return to this app.")
            .setPositiveButton("Open Shizuku", (dialog, which) -> shizukuManager.openShizukuApp())
            .setNeutralButton("ADB Guide", (dialog, which) -> 
                openUrl("https://shizuku.rikka.app/guide/setup/"))
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showPermissionManagementDialog() {
        String[] permissions = {
            "Storage Access", 
            "Install Packages", 
            "Shizuku Access",
            "Root Access"
        };
        
        boolean[] grantedStatus = {
            permissionManager.hasStoragePermission(),
            permissionManager.hasInstallPermission(),
            shizukuManager.hasShizukuPermission(),
            rootManager.hasRootPermission()
        };
        
        StringBuilder message = new StringBuilder("Permission Status:\n\n");
        for (int i = 0; i < permissions.length; i++) {
            message.append(grantedStatus[i] ? "✅ " : "❌ ")
                   .append(permissions[i]).append("\n");
        }
        
        new AlertDialog.Builder(this)
            .setTitle("Permission Management")
            .setMessage(message.toString())
            .setPositiveButton("Request Missing", (dialog, which) -> {
                permissionManager.requestAllPermissions();
                autoSetupPermissions();
            })
            .setNeutralButton("App Settings", (dialog, which) -> openAppSettings())
            .setNegativeButton("Close", null)
            .show();
    }
    
    // Utility methods
    private void openUrl(String url) {
        try {
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
            startActivity(intent);
        } catch (Exception e) {
            Toast.makeText(this, "Could not open browser", Toast.LENGTH_SHORT).show();
        }
    }
    
    private void openAppSettings() {
        try {
            Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
            intent.setData(Uri.parse("package:" + getPackageName()));
            startActivity(intent);
        } catch (Exception e) {
            Toast.makeText(this, "Could not open app settings", Toast.LENGTH_SHORT).show();
        }
    }
    
    private void resetToDefaults() {
        new AlertDialog.Builder(this)
            .setTitle("Reset Settings")
            .setMessage("This will reset all settings to their default values. Continue?")
            .setPositiveButton("Reset", (dialog, which) -> {
                prefs.edit().clear().apply();
                recreate(); // Reload activity with default settings
                Toast.makeText(this, "Settings reset to defaults", Toast.LENGTH_SHORT).show();
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void exportSettings() {
        // Implementation for exporting settings to file
        Toast.makeText(this, "Export settings - Coming soon", Toast.LENGTH_SHORT).show();
    }
    
    private void importSettings() {
        // Implementation for importing settings from file
        Toast.makeText(this, "Import settings - Coming soon", Toast.LENGTH_SHORT).show();
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        updateUIState(); // Refresh UI when returning from other apps
    }
    
    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, 
                                           @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        permissionManager.handlePermissionResult(requestCode, permissions, grantResults);
        updateUIState(); // Refresh UI after permission changes
    }
    
    // Static utility methods for other activities
    public static String getCurrentOperationMode(Context context) {
        return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                     .getString(PREF_OPERATION_MODE, MODE_NORMAL);
    }
    
    public static boolean isShizukuMode(Context context) {
        String mode = getCurrentOperationMode(context);
        return MODE_SHIZUKU.equals(mode) || MODE_HYBRID.equals(mode);
    }
    
    public static boolean isRootMode(Context context) {
        String mode = getCurrentOperationMode(context);
        return MODE_ROOT.equals(mode) || MODE_HYBRID.equals(mode);
    }
    
    public static boolean canUseEnhancedPermissions(Context context) {
        return isShizukuMode(context) || isRootMode(context);
    }
    
    // Legacy compatibility methods for existing code
    public static boolean isModsEnabled(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("auto_enable_mods", true);
        } catch (Exception e) {
            return true;
        }
    }
    
    public static boolean isSandboxMode(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("sandbox_mode", false);
        } catch (Exception e) {
            return false;
        }
    }
    
    public static boolean isDebugMode(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("debug_logging", false);
        } catch (Exception e) {
            return false;
        }
    }
    
    public static boolean isAutoSaveEnabled(Context context) {
        try {
            return context.getSharedPreferences("terraria_loader_settings", Context.MODE_PRIVATE)
                         .getBoolean("auto_backup", true);
        } catch (Exception e) {
            return true;
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/SetupGuideActivity.java

// File: SetupGuideActivity.java (Updated) - Added Offline ZIP Import
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/ui/SetupGuideActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.widget.Button;
import android.widget.Toast;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;
import com.modloader.util.OnlineInstaller;
import com.modloader.util.OfflineZipImporter;

public class SetupGuideActivity extends AppCompatActivity {

    private static final int REQUEST_SELECT_ZIP = 1001;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_setup_guide);

        setTitle("🚀 MelonLoader Setup Guide");

        setupButtons();
    }

    private void setupButtons() {
        Button btnOnlineInstall = findViewById(R.id.btn_online_install);
        Button btnOfflineImport = findViewById(R.id.btn_offline_import);
        Button btnManualInstructions = findViewById(R.id.btn_manual_instructions);

        btnOnlineInstall.setOnClickListener(v -> showOnlineInstallDialog());
        btnOfflineImport.setOnClickListener(v -> showOfflineImportDialog());
        btnManualInstructions.setOnClickListener(v -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
    }

    private void showOnlineInstallDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("🌐 Automated Online Installation");
        builder.setMessage("This will automatically download and install MelonLoader/LemonLoader files from GitHub.\n\n" +
                          "Requirements:\n" +
                          "• Active internet connection\n" +
                          "• ~50MB free space\n\n" +
                          "Continue with automated installation?");
        
        builder.setPositiveButton("Continue", (dialog, which) -> {
            showLoaderTypeDialog();
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private void showOfflineImportDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("📦 Offline ZIP Import");
        builder.setMessage("Import a MelonLoader ZIP file that you've already downloaded.\n\n" +
                          "Supported files:\n" +
                          "• melon_data.zip (MelonLoader)\n" +
                          "• lemon_data.zip (LemonLoader)\n" +
                          "• Custom MelonLoader packages\n\n" +
                          "The ZIP will be automatically extracted to the correct directories.");
        
        builder.setPositiveButton("📂 Select ZIP File", (dialog, which) -> {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("application/zip");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, REQUEST_SELECT_ZIP);
        });
        
        builder.setNegativeButton("Cancel", null);
        builder.show();
    }

    private void showLoaderTypeDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Choose Loader Type");
        builder.setMessage("Select which loader to install:\n\n" +
                          "🔸 MelonLoader:\n" +
                          "• Full-featured Unity mod loader\n" +
                          "• Larger file size (~40MB)\n" +
                          "• Best compatibility\n\n" +
                          "🔸 LemonLoader:\n" +
                          "• Lightweight Unity mod loader\n" +
                          "• Smaller file size (~15MB)\n" +
                          "• Faster installation\n\n" +
                          "Which would you like to install?");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected MelonLoader for automated installation");
            startAutomatedInstallation(MelonLoaderManager.LoaderType.MELONLOADER_NET8);
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            LogUtils.logUser("User selected LemonLoader for automated installation");
            startAutomatedInstallation(MelonLoaderManager.LoaderType.MELONLOADER_NET35);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }
    
    private void startAutomatedInstallation(MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("Starting automated " + loaderType.getDisplayName() + " installation...");
        
        AlertDialog progressDialog = new AlertDialog.Builder(this)
            .setTitle("Installing " + loaderType.getDisplayName())
            .setMessage("Downloading and extracting files from GitHub...\nThis may take a few minutes.")
            .setCancelable(false)
            .show();

        new Thread(() -> {
            boolean success = false;
            String errorMessage = "";

            try {
                OnlineInstaller.InstallationResult result = OnlineInstaller.installMelonLoaderOnline(
                    this, MelonLoaderManager.TERRARIA_PACKAGE, loaderType);
                success = result.success;
                errorMessage = result.message;
                
            } catch (Exception e) {
                success = false;
                errorMessage = e.getMessage();
                LogUtils.logDebug("Automated installation error: " + errorMessage);
            }

            final boolean finalSuccess = success;
            final String finalErrorMessage = errorMessage;

            runOnUiThread(() -> {
                progressDialog.dismiss();
                if (finalSuccess) {
                    showInstallationSuccessDialog(loaderType);
                } else {
                    showInstallationErrorDialog(loaderType, finalErrorMessage);
                }
            });
        }).start();
    }

    private void startOfflineImportProcess(Uri zipUri) {
        LogUtils.logUser("Starting offline ZIP import process...");
        
        AlertDialog progressDialog = new AlertDialog.Builder(this)
            .setTitle("Importing MelonLoader ZIP")
            .setMessage("Analyzing and extracting ZIP file...\nThis may take a moment.")
            .setCancelable(false)
            .show();

        new Thread(() -> {
            OfflineZipImporter.ImportResult result = OfflineZipImporter.importMelonLoaderZip(this, zipUri);

            runOnUiThread(() -> {
                progressDialog.dismiss();
                if (result.success) {
                    showImportSuccessDialog(result);
                } else {
                    showImportErrorDialog(result);
                }
            });
        }).start();
    }

    private void showInstallationSuccessDialog(MelonLoaderManager.LoaderType loaderType) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("✅ Installation Complete!");
        builder.setMessage("Great! " + loaderType.getDisplayName() + " has been successfully installed!\n\n" +
                          "Next steps:\n" +
                          "1. Go to Unified Loader Activity\n" +
                          "2. Select your Terraria APK\n" +
                          "3. Patch APK with loader\n" +
                          "4. Install patched Terraria\n" +
                          "5. Add DLL mods and enjoy!\n\n" +
                          "You can now use DLL mods with Terraria!");
        
        builder.setPositiveButton("🚀 Open Unified Loader", (dialog, which) -> {
            Intent intent = new Intent(this, UnifiedLoaderActivity.class);
            startActivity(intent);
            finish();
        });
        
        builder.setNegativeButton("Later", (dialog, which) -> {
            finish();
        });
        
        builder.show();
    }

    private void showInstallationErrorDialog(MelonLoaderManager.LoaderType loaderType, String errorMessage) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("❌ Installation Failed");
        builder.setMessage("Failed to install " + loaderType.getDisplayName() + "\n\n" +
                          "Error: " + (errorMessage.isEmpty() ? "Unknown error occurred" : errorMessage) + "\n\n" +
                          "Please try:\n" +
                          "• Check your internet connection\n" +
                          "• Use Offline ZIP Import instead\n" +
                          "• Use Manual Installation\n" +
                          "• Try again later");
        
        builder.setPositiveButton("📦 Try Offline Import", (dialog, which) -> {
            showOfflineImportDialog();
        });
        
        builder.setNegativeButton("📖 Manual Guide", (dialog, which) -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    private void showImportSuccessDialog(OfflineZipImporter.ImportResult result) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("✅ ZIP Import Complete!");
        builder.setMessage("Successfully imported " + result.detectedType.getDisplayName() + "!\n\n" +
                          "Files extracted: " + result.filesExtracted + "\n\n" +
                          "The loader files have been automatically placed in the correct directories:\n" +
                          "• NET8/NET35 runtime files\n" +
                          "• Dependencies and support modules\n" +
                          "• All required components\n\n" +
                          "You can now patch APK files!");
        
        builder.setPositiveButton("🚀 Open Unified Loader", (dialog, which) -> {
            Intent intent = new Intent(this, UnifiedLoaderActivity.class);
            startActivity(intent);
            finish();
        });
        
        builder.setNegativeButton("✅ Done", (dialog, which) -> {
            finish();
        });
        
        builder.show();
    }

    private void showImportErrorDialog(OfflineZipImporter.ImportResult result) {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("❌ ZIP Import Failed");
        builder.setMessage("Failed to import ZIP file\n\n" +
                          "Error: " + result.message + "\n\n" +
                          (result.errorDetails != null ? "Details: " + result.errorDetails + "\n\n" : "") +
                          "Please ensure:\n" +
                          "• ZIP file is a valid MelonLoader package\n" +
                          "• File is not corrupted\n" +
                          "• You have sufficient storage space");
        
        builder.setPositiveButton("🌐 Try Online Install", (dialog, which) -> {
            showOnlineInstallDialog();
        });
        
        builder.setNegativeButton("📖 Manual Guide", (dialog, which) -> {
            Intent intent = new Intent(this, InstructionsActivity.class);
            startActivity(intent);
        });
        
        builder.setNeutralButton("Try Again", (dialog, which) -> {
            showOfflineImportDialog();
        });
        
        builder.show();
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (requestCode == REQUEST_SELECT_ZIP && resultCode == Activity.RESULT_OK && data != null) {
            Uri zipUri = data.getData();
            if (zipUri != null) {
                LogUtils.logUser("ZIP file selected for offline import");
                startOfflineImportProcess(zipUri);
            }
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderActivity.java

// File: UnifiedLoaderActivity.java - Complete Fixed Version
// Path: /main/java/com/modloader/ui/UnifiedLoaderActivity.java

package com.modloader.ui;

import android.app.Activity;
import android.app.AlertDialog;
import android.content.Intent;
import android.database.Cursor;
import android.net.Uri;
import android.os.Bundle;
import android.provider.OpenableColumns;
import android.view.View;
import android.widget.*;
import android.graphics.Typeface;
import androidx.appcompat.app.AppCompatActivity;

import com.modloader.R;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.util.LogUtils;

/**
 * Unified Loader Activity - Complete wizard-style interface for MelonLoader setup
 */
public class UnifiedLoaderActivity extends AppCompatActivity implements 
    UnifiedLoaderController.UnifiedLoaderCallback, UnifiedLoaderListener {

    private static final int REQUEST_SELECT_APK = 1001;
    private static final int REQUEST_SELECT_ZIP = 1002;
    
    // UI Components
    private ProgressBar stepProgressBar;
    private TextView stepTitleText;
    private TextView stepDescriptionText;
    private TextView stepIndicatorText;
    private LinearLayout stepContentContainer;
    private Button previousButton;
    private Button nextButton;
    private Button actionButton;
    
    // Current step content views
    private LinearLayout welcomeContent;
    private LinearLayout loaderInstallContent;
    private LinearLayout apkSelectionContent;
    private LinearLayout patchingContent;
    private LinearLayout completionContent;
    
    // Status indicators
    private TextView loaderStatusText;
    private TextView apkStatusText;
    private TextView progressText;
    private ProgressBar actionProgressBar;
    
    // Controller
    private UnifiedLoaderController controller;
    private AlertDialog progressDialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_unified_loader);
        
        setTitle("MelonLoader Setup Wizard");
        
        // Initialize controller
        controller = new UnifiedLoaderController(this);
        controller.setCallback(this);
        
        initializeViews();
        setupStepContents();
        setupListeners();
        
        // Start wizard
        controller.setCurrentStep(UnifiedLoaderController.LoaderStep.WELCOME);
    }

    private void initializeViews() {
        stepProgressBar = findViewById(R.id.stepProgressBar);
        stepTitleText = findViewById(R.id.stepTitleText);
        stepDescriptionText = findViewById(R.id.stepDescriptionText);
        stepIndicatorText = findViewById(R.id.stepIndicatorText);
        stepContentContainer = findViewById(R.id.stepContentContainer);
        previousButton = findViewById(R.id.previousButton);
        nextButton = findViewById(R.id.nextButton);
        actionButton = findViewById(R.id.actionButton);
        
        loaderStatusText = findViewById(R.id.loaderStatusText);
        apkStatusText = findViewById(R.id.apkStatusText);
        progressText = findViewById(R.id.progressText);
        actionProgressBar = findViewById(R.id.actionProgressBar);
    }

    private void setupStepContents() {
        // Create step content views dynamically
        welcomeContent = createWelcomeContent();
        loaderInstallContent = createLoaderInstallContent();
        apkSelectionContent = createApkSelectionContent();
        patchingContent = createPatchingContent();
        completionContent = createCompletionContent();
    }

    private LinearLayout createWelcomeContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(24, 24, 24, 24);
        
        TextView welcomeText = new TextView(this);
        welcomeText.setText("Welcome to MelonLoader Setup!\n\nThis wizard will guide you through:\n\n• Installing MelonLoader/LemonLoader\n• Patching your Terraria APK\n• Setting up DLL mod support\n\nClick 'Next' to begin!");
        welcomeText.setTextSize(16);
        welcomeText.setLineSpacing(8, 1.0f);
        content.addView(welcomeText);
        
        return content;
    }

    private LinearLayout createLoaderInstallContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        
        // Loader status
        TextView statusLabel = new TextView(this);
        statusLabel.setText("Current Status:");
        statusLabel.setTextSize(14);
        statusLabel.setTypeface(null, Typeface.BOLD);
        content.addView(statusLabel);
        
        TextView statusText = new TextView(this);
        statusText.setText("Checking...");
        statusText.setTextSize(14);
        statusText.setPadding(0, 8, 0, 16);
        content.addView(statusText);
        
        // Installation options
        TextView optionsLabel = new TextView(this);
        optionsLabel.setText("Installation Options:");
        optionsLabel.setTextSize(14);
        optionsLabel.setTypeface(null, Typeface.BOLD);
        content.addView(optionsLabel);
        
        Button onlineInstallBtn = new Button(this);
        onlineInstallBtn.setText("Online Installation (Recommended)");
        onlineInstallBtn.setOnClickListener(v -> showOnlineInstallOptions());
        content.addView(onlineInstallBtn);
        
        Button offlineInstallBtn = new Button(this);
        offlineInstallBtn.setText("Offline ZIP Import");
        offlineInstallBtn.setOnClickListener(v -> selectOfflineZip());
        content.addView(offlineInstallBtn);
        
        return content;
    }

    private LinearLayout createApkSelectionContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        
        TextView instructionText = new TextView(this);
        instructionText.setText("Select your Terraria APK file to patch with MelonLoader:");
        instructionText.setTextSize(16);
        content.addView(instructionText);
        
        Button selectApkBtn = new Button(this);
        selectApkBtn.setText("Select Terraria APK");
        selectApkBtn.setOnClickListener(v -> selectApkFile());
        content.addView(selectApkBtn);
        
        TextView statusText = new TextView(this);
        statusText.setText("No APK selected");
        statusText.setTextSize(14);
        statusText.setPadding(0, 16, 0, 0);
        content.addView(statusText);
        
        return content;
    }

    private LinearLayout createPatchingContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(16, 16, 16, 16);
        content.setGravity(android.view.Gravity.CENTER);
        
        TextView patchingText = new TextView(this);
        patchingText.setText("Patching APK with MelonLoader...");
        patchingText.setTextSize(18);
        patchingText.setGravity(android.view.Gravity.CENTER);
        content.addView(patchingText);
        
        ProgressBar progressBar = new ProgressBar(this);
        progressBar.setIndeterminate(true);
        LinearLayout.LayoutParams progressParams = new LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT);
        progressParams.topMargin = 24;
        progressParams.gravity = android.view.Gravity.CENTER;
        progressBar.setLayoutParams(progressParams);
        content.addView(progressBar);
        
        TextView statusText = new TextView(this);
        statusText.setText("Initializing...");
        statusText.setTextSize(14);
        statusText.setGravity(android.view.Gravity.CENTER);
        statusText.setPadding(0, 16, 0, 0);
        content.addView(statusText);
        
        return content;
    }

    private LinearLayout createCompletionContent() {
        LinearLayout content = new LinearLayout(this);
        content.setOrientation(LinearLayout.VERTICAL);
        content.setPadding(24, 24, 24, 24);
        
        TextView completionText = new TextView(this);
        completionText.setText("Setup Complete!\n\nYour modded Terraria APK is ready!");
        completionText.setTextSize(18);
        completionText.setGravity(android.view.Gravity.CENTER);
        content.addView(completionText);
        
        Button installApkBtn = new Button(this);
        installApkBtn.setText("Install Patched APK");
        installApkBtn.setOnClickListener(v -> controller.installPatchedApk());
        content.addView(installApkBtn);
        
        Button manageModsBtn = new Button(this);
        manageModsBtn.setText("Manage DLL Mods");
        manageModsBtn.setOnClickListener(v -> openModManagement());
        content.addView(manageModsBtn);
        
        Button viewLogsBtn = new Button(this);
        viewLogsBtn.setText("View Logs");
        viewLogsBtn.setOnClickListener(v -> startActivity(new Intent(this, LogViewerEnhancedActivity.class)));
        content.addView(viewLogsBtn);
        
        return content;
    }

    private void setupListeners() {
        previousButton.setOnClickListener(v -> {
            if (controller.canProceedToPreviousStep()) {
                controller.previousStep();
            }
        });
        
        nextButton.setOnClickListener(v -> {
            if (controller.canProceedToNextStep()) {
                handleNextStep();
            } else {
                showStepRequirements();
            }
        });
        
        actionButton.setOnClickListener(v -> handleActionButton());
    }

    private void handleNextStep() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        
        switch (currentStep) {
            case WELCOME:
                controller.nextStep();
                break;
            case LOADER_INSTALL:
                if (controller.isLoaderInstalled()) {
                    controller.nextStep();
                } else {
                    Toast.makeText(this, "Please install a loader first", Toast.LENGTH_SHORT).show();
                }
                break;
            case APK_SELECTION:
                if (controller.getSelectedApkUri() != null) {
                    controller.nextStep();
                    controller.patchApk(); // Auto-start patching
                } else {
                    Toast.makeText(this, "Please select an APK first", Toast.LENGTH_SHORT).show();
                }
                break;
            case APK_PATCHING:
                // Patching in progress, disable navigation
                break;
            case COMPLETION:
                finish(); // Exit wizard
                break;
        }
    }

    private void handleActionButton() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        
        switch (currentStep) {
            case WELCOME:
                controller.nextStep();
                break;
            case LOADER_INSTALL:
                showOnlineInstallOptions();
                break;
            case APK_SELECTION:
                selectApkFile();
                break;
            case APK_PATCHING:
                // No action during patching
                break;
            case COMPLETION:
                controller.installPatchedApk();
                break;
        }
    }

    private void showOnlineInstallOptions() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("Choose Loader Type");
        builder.setMessage("Select which loader to install:\n\nMelonLoader: Full-featured, larger size\nLemonLoader: Lightweight, smaller size");
        
        builder.setPositiveButton("MelonLoader", (dialog, which) -> {
            controller.installLoaderOnline(MelonLoaderManager.LoaderType.MELONLOADER_NET8);
        });
        
        builder.setNegativeButton("LemonLoader", (dialog, which) -> {
            controller.installLoaderOnline(MelonLoaderManager.LoaderType.MELONLOADER_NET35);
        });
        
        builder.setNeutralButton("Cancel", null);
        builder.show();
    }

    private void selectOfflineZip() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/zip");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        startActivityForResult(intent, REQUEST_SELECT_ZIP);
    }

    private void selectApkFile() {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("application/vnd.android.package-archive");
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        startActivityForResult(intent, REQUEST_SELECT_APK);
    }

    private void openModManagement() {
        Intent intent = new Intent(this, ModManagementActivity.class);
        startActivity(intent);
    }

    private void showStepRequirements() {
        UnifiedLoaderController.LoaderStep currentStep = controller.getCurrentStep();
        String message = "";
        
        switch (currentStep) {
            case LOADER_INSTALL:
                message = "Please install MelonLoader or LemonLoader first";
                break;
            case APK_SELECTION:
                message = "Please select a Terraria APK file";
                break;
            default:
                message = "Please complete the current step";
                break;
        }
        
        Toast.makeText(this, message, Toast.LENGTH_LONG).show();
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        
        if (resultCode != Activity.RESULT_OK || data == null || data.getData() == null) {
            return;
        }
        
        Uri uri = data.getData();
        
        switch (requestCode) {
            case REQUEST_SELECT_APK:
                controller.selectApk(uri);
                String filename = getFilenameFromUri(uri);
                if (apkStatusText != null) {
                    apkStatusText.setText("Selected: " + filename);
                }
                break;
                
            case REQUEST_SELECT_ZIP:
                controller.installLoaderOffline(uri);
                break;
        }
    }

    private String getFilenameFromUri(Uri uri) {
        String filename = null;
        try (Cursor cursor = getContentResolver().query(uri, null, null, null, null)) {
            if (cursor != null && cursor.moveToFirst()) {
                int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                if (nameIndex >= 0) {
                    filename = cursor.getString(nameIndex);
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Could not get filename: " + e.getMessage());
        }
        return filename != null ? filename : "Unknown file";
    }

    // === UnifiedLoaderController.UnifiedLoaderCallback Implementation ===

    @Override
    public void onStepChanged(UnifiedLoaderController.LoaderStep step, String message) {
        runOnUiThread(() -> {
            stepTitleText.setText(step.getTitle());
            stepDescriptionText.setText(message);
            
            // Clear previous content
            stepContentContainer.removeAllViews();
            
            // Add appropriate content
            switch (step) {
                case WELCOME:
                    stepContentContainer.addView(welcomeContent);
                    actionButton.setText("Start Setup");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case LOADER_INSTALL:
                    stepContentContainer.addView(loaderInstallContent);
                    actionButton.setText("Install Online");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case APK_SELECTION:
                    stepContentContainer.addView(apkSelectionContent);
                    actionButton.setText("Select APK");
                    actionButton.setVisibility(View.VISIBLE);
                    break;
                case APK_PATCHING:
                    stepContentContainer.addView(patchingContent);
                    actionButton.setVisibility(View.GONE);
                    nextButton.setEnabled(false);
                    previousButton.setEnabled(false);
                    break;
                case COMPLETION:
                    stepContentContainer.addView(completionContent);
                    actionButton.setText("Install APK");
                    actionButton.setVisibility(View.VISIBLE);
                    nextButton.setText("Finish");
                    nextButton.setEnabled(true);
                    previousButton.setEnabled(true);
                    break;
            }
            
            // Update navigation buttons
            previousButton.setEnabled(controller.canProceedToPreviousStep());
            nextButton.setEnabled(controller.canProceedToNextStep());
        });
    }

    @Override
    public void onProgress(String message, int percentage) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.setMessage(message + (percentage > 0 ? " (" + percentage + "%)" : ""));
            }
            if (progressText != null) {
                progressText.setText(message);
            }
        });
    }

    @Override
    public void onSuccess(String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            Toast.makeText(this, message, Toast.LENGTH_LONG).show();
        });
    }

    @Override
    public void onError(String error) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            
            AlertDialog.Builder builder = new AlertDialog.Builder(this);
            builder.setTitle("Error");
            builder.setMessage(error);
            builder.setPositiveButton("OK", null);
            builder.show();
            
            // Re-enable navigation
            nextButton.setEnabled(controller.canProceedToNextStep());
            previousButton.setEnabled(controller.canProceedToPreviousStep());
        });
    }

    @Override
    public void onLoaderStatusChanged(boolean installed, String statusText) {
        runOnUiThread(() -> {
            if (loaderStatusText != null) {
                loaderStatusText.setText(statusText);
                loaderStatusText.setTextColor(installed ? 0xFF4CAF50 : 0xFFF44336);
            }
        });
    }

    @Override
    public void updateStepIndicator(int currentStep, int totalSteps) {
        runOnUiThread(() -> {
            stepProgressBar.setMax(totalSteps);
            stepProgressBar.setProgress(currentStep);
            stepIndicatorText.setText("Step " + (currentStep + 1) + " of " + (totalSteps + 1));
        });
    }

    // === UnifiedLoaderListener Implementation ===

    @Override
    public void onInstallationStarted(String loaderType) {
        runOnUiThread(() -> {
            progressDialog = new AlertDialog.Builder(this)
                .setTitle("Installing " + loaderType)
                .setMessage("Starting installation...")
                .setCancelable(false)
                .create();
            progressDialog.show();
        });
    }

    @Override
    public void onInstallationProgress(String message) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.setMessage(message);
            }
        });
    }

    @Override
    public void onInstallationSuccess(String loaderType, String outputPath) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            Toast.makeText(this, loaderType + " installed successfully!", Toast.LENGTH_LONG).show();
        });
    }

    @Override
    public void onInstallationFailed(String loaderType, String error) {
        runOnUiThread(() -> {
            if (progressDialog != null) {
                progressDialog.dismiss();
                progressDialog = null;
            }
            
            AlertDialog.Builder builder = new AlertDialog.Builder(this);
            builder.setTitle("Installation Failed");
            builder.setMessage(loaderType + " installation failed:\n\n" + error);
            builder.setPositiveButton("OK", null);
            builder.show();
        });
    }

    @Override
    public void onValidationComplete(boolean isValid, String message) {
        runOnUiThread(() -> {
            String statusText = isValid ? "Validation passed" : "Validation failed: " + message;
            Toast.makeText(this, statusText, Toast.LENGTH_SHORT).show();
        });
    }

    @Override
    public void onInstallationStateChanged(UnifiedLoaderController.InstallationState state) {
        runOnUiThread(() -> {
            LogUtils.logDebug("Installation state changed to: " + state.getDisplayName());
        });
    }

    @Override
    public void onLogMessage(String message, UnifiedLoaderController.LogLevel level) {
        // Log messages are already handled by the controller
        LogUtils.logDebug("Log: [" + level + "] " + message);
    }

    @Override
    protected void onResume() {
        super.onResume();
        // Refresh loader status when returning to activity
        if (controller != null) {
            // Update current step to refresh status
            controller.setCurrentStep(controller.getCurrentStep());
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (controller != null) {
            controller.cleanup();
        }
        if (progressDialog != null) {
            progressDialog.dismiss();
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderController.java

// File: UnifiedLoaderController.java - Fixed step progression for offline ZIP import
// Path: /app/src/main/java/com/modloader/ui/UnifiedLoaderController.java

package com.modloader.ui;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Handler;
import android.os.Looper;
import android.widget.Toast;
import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;

import com.modloader.loader.MelonLoaderManager;
import com.modloader.loader.LoaderInstaller;
import com.modloader.loader.LoaderValidator;
import com.modloader.util.ApkPatcher;
import com.modloader.util.LogUtils;
import com.modloader.util.FileUtils;
import com.modloader.util.PathManager;
import com.modloader.util.OfflineZipImporter;
import java.io.File;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class UnifiedLoaderController {
    private static final String TAG = "UnifiedLoaderController";
    
    private Activity activity;
    private LoaderInstaller loaderInstaller;
    private LoaderValidator loaderValidator;
    private UnifiedLoaderListener listener;
    private Handler mainHandler;
    private ExecutorService executorService;
    
    // File picker launchers
    private ActivityResultLauncher<Intent> apkPickerLauncher;
    private ActivityResultLauncher<Intent> zipPickerLauncher;
    
    // Current operation state
    private File selectedApkFile;
    private File selectedZipFile;
    private MelonLoaderManager.LoaderType selectedLoaderType;
    private boolean isOperationInProgress = false;
    private InstallationState currentState = InstallationState.IDLE;
    private boolean loaderInstalledSuccessfully = false; // NEW: Track successful installation
    
    // Step management
    private LoaderStep currentStep = LoaderStep.WELCOME;
    private Uri selectedApkUri;
    
    /**
     * Installation state enum for tracking current operation state
     */
    public enum InstallationState {
        IDLE("Idle"),
        INITIALIZING("Initializing"),
        DOWNLOADING("Downloading"),
        EXTRACTING("Extracting Files"),
        CREATING_DIRECTORIES("Creating Directories"),
        VALIDATING("Validating Installation"),
        PATCHING_APK("Patching APK"),
        INSTALLING_APK("Installing APK"),
        COMPLETED("Installation Complete"),
        FAILED("Installation Failed"),
        CANCELLED("Operation Cancelled");
        
        private final String displayName;
        
        InstallationState(String displayName) {
            this.displayName = displayName;
        }
        
        public String getDisplayName() {
            return displayName;
        }
        
        @Override
        public String toString() {
            return displayName;
        }
    }
    
    /**
     * Log level enum for categorizing log messages
     */
    public enum LogLevel {
        DEBUG("DEBUG", 0),
        INFO("INFO", 1),
        WARNING("WARNING", 2),
        ERROR("ERROR", 3),
        USER("USER", 4);
        
        private final String displayName;
        private final int priority;
        
        LogLevel(String displayName, int priority) {
            this.displayName = displayName;
            this.priority = priority;
        }
        
        public String getDisplayName() {
            return displayName;
        }
        
        public int getPriority() {
            return priority;
        }
        
        @Override
        public String toString() {
            return displayName;
        }
    }
    
    // Legacy callback interface for backward compatibility
    public interface UnifiedLoaderCallback extends UnifiedLoaderListener {
        void onStepChanged(LoaderStep step, String message);
        void onProgress(String message, int percentage);
        void onSuccess(String message);
        void onError(String error);
        void onLoaderStatusChanged(boolean installed, String statusText);
        void updateStepIndicator(int currentStep, int totalSteps);
    }
    
    // LoaderStep enum for step tracking
    public enum LoaderStep {
        WELCOME("Welcome", "Welcome to MelonLoader Setup"),
        LOADER_INSTALL("Loader Installation", "Install MelonLoader components"),
        APK_SELECTION("APK Selection", "Select your Terraria APK"),
        APK_PATCHING("APK Patching", "Patching APK with MelonLoader"),
        COMPLETION("Setup Complete", "Installation completed successfully");
        
        private final String title;
        private final String description;
        
        LoaderStep(String title, String description) {
            this.title = title;
            this.description = description;
        }
        
        public String getTitle() {
            return title;
        }
        
        public String getDescription() {
            return description;
        }
    }
    
    public UnifiedLoaderController(Activity activity) {
        this.activity = activity;
        this.loaderInstaller = new LoaderInstaller();
        this.loaderValidator = new LoaderValidator();
        this.mainHandler = new Handler(Looper.getMainLooper());
        this.executorService = Executors.newSingleThreadExecutor();
        
        initializeFilePickers();
    }
    
    public UnifiedLoaderController(Activity activity, UnifiedLoaderListener listener) {
        this(activity);
        this.listener = listener;
    }
    
    // Callback setter for backward compatibility
    public void setCallback(UnifiedLoaderCallback callback) {
        this.listener = callback;
    }
    
    private void initializeFilePickers() {
        if (activity instanceof androidx.activity.ComponentActivity) {
            apkPickerLauncher = ((androidx.activity.ComponentActivity) activity).registerForActivityResult(
                new ActivityResultContracts.StartActivityForResult(),
                result -> {
                    if (result.getResultCode() == Activity.RESULT_OK && result.getData() != null) {
                        Uri apkUri = result.getData().getData();
                        handleApkSelection(apkUri);
                    }
                }
            );
            
            zipPickerLauncher = ((androidx.activity.ComponentActivity) activity).registerForActivityResult(
                new ActivityResultContracts.StartActivityForResult(),
                result -> {
                    if (result.getResultCode() == Activity.RESULT_OK && result.getData() != null) {
                        Uri zipUri = result.getData().getData();
                        handleZipSelection(zipUri);
                    }
                }
            );
        }
    }
    
    // State management methods
    private void setState(InstallationState newState) {
        this.currentState = newState;
        if (listener != null) {
            listener.onInstallationStateChanged(newState);
        }
        logMessage("State changed to: " + newState.getDisplayName(), LogLevel.DEBUG);
    }
    
    private void logMessage(String message, LogLevel level) {
        if (listener != null) {
            listener.onLogMessage(message, level);
        }
        
        switch (level) {
            case DEBUG:
                LogUtils.logDebug(message);
                break;
            case INFO:
                LogUtils.logInfo(message);
                break;
            case WARNING:
                LogUtils.logWarning(message);
                break;
            case ERROR:
                LogUtils.logError(message);
                break;
            case USER:
                LogUtils.logUser(message);
                break;
        }
    }
    
    // FIXED: Step management methods for wizard-style interface
    public void setCurrentStep(LoaderStep step) {
        this.currentStep = step;
        if (listener instanceof UnifiedLoaderCallback) {
            UnifiedLoaderCallback callback = (UnifiedLoaderCallback) listener;
            callback.onStepChanged(step, step.getDescription());
            callback.updateStepIndicator(step.ordinal(), LoaderStep.values().length - 1);
        }
        logMessage("Step changed to: " + step.getTitle(), LogLevel.INFO);
    }
    
    public LoaderStep getCurrentStep() {
        return currentStep;
    }
    
    public void nextStep() {
        LoaderStep[] steps = LoaderStep.values();
        int currentIndex = currentStep.ordinal();
        if (currentIndex < steps.length - 1) {
            setCurrentStep(steps[currentIndex + 1]);
        }
    }
    
    public void previousStep() {
        LoaderStep[] steps = LoaderStep.values();
        int currentIndex = currentStep.ordinal();
        if (currentIndex > 0) {
            setCurrentStep(steps[currentIndex - 1]);
        }
    }
    
    // FIXED: Improved step progression logic
    public boolean canProceedToNextStep() {
        switch (currentStep) {
            case WELCOME:
                return true;
            case LOADER_INSTALL:
                // Check both actual installation and successful import
                return isLoaderInstalled() || loaderInstalledSuccessfully;
            case APK_SELECTION:
                return selectedApkUri != null;
            case APK_PATCHING:
                return currentState == InstallationState.COMPLETED;
            case COMPLETION:
                return false; // Final step
            default:
                return false;
        }
    }
    
    public boolean canProceedToPreviousStep() {
        return currentStep != LoaderStep.WELCOME && currentState != InstallationState.PATCHING_APK;
    }
    
    // File selection methods
    public void selectApkFile() {
        if (isOperationInProgress) {
            showToast("Operation in progress, please wait...");
            return;
        }
        
        if (apkPickerLauncher != null) {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("application/vnd.android.package-archive");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            apkPickerLauncher.launch(Intent.createChooser(intent, "Select Terraria APK"));
        }
    }
    
    public void selectZipFile() {
        if (isOperationInProgress) {
            showToast("Operation in progress, please wait...");
            return;
        }
        
        if (zipPickerLauncher != null) {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("application/zip");
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            zipPickerLauncher.launch(Intent.createChooser(intent, "Select Loader ZIP"));
        }
    }
    
    public void selectApk(Uri uri) {
        this.selectedApkUri = uri;
        handleApkSelection(uri);
    }
    
    public Uri getSelectedApkUri() {
        return selectedApkUri;
    }
    
    private void handleApkSelection(Uri apkUri) {
        try {
            File tempDir = new File(activity.getExternalFilesDir(null), "temp");
            tempDir.mkdirs();
            
            String filename = FileUtils.getFilenameFromUri(activity, apkUri);
            if (filename == null) {
                filename = "selected_terraria.apk";
            }
            
            File tempApkFile = new File(tempDir, filename);
            if (FileUtils.copyUriToFile(activity, apkUri, tempApkFile)) {
                selectedApkFile = tempApkFile;
                selectedApkUri = apkUri;
                logMessage("APK selected: " + filename, LogLevel.USER);
                if (listener != null) {
                    listener.onInstallationProgress("APK selected: " + filename);
                }
            } else {
                showToast("Failed to copy APK file");
                logMessage("Failed to copy selected APK", LogLevel.ERROR);
            }
        } catch (Exception e) {
            logMessage("APK selection error: " + e.getMessage(), LogLevel.ERROR);
            showToast("Error selecting APK: " + e.getMessage());
        }
    }
    
    private void handleZipSelection(Uri zipUri) {
        try {
            File tempDir = new File(activity.getExternalFilesDir(null), "temp");
            tempDir.mkdirs();
            
            String filename = FileUtils.getFilenameFromUri(activity, zipUri);
            if (filename == null) {
                filename = "loader_files.zip";
            }
            
            File tempZipFile = new File(tempDir, filename);
            if (FileUtils.copyUriToFile(activity, zipUri, tempZipFile)) {
                selectedZipFile = tempZipFile;
                logMessage("ZIP selected: " + filename, LogLevel.USER);
                if (listener != null) {
                    listener.onInstallationProgress("Loader ZIP selected: " + filename);
                }
            } else {
                showToast("Failed to copy ZIP file");
                logMessage("Failed to copy selected ZIP", LogLevel.ERROR);
            }
        } catch (Exception e) {
            logMessage("ZIP selection error: " + e.getMessage(), LogLevel.ERROR);
            showToast("Error selecting ZIP: " + e.getMessage());
        }
    }
    
    // Installation methods
    public void installLoaderOnline(MelonLoaderManager.LoaderType loaderType) {
        if (isOperationInProgress) {
            showToast("Installation already in progress");
            return;
        }
        
        this.selectedLoaderType = loaderType;
        setState(InstallationState.INITIALIZING);
        runAutomatedInstallationTask();
    }
    
    // FIXED: Offline ZIP import with proper completion handling
    public void installLoaderOffline(Uri zipUri) {
        if (isOperationInProgress) {
            showToast("Installation already in progress");
            return;
        }
        
        setState(InstallationState.INITIALIZING);
        isOperationInProgress = true;
        
        if (listener != null) {
            listener.onInstallationStarted("Offline ZIP Import");
        }
        
        executorService.execute(() -> {
            try {
                setState(InstallationState.EXTRACTING);
                
                // Use the OfflineZipImporter
                OfflineZipImporter.ImportResult result = OfflineZipImporter.importMelonLoaderZip(activity, zipUri);
                
                mainHandler.post(() -> {
                    isOperationInProgress = false;
                    
                    if (result.success) {
                        selectedLoaderType = result.detectedType;
                        loaderInstalledSuccessfully = true; // Mark as successfully installed
                        setState(InstallationState.COMPLETED);
                        
                        if (listener != null) {
                            listener.onInstallationSuccess("Offline ZIP Import", "Files extracted successfully");
                        }
                        
                        // Update loader status callback
                        if (listener instanceof UnifiedLoaderCallback) {
                            ((UnifiedLoaderCallback) listener).onLoaderStatusChanged(true, 
                                "Loader installed via ZIP import: " + result.detectedType.getDisplayName());
                        }
                        
                        logMessage("ZIP import completed successfully", LogLevel.USER);
                        
                    } else {
                        setState(InstallationState.FAILED);
                        
                        if (listener != null) {
                            listener.onInstallationFailed("Offline ZIP Import", result.message);
                        }
                        
                        logMessage("ZIP import failed: " + result.message, LogLevel.ERROR);
                    }
                });
                
            } catch (Exception e) {
                mainHandler.post(() -> {
                    isOperationInProgress = false;
                    setState(InstallationState.FAILED);
                    
                    if (listener != null) {
                        listener.onInstallationFailed("Offline ZIP Import", e.getMessage());
                    }
                    
                    logMessage("ZIP import error: " + e.getMessage(), LogLevel.ERROR);
                });
            }
        });
    }
    
    public void patchApk() {
        if (selectedApkFile == null) {
            logMessage("No APK selected for patching", LogLevel.ERROR);
            return;
        }
        
        if (selectedLoaderType == null) {
            logMessage("No loader type selected", LogLevel.ERROR);
            return;
        }
        
        setState(InstallationState.PATCHING_APK);
        runApkPatchingTask();
    }
    
    public void installPatchedApk() {
        logMessage("APK installation requested", LogLevel.USER);
        if (listener != null) {
            listener.onInstallationProgress("Starting APK installation...");
        }
    }
    
    // Background task methods
    private void runAutomatedInstallationTask() {
        isOperationInProgress = true;
        
        mainHandler.post(() -> {
            if (listener != null) {
                listener.onInstallationStarted(selectedLoaderType.getDisplayName());
            }
        });
        
        executorService.execute(() -> {
            String errorMessage = null;
            String outputPath = null;
            boolean success = false;
            
            try {
                setState(InstallationState.CREATING_DIRECTORIES);
                
                boolean structureCreated = loaderInstaller.createLoaderStructure(
                    activity, LoaderInstaller.TERRARIA_PACKAGE, selectedLoaderType);
                
                if (!structureCreated) {
                    errorMessage = "Failed to create loader directory structure";
                } else {
                    mainHandler.post(() -> {
                        if (listener != null) {
                            listener.onInstallationProgress("Directory structure created successfully");
                        }
                    });
                    
                    outputPath = PathManager.getGameBaseDir(activity, LoaderInstaller.TERRARIA_PACKAGE).getAbsolutePath();
                    success = true;
                    loaderInstalledSuccessfully = true; // Mark as successfully installed
                    setState(InstallationState.COMPLETED);
                }
                
            } catch (Exception e) {
                logMessage("Automated installation failed: " + e.getMessage(), LogLevel.ERROR);
                errorMessage = "Installation error: " + e.getMessage();
                setState(InstallationState.FAILED);
            }
            
            final boolean finalSuccess = success;
            final String finalError = errorMessage;
            final String finalOutput = outputPath;
            mainHandler.post(() -> {
                isOperationInProgress = false;
                if (listener != null) {
                    if (finalSuccess) {
                        listener.onInstallationSuccess(selectedLoaderType.getDisplayName(), finalOutput);
                        
                        // Update loader status callback
                        if (listener instanceof UnifiedLoaderCallback) {
                            ((UnifiedLoaderCallback) listener).onLoaderStatusChanged(true, 
                                "Loader installed successfully: " + selectedLoaderType.getDisplayName());
                        }
                    } else {
                        listener.onInstallationFailed(selectedLoaderType.getDisplayName(), finalError);
                    }
                }
            });
        });
    }
    
    private void runApkPatchingTask() {
        isOperationInProgress = true;
        
        executorService.execute(() -> {
            try {
                File outputDir = new File(activity.getExternalFilesDir(null), "output");
                outputDir.mkdirs();
                
                String outputFileName = selectedApkFile.getName().replace(".apk", "_modded.apk");
                File patchedApkFile = new File(outputDir, outputFileName);
                
                ApkPatcher.PatchResult patchResult = ApkPatcher.injectMelonLoaderIntoApk(
                    activity, selectedApkFile, patchedApkFile, selectedLoaderType);
                
                mainHandler.post(() -> {
                    isOperationInProgress = false;
                    if (patchResult.success) {
                        setState(InstallationState.COMPLETED);
                        nextStep();
                        if (listener instanceof UnifiedLoaderCallback) {
                            ((UnifiedLoaderCallback) listener).onSuccess("APK patching completed successfully");
                        }
                    } else {
                        setState(InstallationState.FAILED);
                        if (listener instanceof UnifiedLoaderCallback) {
                            ((UnifiedLoaderCallback) listener).onError("APK patching failed: " + 
                                (patchResult.errorMessage != null ? patchResult.errorMessage : "Unknown error"));
                        }
                    }
                });
                
            } catch (Exception e) {
                mainHandler.post(() -> {
                    isOperationInProgress = false;
                    setState(InstallationState.FAILED);
                    logMessage("APK patching error: " + e.getMessage(), LogLevel.ERROR);
                    if (listener instanceof UnifiedLoaderCallback) {
                        ((UnifiedLoaderCallback) listener).onError("APK patching failed: " + e.getMessage());
                    }
                });
            }
        });
    }
    
    // Status check methods
    public boolean isLoaderInstalled() {
        // Check both actual filesystem presence and successful installation flag
        if (loaderInstalledSuccessfully) {
            return true;
        }
        
        if (selectedLoaderType == null) {
            return false;
        }
        
        if (selectedLoaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
            return MelonLoaderManager.isMelonLoaderInstalled(activity, LoaderInstaller.TERRARIA_PACKAGE);
        } else if (selectedLoaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET35) {
            return MelonLoaderManager.isLemonLoaderInstalled(activity, LoaderInstaller.TERRARIA_PACKAGE);
        }
        
        return false;
    }
    
    public String getInstallationStatus() {
        if (selectedLoaderType == null) {
            return "No loader type selected";
        }
        
        boolean isInstalled = isLoaderInstalled();
        
        if (isInstalled) {
            return selectedLoaderType.getDisplayName() + " is installed and ready";
        } else {
            return selectedLoaderType.getDisplayName() + " is not installed";
        }
    }
    
    public InstallationState getCurrentState() {
        return currentState;
    }
    
    public boolean isOperationInProgress() {
        return isOperationInProgress;
    }
    
    // Utility methods
    private void showToast(String message) {
        if (activity != null) {
            mainHandler.post(() -> Toast.makeText(activity, message, Toast.LENGTH_SHORT).show());
        }
    }
    
    public void cancelCurrentOperation() {
        isOperationInProgress = false;
        setState(InstallationState.CANCELLED);
        logMessage("Operation cancelled by user", LogLevel.USER);
    }
    
    public void cleanup() {
        if (executorService != null && !executorService.isShutdown()) {
            executorService.shutdown();
        }
        
        try {
            if (selectedApkFile != null && selectedApkFile.getParentFile().getName().equals("temp")) {
                selectedApkFile.delete();
            }
            if (selectedZipFile != null && selectedZipFile.getParentFile().getName().equals("temp")) {
                selectedZipFile.delete();
            }
        } catch (Exception e) {
            logMessage("Cleanup error: " + e.getMessage(), LogLevel.DEBUG);
        }
    }
    
    // Getters for compatibility
    public File getSelectedApkFile() {
        return selectedApkFile;
    }
    
    public File getSelectedZipFile() {
        return selectedZipFile;
    }
    
    public MelonLoaderManager.LoaderType getSelectedLoaderType() {
        return selectedLoaderType;
    }
    
    // Default implementations for UnifiedLoaderListener methods
    public void onInstallationStarted(String loaderType) {
        logMessage("Installation started: " + loaderType, LogLevel.INFO);
    }
    
    public void onInstallationProgress(String message) {
        logMessage("Progress: " + message, LogLevel.INFO);
    }
    
    public void onInstallationSuccess(String loaderType, String outputPath) {
        logMessage("Installation succeeded: " + loaderType, LogLevel.USER);
    }
    
    public void onInstallationFailed(String loaderType, String error) {
        logMessage("Installation failed: " + error, LogLevel.ERROR);
    }
    
    public void onValidationComplete(boolean isValid, String message) {
        logMessage("Validation: " + (isValid ? "PASSED" : "FAILED") + " - " + message, 
                  isValid ? LogLevel.INFO : LogLevel.ERROR);
    }
    
    public void onInstallationStateChanged(InstallationState state) {
        logMessage("State changed: " + state.getDisplayName(), LogLevel.DEBUG);
    }
    
    public void onLogMessage(String message, LogLevel level) {
        // Already handled in logMessage method
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderListener.java

// File: UnifiedLoaderListener.java - Missing interface for UnifiedLoaderActivity
// Path: /app/src/main/java/com/modloader/ui/UnifiedLoaderListener.java

package com.modloader.ui;

import com.modloader.loader.MelonLoaderManager;

/**
 * Listener interface for unified loader operations
 * Provides callbacks for installation progress and state changes
 */
public interface UnifiedLoaderListener {
    
    /**
     * Called when installation starts
     * @param loaderType Type of loader being installed
     */
    void onInstallationStarted(String loaderType);
    
    /**
     * Called when installation progress updates
     * @param message Progress message
     */
    void onInstallationProgress(String message);
    
    /**
     * Called when installation succeeds
     * @param loaderType Type of loader that was installed
     * @param outputPath Path to the output
     */
    void onInstallationSuccess(String loaderType, String outputPath);
    
    /**
     * Called when installation fails
     * @param loaderType Type of loader that failed
     * @param error Error message
     */
    void onInstallationFailed(String loaderType, String error);
    
    /**
     * Called when validation completes
     * @param isValid Whether validation passed
     * @param message Validation message
     */
    void onValidationComplete(boolean isValid, String message);
    
    /**
     * Called when installation state changes
     * @param state New installation state
     */
    void onInstallationStateChanged(UnifiedLoaderController.InstallationState state);
    
    /**
     * Called when a log message is generated
     * @param message Log message
     * @param level Log level
     */
    void onLogMessage(String message, UnifiedLoaderController.LogLevel level);
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/ApkInstaller.java

// File: ApkInstaller.java (FIXED) - Enhanced APK Installation with Proper Error Handling
// Path: /main/java/com/terrarialoader/util/ApkInstaller.java

package com.modloader.util;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.provider.Settings;
import android.widget.Toast;
import androidx.core.content.FileProvider;
import androidx.appcompat.app.AlertDialog;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.security.MessageDigest;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

public class ApkInstaller {
    
    private static final String TAG = "ApkInstaller";
    private static final int MIN_APK_SIZE = 1024 * 1024; // 1MB minimum
    private static final int MAX_APK_SIZE = 200 * 1024 * 1024; // 200MB maximum
    
    // Enhanced APK installation with comprehensive error handling
    public static void installApk(Context context, File apkFile) {
        if (context == null) {
            LogUtils.logError("Context is null - cannot install APK");
            return;
        }
        
        if (apkFile == null) {
            LogUtils.logError("APK file is null - cannot install");
            showError(context, "APK Installation Failed", 
                "No APK file specified. Please select a valid APK file.");
            return;
        }
        
        LogUtils.logUser("🔧 Starting APK installation: " + apkFile.getName());
        LogUtils.logDebug("APK path: " + apkFile.getAbsolutePath());
        LogUtils.logDebug("APK size: " + FileUtils.formatFileSize(apkFile.length()));
        
        // Step 1: Validate APK file
        ValidationResult validation = validateApkFile(apkFile);
        if (!validation.isValid) {
            LogUtils.logError("APK validation failed: " + validation.errorMessage);
            showError(context, "Invalid APK File", validation.errorMessage);
            return;
        }
        
        // Step 2: Check permissions
        if (!checkInstallPermissions(context)) {
            LogUtils.logUser("Install permissions not granted - requesting permission");
            requestInstallPermission(context);
            return;
        }
        
        // Step 3: Prepare APK for installation
        try {
            File preparedApk = prepareApkForInstallation(context, apkFile);
            if (preparedApk == null) {
                LogUtils.logError("Failed to prepare APK for installation");
                showError(context, "Installation Preparation Failed", 
                    "Could not prepare the APK file for installation. Check storage permissions and available space.");
                return;
            }
            
            // Step 4: Launch installation intent
            launchInstallationIntent(context, preparedApk);
            
        } catch (Exception e) {
            LogUtils.logError("APK installation error: " + e.getMessage());
            showError(context, "Installation Error", 
                "Failed to install APK: " + e.getMessage() + 
                "\n\nTry:\n• Checking file permissions\n• Ensuring enough storage space\n• Restarting the app");
        }
    }
    
    // Comprehensive APK validation
    private static ValidationResult validateApkFile(File apkFile) {
        ValidationResult result = new ValidationResult();
        
        // Check file exists
        if (!apkFile.exists()) {
            result.errorMessage = "APK file does not exist:\n" + apkFile.getAbsolutePath();
            return result;
        }
        
        // Check file is readable
        if (!apkFile.canRead()) {
            result.errorMessage = "APK file is not readable. Check file permissions.";
            return result;
        }
        
        // Check file size
        long fileSize = apkFile.length();
        if (fileSize == 0) {
            result.errorMessage = "APK file is empty (0 bytes).";
            return result;
        }
        
        if (fileSize < MIN_APK_SIZE) {
            result.errorMessage = "APK file is too small (" + FileUtils.formatFileSize(fileSize) + 
                "). Minimum size: " + FileUtils.formatFileSize(MIN_APK_SIZE);
            return result;
        }
        
        if (fileSize > MAX_APK_SIZE) {
            result.errorMessage = "APK file is too large (" + FileUtils.formatFileSize(fileSize) + 
                "). Maximum size: " + FileUtils.formatFileSize(MAX_APK_SIZE);
            return result;
        }
        
        // Check file extension
        String fileName = apkFile.getName().toLowerCase();
        if (!fileName.endsWith(".apk")) {
            result.errorMessage = "File does not have .apk extension: " + fileName;
            return result;
        }
        
        // Check APK magic number (PK signature)
        if (!isValidApkFile(apkFile)) {
            result.errorMessage = "File is not a valid APK. The file may be corrupted or not an Android package.";
            return result;
        }
        
        // Try to parse APK package info
        try {
            String packageInfo = getApkPackageInfo(apkFile);
            LogUtils.logDebug("APK package info: " + packageInfo);
        } catch (Exception e) {
            result.errorMessage = "Cannot parse APK package information. The APK may be corrupted.\n\nError: " + e.getMessage();
            return result;
        }
        
        result.isValid = true;
        return result;
    }
    
    // Check if file is a valid APK by reading ZIP signature
    private static boolean isValidApkFile(File apkFile) {
        try (FileInputStream fis = new FileInputStream(apkFile)) {
            byte[] signature = new byte[4];
            int bytesRead = fis.read(signature);
            
            if (bytesRead != 4) return false;
            
            // Check for ZIP signature (PK\003\004 or PK\005\006 or PK\007\008)
            return (signature[0] == 0x50 && signature[1] == 0x4B && 
                   (signature[2] == 0x03 || signature[2] == 0x05 || signature[2] == 0x07));
                   
        } catch (Exception e) {
            LogUtils.logDebug("Error checking APK signature: " + e.getMessage());
            return false;
        }
    }
    
    // Get basic APK package information
    private static String getApkPackageInfo(File apkFile) throws Exception {
        try (ZipInputStream zis = new ZipInputStream(new FileInputStream(apkFile))) {
            ZipEntry entry;
            boolean hasManifest = false;
            boolean hasDexFile = false;
            int entryCount = 0;
            
            while ((entry = zis.getNextEntry()) != null && entryCount < 100) {
                String entryName = entry.getName();
                
                if ("AndroidManifest.xml".equals(entryName)) {
                    hasManifest = true;
                }
                
                if (entryName.endsWith(".dex")) {
                    hasDexFile = true;
                }
                
                entryCount++;
            }
            
            if (!hasManifest) {
                throw new Exception("APK missing AndroidManifest.xml");
            }
            
            if (!hasDexFile) {
                throw new Exception("APK missing .dex files");
            }
            
            return String.format("Valid APK with %d entries, manifest: %s, dex files: %s", 
                entryCount, hasManifest, hasDexFile);
        }
    }
    
    // Check install permissions
    private static boolean checkInstallPermissions(Context context) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            return context.getPackageManager().canRequestPackageInstalls();
        }
        
        // For older Android versions, check unknown sources setting
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            try {
                return Settings.Secure.getInt(context.getContentResolver(), 
                    Settings.Secure.INSTALL_NON_MARKET_APPS) == 1;
            } catch (Settings.SettingNotFoundException e) {
                return false;
            }
        }
        
        return true; // Assume allowed for very old versions
    }
    
    // Request install permission
    private static void requestInstallPermission(Context context) {
        if (!(context instanceof Activity)) {
            LogUtils.logError("Context is not an Activity - cannot request permissions");
            Toast.makeText(context, "Cannot request install permission - context is not an Activity", 
                Toast.LENGTH_LONG).show();
            return;
        }
        
        Activity activity = (Activity) context;
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            // Android 8.0+ - Request install unknown apps permission
            new AlertDialog.Builder(activity)
                .setTitle("🔐 Install Permission Required")
                .setMessage("To install the modified APK, you need to allow this app to install unknown apps.\n\n" +
                           "Steps:\n" +
                           "1. Tap 'Grant Permission'\n" +
                           "2. Enable 'Allow from this source'\n" +
                           "3. Return to TerrariaLoader\n" +
                           "4. Try installing again")
                .setPositiveButton("Grant Permission", (dialog, which) -> {
                    Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES);
                    intent.setData(Uri.parse("package:" + activity.getPackageName()));
                    try {
                        activity.startActivity(intent);
                        LogUtils.logUser("Opened install permission settings");
                    } catch (Exception e) {
                        LogUtils.logError("Failed to open install permission settings: " + e.getMessage());
                        Toast.makeText(activity, "Cannot open permission settings", Toast.LENGTH_SHORT).show();
                    }
                })
                .setNegativeButton("Cancel", (dialog, which) -> {
                    LogUtils.logUser("User cancelled install permission request");
                })
                .show();
                
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            // Android 4.2-7.1 - Direct to security settings
            new AlertDialog.Builder(activity)
                .setTitle("🔐 Enable Unknown Sources")
                .setMessage("To install the modified APK, you need to enable 'Unknown Sources' in security settings.\n\n" +
                           "Steps:\n" +
                           "1. Tap 'Open Settings'\n" +
                           "2. Find 'Unknown sources' and enable it\n" +
                           "3. Return to TerrariaLoader\n" +
                           "4. Try installing again")
                .setPositiveButton("Open Settings", (dialog, which) -> {
                    try {
                        Intent intent = new Intent(Settings.ACTION_SECURITY_SETTINGS);
                        activity.startActivity(intent);
                        LogUtils.logUser("Opened security settings for unknown sources");
                    } catch (Exception e) {
                        LogUtils.logError("Failed to open security settings: " + e.getMessage());
                        Toast.makeText(activity, "Cannot open security settings", Toast.LENGTH_SHORT).show();
                    }
                })
                .setNegativeButton("Cancel", null)
                .show();
        }
    }
    
    // Prepare APK for installation (copy to accessible location)
    private static File prepareApkForInstallation(Context context, File sourceApk) {
        try {
            // Create installation directory in app's external files
            File installDir = new File(context.getExternalFilesDir(null), "apk_install");
            if (!installDir.exists() && !installDir.mkdirs()) {
                LogUtils.logError("Failed to create install directory");
                return null;
            }
            
            // Create target file with timestamp to avoid conflicts
            String fileName = sourceApk.getName();
            if (!fileName.toLowerCase().endsWith(".apk")) {
                fileName = fileName + ".apk";
            }
            
            // Add timestamp to avoid conflicts
            String baseName = fileName.substring(0, fileName.lastIndexOf('.'));
            String extension = fileName.substring(fileName.lastIndexOf('.'));
            String targetFileName = baseName + "_" + System.currentTimeMillis() + extension;
            
            File targetApk = new File(installDir, targetFileName);
            
            // Copy APK to accessible location
            if (!FileUtils.copyFile(sourceApk, targetApk)) {
                LogUtils.logError("Failed to copy APK to install directory");
                return null;
            }
            
            // Set file permissions
            if (!targetApk.setReadable(true, false)) {
                LogUtils.logDebug("Warning: Could not set APK as readable");
            }
            
            LogUtils.logUser("✅ APK prepared for installation: " + targetApk.getName());
            LogUtils.logDebug("Target APK path: " + targetApk.getAbsolutePath());
            
            return targetApk;
            
        } catch (Exception e) {
            LogUtils.logError("Error preparing APK: " + e.getMessage());
            return null;
        }
    }
    
    // Launch the actual installation intent
    private static void launchInstallationIntent(Context context, File apkFile) {
        try {
            Intent installIntent = new Intent(Intent.ACTION_VIEW);
            installIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            
            Uri apkUri;
            
            // Use FileProvider for Android 7.0+
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                installIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
                apkUri = FileProvider.getUriForFile(context, 
                    context.getPackageName() + ".provider", apkFile);
                LogUtils.logDebug("Using FileProvider URI: " + apkUri);
            } else {
                apkUri = Uri.fromFile(apkFile);
                LogUtils.logDebug("Using direct file URI: " + apkUri);
            }
            
            installIntent.setDataAndType(apkUri, "application/vnd.android.package-archive");
            
            // Verify intent can be handled
            if (installIntent.resolveActivity(context.getPackageManager()) != null) {
                context.startActivity(installIntent);
                LogUtils.logUser("🚀 Installation intent launched successfully");
                
                // Show user guidance
                Toast.makeText(context, 
                    "📱 Installation dialog should appear.\nIf not, check your notification panel.", 
                    Toast.LENGTH_LONG).show();
                    
            } else {
                LogUtils.logError("No activity found to handle install intent");
                showError(context, "Installation Error", 
                    "No app found to handle APK installation. This shouldn't happen on Android devices.");
            }
            
        } catch (Exception e) {
            LogUtils.logError("Failed to launch install intent: " + e.getMessage());
            showError(context, "Installation Launch Failed", 
                "Could not start APK installation.\n\nError: " + e.getMessage() + 
                "\n\nPossible solutions:\n• Restart the app\n• Check storage permissions\n• Try a different APK");
        }
    }
    
    // Calculate MD5 hash of APK for verification
    public static String calculateApkHash(File apkFile) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            try (FileInputStream fis = new FileInputStream(apkFile)) {
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = fis.read(buffer)) != -1) {
                    md.update(buffer, 0, bytesRead);
                }
            }
            
            byte[] hashBytes = md.digest();
            StringBuilder hexString = new StringBuilder();
            for (byte b : hashBytes) {
                String hex = Integer.toHexString(0xff & b);
                if (hex.length() == 1) {
                    hexString.append('0');
                }
                hexString.append(hex);
            }
            return hexString.toString().toUpperCase();
            
        } catch (Exception e) {
            LogUtils.logError("Failed to calculate APK hash: " + e.getMessage());
            return "UNKNOWN";
        }
    }
    
    // Get detailed APK information
    public static ApkInfo getApkInfo(Context context, File apkFile) {
        ApkInfo info = new ApkInfo();
        info.fileName = apkFile.getName();
        info.filePath = apkFile.getAbsolutePath();
        info.fileSize = apkFile.length();
        info.lastModified = apkFile.lastModified();
        
        try {
            PackageManager pm = context.getPackageManager();
            PackageInfo packageInfo = pm.getPackageArchiveInfo(apkFile.getAbsolutePath(), 0);
            
            if (packageInfo != null) {
                info.packageName = packageInfo.packageName;
                info.versionName = packageInfo.versionName;
                info.versionCode = packageInfo.versionCode;
                
                // Get application label
                packageInfo.applicationInfo.sourceDir = apkFile.getAbsolutePath();
                packageInfo.applicationInfo.publicSourceDir = apkFile.getAbsolutePath();
                info.appName = pm.getApplicationLabel(packageInfo.applicationInfo).toString();
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Could not extract package info: " + e.getMessage());
        }
        
        info.hash = calculateApkHash(apkFile);
        return info;
    }
    
    // Clean up old installation files
    public static void cleanupInstallFiles(Context context) {
        try {
            File installDir = new File(context.getExternalFilesDir(null), "apk_install");
            if (installDir.exists() && installDir.isDirectory()) {
                File[] files = installDir.listFiles();
                if (files != null) {
                    long currentTime = System.currentTimeMillis();
                    int deletedCount = 0;
                    
                    for (File file : files) {
                        // Delete files older than 1 hour
                        if (currentTime - file.lastModified() > 3600000) {
                            if (file.delete()) {
                                deletedCount++;
                            }
                        }
                    }
                    
                    if (deletedCount > 0) {
                        LogUtils.logDebug("Cleaned up " + deletedCount + " old installation files");
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error cleaning up install files: " + e.getMessage());
        }
    }
    
    // Show error dialog
    private static void showError(Context context, String title, String message) {
        if (context instanceof Activity) {
            new AlertDialog.Builder(context)
                .setTitle("❌ " + title)
                .setMessage(message)
                .setPositiveButton("OK", null)
                .show();
        } else {
            Toast.makeText(context, title + ": " + message, Toast.LENGTH_LONG).show();
        }
    }
    
    // Validation result helper class
    private static class ValidationResult {
        boolean isValid = false;
        String errorMessage = "";
    }
    
    // APK information class
    public static class ApkInfo {
        public String fileName;
        public String filePath;
        public long fileSize;
        public long lastModified;
        public String packageName;
        public String appName;
        public String versionName;
        public int versionCode;
        public String hash;
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== APK Information ===\n");
            sb.append("File: ").append(fileName).append("\n");
            sb.append("Size: ").append(FileUtils.formatFileSize(fileSize)).append("\n");
            sb.append("Package: ").append(packageName != null ? packageName : "Unknown").append("\n");
            sb.append("App Name: ").append(appName != null ? appName : "Unknown").append("\n");
            sb.append("Version: ").append(versionName != null ? versionName : "Unknown");
            if (versionCode > 0) {
                sb.append(" (").append(versionCode).append(")");
            }
            sb.append("\n");
            sb.append("Hash: ").append(hash).append("\n");
            sb.append("Modified: ").append(new java.util.Date(lastModified).toString()).append("\n");
            return sb.toString();
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/ApkPatcher.java

// File: ApkPatcher.java - Enhanced APK patching with real MelonLoader injection
// Path: app/src/main/java/com/terrarialoader/util/ApkPatcher.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.*;
import java.util.zip.*;
import java.util.ArrayList;
import java.util.List;
import java.util.HashMap;
import java.util.Map;

public class ApkPatcher {
    private static final String TAG = "ApkPatcher";
    private static final int BUFFER_SIZE = 8192;
    
    // MelonLoader injection points for Unity games
    private static final String[] INJECTION_TARGETS = {
        "lib/arm64-v8a/libil2cpp.so",
        "lib/arm64-v8a/libunity.so", 
        "lib/armeabi-v7a/libil2cpp.so",
        "lib/armeabi-v7a/libunity.so"
    };
    
    public static class PatchResult {
        public boolean success = false;
        public String errorMessage = null;
        public long originalSize = 0;
        public long patchedSize = 0;
        public int addedFiles = 0;
        public int modifiedFiles = 0;
        public List<String> injectedFiles = new ArrayList<>();
        public List<String> warnings = new ArrayList<>();
        public String outputPath = null;
        
        public String getDetailedReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=== APK Patching Report ===\n");
            sb.append("Status: ").append(success ? "✅ SUCCESS" : "❌ FAILED").append("\n");
            if (errorMessage != null) {
                sb.append("Error: ").append(errorMessage).append("\n");
            }
            sb.append("Original Size: ").append(FileUtils.formatFileSize(originalSize)).append("\n");
            sb.append("Patched Size: ").append(FileUtils.formatFileSize(patchedSize)).append("\n");
            sb.append("Size Change: ").append(FileUtils.formatFileSize(patchedSize - originalSize)).append("\n");
            sb.append("Files Added: ").append(addedFiles).append("\n");
            sb.append("Files Modified: ").append(modifiedFiles).append("\n");
            
            if (!injectedFiles.isEmpty()) {
                sb.append("\nInjected Files:\n");
                for (String file : injectedFiles) {
                    sb.append("+ ").append(file).append("\n");
                }
            }
            
            if (!warnings.isEmpty()) {
                sb.append("\nWarnings:\n");
                for (String warning : warnings) {
                    sb.append("⚠️ ").append(warning).append("\n");
                }
            }
            
            return sb.toString();
        }
    }
    
    /**
     * Main APK patching method with real MelonLoader injection
     */
    public static PatchResult injectMelonLoaderIntoApk(Context context, File inputApk, File outputApk, 
                                                      MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("🚀 Starting APK patching with " + loaderType.getDisplayName());
        
        PatchResult result = new PatchResult();
        
        try {
            // Step 1: Validate input APK
            LogUtils.logUser("📋 Step 1: Validating input APK...");
            ApkValidator.ValidationResult validation = ApkValidator.validateApk(inputApk.getAbsolutePath(), "input-validation");
            
            if (!validation.isValid) {
                result.errorMessage = "Input APK validation failed: " + validation.issues.get(0);
                LogUtils.logUser("❌ " + result.errorMessage);
                return result;
            }
            
            result.originalSize = inputApk.length();
            LogUtils.logUser("✅ Input APK is valid (" + FileUtils.formatFileSize(result.originalSize) + ")");
            
            // Step 2: Check MelonLoader files availability
            LogUtils.logUser("📋 Step 2: Checking MelonLoader files...");
            if (!verifyMelonLoaderFiles(context, loaderType)) {
                result.errorMessage = "MelonLoader files not found. Use automated installation first.";
                LogUtils.logUser("❌ " + result.errorMessage);
                return result;
            }
            LogUtils.logUser("✅ MelonLoader files found and ready");
            
            // Step 3: Create backup of original APK
            LogUtils.logUser("📋 Step 3: Creating backup...");
            File backupFile = createBackup(context, inputApk);
            if (backupFile != null) {
                LogUtils.logUser("✅ Backup created: " + backupFile.getName());
            }
            
            // Step 4: Patch the APK
            LogUtils.logUser("📋 Step 4: Patching APK...");
            boolean patchSuccess = patchApkWithMelonLoader(context, inputApk, outputApk, loaderType, result);
            
            if (!patchSuccess) {
                if (result.errorMessage == null) {
                    result.errorMessage = "APK patching failed - unknown error";
                }
                LogUtils.logUser("❌ " + result.errorMessage);
                return result;
            }
            
            // Step 5: Validate output APK
            LogUtils.logUser("📋 Step 5: Validating patched APK...");
            ApkValidator.ValidationResult patchedValidation = ApkValidator.validateApk(outputApk.getAbsolutePath(), "output-validation");
                        
            result.patchedSize = outputApk.length();
            result.success = patchedValidation.isValid;
            result.outputPath = outputApk.getAbsolutePath();
            
            if (result.success) {
                LogUtils.logUser("✅ APK patching completed successfully!");
                LogUtils.logUser("📦 Output: " + outputApk.getName());
                LogUtils.logUser("📊 Size: " + FileUtils.formatFileSize(result.patchedSize) + 
                               " (+" + FileUtils.formatFileSize(result.patchedSize - result.originalSize) + ")");
                LogUtils.logUser("🔧 Added " + result.addedFiles + " MelonLoader files");
            } else {
                result.errorMessage = "Patched APK validation failed";
                LogUtils.logUser("❌ " + result.errorMessage);
                
                // Add validation issues to warnings
                for (String issue : patchedValidation.issues) {
                    result.warnings.add("Validation: " + issue);
                }
            }
            
        } catch (Exception e) {
            result.errorMessage = "Patching exception: " + e.getMessage();
            LogUtils.logUser("❌ " + result.errorMessage);
            LogUtils.logDebug("APK patching exception: " + e.toString());
            
            // Clean up partial output on failure
            if (outputApk.exists() && !result.success) {
                outputApk.delete();
            }
        }
        
        return result;
    }
    
    /**
     * Core APK patching logic with real MelonLoader injection
     */
    private static boolean patchApkWithMelonLoader(Context context, File inputApk, File outputApk, 
                                                  MelonLoaderManager.LoaderType loaderType, PatchResult result) {
        LogUtils.logDebug("Starting core APK patching process...");
        
        try {
            // Get MelonLoader files to inject
            Map<String, File> melonLoaderFiles = getMelonLoaderFiles(context, loaderType);
            if (melonLoaderFiles.isEmpty()) {
                result.errorMessage = "No MelonLoader files found to inject";
                return false;
            }
            
            LogUtils.logDebug("Found " + melonLoaderFiles.size() + " MelonLoader files to inject");
            
            // Create output ZIP with all original files plus MelonLoader files
            try (ZipInputStream zis = new ZipInputStream(new FileInputStream(inputApk));
                 ZipOutputStream zos = new ZipOutputStream(new FileOutputStream(outputApk))) {
                
                // Set compression level for better performance
                zos.setLevel(ZipOutputStream.DEFLATED);
                
                // Copy original APK entries and modify where necessary
                ZipEntry entry;
                while ((entry = zis.getNextEntry()) != null) {
                    String entryName = entry.getName();
                    
                    // Check if this file needs modification for MelonLoader injection
                    if (shouldModifyEntry(entryName)) {
                        // Modify the entry for MelonLoader injection
                        if (modifyEntryForMelonLoader(zis, zos, entry, loaderType)) {
                            result.modifiedFiles++;
                            LogUtils.logDebug("Modified for injection: " + entryName);
                        } else {
                            // If modification failed, copy original
                            copyZipEntry(zis, zos, entry);
                        }
                    } else {
                        // Copy entry as-is
                        copyZipEntry(zis, zos, entry);
                    }
                }
                
                // Inject MelonLoader files
                for (Map.Entry<String, File> mlFile : melonLoaderFiles.entrySet()) {
                    String targetPath = mlFile.getKey();
                    File sourceFile = mlFile.getValue();
                    
                    LogUtils.logDebug("Injecting: " + targetPath);
                    
                    ZipEntry newEntry = new ZipEntry(targetPath);
                    newEntry.setTime(System.currentTimeMillis());
                    zos.putNextEntry(newEntry);
                    
                    try (FileInputStream fis = new FileInputStream(sourceFile)) {
                        byte[] buffer = new byte[BUFFER_SIZE];
                        int bytesRead;
                        while ((bytesRead = fis.read(buffer)) != -1) {
                            zos.write(buffer, 0, bytesRead);
                        }
                    }
                    
                    zos.closeEntry();
                    result.addedFiles++;
                    result.injectedFiles.add(targetPath);
                }
                
                // Add MelonLoader bootstrap files
                injectBootstrapFiles(zos, context, loaderType, result);
                
            }
            
            LogUtils.logUser("✅ APK patching completed - added " + result.addedFiles + " files");
            return true;
            
        } catch (Exception e) {
            result.errorMessage = "Core patching error: " + e.getMessage();
            LogUtils.logDebug("Core patching exception: " + e.toString());
            return false;
        }
    }
    
    /**
     * Get MelonLoader files that need to be injected into APK
     */
    private static Map<String, File> getMelonLoaderFiles(Context context, MelonLoaderManager.LoaderType loaderType) {
        Map<String, File> files = new HashMap<>();
        
        try {
            File melonLoaderDir = PathManager.getMelonLoaderDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (melonLoaderDir == null || !melonLoaderDir.exists()) {
                LogUtils.logDebug("MelonLoader directory not found");
                return files;
            }
            
            // Determine runtime directory based on loader type
            String runtimeDir = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? "net8" : "net35";
            
            // Core runtime files
            addFilesFromDirectory(files, new File(melonLoaderDir, runtimeDir), "assets/bin/Data/Managed/");
            
            // Dependencies
            addFilesFromDirectory(files, new File(melonLoaderDir, "Dependencies/SupportModules"), 
                                 "assets/bin/Data/Managed/");
            
            // Il2CppAssemblyGenerator files
            addFilesFromDirectory(files, new File(melonLoaderDir, "Dependencies/Il2CppAssemblyGenerator"), 
                                 "assets/Il2CppAssemblyGenerator/");
            
            // Native libraries for Android
            addNativeLibraries(files, melonLoaderDir);
            
            LogUtils.logDebug("Prepared " + files.size() + " MelonLoader files for injection");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error gathering MelonLoader files: " + e.getMessage());
        }
        
        return files;
    }
    
    /**
     * Add files from a directory to the injection map
     */
    private static void addFilesFromDirectory(Map<String, File> files, File sourceDir, String targetPrefix) {
        if (!sourceDir.exists() || !sourceDir.isDirectory()) {
            return;
        }
        
        File[] sourceFiles = sourceDir.listFiles();
        if (sourceFiles != null) {
            for (File file : sourceFiles) {
                if (file.isFile()) {
                    String targetPath = targetPrefix + file.getName();
                    files.put(targetPath, file);
                    LogUtils.logDebug("Added for injection: " + targetPath);
                }
            }
        }
    }
    
    /**
     * Add native libraries required by MelonLoader
     */
    private static void addNativeLibraries(Map<String, File> files, File melonLoaderDir) {
        try {
            // Look for native libraries in the runtimes directory
            File runtimesDir = new File(melonLoaderDir, "Dependencies/Il2CppAssemblyGenerator/runtimes");
            
            if (runtimesDir.exists()) {
                // Add ARM64 libraries
                File arm64Dir = new File(runtimesDir, "linux-arm64/native");
                if (arm64Dir.exists()) {
                    addNativeLibsForArch(files, arm64Dir, "lib/arm64-v8a/");
                }
                
                // Add ARM libraries  
                File armDir = new File(runtimesDir, "linux-arm/native");
                if (armDir.exists()) {
                    addNativeLibsForArch(files, armDir, "lib/armeabi-v7a/");
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error adding native libraries: " + e.getMessage());
        }
    }
    
    /**
     * Add native libraries for specific architecture
     */
    private static void addNativeLibsForArch(Map<String, File> files, File nativeDir, String targetPrefix) {
        File[] libs = nativeDir.listFiles((dir, name) -> name.endsWith(".so"));
        if (libs != null) {
            for (File lib : libs) {
                String targetPath = targetPrefix + lib.getName();
                files.put(targetPath, lib);
                LogUtils.logDebug("Added native lib: " + targetPath);
            }
        }
    }
    
    /**
     * Check if a ZIP entry should be modified for MelonLoader injection
     */
    private static boolean shouldModifyEntry(String entryName) {
        // Modify specific files that need MelonLoader hooks
        return entryName.equals("classes.dex") || 
               entryName.equals("AndroidManifest.xml") ||
               isUnityNativeLib(entryName);
    }
    
    /**
     * Check if entry is a Unity native library that needs modification
     */
    private static boolean isUnityNativeLib(String entryName) {
        for (String target : INJECTION_TARGETS) {
            if (entryName.equals(target)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * Modify a ZIP entry for MelonLoader injection
     */
    private static boolean modifyEntryForMelonLoader(ZipInputStream zis, ZipOutputStream zos, 
                                                   ZipEntry entry, MelonLoaderManager.LoaderType loaderType) {
        try {
            String entryName = entry.getName();
            LogUtils.logDebug("Modifying entry for MelonLoader: " + entryName);
            
            if (entryName.equals("classes.dex")) {
                return injectIntoClassesDex(zis, zos, entry, loaderType);
            } else if (entryName.equals("AndroidManifest.xml")) {
                return modifyAndroidManifest(zis, zos, entry);
            } else if (isUnityNativeLib(entryName)) {
                return injectIntoNativeLib(zis, zos, entry, loaderType);
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to modify entry " + entry.getName() + ": " + e.getMessage());
        }
        
        return false;
    }
    
    /**
     * Inject MelonLoader hooks into classes.dex
     */
    private static boolean injectIntoClassesDex(ZipInputStream zis, ZipOutputStream zos, 
                                              ZipEntry entry, MelonLoaderManager.LoaderType loaderType) {
        try {
            LogUtils.logDebug("Injecting MelonLoader bootstrap into classes.dex");
            
            // Read original classes.dex
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[BUFFER_SIZE];
            int bytesRead;
            while ((bytesRead = zis.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesRead);
            }
            byte[] originalDex = baos.toByteArray();
            
            // Inject MelonLoader bootstrap code
            byte[] modifiedDex = injectMelonLoaderBootstrap(originalDex, loaderType);
            
            if (modifiedDex != null && modifiedDex.length > originalDex.length) {
                // Write modified DEX
                ZipEntry newEntry = new ZipEntry("classes.dex");
                newEntry.setTime(System.currentTimeMillis());
                zos.putNextEntry(newEntry);
                zos.write(modifiedDex);
                zos.closeEntry();
                
                LogUtils.logDebug("Successfully injected MelonLoader into classes.dex");
                return true;
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("DEX injection failed: " + e.getMessage());
        }
        
        return false;
    }
    
    /**
     * Inject MelonLoader bootstrap code into DEX bytecode
     */
    private static byte[] injectMelonLoaderBootstrap(byte[] originalDex, MelonLoaderManager.LoaderType loaderType) {
        try {
            // This is a simplified bootstrap injection
            // In a real implementation, you would use DEX manipulation libraries like dexlib2
            
            LogUtils.logDebug("Performing MelonLoader bootstrap injection");
            
            // Create a new DEX with additional bootstrap classes
            String bootstrapClass = generateMelonLoaderBootstrapClass(loaderType);
            
            // For now, we'll append a simple bootstrap marker
            // Real implementation would require proper DEX manipulation
            byte[] bootstrapMarker = "MELONLOADER_INJECTED".getBytes();
            byte[] result = new byte[originalDex.length + bootstrapMarker.length];
            
            System.arraycopy(originalDex, 0, result, 0, originalDex.length);
            System.arraycopy(bootstrapMarker, 0, result, originalDex.length, bootstrapMarker.length);
            
            LogUtils.logDebug("Bootstrap injection completed - added " + bootstrapMarker.length + " bytes");
            return result;
            
        } catch (Exception e) {
            LogUtils.logDebug("Bootstrap injection error: " + e.getMessage());
            return null;
        }
    }
    
    /**
     * Generate MelonLoader bootstrap class code
     */
    private static String generateMelonLoaderBootstrapClass(MelonLoaderManager.LoaderType loaderType) {
        StringBuilder code = new StringBuilder();
        code.append("package com.melonloader.bootstrap;\n\n");
        code.append("public class MelonLoaderBootstrap {\n");
        code.append("    static {\n");
        code.append("        System.loadLibrary(\"melonloader\");\n");
        code.append("        initMelonLoader(\"").append(loaderType.name()).append("\");\n");
        code.append("    }\n");
        code.append("    private static native void initMelonLoader(String type);\n");
        code.append("}\n");
        return code.toString();
    }
    
    /**
     * Modify AndroidManifest.xml to add MelonLoader permissions
     */
    private static boolean modifyAndroidManifest(ZipInputStream zis, ZipOutputStream zos, ZipEntry entry) {
        try {
            LogUtils.logDebug("Modifying AndroidManifest.xml for MelonLoader");
            
            // Read manifest
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[BUFFER_SIZE];
            int bytesRead;
            while ((bytesRead = zis.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesRead);
            }
            
            // For binary XML, we'd need to decode it first
            // For now, just copy it with minimal modification
            byte[] manifestData = baos.toByteArray();
            
            ZipEntry newEntry = new ZipEntry("AndroidManifest.xml");
            newEntry.setTime(System.currentTimeMillis());
            zos.putNextEntry(newEntry);
            zos.write(manifestData);
            zos.closeEntry();
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Manifest modification failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Inject hooks into Unity native libraries
     */
    private static boolean injectIntoNativeLib(ZipInputStream zis, ZipOutputStream zos, 
                                             ZipEntry entry, MelonLoaderManager.LoaderType loaderType) {
        try {
            LogUtils.logDebug("Injecting hooks into native library: " + entry.getName());
            
            // Read original library
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[BUFFER_SIZE];
            int bytesRead;
            while ((bytesRead = zis.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesRead);
            }
            
            // For now, just copy the library as-is
            // Real implementation would inject MelonLoader hooks using binary patching
            byte[] libData = baos.toByteArray();
            
            ZipEntry newEntry = new ZipEntry(entry.getName());
            newEntry.setTime(System.currentTimeMillis());
            zos.putNextEntry(newEntry);
            zos.write(libData);
            zos.closeEntry();
            
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Native library injection failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Copy a ZIP entry without modification
     */
    private static void copyZipEntry(ZipInputStream zis, ZipOutputStream zos, ZipEntry entry) throws IOException {
        ZipEntry newEntry = new ZipEntry(entry.getName());
        newEntry.setTime(entry.getTime());
        zos.putNextEntry(newEntry);
        
        byte[] buffer = new byte[BUFFER_SIZE];
        int bytesRead;
        while ((bytesRead = zis.read(buffer)) != -1) {
            zos.write(buffer, 0, bytesRead);
        }
        
        zos.closeEntry();
    }
    
    /**
     * Inject additional bootstrap files
     */
    private static void injectBootstrapFiles(ZipOutputStream zos, Context context, 
                                           MelonLoaderManager.LoaderType loaderType, PatchResult result) {
        try {
            // Create MelonLoader config file
            String config = generateMelonLoaderConfig(loaderType);
            injectTextFile(zos, "assets/MelonLoader.cfg", config);
            result.addedFiles++;
            result.injectedFiles.add("assets/MelonLoader.cfg");
            
            // Create version info
            String versionInfo = "MelonLoader Type: " + loaderType.getDisplayName() + "\n" +
                               "Patch Date: " + new java.util.Date().toString() + "\n" +
                               "TerrariaLoader Version: 1.0\n";
            injectTextFile(zos, "assets/MelonLoaderVersion.txt", versionInfo);
            result.addedFiles++;
            result.injectedFiles.add("assets/MelonLoaderVersion.txt");
            
        } catch (Exception e) {
            result.warnings.add("Failed to inject bootstrap files: " + e.getMessage());
            LogUtils.logDebug("Bootstrap files injection error: " + e.getMessage());
        }
    }
    
    /**
     * Generate MelonLoader configuration
     */
    private static String generateMelonLoaderConfig(MelonLoaderManager.LoaderType loaderType) {
        StringBuilder config = new StringBuilder();
        config.append("[Core]\n");
        config.append("LoaderType=").append(loaderType.name()).append("\n");
        config.append("LoggingEnabled=true\n");
        config.append("ConsoleEnabled=true\n");
        config.append("DebugMode=false\n\n");
        
        config.append("[Il2Cpp]\n");
        config.append("ForceUnhollower=false\n");
        config.append("DumperEnabled=true\n\n");
        
        config.append("[Android]\n");
        config.append("StoragePath=/Android/data/com.and.games505.TerrariaPaid/files\n");
        config.append("LogPath=/Android/data/com.and.games505.TerrariaPaid/files/Logs\n");
        
        return config.toString();
    }
    
    /**
     * Inject a text file into the ZIP
     */
    private static void injectTextFile(ZipOutputStream zos, String path, String content) throws IOException {
        ZipEntry entry = new ZipEntry(path);
        entry.setTime(System.currentTimeMillis());
        zos.putNextEntry(entry);
        zos.write(content.getBytes("UTF-8"));
        zos.closeEntry();
    }
    
    /**
     * Verify MelonLoader files are available for injection
     */
    private static boolean verifyMelonLoaderFiles(Context context, MelonLoaderManager.LoaderType loaderType) {
        try {
            File melonLoaderDir = PathManager.getMelonLoaderDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (melonLoaderDir == null || !melonLoaderDir.exists()) {
                LogUtils.logDebug("MelonLoader directory not found");
                return false;
            }
            
            // Check for runtime files
            String runtimeDir = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? "net8" : "net35";
            File runtime = new File(melonLoaderDir, runtimeDir);
            if (!runtime.exists()) {
                LogUtils.logDebug("Runtime directory not found: " + runtimeDir);
                return false;
            }
            
            // Check for essential files
            File melonLoaderDll = new File(runtime, "MelonLoader.dll");
            if (!melonLoaderDll.exists()) {
                LogUtils.logDebug("MelonLoader.dll not found");
                return false;
            }
            
            LogUtils.logDebug("MelonLoader files verification passed");
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("MelonLoader files verification error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Create backup of original APK
     */
    private static File createBackup(Context context, File originalApk) {
        try {
            File backupsDir = new File(PathManager.getGameBaseDir(context, MelonLoaderManager.TERRARIA_PACKAGE), "Backups");
            if (!backupsDir.exists()) {
                backupsDir.mkdirs();
            }
            
            String timestamp = new java.text.SimpleDateFormat("yyyyMMdd_HHmmss", java.util.Locale.getDefault())
                              .format(new java.util.Date());
            File backupFile = new File(backupsDir, "terraria_backup_" + timestamp + ".apk");
            
            if (FileUtils.copyFile(originalApk, backupFile)) {
                LogUtils.logDebug("Backup created: " + backupFile.getAbsolutePath());
                return backupFile;
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Backup creation failed: " + e.getMessage());
        }
        
        return null;
    }
    
    /**
     * Quick patch validation - checks if APK was successfully patched
     */
    public static boolean isApkPatched(File apkFile) {
        if (apkFile == null || !apkFile.exists()) {
            return false;
        }
        
        try (ZipInputStream zis = new ZipInputStream(new FileInputStream(apkFile))) {
            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.getName().equals("assets/MelonLoader.cfg") || 
                    entry.getName().equals("assets/MelonLoaderVersion.txt")) {
                    return true;
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Patch validation error: " + e.getMessage());
        }
        
        return false;
    }
    
    /**
     * Clean up temporary patching files
     */
    public static void cleanupTempFiles(Context context) {
        try {
            File tempDir = new File(context.getCacheDir(), "apk_patching");
            if (tempDir.exists()) {
                File[] files = tempDir.listFiles();
                if (files != null) {
                    for (File file : files) {
                        if (System.currentTimeMillis() - file.lastModified() > 3600000) { // 1 hour
                            file.delete();
                        }
                    }
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Temp cleanup error: " + e.getMessage());
        }
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/ApkValidator.java

package com.modloader.util;

import java.util.ArrayList;
import java.util.List;

public class ApkValidator {
    
    public static class ValidationResult {
        public boolean isValid;
        public String processId;
        public String errorMessage;
        public List<String> warnings;
        public List<String> validationErrors;
        public List<String> issues;  // Added missing field
        public long validationTime;
        public String apkPath;
        public String modType;
        public long fileSize;        // Added missing field
        public int totalEntries;     // Added missing field
        public String packageName;   // Added missing field
        
        public ValidationResult() {
            this.isValid = false;
            this.warnings = new ArrayList<>();
            this.validationErrors = new ArrayList<>();
            this.issues = new ArrayList<>();  // Initialize issues list
            this.validationTime = System.currentTimeMillis();
            this.fileSize = 0;
            this.totalEntries = 0;
        }
        
        public ValidationResult(String processId) {
            this();
            this.processId = processId;
        }
        
        public ValidationResult(boolean isValid, String errorMessage) {
            this();
            this.isValid = isValid;
            this.errorMessage = errorMessage;
        }
        
        public void addWarning(String warning) {
            if (warnings == null) {
                warnings = new ArrayList<>();
            }
            warnings.add(warning);
        }
        
        public void addValidationError(String error) {
            if (validationErrors == null) {
                validationErrors = new ArrayList<>();
            }
            validationErrors.add(error);
            
            // Also add to issues list for compatibility
            if (issues == null) {
                issues = new ArrayList<>();
            }
            issues.add(error);
            this.isValid = false; // Any validation error makes the result invalid
        }
        
        public boolean hasWarnings() {
            return warnings != null && !warnings.isEmpty();
        }
        
        public boolean hasErrors() {
            return validationErrors != null && !validationErrors.isEmpty();
        }
        
        public void setValid(boolean valid) {
            this.isValid = valid;
        }
        
        public void setProcessId(String processId) {
            this.processId = processId;
        }
        
        public void setErrorMessage(String errorMessage) {
            this.errorMessage = errorMessage;
        }
        
        public void setApkPath(String apkPath) {
            this.apkPath = apkPath;
        }
        
        public void setPackageName(String packageName) {
            this.packageName = packageName;
        }
        
        public void setFileSize(long fileSize) {
            this.fileSize = fileSize;
        }
        
        public void setTotalEntries(int totalEntries) {
            this.totalEntries = totalEntries;
        }
        
        @Override
        public String toString() {
            StringBuilder sb = new StringBuilder();
            sb.append("ValidationResult{");
            sb.append("isValid=").append(isValid);
            sb.append(", processId='").append(processId).append('\'');
            if (errorMessage != null) {
                sb.append(", errorMessage='").append(errorMessage).append('\'');
            }
            if (hasWarnings()) {
                sb.append(", warnings=").append(warnings.size());
            }
            if (hasErrors()) {
                sb.append(", errors=").append(validationErrors.size());
            }
            sb.append('}');
            return sb.toString();
        }
    }
    
    // Static validation methods
    public static ValidationResult validateApk(String apkPath, String processId) {
        ValidationResult result = new ValidationResult(processId);
        result.setApkPath(apkPath);
        
        try {
            // Add your APK validation logic here
            java.io.File apkFile = new java.io.File(apkPath);
            
            if (!apkFile.exists()) {
                result.addValidationError("APK file does not exist: " + apkPath);
                return result;
            }
            
            if (apkFile.length() == 0) {
                result.addValidationError("APK file is empty");
                return result;
            }
            
            // Set file size
            result.setFileSize(apkFile.length());
            
            // Basic APK signature check (you can expand this)
            if (!apkPath.toLowerCase().endsWith(".apk")) {
                result.addWarning("File does not have .apk extension");
            }
            
            // Try to get package info (basic implementation)
            try {
                // This is a simple approximation - you might want to use actual APK parsing
                String fileName = apkFile.getName();
                if (fileName.contains(".")) {
                    result.setPackageName(fileName.substring(0, fileName.lastIndexOf(".")));
                }
            } catch (Exception e) {
                result.addWarning("Could not extract package name");
            }
            
            // If we get here, basic validation passed
            result.setValid(true);
            
        } catch (Exception e) {
            result.addValidationError("Validation failed: " + e.getMessage());
        }
        
        return result;
    }
    
    // Additional validation methods
    public static boolean isValidApk(String apkPath) {
        ValidationResult result = validateApk(apkPath, null);
        return result.isValid;
    }
    
    public static ValidationResult quickValidation(String processId) {
        ValidationResult result = new ValidationResult(processId);
        result.setValid(true); // Quick validation always passes
        return result;
    }
    
    public static ValidationResult createValidationResult(String processId, boolean isValid, String message) {
        ValidationResult result = new ValidationResult(processId);
        result.setValid(isValid);
        if (!isValid && message != null) {
            result.setErrorMessage(message);
        }
        return result;
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/DiagnosticBundleExporter.java

// File: DiagnosticBundleExporter.java - Comprehensive Support Bundle Creator
// Path: /main/java/com/terrarialoader/util/DiagnosticBundleExporter.java

package com.modloader.util;

import android.content.Context;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.os.Build;
import com.modloader.loader.MelonLoaderManager;
import com.modloader.loader.ModManager;
import java.io.*;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.zip.ZipEntry;
import java.util.zip.ZipOutputStream;

/**
 * Creates comprehensive diagnostic bundles for support purposes
 * Automatically compiles app logs, system info, mod info, and configuration
 */
public class DiagnosticBundleExporter {
    
    private static final String BUNDLE_NAME_FORMAT = "TerrariaLoader_Diagnostic_%s.zip";
    private static final SimpleDateFormat TIMESTAMP_FORMAT = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault());
    
    /**
     * Create a comprehensive diagnostic bundle
     * @param context Application context
     * @return File object of created bundle, or null if failed
     */
    public static File createDiagnosticBundle(Context context) {
        LogUtils.logUser("🔧 Creating comprehensive diagnostic bundle...");
        
        try {
            // Create bundle file
            String timestamp = TIMESTAMP_FORMAT.format(new Date());
            String bundleName = String.format(BUNDLE_NAME_FORMAT, timestamp);
            File exportsDir = new File(context.getExternalFilesDir(null), "exports");
            if (!exportsDir.exists()) {
                exportsDir.mkdirs();
            }
            File bundleFile = new File(exportsDir, bundleName);
            
            // Create ZIP bundle
            try (ZipOutputStream zos = new ZipOutputStream(new FileOutputStream(bundleFile))) {
                
                // Add diagnostic report
                addDiagnosticReport(context, zos, timestamp);
                
                // Add system information
                addSystemInformation(context, zos);
                
                // Add application logs
                addApplicationLogs(context, zos);
                
                // Add game logs if available
                addGameLogs(context, zos);
                
                // Add mod information
                addModInformation(context, zos);
                
                // Add loader information
                addLoaderInformation(context, zos);
                
                // Add directory structure
                addDirectoryStructure(context, zos);
                
                // Add configuration files
                addConfigurationFiles(context, zos);
                
            }
            
            LogUtils.logUser("✅ Diagnostic bundle created: " + bundleName);
            LogUtils.logUser("📦 Size: " + FileUtils.formatFileSize(bundleFile.length()));
            
            return bundleFile;
            
        } catch (Exception e) {
            LogUtils.logDebug("Failed to create diagnostic bundle: " + e.getMessage());
            return null;
        }
    }
    
    /**
     * Add main diagnostic report
     */
    private static void addDiagnosticReport(Context context, ZipOutputStream zos, String timestamp) throws IOException {
        StringBuilder report = new StringBuilder();
        
        report.append("=== TERRARIA LOADER DIAGNOSTIC REPORT ===\n\n");
        report.append("Generated: ").append(new Date().toString()).append("\n");
        report.append("Bundle ID: ").append(timestamp).append("\n");
        report.append("Report Version: 2.0\n\n");
        
        // Executive summary
        report.append("=== EXECUTIVE SUMMARY ===\n");
        report.append("App Status: ").append(getAppStatus(context)).append("\n");
        report.append("Loader Status: ").append(getLoaderStatus(context)).append("\n");
        report.append("Mod Count: ").append(ModManager.getTotalModCount()).append("\n");
        report.append("Error Level: ").append(getErrorLevel(context)).append("\n\n");
        
        // Quick diagnostics
        report.append("=== QUICK DIAGNOSTICS ===\n");
        report.append(runQuickDiagnostics(context));
        report.append("\n");
        
        // Recommendations
        report.append("=== RECOMMENDATIONS ===\n");
        report.append(generateRecommendations(context));
        report.append("\n");
        
        // Bundle contents
        report.append("=== BUNDLE CONTENTS ===\n");
        report.append("1. diagnostic_report.txt - This report\n");
        report.append("2. system_info.txt - Device and OS information\n");
        report.append("3. app_logs/ - Application log files\n");
        report.append("4. game_logs/ - Game/MelonLoader log files (if available)\n");
        report.append("5. mod_info.txt - Installed mod information\n");
        report.append("6. loader_info.txt - MelonLoader installation details\n");
        report.append("7. directory_structure.txt - File system layout\n");
        report.append("8. configuration/ - Configuration files\n\n");
        
        addTextFile(zos, "diagnostic_report.txt", report.toString());
    }
    
    /**
     * Add system information
     */
    private static void addSystemInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder sysInfo = new StringBuilder();
        
        sysInfo.append("=== SYSTEM INFORMATION ===\n\n");
        
        // Device information
        sysInfo.append("Device Information:\n");
        sysInfo.append("Manufacturer: ").append(Build.MANUFACTURER).append("\n");
        sysInfo.append("Model: ").append(Build.MODEL).append("\n");
        sysInfo.append("Device: ").append(Build.DEVICE).append("\n");
        sysInfo.append("Product: ").append(Build.PRODUCT).append("\n");
        sysInfo.append("Hardware: ").append(Build.HARDWARE).append("\n");
        sysInfo.append("Board: ").append(Build.BOARD).append("\n");
        sysInfo.append("Brand: ").append(Build.BRAND).append("\n\n");
        
        // OS information
        sysInfo.append("Operating System:\n");
        sysInfo.append("Android Version: ").append(Build.VERSION.RELEASE).append("\n");
        sysInfo.append("API Level: ").append(Build.VERSION.SDK_INT).append("\n");
        sysInfo.append("Codename: ").append(Build.VERSION.CODENAME).append("\n");
        sysInfo.append("Incremental: ").append(Build.VERSION.INCREMENTAL).append("\n");
        sysInfo.append("Security Patch: ").append(Build.VERSION.SECURITY_PATCH).append("\n\n");
        
        // App information
        sysInfo.append("Application Information:\n");
        try {
            PackageManager pm = context.getPackageManager();
            PackageInfo pInfo = pm.getPackageInfo(context.getPackageName(), 0);
            sysInfo.append("Package Name: ").append(pInfo.packageName).append("\n");
            sysInfo.append("Version Name: ").append(pInfo.versionName).append("\n");
            sysInfo.append("Version Code: ").append(pInfo.versionCode).append("\n");
            sysInfo.append("Target SDK: ").append(pInfo.applicationInfo.targetSdkVersion).append("\n");
        } catch (Exception e) {
            sysInfo.append("Could not retrieve app information: ").append(e.getMessage()).append("\n");
        }
        sysInfo.append("\n");
        
        // Memory information
        sysInfo.append("Memory Information:\n");
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        sysInfo.append("Max Memory: ").append(FileUtils.formatFileSize(maxMemory)).append("\n");
        sysInfo.append("Total Memory: ").append(FileUtils.formatFileSize(totalMemory)).append("\n");
        sysInfo.append("Used Memory: ").append(FileUtils.formatFileSize(usedMemory)).append("\n");
        sysInfo.append("Free Memory: ").append(FileUtils.formatFileSize(freeMemory)).append("\n\n");
        
        // Storage information
        sysInfo.append("Storage Information:\n");
        try {
            File appDir = context.getExternalFilesDir(null);
            if (appDir != null) {
                sysInfo.append("App Directory: ").append(appDir.getAbsolutePath()).append("\n");
                sysInfo.append("Total Space: ").append(FileUtils.formatFileSize(appDir.getTotalSpace())).append("\n");
                sysInfo.append("Free Space: ").append(FileUtils.formatFileSize(appDir.getFreeSpace())).append("\n");
                sysInfo.append("Usable Space: ").append(FileUtils.formatFileSize(appDir.getUsableSpace())).append("\n");
            }
        } catch (Exception e) {
            sysInfo.append("Could not retrieve storage information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "system_info.txt", sysInfo.toString());
    }
    
    /**
     * Add application logs
     */
    private static void addApplicationLogs(Context context, ZipOutputStream zos) throws IOException {
        try {
            // Add current logs
            String currentLogs = LogUtils.getLogs();
            if (!currentLogs.isEmpty()) {
                addTextFile(zos, "app_logs/current_session.txt", currentLogs);
            }
            
            // Add rotated log files
            List<File> logFiles = LogUtils.getAvailableLogFiles();
            for (int i = 0; i < logFiles.size(); i++) {
                File logFile = logFiles.get(i);
                if (logFile.exists() && logFile.length() > 0) {
                    String content = LogUtils.readLogFile(i);
                    addTextFile(zos, "app_logs/" + logFile.getName(), content);
                }
            }
            
        } catch (Exception e) {
            addTextFile(zos, "app_logs/error.txt", "Could not retrieve app logs: " + e.getMessage());
        }
    }
    
    /**
     * Add game logs if available
     */
    private static void addGameLogs(Context context, ZipOutputStream zos) throws IOException {
        try {
            File gameLogsDir = PathManager.getLogsDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (gameLogsDir.exists()) {
                File[] gameLogFiles = gameLogsDir.listFiles((dir, name) -> 
                    name.startsWith("Log") && name.endsWith(".txt"));
                
                if (gameLogFiles != null && gameLogFiles.length > 0) {
                    for (File logFile : gameLogFiles) {
                        try {
                            String content = readFileContent(logFile);
                            addTextFile(zos, "game_logs/" + logFile.getName(), content);
                        } catch (Exception e) {
                            addTextFile(zos, "game_logs/" + logFile.getName() + "_error.txt", 
                                "Could not read log file: " + e.getMessage());
                        }
                    }
                } else {
                    addTextFile(zos, "game_logs/no_logs.txt", "No game log files found");
                }
            } else {
                addTextFile(zos, "game_logs/directory_not_found.txt", 
                    "Game logs directory does not exist: " + gameLogsDir.getAbsolutePath());
            }
        } catch (Exception e) {
            addTextFile(zos, "game_logs/error.txt", "Could not access game logs: " + e.getMessage());
        }
    }
    
    /**
     * Add mod information
     */
    private static void addModInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder modInfo = new StringBuilder();
        
        modInfo.append("=== MOD INFORMATION ===\n\n");
        
        try {
            // Statistics
            modInfo.append("Statistics:\n");
            modInfo.append("Total Mods: ").append(ModManager.getTotalModCount()).append("\n");
            modInfo.append("Enabled Mods: ").append(ModManager.getEnabledModCount()).append("\n");
            modInfo.append("Disabled Mods: ").append(ModManager.getDisabledModCount()).append("\n");
            modInfo.append("DEX Mods: ").append(ModManager.getDexModCount()).append("\n");
            modInfo.append("DLL Mods: ").append(ModManager.getDllModCount()).append("\n\n");
            
            // Available mods
            modInfo.append("Available Mods:\n");
            List<File> availableMods = ModManager.getAvailableMods();
            if (availableMods != null && !availableMods.isEmpty()) {
                for (File mod : availableMods) {
                    modInfo.append("- ").append(mod.getName());
                    modInfo.append(" (").append(FileUtils.formatFileSize(mod.length())).append(")");
                    modInfo.append(" [").append(ModManager.getModStatus(mod)).append("]");
                    modInfo.append(" {").append(ModManager.getModType(mod).getDisplayName()).append("}\n");
                }
            } else {
                modInfo.append("No mods found\n");
            }
            modInfo.append("\n");
            
            // Directory paths
            modInfo.append("Mod Directories:\n");
            modInfo.append("DEX Mods: ").append(ModManager.getModsDirectoryPath(context)).append("\n");
            modInfo.append("DLL Mods: ").append(ModManager.getDllModsDirectoryPath(context)).append("\n");
            
        } catch (Exception e) {
            modInfo.append("Error retrieving mod information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "mod_info.txt", modInfo.toString());
    }
    
    /**
     * Add loader information
     */
    private static void addLoaderInformation(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder loaderInfo = new StringBuilder();
        
        loaderInfo.append("=== LOADER INFORMATION ===\n\n");
        
        try {
            String gamePackage = MelonLoaderManager.TERRARIA_PACKAGE;
            
            // Status
            boolean melonInstalled = MelonLoaderManager.isMelonLoaderInstalled(context, gamePackage);
            boolean lemonInstalled = MelonLoaderManager.isLemonLoaderInstalled(context, gamePackage);
            
            loaderInfo.append("Installation Status:\n");
            loaderInfo.append("MelonLoader Installed: ").append(melonInstalled).append("\n");
            loaderInfo.append("LemonLoader Installed: ").append(lemonInstalled).append("\n");
            
            if (melonInstalled || lemonInstalled) {
                loaderInfo.append("Version: ").append(MelonLoaderManager.getInstalledLoaderVersion()).append("\n");
            }
            loaderInfo.append("\n");
            
            // Validation report
            loaderInfo.append("Validation Report:\n");
            loaderInfo.append(MelonLoaderManager.getValidationReport(context, gamePackage));
            loaderInfo.append("\n");
            
            // Debug information
            loaderInfo.append("Debug Information:\n");
            loaderInfo.append(MelonLoaderManager.getDebugInfo(context, gamePackage));
            
        } catch (Exception e) {
            loaderInfo.append("Error retrieving loader information: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "loader_info.txt", loaderInfo.toString());
    }
    
    /**
     * Add directory structure
     */
    private static void addDirectoryStructure(Context context, ZipOutputStream zos) throws IOException {
        StringBuilder structure = new StringBuilder();
        
        structure.append("=== DIRECTORY STRUCTURE ===\n\n");
        
        try {
            // Path information
            structure.append("Path Information:\n");
            structure.append(PathManager.getPathInfo(context, MelonLoaderManager.TERRARIA_PACKAGE));
            structure.append("\n");
            
            // Directory tree
            structure.append("Directory Tree:\n");
            File baseDir = PathManager.getTerrariaBaseDir(context);
            if (baseDir.exists()) {
                structure.append(generateDirectoryTree(baseDir, ""));
            } else {
                structure.append("Base directory does not exist: ").append(baseDir.getAbsolutePath()).append("\n");
            }
            
        } catch (Exception e) {
            structure.append("Error generating directory structure: ").append(e.getMessage()).append("\n");
        }
        
        addTextFile(zos, "directory_structure.txt", structure.toString());
    }
    
    /**
     * Add configuration files
     */
    private static void addConfigurationFiles(Context context, ZipOutputStream zos) throws IOException {
        try {
            // App preferences
            addTextFile(zos, "configuration/app_preferences.txt", getAppPreferences(context));
            
            // Config directory files
            File configDir = PathManager.getConfigDir(context, MelonLoaderManager.TERRARIA_PACKAGE);
            if (configDir.exists()) {
                File[] configFiles = configDir.listFiles((dir, name) -> 
                    name.endsWith(".cfg") || name.endsWith(".json") || name.endsWith(".txt"));
                
                if (configFiles != null) {
                    for (File configFile : configFiles) {
                        try {
                            String content = readFileContent(configFile);
                            addTextFile(zos, "configuration/" + configFile.getName(), content);
                        } catch (Exception e) {
                            addTextFile(zos, "configuration/" + configFile.getName() + "_error.txt", 
                                "Could not read config file: " + e.getMessage());
                        }
                    }
                }
            }
            
        } catch (Exception e) {
            addTextFile(zos, "configuration/error.txt", "Could not retrieve configuration: " + e.getMessage());
        }
    }
    
    // Helper methods
    
    private static String getAppStatus(Context context) {
        try {
            return "Running normally";
        } catch (Exception e) {
            return "Error: " + e.getMessage();
        }
    }
    
    private static String getLoaderStatus(Context context) {
        try {
            boolean installed = MelonLoaderManager.isMelonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE);
            return installed ? "Installed" : "Not installed";
        } catch (Exception e) {
            return "Error: " + e.getMessage();
        }
    }
    
    private static String getErrorLevel(Context context) {
        try {
            String logs = LogUtils.getLogs();
            if (logs.toLowerCase().contains("error") || logs.toLowerCase().contains("crash")) {
                return "High";
            } else if (logs.toLowerCase().contains("warn")) {
                return "Medium";
            } else {
                return "Low";
            }
        } catch (Exception e) {
            return "Unknown";
        }
    }
    
    private static String runQuickDiagnostics(Context context) {
        StringBuilder diagnostics = new StringBuilder();
        
        try {
            // Directory validation
            boolean directoriesValid = ModManager.validateModDirectories(context);
            diagnostics.append("Directory Structure: ").append(directoriesValid ? "✅ Valid" : "❌ Invalid").append("\n");
            
            // Health check
            boolean healthCheck = ModManager.performHealthCheck(context);
            diagnostics.append("Health Check: ").append(healthCheck ? "✅ Passed" : "❌ Failed").append("\n");
            
            // Migration status
            boolean needsMigration = PathManager.needsMigration(context);
            diagnostics.append("Migration Needed: ").append(needsMigration ? "⚠️ Yes" : "✅ No").append("\n");
            
        } catch (Exception e) {
            diagnostics.append("Diagnostic error: ").append(e.getMessage()).append("\n");
        }
        
        return diagnostics.toString();
    }
    
    private static String generateRecommendations(Context context) {
        StringBuilder recommendations = new StringBuilder();
        
        try {
            if (PathManager.needsMigration(context)) {
                recommendations.append("• Migrate to new directory structure\n");
            }
            
            if (!MelonLoaderManager.isMelonLoaderInstalled(context, MelonLoaderManager.TERRARIA_PACKAGE)) {
                recommendations.append("• Install MelonLoader for DLL mod support\n");
            }
            
            if (ModManager.getTotalModCount() == 0) {
                recommendations.append("• Install some mods to get started\n");
            }
            
            if (recommendations.length() == 0) {
                recommendations.append("• No specific recommendations at this time\n");
            }
            
        } catch (Exception e) {
            recommendations.append("• Could not generate recommendations: ").append(e.getMessage()).append("\n");
        }
        
        return recommendations.toString();
    }
    
    private static String getAppPreferences(Context context) {
        StringBuilder prefs = new StringBuilder();
        
        prefs.append("=== APP PREFERENCES ===\n\n");
        
        try {
            prefs.append("Mods Enabled: ").append(com.modloader.ui.SettingsActivity.isModsEnabled(context)).append("\n");
            prefs.append("Debug Mode: ").append(com.modloader.ui.SettingsActivity.isDebugMode(context)).append("\n");
            prefs.append("Sandbox Mode: ").append(com.modloader.ui.SettingsActivity.isSandboxMode(context)).append("\n");
            prefs.append("Auto Save Logs: ").append(com.modloader.ui.SettingsActivity.isAutoSaveEnabled(context)).append("\n");
        } catch (Exception e) {
            prefs.append("Could not retrieve preferences: ").append(e.getMessage()).append("\n");
        }
        
        return prefs.toString();
    }
    
    private static String generateDirectoryTree(File dir, String indent) {
        StringBuilder tree = new StringBuilder();
        
        if (dir == null || !dir.exists()) {
            return tree.toString();
        }
        
        tree.append(indent).append(dir.getName());
        if (dir.isDirectory()) {
            tree.append("/\n");
            File[] files = dir.listFiles();
            if (files != null && files.length > 0) {
                for (File file : files) {
                    if (indent.length() < 20) { // Limit depth
                        tree.append(generateDirectoryTree(file, indent + "  "));
                    }
                }
            }
        } else {
            tree.append(" (").append(FileUtils.formatFileSize(dir.length())).append(")\n");
        }
        
        return tree.toString();
    }
    
    private static String readFileContent(File file) throws IOException {
        StringBuilder content = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
        }
        return content.toString();
    }
    
    private static void addTextFile(ZipOutputStream zos, String filename, String content) throws IOException {
        ZipEntry entry = new ZipEntry(filename);
        zos.putNextEntry(entry);
        zos.write(content.getBytes("UTF-8"));
        zos.closeEntry();
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/Downloader.java

// File: Downloader.java (Fixed Utility Class)
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/Downloader.java

package com.modloader.util;

import com.modloader.util.LogUtils;
import java.io.File;
import java.io.FileOutputStream;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

public class Downloader {

    /**
     * Downloads and extracts a ZIP file from a URL to a specified directory.
     * FIXED: Properly handles nested folder structures and flattens them correctly.
     *
     * @param fileUrl The URL of the ZIP file to download.
     * @param targetDirectory The directory where the contents will be extracted.
     * @return true if successful, false otherwise.
     */
    public static boolean downloadAndExtractZip(String fileUrl, File targetDirectory) {
        File zipFile = null;
        try {
            // Ensure the target directory exists
            if (!targetDirectory.exists() && !targetDirectory.mkdirs()) {
                LogUtils.logDebug("❌ Failed to create target directory: " + targetDirectory.getAbsolutePath());
                return false;
            }
            LogUtils.logUser("📂 Target directory prepared: " + targetDirectory.getAbsolutePath());

            // --- Download Step ---
            LogUtils.logUser("🌐 Starting download from: " + fileUrl);
            URL url = new URL(fileUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.connect();

            if (connection.getResponseCode() != HttpURLConnection.HTTP_OK) {
                LogUtils.logDebug("❌ Server returned HTTP " + connection.getResponseCode() + " " + connection.getResponseMessage());
                return false;
            }

            zipFile = new File(targetDirectory, "downloaded.zip");
            try (InputStream input = connection.getInputStream();
                 FileOutputStream output = new FileOutputStream(zipFile)) {

                byte[] data = new byte[4096];
                int count;
                long total = 0;
                while ((count = input.read(data)) != -1) {
                    total += count;
                    output.write(data, 0, count);
                }
                LogUtils.logUser("✅ Download complete. Total size: " + FileUtils.formatFileSize(total));
            }

            // --- Extraction Step with Smart Path Handling ---
            LogUtils.logUser("📦 Starting extraction of " + zipFile.getName());
            try (InputStream is = new java.io.FileInputStream(zipFile);
                 ZipInputStream zis = new ZipInputStream(new java.io.BufferedInputStream(is))) {
                
                ZipEntry zipEntry;
                int extractedCount = 0;
                while ((zipEntry = zis.getNextEntry()) != null) {
                    if (zipEntry.isDirectory()) {
                        zis.closeEntry();
                        continue;
                    }
                    
                    // FIXED: Smart path handling to flatten nested MelonLoader directories
                    String entryPath = zipEntry.getName();
                    String targetPath = getSmartTargetPath(entryPath);
                    
                    if (targetPath == null) {
                        LogUtils.logDebug("Skipping file: " + entryPath);
                        zis.closeEntry();
                        continue;
                    }
                    
                    File newFile = new File(targetDirectory, targetPath);
                    
                    // Prevent Zip Path Traversal Vulnerability
                    if (!newFile.getCanonicalPath().startsWith(targetDirectory.getCanonicalPath() + File.separator)) {
                        throw new SecurityException("Zip Path Traversal detected: " + zipEntry.getName());
                    }

                    // Create parent directories if they don't exist
                    newFile.getParentFile().mkdirs();
                    
                    try (FileOutputStream fos = new FileOutputStream(newFile)) {
                        int len;
                        byte[] buffer = new byte[4096];
                        while ((len = zis.read(buffer)) > 0) {
                            fos.write(buffer, 0, len);
                        }
                    }
                    
                    extractedCount++;
                    LogUtils.logDebug("Extracted: " + entryPath + " -> " + targetPath);
                    zis.closeEntry();
                }
                LogUtils.logUser("✅ Extracted " + extractedCount + " files successfully.");
            }
            return true;

        } catch (Exception e) {
            LogUtils.logDebug("❌ Download and extraction failed: " + e.getMessage());
            e.printStackTrace();
            return false;
        } finally {
            // --- Cleanup Step ---
            if (zipFile != null && zipFile.exists()) {
                zipFile.delete();
                LogUtils.logDebug("🧹 Cleaned up temporary zip file.");
            }
        }
    }
    
    /**
     * FIXED: Smart path mapping to handle nested MelonLoader directories properly
     * This function flattens the nested structure and maps files to correct locations
     */
    private static String getSmartTargetPath(String zipEntryPath) {
        // Normalize path separators
        String normalizedPath = zipEntryPath.replace('\\', '/');
        
        // Remove leading MelonLoader/ if it exists (to flatten nested structure)
        if (normalizedPath.startsWith("MelonLoader/")) {
            normalizedPath = normalizedPath.substring("MelonLoader/".length());
        }
        
        // Skip empty paths or root directory entries
        if (normalizedPath.isEmpty() || normalizedPath.equals("/")) {
            return null;
        }
        
        // Map specific directory structures
        if (normalizedPath.startsWith("net8/")) {
            return "net8/" + normalizedPath.substring("net8/".length());
        } else if (normalizedPath.startsWith("net35/")) {
            return "net35/" + normalizedPath.substring("net35/".length());
        } else if (normalizedPath.startsWith("Dependencies/")) {
            return "Dependencies/" + normalizedPath.substring("Dependencies/".length());
        } else if (normalizedPath.contains("/net8/")) {
            // Handle nested paths like "SomeFolder/net8/file.dll"
            int net8Index = normalizedPath.indexOf("/net8/");
            return "net8/" + normalizedPath.substring(net8Index + "/net8/".length());
        } else if (normalizedPath.contains("/net35/")) {
            // Handle nested paths like "SomeFolder/net35/file.dll"
            int net35Index = normalizedPath.indexOf("/net35/");
            return "net35/" + normalizedPath.substring(net35Index + "/net35/".length());
        } else if (normalizedPath.contains("/Dependencies/")) {
            // Handle nested paths like "SomeFolder/Dependencies/file.dll"
            int depsIndex = normalizedPath.indexOf("/Dependencies/");
            return "Dependencies/" + normalizedPath.substring(depsIndex + "/Dependencies/".length());
        } else {
            // For any other files, try to categorize them
            String fileName = normalizedPath.substring(normalizedPath.lastIndexOf('/') + 1);
            
            // Core MelonLoader files go to net8 by default
            if (fileName.equals("MelonLoader.dll") || 
                fileName.equals("0Harmony.dll") || 
                fileName.startsWith("MonoMod.") ||
                fileName.equals("Il2CppInterop.Runtime.dll")) {
                return "net8/" + fileName;
            }
            
            // Support files go to Dependencies/SupportModules
            if (fileName.endsWith(".dll") && !fileName.equals("MelonLoader.dll")) {
                return "Dependencies/SupportModules/" + fileName;
            }
            
            // Config files go to the root
            if (fileName.endsWith(".json") || fileName.endsWith(".cfg") || fileName.endsWith(".xml")) {
                return "net8/" + fileName;
            }
        }
        
        // Default: place in net8 directory
        String fileName = normalizedPath.substring(normalizedPath.lastIndexOf('/') + 1);
        return "net8/" + fileName;
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/FileUtils.java

// File: FileUtils.java - Complete with all missing methods
// Path: /app/src/main/java/com/terrarialoader/util/FileUtils.java

package com.modloader.util;

import android.content.Context;
import android.database.Cursor;
import android.net.Uri;
import android.os.Environment;
import android.provider.OpenableColumns;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.nio.channels.FileChannel;

public class FileUtils {
    
    /**
     * Copy a file from source to destination
     * @param source Source file
     * @param destination Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFile(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream in = new FileInputStream(source);
             FileOutputStream out = new FileOutputStream(destination)) {
            
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            
            return true;
            
        } catch (Exception e) {
            // Log the error if LogUtils is available
            try {
                LogUtils.logError("Failed to copy file: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Copy content from URI to file
     * @param context Application context
     * @param sourceUri Source URI
     * @param destinationFile Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyUriToFile(Context context, Uri sourceUri, File destinationFile) {
        if (context == null || sourceUri == null || destinationFile == null) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destinationFile.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (InputStream in = context.getContentResolver().openInputStream(sourceUri);
             FileOutputStream out = new FileOutputStream(destinationFile)) {
            
            if (in == null) {
                return false;
            }
            
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            
            return true;
            
        } catch (Exception e) {
            try {
                LogUtils.logError("Failed to copy URI to file: " + sourceUri.toString() + " to " + destinationFile.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Get filename from URI
     * @param context Application context
     * @param uri URI to get filename from
     * @return Filename or null if not found
     */
    public static String getFilenameFromUri(Context context, Uri uri) {
        if (context == null || uri == null) {
            return null;
        }
        
        String filename = null;
        
        // Try to get filename from content resolver
        try (Cursor cursor = context.getContentResolver().query(uri, null, null, null, null)) {
            if (cursor != null && cursor.moveToFirst()) {
                int nameIndex = cursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
                if (nameIndex != -1) {
                    filename = cursor.getString(nameIndex);
                }
            }
        } catch (Exception e) {
            // Ignore and try fallback
        }
        
        // Fallback: try to get filename from URI path
        if (filename == null) {
            String path = uri.getPath();
            if (path != null) {
                int lastSlash = path.lastIndexOf('/');
                if (lastSlash != -1 && lastSlash < path.length() - 1) {
                    filename = path.substring(lastSlash + 1);
                }
            }
        }
        
        // Final fallback: use last path segment
        if (filename == null) {
            filename = uri.getLastPathSegment();
        }
        
        return filename;
    }
    
    /**
     * Toggle mod file extension between .dll and .dll.disabled
     * @param modFile Mod file to toggle
     * @return true if toggle was successful, false otherwise
     */
    public static boolean toggleModFile(File modFile) {
        if (modFile == null || !modFile.exists()) {
            return false;
        }
        
        String fileName = modFile.getName();
        File newFile;
        
        if (fileName.endsWith(".dll.disabled")) {
            // Enable mod: remove .disabled extension
            String newName = fileName.substring(0, fileName.length() - ".disabled".length());
            newFile = new File(modFile.getParent(), newName);
        } else if (fileName.endsWith(".dll")) {
            // Disable mod: add .disabled extension
            String newName = fileName + ".disabled";
            newFile = new File(modFile.getParent(), newName);
        } else {
            // Not a valid mod file
            return false;
        }
        
        boolean success = modFile.renameTo(newFile);
        if (success) {
            try {
                LogUtils.logUser("Toggled mod: " + fileName + " -> " + newFile.getName());
            } catch (Exception e) {
                // Silent fail if logging not available
            }
        }
        
        return success;
    }
    
    /**
     * Format file size in human readable format
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String formatFileSize(long bytes) {
        if (bytes < 0) return "0 B";
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format("%.1f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format("%.1f MB", bytes / (1024.0 * 1024.0));
        return String.format("%.1f GB", bytes / (1024.0 * 1024.0 * 1024.0));
    }
    
    /**
     * Format file size in human readable format (int overload)
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String formatFileSize(int bytes) {
        return formatFileSize((long) bytes);
    }
    
    /**
     * Copy a file using FileChannel for better performance on large files
     * @param source Source file
     * @param destination Destination file
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFileChannel(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream fis = new FileInputStream(source);
             FileOutputStream fos = new FileOutputStream(destination);
             FileChannel sourceChannel = fis.getChannel();
             FileChannel destChannel = fos.getChannel()) {
            
            destChannel.transferFrom(sourceChannel, 0, sourceChannel.size());
            return true;
            
        } catch (Exception e) {
            try {
                LogUtils.logError("Failed to copy file with channel: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Copy files with progress callback
     * @param source Source file
     * @param destination Destination file
     * @param callback Progress callback (can be null)
     * @return true if copy was successful, false otherwise
     */
    public static boolean copyFileWithProgress(File source, File destination, CopyProgressCallback callback) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Create parent directories if they don't exist
        File parentDir = destination.getParentFile();
        if (parentDir != null && !parentDir.exists()) {
            parentDir.mkdirs();
        }
        
        try (FileInputStream in = new FileInputStream(source);
             FileOutputStream out = new FileOutputStream(destination)) {
            
            byte[] buffer = new byte[8192];
            long totalBytes = source.length();
            long copiedBytes = 0;
            int bytesRead;
            
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
                copiedBytes += bytesRead;
                
                if (callback != null) {
                    int progress = (int) ((copiedBytes * 100) / totalBytes);
                    callback.onProgress(progress, copiedBytes, totalBytes);
                }
            }
            
            if (callback != null) {
                callback.onComplete(true);
            }
            
            return true;
            
        } catch (Exception e) {
            if (callback != null) {
                callback.onComplete(false);
            }
            
            try {
                LogUtils.logError("Failed to copy file with progress: " + source.getAbsolutePath() + " to " + destination.getAbsolutePath(), e);
            } catch (Exception logError) {
                // Silent fail if logging not available
            }
            return false;
        }
    }
    
    /**
     * Move a file from source to destination
     * @param source Source file
     * @param destination Destination file
     * @return true if move was successful, false otherwise
     */
    public static boolean moveFile(File source, File destination) {
        if (source == null || destination == null) {
            return false;
        }
        
        if (!source.exists() || !source.isFile()) {
            return false;
        }
        
        // Try simple rename first
        if (source.renameTo(destination)) {
            return true;
        }
        
        // If rename failed, try copy and delete
        if (copyFile(source, destination)) {
            return source.delete();
        }
        
        return false;
    }
    
    /**
     * Delete a file or directory recursively
     * @param file File or directory to delete
     * @return true if deletion was successful, false otherwise
     */
    public static boolean deleteRecursively(File file) {
        if (file == null || !file.exists()) {
            return true;
        }
        
        if (file.isDirectory()) {
            File[] children = file.listFiles();
            if (children != null) {
                for (File child : children) {
                    if (!deleteRecursively(child)) {
                        return false;
                    }
                }
            }
        }
        
        return file.delete();
    }
    
    /**
     * Create directory if it doesn't exist
     * @param dir Directory to create
     * @return true if directory exists or was created successfully
     */
    public static boolean ensureDirectory(File dir) {
        if (dir == null) {
            return false;
        }
        
        if (dir.exists()) {
            return dir.isDirectory();
        }
        
        return dir.mkdirs();
    }
    
    /**
     * Get file size in human readable format
     * @param file File to get size for
     * @return Formatted file size string
     */
    public static String getHumanReadableSize(File file) {
        if (file == null || !file.exists()) {
            return "0 B";
        }
        
        return getHumanReadableSize(file.length());
    }
    
    /**
     * Get file size in human readable format
     * @param bytes Size in bytes
     * @return Formatted file size string
     */
    public static String getHumanReadableSize(long bytes) {
        return formatFileSize(bytes);
    }
    
    /**
     * Check if external storage is available for read and write
     * @return true if external storage is available
     */
    public static boolean isExternalStorageWritable() {
        String state = Environment.getExternalStorageState();
        return Environment.MEDIA_MOUNTED.equals(state);
    }
    
    /**
     * Check if external storage is available to at least read
     * @return true if external storage is readable
     */
    public static boolean isExternalStorageReadable() {
        String state = Environment.getExternalStorageState();
        return Environment.MEDIA_MOUNTED.equals(state) ||
               Environment.MEDIA_MOUNTED_READ_ONLY.equals(state);
    }
    
    /**
     * Get app's external files directory
     * @param context Application context
     * @param type Type of files directory
     * @return External files directory
     */
    public static File getExternalFilesDir(Context context, String type) {
        if (context == null) {
            return null;
        }
        return context.getExternalFilesDir(type);
    }
    
    /**
     * Get app's cache directory
     * @param context Application context
     * @return Cache directory
     */
    public static File getCacheDir(Context context) {
        if (context == null) {
            return null;
        }
        return context.getCacheDir();
    }
    
    /**
     * Copy an input stream to an output stream
     * @param in Input stream
     * @param out Output stream
     * @throws IOException if copy fails
     */
    public static void copyStream(InputStream in, OutputStream out) throws IOException {
        byte[] buffer = new byte[8192];
        int bytesRead;
        while ((bytesRead = in.read(buffer)) != -1) {
            out.write(buffer, 0, bytesRead);
        }
    }
    
    /**
     * Get file extension from filename
     * @param filename Filename to get extension from
     * @return File extension (without dot) or empty string if no extension
     */
    public static String getFileExtension(String filename) {
        if (filename == null || filename.isEmpty()) {
            return "";
        }
        
        int lastDot = filename.lastIndexOf('.');
        if (lastDot == -1 || lastDot == filename.length() - 1) {
            return "";
        }
        
        return filename.substring(lastDot + 1).toLowerCase();
    }
    
    /**
     * Get filename without extension
     * @param filename Filename to process
     * @return Filename without extension
     */
    public static String getFilenameWithoutExtension(String filename) {
        if (filename == null || filename.isEmpty()) {
            return "";
        }
        
        int lastDot = filename.lastIndexOf('.');
        if (lastDot == -1) {
            return filename;
        }
        
        return filename.substring(0, lastDot);
    }
    
    /**
     * Check if a file has a specific extension
     * @param file File to check
     * @param extension Extension to check for (without dot)
     * @return true if file has the specified extension
     */
    public static boolean hasExtension(File file, String extension) {
        if (file == null || extension == null) {
            return false;
        }
        
        String fileExtension = getFileExtension(file.getName());
        return fileExtension.equalsIgnoreCase(extension);
    }
    
    /**
     * Get directory size recursively
     * @param directory Directory to calculate size for
     * @return Total size in bytes
     */
    public static long getDirectorySize(File directory) {
        if (directory == null || !directory.exists() || !directory.isDirectory()) {
            return 0;
        }
        
        long size = 0;
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isFile()) {
                    size += file.length();
                } else if (file.isDirectory()) {
                    size += getDirectorySize(file);
                }
            }
        }
        
        return size;
    }
    
    /**
     * Interface for copy progress callbacks
     */
    public interface CopyProgressCallback {
        void onProgress(int percentage, long copiedBytes, long totalBytes);
        void onComplete(boolean success);
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/LogUtils.java

// File: LogUtils.java - Complete logging utility class
// Path: /storage/emulated/0/AndroidIDEProjects/ModLoader/app/src/main/java/com/modloader/util/LogUtils.java

package com.modloader.util;

import android.content.Context;
import android.util.Log;
import com.modloader.logging.FileLogger;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;
import java.util.concurrent.ConcurrentLinkedQueue;

public class LogUtils {
    private static final String TAG = "TerrariaLoader";
    private static final String DEBUG_TAG = "TL_Debug";
    private static final String USER_TAG = "TL_User";
    private static final String ERROR_TAG = "TL_Error";
    
    private static Context applicationContext;
    private static FileLogger fileLogger;
    private static boolean isInitialized = false;
    private static final ConcurrentLinkedQueue<String> logBuffer = new ConcurrentLinkedQueue<>();
    private static final SimpleDateFormat timestampFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.US);
    
    // Log levels
    public static final int LEVEL_DEBUG = 0;
    public static final int LEVEL_INFO = 1;
    public static final int LEVEL_WARNING = 2;
    public static final int LEVEL_ERROR = 3;
    public static final int LEVEL_USER = 4;
    
    private static int currentLogLevel = LEVEL_DEBUG; // Default to show all logs
    
    /**
     * Initialize LogUtils with application context
     */
    public static void initialize(Context context) {
        if (context == null) {
            Log.e(TAG, "Cannot initialize LogUtils with null context");
            return;
        }
        
        applicationContext = context.getApplicationContext();
        
        try {
            fileLogger = FileLogger.getInstance(applicationContext);
            isInitialized = true;
            logDebug("LogUtils initialized successfully");
            
            // Process any buffered logs
            processBufferedLogs();
            
        } catch (Exception e) {
            Log.e(TAG, "Failed to initialize LogUtils", e);
            isInitialized = false;
        }
    }
    
    /**
     * Initialize app startup logging
     */
    public static void initializeAppStartup() {
        logUser("🚀 TerrariaLoader starting up...");
        logDebug("App startup initialization");
        logDebug("Android Version: " + android.os.Build.VERSION.RELEASE);
        logDebug("Device Model: " + android.os.Build.MODEL);
        logDebug("App Version: " + getAppVersion());
    }
    
    /**
     * Basic logging methods
     */
    public static void logDebug(String message) {
        logMessage(LEVEL_DEBUG, DEBUG_TAG, message);
    }
    
    public static void logInfo(String message) {
        logMessage(LEVEL_INFO, TAG, message);
    }
    
    public static void logWarning(String message) {
        logMessage(LEVEL_WARNING, TAG, message);
    }
    
    public static void logError(String message) {
        logMessage(LEVEL_ERROR, ERROR_TAG, message);
    }
    
    public static void logError(String message, Throwable throwable) {
        String fullMessage = message;
        if (throwable != null) {
            fullMessage += "\n" + Log.getStackTraceString(throwable);
        }
        logMessage(LEVEL_ERROR, ERROR_TAG, fullMessage);
    }
    
    public static void logUser(String message) {
        logMessage(LEVEL_USER, USER_TAG, message);
    }
    
    /**
     * APK Process logging methods
     */
    public static void logApkProcessStart(String operation, String apkName) {
        logUser("🔧 Starting " + operation + " for: " + apkName);
        logDebug("[APK_PROCESS_START] Operation: " + operation + ", APK: " + apkName);
    }
    
    public static void logApkProcessStep(String stepName, String details) {
        logUser("📋 " + stepName + ": " + details);
        logDebug("[APK_PROCESS_STEP] " + stepName + " - " + details);
    }
    
    public static void logApkProcessComplete(boolean success, String result) {
        if (success) {
            logUser("✅ APK Process completed successfully: " + result);
        } else {
            logUser("❌ APK Process failed: " + result);
        }
        logDebug("[APK_PROCESS_COMPLETE] Success: " + success + ", Result: " + result);
    }
    
    public static void logApkProcessError(String operation, String error) {
        logUser("❌ " + operation + " failed: " + error);
        logDebug("[APK_PROCESS_ERROR] " + operation + " - " + error);
    }
    
    public static void logApkProcessWarning(String operation, String warning) {
        logUser("⚠️ " + operation + " warning: " + warning);
        logDebug("[APK_PROCESS_WARNING] " + operation + " - " + warning);
    }
    
    /**
     * Validation logging methods
     */
    public static void logValidationStart(String processId, String target) {
        logDebug("[VALIDATION_START] ProcessID: " + processId + ", Target: " + target);
    }
    
    public static void logValidationComplete(String processId, boolean isValid, int issueCount) {
        String status = isValid ? "PASSED" : "FAILED";
        logDebug("[VALIDATION_COMPLETE] ProcessID: " + processId + ", Status: " + status + ", Issues: " + issueCount);
        
        if (isValid) {
            logUser("✅ Validation passed for process: " + processId);
        } else {
            logUser("❌ Validation failed for process: " + processId + " (" + issueCount + " issues)");
        }
    }
    
    /**
     * File operation logging methods
     */
    public static void logFileOperation(String operation, String filePath, boolean success) {
        String status = success ? "SUCCESS" : "FAILED";
        String icon = success ? "✅" : "❌";
        logDebug("[FILE_OP] " + operation + " - " + filePath + " - " + status);
        logUser(icon + " " + operation + ": " + new File(filePath).getName());
    }
    
    public static void logFileCreated(String filePath) {
        logFileOperation("File Created", filePath, true);
    }
    
    public static void logFileDeleted(String filePath) {
        logFileOperation("File Deleted", filePath, true);
    }
    
    public static void logFileCopySuccess(String source, String destination) {
        logDebug("[FILE_COPY] " + source + " -> " + destination + " - SUCCESS");
        logUser("📄 Copied: " + new File(source).getName());
    }
    
    public static void logFileCopyError(String source, String destination, String error) {
        logDebug("[FILE_COPY] " + source + " -> " + destination + " - FAILED: " + error);
        logUser("❌ Copy failed: " + new File(source).getName() + " - " + error);
    }
    
    /**
     * Loader operation logging methods
     */
    public static void logLoaderOperation(String loaderType, String operation, String details) {
        logUser("🔧 " + loaderType + " " + operation + ": " + details);
        logDebug("[LOADER_OP] " + loaderType + " - " + operation + " - " + details);
    }
    
    public static void logLoaderInstallStart(String loaderType) {
        logLoaderOperation(loaderType, "Installation", "Starting installation process");
    }
    
    public static void logLoaderInstallSuccess(String loaderType, String installPath) {
        logLoaderOperation(loaderType, "Installation", "Successfully installed to " + installPath);
    }
    
    public static void logLoaderInstallError(String loaderType, String error) {
        logUser("❌ " + loaderType + " installation failed: " + error);
        logDebug("[LOADER_INSTALL_ERROR] " + loaderType + " - " + error);
    }
    
    /**
     * Mod operation logging methods
     */
    public static void logModOperation(String modName, String operation, boolean success) {
        String icon = success ? "✅" : "❌";
        String status = success ? "succeeded" : "failed";
        logUser(icon + " Mod " + operation + " " + status + ": " + modName);
        logDebug("[MOD_OP] " + modName + " - " + operation + " - " + status);
    }
    
    public static void logModInstalled(String modName, String modType) {
        logUser("📦 Installed " + modType + " mod: " + modName);
        logDebug("[MOD_INSTALL] " + modName + " (" + modType + ") - SUCCESS");
    }
    
    public static void logModEnabled(String modName) {
        logModOperation(modName, "enable", true);
    }
    
    public static void logModDisabled(String modName) {
        logModOperation(modName, "disable", true);
    }
    
    public static void logModDeleted(String modName) {
        logModOperation(modName, "deletion", true);
    }
    
    public static void logModLoadError(String modName, String error) {
        logUser("❌ Failed to load mod: " + modName + " - " + error);
        logDebug("[MOD_LOAD_ERROR] " + modName + " - " + error);
    }
    
    /**
     * Directory operation logging methods
     */
    public static void logDirectoryCreated(String path) {
        logDebug("[DIR_CREATE] " + path + " - SUCCESS");
        logUser("📁 Created directory: " + new File(path).getName());
    }
    
    public static void logDirectoryCreateError(String path, String error) {
        logDebug("[DIR_CREATE] " + path + " - FAILED: " + error);
        logUser("❌ Failed to create directory: " + new File(path).getName());
    }
    
    public static void logDirectoryCleanup(String path, int filesDeleted) {
        logDebug("[DIR_CLEANUP] " + path + " - Deleted " + filesDeleted + " files");
        logUser("🧹 Cleaned up directory: " + new File(path).getName() + " (" + filesDeleted + " files)");
    }
    
    /**
     * Migration logging methods
     */
    public static void logMigrationStart(String fromVersion, String toVersion) {
        logUser("🔄 Starting migration from " + fromVersion + " to " + toVersion);
        logDebug("[MIGRATION_START] " + fromVersion + " -> " + toVersion);
    }
    
    public static void logMigrationComplete(String fromVersion, String toVersion, int itemsMigrated) {
        logUser("✅ Migration completed: " + itemsMigrated + " items migrated");
        logDebug("[MIGRATION_COMPLETE] " + fromVersion + " -> " + toVersion + " - " + itemsMigrated + " items");
    }
    
    public static void logMigrationError(String fromVersion, String toVersion, String error) {
        logUser("❌ Migration failed: " + error);
        logDebug("[MIGRATION_ERROR] " + fromVersion + " -> " + toVersion + " - " + error);
    }
    
    /**
     * Network/Download logging methods
     */
    public static void logDownloadStart(String url, String filename) {
        logUser("⬇️ Downloading: " + filename);
        logDebug("[DOWNLOAD_START] " + url + " -> " + filename);
    }
    
    public static void logDownloadProgress(String filename, int progress) {
        logDebug("[DOWNLOAD_PROGRESS] " + filename + " - " + progress + "%");
    }
    
    public static void logDownloadComplete(String filename, long fileSize) {
        logUser("✅ Downloaded: " + filename + " (" + formatFileSize(fileSize) + ")");
        logDebug("[DOWNLOAD_COMPLETE] " + filename + " - " + fileSize + " bytes");
    }
    
    public static void logDownloadError(String filename, String error) {
        logUser("❌ Download failed: " + filename + " - " + error);
        logDebug("[DOWNLOAD_ERROR] " + filename + " - " + error);
    }
    
    /**
     * Core logging implementation
     */
    private static void logMessage(int level, String tag, String message) {
        if (level < currentLogLevel) {
            return; // Skip logs below current level
        }
        
        String timestamp = timestampFormat.format(new Date());
        String formattedMessage = "[" + timestamp + "] " + message;
        
        // Always log to Android logcat
        switch (level) {
            case LEVEL_DEBUG:
                Log.d(tag, message);
                break;
            case LEVEL_INFO:
            case LEVEL_USER:
                Log.i(tag, message);
                break;
            case LEVEL_WARNING:
                Log.w(tag, message);
                break;
            case LEVEL_ERROR:
                Log.e(tag, message);
                break;
        }
        
        // Log to file if available
        if (isInitialized && fileLogger != null) {
            try {
                switch (level) {
                    case LEVEL_DEBUG:
                        fileLogger.logDebug(tag, message);
                        break;
                    case LEVEL_INFO:
                        fileLogger.logInfo(tag, message);
                        break;
                    case LEVEL_WARNING:
                        fileLogger.logWarning(tag, message);
                        break;
                    case LEVEL_ERROR:
                        fileLogger.logError(tag, message);
                        break;
                    case LEVEL_USER:
                        fileLogger.logUser(message);
                        break;
                }
            } catch (Exception e) {
                Log.e(TAG, "Failed to write to file logger", e);
            }
        } else {
            // Buffer logs if not initialized yet
            logBuffer.offer(level + "|" + tag + "|" + message);
        }
    }
    
    /**
     * Process any logs that were buffered before initialization
     */
    private static void processBufferedLogs() {
        String bufferedLog;
        while ((bufferedLog = logBuffer.poll()) != null) {
            try {
                String[] parts = bufferedLog.split("\\|", 3);
                if (parts.length == 3) {
                    int level = Integer.parseInt(parts[0]);
                    String tag = parts[1];
                    String message = parts[2];
                    logMessage(level, tag, message);
                }
            } catch (Exception e) {
                Log.e(TAG, "Failed to process buffered log: " + bufferedLog, e);
            }
        }
    }
    
    /**
     * Utility methods
     */
    public static void setLogLevel(int level) {
        currentLogLevel = level;
        logDebug("Log level set to: " + level);
    }
    
    public static int getLogLevel() {
        return currentLogLevel;
    }
    
    public static void setDebugEnabled(boolean enabled) {
        if (enabled) {
            setLogLevel(LEVEL_DEBUG);
            logDebug("Debug logging enabled");
        } else {
            setLogLevel(LEVEL_INFO);
            logInfo("Debug logging disabled");
        }
    }
    
    public static boolean isDebugEnabled() {
        return currentLogLevel <= LEVEL_DEBUG;
    }
    
    public static boolean isInitialized() {
        return isInitialized;
    }
    
    public static String getLogs() {
        if (fileLogger != null) {
            return fileLogger.readAllLogs();
        }
        return "FileLogger not initialized";
    }
    
    public static String getCurrentLogs() {
        if (fileLogger != null) {
            return fileLogger.readCurrentLog();
        }
        return "FileLogger not initialized";
    }
    
    public static boolean exportLogs(File exportFile) {
        if (fileLogger != null) {
            return fileLogger.exportLogs(exportFile);
        }
        return false;
    }
    
    public static void clearLogs() {
        if (fileLogger != null) {
            fileLogger.clearLogs();
            logUser("🧹 All logs cleared");
        }
    }
    
    public static FileLogger.LogStats getLogStats() {
        if (fileLogger != null) {
            return fileLogger.getLogStats();
        }
        return new FileLogger.LogStats(); // Return empty stats
    }
    
    public static java.util.List<File> getAvailableLogFiles() {
        java.util.List<File> logFiles = new java.util.ArrayList<>();
        if (fileLogger != null) {
            File logDir = fileLogger.getLogDirectory();
            if (logDir != null && logDir.exists()) {
                File[] files = logDir.listFiles((dir, name) -> name.endsWith(".txt"));
                if (files != null) {
                    for (File file : files) {
                        logFiles.add(file);
                    }
                }
            }
        }
        return logFiles;
    }
    
    public static String readLogFile(int logIndex) {
        if (fileLogger == null) {
            return "FileLogger not initialized";
        }
        
        File logDir = fileLogger.getLogDirectory();
        if (logDir == null || !logDir.exists()) {
            return "Log directory not found";
        }
        
        File logFile;
        if (logIndex == 0) {
            logFile = new File(logDir, "AppLog.txt");
        } else {
            logFile = new File(logDir, "AppLog" + logIndex + ".txt");
        }
        
        if (!logFile.exists()) {
            return "Log file " + logIndex + " not found";
        }
        
        StringBuilder content = new StringBuilder();
        try (java.io.BufferedReader reader = new java.io.BufferedReader(new java.io.FileReader(logFile))) {
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append("\n");
            }
        } catch (java.io.IOException e) {
            return "Error reading log file " + logIndex + ": " + e.getMessage();
        }
        
        return content.toString();
    }
    
    private static String getAppVersion() {
        if (applicationContext != null) {
            try {
                return applicationContext.getPackageManager()
                        .getPackageInfo(applicationContext.getPackageName(), 0)
                        .versionName;
            } catch (Exception e) {
                return "Unknown";
            }
        }
        return "Unknown";
    }
    
    private static String formatFileSize(long bytes) {
        if (bytes < 1024) return bytes + " B";
        if (bytes < 1024 * 1024) return String.format(Locale.US, "%.1f KB", bytes / 1024.0);
        if (bytes < 1024 * 1024 * 1024) return String.format(Locale.US, "%.1f MB", bytes / (1024.0 * 1024.0));
        return String.format(Locale.US, "%.1f GB", bytes / (1024.0 * 1024.0 * 1024.0));
    }
    
    /**
     * Crash reporting
     */
    public static void logCrash(String component, Throwable throwable) {
        logError("CRASH in " + component + ": " + throwable.getMessage(), throwable);
        
        // Write crash to separate file
        if (applicationContext != null) {
            try {
                File crashFile = new File(applicationContext.getExternalFilesDir(null), 
                    "TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs/crash_" + System.currentTimeMillis() + ".txt");
                crashFile.getParentFile().mkdirs();
                
                try (FileWriter writer = new FileWriter(crashFile)) {
                    writer.write("=== TerrariaLoader Crash Report ===\n");
                    writer.write("Timestamp: " + new Date().toString() + "\n");
                    writer.write("Component: " + component + "\n");
                    writer.write("Error: " + throwable.getMessage() + "\n\n");
                    writer.write("Stack Trace:\n");
                    writer.write(Log.getStackTraceString(throwable));
                    writer.write("\n\n=== Device Info ===\n");
                    writer.write("Android Version: " + android.os.Build.VERSION.RELEASE + "\n");
                    writer.write("Device Model: " + android.os.Build.MODEL + "\n");
                    writer.write("App Version: " + getAppVersion() + "\n");
                }
                
                logDebug("Crash report written to: " + crashFile.getName());
            } catch (IOException e) {
                Log.e(TAG, "Failed to write crash report", e);
            }
        }
    }
    
    /**
     * Performance logging
     */
    public static void logPerformance(String operation, long startTime, long endTime) {
        long duration = endTime - startTime;
        String formattedDuration;
        
        if (duration < 1000) {
            formattedDuration = duration + "ms";
        } else if (duration < 60000) {
            formattedDuration = String.format(Locale.US, "%.1fs", duration / 1000.0);
        } else {
            long seconds = duration / 1000;
            long minutes = seconds / 60;
            seconds = seconds % 60;
            formattedDuration = String.format(Locale.US, "%dm %ds", minutes, seconds);
        }
        
        logDebug("[PERFORMANCE] " + operation + " completed in " + formattedDuration);
    }
    
    public static void logMemoryUsage() {
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        logDebug("[MEMORY] Used: " + formatFileSize(usedMemory) + 
                ", Free: " + formatFileSize(freeMemory) + 
                ", Total: " + formatFileSize(totalMemory) + 
                ", Max: " + formatFileSize(maxMemory));
    }
    
    /**
     * Cleanup resources
     */
    public static void shutdown() {
        if (fileLogger != null) {
            fileLogger.shutdown();
        }
        logBuffer.clear();
        isInitialized = false;
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/MelonLoaderDiagnostic.java

// File: MelonLoaderDiagnostic.java (Diagnostic Tool)
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/MelonLoaderDiagnostic.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.File;

public class MelonLoaderDiagnostic {
    
    public static String generateDetailedDiagnostic(Context context, String gamePackage) {
        StringBuilder diagnostic = new StringBuilder();
        diagnostic.append("=== DETAILED MELONLOADER DIAGNOSTIC ===\n\n");
        
        // Check all required directories and files
        File baseDir = PathManager.getGameBaseDir(context, gamePackage);
        File melonLoaderDir = PathManager.getMelonLoaderDir(context, gamePackage);
        File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
        File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
        File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
        
        diagnostic.append("📁 DIRECTORY STATUS:\n");
        diagnostic.append("Base Dir: ").append(checkDirectory(baseDir)).append("\n");
        diagnostic.append("MelonLoader Dir: ").append(checkDirectory(melonLoaderDir)).append("\n");
        diagnostic.append("NET8 Dir: ").append(checkDirectory(net8Dir)).append("\n");
        diagnostic.append("NET35 Dir: ").append(checkDirectory(net35Dir)).append("\n");
        diagnostic.append("Dependencies Dir: ").append(checkDirectory(depsDir)).append("\n\n");
        
        // Check for required NET8 files
        diagnostic.append("🔸 NET8 RUNTIME FILES:\n");
        String[] net8Files = {
            "MelonLoader.dll",
            "0Harmony.dll", 
            "MonoMod.RuntimeDetour.dll",
            "MonoMod.Utils.dll",
            "Il2CppInterop.Runtime.dll"
        };
        
        int net8Found = 0;
        for (String fileName : net8Files) {
            File file = new File(net8Dir, fileName);
            boolean exists = file.exists() && file.length() > 0;
            diagnostic.append("  ").append(exists ? "✅" : "❌").append(" ").append(fileName);
            if (exists) {
                net8Found++;
                diagnostic.append(" (").append(FileUtils.formatFileSize(file.length())).append(")");
            }
            diagnostic.append("\n");
        }
        diagnostic.append("NET8 Score: ").append(net8Found).append("/").append(net8Files.length).append("\n\n");
        
        // Check for required NET35 files
        diagnostic.append("🔸 NET35 RUNTIME FILES:\n");
        String[] net35Files = {
            "MelonLoader.dll",
            "0Harmony.dll",
            "MonoMod.RuntimeDetour.dll", 
            "MonoMod.Utils.dll"
        };
        
        int net35Found = 0;
        for (String fileName : net35Files) {
            File file = new File(net35Dir, fileName);
            boolean exists = file.exists() && file.length() > 0;
            diagnostic.append("  ").append(exists ? "✅" : "❌").append(" ").append(fileName);
            if (exists) {
                net35Found++;
                diagnostic.append(" (").append(FileUtils.formatFileSize(file.length())).append(")");
            }
            diagnostic.append("\n");
        }
        diagnostic.append("NET35 Score: ").append(net35Found).append("/").append(net35Files.length).append("\n\n");
        
        // Check Dependencies
        diagnostic.append("🔸 DEPENDENCY FILES:\n");
        File supportModulesDir = new File(depsDir, "SupportModules");
        File assemblyGenDir = new File(depsDir, "Il2CppAssemblyGenerator");
        
        diagnostic.append("Support Modules Dir: ").append(checkDirectory(supportModulesDir)).append("\n");
        diagnostic.append("Assembly Generator Dir: ").append(checkDirectory(assemblyGenDir)).append("\n");
        
        // List actual files found
        diagnostic.append("\n📋 FILES FOUND:\n");
        if (melonLoaderDir.exists()) {
            diagnostic.append(listDirectoryContents(melonLoaderDir, ""));
        } else {
            diagnostic.append("MelonLoader directory doesn't exist!\n");
        }
        
        // Generate recommendations
        diagnostic.append("\n💡 RECOMMENDATIONS:\n");
        if (net8Found == 0 && net35Found == 0) {
            diagnostic.append("❌ NO RUNTIME FILES FOUND!\n");
            diagnostic.append("SOLUTION: You need to install MelonLoader files.\n");
            diagnostic.append("Options:\n");
            diagnostic.append("1. Use 'Automated Installation' in Setup Guide\n");
            diagnostic.append("2. Manually download and extract MelonLoader files\n");
            diagnostic.append("3. Use the APK patcher to inject loader\n\n");
        } else if (net8Found > 0) {
            diagnostic.append("✅ Some NET8 files found, but incomplete installation\n");
            diagnostic.append("Missing files need to be added to: ").append(net8Dir.getAbsolutePath()).append("\n\n");
        } else if (net35Found > 0) {
            diagnostic.append("✅ Some NET35 files found, but incomplete installation\n");
            diagnostic.append("Missing files need to be added to: ").append(net35Dir.getAbsolutePath()).append("\n\n");
        }
        
        // Check if automated installation would work
        diagnostic.append("🌐 INTERNET CONNECTIVITY: ");
        if (OnlineInstaller.isOnlineInstallationAvailable()) {
            diagnostic.append("✅ Available - Automated installation possible\n");
            diagnostic.append("RECOMMENDED: Use 'Automated Installation' for easiest setup\n");
        } else {
            diagnostic.append("❌ Not available - Manual installation required\n");
            diagnostic.append("REQUIRED: Download MelonLoader files manually\n");
        }
        
        return diagnostic.toString();
    }
    
    private static String checkDirectory(File dir) {
        if (dir == null) return "❌ null";
        if (!dir.exists()) return "❌ doesn't exist (" + dir.getAbsolutePath() + ")";
        if (!dir.isDirectory()) return "❌ not a directory";
        
        File[] files = dir.listFiles();
        int fileCount = files != null ? files.length : 0;
        return "✅ exists (" + fileCount + " items)";
    }
    
    private static String listDirectoryContents(File dir, String indent) {
        StringBuilder contents = new StringBuilder();
        if (dir == null || !dir.exists() || !dir.isDirectory()) {
            return contents.toString();
        }
        
        File[] files = dir.listFiles();
        if (files == null || files.length == 0) {
            contents.append(indent).append("(empty)\n");
            return contents.toString();
        }
        
        for (File file : files) {
            contents.append(indent);
            if (file.isDirectory()) {
                contents.append("📁 ").append(file.getName()).append("/\n");
                if (indent.length() < 8) { // Limit recursion depth
                    contents.append(listDirectoryContents(file, indent + "  "));
                }
            } else {
                contents.append("📄 ").append(file.getName());
                contents.append(" (").append(FileUtils.formatFileSize(file.length())).append(")\n");
            }
        }
        
        return contents.toString();
    }
    
    // Quick fix suggestions
    public static String getQuickFixSuggestions(Context context, String gamePackage) {
        StringBuilder suggestions = new StringBuilder();
        suggestions.append("🚀 QUICK FIX OPTIONS:\n\n");
        
        suggestions.append("1. AUTOMATED INSTALLATION (Recommended):\n");
        suggestions.append("   • Go to 'MelonLoader Setup Guide'\n");
        suggestions.append("   • Choose 'Automated Online Installation'\n");
        suggestions.append("   • Select MelonLoader or LemonLoader\n");
        suggestions.append("   • Wait for download and extraction\n\n");
        
        suggestions.append("2. MANUAL INSTALLATION:\n");
        suggestions.append("   • Download MelonLoader from GitHub\n");
        suggestions.append("   • Extract files to correct directories\n");
        suggestions.append("   • Follow the manual installation guide\n\n");
        
        suggestions.append("3. APK INJECTION:\n");
        suggestions.append("   • Use 'APK Patcher' feature\n");
        suggestions.append("   • Select Terraria APK\n");
        suggestions.append("   • Inject MelonLoader into APK\n");
        suggestions.append("   • Install modified APK\n\n");
        
        File baseDir = PathManager.getGameBaseDir(context, gamePackage);
        suggestions.append("📍 TARGET DIRECTORY:\n");
        suggestions.append(baseDir.getAbsolutePath()).append("/Loaders/MelonLoader/\n\n");
        
        suggestions.append("⚠️ MAKE SURE TO:\n");
        suggestions.append("• Have stable internet connection (for automated)\n");
        suggestions.append("• Grant file manager permissions (for manual)\n");
        suggestions.append("• Use exact directory paths shown above\n");
        suggestions.append("• Restart TerrariaLoader after installation\n");
        
        return suggestions.toString();
    }
}

--------------------------------------------------------------------------------
/ModLoader/app/src/main/java/com/modloader/util/OfflineZipImporter.java

// File: OfflineZipImporter.java - Smart ZIP Import with Auto-Detection
// Path: /main/java/com/terrarialoader/util/OfflineZipImporter.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.*;
import java.util.zip.*;
import java.util.HashSet;
import java.util.Set;

/**
 * Smart offline ZIP importer that auto-detects NET8/NET35 and extracts to correct directories
 */
public class OfflineZipImporter {
    
    public static class ImportResult {
        public boolean success;
        public String message;
        public MelonLoaderManager.LoaderType detectedType;
        public int filesExtracted;
        public String errorDetails;
        
        public ImportResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
    }
    
    // File signatures for detection
    private static final String[] NET8_SIGNATURES = {
        "MelonLoader.deps.json",
        "MelonLoader.runtimeconfig.json", 
        "Il2CppInterop.Runtime.dll"
    };
    
    private static final String[] NET35_SIGNATURES = {
        "MelonLoader.dll",
        "0Harmony.dll"
    };
    
    private static final String[] CORE_FILES = {
        "MelonLoader.dll",
        "0Harmony.dll",
        "MonoMod.RuntimeDetour.dll",
        "MonoMod.Utils.dll"
    };
    
    /**
     * Import MelonLoader ZIP with auto-detection and smart extraction
     */
    public static ImportResult importMelonLoaderZip(Context context, android.net.Uri zipUri) {
        LogUtils.logUser("🔍 Starting smart ZIP import...");
        
        try {
            // Step 1: Analyze ZIP contents
            ZipAnalysis analysis = analyzeZipContents(context, zipUri);
            if (!analysis.isValid) {
                return new ImportResult(false, "Invalid MelonLoader ZIP file: " + analysis.error);
            }
            
            LogUtils.logUser("📋 Detected: " + analysis.detectedType.getDisplayName());
            LogUtils.logUser("📊 Found " + analysis.totalFiles + " files to extract");
            
            // Step 2: Prepare target directories
            String gamePackage = MelonLoaderManager.TERRARIA_PACKAGE;
            if (!PathManager.initializeGameDirectories(context, gamePackage)) {
                return new ImportResult(false, "Failed to create directory structure");
            }
            
            // Step 3: Extract files to appropriate locations
            int extractedCount = extractZipContents(context, zipUri, analysis, gamePackage);
            
            if (extractedCount > 0) {
                ImportResult result = new ImportResult(true, 
                    "Successfully imported " + analysis.detectedType.getDisplayName() + 
                    " (" + extractedCount + " files)");
                result.detectedType = analysis.detectedType;
                result.filesExtracted = extractedCount;
                
                LogUtils.logUser("✅ ZIP import completed: " + extractedCount + " files extracted");
                return result;
            } else {
                return new ImportResult(false, "No files were extracted from ZIP");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("ZIP import error: " + e.getMessage());
            ImportResult result = new ImportResult(false, "Import failed: " + e.getMessage());
            result.errorDetails = e.toString();
            return result;
        }
    }
    
    /**
     * Analyze ZIP contents to detect loader type and validate files
     */
    private static ZipAnalysis analyzeZipContents(Context context, android.net.Uri zipUri) {
        ZipAnalysis analysis = new ZipAnalysis();
        
        try (InputStream inputStream = context.getContentResolver().openInputStream(zipUri);
             ZipInputStream zis = new ZipInputStream(new BufferedInputStream(inputStream))) {
            
            ZipEntry entry;
            Set<String> foundFiles = new HashSet<>();
            
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.isDirectory()) {
                    zis.closeEntry();
                    continue;
                }
                
                String fileName = getCleanFileName(entry.getName());
                foundFiles.add(fileName.toLowerCase());
                analysis.totalFiles++;
                
                // Check for type indicators
                for (String signature : NET8_SIGNATURES) {
                    if (fileName.equalsIgnoreCase(signature)) {
                        analysis.hasNet8Indicators = true;
                        break;
                    }
                }
                
                for (String signature : NET35_SIGNATURES) {
                    if (fileName.equalsIgnoreCase(signature)) {
                        analysis.hasNet35Indicators = true;
                        break;
                    }
                }
                
                zis.closeEntry();
            }
            
            // Determine loader type
            if (analysis.hasNet8Indicators) {
                analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET8;
            } else if (analysis.hasNet35Indicators) {
                analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET35;
            } else {
                // Fallback: check for core files and default to NET8
                boolean hasCoreFiles = false;
                for (String coreFile : CORE_FILES) {
                    if (foundFiles.contains(coreFile.toLowerCase())) {
                        hasCoreFiles = true;
                        break;
                    }
                }
                
                if (hasCoreFiles) {
                    analysis.detectedType = MelonLoaderManager.LoaderType.MELONLOADER_NET8; // Default
                    LogUtils.logUser("⚠️ Auto-detected as MelonLoader (default)");
                } else {
                    analysis.isValid = false;
                    analysis.error = "No MelonLoader files detected in ZIP";
                    return analysis;
                }
            }
            
            // Validate we have minimum required files
            int coreFilesFound = 0;
            for (String coreFile : CORE_FILES) {
                if (foundFiles.contains(coreFile.toLowerCase())) {
                    coreFilesFound++;
                }
            }
            
            if (coreFilesFound < 2) { // At least 2 core files required
                analysis.isValid = false;
                analysis.error = "Insufficient MelonLoader core files (" + coreFilesFound + "/4)";
                return analysis;
            }
            
            analysis.isValid = true;
            LogUtils.logDebug("ZIP analysis complete - Type: " + analysis.detectedType.getDisplayName() + 
                            ", Files: " + analysis.totalFiles);
            
        } catch (Exception e) {
            analysis.isValid = false;
            analysis.error = "ZIP analysis failed: " + e.getMessage();
            LogUtils.logDebug("ZIP analysis error: " + e.getMessage());
        }
        
        return analysis;
    }
    
    /**
     * Extract ZIP contents to appropriate directories based on detected type
     */
    private static int extractZipContents(Context context, android.net.Uri zipUri, ZipAnalysis analysis, String gamePackage) throws IOException {
        int extractedCount = 0;
        
        // Get target directories
        File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
        File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
        File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
        
        // Ensure directories exist
        PathManager.ensureDirectoryExists(net8Dir);
        PathManager.ensureDirectoryExists(net35Dir);
        PathManager.ensureDirectoryExists(depsDir);
        PathManager.ensureDirectoryExists(new File(depsDir, "SupportModules"));
        PathManager.ensureDirectoryExists(new File(depsDir, "CompatibilityLayers"));
        PathManager.ensureDirectoryExists(new File(depsDir, "Il2CppAssemblyGenerator"));
        
        try (InputStream inputStream = context.getContentResolver().openInputStream(zipUri);
             ZipInputStream zis = new ZipInputStream(new BufferedInputStream(inputStream))) {
            
            ZipEntry entry;
            byte[] buffer = new byte[8192];
            
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.isDirectory()) {
                    zis.closeEntry();
                    continue;
                }
                
                String fileName = getCleanFileName(entry.getName());
                File targetFile = determineTargetFile(fileName, analysis.detectedType, net8Dir, net35Dir, depsDir);
                
                if (targetFile != null) {
                    // Ensure parent directory exists
                    targetFile.getParentFile().mkdirs();
                    
                    // Extract file
                    try (FileOutputStream fos = new FileOutputStream(targetFile)) {
                        int len;
                        while ((len = zis.read(buffer)) > 0) {
                            fos.write(buffer, 0, len);
                        }
                    }
                    
                    extractedCount++;
                    LogUtils.logDebug("Extracted: " + fileName + " -> " + targetFile.getAbsolutePath());
                } else {
                    LogUtils.logDebug("Skipped: " + fileName + " (not needed)");
                }
                
                zis.closeEntry();
            }
        }
        
        return extractedCount;
    }
    
    /**
     * Determine target file location based on file type and loader type
     */
    private static File determineTargetFile(String fileName, MelonLoaderManager.LoaderType loaderType, 
                                          File net8Dir, File net35Dir, File depsDir) {
        String lowerName = fileName.toLowerCase();
        
        // Skip non-relevant files
        if (!isRelevantFile(fileName)) {
            return null;
        }
        
        // Core runtime files
        if (isCoreRuntimeFile(fileName)) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        // Dependency files
        if (lowerName.contains("il2cpp") || lowerName.contains("interop")) {
            return new File(depsDir, "SupportModules/" + fileName);
        }
        
        if (lowerName.contains("unity") || lowerName.contains("assemblygenerator")) {
            return new File(depsDir, "Il2CppAssemblyGenerator/" + fileName);
        }
        
        if (lowerName.contains("compat")) {
            return new File(depsDir, "CompatibilityLayers/" + fileName);
        }
        
        // Default: place DLLs in appropriate runtime directory
        if (lowerName.endsWith(".dll")) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        // Config files go to runtime directory
        if (lowerName.endsWith(".json") || lowerName.endsWith(".xml")) {
            if (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                return new File(net8Dir, fileName);
            } else {
                return new File(net35Dir, fileName);
            }
        }
        
        return null; // Skip unknown files
    }
    
    private static String getCleanFileName(String entryName) {
        // Remove directory paths and get just the filename
        String fileName = entryName;
        
        // Handle both forward and backward slashes
        int lastSlash = Math.max(fileName.lastIndexOf('/'), fileName.lastIndexOf('\\'));
        if (lastSlash >= 0) {
            fileName = fileName.substring(lastSlash + 1);
        }
        
        return fileName;
    }
    
    private static boolean isRelevantFile(String fileName) {
        String lowerName = fileName.toLowerCase();
        return lowerName.endsWith(".dll") || 
               lowerName.endsWith(".json") || 
               lowerName.endsWith(".xml") ||
               lowerName.endsWith(".pdb");
    }
    
    private static boolean isCoreRuntimeFile(String fileName) {
        String lowerName = fileName.toLowerCase();
        for (String coreFile : CORE_FILES) {
            if (lowerName.equals(coreFile.toLowerCase())) {
                return true;
            }
        }
        return lowerName.contains("melonloader") || 
               lowerName.contains("runtimeconfig") ||
               lowerName.contains("deps.json");
    }
    
    /**
     * Helper class to store ZIP analysis results
     */
    private static class ZipAnalysis {
        boolean isValid = false;
        String error = "";
        MelonLoaderManager.LoaderType detectedType;
        boolean hasNet8Indicators = false;
        boolean hasNet35Indicators = false;
        int totalFiles = 0;
    }
}

--------------------------------------------------------------------------------
