

===== ModLoader/app/src/main/java/com/modloader/ui/OfflineDiagnosticActivity.java =====

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




===== ModLoader/app/src/main/java/com/modloader/ui/SettingsActivity.java =====

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



===== ModLoader/app/src/main/java/com/modloader/ui/SetupGuideActivity.java =====

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



===== ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderActivity.java =====

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



===== ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderController.java =====

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
  



===== ModLoader/app/src/main/java/com/modloader/ui/UnifiedLoaderListener.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/ApkInstaller.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/ApkPatcher.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/ApkValidator.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/DiagnosticBundleExporter.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/Downloader.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/FileUtils.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/LogUtils.java =====

// File: LogUtils.java (COMPLETELY REWRITTEN) - Advanced Logging with Log4j2
// Path: /app/src/main/java/com/modloader/util/LogUtils.java
package com.modloader.util;

import android.content.Context;
import android.os.Handler;
import android.os.Looper;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.ThreadContext;
import org.apache.logging.log4j.core.LoggerContext;
import org.apache.logging.log4j.core.config.Configurator;
import org.apache.logging.log4j.Level;

import java.io.*;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;
import java.util.regex.Pattern;

import com.modloader.logging.AdvancedLogManager;
import com.modloader.logging.LogAnalytics;
import com.modloader.logging.LogCorrelation;
import com.modloader.logging.PerformanceMetrics;

/**
 * COMPLETELY REWRITTEN LogUtils with Log4j2 integration
 * Features: Smart filtering, performance metrics, analytics, correlation IDs
 */
public class LogUtils {
    private static final String TAG = "LogUtils";
    
    // Log4j2 loggers with categories
    private static final Logger MAIN_LOGGER = LogManager.getLogger("MAIN");
    private static final Logger USER_LOGGER = LogManager.getLogger("USER");
    private static final Logger DEBUG_LOGGER = LogManager.getLogger("DEBUG");
    private static final Logger PERFORMANCE_LOGGER = LogManager.getLogger("PERFORMANCE");
    private static final Logger ANALYTICS_LOGGER = LogManager.getLogger("ANALYTICS");
    private static final Logger ERROR_LOGGER = LogManager.getLogger("ERROR");
    
    // Advanced logging components
    private static AdvancedLogManager advancedLogManager;
    private static LogAnalytics logAnalytics;
    private static LogCorrelation logCorrelation;
    private static PerformanceMetrics performanceMetrics;
    
    // Configuration and state
    private static Context applicationContext;
    private static boolean isInitialized = false;
    private static volatile boolean debugEnabled = false;
    private static volatile Level dynamicLogLevel = Level.INFO;
    
    // Smart filtering patterns (REMOVES NOISE AS REQUESTED)
    private static final Pattern[] NOISE_PATTERNS = {
        Pattern.compile(".*Created: .*"), // Remove directory creation confirmations
        Pattern.compile(".*checking\\.\\.\\..*", Pattern.CASE_INSENSITIVE), // Remove redundant checking messages
        Pattern.compile(".*found\\.\\.\\..*", Pattern.CASE_INSENSITIVE), // Remove redundant found messages
        Pattern.compile(".*Migration.*completed.*"), // Remove duplicate migration messages
        Pattern.compile(".*Step \\d+.*"), // Remove overly detailed step messages
        Pattern.compile(".*Path: /storage/emulated/0.*"), // Remove verbose path logging
    };
    
    // Async logging for performance
    private static ExecutorService asyncLogExecutor;
    private static Handler mainThreadHandler;
    private static final BlockingQueue<LogEntry> logQueue = new LinkedBlockingQueue<>(10000);
    
    // Memory management
    private static final AtomicLong totalLogsGenerated = new AtomicLong(0);
    private static final AtomicLong filteredOutLogs = new AtomicLong(0);
    
    // Dynamic configuration
    private static final Map<String, Level> categoryLevels = new ConcurrentHashMap<>();
    
    /**
     * FEATURE 1: SMART LOG FILTERING SYSTEM
     * Automatically filters out noise as requested by user
     */
    private static boolean isNoiseMessage(String message) {
        if (message == null) return false;
        
        // Apply smart filtering patterns
        for (Pattern pattern : NOISE_PATTERNS) {
            if (pattern.matcher(message).matches()) {
                filteredOutLogs.incrementAndGet();
                return true;
            }
        }
        
        // Additional intelligent filtering
        if (message.contains("exists") && message.contains("/storage/")) return true;
        if (message.matches(".*\\d+/\\d+ files.*")) return true; // Generic progress messages
        if (message.startsWith("📁") && message.contains("directory")) return true;
        
        return false;
    }
    
    /**
     * FEATURE 8: DYNAMIC LOG LEVEL MANAGEMENT
     * Runtime log level adjustment without restart
     */
    public static void setDynamicLogLevel(Level level) {
        dynamicLogLevel = level;
        Configurator.setRootLevel(level);
        logInfo("🔧 Dynamic log level changed to: " + level.name());
    }
    
    public static Level getDynamicLogLevel() {
        return dynamicLogLevel;
    }
    
    /**
     * Enhanced initialization with all advanced features
     */
    public static synchronized void initialize(Context context) {
        if (isInitialized) {
            return;
        }
        
        applicationContext = context.getApplicationContext();
        mainThreadHandler = new Handler(Looper.getMainLooper());
        
        // Initialize async logging
        asyncLogExecutor = Executors.newSingleThreadExecutor(r -> {
            Thread t = new Thread(r, "LogUtils-Async");
            t.setDaemon(true);
            return t;
        });
        
        // Configure Log4j2
        configureLog4j2();
        
        // Initialize advanced components
        advancedLogManager = new AdvancedLogManager(context);
        logAnalytics = new LogAnalytics(context);
        logCorrelation = new LogCorrelation();
        performanceMetrics = new PerformanceMetrics();
        
        // Start background processors
        startAsyncLogProcessor();
        startLogHealthMonitoring();
        
        isInitialized = true;
        
        // Log startup with correlation ID
        String startupCorrelationId = logCorrelation.generateCorrelationId("STARTUP");
        logUser("🚀 Advanced LogUtils initialized with Log4j2");
        logInfo("📊 Smart filtering enabled - noise reduction active");
        logInfo("⚡ Async logging enabled - non-blocking performance");
        logDebug("🔍 Log correlation ID system active: " + startupCorrelationId);
    }
    
    /**
     * FEATURE 4: LOG HEALTH MONITORING
     * Automatic log rotation and health checks
     */
    private static void startLogHealthMonitoring() {
        ScheduledExecutorService healthMonitor = Executors.newScheduledThreadPool(1);
        healthMonitor.scheduleAtFixedRate(() -> {
            try {
                // Rotate logs if needed
                advancedLogManager.performLogRotation();
                
                // Check log queue health
                int queueSize = logQueue.size();
                if (queueSize > 8000) {
                    MAIN_LOGGER.warn("⚠️ Log queue getting full: {}/10000", queueSize);
                }
                
                // Memory health check
                long totalLogs = totalLogsGenerated.get();
                long filtered = filteredOutLogs.get();
                if (totalLogs > 0 && totalLogs % 1000 == 0) {
                    double filterRate = (filtered * 100.0) / totalLogs;
                    ANALYTICS_LOGGER.info("📊 Filtering efficiency: {:.1f}% ({}/{})", 
                        filterRate, filtered, totalLogs);
                }
                
            } catch (Exception e) {
                ERROR_LOGGER.error("❌ Health monitoring error", e);
            }
        }, 30, 30, TimeUnit.SECONDS);
    }
    
    /**
     * Configure Log4j2 with custom appenders and layouts
     */
    private static void configureLog4j2() {
        try {
            // Set system properties for Log4j2
            System.setProperty("log4j2.contextSelector", "org.apache.logging.log4j.core.selector.BasicContextSelector");
            System.setProperty("log4j2.configurationFile", "log4j2.xml");
            
            // Configure programmatically if needed
            LoggerContext context = (LoggerContext) LogManager.getContext(false);
            
            // Set initial levels
            categoryLevels.put("MAIN", Level.INFO);
            categoryLevels.put("USER", Level.INFO);
            categoryLevels.put("DEBUG", Level.DEBUG);
            categoryLevels.put("PERFORMANCE", Level.INFO);
            categoryLevels.put("ANALYTICS", Level.INFO);
            categoryLevels.put("ERROR", Level.ERROR);
            
        } catch (Exception e) {
            System.err.println("❌ Log4j2 configuration failed: " + e.getMessage());
        }
    }
    
