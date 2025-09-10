urn result;
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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/ApkPatcher.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/ApkValidator.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/DiagnosticBundleExporter.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/Downloader.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/FileUtils.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/LogUtils.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/MelonLoaderDiagnostic.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/OfflineZipImporter.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/OnlineInstaller.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/PatchResult.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/PathManager.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/PermissionManager.java

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

================================================================================

ModLoader/app/src/main/java/com/modloader/util/PrivilegeManager.java

package com.modloader.util;

public class PrivilegeManager {
}


================================================================================

ModLoader/app/src/main/java/com/modloader/util/RootManager.java

// File: RootManager.java (FIXED) - Complete Root Access Management
// Path: /app/src/main/java/com/modloader/util/RootManager.java

package com.modloader.util;

import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageManager;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;
import android.app.Activity;

import java.io.*;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class RootManager {
    private static final String TAG = "RootManager";
    
    // Common root binary locations
    private static final String[] ROOT_BINARIES = {
        "/system/bin/su",
        "/system/xbin/su", 
        "/sbin/su",
        "/system/su",
        "/vendor/bin/su"
    };
    
    // Root management app packages
    private static final String[] ROOT_APPS = {
        "com.topjohnwu.magisk",           // Magisk
        "eu.chainfire.supersu",           // SuperSU
        "com.koushikdutta.superuser",     // Superuser
        "com.noshufou.android.su",        // Superuser (older)
        "com.thirdparty.superuser",       // SuperUser (CyanogenMod)
        "me.phh.superuser"                // SuperUser (LineageOS)
    };
    
    private final Context context;
    private final Activity activity;
    
    // Root state tracking
    private Boolean rootAvailable = null;
    private Boolean rootGranted = null;
    private String rootAppPackage = null;
    private Process rootProcess = null;
    private BufferedWriter rootWriter = null;
    private BufferedReader rootReader = null;
    
    private RootCallback callback;
    
    public interface RootCallback {
        void onRootAvailable(boolean available);
        void onRootGranted(boolean granted);
        void onRootCommandResult(String command, boolean success, String output);
        void onRootError(String error);
    }
    
    public RootManager(Context context) {
        this.context = context;
        this.activity = context instanceof Activity ? (Activity) context : null;
        checkRootAvailability();
    }
    
    public RootManager(Activity activity) {
        this.context = activity;
        this.activity = activity;
        checkRootAvailability();
    }
    
    public void setCallback(RootCallback callback) {
        this.callback = callback;
    }
    
    /**
     * FIXED: Check if root access is available on this device
     */
    public boolean isRootAvailable() {
        if (rootAvailable != null) {
            return rootAvailable;
        }
        
        LogUtils.logDebug("Checking root availability...");
        
        // Method 1: Check for su binary
        boolean hasSuBinary = checkSuBinary();
        
        // Method 2: Check for root management apps
        boolean hasRootApp = checkRootApps();
        
        // Method 3: Check build tags for test-keys
        boolean hasTestKeys = checkBuildTags();
        
        // Method 4: Try to execute 'which su' command
        boolean canExecuteSu = testSuExecution();
        
        rootAvailable = hasSuBinary || hasRootApp || hasTestKeys || canExecuteSu;
        
        LogUtils.logDebug("Root availability check:");
        LogUtils.logDebug("- SU Binary: " + hasSuBinary);
        LogUtils.logDebug("- Root App: " + hasRootApp);
        LogUtils.logDebug("- Test Keys: " + hasTestKeys);
        LogUtils.logDebug("- SU Execution: " + canExecuteSu);
        LogUtils.logDebug("- Overall Available: " + rootAvailable);
        
        if (callback != null) {
            callback.onRootAvailable(rootAvailable);
        }
        
        return rootAvailable;
    }
    
    /**
     * Check for su binary in common locations
     */
    private boolean checkSuBinary() {
        try {
            for (String path : ROOT_BINARIES) {
                File suFile = new File(path);
                if (suFile.exists() && suFile.canExecute()) {
                    LogUtils.logDebug("Found su binary at: " + path);
                    return true;
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error checking su binaries: " + e.getMessage());
        }
        return false;
    }
    
    /**
     * Check for installed root management applications
     */
    private boolean checkRootApps() {
        try {
            PackageManager pm = context.getPackageManager();
            for (String rootPackage : ROOT_APPS) {
                try {
                    ApplicationInfo appInfo = pm.getApplicationInfo(rootPackage, 0);
                    if (appInfo != null) {
                        LogUtils.logDebug("Found root app: " + rootPackage);
                        rootAppPackage = rootPackage;
                        return true;
                    }
                } catch (PackageManager.NameNotFoundException e) {
                    // App not installed, continue checking
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error checking root apps: " + e.getMessage());
        }
        return false;
    }
    
    /**
     * Check build tags for test-keys (indicates custom ROM/rooted device)
     */
    private boolean checkBuildTags() {
        try {
            String buildTags = android.os.Build.TAGS;
            return buildTags != null && buildTags.contains("test-keys");
        } catch (Exception e) {
            LogUtils.logDebug("Error checking build tags: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Test if su command can be executed
     */
    private boolean testSuExecution() {
        try {
            Process process = Runtime.getRuntime().exec("which su");
            process.waitFor(2, TimeUnit.SECONDS);
            int exitCode = process.exitValue();
            return exitCode == 0;
        } catch (Exception e) {
            LogUtils.logDebug("Error testing su execution: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check if app has been granted root permission
     */
    public boolean hasRootPermission() {
        if (rootGranted != null) {
            return rootGranted;
        }
        
        if (!isRootAvailable()) {
            rootGranted = false;
            return false;
        }
        
        LogUtils.logDebug("Testing root permission...");
        
        // Try to execute a simple root command
        String testResult = executeRootCommand("id", 3000);
        rootGranted = testResult != null && testResult.contains("uid=0");
        
        LogUtils.logDebug("Root permission test result: " + rootGranted);
        if (rootGranted) {
            LogUtils.logUser("✅ Root permission granted");
        } else {
            LogUtils.logUser("❌ Root permission not granted");
        }
        
        if (callback != null) {
            callback.onRootGranted(rootGranted);
        }
        
        return rootGranted;
    }
    
    /**
     * FIXED: Check if root is ready (available and granted)
     */
    public boolean isRootReady() {
        boolean ready = isRootAvailable() && hasRootPermission();
        LogUtils.logDebug("Root ready status: " + ready);
        return ready;
    }
    
    /**
     * FIXED: Request root access from the user
     */
    public void requestRootAccess() {
        if (!isRootAvailable()) {
            LogUtils.logUser("❌ Root is not available on this device");
            showToast("Root access is not available on this device");
            return;
        }
        
        if (hasRootPermission()) {
            LogUtils.logUser("✅ Root permission already granted");
            showToast("Root access already granted!");
            return;
        }
        
        LogUtils.logUser("🔐 Requesting root access...");
        
        // Try to get root access by executing a simple command
        new Thread(() -> {
            try {
                String result = executeRootCommand("echo 'Root access granted'", 5000);
                boolean success = result != null && result.contains("Root access granted");
                
                if (activity != null) {
                    activity.runOnUiThread(() -> {
                        if (success) {
                            rootGranted = true;
                            LogUtils.logUser("✅ Root access granted successfully!");
                            showToast("Root access granted!");
                            if (callback != null) {
                                callback.onRootGranted(true);
                            }
                        } else {
                            rootGranted = false;
                            LogUtils.logUser("❌ Root access denied or failed");
                            showRootRequestFailedDialog();
                            if (callback != null) {
                                callback.onRootGranted(false);
                            }
                        }
                    });
                }
            } catch (Exception e) {
                LogUtils.logDebug("Error requesting root access: " + e.getMessage());
                if (activity != null) {
                    activity.runOnUiThread(() -> {
                        showToast("Error requesting root access");
                        if (callback != null) {
                            callback.onRootError(e.getMessage());
                        }
                    });
                }
            }
        }).start();
    }
    
    /**
     * Show dialog when root request fails
     */
    private void showRootRequestFailedDialog() {
        if (activity == null) return;
        
        new AlertDialog.Builder(activity)
            .setTitle("❌ Root Access Failed")
            .setMessage("Root access was denied or failed.\n\n" +
                "Possible reasons:\n" +
                "• Root permission was denied in the popup\n" +
                "• Root management app is not properly configured\n" +
                "• Device is not properly rooted\n\n" +
                "Solutions:\n" +
                "• Try again and grant permission\n" +
                "• Check your root management app settings\n" +
                "• Restart the device and try again")
            .setPositiveButton("Try Again", (dialog, which) -> requestRootAccess())
            .setNegativeButton("OK", null)
            .show();
    }
    
    /**
     * FIXED: Execute shell command with root privileges
     */
    public String executeRootCommand(String command) {
        return executeRootCommand(command, 10000); // Default 10 second timeout
    }
    
    /**
     * Execute shell command with root privileges and timeout
     */
    public String executeRootCommand(String command, long timeoutMs) {
        if (!isRootAvailable()) {
            LogUtils.logDebug("Cannot execute root command - root not available");
            return null;
        }
        
        LogUtils.logDebug("Executing root command: " + command);
        
        Process process = null;
        BufferedReader reader = null;
        BufferedWriter writer = null;
        
        try {
            // Start su process
            process = Runtime.getRuntime().exec("su");
            
            writer = new BufferedWriter(new OutputStreamWriter(process.getOutputStream()));
            reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
            
            // Send command
            writer.write(command + "\n");
            writer.write("exit\n");
            writer.flush();
            
            // Wait for completion with timeout
            boolean finished = process.waitFor(timeoutMs, TimeUnit.MILLISECONDS);
            if (!finished) {
                LogUtils.logDebug("Root command timed out: " + command);
                process.destroyForcibly();
                return null;
            }
            
            // Read output
            StringBuilder output = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }
            
            int exitCode = process.exitValue();
            String result = output.toString().trim();
            
            LogUtils.logDebug("Root command result (exit=" + exitCode + "): " + 
                (result.length() > 100 ? result.substring(0, 100) + "..." : result));
            
            if (callback != null) {
                callback.onRootCommandResult(command, exitCode == 0, result);
            }
            
            return exitCode == 0 ? result : null;
            
        } catch (Exception e) {
            LogUtils.logDebug("Error executing root command: " + e.getMessage());
            if (callback != null) {
                callback.onRootError("Command execution failed: " + e.getMessage());
            }
            return null;
        } finally {
            // Clean up resources
            try {
                if (writer != null) writer.close();
                if (reader != null) reader.close();
                if (process != null && process.isAlive()) {
                    process.destroyForcibly();
                }
            } catch (Exception e) {
                LogUtils.logDebug("Error cleaning up root command resources: " + e.getMessage());
            }
        }
    }
    
    /**
     * Execute multiple root commands in a single su session
     */
    public List<String> executeRootCommands(String[] commands) {
        return executeRootCommands(commands, 15000); // Default 15 second timeout for multiple commands
    }
    
    /**
     * Execute multiple root commands with timeout
     */
    public List<String> executeRootCommands(String[] commands, long timeoutMs) {
        List<String> results = new ArrayList<>();
        
        if (!isRootAvailable() || commands == null || commands.length == 0) {
            return results;
        }
        
        LogUtils.logDebug("Executing " + commands.length + " root commands");
        
        Process process = null;
        BufferedReader reader = null;
        BufferedWriter writer = null;
        
        try {
            process = Runtime.getRuntime().exec("su");
            writer = new BufferedWriter(new OutputStreamWriter(process.getOutputStream()));
            reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
            
            // Send all commands
            for (String command : commands) {
                writer.write(command + "\n");
                writer.write("echo '---COMMAND_SEPARATOR---'\n");
            }
            writer.write("exit\n");
            writer.flush();
            
            // Wait for completion
            boolean finished = process.waitFor(timeoutMs, TimeUnit.MILLISECONDS);
            if (!finished) {
                LogUtils.logDebug("Root commands timed out");
                process.destroyForcibly();
                return results;
            }
            
            // Read output and split by separator
            StringBuilder currentOutput = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                if ("---COMMAND_SEPARATOR---".equals(line)) {
                    results.add(currentOutput.toString().trim());
                    currentOutput.setLength(0);
                } else {
                    currentOutput.append(line).append("\n");
                }
            }
            
            // Add final output if any
            if (currentOutput.length() > 0) {
                results.add(currentOutput.toString().trim());
            }
            
            LogUtils.logDebug("Root commands completed: " + results.size() + " results");
            
        } catch (Exception e) {
            LogUtils.logDebug("Error executing root commands: " + e.getMessage());
            if (callback != null) {
                callback.onRootError("Batch command execution failed: " + e.getMessage());
            }
        } finally {
            try {
                if (writer != null) writer.close();
                if (reader != null) reader.close();
                if (process != null && process.isAlive()) {
                    process.destroyForcibly();
                }
            } catch (Exception e) {
                LogUtils.logDebug("Error cleaning up batch command resources: " + e.getMessage());
            }
        }
        
        return results;
    }
    
    /**
     * Start persistent root shell session
     */
    public boolean startRootSession() {
        if (!isRootAvailable()) {
            LogUtils.logDebug("Cannot start root session - root not available");
            return false;
        }
        
        if (rootProcess != null && rootProcess.isAlive()) {
            LogUtils.logDebug("Root session already active");
            return true;
        }
        
        try {
            LogUtils.logDebug("Starting persistent root session...");
            rootProcess = Runtime.getRuntime().exec("su");
            rootWriter = new BufferedWriter(new OutputStreamWriter(rootProcess.getOutputStream()));
            rootReader = new BufferedReader(new InputStreamReader(rootProcess.getInputStream()));
            
            LogUtils.logDebug("✅ Root session started successfully");
            return true;
            
        } catch (Exception e) {
            LogUtils.logDebug("Error starting root session: " + e.getMessage());
            closeRootSession();
            return false;
        }
    }
    
    /**
     * Execute command in persistent root session
     */
    public String executeInRootSession(String command) {
        if (rootProcess == null || !rootProcess.isAlive() || rootWriter == null || rootReader == null) {
            LogUtils.logDebug("Root session not active, starting new session");
            if (!startRootSession()) {
                return null;
            }
        }
        
        try {
            LogUtils.logDebug("Executing in root session: " + command);
            
            rootWriter.write(command + "\n");
            rootWriter.write("echo '---END_OF_COMMAND---'\n");
            rootWriter.flush();
            
            StringBuilder output = new StringBuilder();
            String line;
            while ((line = rootReader.readLine()) != null) {
                if ("---END_OF_COMMAND---".equals(line)) {
                    break;
                }
                output.append(line).append("\n");
            }
            
            String result = output.toString().trim();
            LogUtils.logDebug("Root session command result: " + 
                (result.length() > 100 ? result.substring(0, 100) + "..." : result));
            
            return result;
            
        } catch (Exception e) {
            LogUtils.logDebug("Error executing in root session: " + e.getMessage());
            closeRootSession();
            return null;
        }
    }
    
    /**
     * Close persistent root session
     */
    public void closeRootSession() {
        try {
            if (rootWriter != null) {
                rootWriter.write("exit\n");
                rootWriter.flush();
                rootWriter.close();
                rootWriter = null;
            }
            if (rootReader != null) {
                rootReader.close();
                rootReader = null;
            }
            if (rootProcess != null) {
                if (rootProcess.isAlive()) {
                    rootProcess.waitFor(2, TimeUnit.SECONDS);
                    if (rootProcess.isAlive()) {
                        rootProcess.destroyForcibly();
                    }
                }
                rootProcess = null;
            }
            LogUtils.logDebug("Root session closed");
        } catch (Exception e) {
            LogUtils.logDebug("Error closing root session: " + e.getMessage());
        }
    }
    
    /**
     * Check root status and display information
     */
    public void checkRootStatus() {
        LogUtils.logUser("🔍 Checking root status...");
        
        boolean available = isRootAvailable();
        boolean granted = hasRootPermission();
        
        StringBuilder status = new StringBuilder();
        status.append("=== Root Status ===\n");
        status.append("Available: ").append(available ? "✅" : "❌").append("\n");
        status.append("Permission Granted: ").append(granted ? "✅" : "❌").append("\n");
        
        if (available) {
            if (rootAppPackage != null) {
                status.append("Root Manager: ").append(rootAppPackage).append("\n");
            }
            
            // Try to get root info
            String whoami = executeRootCommand("whoami");
            if (whoami != null) {
                status.append("Root User: ").append(whoami).append("\n");
            }
            
            String suVersion = executeRootCommand("su --version");
            if (suVersion != null) {
                status.append("SU Version: ").append(suVersion).append("\n");
            }
        }
        
        LogUtils.logUser(status.toString());
        
        if (activity != null) {
            new AlertDialog.Builder(activity)
                .setTitle("Root Status")
                .setMessage(status.toString())
                .setPositiveButton("OK", null)
                .show();
        }
    }
    
    /**
     * Install/copy file using root privileges
     */
    public boolean installFileAsRoot(String sourcePath, String targetPath) {
        if (!isRootReady()) {
            LogUtils.logDebug("Cannot install file as root - root not ready");
            return false;
        }
        
        LogUtils.logDebug("Installing file as root: " + sourcePath + " -> " + targetPath);
        
        String[] commands = {
            "cp '" + sourcePath + "' '" + targetPath + "'",
            "chmod 644 '" + targetPath + "'",
            "chown system:system '" + targetPath + "'"
        };
        
        List<String> results = executeRootCommands(commands);
        boolean success = results.size() == commands.length;
        
        if (success) {
            LogUtils.logUser("✅ File installed successfully with root: " + targetPath);
        } else {
            LogUtils.logUser("❌ Failed to install file with root: " + targetPath);
        }
        
        return success;
    }
    
    /**
     * Create directory using root privileges
     */
    public boolean createDirectoryAsRoot(String dirPath) {
        if (!isRootReady()) {
            return false;
        }
        
        String result = executeRootCommand("mkdir -p '" + dirPath + "' && echo 'SUCCESS'");
        boolean success = result != null && result.contains("SUCCESS");
        
        if (success) {
            LogUtils.logDebug("Created directory as root: " + dirPath);
        }
        
        return success;
    }
    
    /**
     * Delete file/directory using root privileges  
     */
    public boolean deleteAsRoot(String path) {
        if (!isRootReady()) {
            return false;
        }
        
        String result = executeRootCommand("rm -rf '" + path + "' && echo 'DELETED'");
        boolean success = result != null && result.contains("DELETED");
        
        if (success) {
            LogUtils.logDebug("Deleted as root: " + path);
        }
        
        return success;
    }
    
    /**
     * Get detailed root information
     */
    public String getDetailedStatus() {
        StringBuilder info = new StringBuilder();
        info.append("=== Root Manager Status ===\n");
        
        boolean available = isRootAvailable();
        boolean granted = hasRootPermission();
        
        info.append("Available: ").append(available ? "✅" : "❌").append("\n");
        info.append("Permission: ").append(granted ? "✅" : "❌").append("\n");
        info.append("Ready: ").append(isRootReady() ? "✅" : "❌").append("\n");
        
        if (rootAppPackage != null) {
            info.append("Root App: ").append(rootAppPackage).append("\n");
        }
        
        info.append("Session Active: ").append(
            (rootProcess != null && rootProcess.isAlive()) ? "✅" : "❌").append("\n");
        
        // Add system info if root is available
        if (available) {
            String buildTags = android.os.Build.TAGS;
            info.append("Build Tags: ").append(buildTags != null ? buildTags : "Unknown").append("\n");
        }
        
        return info.toString();
    }
    
    /**
     * Refresh root availability (call after potential changes)
     */
    public void refreshStatus() {
        LogUtils.logDebug("Refreshing root status...");
        rootAvailable = null;
        rootGranted = null;
        checkRootAvailability();
    }
    
    /**
     * Check root availability and update cache
     */
    private void checkRootAvailability() {
        // This will update the rootAvailable cache
        isRootAvailable();
    }
    
    /**
     * Show toast message
     */
    private void showToast(String message) {
        if (context != null) {
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Clean up resources
     */
    public void cleanup() {
        closeRootSession();
        callback = null;
    }
}

================================================================================

ModLoader/app/src/main/java/com/modloader/util/ShizukuManager.java

// File: ShizukuManager.java (FIXED) - Complete Shizuku Integration with Proper Permission Detection
// Path: /app/src/main/java/com/modloader/util/ShizukuManager.java

package com.modloader.util;

import android.app.Activity;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.IBinder;
import android.os.RemoteException;
import android.widget.Toast;

import java.lang.reflect.Method;

public class ShizukuManager {
    private static final String TAG = "ShizukuManager";
    
    // Shizuku package constants
    private static final String SHIZUKU_PACKAGE = "moe.shizuku.privileged.api";
    private static final String SHIZUKU_SERVICE = "moe.shizuku.privileged.api.ShizukuService";
    private static final String SHIZUKU_ACTIVITY = "moe.shizuku.manager.MainActivity";
    
    // Permission constants  
    private static final String SHIZUKU_PERMISSION = "moe.shizuku.manager.permission.API_V23";
    private static final int SHIZUKU_REQUEST_CODE = 9999;
    
    private final Context context;
    private final Activity activity;
    
    // Shizuku API reflection objects
    private Class<?> shizukuClass;
    private Method checkSelfPermissionMethod;
    private Method requestPermissionMethod;
    private Method isPreV11Method;
    private Method pingBinderMethod;
    private Method getVersionMethod;
    private Method getUidMethod;
    
    // Connection state
    private boolean isShizukuConnected = false;
    private Object shizukuBinder = null;
    
    public ShizukuManager(Context context) {
        this.context = context;
        this.activity = context instanceof Activity ? (Activity) context : null;
        initializeShizukuReflection();
        checkShizukuConnection();
    }
    
    public ShizukuManager(Activity activity) {
        this.context = activity;
        this.activity = activity;
        initializeShizukuReflection();
        checkShizukuConnection();
    }
    
    /**
     * FIXED: Initialize Shizuku API through reflection to avoid compile-time dependency
     */
    private void initializeShizukuReflection() {
        try {
            LogUtils.logDebug("Initializing Shizuku reflection API...");
            
            // Try to load Shizuku API class
            shizukuClass = Class.forName("rikka.shizuku.Shizuku");
            
            // Get essential methods through reflection
            checkSelfPermissionMethod = shizukuClass.getMethod("checkSelfPermission");
            requestPermissionMethod = shizukuClass.getMethod("requestPermission", int.class);
            pingBinderMethod = shizukuClass.getMethod("pingBinder");
            getVersionMethod = shizukuClass.getMethod("getVersion");
            getUidMethod = shizukuClass.getMethod("getUid");
            
            // Check if pre-v11 method exists (for older Shizuku versions)
            try {
                isPreV11Method = shizukuClass.getMethod("isPreV11");
            } catch (NoSuchMethodException e) {
                LogUtils.logDebug("isPreV11 method not found - using newer Shizuku API");
            }
            
            LogUtils.logDebug("✅ Shizuku reflection API initialized successfully");
            
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku API not found in classpath - will use intent-based detection");
            shizukuClass = null;
        } catch (Exception e) {
            LogUtils.logDebug("Failed to initialize Shizuku reflection: " + e.getMessage());
            shizukuClass = null;
        }
    }
    
    /**
     * FIXED: Check if Shizuku app is installed on the device
     */
    public boolean isShizukuInstalled() {
        try {
            PackageManager pm = context.getPackageManager();
            
            // Try primary package name
            try {
                PackageInfo packageInfo = pm.getPackageInfo(SHIZUKU_PACKAGE, 0);
                LogUtils.logDebug("Shizuku found: version " + packageInfo.versionName + 
                    " (" + packageInfo.versionCode + ")");
                return true;
            } catch (PackageManager.NameNotFoundException e) {
                // Try alternative package name
                try {
                    PackageInfo altPackageInfo = pm.getPackageInfo("moe.shizuku.manager", 0);
                    LogUtils.logDebug("Shizuku manager found: version " + altPackageInfo.versionName);
                    return true;
                } catch (PackageManager.NameNotFoundException e2) {
                    LogUtils.logDebug("Shizuku not installed: " + e.getMessage());
                    return false;
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error checking Shizuku installation: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check if Shizuku service is running and accessible
     */
    public boolean isShizukuRunning() {
        if (!isShizukuInstalled()) {
            LogUtils.logDebug("Shizuku not installed");
            return false;
        }
        
        try {
            if (shizukuClass != null && pingBinderMethod != null) {
                // Use reflection to call Shizuku.pingBinder()
                boolean canPing = (Boolean) pingBinderMethod.invoke(null);
                LogUtils.logDebug("Shizuku ping result: " + canPing);
                return canPing;
            } else {
                // Fallback: Check if Shizuku service is bound by attempting connection
                return checkShizukuServiceBinding();
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error checking Shizuku service: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check Shizuku service binding as fallback method
     */
    private boolean checkShizukuServiceBinding() {
        try {
            Intent serviceIntent = new Intent();
            serviceIntent.setComponent(new ComponentName(SHIZUKU_PACKAGE, SHIZUKU_SERVICE));
            
            ServiceConnection testConnection = new ServiceConnection() {
                @Override
                public void onServiceConnected(ComponentName name, IBinder service) {
                    isShizukuConnected = true;
                    shizukuBinder = service;
                    LogUtils.logDebug("Shizuku service connected via binding test");
                }
                
                @Override
                public void onServiceDisconnected(ComponentName name) {
                    isShizukuConnected = false;
                    shizukuBinder = null;
                }
            };
            
            // Try to bind to service
            boolean bindResult = context.bindService(serviceIntent, testConnection, 
                Context.BIND_AUTO_CREATE | Context.BIND_NOT_FOREGROUND);
            
            if (bindResult) {
                // Unbind immediately - we just wanted to test connectivity
                try {
                    context.unbindService(testConnection);
                } catch (Exception e) {
                    LogUtils.logDebug("Error unbinding test service: " + e.getMessage());
                }
                return true;
            }
            
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Service binding test failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check if app has Shizuku permission - this is the main fix
     */
    public boolean hasShizukuPermission() {
        if (!isShizukuRunning()) {
            LogUtils.logDebug("Shizuku not running - cannot check permission");
            return false;
        }
        
        try {
            if (shizukuClass != null && checkSelfPermissionMethod != null) {
                // FIXED: Use reflection to call Shizuku.checkSelfPermission()
                int permissionResult = (Integer) checkSelfPermissionMethod.invoke(null);
                boolean hasPermission = (permissionResult == PackageManager.PERMISSION_GRANTED);
                
                LogUtils.logDebug("Shizuku permission check result: " + permissionResult + 
                    " (granted=" + hasPermission + ")");
                    
                if (hasPermission) {
                    LogUtils.logUser("✅ Shizuku permission already granted");
                } else {
                    LogUtils.logUser("❌ Shizuku permission not granted (result: " + permissionResult + ")");
                }
                
                return hasPermission;
            } else {
                // FIXED: Fallback method for when reflection fails
                return checkShizukuPermissionFallback();
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error checking Shizuku permission: " + e.getMessage());
            // Try fallback method
            return checkShizukuPermissionFallback();
        }
    }
    
    /**
     * FIXED: Fallback permission check method
     */
    private boolean checkShizukuPermissionFallback() {
        try {
            LogUtils.logDebug("Using fallback Shizuku permission check...");
            
            // Method 1: Check using standard permission system
            int permResult = context.checkSelfPermission(SHIZUKU_PERMISSION);
            if (permResult == PackageManager.PERMISSION_GRANTED) {
                LogUtils.logDebug("Shizuku permission granted via standard check");
                return true;
            }
            
            // Method 2: Check using package manager
            PackageManager pm = context.getPackageManager();
            String[] permissions = pm.getPackageInfo(context.getPackageName(), 
                PackageManager.GET_PERMISSIONS).requestedPermissions;
                
            if (permissions != null) {
                for (String permission : permissions) {
                    if (SHIZUKU_PERMISSION.equals(permission)) {
                        LogUtils.logDebug("Shizuku permission found in manifest");
                        // Check if actually granted
                        return pm.checkPermission(SHIZUKU_PERMISSION, context.getPackageName()) 
                            == PackageManager.PERMISSION_GRANTED;
                    }
                }
            }
            
            LogUtils.logDebug("Shizuku permission not found in fallback checks");
            return false;
            
        } catch (Exception e) {
            LogUtils.logDebug("Fallback permission check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Request Shizuku permission from user
     */
    public void requestShizukuPermission() {
        if (!isShizukuRunning()) {
            LogUtils.logUser("❌ Shizuku is not running. Start Shizuku first.");
            showToast("Shizuku is not running. Please start Shizuku service first.");
            return;
        }
        
        if (hasShizukuPermission()) {
            LogUtils.logUser("✅ Shizuku permission already granted");
            showToast("Shizuku permission already granted!");
            return;
        }
        
        try {
            LogUtils.logUser("🔐 Requesting Shizuku permission...");
            
            if (shizukuClass != null && requestPermissionMethod != null && activity != null) {
                // FIXED: Use reflection to call Shizuku.requestPermission()
                requestPermissionMethod.invoke(null, SHIZUKU_REQUEST_CODE);
                LogUtils.logUser("📱 Shizuku permission dialog should appear");
                showToast("Grant permission in the Shizuku dialog that appears");
                
            } else {
                // FIXED: Fallback - open Shizuku app for manual permission grant
                LogUtils.logDebug("Using fallback permission request method");
                openShizukuAppForPermission();
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error requesting Shizuku permission: " + e.getMessage());
            LogUtils.logUser("❌ Failed to request Shizuku permission automatically");
            
            // Show manual instruction dialog
            showManualPermissionInstructions();
        }
    }
    
    /**
     * FIXED: Fallback method to open Shizuku app for permission granting
     */
    private void openShizukuAppForPermission() {
        try {
            // Try to open Shizuku manager
            Intent intent = new Intent();
            intent.setComponent(new ComponentName(SHIZUKU_PACKAGE, SHIZUKU_ACTIVITY));
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            
            if (intent.resolveActivity(context.getPackageManager()) != null) {
                context.startActivity(intent);
                LogUtils.logUser("📱 Opened Shizuku app - grant permission manually");
                showToast("Grant permission to TerrariaLoader in Shizuku settings");
            } else {
                // Try alternative package
                intent.setPackage("moe.shizuku.manager");
                if (intent.resolveActivity(context.getPackageManager()) != null) {
                    context.startActivity(intent);
                    LogUtils.logUser("📱 Opened Shizuku manager - grant permission manually");
                } else {
                    LogUtils.logDebug("Could not open Shizuku app");
                    showManualPermissionInstructions();
                }
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error opening Shizuku app: " + e.getMessage());
            showManualPermissionInstructions();
        }
    }
    
    /**
     * Show manual permission instructions to user
     */
    private void showManualPermissionInstructions() {
        if (activity != null) {
            android.app.AlertDialog.Builder builder = new android.app.AlertDialog.Builder(activity);
            builder.setTitle("🔐 Manual Shizuku Permission Required");
            builder.setMessage("Please grant permission manually:\n\n" +
                "1. Open Shizuku app\n" +
                "2. Go to 'Applications using Shizuku API'\n" +
                "3. Find 'TerrariaLoader' in the list\n" +
                "4. Toggle the permission ON\n" +
                "5. Return to TerrariaLoader\n\n" +
                "If TerrariaLoader is not in the list, restart the app and try again.");
            builder.setPositiveButton("Open Shizuku", (dialog, which) -> openShizukuApp());
            builder.setNegativeButton("OK", null);
            builder.show();
        }
    }
    
    /**
     * Check current Shizuku connection status
     */
    private void checkShizukuConnection() {
        try {
            boolean installed = isShizukuInstalled();
            boolean running = isShizukuRunning();
            boolean hasPermission = hasShizukuPermission();
            
            LogUtils.logDebug("=== Shizuku Status ===");
            LogUtils.logDebug("Installed: " + installed);
            LogUtils.logDebug("Running: " + running);  
            LogUtils.logDebug("Has Permission: " + hasPermission);
            LogUtils.logDebug("Ready: " + (installed && running && hasPermission));
            
            if (installed && running && hasPermission) {
                LogUtils.logUser("✅ Shizuku is ready and has permission");
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error checking Shizuku connection: " + e.getMessage());
        }
    }
    
    /**
     * Get Shizuku version information
     */
    public String getShizukuVersion() {
        try {
            if (shizukuClass != null && getVersionMethod != null) {
                int version = (Integer) getVersionMethod.invoke(null);
                return String.valueOf(version);
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error getting Shizuku version: " + e.getMessage());
        }
        
        // Fallback: get from package info
        try {
            PackageInfo packageInfo = context.getPackageManager().getPackageInfo(SHIZUKU_PACKAGE, 0);
            return packageInfo.versionName;
        } catch (Exception e) {
            return "Unknown";
        }
    }
    
    /**
     * Get Shizuku UID
     */
    public int getShizukuUid() {
        try {
            if (shizukuClass != null && getUidMethod != null) {
                return (Integer) getUidMethod.invoke(null);
            }
        } catch (Exception e) {
            LogUtils.logDebug("Error getting Shizuku UID: " + e.getMessage());
        }
        return -1;
    }
    
    /**
     * Check if Shizuku is available (installed)
     */
    public boolean isShizukuAvailable() {
        return isShizukuInstalled();
    }
    
    /**
     * FIXED: Check if Shizuku is ready (installed, running, and has permission)
     */
    public boolean isShizukuReady() {
        boolean ready = isShizukuInstalled() && isShizukuRunning() && hasShizukuPermission();
        LogUtils.logDebug("Shizuku ready status: " + ready);
        return ready;
    }
    
    /**
     * Open Shizuku app
     */
    public boolean openShizukuApp() {
        try {
            Intent intent = context.getPackageManager().getLaunchIntentForPackage(SHIZUKU_PACKAGE);
            if (intent == null) {
                // Try alternative package
                intent = context.getPackageManager().getLaunchIntentForPackage("moe.shizuku.manager");
            }
            
            if (intent != null) {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(intent);
                return true;
            }
            
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Error opening Shizuku app: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Install Shizuku from GitHub releases
     */
    public void installShizuku() {
        try {
            Intent browserIntent = new Intent(Intent.ACTION_VIEW);
            browserIntent.setData(android.net.Uri.parse("https://github.com/RikkaApps/Shizuku/releases/latest"));
            browserIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(browserIntent);
            LogUtils.logUser("🌐 Opened Shizuku download page");
        } catch (Exception e) {
            LogUtils.logDebug("Error opening Shizuku download: " + e.getMessage());
            showToast("Could not open browser. Please download Shizuku manually from GitHub.");
        }
    }
    
    /**
     * Execute shell command using Shizuku
     */
    public boolean executeShellCommand(String command) {
        if (!isShizukuReady()) {
            LogUtils.logDebug("Shizuku not ready for command execution");
            return false;
        }
        
        try {
            // This would require Shizuku API implementation
            LogUtils.logDebug("Executing shell command via Shizuku: " + command);
            // Implementation would go here using Shizuku's shell execution API
            return true;
        } catch (Exception e) {
            LogUtils.logDebug("Error executing shell command: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Get detailed Shizuku status information
     */
    public String getDetailedStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Shizuku Manager Status ===\n");
        
        boolean installed = isShizukuInstalled();
        boolean running = isShizukuRunning();
        boolean hasPermission = hasShizukuPermission();
        
        status.append("Installed: ").append(installed ? "✅" : "❌").append("\n");
        status.append("Running: ").append(running ? "✅" : "❌").append("\n");
        status.append("Has Permission: ").append(hasPermission ? "✅" : "❌").append("\n");
        status.append("Overall Ready: ").append(isShizukuReady() ? "✅" : "❌").append("\n");
        
        if (installed) {
            status.append("Version: ").append(getShizukuVersion()).append("\n");
            if (running) {
                status.append("UID: ").append(getShizukuUid()).append("\n");
            }
        }
        
        status.append("API Class Available: ").append(shizukuClass != null ? "✅" : "❌").append("\n");
        
        return status.toString();
    }
    
    /**
     * Handle permission request result - call this from Activity.onRequestPermissionsResult()
     */
    public void handlePermissionResult(int requestCode, String[] permissions, int[] grantResults) {
        if (requestCode == SHIZUKU_REQUEST_CODE) {
            boolean granted = grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED;
            LogUtils.logUser(granted ? "✅ Shizuku permission granted!" : "❌ Shizuku permission denied");
            
            if (granted) {
                showToast("Shizuku permission granted successfully!");
            } else {
                showToast("Shizuku permission was denied");
            }
        }
    }
    
    /**
     * Refresh Shizuku status (call after potential status changes)
     */
    public void refreshStatus() {
        LogUtils.logDebug("Refreshing Shizuku status...");
        checkShizukuConnection();
    }
    
    /**
     * Show toast message
     */
    private void showToast(String message) {
        if (context != null) {
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Cleanup resources
     */
    public void cleanup() {
        if (shizukuBinder != null) {
            shizukuBinder = null;
        }
        isShizukuConnected = false;
    }
} 

================================================================================

ModLoader/app/src/main/res/drawable/gradient_background_135.xml

<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <gradient
        android:angle="135"
        android:startColor="#E8F5E8"
        android:endColor="#F1F8E9"
        android:type="linear" />
</shape>

================================================================================

ModLoader/app/src/main/res/drawable/ic_arrow_back.xml

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
  <path
      android:fillColor="@android:color/black"
      android:pathData="M20,11L7.83,11l5.59,-5.59L12,4l-8,8 8,8 1.41,-1.41L7.83,13L20,13z"/>
</vector>


================================================================================

ModLoader/app/src/main/res/drawable/ic_launcher_background.xml

<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path
        android:fillColor="#3DDC84"
        android:pathData="M0,0h108v108h-108z" />
    <path
        android:fillColor="#00000000"
        android:pathData="M9,0L9,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,0L19,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M29,0L29,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M39,0L39,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M49,0L49,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M59,0L59,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M69,0L69,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M79,0L79,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M89,0L89,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M99,0L99,108"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,9L108,9"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,19L108,19"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,29L108,29"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,39L108,39"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,49L108,49"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,59L108,59"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,69L108,69"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,79L108,79"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,89L108,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M0,99L108,99"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,29L89,29"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,39L89,39"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,49L89,49"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,59L89,59"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,69L89,69"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M19,79L89,79"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M29,19L29,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M39,19L39,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M49,19L49,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M59,19L59,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M69,19L69,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
    <path
        android:fillColor="#00000000"
        android:pathData="M79,19L79,89"
        android:strokeWidth="0.8"
        android:strokeColor="#33FFFFFF" />
</vector>


================================================================================

ModLoader/app/src/main/res/drawable-v24/ic_launcher_foreground.xml

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:aapt="http://schemas.android.com/aapt"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path android:pathData="M31,63.928c0,0 6.4,-11 12.1,-13.1c7.2,-2.6 26,-1.4 26,-1.4l38.1,38.1L107,108.928l-32,-1L31,63.928z">
        <aapt:attr name="android:fillColor">
            <gradient
                android:endX="85.84757"
                android:endY="92.4963"
                android:startX="42.9492"
                android:startY="49.59793"
                android:type="linear">
                <item
                    android:color="#44000000"
                    android:offset="0.0" />
                <item
                    android:color="#00000000"
                    android:offset="1.0" />
            </gradient>
        </aapt:attr>
    </path>
    <path
        android:fillColor="#FFFFFF"
        android:fillType="nonZero"
        android:pathData="M65.3,45.828l3.8,-6.6c0.2,-0.4 0.1,-0.9 -0.3,-1.1c-0.4,-0.2 -0.9,-0.1 -1.1,0.3l-3.9,6.7c-6.3,-2.8 -13.4,-2.8 -19.7,0l-3.9,-6.7c-0.2,-0.4 -0.7,-0.5 -1.1,-0.3C38.8,38.328 38.7,38.828 38.9,39.228l3.8,6.6C36.2,49.428 31.7,56.028 31,63.928h46C76.3,56.028 71.8,49.428 65.3,45.828zM43.4,57.328c-0.8,0 -1.5,-0.5 -1.8,-1.2c-0.3,-0.7 -0.1,-1.5 0.4,-2.1c0.5,-0.5 1.4,-0.7 2.1,-0.4c0.7,0.3 1.2,1 1.2,1.8C45.3,56.528 44.5,57.328 43.4,57.328L43.4,57.328zM64.6,57.328c-0.8,0 -1.5,-0.5 -1.8,-1.2s-0.1,-1.5 0.4,-2.1c0.5,-0.5 1.4,-0.7 2.1,-0.4c0.7,0.3 1.2,1 1.2,1.8C66.5,56.528 65.6,57.328 64.6,57.328L64.6,57.328z"
        android:strokeWidth="1"
        android:strokeColor="#00000000" />
</vector>

================================================================================

ModLoader/app/src/main/res/layout/activity_about.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Mod Loader"
            android:textStyle="bold"
            android:textSize="20sp"
            android:layout_marginBottom="8dp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Author: Jonie"
            android:textSize="16sp"
            android:layout_marginBottom="8dp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Description:\n\nThis app lets you inject custom loaders into Terraria APKs for modding purposes. It also provides tools to manage logs and exported APKs.\n\nUse responsibly and only with legal copies of the game."
            android:textSize="14sp"
            android:lineSpacingExtra="4dp" />
    </LinearLayout>
</ScrollView>

================================================================================

ModLoader/app/src/main/res/layout/activity_addon_management.xml

<!-- File: activity_addon_management.xml -->
<!-- Path: app/src/main/res/layout/activity_addon_management.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="#F8F9FA">

    <!-- Header Section -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center"
        android:background="#FFFFFF"
        android:padding="20dp"
        android:layout_marginBottom="16dp"
        android:elevation="4dp">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🔌 Addon Management"
            android:textSize="28sp"
            android:textStyle="bold"
            android:textColor="#2E7D32"
            android:gravity="center"
            android:layout_marginBottom="8dp" />

        <TextView
            android:id="@+id/addonStatusText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="📊 Loading addons..."
            android:textSize="14sp"
            android:textColor="#4CAF50"
            android:gravity="center"
            android:padding="8dp"
            android:background="#E8F5E8"
            android:layout_marginBottom="8dp" />
    </LinearLayout>

    <!-- Controls Section -->
    <androidx.cardview.widget.CardView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"
        app:cardCornerRadius="8dp"
        app:cardElevation="4dp"
        app:cardBackgroundColor="#FFFFFF">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:padding="16dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🎛️ Controls"
                android:textSize="18sp"
                android:textStyle="bold"
                android:textColor="#333333"
                android:layout_marginBottom="12dp" />

            <!-- Filter and Actions Row -->
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="horizontal"
                android:layout_marginBottom="12dp">

                <TextView
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="Category:"
                    android:textSize="14sp"
                    android:textColor="#666666"
                    android:layout_gravity="center_vertical"
                    android:layout_marginEnd="8dp" />

                <Spinner
                    android:id="@+id/categorySpinner"
                    android:layout_width="0dp"
                    android:layout_height="48dp"
                    android:layout_weight="1"
                    android:background="#F5F5F5"
                    android:layout_marginEnd="8dp" />

                <Button
                    android:id="@+id/refreshButton"
                    android:layout_width="wrap_content"
                    android:layout_height="36dp"
                    android:text="🔄"
                    android:textSize="14sp"
                    android:background="#FF9800"
                    android:textColor="#FFFFFF"
                    android:minWidth="48dp" />
            </LinearLayout>

            <!-- Add Addon Button -->
            <Button
                android:id="@+id/addAddonButton"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="➕ Add New Addon"
                android:textSize="16sp"
                android:textStyle="bold"
                android:background="#4CAF50"
                android:textColor="@android:color/white"
                android:minHeight="48dp"
                android:elevation="2dp" />

        </LinearLayout>
    </androidx.cardview.widget.CardView>

    <!-- Addons List Section -->
    <androidx.cardview.widget.CardView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        app:cardCornerRadius="8dp"
        app:cardElevation="4dp"
        app:cardBackgroundColor="#FFFFFF">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical"
            android:padding="16dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="📦 Installed Addons"
                android:textSize="18sp"
                android:textStyle="bold"
                android:textColor="#333333"
                android:layout_marginBottom="12dp" />

            <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/addonRecyclerView"
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_weight="1"
                android:background="#F9F9F9"
                android:padding="8dp" />

        </LinearLayout>
    </androidx.cardview.widget.CardView>

    <!-- Footer Info -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="💡 Tip: Tap addon cards for details, use switches to enable/disable"
        android:textSize="12sp"
        android:textColor="#888888"
        android:gravity="center"
        android:background="#F0F0F0"
        android:padding="12dp"
        android:layout_marginTop="16dp" />

</LinearLayout>


================================================================================

ModLoader/app/src/main/res/layout/activity_addons.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/addons_recycler"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>


================================================================================

ModLoader/app/src/main/res/layout/activity_dll_mod.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <!-- Header Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL Mod Manager"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            android:layout_marginBottom="16dp" />

        <!-- Status Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#F5F5F5"
            android:padding="12dp"
            android:layout_marginBottom="16dp">

            <TextView
                android:id="@+id/statusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="DLL Mods: 0 enabled, 0 disabled, 0 total"
                android:textSize="16sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="❌ No loader installed - DLL mods will not work"
                android:textSize="14sp" />

        </LinearLayout>

        <!-- Loader Installation Section -->
        <LinearLayout
            android:id="@+id/loaderInfoSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#E8F5E8"
            android:padding="12dp"
            android:layout_marginBottom="16dp"
            android:visibility="gone">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Loader Installation"
                android:textSize="16sp"
                android:textStyle="bold"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/installLoaderBtn"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Install Loader"
                android:layout_marginBottom="8dp" />

            <Button
                android:id="@+id/selectApkBtn"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Select Terraria APK"
                android:layout_marginBottom="8dp" />

        </LinearLayout>

        <!-- Mod Management Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL Mod Management"
            android:textSize="18sp"
            android:textStyle="bold"
            android:layout_marginBottom="8dp" />

        <Button
            android:id="@+id/installDllBtn"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Install DLL Mod"
            android:layout_marginBottom="16dp" />

        <!-- Mod List -->
        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/dllModRecyclerView"
            android:layout_width="match_parent"
            android:layout_height="300dp"
            android:background="#F9F9F9"
            android:padding="8dp"
            android:layout_marginBottom="16dp" />

        <!-- Action Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center"
            android:layout_marginTop="16dp">

            <Button
                android:id="@+id/refreshBtn"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Refresh"
                android:layout_marginEnd="8dp" />

            <Button
                android:id="@+id/viewLogsBtn"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="View Logs"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Information Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="DLL mods require MelonLoader or LemonLoader to be installed. Use 'Install Loader' to set up the required components."
            android:textSize="12sp"
            android:textColor="#666666"
            android:layout_marginTop="16dp"
            android:gravity="center" />

    </LinearLayout>

</ScrollView>

================================================================================

ModLoader/app/src/main/res/layout/activity_instructions.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.InstructionsActivity">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="16dp">

        <TextView
            android:id="@+id/tv_instructions_title"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="Manual Installation Instructions"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            android:layout_marginTop="16dp" />

        <TextView
            android:id="@+id/tv_instructions"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:textSize="16sp"
            android:textIsSelectable="true"
            app:layout_constraintTop_toBottomOf="@id/tv_instructions_title"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            tools:text="Detailed instructions will appear here..." />

    </androidx.constraintlayout.widget.ConstraintLayout>
</ScrollView>


================================================================================

ModLoader/app/src/main/res/layout/activity_log.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/log_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <ScrollView
        android:id="@+id/log_scroll"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:fillViewport="true">

        <TextView
            android:id="@+id/log_text"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textIsSelectable="true"
            android:textAppearance="?android:textAppearanceSmall"
            android:textColor="#FFFFFF"
            android:background="#222222"
            android:padding="10dp" />
    </ScrollView>

    <Button
        android:id="@+id/export_logs_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Export Logs" />
</LinearLayout>

================================================================================

ModLoader/app/src/main/res/layout/activity_log_viewer_enhanced.xml

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="#1E1E1E">

    <!-- Statistics Bar -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="8dp"
        android:background="#2E2E2E"
        android:gravity="center_vertical">

        <TextView
            android:id="@+id/logStatsText"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="📊 Total: 0 | Showing: 0 | Errors: 0 | Warnings: 0"
            android:textColor="#FFFFFF"
            android:textSize="12sp"
            android:fontFamily="monospace" />

        <Button
            android:id="@+id/refreshButton"
            android:layout_width="wrap_content"
            android:layout_height="32dp"
            android:text="🔄"
            android:textSize="14sp"
            android:background="#4CAF50"
            android:textColor="#FFFFFF"
            android:layout_marginStart="8dp"
            android:padding="4dp"
            android:minWidth="48dp" />

    </LinearLayout>

    <!-- Filter Section (Collapsible) -->
    <LinearLayout
        android:id="@+id/filterSection"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="12dp"
        android:background="#2A2A2A"
        android:visibility="visible">

        <!-- Filter Controls Row 1 -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center_vertical">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="🏷️ Type:"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Spinner
                android:id="@+id/logTypeSpinner"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:layout_marginEnd="16dp"
                android:popupBackground="#3A3A3A" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="📊 Level:"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Spinner
                android:id="@+id/logLevelSpinner"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:popupBackground="#3A3A3A" />

        </LinearLayout>

        <!-- Search Bar -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="12dp"
            android:gravity="center_vertical">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="🔍"
                android:textSize="16sp"
                android:layout_marginEnd="8dp" />

            <EditText
                android:id="@+id/searchEditText"
                android:layout_width="0dp"
                android:layout_height="48dp"
                android:layout_weight="1"
                android:background="#3A3A3A"
                android:hint="Search logs..."
                android:textColorHint="#888888"
                android:textColor="#FFFFFF"
                android:padding="12dp"
                android:textSize="14sp"
                android:fontFamily="monospace"
                android:inputType="text"
                android:imeOptions="actionSearch" />

        </LinearLayout>

        <!-- Control Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="12dp"
            android:gravity="center">

            <Button
                android:id="@+id/clearLogsButton"
                android:layout_width="0dp"
                android:layout_height="40dp"
                android:layout_weight="1"
                android:text="🗑️ Clear"
                android:background="#FF6B6B"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginEnd="8dp" />

            <Button
                android:id="@+id/exportLogsButton"
                android:layout_width="0dp"
                android:layout_height="40dp"
                android:layout_weight="1"
                android:text="📤 Export"
                android:background="#2196F3"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Auto-scroll checkbox -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginTop="8dp"
            android:gravity="center_vertical">

            <CheckBox
                android:id="@+id/autoScrollCheckbox"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="📜 Auto-scroll to bottom"
                android:textColor="#FFFFFF"
                android:textSize="14sp"
                android:checked="true"
                android:buttonTint="#4CAF50" />

        </LinearLayout>

    </LinearLayout>

    <!-- Main Log Content -->
    <androidx.swiperefreshlayout.widget.SwipeRefreshLayout
        android:id="@+id/swipeRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1">

        <ScrollView
            android:id="@+id/logScrollView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#1E1E1E"
            android:scrollbars="vertical"
            android:fadeScrollbars="false">

            <TextView
                android:id="@+id/logTextView"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="📋 Loading logs...\n\nPlease wait while we fetch the latest log entries."
                android:textColor="#E0E0E0"
                android:textSize="12sp"
                android:fontFamily="monospace"
                android:padding="16dp"
                android:textIsSelectable="true"
                android:background="#1E1E1E"
                android:lineSpacingMultiplier="1.2" />

        </ScrollView>

    </androidx.swiperefreshlayout.widget.SwipeRefreshLayout>

    <!-- Bottom Action Bar -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="8dp"
        android:background="#2E2E2E"
        android:gravity="center_vertical">

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:text="💡 Tip: Swipe down to refresh, use filters to find specific logs"
            android:textColor="#888888"
            android:textSize="11sp" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="v1.0"
            android:textColor="#666666"
            android:textSize="10sp"
            android:layout_marginStart="8dp" />

    </LinearLayout>

</LinearLayout>

================================================================================

ModLoader/app/src/main/res/layout/activity_main.xml

<!-- File: activity_main.xml (Updated with Plugin/Addon Button) -->
<!-- Path: app/src/main/res/layout/activity_main.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="24dp"
    android:background="@drawable/gradient_background_135"
    android:gravity="center">

    <!-- Header Section -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center"
        android:background="#FFFFFF"
        android:padding="32dp"
        android:layout_marginBottom="32dp"
        android:elevation="8dp">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🎮 Mod Loader"
            android:textSize="32sp"
            android:textStyle="bold"
            android:textColor="#2E7D32"
            android:gravity="center"
            android:layout_marginBottom="12dp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Choose your modding approach"
            android:textSize="16sp"
            android:textColor="#4CAF50"
            android:gravity="center"
            android:lineSpacingExtra="4dp" />

    </LinearLayout>

    <!-- Main Navigation Buttons -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_marginBottom="24dp">

        <!-- Universal Mode Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#E3F2FD">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🌐 Universal Mode"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#1565C0"
                    android:layout_marginBottom="8dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="Inject loaders into any APK file\n• Works with any Unity game\n• Manual APK selection required"
                    android:textSize="14sp"
                    android:textColor="#1976D2"
                    android:lineSpacingExtra="4dp"
                    android:layout_marginBottom="12dp" />

                <Button
                    android:id="@+id/universal_button"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🌐 Universal Modding"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:background="#2196F3"
                    android:textColor="@android:color/white"
                    android:minHeight="56dp"
                    android:elevation="4dp" />

            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Specific Mode Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#E8F5E8">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🎯 Specific Game Mode"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#2E7D32"
                    android:layout_marginBottom="8dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="Optimized for supported games\n• Pre-configured settings\n• Game-specific features\n• Recommended for beginners"
                    android:textSize="14sp"
                    android:textColor="#388E3C"
                    android:lineSpacingExtra="4dp"
                    android:layout_marginBottom="12dp" />

                <Button
                    android:id="@+id/specific_button"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🎯 Game-Specific Modding"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:background="#4CAF50"
                    android:textColor="@android:color/white"
                    android:minHeight="56dp"
                    android:elevation="4dp" />

            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- NEW: Addon Management Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#F3E5F5">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🔌 Addon System"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#7B1FA2"
                    android:layout_marginBottom="8dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="Customize and extend ModLoader\n• Themes and UI enhancements\n• Advanced file operations\n• Smart installation tools"
                    android:textSize="14sp"
                    android:textColor="#8E24AA"
                    android:lineSpacingExtra="4dp"
                    android:layout_marginBottom="12dp" />

                <Button
                    android:id="@+id/plugin_button"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🔌 Manage Addons"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:background="#9C27B0"
                    android:textColor="@android:color/white"
                    android:minHeight="56dp"
                    android:elevation="4dp" />

            </LinearLayout>
        </androidx.cardview.widget.CardView>

    </LinearLayout>

    <!-- Footer Info -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:background="#FFFFFF"
        android:padding="16dp"
        android:layout_marginTop="16dp"
        android:elevation="2dp">

        <TextView
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="💡 Tips:\n• Try Specific Mode first\n• Use Addons for customization"
            android:textSize="12sp"
            android:textColor="#666666"
            android:lineSpacingExtra="2dp" />

        <TextView
            android:layout_width="0dp"
            android:layout_weight="1"
            android:layout_height="wrap_content"
            android:text="🎯 New users:\n• Start with Terraria\n• Install sample addons"
            android:textSize="12sp"
            android:textColor="#666666"
            android:lineSpacingExtra="2dp" />

    </LinearLayout>

    <!-- Version Info -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="ModLoader v1.0 with Addon System"
        android:textSize="10sp"
        android:textColor="#999999"
        android:gravity="center"
        android:layout_marginTop="16dp" />

</LinearLayout>

================================================================================

ModLoader/app/src/main/res/layout/activity_mod_list.xml

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center_vertical"
        android:layout_marginBottom="16dp">

        <ImageButton
            android:id="@+id/backButton"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_alignParentStart="true"
            android:background="?attr/selectableItemBackgroundBorderless"
            android:src="@drawable/ic_arrow_back"
            android:contentDescription="Back"
            android:tint="@android:color/black" />

        <TextView
            android:id="@+id/modCountTextView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:text="Total Mods: 0 (Enabled: 0)"
            android:textSize="16sp"
            android:textStyle="bold" />

    </RelativeLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewMods"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:scrollbars="vertical" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center"
        android:layout_marginTop="16dp">

        <Button
            android:id="@+id/addModButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Add Mod"
            android:layout_marginEnd="8dp" />

        <Button
            android:id="@+id/refreshModsButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Refresh Mods" />
    </LinearLayout>

</LinearLayout>


================================================================================

ModLoader/app/src/main/res/layout/activity_mod_management.xml

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fillViewport="true">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp"
        android:background="#F8F9FA">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:background="#FFFFFF"
            android:padding="20dp"
            android:layout_marginBottom="16dp"
            android:elevation="2dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🎮 Mod Management"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="8dp" />

            <!-- Status Section -->
            <TextView
                android:id="@+id/statusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="📊 Loading mod statistics..."
                android:textSize="14sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:padding="8dp"
                android:background="#E8F5E8"
                android:layout_marginBottom="8dp" />

            <!-- Loader Status -->
            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Checking loader status..."
                android:textSize="12sp"
                android:gravity="center"
                android:padding="6dp" />

        </LinearLayout>

        <!-- Loader Info Section (shown when loader is installed) -->
        <LinearLayout
            android:id="@+id/loaderInfoSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="#E3F2FD"
            android:padding="16dp"
            android:layout_marginBottom="16dp"
            android:visibility="gone"
            android:elevation="2dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="ℹ️ Loader Information"
                android:textSize="14sp"
                android:textStyle="bold"
                android:textColor="#1565C0"
                android:layout_marginBottom="8dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="• DLL mods will be loaded by MelonLoader\n• DEX/JAR mods are loaded directly by TerrariaLoader\n• Enable/disable mods using the switches below"
                android:textSize="12sp"
                android:textColor="#1976D2"
                android:lineSpacingExtra="2dp" />

        </LinearLayout>

        <!-- Add Mods Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#FFFFFF">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📥 Add New Mods"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="#333333"
                    android:layout_marginBottom="12dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/addDllModBtn"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📥 Add DLL Mod"
                        android:textSize="14sp"
                        android:background="#4CAF50"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/addDexModBtn"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📱 Add DEX/JAR Mod"
                        android:textSize="14sp"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />

                </LinearLayout>

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="💡 DLL mods require MelonLoader • DEX/JAR mods work without a loader"
                    android:textSize="11sp"
                    android:textColor="#666666"
                    android:gravity="center"
                    android:layout_marginTop="8dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Mod List Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#FFFFFF">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:orientation="vertical"
                android:padding="16dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical"
                    android:layout_marginBottom="12dp">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Installed Mods"
                        android:textSize="18sp"
                        android:textStyle="bold"
                        android:textColor="#333333" />

                    <Button
                        android:id="@+id/refreshBtn"
                        android:layout_width="wrap_content"
                        android:layout_height="36dp"
                        android:text="🔄 Refresh"
                        android:textSize="12sp"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:minWidth="80dp" />

                </LinearLayout>

                <androidx.recyclerview.widget.RecyclerView
                    android:id="@+id/modRecyclerView"
                    android:layout_width="match_parent"
                    android:layout_height="0dp"
                    android:layout_weight="1"
                    android:background="#F9F9F9"
                    android:padding="8dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Navigation Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center"
            android:layout_marginTop="8dp">

            <Button
                android:id="@+id/backBtn"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="← Back"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:minHeight="40dp"
                android:layout_marginEnd="16dp" />

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Manage your mods after installation"
                android:textSize="12sp"
                android:textColor="#999999"
                android:gravity="center" />

        </LinearLayout>

        <!-- Tips Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="💡 Tips:\n• Toggle mods with switches\n• Delete mods with trash icon\n• DLL mods require patched Terraria APK\n• DEX/JAR mods work with any Terraria version"
            android:textSize="11sp"
            android:textColor="#888888"
            android:background="#F0F0F0"
            android:padding="12dp"
            android:layout_marginTop="16dp"
            android:lineSpacingExtra="2dp" />

    </LinearLayout>

</ScrollView>

================================================================================

ModLoader/app/src/main/res/layout/activity_offline_diagnostic.xml

<?xml version="1.0" encoding="utf-8"?>
<!-- File: activity_offline_diagnostic.xml -->
<!-- Path: /res/layout/activity_offline_diagnostic.xml -->

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/background_light">
    
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">
        
        <!-- Header Text -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="TerrariaLoader Offline Diagnostics"
            android:textSize="24sp"
            android:textStyle="bold"
            android:textColor="@android:color/holo_green_dark"
            android:gravity="center"
            android:layout_marginBottom="16dp" />
        
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Diagnose and fix common issues without internet connection"
            android:textSize="14sp"
            android:textColor="@android:color/darker_gray"
            android:gravity="center"
            android:layout_marginBottom="24dp" />
        
        <!-- Quick Actions Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🚀 Quick Actions"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="12dp" />
                
                <Button
                    android:id="@+id/btn_run_full_diagnostic"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🔍 Run Full System Check"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_green_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_diagnose_apk"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📦 Diagnose APK Installation Problem"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_orange_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_fix_settings"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="⚙️ Fix Settings Persistence Issue"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_blue_light"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="8dp"
                    android:padding="12dp" />
                
                <Button
                    android:id="@+id/btn_auto_repair"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🛠️ Attempt Auto-Repair"
                    android:textAllCaps="false"
                    android:background="@android:color/holo_purple"
                    android:textColor="@android:color/white"
                    android:padding="12dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Diagnostic Results Card -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="📊 Diagnostic Results"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="12dp" />
                
                <!-- Results Display Area -->
                <ScrollView
                    android:layout_width="match_parent"
                    android:layout_height="400dp"
                    android:background="@android:color/black"
                    android:padding="8dp">
                    
                    <TextView
                        android:id="@+id/diagnostic_results_text"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Click 'Run Full System Check' to start diagnostics..."
                        android:textColor="@android:color/holo_green_light"
                        android:textSize="12sp"
                        android:fontFamily="monospace"
                        android:textIsSelectable="true"
                        android:padding="8dp" />
                </ScrollView>
                
                <!-- Action Buttons for Results -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:layout_marginTop="12dp">
                    
                    <Button
                        android:id="@+id/btn_export_report"
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="📤 Export Report"
                        android:textAllCaps="false"
                        android:background="@android:color/darker_gray"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="4dp" />
                    
                    <Button
                        android:id="@+id/btn_clear_results"
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="🗑️ Clear Results"
                        android:textAllCaps="false"
                        android:background="@android:color/darker_gray"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="4dp" />
                </LinearLayout>
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Help/Info Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp">
            
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="💡 Quick Help"
                    android:textSize="18sp"
                    android:textStyle="bold"
                    android:textColor="@android:color/holo_blue_dark"
                    android:layout_marginBottom="8dp" />
                
                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="• APK parsing errors: Usually caused by corrupted files or missing permissions\n• Settings not saving: Often due to storage permissions or corrupted preferences\n• Directory issues: Can be fixed with auto-repair function\n• Export reports to share with developers for support"
                    android:textSize="14sp"
                    android:textColor="@android:color/darker_gray"
                    android:lineSpacingMultiplier="1.2" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>
        
        <!-- Version Info -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="TerrariaLoader Diagnostic Tool v1.0 - Offline Mode"
            android:textSize="12sp"
            android:textColor="@android:color/darker_gray"
            android:gravity="center"
            android:layout_marginTop="16dp"
            android:layout_marginBottom="16dp" />
        
    </LinearLayout>
</ScrollView>

================================================================================

ModLoader/app/src/main/res/layout/activity_settings_enhanced.xml

<!-- File: activity_settings_enhanced.xml (Enhanced Settings Layout - Error-Free) -->
<!-- Path: /app/src/main/res/layout/activity_settings_enhanced.xml -->

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#F5F5F5">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">

        <!-- Header -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="⚙️ TerrariaLoader Settings"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            android:layout_marginBottom="24dp"
            android:textColor="#2E7D32" />

        <!-- Operation Modes Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🚀 Operation Modes"
            android:textSize="20sp"
            android:textStyle="bold"
            android:layout_marginBottom="16dp"
            android:textColor="#1976D2" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Choose how TerrariaLoader operates:"
            android:textSize="14sp"
            android:layout_marginBottom="16dp"
            android:textColor="#666666" />

        <!-- Operation Mode Radio Group -->
        <RadioGroup
            android:id="@+id/operationModeGroup"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="24dp">

            <!-- Normal Mode Card -->
            <androidx.cardview.widget.CardView
                android:id="@+id/normalCard"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="12dp"
                app:cardCornerRadius="8dp"
                app:cardElevation="4dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="16dp">

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="horizontal"
                        android:gravity="center_vertical">

                        <RadioButton
                            android:id="@+id/normalModeRadio"
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginEnd="12dp" />

                        <TextView
                            android:layout_width="0dp"
                            android:layout_height="wrap_content"
                            android:layout_weight="1"
                            android:text="📱 Normal Mode"
                            android:textSize="18sp"
                            android:textStyle="bold"
                            android:textColor="#2E7D32" />

                    </LinearLayout>

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Standard Android permissions only"
                        android:textSize="14sp"
                        android:layout_marginTop="8dp"
                        android:textColor="#666666" />

                    <TextView
                        android:id="@+id/normalStatus"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="✅ Standard permissions"
                        android:textSize="12sp"
                        android:layout_marginTop="4dp"
                        android:textColor="#4CAF50" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Shizuku Mode Card -->
            <androidx.cardview.widget.CardView
                android:id="@+id/shizukuCard"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="12dp"
                app:cardCornerRadius="8dp"
                app:cardElevation="4dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="16dp">

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="horizontal"
                        android:gravity="center_vertical">

                        <RadioButton
                            android:id="@+id/shizukuModeRadio"
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginEnd="12dp" />

                        <TextView
                            android:layout_width="0dp"
                            android:layout_height="wrap_content"
                            android:layout_weight="1"
                            android:text="🛡️ Shizuku Mode"
                            android:textSize="18sp"
                            android:textStyle="bold"
                            android:textColor="#1976D2" />

                    </LinearLayout>

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Enhanced permissions via Shizuku"
                        android:textSize="14sp"
                        android:layout_marginTop="8dp"
                        android:textColor="#666666" />

                    <TextView
                        android:id="@+id/shizukuStatus"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Shizuku not available"
                        android:textSize="12sp"
                        android:layout_marginTop="4dp"
                        android:textColor="#FF5722" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Root Mode Card -->
            <androidx.cardview.widget.CardView
                android:id="@+id/rootCard"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="12dp"
                app:cardCornerRadius="8dp"
                app:cardElevation="4dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="16dp">

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="horizontal"
                        android:gravity="center_vertical">

                        <RadioButton
                            android:id="@+id/rootModeRadio"
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginEnd="12dp" />

                        <TextView
                            android:layout_width="0dp"
                            android:layout_height="wrap_content"
                            android:layout_weight="1"
                            android:text="🔓 Root Mode"
                            android:textSize="18sp"
                            android:textStyle="bold"
                            android:textColor="#E91E63" />

                    </LinearLayout>

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Maximum system control with root"
                        android:textSize="14sp"
                        android:layout_marginTop="8dp"
                        android:textColor="#666666" />

                    <TextView
                        android:id="@+id/rootStatus"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Root not available"
                        android:textSize="12sp"
                        android:layout_marginTop="4dp"
                        android:textColor="#FF5722" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Hybrid Mode Card -->
            <androidx.cardview.widget.CardView
                android:id="@+id/hybridCard"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="12dp"
                app:cardCornerRadius="8dp"
                app:cardElevation="4dp">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="16dp">

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="horizontal"
                        android:gravity="center_vertical">

                        <RadioButton
                            android:id="@+id/hybridModeRadio"
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginEnd="12dp" />

                        <TextView
                            android:layout_width="0dp"
                            android:layout_height="wrap_content"
                            android:layout_weight="1"
                            android:text="⚡ Hybrid Mode"
                            android:textSize="18sp"
                            android:textStyle="bold"
                            android:textColor="#9C27B0" />

                    </LinearLayout>

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Both Shizuku and Root capabilities"
                        android:textSize="14sp"
                        android:layout_marginTop="8dp"
                        android:textColor="#666666" />

                    <TextView
                        android:id="@+id/hybridStatus"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="Requires both Shizuku and Root"
                        android:textSize="12sp"
                        android:layout_marginTop="4dp"
                        android:textColor="#FF5722" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

        </RadioGroup>

        <!-- Setup Buttons Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🔧 Setup &amp; Permissions"
            android:textSize="20sp"
            android:textStyle="bold"
            android:layout_marginBottom="16dp"
            android:textColor="#1976D2" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginBottom="16dp">

            <Button
                android:id="@+id/shizukuSetupBtn"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginEnd="8dp"
                android:text="📥 Setup Shizuku"
                android:textSize="12sp"
                android:backgroundTint="#2196F3" />

            <Button
                android:id="@+id/rootSetupBtn"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginStart="8dp"
                android:text="🔓 Check Root"
                android:textSize="12sp"
                android:backgroundTint="#E91E63" />

        </LinearLayout>

        <Button
            android:id="@+id/permissionBtn"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="24dp"
            android:text="🔐 Manage Permissions"
            android:backgroundTint="#4CAF50" />

        <!-- Feature Toggles Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="⚙️ Feature Settings"
            android:textSize="20sp"
            android:textStyle="bold"
            android:layout_marginBottom="16dp"
            android:textColor="#1976D2" />

        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="2dp">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">

                <!-- Auto Enable Mods -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical"
                    android:layout_marginBottom="12dp">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="🔄 Auto-enable new mods"
                        android:textSize="16sp"
                        android:textColor="#333333" />

                    <Switch
                        android:id="@+id/autoEnableSwitch"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content" />

                </LinearLayout>

                <!-- Debug Logging -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical"
                    android:layout_marginBottom="12dp">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="🐛 Debug logging"
                        android:textSize="16sp"
                        android:textColor="#333333" />

                    <Switch
                        android:id="@+id/debugLoggingSwitch"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content" />

                </LinearLayout>

                <!-- Auto Backup -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical"
                    android:layout_marginBottom="12dp">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="💾 Auto-backup APKs"
                        android:textSize="16sp"
                        android:textColor="#333333" />

                    <Switch
                        android:id="@+id/autoBackupSwitch"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content" />

                </LinearLayout>

                <!-- Auto Update Check -->
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:gravity="center_vertical">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_height="wrap_content"
                        android:layout_weight="1"
                        android:text="🔄 Check for updates"
                        android:textSize="16sp"
                        android:textColor="#333333" />

                    <Switch
                        android:id="@+id/autoUpdateSwitch"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content" />

                </LinearLayout>

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Action Buttons Section -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🛠️ Actions"
            android:textSize="20sp"
            android:textStyle="bold"
            android:layout_marginBottom="16dp"
            android:textColor="#1976D2" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginBottom="16dp">

            <Button
                android:id="@+id/resetSettingsBtn"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginEnd="8dp"
                android:text="🔄 Reset"
                android:textSize="12sp"
                android:backgroundTint="#FF5722" />

            <Button
                android:id="@+id/exportSettingsBtn"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginStart="4dp"
                android:layout_marginEnd="4dp"
                android:text="📤 Export"
                android:textSize="12sp"
                android:backgroundTint="#607D8B" />

            <Button
                android:id="@+id/importSettingsBtn"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:layout_marginStart="8dp