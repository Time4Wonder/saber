# Saber Modifications Summary

> [!IMPORTANT]
> **Disclaimer / Info:**
> These changes were quickly "vibe-coded" on Ubuntu to fix specific bugs in this workspace. There is no official guarantee or warranty of any kind.

### 🔑 Manual Refresh Shortcuts
* **Shortcuts**: `Ctrl+R` or `F5` (inside the note editor)
* **User Perspective**:
  When you are editing a note, pressing `Ctrl+R` or `F5` will automatically:
  1. Save your current local drawing/progress to disk.
  2. Perform a live query to Nextcloud bypassing local caches to check if there is a newer version of the note on the server.
  3. If a newer version is found on Nextcloud, it automatically downloads the file and instantly reloads the editor screen to show the new content.

This document lists the modifications made to the Saber codebase to fix encryption issues, manual refresh bugs, and real-time collaboration/synchronization.

---

## 1. Startup Encryption/Decryption Sync Fix
* **File affected**: [lib/main.dart](file:///home/t4w/development/saber/lib/main.dart)
* **Problem**: 
  When the application launched, the background sync interface started running immediately, attempting to download and decrypt sync files from the Nextcloud server. However, the decryption key, IV, and password were read asynchronously from secure storage. This created a race condition where files were decrypted using uninitialized (null/empty) keys, throwing exceptions like:
  `WARNING: SaberSyncInterface: Failed to get sync file from remote file: Invalid argument(s): Invalid or corrupted pad block`
* **Fix**:
  We updated `startSyncAfterLoaded()` to await the loading of all critical credentials from secure storage before initializing any sync tasks or registering listeners:
  ```dart
  await Future.wait([
    stows.username.waitUntilRead(),
    stows.ncPassword.waitUntilRead(),
    stows.encPassword.waitUntilRead(),
    stows.key.waitUntilRead(),
    stows.iv.waitUntilRead(),
  ]);
  ```

---

## 2. Editor Manual Refresh (Ctrl+R / F5) Cache Bypass
* **Files affected**: 
  - [lib/data/nextcloud/saber_syncer.dart](file:///home/t4w/development/saber/lib/data/nextcloud/saber_syncer.dart)
  - [lib/pages/editor/editor.dart](file:///home/t4w/development/saber/lib/pages/editor/editor.dart)
* **Problem**: 
  Pressing `Ctrl+R` or `F5` in the editor did not pull down new modifications from the server. The check for updates relied on cached remote file metadata. The cache was only updated when returning to the home screen and syncing, so manual refreshes in the editor compared the local file against stale remote metadata and concluded the file was already up to date.
* **Fix**:
  - We added an optional `useCache` parameter to `getSyncFileFromLocalFile` and `SaberSyncFile.relative` (defaulting to `true`).
  - In `_refreshCurrentNote()`, we pass `useCache: !isManual`. When the refresh is manual, it bypasses the cache and queries Nextcloud directly to check if a newer version exists.

---

## 3. "Watch Server" Live Polling Cache Bypass and Interval reduction
* **File affected**: [lib/pages/editor/editor.dart](file:///home/t4w/development/saber/lib/pages/editor/editor.dart)
* **Problem**: 
  The built-in "Watch Server" (Server beobachten) feature is designed to poll for remote updates in the background. However, it also relied on the cached file list. This meant that even when enabled, the periodic check never queried Nextcloud directly and failed to download live updates.
* **Fix**:
  - We modified `_refreshCurrentNote` to bypass the cache when watching the server:
    ```dart
    useCache: !isManual && !coreInfo.readOnlyBecauseWatchingServer
    ```
  - We changed the polling interval of `_watchServerTimer` from **5 seconds** to **3 seconds** for faster live updates:
    ```dart
    _watchServerTimer ??= Timer.periodic(
      const Duration(seconds: 3),
      (_) => _refreshCurrentNote(),
    );
    ```

---

## 4. Test Compilation Fix (Linux/Windows Icon Casting)
* **File affected**: [test/glassy_container_test.dart](file:///home/t4w/development/saber/test/glassy_container_test.dart)
* **Problem**: 
  The unit/widget tests failed to compile on Linux due to type inference failure when specifying platform icons (`FontAwesomeIcons.linux` and `FontAwesomeIcons.windows`).
* **Fix**:
  We explicitly cast the FontAwesome icons to `IconData` in the switch cases:
  ```dart
  case TargetPlatform.linux:
    icon = FontAwesomeIcons.linux as IconData;
  ```

---

## 📥 Installation and Execution on Ubuntu/Linux

> [!NOTE]
> **No Flutter/Dart Required:** The release builds (both local release bundle and the AppImage from GitHub) are fully compiled native binaries. You **do not** need to have Flutter or Dart installed on the machine where you run them.

Since you are running Ubuntu/Linux, you have two ways to run/install this modified version:

### Option A: Use your locally built version (Recommended for testing immediately)
Since you successfully ran `flutter build linux --release` locally, the compiled binary is sitting in your build folder.

1. **Test run it immediately:**
   ```bash
   ./build/linux/x64/release/bundle/saber
   ```

2. **Install it globally on your system:**
   To make it launchable from any terminal session:
   ```bash
   # Create a system directory for the app
   sudo mkdir -p /opt/saber
   
   # Copy the built release files there
   sudo cp -r build/linux/x64/release/bundle/* /opt/saber/
   
   # Create a symlink to run it via the 'saber' command
   sudo ln -sf /opt/saber/saber /usr/local/bin/saber
   ```
   Now you can just type `saber` in any terminal to launch your modded version.

---

### Option B: Download the built AppImage from GitHub Actions
After you push a release tag (like `v1.29.3`), GitHub Actions will build an AppImage automatically:

1. Go to your repository's **Releases** page on GitHub.
2. Download the `Saber-*-x86_64.AppImage` file.
3. Make it executable:
   ```bash
   chmod +x Saber-*.AppImage
   ```
4. Run it:
   ```bash
   ./Saber-*.AppImage
   ```