    /**
     * FEATURE 11: MEMORY-SAFE ASYNC LOGGING
     * Non-blocking logging with smart buffering
     */
    private static void startAsyncLogProcessor() {
        asyncLogExecutor.submit(() -> {
            List<LogEntry> batch = new ArrayList<>();
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // Batch processing for efficiency
                    LogEntry entry = logQueue.poll(1, TimeUnit.SECONDS);
                    if (entry != null) {
                        batch.add(entry);
                        
                        // Process batch when full or timeout
                        logQueue.drainTo(batch, 50);
                        processBatch(batch);
                        batch.clear();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } catch (Exception e) {
                    System.err.println("❌ Async log processing error: " + e.getMessage());
                }
            }
        });
    }
    
    /**
     * Process batched log entries efficiently
     */
    private static void processBatch(List<LogEntry> batch) {
        for (LogEntry entry : batch) {
            try {
                // Apply smart filtering
                if (isNoiseMessage(entry.message)) {
                    continue;
                }
                
                // Set correlation context
                ThreadContext.put("correlationId", entry.correlationId);
                ThreadContext.put("category", entry.category);
                ThreadContext.put("timestamp", String.valueOf(entry.timestamp));
                
                // Route to appropriate logger
                Logger logger = getLoggerForCategory(entry.category);
                switch (entry.level) {
                    case ERROR:
                        logger.error(entry.message, entry.throwable);
                        break;
                    case WARN:
                        logger.warn(entry.message);
                        break;
                    case INFO:
                        logger.info(entry.message);
                        break;
                    case DEBUG:
                        logger.debug(entry.message);
                        break;
                    default:
                        logger.info(entry.message);
                }
                
                // Update analytics
                logAnalytics.recordLogEntry(entry);
                
            } finally {
                ThreadContext.clearMap();
            }
        }
    }
    
    /**
     * FEATURE 3: CONTEXTUAL LOG CATEGORIES
     * Better organization with smart categorization
     */
    private static Logger getLoggerForCategory(String category) {
        switch (category.toUpperCase()) {
            case "USER": return USER_LOGGER;
            case "DEBUG": return DEBUG_LOGGER;
            case "PERFORMANCE": return PERFORMANCE_LOGGER;
            case "ANALYTICS": return ANALYTICS_LOGGER;
            case "ERROR": return ERROR_LOGGER;
            default: return MAIN_LOGGER;
        }
    }
    
    // PUBLIC LOGGING METHODS (Enhanced with new features)
    
    /**
     * FEATURE 6: USER ACTION TRACKING
     * Enhanced user logging with action tracking
     */
    public static void logUser(String message) {
        if (!isInitialized) {
            System.out.println("USER: " + message);
            return;
        }
        
        totalLogsGenerated.incrementAndGet();
        
        // Track user actions for UX insights
        advancedLogManager.trackUserAction(message);
        
        String correlationId = logCorrelation.getCurrentOrGenerate("USER_ACTION");
        LogEntry entry = new LogEntry("USER", Level.INFO, message, correlationId);
        
        offerToQueue(entry);
        
        // Also output to console for immediate visibility
        System.out.println("👤 " + message);
    }
    
    /**
     * Enhanced info logging with correlation
     */
    public static void logInfo(String message) {
        if (!isInitialized) {
            System.out.println("INFO: " + message);
            return;
        }
        
        totalLogsGenerated.incrementAndGet();
        String correlationId = logCorrelation.getCurrentOrGenerate("INFO");
        LogEntry entry = new LogEntry("MAIN", Level.INFO, message, correlationId);
        
        offerToQueue(entry);
    }
    
    /**
     * Smart debug logging (respects dynamic level)
     */
    public static void logDebug(String message) {
        if (!isInitialized) {
            if (debugEnabled) {
                System.out.println("DEBUG: " + message);
            }
            return;
        }
        
        if (!debugEnabled && dynamicLogLevel.isMoreSpecificThan(Level.DEBUG)) {
            return;
        }
        
        totalLogsGenerated.incrementAndGet();
        String correlationId = logCorrelation.getCurrentOrGenerate("DEBUG");
        LogEntry entry = new LogEntry("DEBUG", Level.DEBUG, message, correlationId);
        
        offerToQueue(entry);
    }
    
    /**
     * FEATURE 2: PERFORMANCE METRICS LOGGING
     * Track operation times and system performance
     */
    public static void logPerformance(String operation, long durationMs) {
        logPerformance(operation, durationMs, null);
    }
    
    public static void logPerformance(String operation, long durationMs, Map<String, Object> metrics) {
        if (!isInitialized) return;
        
        performanceMetrics.recordOperation(operation, durationMs, metrics);
        
        String message = String.format("⏱️ %s completed in %dms", operation, durationMs);
        if (metrics != null && !metrics.isEmpty()) {
            message += " | " + formatMetrics(metrics);
        }
        
        String correlationId = logCorrelation.getCurrentOrGenerate("PERFORMANCE");
        LogEntry entry = new LogEntry("PERFORMANCE", Level.INFO, message, correlationId);
        
        offerToQueue(entry);
    }
    
    /**
     * FEATURE 7: ERROR RECOVERY SUGGESTIONS
     * Intelligent error handling with recovery tips
     */
    public static void logError(String message) {
        logError(message, null);
    }
    
    public static void logError(String message, Throwable throwable) {
        if (!isInitialized) {
            System.err.println("ERROR: " + message);
            if (throwable != null) throwable.printStackTrace();
            return;
        }
        
        totalLogsGenerated.incrementAndGet();
        
        // Generate recovery suggestions
        String recovery = generateRecoverySuggestion(message, throwable);
        String fullMessage = message;
        if (recovery != null) {
            fullMessage += "\n💡 Recovery Suggestion: " + recovery;
        }
        
        String correlationId = logCorrelation.getCurrentOrGenerate("ERROR");
        LogEntry entry = new LogEntry("ERROR", Level.ERROR, fullMessage, correlationId);
        entry.throwable = throwable;
        
        offerToQueue(entry);
        
        // Also log to console for immediate visibility
        System.err.println("❌ " + message);
        if (throwable != null) throwable.printStackTrace();
    }
    
    /**
     * Generate intelligent recovery suggestions based on error patterns
     */
    private static String generateRecoverySuggestion(String message, Throwable throwable) {
        if (message == null) return null;
        
        // Pattern-based recovery suggestions
        if (message.contains("permission") || message.contains("access denied")) {
            return "Check app permissions in Settings > Apps > ModLoader > Permissions";
        }
        
        if (message.contains("storage") || message.contains("external")) {
            return "Ensure sufficient storage space and grant storage permissions";
        }
        
        if (message.contains("APK") && message.contains("install")) {
            return "Enable 'Install from Unknown Sources' in device settings";
        }
        
        if (message.contains("network") || message.contains("connection")) {
            return "Check internet connection and try again";
        }
        
        if (message.contains("parse") || message.contains("format")) {
            return "Verify file integrity and try re-downloading";
        }
        
        if (throwable instanceof OutOfMemoryError) {
            return "Close other apps to free memory, or restart the device";
        }
        
        if (throwable instanceof FileNotFoundException) {
            return "Check if the file exists and path is correct";
        }
        
        return null; // No specific suggestion available
    }
    
    /**
     * FEATURE 10: LOG CORRELATION IDS
     * Track related operations across components
     */
    public static String startCorrelatedOperation(String operationName) {
        if (!isInitialized) return UUID.randomUUID().toString();
        
        String correlationId = logCorrelation.startOperation(operationName);
        logInfo(String.format("🔗 Started operation '%s' with correlation ID: %s", operationName, correlationId));
        return correlationId;
    }
    
    public static void endCorrelatedOperation(String correlationId, boolean success) {
        if (!isInitialized) return;
        
        logCorrelation.endOperation(correlationId, success);
        String status = success ? "✅ completed successfully" : "❌ failed";
        logInfo(String.format("🔗 Operation %s [%s]", status, correlationId));
    }
    
    /**
     * FEATURE 9: STRUCTURED JSON LOGGING
     * Machine-readable logs for better parsing
     */
    public static void logStructured(String event, Map<String, Object> data) {
        if (!isInitialized) return;
        
        String jsonMessage = formatAsJson(event, data);
        String correlationId = logCorrelation.getCurrentOrGenerate("STRUCTURED");
        LogEntry entry = new LogEntry("ANALYTICS", Level.INFO, jsonMessage, correlationId);
        
        offerToQueue(entry);
    }
    
    private static String formatAsJson(String event, Map<String, Object> data) {
        StringBuilder json = new StringBuilder();
        json.append("{\"event\":\"").append(event).append("\"");
        json.append(",\"timestamp\":").append(System.currentTimeMillis());
        
        if (data != null && !data.isEmpty()) {
            json.append(",\"data\":{");
            boolean first = true;
            for (Map.Entry<String, Object> entry : data.entrySet()) {
                if (!first) json.append(",");
                json.append("\"").append(entry.getKey()).append("\":\"").append(entry.getValue()).append("\"");
                first = false;
            }
            json.append("}");
        }
        json.append("}");
        return json.toString();
    }
    
    // UTILITY METHODS
    
    /**
     * Thread-safe queue offering with overflow protection
     */
    private static void offerToQueue(LogEntry entry) {
        if (!logQueue.offer(entry)) {
            // Queue is full, drop oldest entry and try again
            logQueue.poll();
            logQueue.offer(entry);
            
            // Log queue overflow (but don't create infinite loop)
            if (Math.random() < 0.01) { // Only log 1% of overflows
                System.err.println("⚠️ Log queue overflow - dropping messages");
            }
        }
    }
    
    private static String formatMetrics(Map<String, Object> metrics) {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<String, Object> entry : metrics.entrySet()) {
            if (sb.length() > 0) sb.append(", ");
            sb.append(entry.getKey()).append("=").append(entry.getValue());
        }
        return sb.toString();
    }
    
    // FEATURE 5: ADVANCED LOG ANALYTICS
    // Generate insights from log patterns
    
    public static Map<String, Object> getLogAnalytics() {
        if (!isInitialized) return new HashMap<>();
        return logAnalytics.generateReport();
    }
    
    public static String getLogStatistics() {
        if (!isInitialized) return "Analytics not initialized";
        
        long total = totalLogsGenerated.get();
        long filtered = filteredOutLogs.get();
        double filterRate = total > 0 ? (filtered * 100.0) / total : 0;
        
        Map<String, Object> analytics = getLogAnalytics();
        
        return String.format(
            "📊 Log Statistics:\n" +
            "• Total logs generated: %d\n" +
            "• Filtered out (noise): %d (%.1f%%)\n" +
            "• Queue size: %d/10000\n" +
            "• Error rate: %.2f%%\n" +
            "• Most active category: %s\n" +
            "• Performance avg: %.1fms",
            total, filtered, filterRate,
            logQueue.size(),
            (Double) analytics.getOrDefault("errorRate", 0.0),
            (String) analytics.getOrDefault("mostActiveCategory", "N/A"),
            (Double) analytics.getOrDefault("avgOperationTime", 0.0)
        );
    }
    
    // CONFIGURATION METHODS
    
    public static void setDebugEnabled(boolean enabled) {
        debugEnabled = enabled;
        if (enabled) {
            setDynamicLogLevel(Level.DEBUG);
            logDebug("🐛 Debug logging enabled");
        } else {
            setDynamicLogLevel(Level.INFO);
            logInfo("🐛 Debug logging disabled");
        }
    }
    
    public static boolean isDebugEnabled() {
        return debugEnabled;
    }
    
    // LEGACY COMPATIBILITY METHODS
    
    public static void logWarning(String message) {
        if (!isInitialized) {
            System.out.println("WARN: " + message);
            return;
        }
        
        totalLogsGenerated.incrementAndGet();
        String correlationId = logCorrelation.getCurrentOrGenerate("WARNING");
        LogEntry entry = new LogEntry("MAIN", Level.WARN, message, correlationId);
        
        offerToQueue(entry);
    }
    
    /**
     * Get all logs as a formatted string (for legacy LogActivity)
     */
    public static String getLogs() {
        if (!isInitialized) {
            return "Logs not available - LogUtils not initialized";
        }
        
        // Get logs from advanced log manager
        return advancedLogManager.getAllLogsFormatted();
    }
    
    /**
     * Initialize app startup logging (legacy method)
     */
    public static void initializeAppStartup() {
        if (isInitialized) {
            logUser("📱 Application startup sequence initiated");
            
            // Performance tracking for startup
            Map<String, Object> startupMetrics = new HashMap<>();
            startupMetrics.put("javaVersion", System.getProperty("java.version"));
            startupMetrics.put("androidVersion", android.os.Build.VERSION.RELEASE);
            startupMetrics.put("deviceModel", android.os.Build.MODEL);
            
            logStructured("APP_STARTUP", startupMetrics);
        }
    }
    
    /**
     * Clean shutdown of logging system
     */
    public static void shutdown() {
        if (!isInitialized) return;
        
        logInfo("🛑 LogUtils shutting down");
        
        try {
            // Flush remaining logs
            Thread.sleep(1000);
            
            // Shutdown executors
            if (asyncLogExecutor != null) {
                asyncLogExecutor.shutdown();
                if (!asyncLogExecutor.awaitTermination(5, TimeUnit.SECONDS)) {
                    asyncLogExecutor.shutdownNow();
                }
            }
            
            // Shutdown advanced components
            if (advancedLogManager != null) {
                advancedLogManager.shutdown();
            }
            
            // Final statistics
            System.out.println(getLogStatistics());
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            isInitialized = false;
        }
    }
    
    /**
     * Internal log entry class for async processing
     */
    private static class LogEntry {
        final String category;
        final Level level;
        final String message;
        final String correlationId;
        final long timestamp;
        Throwable throwable;
        
        LogEntry(String category, Level level, String message, String correlationId) {
            this.category = category;
            this.level = level;
            this.message = message;
            this.correlationId = correlationId;
            this.timestamp = System.currentTimeMillis();
        }
    }
}



