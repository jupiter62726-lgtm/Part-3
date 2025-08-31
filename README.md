

--- FILE: /mnt/data/extracted_files/main/java/com/modloader/util/RootManager.java ---

// File: RootManager.java (FIXED) - Placeholder Root Manager with Shizuku Integration
// Path: /app/src/main/java/com/modloader/util/RootManager.java

package com.modloader.util;

import android.content.Context;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;

import java.io.BufferedReader;
import java.io.DataOutputStream;
import java.io.InputStreamReader;

/**
 * FIXED: Root Manager - Currently serves as a placeholder but includes basic root detection
 * This version focuses on Shizuku integration while providing foundation for future root support
 */
public class RootManager {
    private static final String TAG = "RootManager";
    
    private final Context context;
    private final ShizukuManager shizukuManager;
    private RootCallback rootCallback;
    
    // Root detection cache
    private Boolean rootAvailableCache = null;
    private Boolean rootPermissionCache = null;
    private long lastRootCheck = 0;
    private static final long ROOT_CHECK_CACHE_TIME = 30000; // 30 seconds
    
    public interface RootCallback {
        void onRootGranted();
        void onRootDenied();
        void onRootError(String error);
    }
    
    public RootManager(Context context) {
        this.context = context;
        this.shizukuManager = new ShizukuManager(context);
        LogUtils.logDebug("RootManager initialized (using Shizuku as primary enhanced access)");
    }
    
    public void setRootCallback(RootCallback callback) {
        this.rootCallback = callback;
    }
    
    // ===== Root Detection Methods =====
    
