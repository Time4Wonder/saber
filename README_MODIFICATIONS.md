# Saber Modifications Summary

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