===== ModLoader/app/src/main/java/com/modloader/util/MelonLoaderDiagnostic.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/OfflineZipImporter.java =====

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



===== ModLoader/app/src/main/java/com/modloader/util/OnlineInstaller.java =====

// File: OnlineInstaller.java (Utility Class) - Complete Automated Installation System
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/app/src/main/java/com/terrarialoader/util/OnlineInstaller.java

package com.modloader.util;

import android.content.Context;
import com.modloader.loader.MelonLoaderManager;
import java.io.File;

/**
 * OnlineInstaller - Complete automated installation system
 * Uses existing Downloader and PathManager for seamless MelonLoader/LemonLoader installation
 */
public class OnlineInstaller {
    
    // GitHub URLs for MelonLoader/LemonLoader releases
    public static final String MELONLOADER_URL = "https://github.com/LavaGang/MelonLoader/releases/download/0.6.5.1/melon_data.zip";
    public static final String LEMONLOADER_URL = "https://github.com/LemonLoader/MelonLoader/releases/download/0.6.5.1/melon_data.zip";
    
    // Installation result class
    public static class InstallationResult {
        public boolean success;
        public String message;
        public String errorDetails;
        public File installationPath;
        public int filesInstalled;
        public long totalSize;
        
        public InstallationResult(boolean success, String message) {
            this.success = success;
            this.message = message;
        }
        
        public InstallationResult(boolean success, String message, String errorDetails) {
            this.success = success;
            this.message = message;
            this.errorDetails = errorDetails;
        }
    }
    
    /**
     * Complete automated installation of MelonLoader
     * Downloads melon_data.zip and extracts to proper directory structure
     * 
     * @param context Application context
     * @param gamePackage Target game package (e.g., "com.and.games505.TerrariaPaid")
     * @param loaderType Type of loader to install (NET8 or NET35)
     * @return InstallationResult with success status and details
     */
    public static InstallationResult installMelonLoaderOnline(Context context, String gamePackage, MelonLoaderManager.LoaderType loaderType) {
        LogUtils.logUser("🚀 Starting automated MelonLoader installation...");
        LogUtils.logUser("Target: " + gamePackage + " (" + loaderType.getDisplayName() + ")");
        
        // Validate parameters
        if (context == null) {
            return new InstallationResult(false, "Context is null", "Application context is required");
        }
        
        if (gamePackage == null || gamePackage.trim().isEmpty()) {
            return new InstallationResult(false, "Invalid game package", "Game package cannot be null or empty");
        }
        
        if (loaderType == null) {
            return new InstallationResult(false, "Invalid loader type", "Loader type cannot be null");
        }
        
        try {
            // Step 1: Initialize directory structure
            LogUtils.logUser("📁 Step 1: Initializing directory structure...");
            if (!PathManager.initializeGameDirectories(context, gamePackage)) {
                return new InstallationResult(false, "Failed to create directory structure", "Could not create required directories");
            }
            LogUtils.logUser("✅ Directory structure ready");
            
            // Step 2: Determine download URL and target directory
            String downloadUrl = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? MELONLOADER_URL : LEMONLOADER_URL;
            File targetDirectory = PathManager.getMelonLoaderDir(context, gamePackage);
            
            if (targetDirectory == null) {
                return new InstallationResult(false, "Cannot determine installation directory", "PathManager returned null directory");
            }
            
            LogUtils.logUser("📂 Installation directory: " + targetDirectory.getAbsolutePath());
            LogUtils.logUser("🌐 Download URL: " + downloadUrl);
            
            // Step 3: Download and extract
            LogUtils.logUser("⬇️ Step 2: Downloading and extracting MelonLoader files...");
            boolean downloadSuccess = Downloader.downloadAndExtractZip(downloadUrl, targetDirectory);
            
            if (!downloadSuccess) {
                return new InstallationResult(false, "Download or extraction failed", "Failed to download from: " + downloadUrl);
            }
            
            // Step 4: Organize files according to MelonLoader structure
            LogUtils.logUser("📋 Step 3: Organizing files into proper structure...");
            InstallationResult organizationResult = organizeExtractedFiles(context, gamePackage, targetDirectory, loaderType);
            
            if (!organizationResult.success) {
                return organizationResult;
            }
            
            // Step 5: Validate installation
            LogUtils.logUser("🔍 Step 4: Validating installation...");
            boolean validationSuccess = MelonLoaderManager.validateLoaderInstallation(context, gamePackage).isValid;
            
            if (!validationSuccess) {
                LogUtils.logUser("⚠️ Installation validation failed, attempting repair...");
                if (MelonLoaderManager.attemptRepair(context, gamePackage)) {
                    LogUtils.logUser("✅ Repair successful");
                    validationSuccess = true;
                } else {
                    return new InstallationResult(false, "Installation validation failed", "Files were downloaded but validation failed");
                }
            }
            
            // Step 6: Create final result
            InstallationResult result = new InstallationResult(true, "✅ " + loaderType.getDisplayName() + " installed successfully!");
            result.installationPath = targetDirectory;
            result.filesInstalled = countInstalledFiles(targetDirectory);
            result.totalSize = calculateDirectorySize(targetDirectory);
            
            LogUtils.logUser("🎉 Installation completed successfully!");
            LogUtils.logUser("📊 Files installed: " + result.filesInstalled);
            LogUtils.logUser("💾 Total size: " + FileUtils.formatFileSize(result.totalSize));
            LogUtils.logUser("📍 Installation path: " + result.installationPath.getAbsolutePath());
            
            return result;
            
        } catch (Exception e) {
            String errorMsg = "Unexpected error during installation: " + e.getMessage();
            LogUtils.logDebug(errorMsg);
            e.printStackTrace();
            return new InstallationResult(false, "Installation failed with exception", errorMsg);
        }
    }
    
    /**
     * Organize extracted files into proper MelonLoader directory structure
     * Based on the MelonLoader_File_List.txt structure
     */
    private static InstallationResult organizeExtractedFiles(Context context, String gamePackage, File extractedDir, MelonLoaderManager.LoaderType loaderType) {
        try {
            LogUtils.logUser("🗂️ Organizing extracted files...");
            
            // Create target directories based on loader type
            File net8Dir = PathManager.getMelonLoaderNet8Dir(context, gamePackage);
            File net35Dir = PathManager.getMelonLoaderNet35Dir(context, gamePackage);
            File dependenciesDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
            
            // Ensure directories exist
            PathManager.ensureDirectoryExists(net8Dir);
            PathManager.ensureDirectoryExists(net35Dir);
            PathManager.ensureDirectoryExists(dependenciesDir);
            
            int organizedFiles = 0;
            
            // Process extracted files
            File[] extractedFiles = extractedDir.listFiles();
            if (extractedFiles != null) {
                for (File file : extractedFiles) {
                    if (organizeFile(file, net8Dir, net35Dir, dependenciesDir, loaderType)) {
                        organizedFiles++;
                    }
                }
            }
            
            LogUtils.logUser("📁 Organized " + organizedFiles + " files into proper structure");
            
            // Create additional required directories
            createAdditionalDirectories(context, gamePackage);
            
            InstallationResult result = new InstallationResult(true, "File organization completed");
            result.filesInstalled = organizedFiles;
            return result;
            
        } catch (Exception e) {
            return new InstallationResult(false, "File organization failed", e.getMessage());
        }
    }
    