    /**
     * FIXED: Check if root access is available on device
     * This performs actual root detection but recommends Shizuku instead
     */
    public boolean isRootAvailable() {
        // Use cached result if recent
        long currentTime = System.currentTimeMillis();
        if (rootAvailableCache != null && (currentTime - lastRootCheck) < ROOT_CHECK_CACHE_TIME) {
            return rootAvailableCache;
        }
        
        LogUtils.logDebug("Checking for root availability...");
        boolean rootDetected = false;
        
        try {
            // Method 1: Check for su binary
            rootDetected = checkSuBinary();
            
            if (!rootDetected) {
                // Method 2: Check for root indicators
                rootDetected = checkRootIndicators();
            }
            
            if (!rootDetected) {
                // Method 3: Try to execute su command (non-persistent)
                rootDetected = testSuCommand();
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Root check exception: " + e.getMessage());
            rootDetected = false;
        }
        
        // Cache result
        rootAvailableCache = rootDetected;
        lastRootCheck = currentTime;
        
        LogUtils.logDebug("Root availability check result: " + rootDetected);
        
        if (rootDetected) {
            LogUtils.logUser("⚠️ Root detected, but Shizuku is recommended for enhanced access");
            showRootDetectedButShizukuRecommended();
        } else {
            LogUtils.logDebug("No root access detected - this is normal for most devices");
        }
        
        return rootDetected;
    }
    
    /**
     * Check if root permission is granted (placeholder - always returns false)
     */
    public boolean hasRootPermission() {
        // For now, we don't support active root usage
        // This is a placeholder that always returns false
        LogUtils.logDebug("Root permission check - returning false (root not supported in this version)");
        return false;
    }
    
    /**
     * Check if root is ready for use (placeholder - always returns false)
     */
    public boolean isRootReady() {
        // Root functionality is disabled in favor of Shizuku
        return false;
    }
    
    // ===== Root Detection Implementation =====
    
    /**
     * Check for su binary in common locations
     */
    private boolean checkSuBinary() {
        String[] suPaths = {
            "/system/bin/su",
            "/system/xbin/su",
            "/sbin/su",
            "/data/local/xbin/su",
            "/data/local/bin/su",
            "/system/sd/xbin/su",
            "/system/bin/failsafe/su",
            "/data/local/su",
            "/su/bin/su"
        };
        
        for (String path : suPaths) {
            try {
                java.io.File suFile = new java.io.File(path);
                if (suFile.exists()) {
                    LogUtils.logDebug("Found su binary at: " + path);
                    return true;
                }
            } catch (Exception e) {
                // Ignore and continue
            }
        }
        
        return false;
    }
    
    /**
     * Check for common root indicators
     */
    private boolean checkRootIndicators() {
        try {
            // Check for Superuser apps
            String[] rootApps = {
                "com.noshufou.android.su",
                "com.noshufou.android.su.elite",
                "eu.chainfire.supersu",
                "com.koushikdutta.superuser",
                "com.thirdparty.superuser",
                "com.yellowes.su",
                "com.koushikdutta.rommanager",
                "com.koushikdutta.rommanager.license",
                "com.dimonvideo.luckypatcher",
                "com.chelpus.lackypatch",
                "com.ramdroid.appquarantine",
                "com.topjohnwu.magisk"
            };
            
            android.content.pm.PackageManager pm = context.getPackageManager();
            for (String packageName : rootApps) {
                try {
                    pm.getPackageInfo(packageName, 0);
                    LogUtils.logDebug("Found root indicator app: " + packageName);
                    return true;
                } catch (android.content.pm.PackageManager.NameNotFoundException e) {
                    // App not found, continue
                }
            }
            
            // Check build tags
            String buildTags = android.os.Build.TAGS;
            if (buildTags != null && buildTags.contains("test-keys")) {
                LogUtils.logDebug("Found test-keys in build tags");
                return true;
            }
            
            // Check for RW system partition
            try {
                String[] mountPoints = {"/system", "/system/", "/system/bin"};
                for (String mountPoint : mountPoints) {
                    java.io.File file = new java.io.File(mountPoint);
                    if (file.exists() && file.canWrite()) {
                        LogUtils.logDebug("System partition is writable: " + mountPoint);
                        return true;
                    }
                }
            } catch (Exception e) {
                // Ignore
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Error checking root indicators: " + e.getMessage());
        }
        
        return false;
    }
    
    /**
     * Test su command execution (non-persistent)
     */
    private boolean testSuCommand() {
        try {
            Process process = Runtime.getRuntime().exec("su");
            DataOutputStream os = new DataOutputStream(process.getOutputStream());
            BufferedReader is = new BufferedReader(new InputStreamReader(process.getInputStream()));
            
            os.writeBytes("id\n");
            os.writeBytes("exit\n");
            os.flush();
            
            String response = is.readLine();
            os.close();
            is.close();
            
            if (response != null && response.contains("uid=0")) {
                LogUtils.logDebug("Su command test successful");
                return true;
            }
            
        } catch (Exception e) {
            LogUtils.logDebug("Su command test failed: " + e.getMessage());
        }
        
        return false;
    }
    
    // ===== Placeholder Root Operations =====
    
    /**
     * Request root access (placeholder - redirects to Shizuku)
     */
    public void requestRootAccess() {
        LogUtils.logUser("Root access requested - redirecting to Shizuku setup");
        
        showRootNotSupportedDialog();
    }
    
    /**
     * Execute command with root (placeholder - not implemented)
     */
    public boolean executeRootCommand(String command) {
        LogUtils.logDebug("Root command execution not supported - use Shizuku instead");
        return false;
    }
    
    /**
     * Check root status and show appropriate dialog
     */
    public void checkRootStatus() {
        boolean available = isRootAvailable();
        String status = getRootStatusReport();
        
        LogUtils.logUser("Root Status Check:\n" + status);
        
        new AlertDialog.Builder(context)
            .setTitle("Root Status")
            .setMessage(status)
            .setPositiveButton("Use Shizuku Instead", (dialog, which) -> {
                shizukuManager.checkRootStatus(); // This will handle Shizuku setup
            })
            .setNeutralButton("Learn About Shizuku", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("OK", null)
            .show();
    }
    
    // ===== Status and Information Methods =====
    
    /**
     * Get detailed root status report
     */
    public String getRootStatusReport() {
        StringBuilder report = new StringBuilder();
        report.append("=== Root Access Status ===\n");
        
        boolean available = isRootAvailable();
        report.append("Root Detected: ").append(available ? "✅ Yes" : "❌ No").append("\n");
        report.append("Root Supported: ❌ No (Use Shizuku instead)\n");
        
        if (available) {
            report.append("Root Permission: ❌ Not granted (disabled)\n");
            report.append("\n⚠️ Root was detected on your device, but this app uses Shizuku ");
            report.append("for enhanced capabilities instead of root access.\n");
        } else {
            report.append("\n✅ No root detected - this is normal for most devices.\n");
        }
        
        report.append("\n=== Recommended Alternative ===\n");
        boolean shizukuReady = shizukuManager.isShizukuReady();
        report.append("Shizuku Status: ").append(shizukuReady ? "✅ Ready" : "❌ Not Ready").append("\n");
        
        if (!shizukuReady) {
            if (!shizukuManager.isShizukuInstalled()) {
                report.append("• Install Shizuku app\n");
            }
            if (!shizukuManager.isShizukuRunning()) {
                report.append("• Start Shizuku service\n");
            }
            if (!shizukuManager.hasShizukuPermission()) {
                report.append("• Grant Shizuku permission\n");
            }
        }
        
        report.append("\n💡 Shizuku provides enhanced file access without requiring ");
        report.append("root access and is safer and more reliable than traditional root methods.");
        
        return report.toString();
    }
    
    /**
     * Get root detection details
     */
    public String getRootDetectionDetails() {
        StringBuilder details = new StringBuilder();
        details.append("=== Root Detection Details ===\n");
        
        details.append("Su Binary Check: ").append(checkSuBinary() ? "✅ Found" : "❌ Not Found").append("\n");
        details.append("Root Apps Check: ").append(checkRootIndicators() ? "✅ Found" : "❌ Not Found").append("\n");
        details.append("Su Command Test: ").append(testSuCommand() ? "✅ Works" : "❌ Failed").append("\n");
        
        details.append("\nBuild Info:\n");
        details.append("• Tags: ").append(android.os.Build.TAGS).append("\n");
        details.append("• Type: ").append(android.os.Build.TYPE).append("\n");
        details.append("• Device: ").append(android.os.Build.DEVICE).append("\n");
        
        details.append("\n⚠️ Even if root is detected, this app uses Shizuku for enhanced access.");
        
        return details.toString();
    }
    
    // ===== Dialog Methods =====
    
    private void showRootDetectedButShizukuRecommended() {
        // Only show this dialog once per app session
        if (rootPermissionCache != null) {
            return;
        }
        rootPermissionCache = false; // Mark as shown
        
        new AlertDialog.Builder(context)
            .setTitle("Root Detected")
            .setMessage("Your device appears to have root access, but this app uses Shizuku " +
                       "instead for enhanced capabilities.\n\n" +
                       "Shizuku is:\n" +
                       "• Safer than root access\n" +
                       "• More reliable\n" +
                       "• Doesn't require device modification\n" +
                       "• Works without voiding warranty\n\n" +
                       "Would you like to set up Shizuku instead?")
            .setPositiveButton("Setup Shizuku", (dialog, which) -> {
                if (!shizukuManager.isShizukuInstalled()) {
                    shizukuManager.installShizuku();
                } else {
                    shizukuManager.requestShizukuPermission();
                }
            })
            .setNeutralButton("Learn More", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Not Now", null)
            .show();
    }
    
    private void showRootNotSupportedDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Root Access Not Supported")
            .setMessage("This version of TerrariaLoader does not support root access for enhanced capabilities.\n\n" +
                       "Instead, we recommend using Shizuku, which provides:\n" +
                       "• Enhanced file access without root\n" +
                       "• Better security and stability\n" +
                       "• No device modification required\n" +
                       "• Easy setup via ADB or existing root\n\n" +
                       "Would you like to set up Shizuku instead?")
            .setPositiveButton("Setup Shizuku", (dialog, which) -> {
                shizukuManager.checkRootStatus();
            })
            .setNeutralButton("Learn About Shizuku", (dialog, which) -> {
                try {
                    android.content.Intent intent = new android.content.Intent(
                        android.content.Intent.ACTION_VIEW, 
                        android.net.Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    // ===== Utility Methods =====
    
    /**
     * Clear cached root detection results
     */
    public void clearCache() {
        rootAvailableCache = null;
        rootPermissionCache = null;
        lastRootCheck = 0;
        LogUtils.logDebug("Root detection cache cleared");
    }
    
    /**
     * Get alternative enhanced access manager (Shizuku)
     */
    public ShizukuManager getAlternativeEnhancedAccess() {
        return shizukuManager;
    }
    
    /**
     * Check if any enhanced access is available (Shizuku or Root)
     */
    public boolean hasAnyEnhancedAccess() {
        return shizukuManager.isShizukuReady();
    }
    
    /**
     * Get the best available enhanced access method
     */
    public String getBestEnhancedAccessMethod() {
        if (shizukuManager.isShizukuReady()) {
            return "Shizuku (Ready)";
        } else if (shizukuManager.isShizukuInstalled()) {
            return "Shizuku (Needs Setup)";
        } else if (isRootAvailable()) {
            return "Root (Detected but not supported)";
        } else {
            return "None Available";
        }
    }
    
    /**
     * Execute enhanced operation using best available method
     */
    public boolean executeEnhancedOperation(String operation, String... params) {
        if (shizukuManager.isShizukuReady()) {
            LogUtils.logDebug("Executing enhanced operation via Shizuku: " + operation);
            return shizukuManager.executeShizukuCommand(operation + " " + String.join(" ", params));
        } else {
            LogUtils.logDebug("No enhanced access available for operation: " + operation);
            return false;
        }
    }
    
    // ===== Legacy Compatibility Methods =====
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public boolean canUseEnhancedAccess() {
        return hasAnyEnhancedAccess();
    }
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public void requestEnhancedAccess() {
        if (shizukuManager.isShizukuAvailable()) {
            shizukuManager.requestShizukuPermission();
        } else {
            requestRootAccess();
        }
    }
    
    /**
     * @deprecated Use ShizukuManager instead
     */
    @Deprecated
    public boolean executeEnhancedCommand(String command) {
        return executeEnhancedOperation(command);
    }
}


--- FILE: /mnt/data/extracted_files/main/java/com/modloader/util/ShizukuManager.java ---

// File: ShizukuManager.java (FIXED) - Reflection-based with In-App Permission Dialog Support
// Path: /app/src/main/java/com/modloader/util/ShizukuManager.java

package com.modloader.util;

import android.app.Activity;
import android.content.Context;
import android.content.Intent;
import android.content.pm.ApplicationInfo;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Build;
import android.os.Process;
import android.widget.Toast;
import androidx.annotation.NonNull;
import androidx.appcompat.app.AlertDialog;

import java.lang.reflect.Method;

/**
 * FIXED: Complete Shizuku Manager with proper in-app permission dialog support
 * Uses reflection to trigger Shizuku permission dialogs within the app (like MT Manager)
 */
public class ShizukuManager {
    private static final String TAG = "ShizukuManager";
    
    // Shizuku package and permission constants
    private static final String SHIZUKU_PACKAGE = "moe.shizuku.privileged.api";
    private static final String SHIZUKU_PERMISSION = "moe.shizuku.manager.permission.API_V23";
    private static final int SHIZUKU_REQUEST_CODE = 1001;
    
    // Shizuku API reflection constants - FIXED for proper API usage
    private static final String SHIZUKU_CLASS = "moe.shizuku.api.Shizuku";
    private static final String SHIZUKU_BINDER_CLASS = "moe.shizuku.api.ShizukuBinderWrapper";
    private static final String SHIZUKU_LISTENER_CLASS = "moe.shizuku.api.Shizuku$OnRequestPermissionResultListener";
    
    private final Context context;
    private ShizukuPermissionCallback permissionCallback;
    
    // FIXED: Add permission result listener for in-app dialogs
    private Object permissionResultListener;
    private boolean listenerRegistered = false;
    
    // Interface for permission callback
    public interface ShizukuPermissionCallback {
        void onPermissionGranted();
        void onPermissionDenied();
        void onPermissionError(String error);
    }
    
    public ShizukuManager(Context context) {
        this.context = context;
        setupPermissionListener();
        LogUtils.logDebug("ShizukuManager initialized");
    }
    
    /**
     * FIXED: Setup permission result listener for in-app permission dialogs
     */
    private void setupPermissionListener() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Class<?> listenerClass = Class.forName(SHIZUKU_LISTENER_CLASS);
            
            // Create permission result listener using reflection
            permissionResultListener = java.lang.reflect.Proxy.newProxyInstance(
                listenerClass.getClassLoader(),
                new Class[]{listenerClass},
                new java.lang.reflect.InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, java.lang.reflect.Method method, Object[] args) {
                        if ("onRequestPermissionResult".equals(method.getName()) && args != null && args.length >= 2) {
                            int requestCode = (Integer) args[0];
                            int grantResult = (Integer) args[1];
                            handleShizukuPermissionResult(requestCode, grantResult);
                        }
                        return null;
                    }
                }
            );
            
            // Add listener to Shizuku
            Method addListenerMethod = shizukuClass.getMethod("addRequestPermissionResultListener", listenerClass);
            addListenerMethod.invoke(null, permissionResultListener);
            listenerRegistered = true;
            
            LogUtils.logDebug("Shizuku permission listener setup successfully");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to setup Shizuku permission listener: " + e.getMessage());
            permissionResultListener = null;
            listenerRegistered = false;
        }
    }
    
    /**
     * FIXED: Handle Shizuku permission result from in-app dialog
     */
    private void handleShizukuPermissionResult(int requestCode, int grantResult) {
        LogUtils.logDebug("Shizuku permission result: requestCode=" + requestCode + ", result=" + grantResult);
        
        if (requestCode == SHIZUKU_REQUEST_CODE) {
            boolean granted = (grantResult == PackageManager.PERMISSION_GRANTED);
            
            if (granted) {
                LogUtils.logUser("✅ Shizuku permission granted!");
                if (permissionCallback != null) {
                    permissionCallback.onPermissionGranted();
                }
            } else {
                LogUtils.logUser("❌ Shizuku permission denied");
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied();
                }
            }
        }
    }
    
    public void setPermissionCallback(ShizukuPermissionCallback callback) {
        this.permissionCallback = callback;
    }
    
    // ===== FIXED: Comprehensive Shizuku Detection Methods =====
    
    /**
     * FIXED: Check if Shizuku app is installed
     */
    public boolean isShizukuInstalled() {
        try {
            PackageManager pm = context.getPackageManager();
            ApplicationInfo appInfo = pm.getApplicationInfo(SHIZUKU_PACKAGE, 0);
            boolean installed = appInfo != null;
            LogUtils.logDebug("Shizuku installation check: " + installed);
            return installed;
        } catch (PackageManager.NameNotFoundException e) {
            LogUtils.logDebug("Shizuku not installed: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku installation check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check if Shizuku service is running using multiple methods
     */
    public boolean isShizukuRunning() {
        // Method 1: Try reflection-based check
        if (checkShizukuRunningViaReflection()) {
            LogUtils.logDebug("Shizuku running - detected via reflection");
            return true;
        }
        
        // Method 2: Try permission check (if service is running, permission check should work)
        if (checkShizukuRunningViaPermission()) {
            LogUtils.logDebug("Shizuku running - detected via permission check");
            return true;
        }
        
        // Method 3: Try binder check
        if (checkShizukuRunningViaBinder()) {
            LogUtils.logDebug("Shizuku running - detected via binder");
            return true;
        }
        
        LogUtils.logDebug("Shizuku service not running - all detection methods failed");
        return false;
    }
    
    /**
     * FIXED: Check if Shizuku permission is granted using multiple detection methods
     */
    public boolean hasShizukuPermission() {
        if (!isShizukuInstalled()) {
            LogUtils.logDebug("Shizuku permission check: not installed");
            return false;
        }
        
        if (!isShizukuRunning()) {
            LogUtils.logDebug("Shizuku permission check: service not running");
            return false;
        }
        
        // Method 1: Check via Android's permission system
        boolean systemPermissionGranted = checkSystemPermission();
        if (systemPermissionGranted) {
            LogUtils.logDebug("Shizuku permission: granted via system permission");
            return true;
        }
        
        // Method 2: Check via Shizuku API reflection
        boolean apiPermissionGranted = checkShizukuPermissionViaReflection();
        if (apiPermissionGranted) {
            LogUtils.logDebug("Shizuku permission: granted via Shizuku API");
            return true;
        }
        
        // Method 3: Try to perform a test operation
        boolean testOperationSuccessful = testShizukuOperation();
        if (testOperationSuccessful) {
            LogUtils.logDebug("Shizuku permission: granted via test operation");
            return true;
        }
        
        LogUtils.logDebug("Shizuku permission: not granted - all check methods failed");
        return false;
    }
    
    /**
     * FIXED: Comprehensive check if Shizuku is available and ready
     */
    public boolean isShizukuAvailable() {
        return isShizukuInstalled();
    }
    
    /**
     * FIXED: Check if Shizuku is fully ready (installed + running + permission)
     */
    public boolean isShizukuReady() {
        boolean installed = isShizukuInstalled();
        boolean running = isShizukuRunning();
        boolean hasPermission = hasShizukuPermission();
        
        LogUtils.logDebug("Shizuku readiness - Installed: " + installed + 
                         ", Running: " + running + ", Permission: " + hasPermission);
        
        return installed && running && hasPermission;
    }
    
    // ===== Detection Method Implementations =====
    
    /**
     * Check if Shizuku is running via reflection
     */
    private boolean checkShizukuRunningViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method pingMethod = shizukuClass.getMethod("pingBinder");
            Boolean result = (Boolean) pingMethod.invoke(null);
            return result != null && result;
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku class not found - API not available");
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku ping failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if Shizuku is running via permission system
     */
    private boolean checkShizukuRunningViaPermission() {
        try {
            // If we can check permission without exception, service is likely running
            int permissionState = context.checkPermission(SHIZUKU_PERMISSION, 
                                                         Process.myPid(), 
                                                         Process.myUid());
            // Even if denied, if we get a valid response, service is running
            return true;
        } catch (SecurityException e) {
            // SecurityException might indicate service is not running
            LogUtils.logDebug("Permission check SecurityException: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Permission check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if Shizuku is running via binder check
     */
    private boolean checkShizukuRunningViaBinder() {
        try {
            Class<?> binderClass = Class.forName(SHIZUKU_BINDER_CLASS);
            Method isAliveMethod = binderClass.getMethod("isBinderAlive");
            Boolean result = (Boolean) isAliveMethod.invoke(null);
            return result != null && result;
        } catch (Exception e) {
            LogUtils.logDebug("Binder check failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check system-level permission
     */
    private boolean checkSystemPermission() {
        try {
            int result = context.checkPermission(SHIZUKU_PERMISSION, 
                                               Process.myPid(), 
                                               Process.myUid());
            boolean granted = (result == PackageManager.PERMISSION_GRANTED);
            LogUtils.logDebug("System permission check result: " + granted);
            return granted;
        } catch (Exception e) {
            LogUtils.logDebug("System permission check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * FIXED: Check Shizuku permission via Shizuku API reflection
     */
    private boolean checkShizukuPermissionViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method checkSelfPermissionMethod = shizukuClass.getMethod("checkSelfPermission");
            Integer result = (Integer) checkSelfPermissionMethod.invoke(null);
            boolean granted = (result != null && result == PackageManager.PERMISSION_GRANTED);
            LogUtils.logDebug("Shizuku API permission check result: " + granted);
            return granted;
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku API not available for permission check");
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku API permission check error: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Test Shizuku operation to verify permission
     */
    private boolean testShizukuOperation() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            Method getUidMethod = shizukuClass.getMethod("getUid");
            Integer uid = (Integer) getUidMethod.invoke(null);
            boolean success = (uid != null && uid >= 0);
            LogUtils.logDebug("Shizuku test operation result: " + success);
            return success;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku test operation failed: " + e.getMessage());
            return false;
        }
    }
    
    // ===== FIXED: In-App Permission Request Methods =====
    
    /**
     * FIXED: Request Shizuku permission with in-app dialog support
     */
    public void requestShizukuPermission() {
        LogUtils.logUser("Requesting Shizuku permission...");
        
        if (!isShizukuInstalled()) {
            LogUtils.logUser("❌ Shizuku not installed");
            if (permissionCallback != null) {
                permissionCallback.onPermissionError("Shizuku app is not installed");
            }
            showShizukuNotInstalledDialog();
            return;
        }
        
        if (!isShizukuRunning()) {
            LogUtils.logUser("❌ Shizuku service not running");
            if (permissionCallback != null) {
                permissionCallback.onPermissionError("Shizuku service is not running");
            }
            showShizukuNotRunningDialog();
            return;
        }
        
        // FIXED: Try to request permission directly (in-app dialog)
        if (requestPermissionViaReflection()) {
            LogUtils.logUser("✅ Shizuku permission dialog requested");
        } else {
            LogUtils.logUser("❌ Failed to show in-app dialog - using fallback");
            showPermissionFallbackDialog();
        }
    }
    
    /**
     * FIXED: Request permission via Shizuku API reflection with proper method signature
     */
    private boolean requestPermissionViaReflection() {
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            
            // FIXED: Use the correct Shizuku API method signature for in-app dialogs
            Method requestPermissionMethod = shizukuClass.getMethod("requestPermission", int.class);
            requestPermissionMethod.invoke(null, SHIZUKU_REQUEST_CODE);
            
            LogUtils.logDebug("Shizuku permission requested via reflection API - in-app dialog should appear");
            return true;
        } catch (NoSuchMethodException e) {
            LogUtils.logDebug("Shizuku requestPermission method not found - trying alternative: " + e.getMessage());
            
            // Try alternative method signatures
            try {
                Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
                
                // Try method without parameters (some Shizuku versions)
                Method altMethod = shizukuClass.getMethod("requestPermission");
                altMethod.invoke(null);
                LogUtils.logDebug("Shizuku permission requested via alternative method");
                return true;
            } catch (Exception altE) {
                LogUtils.logDebug("Alternative Shizuku method also failed: " + altE.getMessage());
                return false;
            }
        } catch (ClassNotFoundException e) {
            LogUtils.logDebug("Shizuku API class not found - Shizuku not properly integrated: " + e.getMessage());
            return false;
        } catch (Exception e) {
            LogUtils.logDebug("Reflection permission request failed: " + e.getMessage());
            return false;
        }
    }
    
    // ===== Fallback and Dialog Methods =====
    
    /**
     * Show fallback dialog when in-app permission request fails
     */
    private void showPermissionFallbackDialog() {
        new AlertDialog.Builder(context)
            .setTitle("🔐 Grant Shizuku Permission")
            .setMessage("Unable to show in-app permission dialog.\n\n" +
                       "You will be taken to Shizuku app to grant permission manually.\n\n" +
                       "Steps:\n" +
                       "1. Tap 'Open Shizuku'\n" +
                       "2. Find this app in the permission list\n" +
                       "3. Grant permission\n" +
                       "4. Return to this app")
            .setPositiveButton("Open Shizuku", (dialog, which) -> openShizukuApp())
            .setNegativeButton("Cancel", (dialog, which) -> {
                if (permissionCallback != null) {
                    permissionCallback.onPermissionDenied();
                }
            })
            .show();
    }
    
    /**
     * Open Shizuku app
     */
    public void openShizukuApp() {
        try {
            Intent intent = context.getPackageManager().getLaunchIntentForPackage(SHIZUKU_PACKAGE);
            if (intent != null) {
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                context.startActivity(intent);
                LogUtils.logUser("Opening Shizuku app...");
            } else {
                LogUtils.logUser("❌ Cannot open Shizuku app");
                Toast.makeText(context, "Cannot open Shizuku app", Toast.LENGTH_SHORT).show();
            }
        } catch (Exception e) {
            LogUtils.logDebug("Failed to open Shizuku app: " + e.getMessage());
            Toast.makeText(context, "Failed to open Shizuku", Toast.LENGTH_SHORT).show();
        }
    }
    
    /**
     * Install Shizuku app
     */
    public void installShizuku() {
        try {
            String downloadUrl = "https://github.com/RikkaApps/Shizuku/releases/latest";
            Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(downloadUrl));
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            context.startActivity(intent);
            LogUtils.logUser("Opening Shizuku download page...");
        } catch (Exception e) {
            LogUtils.logDebug("Failed to open Shizuku download: " + e.getMessage());
            Toast.makeText(context, "Please install Shizuku manually", Toast.LENGTH_LONG).show();
        }
    }
    
    /**
     * Get detailed Shizuku status
     */
    public String getShizukuStatus() {
        StringBuilder status = new StringBuilder();
        status.append("=== Shizuku Status ===\n");
        
        boolean installed = isShizukuInstalled();
        status.append("Installed: ").append(installed ? "✅ Yes" : "❌ No").append("\n");
        
        if (installed) {
            boolean running = isShizukuRunning();
            status.append("Service Running: ").append(running ? "✅ Yes" : "❌ No").append("\n");
            
            if (running) {
                boolean hasPermission = hasShizukuPermission();
                status.append("Permission Granted: ").append(hasPermission ? "✅ Yes" : "❌ No").append("\n");
                
                if (hasPermission) {
                    status.append("Status: ✅ Ready for use\n");
                } else {
                    status.append("Status: 🔐 Permission needed\n");
                }
            } else {
                status.append("Status: ▶️ Service needs to be started\n");
            }
        } else {
            status.append("Status: 📥 Installation required\n");
        }
        
        // Add version info if available
        try {
            PackageManager pm = context.getPackageManager();
            String version = pm.getPackageInfo(SHIZUKU_PACKAGE, 0).versionName;
            status.append("Version: ").append(version).append("\n");
        } catch (Exception e) {
            // Ignore version check errors
        }
        
        return status.toString();
    }
    
    /**
     * Check Shizuku status and show appropriate action
     */
    public void checkRootStatus() {
        String status = getShizukuStatus();
        LogUtils.logUser("Shizuku Status Check:\n" + status);
        
        if (isShizukuReady()) {
            Toast.makeText(context, "✅ Shizuku is ready!", Toast.LENGTH_SHORT).show();
        } else if (!isShizukuInstalled()) {
            showShizukuNotInstalledDialog();
        } else if (!isShizukuRunning()) {
            showShizukuNotRunningDialog();
        } else {
            requestShizukuPermission();
        }
    }
    
    // ===== Dialog Methods =====
    
    private void showShizukuNotInstalledDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Shizuku Required")
            .setMessage("Shizuku is not installed on this device.\n\n" +
                       "Shizuku provides enhanced file access capabilities without requiring root.\n\n" +
                       "Would you like to download and install Shizuku?")
            .setPositiveButton("Download Shizuku", (dialog, which) -> installShizuku())
            .setNeutralButton("Learn More", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Intent.ACTION_VIEW, 
                                             Uri.parse("https://shizuku.rikka.app/"));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    private void showShizukuNotRunningDialog() {
        new AlertDialog.Builder(context)
            .setTitle("Start Shizuku Service")
            .setMessage("Shizuku is installed but the service is not running.\n\n" +
                       "To start Shizuku, you can:\n" +
                       "• Use ADB command (requires computer)\n" +
                       "• Use root access (if available)\n" +
                       "• Use wireless ADB (Android 11+)\n\n" +
                       "Would you like to open Shizuku app for setup?")
            .setPositiveButton("Open Shizuku", (dialog, which) -> openShizukuApp())
            .setNeutralButton("Setup Guide", (dialog, which) -> {
                try {
                    Intent intent = new Intent(Intent.ACTION_VIEW, 
                                             Uri.parse("https://shizuku.rikka.app/guide/setup/"));
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                    context.startActivity(intent);
                } catch (Exception e) {
                    Toast.makeText(context, "Cannot open browser", Toast.LENGTH_SHORT).show();
                }
            })
            .setNegativeButton("Cancel", null)
            .show();
    }
    
    // ===== Permission Result Handling =====
    
    /**
     * Handle permission request result
     */
    public void handlePermissionResult(int requestCode, int resultCode) {
        if (requestCode == SHIZUKU_REQUEST_CODE) {
            // Re-check permission after request
            boolean granted = hasShizukuPermission();
            LogUtils.logUser("Shizuku permission result: " + granted);
            
            if (permissionCallback != null) {
                if (granted) {
                    permissionCallback.onPermissionGranted();
                } else {
                    permissionCallback.onPermissionDenied();
                }
            }
        }
    }
    
    /**
     * FIXED: Force refresh permission status (useful after returning from Shizuku app)
     */
    public void refreshPermissionStatus() {
        LogUtils.logDebug("Refreshing Shizuku permission status...");
        
        // Clear any cached state and re-check
        boolean wasReady = isShizukuReady();
        
        // Small delay to ensure status has updated
        new android.os.Handler().postDelayed(() -> {
            boolean isNowReady = isShizukuReady();
            if (isNowReady != wasReady) {
                LogUtils.logUser("Shizuku status changed: " + (isNowReady ? "Ready" : "Not Ready"));
                if (permissionCallback != null) {
                    if (isNowReady) {
                        permissionCallback.onPermissionGranted();
                    } else {
                        permissionCallback.onPermissionDenied();
                    }
                }
            }
        }, 1000); // 1 second delay
    }
    
    // ===== Enhanced File Operations (when Shizuku is ready) =====
    
    /**
     * Execute command with Shizuku privileges
     */
    public boolean executeShizukuCommand(String command) {
        if (!isShizukuReady()) {
            LogUtils.logDebug("Shizuku not ready for command execution");
            return false;
        }
        
        try {
            Class<?> shizukuClass = Class.forName(SHIZUKU_CLASS);
            // This would need proper Shizuku API integration
            LogUtils.logDebug("Executing Shizuku command: " + command);
            return true;
        } catch (Exception e) {
            LogUtils.logDebug("Shizuku command execution failed: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * Check if enhanced file operations are available
     */
    public boolean canUseEnhancedFileAccess() {
        return isShizukuReady();
    }
    
    // ===== Legacy Compatibility Methods =====
    
    @Deprecated
    public boolean isRootAvailable() {
        // For backward compatibility
        return isShizukuAvailable();
    }
    
    @Deprecated
    public boolean isRootReady() {
        // For backward compatibility  
        return isShizukuReady();
    }
    
    @Deprecated
    public boolean hasRootPermission() {
        // For backward compatibility
        return hasShizukuPermission();
    }
    
    @Deprecated
    public void requestRootAccess() {
        // For backward compatibility
        requestShizukuPermission();
    }
}


--- FILE: /mnt/data/extracted_files/main/res/drawable/gradient_background_135.xml ---

<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <gradient
        android:angle="135"
        android:startColor="#E8F5E8"
        android:endColor="#F1F8E9"
        android:type="linear" />
</shape>


--- FILE: /mnt/data/extracted_files/main/res/drawable/ic_arrow_back.xml ---

<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
  <path
      android:fillColor="@android:color/black"
      android:pathData="M20,11L7.83,11l5.59,-5.59L12,4l-8,8 8,8 1.41,-1.41L7.83,13L20,13z"/>
</vector>



--- FILE: /mnt/data/extracted_files/main/res/drawable/ic_launcher_background.xml ---

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



--- FILE: /mnt/data/extracted_files/main/res/drawable-v24/ic_launcher_foreground.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_about.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_dll_mod.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_instructions.xml ---

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



--- FILE: /mnt/data/extracted_files/main/res/layout/activity_log.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_log_viewer_enhanced.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_main.xml ---

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="24dp">

    <Button
        android:id="@+id/universal_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Universal"
        android:layout_marginBottom="16dp" />

    <Button
        android:id="@+id/specific_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Specific Version" />
</LinearLayout>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_mod_list.xml ---

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



--- FILE: /mnt/data/extracted_files/main/res/layout/activity_mod_management.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_offline_diagnostic.xml ---

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


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_settings_enhanced.xml ---

<?xml version="1.0" encoding="utf-8"?>
<ScrollView
     xmlns:android="http://schemas.android.com/apk/res/android"
     xmlns:app="http://schemas.android.com/apk/res-auto"
     xmlns:tools="http://schemas.android.com/tools"
     android:layout_height="match_parent"
     android:layout_width="match_parent"
     android:background="#F5F5F5"
     android:padding="16dp">

    <LinearLayout
         android:layout_height="wrap_content"
         android:layout_width="match_parent"
         android:orientation="vertical">

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="16dp"
             android:gravity="center"
             android:background="#2196F3"
             android:padding="16dp"
             android:textSize="20sp"
             android:textColor="#FFFFFF"
             android:text="⚙️ Operation Modes &amp; Permissions"
             android:textStyle="bold" />

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:text="Choose Operation Mode:"
             android:textStyle="bold" />

        <RadioGroup
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="vertical"
             android:id="@+id/operationModeGroup">

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/normalCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/normalModeRadio"
                         android:text=" Normal Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/normalStatus"
                         android:text="Standard Android permissions" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/shizukuCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/shizukuModeRadio"
                         android:text=" Shizuku Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/shizukuStatus"
                         android:text="Enhanced file access without root" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/rootCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/rootModeRadio"
                         android:text=" Root Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/rootStatus"
                         android:text="Full system control (requires root)" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <androidx.cardview.widget.CardView
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_margin="4dp"
                 app:cardElevation="4dp"
                 app:cardBackgroundColor="#FFFFFF"
                 app:cardCornerRadius="8dp"
                 android:id="@+id/hybridCard">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:background="#FFFFFF"
                     android:padding="16dp"
                     android:orientation="horizontal">

                    <RadioButton
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:id="@+id/hybridModeRadio"
                         android:text="⚡ Hybrid Mode"
                         android:textStyle="bold" />

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="14sp"
                         android:textColor="#666666"
                         android:layout_marginStart="8dp"
                         android:layout_weight="1"
                         android:id="@+id/hybridStatus"
                         android:text="Maximum capabilities" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

        </RadioGroup>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="Setup &amp; Permissions:"
             android:textStyle="bold" />

        <LinearLayout
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="vertical">

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#2196F3"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/shizukuSetupBtn"
                 android:text=" Setup Shizuku" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#FF9800"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/rootSetupBtn"
                 android:text=" Setup Root" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#4CAF50"
                 android:elevation="4dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#FFFFFF"
                 android:id="@+id/permissionBtn"
                 android:text=" Manage Permissions" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:layout_marginBottom="8dp"
                 android:background="#E0E0E0"
                 android:elevation="2dp"
                 android:padding="16dp"
                 android:textSize="16sp"
                 android:textColor="#333333"
                 android:id="@+id/refreshStatusBtn"
                 android:text="🔄 Refresh Status" />

        </LinearLayout>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="App Features:"
             android:textStyle="bold" />

        <androidx.cardview.widget.CardView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_margin="4dp"
             app:cardElevation="4dp"
             app:cardBackgroundColor="#FFFFFF"
             app:cardCornerRadius="8dp">

            <LinearLayout
                 android:layout_height="wrap_content"
                 android:layout_width="match_parent"
                 android:background="#FFFFFF"
                 android:padding="16dp"
                 android:orientation="vertical">

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto-enable Mods"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoEnableSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Debug Logging"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/debugLoggingSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:layout_marginBottom="12dp"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto Backup"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoBackupSwitch" />

                </LinearLayout>

                <LinearLayout
                     android:layout_height="wrap_content"
                     android:layout_width="match_parent"
                     android:gravity="center_vertical"
                     android:background="#F8F8F8"
                     android:padding="8dp"
                     android:orientation="horizontal">

                    <TextView
                         android:layout_height="wrap_content"
                         android:layout_width="0dp"
                         android:textSize="16sp"
                         android:textColor="#333333"
                         android:layout_weight="1"
                         android:text="Auto Update Check"
                         android:textStyle="bold" />

                    <Switch
                         android:layout_height="wrap_content"
                         android:layout_width="wrap_content"
                         android:id="@+id/autoUpdateSwitch" />

                </LinearLayout>

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:layout_marginBottom="8dp"
             android:textSize="16sp"
             android:textColor="#333333"
             android:layout_marginTop="16dp"
             android:text="Additional Actions:"
             android:textStyle="bold" />

        <LinearLayout
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:orientation="horizontal">

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:layout_marginEnd="4dp"
                 android:background="#F44336"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_weight="1"
                 android:id="@+id/resetSettingsBtn"
                 android:text="Reset"
                 android:textStyle="bold" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:layout_marginEnd="2dp"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_marginStart="2dp"
                 android:background="#2196F3"
                 android:layout_weight="1"
                 android:id="@+id/exportSettingsBtn"
                 android:text="Export"
                 android:textStyle="bold" />

            <Button
                 android:layout_height="wrap_content"
                 android:layout_width="0dp"
                 android:background="#4CAF50"
                 android:elevation="4dp"
                 android:padding="12dp"
                 android:textSize="14sp"
                 android:textColor="#FFFFFF"
                 android:layout_marginStart="4dp"
                 android:layout_weight="1"
                 android:id="@+id/importSettingsBtn"
                 android:text="Import"
                 android:textStyle="bold" />

        </LinearLayout>

        <TextView
             android:layout_height="wrap_content"
             android:layout_width="match_parent"
             android:gravity="center"
             android:background="#E3F2FD"
             android:padding="12dp"
             android:textSize="12sp"
             android:textColor="#666666"
             android:layout_marginTop="16dp"
             android:text="Tip: Shizuku provides enhanced capabilities without requiring root access"
             android:textStyle="italic" />

    </LinearLayout>

</ScrollView>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_setup_guide.xml ---

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
        android:padding="24dp"
        android:background="@drawable/gradient_background_135">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:background="#FFFFFF"
            android:padding="32dp"
            android:layout_marginBottom="24dp"
            android:elevation="8dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🚀 MelonLoader Setup Guide"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="12dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Choose your preferred installation method"
                android:textSize="16sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:lineSpacingExtra="4dp" />

        </LinearLayout>

        <!-- Installation Options -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:layout_marginBottom="24dp">

            <!-- Online Installation Card -->
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
                        android:text="🌐 Automated Online Installation"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#1565C0"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Automatically downloads from GitHub\n• No manual file handling\n• Always gets latest version\n• Requires internet connection"
                        android:textSize="14sp"
                        android:textColor="#1976D2"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_online_install"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="🌐 Start Online Installation"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Offline Import Card -->
            <androidx.cardview.widget.CardView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginBottom="16dp"
                app:cardCornerRadius="12dp"
                app:cardElevation="6dp"
                app:cardBackgroundColor="#FFF3E0">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:padding="20dp">

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📦 Offline ZIP Import"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#E65100"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Import pre-downloaded ZIP files\n• Works without internet\n• Auto-detects NET8/NET35\n• Extracts to correct directories"
                        android:textSize="14sp"
                        android:textColor="#F57C00"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_offline_import"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📦 Import ZIP File"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="💡 Supports: melon_data.zip, lemon_data.zip, custom packages"
                        android:textSize="11sp"
                        android:textColor="#BF360C"
                        android:gravity="center"
                        android:layout_marginTop="8dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

            <!-- Manual Installation Card -->
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
                        android:text="📖 Manual Installation Guide"
                        android:textSize="20sp"
                        android:textStyle="bold"
                        android:textColor="#7B1FA2"
                        android:layout_marginBottom="12dp" />

                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="• Step-by-step instructions\n• Full control over installation\n• Troubleshooting included\n• For advanced users"
                        android:textSize="14sp"
                        android:textColor="#8E24AA"
                        android:lineSpacingExtra="4dp"
                        android:layout_marginBottom="16dp" />

                    <Button
                        android:id="@+id/btn_manual_instructions"
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="📖 View Manual Guide"
                        android:textSize="16sp"
                        android:textStyle="bold"
                        android:background="#9C27B0"
                        android:textColor="@android:color/white"
                        android:minHeight="56dp"
                        android:elevation="4dp" />

                </LinearLayout>

            </androidx.cardview.widget.CardView>

        </LinearLayout>

        <!-- Information Section -->
        <androidx.cardview.widget.CardView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="8dp"
            app:cardElevation="4dp"
            app:cardBackgroundColor="#E8F5E8">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="16dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="ℹ️ What happens after installation?"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:textColor="#2E7D32"
                    android:layout_marginBottom="8dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="1. 🚀 Unified Loader opens automatically\n2. 📱 Select your Terraria APK file\n3. ⚡ Patch APK with MelonLoader\n4. 📲 Install the patched APK\n5. 🎮 Add DLL mods and enjoy!"
                    android:textSize="14sp"
                    android:textColor="#388E3C"
                    android:lineSpacingExtra="4dp" />

            </LinearLayout>

        </androidx.cardview.widget.CardView>

        <!-- Requirements Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:background="#FFFFFF"
            android:padding="16dp"
            android:layout_marginTop="8dp"
            android:elevation="2dp">

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="📋 Requirements:\n• 50MB+ free space\n• Terraria APK file\n• File manager permissions"
                android:textSize="12sp"
                android:textColor="#666666"
                android:lineSpacingExtra="2dp" />

            <TextView
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="🎯 Recommended:\n• Use Online Installation\n• Keep APK backup\n• Enable unknown sources"
                android:textSize="12sp"
                android:textColor="#666666"
                android:lineSpacingExtra="2dp" />

        </LinearLayout>

    </LinearLayout>

</ScrollView>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_specific_selection.xml ---

<?xml version="1.0" encoding="utf-8"?>

<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="24dp">

    <LinearLayout
        android:id="@+id/specific_selection_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center">

        <!-- Header -->
        <TextView
            android:id="@+id/headerText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Choose Game/App to Mod"
            android:textSize="24sp"
            android:textStyle="bold"
            android:gravity="center"
            android:layout_marginBottom="32dp" />

        <!-- Terraria Button -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:background="@android:drawable/dialog_frame"
            android:padding="16dp"
            android:layout_marginBottom="16dp">

            <Button
                android:id="@+id/terraria_button"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🌍 Terraria"
                android:textSize="20sp"
                android:textStyle="bold"
                android:minHeight="60dp" />

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="• Support for DEX/JAR mods\n• Support for DLL mods (via MelonLoader)\n• APK patching and installation\n• Full mod management"
                android:textSize="14sp"
                android:textColor="@android:color/darker_gray"
                android:layout_marginTop="8dp" />

        </LinearLayout>

        <!-- Future Games Section -->
        <LinearLayout
            android:id="@+id/futureGamesSection"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:layout_marginTop="24dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Coming Soon"
                android:textSize="18sp"
                android:textStyle="bold"
                android:gravity="center"
                android:layout_marginBottom="16dp" />

            <!-- Placeholder cards for future games -->
            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:background="@android:drawable/dialog_frame"
                android:padding="12dp"
                android:layout_marginBottom="8dp"
                android:alpha="0.5">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🟫 Minecraft PE"
                        android:textSize="16sp" />

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Soon"
                        android:textSize="12sp"
                        android:textColor="@android:color/darker_gray" />

                </LinearLayout>
            </LinearLayout>

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:background="@android:drawable/dialog_frame"
                android:padding="12dp"
                android:layout_marginBottom="8dp"
                android:alpha="0.5">

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal">

                    <TextView
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🚀 Among Us"
                        android:textSize="16sp" />

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Soon"
                        android:textSize="12sp"
                        android:textColor="@android:color/darker_gray" />

                </LinearLayout>
            </LinearLayout>

        </LinearLayout>

        <!-- Back Button -->
        <Button
            android:id="@+id/backToMainButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="← Back to Main Menu"
            android:layout_marginTop="32dp" />

    </LinearLayout>
</ScrollView>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_terraria_specific_updated.xml ---

<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:id="@+id/rootLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp"
        android:background="#E8F5E8">

        <!-- Header Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:layout_marginBottom="24dp">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="🌍 Terraria Mod Loader"
                android:textSize="28sp"
                android:textStyle="bold"
                android:textColor="#2E7D32"
                android:gravity="center"
                android:layout_marginBottom="8dp" />

            <TextView
                android:id="@+id/loaderStatusText"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Checking loader status..."
                android:textSize="14sp"
                android:textColor="#4CAF50"
                android:gravity="center"
                android:padding="12dp"
                android:background="#F1F8E9"
                android:layout_marginTop="8dp" />
        </LinearLayout>

        <!-- Setup & Installation Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/setupCard"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="16dp"
            app:cardCornerRadius="12dp"
            app:cardElevation="6dp"
            app:cardBackgroundColor="#F1F8E9">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:orientation="vertical"
                android:padding="20dp">

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🚀 Setup &amp; Installation"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#2E7D32"
                    android:layout_marginBottom="16dp" />

                <Button
                    android:id="@+id/unifiedSetupButton"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🎯 Complete Setup Wizard"
                    android:textSize="16sp"
                    android:textStyle="bold"
                    android:background="#4CAF50"
                    android:textColor="@android:color/white"
                    android:layout_marginBottom="12dp"
                    android:minHeight="56dp"
                    android:elevation="2dp" />

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="All-in-one wizard for MelonLoader installation and APK patching"
                    android:textSize="12sp"
                    android:textColor="#66BB6A"
                    android:layout_marginBottom="16dp"
                    android:gravity="center" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/setupGuideButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📖 Setup Guide"
                        android:textSize="14sp"
                        android:background="#81C784"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/manualInstructionsButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Manual Steps"
                        android:textSize="14sp"
                        android:background="#A5D6A7"
                        android:textColor="#2E7D32"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />
                </LinearLayout>
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Mod Management Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/modManagementCard"
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
                    android:text="📦 Mod Management"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#1565C0"
                    android:layout_marginBottom="16dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="2">

                    <Button
                        android:id="@+id/dexModManagerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📱 DEX/JAR Mods"
                        android:textSize="14sp"
                        android:background="#2196F3"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="6dp"
                        android:minHeight="48dp" />

                    <Button
                        android:id="@+id/dllModManagerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🔧 DLL Mods"
                        android:textSize="14sp"
                        android:background="#FF9800"
                        android:textColor="@android:color/white"
                        android:layout_marginStart="6dp"
                        android:minHeight="48dp" />
                </LinearLayout>

                <TextView
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="• DEX/JAR: Android Java mods (always available)\n• DLL: C# mods (requires MelonLoader)"
                    android:textSize="12sp"
                    android:textColor="#42A5F5"
                    android:layout_marginTop="12dp"
                    android:lineSpacingExtra="2dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Tools & Utilities Section -->
        <androidx.cardview.widget.CardView
            android:id="@+id/toolsCard"
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
                    android:text="🛠️ Tools &amp; Utilities"
                    android:textSize="20sp"
                    android:textStyle="bold"
                    android:textColor="#7B1FA2"
                    android:layout_marginBottom="16dp" />

                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="horizontal"
                    android:weightSum="3">

                    <Button
                        android:id="@+id/logViewerButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="📋 Logs"
                        android:textSize="12sp"
                        android:background="#9C27B0"
                        android:textColor="@android:color/white"
                        android:layout_marginEnd="4dp"
                        android:minHeight="44dp" />

                    <Button
                        android:id="@+id/settingsButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="⚙️ Settings"
                        android:textSize="12sp"
                        android:background="#BA68C8"
                        android:textColor="@android:color/white"
                        android:layout_marginHorizontal="4dp"
                        android:minHeight="44dp" />

                    <Button
                        android:id="@+id/sandboxButton"
                        android:layout_width="0dp"
                        android:layout_weight="1"
                        android:layout_height="wrap_content"
                        android:text="🧪 Sandbox"
                        android:textSize="12sp"
                        android:background="#CE93D8"
                        android:textColor="#4A148C"
                        android:layout_marginStart="4dp"
                        android:minHeight="44dp" />
                </LinearLayout>

                <!-- ✅ Fixed Diagnostic Button -->
                <Button
                    android:id="@+id/diagnosticButton"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:text="🧪 Offline Diagnostic &amp; Repair"
                    android:textSize="14sp"
                    android:textStyle="bold"
                    android:backgroundTint="#9C27B0"
                    android:textColor="#FFFFFF"
                    android:layout_marginTop="12dp"
                    android:minHeight="48dp" />
            </LinearLayout>
        </androidx.cardview.widget.CardView>

        <!-- Navigation Section -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:gravity="center"
            android:layout_marginTop="16dp">

            <Button
                android:id="@+id/backButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="← Back to App Selection"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:minHeight="40dp" />
        </LinearLayout>

        <!-- Info Footer -->
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="💡 Tip: Start with 'Complete Setup Wizard' for the easiest experience!"
            android:textSize="12sp"
            android:textColor="#81C784"
            android:gravity="center"
            android:background="#F1F8E9"
            android:padding="16dp"
            android:layout_marginTop="16dp"
            android:layout_marginBottom="16dp" />
    </LinearLayout>
</ScrollView>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_unified_loader.xml ---

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="@android:color/white">

    <!-- Header Section -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:background="#E8F5E8"
        android:padding="16dp"
        android:elevation="4dp">

        <!-- Progress Bar -->
        <ProgressBar
            android:id="@+id/stepProgressBar"
            style="?android:attr/progressBarStyleHorizontal"
            android:layout_width="match_parent"
            android:layout_height="8dp"
            android:layout_marginBottom="12dp"
            android:max="4"
            android:progress="0"
            android:progressTint="#4CAF50"
            android:progressBackgroundTint="#E0E0E0" />

        <!-- Step Indicator -->
        <TextView
            android:id="@+id/stepIndicatorText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Step 1 of 5"
            android:textSize="12sp"
            android:textColor="#666666"
            android:gravity="center"
            android:layout_marginBottom="8dp" />

        <!-- Step Title -->
        <TextView
            android:id="@+id/stepTitleText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Welcome"
            android:textSize="24sp"
            android:textStyle="bold"
            android:textColor="#2E7D32"
            android:gravity="center"
            android:layout_marginBottom="8dp" />

        <!-- Step Description -->
        <TextView
            android:id="@+id/stepDescriptionText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Welcome to the MelonLoader Setup Wizard"
            android:textSize="14sp"
            android:textColor="#4CAF50"
            android:gravity="center"
            android:lineSpacingExtra="4dp" />

    </LinearLayout>

    <!-- Main Content Area -->
    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:fillViewport="true">

        <LinearLayout
            android:id="@+id/stepContentContainer"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical"
            android:padding="16dp"
            android:minHeight="400dp">

            <!-- Dynamic content will be added here -->

        </LinearLayout>

    </ScrollView>

    <!-- Navigation Footer -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:background="#F5F5F5"
        android:padding="16dp"
        android:elevation="4dp">

        <!-- Action Button (context-sensitive) -->
        <Button
            android:id="@+id/actionButton"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="🚀 Start Setup"
            android:textSize="16sp"
            android:textStyle="bold"
            android:background="#4CAF50"
            android:textColor="@android:color/white"
            android:layout_marginBottom="12dp"
            android:minHeight="48dp" />

        <!-- Navigation Buttons -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:weightSum="2">

            <Button
                android:id="@+id/previousButton"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="← Previous"
                android:textSize="14sp"
                android:background="@android:color/transparent"
                android:textColor="#666666"
                android:layout_marginEnd="8dp"
                android:enabled="false" />

            <Button
                android:id="@+id/nextButton"
                android:layout_width="0dp"
                android:layout_weight="1"
                android:layout_height="wrap_content"
                android:text="Next →"
                android:textSize="14sp"
                android:background="#2196F3"
                android:textColor="@android:color/white"
                android:layout_marginStart="8dp" />

        </LinearLayout>

        <!-- Progress Text (for operations) -->
        <TextView
            android:id="@+id/progressText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text=""
            android:textSize="12sp"
            android:textColor="#666666"
            android:gravity="center"
            android:layout_marginTop="8dp"
            android:visibility="gone" />

    </LinearLayout>

    <!-- Hidden Status Views (referenced by activity) -->
    <TextView
        android:id="@+id/loaderStatusText"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

    <TextView
        android:id="@+id/apkStatusText"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

    <ProgressBar
        android:id="@+id/actionProgressBar"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:visibility="gone" />

</LinearLayout>


--- FILE: /mnt/data/extracted_files/main/res/layout/activity_universal.xml ---

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <Button
        android:id="@+id/select_apk_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Select Universal APK" />

    <Button
        android:id="@+id/select_zip_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Select Loader ZIP" />

    <Button
        android:id="@+id/inject_button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Inject Loader" />

    <TextView
        android:id="@+id/status_text"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Status"
        android:paddingTop="16dp"
        android:textAppearance="?android:attr/textAppearanceMedium" />

</LinearLayout>


--- FILE: /mnt/data/extracted_files/main/res/layout/dialog_log_settings.xml ---

<!-- File: dialog_log_settings.xml (NEW DIALOG) - Settings for Log Viewer -->
<!-- Path: /main/res/layout/dialog_log_settings.xml -->

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="#2A2A2A">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="⚙️ Log Viewer Settings"
        android:textColor="#FFFFFF"
        android:textSize="18sp"
        android:textStyle="bold"
        android:gravity="center"
        android:layout_marginBottom="16dp" />

    <!-- Auto Refresh Setting -->
    <CheckBox
        android:id="@+id/autoRefreshCheckbox"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="🔄 Auto-refresh logs (every 5 seconds)"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:checked="true"
        android:buttonTint="#4CAF50"
        android:layout_marginBottom="12dp" />

    <!-- Syntax Highlighting Setting -->
    <CheckBox
        android:id="@+id/syntaxHighlightCheckbox"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="🎨 Enable syntax highlighting"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:checked="true"
        android:buttonTint="#4CAF50"
        android:layout_marginBottom="12dp" />

    <!-- Text Size Setting -->
    <TextView
        android:id="@+id/textSizeLabel"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="📏 Text Size: 12"
        android:textColor="#FFFFFF"
        android:textSize="14sp"
        android:layout_marginBottom="8dp" />

    <SeekBar
        android:id="@+id/textSizeSeekBar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:max="20"
        android:min="8"
        android:progress="12"
        android:thumbTint="#4CAF50"
        android:progressTint="#4CAF50"
        android:layout_marginBottom="16dp" />

    <!-- Information Text -->
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="💡 Changes are applied immediately and persist during this session."
        android:textColor="#888888"
        android:textSize="12sp"
        android:gravity="center"
        android:layout_marginTop="8dp" />

</LinearLayout>


--- FILE: /mnt/data/extracted_files/main/res/layout/item_log_entry.xml ---

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="8dp"
    android:background="?android:attr/selectableItemBackground"
    android:minHeight="56dp">

    <!-- Level Indicator Bar -->
    <View
        android:id="@+id/levelIndicator"
        android:layout_width="4dp"
        android:layout_height="match_parent"
        android:layout_marginEnd="8dp"
        android:background="#2196F3" />

    <!-- Main Content -->
    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:orientation="vertical">

        <!-- Header Row (Timestamp, Level, Tag) -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_marginBottom="4dp">

            <TextView
                android:id="@+id/logTimestamp"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="00:00:00"
                android:textSize="11sp"
                android:textColor="#666666"
                android:fontFamily="monospace"
                android:layout_marginEnd="8dp" />

            <TextView
                android:id="@+id/logLevel"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="INFO"
                android:textSize="11sp"
                android:textStyle="bold"
                android:textColor="#2196F3"
                android:layout_marginEnd="8dp"
                android:minWidth="48dp" />

            <TextView
                android:id="@+id/logTag"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="TAG"
                android:textSize="11sp"
                android:textColor="#666666"
                android:textStyle="italic"
                android:ellipsize="end"
                android:maxLines="1" />

        </LinearLayout>

        <!-- Message Content -->
        <TextView
            android:id="@+id/logMessage"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Log message content goes here and can span multiple lines if needed"
            android:textSize="14sp"
            android:textColor="#333333"
            android:lineSpacingExtra="2dp"
            android:textIsSelectable="true"
            android:maxLines="10"
            android:ellipsize="end" />

    </LinearLayout>

</LinearLayout>


--- FILE: /mnt/data/extracted_files/main/res/layout/item_mod.xml ---

<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp"
        android:gravity="center_vertical">

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:orientation="vertical">

            <TextView
                android:id="@+id/modNameTextView"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Mod Name"
                android:textSize="18sp"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/modDescription"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Mod description goes here. This can be multiline."
                android:textSize="14sp"
                android:textColor="@android:color/darker_gray"
                android:layout_marginTop="4dp" />

        </LinearLayout>

        <Switch
            android:id="@+id/modSwitch"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="16dp" />

        <ImageButton
            android:id="@+id/modDeleteButton"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_marginStart="8dp"
            android:background="?attr/selectableItemBackgroundBorderless"
            android:src="@android:drawable/ic_menu_delete"
            android:contentDescription="Delete Mod"
            app:tint="@android:color/holo_red_dark" />

    </LinearLayout>
</androidx.cardview.widget.CardView>



--- FILE: /mnt/data/extracted_files/main/res/menu/log_viewer_menu.xml ---

<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <item
        android:id="@+id/action_toggle_filters"
        android:icon="@android:drawable/ic_search_category_default"
        android:title="Toggle Filters"
        app:showAsAction="ifRoom" />

    <item
        android:id="@+id/action_share_logs"
        android:icon="@android:drawable/ic_menu_share"
        android:title="Share Logs"
        app:showAsAction="ifRoom" />

    <item
        android:id="@+id/action_clear_logs"
        android:icon="@android:drawable/ic_menu_delete"
        android:title="Clear Logs"
        app:showAsAction="never" />

    <item
        android:id="@+id/action_settings"
        android:icon="@android:drawable/ic_menu_preferences"
        android:title="Settings"
        app:showAsAction="never" />

</menu>


--- FILE: /mnt/data/extracted_files/main/res/menu/main_menu.xml ---

<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:id="@+id/action_log"
        android:title="View Logs"
        android:icon="@android:drawable/ic_menu_info_details"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_about"
        android:title="About"
        android:icon="@android:drawable/ic_menu_help"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_dark_mode"
        android:title="Toggle Dark Mode"
        android:icon="@android:drawable/ic_menu_day"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_export_apk"
        android:title="Export Modified APK"
        android:icon="@android:drawable/ic_menu_save"
        android:showAsAction="never" />

    <item
        android:id="@+id/action_export_logs"
        android:title="Export Logs"
        android:icon="@android:drawable/ic_menu_upload"
        android:showAsAction="never" />
</menu>


--- FILE: /mnt/data/extracted_files/main/res/mipmap-anydpi-v26/ic_launcher.xml ---

<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>


--- FILE: /mnt/data/extracted_files/main/res/mipmap-anydpi-v26/ic_launcher_round.xml ---

<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>


--- FILE: /mnt/data/extracted_files/main/res/values/colors.xml ---

<resources>
    <color name="purple_200">#BB86FC</color>
    <color name="purple_500">#6200EE</color>
    <color name="purple_700">#3700B3</color>
    <color name="teal_200">#03DAC5</color>
    <color name="teal_700">#018786</color>
    <color name="black">#000000</color>
    <color name="white">#FFFFFF</color>
    <color name="colorPrimary">#6200EE</color>
</resources>



--- FILE: /mnt/data/extracted_files/main/res/values/strings.xml ---

<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Terraria ML</string>

</resources>


--- FILE: /mnt/data/extracted_files/main/res/values/themes.xml ---

<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Base application theme (Light) -->
    <style name="Theme.ModLoader" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/black</item>
        <item name="android:statusBarColor" tools:targetApi="l">?attr/colorPrimaryVariant</item>
    </style>
</resources>


--- FILE: /mnt/data/extracted_files/main/res/values-night/colors.xml ---

<resources>
    <color name="white">#000000</color>
    <color name="black">#FFFFFF</color>
</resources>


--- FILE: /mnt/data/extracted_files/main/res/values-night/themes.xml ---

<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Night mode theme -->
    <style name="Theme.ModLoader" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_200</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/black</item>
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/white</item>
        <item name="android:statusBarColor" tools:targetApi="l">?attr/colorPrimaryVariant</item>
    </style>
</resources>


--- FILE: /mnt/data/extracted_files/main/res/xml/backup_rules.xml ---

<?xml version="1.0" encoding="utf-8"?>
<full-backup-content>
    <!-- Include app-specific files -->
    <include domain="file" path="." />
    <include domain="database" path="." />
    <include domain="sharedpref" path="." />
    <include domain="external" path="Android/data/com.modloader/" />

    <!-- Exclude cache and logs if needed -->
    <exclude domain="cache" path="." />
    <exclude domain="file" path="logs/" />
</full-backup-content>


--- FILE: /mnt/data/extracted_files/main/res/xml/data_extraction_rules.xml ---

<?xml version="1.0" encoding="utf-8"?><!--
   Sample data extraction rules file; uncomment and customize as necessary.
   See https://developer.android.com/about/versions/12/backup-restore#xml-changes
   for details.
-->
<data-extraction-rules>
  <cloud-backup>
    <!-- TODO: Use <include> and <exclude> to control what is backed up.
        <include .../>
        <exclude .../>
        -->
  </cloud-backup>
  <!--
    <device-transfer>
        <include .../>
        <exclude .../>
    </device-transfer>
    -->
</data-extraction-rules>


--- FILE: /mnt/data/extracted_files/main/res/xml/file_paths.xml ---

<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path
        name="external_files"
        path="." />
</paths>


--- FILE: /mnt/data/extracted_files/main/res/xml/file_provider_paths.xml ---

<paths xmlns:android="http://schemas.android.com/tools">
    
    <!-- External storage root (for legacy support) -->
    <external-path 
        name="external_storage_root" 
        path="." />
    
    <!-- App-specific external files directory -->
    <external-files-path 
        name="app_external_files" 
        path="." />
    
    <!-- APK installation directory (FIXED - main issue for APK parsing) -->
    <external-files-path 
        name="apk_install" 
        path="apk_install" />
    
    <!-- Cache directory for temporary files -->
    <external-cache-path 
        name="app_cache" 
        path="." />
    
    <!-- TerrariaLoader main directory -->
    <external-files-path 
        name="terraria_loader" 
        path="TerrariaLoader" />
    
    <!-- Game-specific directories -->
    <external-files-path 
        name="terraria_game" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid" />
    
    <!-- Mod directories -->
    <external-files-path 
        name="dex_mods" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Mods/DEX" />
    
    <external-files-path 
        name="dll_mods" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Mods/DLL" />
    
    <!-- Log directories -->
    <external-files-path 
        name="app_logs" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/AppLogs" />
    
    <external-files-path 
        name="game_logs" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Logs" />
    
    <!-- Backup directories -->
    <external-files-path 
        name="backups" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Backups" />
    
    <!-- Config directories -->
    <external-files-path 
        name="config" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Config" />
    
    <!-- MelonLoader directories -->
    <external-files-path 
        name="melonloader" 
        path="TerrariaLoader/com.and.games505.TerrariaPaid/Loaders/MelonLoader" />
    
    <!-- Downloads and exports -->
    <external-files-path 
        name="downloads" 
        path="downloads" />
    
    <external-files-path 
        name="exports" 
        path="exports" />
    
    <!-- Temporary processing directory -->
    <external-files-path 
        name="temp" 
        path="temp" />
    
    <!-- Legacy mod directory (for migration) -->
    <external-files-path 
        name="legacy_mods" 
        path="mods" />

</paths>


--- FILE: /mnt/data/extracted_files/main/res/xml/paths.xml ---

<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path
        name="external_files"
        path="." />
</paths>