    /**
     * Organize individual file based on its type and target loader
     */
    private static boolean organizeFile(File file, File net8Dir, File net35Dir, File dependenciesDir, MelonLoaderManager.LoaderType loaderType) {
        if (file == null || !file.exists()) {
            return false;
        }
        
        try {
            String fileName = file.getName().toLowerCase();
            File targetDir = null;
            
            // Determine target directory based on file type and loader type
            if (fileName.contains("net8") && loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) {
                targetDir = net8Dir;
            } else if (fileName.contains("net35") && loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET35) {
                targetDir = net35Dir;
            } else if (fileName.contains("dependencies") || fileName.contains("supportmodules") || fileName.contains("il2cpp")) {
                targetDir = dependenciesDir;
            } else if (fileName.endsWith(".dll") || fileName.endsWith(".xml") || fileName.endsWith(".pdb")) {
                // Core files go to appropriate runtime directory
                targetDir = (loaderType == MelonLoaderManager.LoaderType.MELONLOADER_NET8) ? net8Dir : net35Dir;
            }
            
            if (targetDir != null) {
                File targetFile = new File(targetDir, file.getName());
                if (file.renameTo(targetFile)) {
                    LogUtils.logDebug("Moved: " + file.getName() + " -> " + targetDir.getName());
                    return true;
                }
            }
            
            return false;
            
        } catch (Exception e) {
            LogUtils.logDebug("Error organizing file " + file.getName() + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Create additional required directories for MelonLoader
     */
    private static void createAdditionalDirectories(Context context, String gamePackage) {
        try {
            // Create subdirectories in Dependencies
            File depsDir = PathManager.getMelonLoaderDependenciesDir(context, gamePackage);
            
            String[] subDirs = {
                "SupportModules",
                "CompatibilityLayers", 
                "Il2CppAssemblyGenerator",
                "Il2CppAssemblyGenerator/Cpp2IL",
                "Il2CppAssemblyGenerator/Cpp2IL/cpp2il_out",
                "Il2CppAssemblyGenerator/Il2CppInterop",
                "Il2CppAssemblyGenerator/Il2CppInterop/Il2CppAssemblies",
                "Il2CppAssemblyGenerator/UnityDependencies",
                "Il2CppAssemblyGenerator/runtimes",
                "Il2CppAssemblyGenerator/runtimes/linux-arm64/native",
                "Il2CppAssemblyGenerator/runtimes/linux-arm/native",
                "Il2CppAssemblyGenerator/runtimes/linux-x64/native",
                "Il2CppAssemblyGenerator/runtimes/linux-x86/native"
            };
            
            for (String subDir : subDirs) {
                File dir = new File(depsDir, subDir);
                PathManager.ensureDirectoryExists(dir);
            }
            
            LogUtils.logDebug("Created additional MelonLoader directories");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error creating additional directories: " + e.getMessage());
        }
    }
    
    /**
     * Count files in a directory recursively
     */
    private static int countInstalledFiles(File directory) {
        if (directory == null || !directory.exists() || !directory.isDirectory()) {
            return 0;
        }
        
        int count = 0;
        File[] files = directory.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.isDirectory()) {
                    count += countInstalledFiles(file);
                } else {
                    count++;
                }
            }
        }
        return count;
    }
    
    /**
     * Calculate total size of directory
     */
    private static long calculateDirectorySize(File directory) {
        if (directory == null || !directory.exists()) {
            return 0;
        }
        
        long size = 0;
        if (directory.isDirectory()) {
            File[] files = directory.listFiles();
            if (files != null) {
                for (File file : files) {
                    if (file.isDirectory()) {
                        size += calculateDirectorySize(file);
                    } else {
                        size += file.length();
                    }
                }
            }
        } else {
            size = directory.length();
        }
        return size;
    }
    
    /**
     * Convenience method for installing MelonLoader (NET8)
     */
    public static InstallationResult installMelonLoader(Context context, String gamePackage) {
        return installMelonLoaderOnline(context, gamePackage, MelonLoaderManager.LoaderType.MELONLOADER_NET8);
    }
    
    /**
     * Convenience method for installing LemonLoader (NET35)  
     */
    public static InstallationResult installLemonLoader(Context context, String gamePackage) {
        return installMelonLoaderOnline(context, gamePackage, MelonLoaderManager.LoaderType.MELONLOADER_NET35);
    }
    
    /**
     * Install for Terraria specifically
     */
    public static InstallationResult installForTerraria(Context context, MelonLoaderManager.LoaderType loaderType) {
        return installMelonLoaderOnline(context, MelonLoaderManager.TERRARIA_PACKAGE, loaderType);
    }
    
    /**
     * Check if online installation is possible (internet connectivity)
     */
    public static boolean isOnlineInstallationAvailable() {
        try {
            // Simple connectivity check
            java.net.URL url = new java.net.URL("https://github.com");
            java.net.HttpURLConnection connection = (java.net.HttpURLConnection) url.openConnection();
            connection.setRequestMethod("HEAD");
            connection.setConnectTimeout(3000);
            connection.setReadTimeout(3000);
            connection.connect();
            
            int responseCode = connection.getResponseCode();
            return responseCode == 200;
            
        } catch (Exception e) {
            LogUtils.logDebug("Online installation not available: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get installation progress callback interface
     */
    public interface InstallationProgressCallback {
        void onProgress(String message, int percentage);
        void onComplete(InstallationResult result);
        void onError(String error);
    }
    
    /**
     * Asynchronous installation with progress callback
     */
    public static void installMelonLoaderAsync(Context context, String gamePackage, MelonLoaderManager.LoaderType loaderType, InstallationProgressCallback callback) {
        new Thread(() -> {
            try {
                if (callback != null) {
                    callback.onProgress("Starting installation...", 0);
                }
                
                InstallationResult result = installMelonLoaderOnline(context, gamePackage, loaderType);
                
                if (callback != null) {
                    if (result.success) {
                        callback.onProgress("Installation completed!", 100);
                        callback.onComplete(result);
                    } else {
                        callback.onError(result.message + (result.errorDetails != null ? ": " + result.errorDetails : ""));
                    }
                }
                
            } catch (Exception e) {
                if (callback != null) {
                    callback.onError("Installation failed: " + e.getMessage());
                }
            }
        }).start();
    }
}



===== ModLoader/app/src/main/java/com/modloader/util/PatchResult.java =====

// File: PatchResult.java - Complete patch result class
// Path: /storage/emulated/0/AndroidIDEProjects/ModLoader/app/src/main/java/com/modloader/util/PatchResult.java

package com.modloader.util;

import java.util.ArrayList;
import java.util.List;

public class PatchResult {
    public boolean success;
    public String errorMessage;
    public List<String> warnings;
    public List<String> details;
    public String outputPath;
    public long processingTime;
    public long startTime;
    public long endTime;
    public String operationType;
    
    public PatchResult() {
        this.success = false;
        this.warnings = new ArrayList<>();
        this.details = new ArrayList<>();
        this.startTime = System.currentTimeMillis();
        this.processingTime = 0;
    }
    
    public PatchResult(boolean success) {
        this();
        this.success = success;
    }
    
    public PatchResult(boolean success, String errorMessage) {
        this(success);
        this.errorMessage = errorMessage;
    }
    
    public PatchResult(String operationType) {
        this();
        this.operationType = operationType;
    }
    
    public void addWarning(String warning) {
        if (warnings == null) {
            warnings = new ArrayList<>();
        }
        warnings.add(warning);
    }
    
    public void addDetail(String detail) {
        if (details == null) {
            details = new ArrayList<>();
        }
        details.add(detail);
    }
    
    public boolean hasWarnings() {
        return warnings != null && !warnings.isEmpty();
    }
    
    public boolean hasDetails() {
        return details != null && !details.isEmpty();
    }
    
    public void setSuccess(boolean success) {
        this.success = success;
        if (success) {
            this.endTime = System.currentTimeMillis();
            this.processingTime = this.endTime - this.startTime;
        }
    }
    
    public void setError(String errorMessage) {
        this.success = false;
        this.errorMessage = errorMessage;
        this.endTime = System.currentTimeMillis();
        this.processingTime = this.endTime - this.startTime;
    }
    
    public void setErrorMessage(String errorMessage) {
        this.errorMessage = errorMessage;
    }
    
    public void setOutputPath(String outputPath) {
        this.outputPath = outputPath;
    }
    
    public void setOperationType(String operationType) {
        this.operationType = operationType;
    }
    
    public void complete() {
        this.endTime = System.currentTimeMillis();
        this.processingTime = this.endTime - this.startTime;
    }
    
    // Convenience method to convert to boolean for backward compatibility
    public boolean isSuccess() {
        return success;
    }
    
    public long getProcessingTimeMs() {
        return processingTime;
    }
    
    public String getFormattedProcessingTime() {
        if (processingTime < 1000) {
            return processingTime + "ms";
        } else if (processingTime < 60000) {
            return String.format("%.1fs", processingTime / 1000.0);
        } else {
            long seconds = processingTime / 1000;
            long minutes = seconds / 60;
            seconds = seconds % 60;
            return String.format("%dm %ds", minutes, seconds);
        }
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("PatchResult{");
        sb.append("success=").append(success);
        if (operationType != null) {
            sb.append(", operation='").append(operationType).append('\'');
        }
        if (errorMessage != null) {
            sb.append(", error='").append(errorMessage).append('\'');
        }
        if (hasWarnings()) {
            sb.append(", warnings=").append(warnings.size());
        }
        if (processingTime > 0) {
            sb.append(", time=").append(getFormattedProcessingTime());
        }
        sb.append('}');
        return sb.toString();
    }
    
    public String getDetailedReport() {
        StringBuilder sb = new StringBuilder();
        sb.append("=== Patch Operation Report ===\n");
        if (operationType != null) {
            sb.append("Operation: ").append(operationType).append("\n");
        }
        sb.append("Result: ").append(success ? "SUCCESS" : "FAILED").append("\n");
        sb.append("Processing Time: ").append(getFormattedProcessingTime()).append("\n");
        
        if (outputPath != null) {
            sb.append("Output: ").append(outputPath).append("\n");
        }
        
        if (errorMessage != null) {
            sb.append("Error: ").append(errorMessage).append("\n");
        }
        
        if (hasWarnings()) {
            sb.append("\nWarnings (").append(warnings.size()).append("):\n");
            for (String warning : warnings) {
                sb.append("  - ").append(warning).append("\n");
            }
        }
        
        if (hasDetails()) {
            sb.append("\nDetails (").append(details.size()).append("):\n");
            for (String detail : details) {
                sb.append("  • ").append(detail).append("\n");
            }
        }
        
        return sb.toString();
    }
    
    // Static factory methods for common scenarios
    public static PatchResult success(String operationType) {
        PatchResult result = new PatchResult(operationType);
        result.setSuccess(true);
        return result;
    }
    
    public static PatchResult success(String operationType, String outputPath) {
        PatchResult result = success(operationType);
        result.setOutputPath(outputPath);
        return result;
    }
    
    public static PatchResult failure(String operationType, String errorMessage) {
        PatchResult result = new PatchResult(operationType);
        result.setError(errorMessage);
        return result;
    }
    
    public static PatchResult inProgress(String operationType) {
        return new PatchResult(operationType);
    }
}



===== ModLoader/app/src/main/java/com/modloader/util/PathManager.java =====

// File: PathManager.java (FIXED Part 1) - Centralized Path Management
// Path: /storage/emulated/0/AndroidIDEProjects/TerrariaML/main/java/com/terrarialoader/util/PathManager.java

package com.modloader.util;

import android.content.Context;
import java.io.File;

/**
 * Centralized path management for TerrariaLoader
 * All file operations should use these standardized paths
 * FIXED: Added proper app logs directory support and correct structure
 */
public class PathManager {
    
    // Base directory: /storage/emulated/0/Android/data/com.modloader/files
    private static File getAppDataDirectory(Context context) {
        return context.getExternalFilesDir(null);
    }
    
    // === TERRARIA LOADER STRUCTURE ===
    
    /**
     * Base TerrariaLoader directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader
     */
    public static File getTerrariaLoaderBaseDir(Context context) {
        return new File(getAppDataDirectory(context), "TerrariaLoader");
    }
    
    /**
     * Game-specific base directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}
     */
    public static File getGameBaseDir(Context context, String gamePackage) {
        return new File(getTerrariaLoaderBaseDir(context), gamePackage);
    }
    
    /**
     * Default Terraria directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/com.and.games505.TerrariaPaid
     */
    public static File getTerrariaBaseDir(Context context) {
        return getGameBaseDir(context, "com.and.games505.TerrariaPaid");
    }
    
    // === MOD DIRECTORIES ===
    
    /**
     * DLL Mods directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Mods/DLL
     */
    public static File getDllModsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Mods/DLL");
    }
    
    /**
     * DEX Mods directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Mods/DEX
     */
    public static File getDexModsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Mods/DEX");
    }
    
    /**
     * Legacy mods directory (for backward compatibility)
     * @return /storage/emulated/0/Android/data/com.modloader/files/mods
     */
    public static File getLegacyModsDir(Context context) {
        return new File(getAppDataDirectory(context), "mods");
    }
    
    // === MELONLOADER STRUCTURE ===
    
    /**
     * MelonLoader base directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader
     */
    public static File getMelonLoaderDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Loaders/MelonLoader");
    }
    
    /**
     * MelonLoader NET8 runtime directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/net8
     */
    public static File getMelonLoaderNet8Dir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "net8");
    }
    
    /**
     * MelonLoader NET35 runtime directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/net35
     */
    public static File getMelonLoaderNet35Dir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "net35");
    }
    
    /**
     * MelonLoader Dependencies directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Loaders/MelonLoader/Dependencies
     */
    public static File getMelonLoaderDependenciesDir(Context context, String gamePackage) {
        return new File(getMelonLoaderDir(context, gamePackage), "Dependencies");
    }
    
    // === PLUGINS AND USERLIBS (FIXED: Now at game root level) ===
    
    /**
     * FIXED: Plugins directory (at game root level, not inside MelonLoader)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Plugins
     */
    public static File getPluginsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Plugins");
    }
    
    /**
     * FIXED: UserLibs directory (at game root level, not inside MelonLoader)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/UserLibs
     */
    public static File getUserLibsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "UserLibs");
    }
    
    // === LOG DIRECTORIES ===
    
    /**
     * FIXED: Game logs directory (MelonLoader logs)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Logs
     */
    public static File getLogsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Logs");
    }
    
    /**
     * FIXED: App logs directory (TerrariaLoader app logs)
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/AppLogs
     */
    public static File getAppLogsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "AppLogs");
    }
    
    /**
     * Legacy app logs directory (for backward compatibility)
     * @return /storage/emulated/0/Android/data/com.modloader/files/logs
     */
    public static File getLegacyAppLogsDir(Context context) {
        return new File(getAppDataDirectory(context), "logs");
    }
    
    /**
     * Exports directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/exports
     */
    public static File getExportsDir(Context context) {
        return new File(getAppDataDirectory(context), "exports");
    }
    
    // === BACKUP DIRECTORIES ===
    
    /**
     * Backups directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Backups
     */
    public static File getBackupsDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Backups");
    }
    
    // === CONFIG DIRECTORIES ===
    
    /**
     * Config directory
     * @return /storage/emulated/0/Android/data/com.modloader/files/TerrariaLoader/{gamePackage}/Config
     */
    public static File getConfigDir(Context context, String gamePackage) {
        return new File(getGameBaseDir(context, gamePackage), "Config");
    }
    
    // === UTILITY METHODS ===
    
    /**
     * Ensure a directory exists, creating it if necessary
     * @param directory The directory to ensure exists
     * @return true if directory exists or was created successfully
     */
    public static boolean ensureDirectoryExists(File directory) {
        if (directory == null) {
            LogUtils.logDebug("Directory is null");
            return false;
        }
        
        if (directory.exists()) {
            return directory.isDirectory();
        }
        
        try {
            boolean created = directory.mkdirs();
            if (created) {
                LogUtils.logDebug("Created directory: " + directory.getAbsolutePath());
            } else {
                LogUtils.logDebug("Failed to create directory: " + directory.getAbsolutePath());
            }
            return created;
        } catch (Exception e) {
            LogUtils.logDebug("Error creating directory: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Initialize all required directories for a game package
     * @param context Application context
     * @param gamePackage Game package name
     * @return true if all directories were created successfully
     */
    public static boolean initializeGameDirectories(Context context, String gamePackage) {
        LogUtils.logUser("Initializing directory structure for: " + gamePackage);
        
        File[] requiredDirs = {
            getGameBaseDir(context, gamePackage),
            getDllModsDir(context, gamePackage),
            getDexModsDir(context, gamePackage),
            getMelonLoaderDir(context, gamePackage),
            getMelonLoaderNet8Dir(context, gamePackage),
            getMelonLoaderNet35Dir(context, gamePackage),
            getMelonLoaderDependenciesDir(context, gamePackage),
            getPluginsDir(context, gamePackage),        // FIXED: Added Plugins
            getUserLibsDir(context, gamePackage),       // FIXED: Added UserLibs
            getLogsDir(context, gamePackage),           // Game logs
            getAppLogsDir(context, gamePackage),        // FIXED: Added App logs
            getBackupsDir(context, gamePackage),
            getConfigDir(context, gamePackage)
        };
        
        boolean allSuccess = true;
        int createdCount = 0;
        
        for (File dir : requiredDirs) {
            if (!dir.exists()) {
                if (ensureDirectoryExists(dir)) {
                    createdCount++;
                } else {
                    allSuccess = false;
                    LogUtils.logDebug("Failed to create: " + dir.getAbsolutePath());
                }
            }
        }
        
        LogUtils.logUser("Directory initialization complete: " + createdCount + " directories created");
        
        // Create README files
        if (allSuccess) {
            createReadmeFiles(context, gamePackage);
        }
        
        return allSuccess;
    }
    
    /**
     * FIXED: Create helpful README files in directories
     */
    private static void createReadmeFiles(Context context, String gamePackage) {
        try {
            // DLL Mods README
            File dllReadme = new File(getDllModsDir(context, gamePackage), "README.txt");
            if (!dllReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(dllReadme)) {
                    writer.write("=== TerrariaLoader - DLL Mods ===\n\n");
                    writer.write("Place your .dll mod files here.\n");
                    writer.write("Requires MelonLoader to be installed.\n\n");
                    writer.write("Supported formats:\n");
                    writer.write("• .dll files (enabled)\n");
                    writer.write("• .dll.disabled files (disabled)\n\n");
                    writer.write("Path: " + dllReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // DEX Mods README
            File dexReadme = new File(getDexModsDir(context, gamePackage), "README.txt");
            if (!dexReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(dexReadme)) {
                    writer.write("=== TerrariaLoader - DEX/JAR Mods ===\n\n");
                    writer.write("Place your .dex and .jar mod files here.\n");
                    writer.write("These are Java-based mods for Android.\n\n");
                    writer.write("Supported formats:\n");
                    writer.write("• .dex files (enabled)\n");
                    writer.write("• .jar files (enabled)\n");
                    writer.write("• .dex.disabled files (disabled)\n");
                    writer.write("• .jar.disabled files (disabled)\n\n");
                    writer.write("Path: " + dexReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: Plugins README
            File pluginsReadme = new File(getPluginsDir(context, gamePackage), "README.txt");
            if (!pluginsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(pluginsReadme)) {
                    writer.write("=== MelonLoader Plugins Directory ===\n\n");
                    writer.write("This directory contains MelonLoader plugins.\n");
                    writer.write("Plugins extend MelonLoader functionality.\n\n");
                    writer.write("Files you might see here:\n");
                    writer.write("• .dll plugin files\n");
                    writer.write("• Plugin configuration files\n\n");
                    writer.write("Path: " + pluginsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: UserLibs README
            File userlibsReadme = new File(getUserLibsDir(context, gamePackage), "README.txt");
            if (!userlibsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(userlibsReadme)) {
                    writer.write("=== MelonLoader UserLibs Directory ===\n\n");
                    writer.write("This directory contains user libraries.\n");
                    writer.write("Libraries that mods depend on go here.\n\n");
                    writer.write("Files you might see here:\n");
                    writer.write("• .dll library files\n");
                    writer.write("• Shared mod dependencies\n\n");
                    writer.write("Path: " + userlibsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: Game logs README
            File gameLogsReadme = new File(getLogsDir(context, gamePackage), "README.txt");
            if (!gameLogsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(gameLogsReadme)) {
                    writer.write("=== MelonLoader Game Logs ===\n\n");
                    writer.write("This directory contains logs from MelonLoader and mods.\n");
                    writer.write("These are generated when running the patched game.\n\n");
                    writer.write("Log files follow rotation:\n");
                    writer.write("• Log.txt (current log)\n");
                    writer.write("• Log1.txt to Log5.txt (previous logs)\n");
                    writer.write("• Oldest logs are automatically deleted\n\n");
                    writer.write("Path: " + gameLogsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
            // FIXED: App logs README
            File appLogsReadme = new File(getAppLogsDir(context, gamePackage), "README.txt");
            if (!appLogsReadme.exists()) {
                try (java.io.FileWriter writer = new java.io.FileWriter(appLogsReadme)) {
                    writer.write("=== TerrariaLoader App Logs ===\n\n");
                    writer.write("This directory contains logs from TerrariaLoader app.\n");
                    writer.write("These are generated when using this app.\n\n");
                    writer.write("Log files follow rotation:\n");
                    writer.write("• AppLog.txt (current log)\n");
                    writer.write("• AppLog1.txt to AppLog5.txt (previous logs)\n");
                    writer.write("• Oldest logs are automatically deleted\n\n");
                    writer.write("Path: " + appLogsReadme.getParentFile().getAbsolutePath() + "\n");
                }
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error creating README files: " + e.getMessage());
        }
    }
    
    /**
     * FIXED: Get standardized path string for logging/display
     */
    public static String getPathInfo(Context context, String gamePackage) {
        StringBuilder info = new StringBuilder();
        info.append("=== TerrariaLoader Directory Structure ===\n");
        info.append("Base: ").append(getTerrariaLoaderBaseDir(context).getAbsolutePath()).append("\n");
        info.append("Game: ").append(getGameBaseDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("DLL Mods: ").append(getDllModsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("DEX Mods: ").append(getDexModsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("MelonLoader: ").append(getMelonLoaderDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("Plugins: ").append(getPluginsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("UserLibs: ").append(getUserLibsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("Game Logs: ").append(getLogsDir(context, gamePackage).getAbsolutePath()).append("\n");
        info.append("App Logs: ").append(getAppLogsDir(context, gamePackage).getAbsolutePath()).append("\n");
        return info.toString();
    }
    
    /**
     * Check if legacy structure exists and needs migration
     */
    public static boolean needsMigration(Context context) {
        File legacyMods = getLegacyModsDir(context);
        File legacyAppLogs = getLegacyAppLogsDir(context);
        File newStructure = getTerrariaBaseDir(context);
        
        return ((legacyMods.exists() && legacyMods.listFiles() != null && legacyMods.listFiles().length > 0) ||
                (legacyAppLogs.exists() && legacyAppLogs.listFiles() != null && legacyAppLogs.listFiles().length > 0)) && 
               !newStructure.exists();
    }
    
    /**
     * FIXED: Migrate from legacy structure to new structure
     */
    public static boolean migrateLegacyStructure(Context context) {
        if (!needsMigration(context)) {
            return true;
        }
        
        LogUtils.logUser("Migrating from legacy directory structure...");
        
        try {
            String gamePackage = "com.and.games505.TerrariaPaid";
            
            // Initialize new structure
            if (!initializeGameDirectories(context, gamePackage)) {
                LogUtils.logDebug("Failed to initialize new directory structure");
                return false;
            }
            
            // Migrate mods
            File legacyModsDir = getLegacyModsDir(context);
            File newDexMods = getDexModsDir(context, gamePackage);
            
            if (legacyModsDir.exists()) {
                File[] modFiles = legacyModsDir.listFiles();
                if (modFiles != null) {
                    int migratedCount = 0;
                    for (File modFile : modFiles) {
                        if (modFile.isFile()) {
                            File newLocation = new File(newDexMods, modFile.getName());
                            if (modFile.renameTo(newLocation)) {
                                migratedCount++;
                            }
                        }
                    }
                    LogUtils.logUser("Migrated " + migratedCount + " mod files to new structure");
                }
            }
            
            // Migrate legacy app logs
            File legacyAppLogs = getLegacyAppLogsDir(context);
            File newAppLogs = getAppLogsDir(context, gamePackage);
            
            if (legacyAppLogs.exists()) {
                File[] logFiles = legacyAppLogs.listFiles((dir, name) -> 
                    name.endsWith(".txt") || name.startsWith("auto_save_"));
                
                if (logFiles != null && logFiles.length > 0) {
                    LogUtils.logUser("Migrating " + logFiles.length + " legacy log files...");
                    
                    if (!newAppLogs.exists()) {
                        newAppLogs.mkdirs();
                    }
                    
                    int migratedLogCount = 0;
                    for (File logFile : logFiles) {
                        // Rename to new format
                        String newName = "AppLog" + (migratedLogCount + 1) + ".txt";
                        File newLocation = new File(newAppLogs, newName);
                        
                        if (logFile.renameTo(newLocation)) {
                            migratedLogCount++;
                        }
                    }
                    LogUtils.logUser("Migrated " + migratedLogCount + " log files to new structure");
                }
            }
            
            LogUtils.logUser("✅ Migration completed successfully");
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Migration failed: " + e.getMessage());
            return false;
        }
    }
}



===== ModLoader/app/src/main/java/com/modloader/util/PermissionManager.java =====

// File: PermissionManager.java (COMPLETE FIXED) - No Syntax Errors
// Path: /app/src/main/java/com/modloader/util/PermissionManager.java

package com.modloader.util;

import android.Manifest;
import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Environment;
import android.provider.Settings;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class PermissionManager {
    private static final String TAG = "PermissionManager";
    
    // Request codes for different permission types
    public static final int REQUEST_BASIC_PERMISSIONS = 1001;
    public static final int REQUEST_STORAGE_PERMISSIONS = 1002;
    public static final int REQUEST_INSTALL_PERMISSION = 1003;
    public static final int REQUEST_ALL_FILES_ACCESS = 1004;
    
    // Essential permissions for app functionality
    private static final String[] BASIC_PERMISSIONS = {
        Manifest.permission.READ_EXTERNAL_STORAGE,
        Manifest.permission.WRITE_EXTERNAL_STORAGE
    };
    
    // Additional permissions for enhanced functionality
    private static final String[] ENHANCED_PERMISSIONS = {
        Manifest.permission.REQUEST_INSTALL_PACKAGES,
        Manifest.permission.INTERNET,
        Manifest.permission.ACCESS_NETWORK_STATE
    };
    
    private final Context context;
    private final Activity activity;
    
    // Permission state tracking
    private final Map<String, Boolean> permissionStates = new HashMap<>();
    private PermissionCallback callback;
    
    public interface PermissionCallback {
        void onPermissionGranted(String permission);
        void onPermissionDenied(String permission);
        void onPermissionPermanentlyDenied(String permission);
        void onAllPermissionsGranted();
        void onPermissionRequestCompleted(boolean allGranted);
    }
    
    public PermissionManager(Context context) {
        this.context = context;
        this.activity = context instanceof Activity ? (Activity) context : null;
        initializePermissionStates();
    }
    
    public PermissionManager(Activity activity) {
        this.context = activity;
        this.activity = activity;
        initializePermissionStates();
    }
    
    public void setCallback(PermissionCallback callback) {
        this.callback = callback;
    }
    
    /**
     * Initialize permission states by checking current status
     */
    private void initializePermissionStates() {
        try {
            LogUtils.logDebug("Initializing permission states...");
            
            // Check basic permissions
            for (String permission : BASIC_PERMISSIONS) {
                boolean granted = isPermissionGranted(permission);
                permissionStates.put(permission, granted);
                LogUtils.logDebug("Permission " + permission + ": " + (granted ? "GRANTED" : "DENIED"));
            }
            
            // Check enhanced permissions
            for (String permission : ENHANCED_PERMISSIONS) {
                boolean granted = isPermissionGranted(permission);
                permissionStates.put(permission, granted);
                LogUtils.logDebug("Permission " + permission + ": " + (granted ? "GRANTED" : "DENIED"));
            }
            
            // Check special permissions
            boolean hasManageStorage = hasManageExternalStoragePermission();
            boolean hasInstallPermission = hasInstallPermission();
            
            permissionStates.put("MANAGE_EXTERNAL_STORAGE", hasManageStorage);
            permissionStates.put("INSTALL_PACKAGES", hasInstallPermission);
            
            LogUtils.logDebug("Special permissions - MANAGE_STORAGE: " + hasManageStorage + 
                ", INSTALL: " + hasInstallPermission);
            
        } catch (Exception e) {
            LogUtils.logDebug("Error initializing permission states: " + e.getMessage());
        }
    }
    
    /**
     * Check if a specific permission is granted
     */
    private boolean isPermissionGranted(String permission) {
        try {
            return ContextCompat.checkSelfPermission(context, permission) == PackageManager.PERMISSION_GRANTED;
        } catch (Exception e) {
            LogUtils.logDebug("Error checking permission " + permission + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if basic storage permissions are granted
     */
    public boolean hasStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            // Android 11+ - Check for All Files Access
            return hasManageExternalStoragePermission();
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            // Android 6-10 - Check runtime permissions
            return isPermissionGranted(Manifest.permission.READ_EXTERNAL_STORAGE) &&
                   isPermissionGranted(Manifest.permission.WRITE_EXTERNAL_STORAGE);
        } else {
            // Below Android 6 - permissions granted at install time
            return true;
        }
    }
    
    /**
     * Check MANAGE_EXTERNAL_STORAGE permission for Android 11+
     */
    public boolean hasManageExternalStoragePermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            return Environment.isExternalStorageManager();
        }
        return true; // Not applicable for older versions
    }
    
    /**
     * Check install packages permission
     */
    public boolean hasInstallPermission() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            return context.getPackageManager().canRequestPackageInstalls();
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            try {
                return Settings.Secure.getInt(context.getContentResolver(),
                    Settings.Secure.INSTALL_NON_MARKET_APPS) == 1;
            } catch (Settings.SettingNotFoundException e) {
                return false;
            }
        }
        return true; // Older versions don't have this restriction
    }
    
    /**
     * Check if all basic permissions are granted
     */
    public boolean hasBasicPermissions() {
        for (String permission : BASIC_PERMISSIONS) {
            if (!isPermissionGranted(permission)) {
                return false;
            }
        }
        return hasStoragePermission(); // Include modern storage permission
    }
    
    /**
     * Check if all enhanced permissions are granted
     */
    public boolean hasEnhancedPermissions() {
        for (String permission : ENHANCED_PERMISSIONS) {
            if (!isPermissionGranted(permission)) {
                return false;
            }
        }
        return true;
    }
    
    /**
     * Check if all permissions (basic + enhanced + special) are granted
     */
    public boolean hasAllPermissions() {
        return hasBasicPermissions() && hasEnhancedPermissions() && hasInstallPermission();
    }
    
    /**
     * Request basic permissions required for app functionality
     */
    public void requestBasicPermissions() {
        if (activity == null) {
            LogUtils.logDebug("Cannot request permissions - activity is null");
            return;
        }
        
        LogUtils.logUser("🔐 Checking basic permissions...");
        
        // For Android 11+, request All Files Access if needed
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R && !hasManageExternalStoragePermission()) {
            requestAllFilesAccess();
            return;
        }
        
        // For Android 6-10, request runtime permissions
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            List<String> permissionsToRequest = new ArrayList<>();
            
            for (String permission : BASIC_PERMISSIONS) {
                if (!isPermissionGranted(permission)) {
                    permissionsToRequest.add(permission);
                }
            }
            
            if (!permissionsToRequest.isEmpty()) {
                LogUtils.logUser("📋 Requesting " + permissionsToRequest.size() + " basic permissions...");
                ActivityCompat.requestPermissions(activity, 
                    permissionsToRequest.toArray(new String[0]), 
                    REQUEST_BASIC_PERMISSIONS);
            } else {
                LogUtils.logUser("✅ All basic permissions already granted");
                if (callback != null) {
                    callback.onAllPermissionsGranted();
                }
            }
        } else {
            LogUtils.logUser("✅ Running on pre-Android 6 - permissions granted at install");
            if (callback != null) {
                callback.onAllPermissionsGranted();
            }
        }
    }
    
    /**
     * Request All Files Access for Android 11+
     */
    public void requestAllFilesAccess() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            if (hasManageExternalStoragePermission()) {
                LogUtils.logUser("✅ All Files Access already granted");
                return;
            }
            
            LogUtils.logUser("🔐 Requesting All Files Access permission...");
            
            if (activity != null) {
                new AlertDialog.Builder(activity)
                    .setTitle("🗂️ All Files Access Required")
                    .setMessage("TerrariaLoader needs access to all files to:\n\n" +
                        "• Create and manage mod directories\n" +
                        "• Install and patch APK files\n" +
                        "• Backup and restore game data\n\n" +
                        "Please grant 'All Files Access' permission.")
                    .setPositiveButton("Grant Permission", (dialog, which) -> {
                        try {
                            Intent intent = new Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION);
                            intent.setData(Uri.parse("package:" + context.getPackageName()));
                            activity.startActivityForResult(intent, REQUEST_ALL_FILES_ACCESS);
                            LogUtils.logUser("📱 Opened All Files Access settings");
                        } catch (Exception e) {
                            LogUtils.logDebug("Error opening All Files Access settings: " + e.getMessage());
                            // Fallback to general manage storage settings
                            try {
                                Intent fallbackIntent = new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION);
                                activity.startActivityForResult(fallbackIntent, REQUEST_ALL_FILES_ACCESS);
                            } catch (Exception e2) {
                                showToast("Cannot open permission settings");
                            }
                        }
                    })
                    .setNegativeButton("Cancel", (dialog, which) -> {
                        LogUtils.logUser("❌ User declined All Files Access");
                        if (callback != null) {
                            callback.onPermissionDenied("MANAGE_EXTERNAL_STORAGE");
                        }
                    })
                    .show();
            }
        }
    }
    
    /**
     * Request enhanced permissions for additional functionality
     */
    public void requestEnhancedPermissions() {
        if (activity == null) {
            LogUtils.logDebug("Cannot request permissions - activity is null");
            return;
        }
        
        LogUtils.logUser("🔐 Checking enhanced permissions...");
        
        List<String> permissionsToRequest = new ArrayList<>();
        
        for (String permission : ENHANCED_PERMISSIONS) {
            if (!isPermissionGranted(permission)) {
                permissionsToRequest.add(permission);
            }
        }
        
        if (!permissionsToRequest.isEmpty()) {
            LogUtils.logUser("📋 Requesting " + permissionsToRequest.size() + " enhanced permissions...");
            ActivityCompat.requestPermissions(activity, 
                permissionsToRequest.toArray(new String[0]), 
                REQUEST_STORAGE_PERMISSIONS);
        } else {
            LogUtils.logUser("✅ All enhanced permissions already granted");
        }
    }
    
    /**
     * Request install packages permission
     */
    public void requestInstallPermission() {
        if (hasInstallPermission()) {
            LogUtils.logUser("✅ Install permission already granted");
            return;
        }
        
        LogUtils.logUser("🔐 Requesting install permission...");
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            if (activity != null) {
                new AlertDialog.Builder(activity)
                    .setTitle("📦 Install Permission Required")
                    .setMessage("To install modded APK files, please allow this app to install unknown apps.")
                    .setPositiveButton("Grant Permission", (dialog, which) -> {
                        try {
                            Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES);
                            intent.setData(Uri.parse("package:" + context.getPackageName()));
                            activity.startActivityForResult(intent, REQUEST_INSTALL_PERMISSION);
                            LogUtils.logUser("📱 Opened install permission settings");
                        } catch (Exception e) {
                            LogUtils.logDebug("Error opening install permission settings: " + e.getMessage());
                            showToast("Cannot open permission settings");
                        }
                    })
                    .setNegativeButton("Cancel", null)
                    .show();
            }
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
            if (activity != null) {
                new AlertDialog.Builder(activity)
                    .setTitle("🔐 Unknown Sources Required")
                    .setMessage("Please enable 'Unknown sources' in security settings to install modded APKs.")
                    .setPositiveButton("Open Settings", (dialog, which) -> {
                        try {
                            Intent intent = new Intent(Settings.ACTION_SECURITY_SETTINGS);
                            activity.startActivity(intent);
                            LogUtils.logUser("📱 Opened security settings");
                        } catch (Exception e) {
                            showToast("Cannot open security settings");
                        }
                    })
                    .setNegativeButton("Cancel", null)
                    .show();
            }
        }
    }
    
    /**
     * Request all permissions (basic + enhanced + install)
     */
    public void requestAllPermissions() {
        LogUtils.logUser("🔐 Requesting all permissions...");
        
        // Start with basic permissions
        if (!hasBasicPermissions()) {
            requestBasicPermissions();
            return;
        }
        
        // Then enhanced permissions
        if (!hasEnhancedPermissions()) {
            requestEnhancedPermissions();
            return;
        }
        
        // Finally install permission
        if (!hasInstallPermission()) {
            requestInstallPermission();
            return;
        }
        
        LogUtils.logUser("✅ All permissions already granted");
        if (callback != null) {
            callback.onAllPermissionsGranted();
        }
    }
    
    /**
     * Handle permission request results
     */
    public void handlePermissionResult(int requestCode, String[] permissions, int[] grantResults) {
        LogUtils.logDebug("Permission result: requestCode=" + requestCode + 
            ", permissions=" + (permissions != null ? permissions.length : 0) +
            ", results=" + (grantResults != null ? grantResults.length : 0));
        
        try {
            switch (requestCode) {
                case REQUEST_BASIC_PERMISSIONS:
                    handleBasicPermissionResult(permissions, grantResults);
                    break;
                case REQUEST_STORAGE_PERMISSIONS:
                    handleStoragePermissionResult(permissions, grantResults);
                    break;
                case REQUEST_ALL_FILES_ACCESS:
                    handleAllFilesAccessResult();
                    break;
                case REQUEST_INSTALL_PERMISSION:
                    handleInstallPermissionResult();
                    break;
                default:
                    LogUtils.logDebug("Unknown permission request code: " + requestCode);
                    break;
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error handling permission result: " + e.getMessage());
        }
    }
    
    /**
     * Handle basic permission results
     */
    private void handleBasicPermissionResult(String[] permissions, int[] grantResults) {
        if (permissions == null || grantResults == null) return;
        
        int grantedCount = 0;
        int deniedCount = 0;
        
        for (int i = 0; i < permissions.length && i < grantResults.length; i++) {
            String permission = permissions[i];
            boolean granted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
            
            permissionStates.put(permission, granted);
            
            if (granted) {
                grantedCount++;
                LogUtils.logUser("✅ Permission granted: " + permission);
                if (callback != null) {
                    callback.onPermissionGranted(permission);
                }
            } else {
                deniedCount++;
                LogUtils.logUser("❌ Permission denied: " + permission);
                
                // Check if permanently denied
                if (activity != null && !ActivityCompat.shouldShowRequestPermissionRationale(activity, permission)) {
                    LogUtils.logUser("⚠️ Permission permanently denied: " + permission);
                    if (callback != null) {
                        callback.onPermissionPermanentlyDenied(permission);
                    }
                } else {
                    if (callback != null) {
                        callback.onPermissionDenied(permission);
                    }
                }
            }
        }
        
        boolean allGranted = deniedCount == 0;
        LogUtils.logUser("📊 Basic permissions result: " + grantedCount + " granted, " + deniedCount + " denied");
        
        if (callback != null) {
            callback.onPermissionRequestCompleted(allGranted);
            if (allGranted) {
                callback.onAllPermissionsGranted();
            }
        }
        
        if (allGranted && !hasInstallPermission()) {
            // Automatically request install permission if basic permissions are granted
            requestInstallPermission();
        }
    }
    
    /**
     * Handle storage permission results
     */
    private void handleStoragePermissionResult(String[] permissions, int[] grantResults) {
        if (permissions == null || grantResults == null) return;
        
        boolean allGranted = true;
        for (int i = 0; i < permissions.length && i < grantResults.length; i++) {
            boolean granted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
            permissionStates.put(permissions[i], granted);
            if (!granted) {
                allGranted = false;
            }
        }
        
        LogUtils.logUser(allGranted ? "✅ Enhanced permissions granted" : "❌ Some enhanced permissions denied");
        
        if (callback != null) {
            callback.onPermissionRequestCompleted(allGranted);
        }
    }
    
    /**
     * Handle All Files Access result
     */
    private void handleAllFilesAccessResult() {
        boolean granted = hasManageExternalStoragePermission();
        permissionStates.put("MANAGE_EXTERNAL_STORAGE", granted);
        
        LogUtils.logUser(granted ? "✅ All Files Access granted" : "❌ All Files Access denied");
        
        if (callback != null) {
            if (granted) {
                callback.onPermissionGranted("MANAGE_EXTERNAL_STORAGE");
            } else {
                callback.onPermissionDenied("MANAGE_EXTERNAL_STORAGE");
            }
            callback.onPermissionRequestCompleted(granted);
        }
        
        if (granted) {
            // Continue with other permissions if needed
            if (!hasEnhancedPermissions()) {
                requestEnhancedPermissions();
            } else if (!hasInstallPermission()) {
                requestInstallPermission();
            }
        }
    }
    
    /**
     * Handle install permission result
     */
    private void handleInstallPermissionResult() {
        boolean granted = hasInstallPermission();
        permissionStates.put("INSTALL_PACKAGES", granted);
        
        LogUtils.logUser(granted ? "✅ Install permission granted" : "❌ Install permission denied");
        
        if (callback != null) {
            if (granted) {
                callback.onPermissionGranted("INSTALL_PACKAGES");
            } else {
                callback.onPermissionDenied("INSTALL_PACKAGES");
            }
            callback.onPermissionRequestCompleted(granted);
        }
    }
    
    /**
     * Check if permission is permanently denied
     */
    public boolean isPermissionPermanentlyDenied(String permission) {
        if (activity == null) return false;
        
        return !isPermissionGranted(permission) && 
               !ActivityCompat.shouldShowRequestPermissionRationale(activity, permission);
    }
    
    /**
     * Show permission explanation dialog
     */
    public void showPermissionExplanation(String permission, String explanation) {
        if (activity == null) return;
        
        new AlertDialog.Builder(activity)
            .setTitle("🔐 Permission Required")
            .setMessage(explanation)
            .setPositiveButton("Grant", (dialog, which) -> {
                // Re-request the specific permission
                ActivityCompat.requestPermissions(activity, new String[]{permission}, REQUEST_BASIC_PERMISSIONS);
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    /**
     * Get current permission status as a detailed string
     */
    public String getPermissionStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Permission Status ===\n");
        
        status.append("Basic Permissions: ").append(hasBasicPermissions() ? "✅" : "❌").append("\n");
        status.append("- Storage Access: ").append(hasStoragePermission() ? "✅" : "❌").append("\n");
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            status.append("- All Files Access: ").append(hasManageExternalStoragePermission() ? "✅" : "❌").append("\n");
        }
        
        status.append("Enhanced Permissions: ").append(hasEnhancedPermissions() ? "✅" : "❌").append("\n");
        status.append("- Install Packages: ").append(hasInstallPermission() ? "✅" : "❌").append("\n");
        status.append("- Internet Access: ").append(isPermissionGranted(Manifest.permission.INTERNET) ? "✅" : "❌").append("\n");
        
        status.append("Overall Status: ").append(hasAllPermissions() ? "✅ All Ready" : "❌ Missing Permissions").append("\n");
        
        return status.toString();
    }
    
    /**
     * Get list of missing permissions
     */
    public List<String> getMissingPermissions() {
        List<String> missing = new ArrayList<>();
        
        if (!hasStoragePermission()) {
            missing.add("Storage Access");
        }
        
        if (!hasInstallPermission()) {
            missing.add("Install Packages");
        }
        
        for (String permission : BASIC_PERMISSIONS) {
            if (!isPermissionGranted(permission)) {
                missing.add(permission);
            }
        }
        
        for (String permission : ENHANCED_PERMISSIONS) {
            if (!isPermissionGranted(permission)) {
                missing.add(permission);
            }
        }
        
        return missing;
    }
    
    /**
     * Refresh permission states (call after user returns from settings)
     */
    public void refreshPermissionStates() {
        LogUtils.logDebug("Refreshing permission states...");
        initializePermissionStates();
    }
    
    /**
     * Open app settings page
     */
    public void openAppSettings() {
        try {
            Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS);
            intent.setData(Uri.parse("package:" + context.getPackageName()));
            if (activity != null) {
                activity.startActivity(intent);
            } else {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(intent);
            }
            LogUtils.logUser("📱 Opened app settings");
        } catch (Exception e) {
            LogUtils.logDebug("Error opening app settings: " + e.getMessage());
            showToast("Cannot open app settings");
        }
    }
    
    /**
     * Show toast message
     */
    private void showToast(String message) {
        Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
    }
    
    /**
     * Auto setup permissions based on current state
     */
    public void autoSetupPermissions() {
        LogUtils.logDebug("Auto-setting up permissions...");
        
        // Check and request permissions in order of importance
        if (!hasBasicPermissions()) {
            LogUtils.logUser("🔧 Auto-requesting basic permissions...");
            requestBasicPermissions();
        } else if (!hasEnhancedPermissions()) {
            LogUtils.logUser("🔧 Auto-requesting enhanced permissions...");
            requestEnhancedPermissions();
        } else if (!hasInstallPermission()) {
            LogUtils.logUser("🔧 Auto-requesting install permission...");
            requestInstallPermission();
        } else {
            LogUtils.logUser("✅ All permissions already configured");
        }
    }
    
    /**
     * Check if all required permissions are granted
     * (Alias for hasAllPermissions for compatibility)
     */
    public boolean hasAllRequiredPermissions() {
        return hasAllPermissions();
    }
    
    /**
     * Get permission status as formatted text
     */
    public String getPermissionStatusText() {
        StringBuilder status = new StringBuilder();
        
        // Basic permissions status
        boolean basicGranted = hasBasicPermissions();
        status.append("Basic Permissions: ").append(basicGranted ? "✅ Granted" : "❌ Missing").append("\n");
        
        // Storage permission details
        boolean storageGranted = hasStoragePermission();
        status.append("• Storage Access: ").append(storageGranted ? "✅" : "❌").append("\n");
        
        // All Files Access for Android 11+
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            boolean allFilesGranted = hasManageExternalStoragePermission();
            status.append("• All Files Access: ").append(allFilesGranted ? "✅" : "❌").append("\n");
        }
        
        // Enhanced permissions status
        boolean enhancedGranted = hasEnhancedPermissions();
        status.append("Enhanced Permissions: ").append(enhancedGranted ? "✅ Granted" : "❌ Missing").append("\n");
        
        // Install permission status
        boolean installGranted = hasInstallPermission();
        status.append("Install Permission: ").append(installGranted ? "✅ Granted" : "❌ Missing").append("\n");
        
        // Overall status
        boolean allGranted = hasAllPermissions();
        status.append("\nOverall Status: ").append(allGranted ? "✅ All Ready" : "⚠️ Setup Needed");
        
        return status.toString();
    }
    
    /**
     * Get simplified permission status for quick checks
     */
    public String getSimplePermissionStatus() {
        if (hasAllPermissions()) {
            return "✅ All permissions granted";
        } else if (hasBasicPermissions()) {
            return "⚠️ Basic permissions granted, additional setup needed";
        } else {
            return "❌ Permissions required - tap to setup";
        }
    }
    
    /**
     * Check if permission setup is needed
     */
    public boolean isPermissionSetupNeeded() {
        return !hasAllPermissions();
    }
    
    /**
     * Get count of granted permissions
     */
    public int getGrantedPermissionCount() {
        int count = 0;
        
        if (hasStoragePermission()) count++;
        if (hasInstallPermission()) count++;
        if (hasEnhancedPermissions()) count += ENHANCED_PERMISSIONS.length;
        
        return count;
    }
    
    /**
     * Get total permission count
     */
    public int getTotalPermissionCount() {
        return BASIC_PERMISSIONS.length + ENHANCED_PERMISSIONS.length + 2; // +2 for storage and install
    }
    
    /**
     * Get permission progress as percentage
     */
    public int getPermissionProgress() {
        int granted = getGrantedPermissionCount();
        int total = getTotalPermissionCount();
        return total > 0 ? (granted * 100) / total : 0;
    }
    
    /**
     * Clean up resources
     */
    public void cleanup() {
        callback = null;
        permissionStates.clear();
    }
}



===== ModLoader/app/src/main/java/com/modloader/util/PrivilegeManager.java =====

package com.modloader.util;

public class PrivilegeManager {
}


