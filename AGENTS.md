# AGENTS.md

Menu-bar-only macOS app (Swift 5, deployment target 10.13) that locks/unlocks the Mac by BLE device proximity. Two targets in `BLEUnlock.xcodeproj`: `BLEUnlock` (Swift) and `Launcher` (ObjC login item). User-facing docs live in `README.md` / `README.ja.md` — don't duplicate.

## Build & verify

- No tests, no SPM/CocoaPods, no linter. CI (`.github/workflows/test.yml`) only runs `xcodebuild clean build -project BLEUnlock.xcodeproj -scheme BLEUnlock`. Compiling = green. Use that command locally to verify.
- Every build runs a shell phase that increments `CFBundleVersion` in `BLEUnlock/Info.plist` via PlistBuddy. Any build dirties that file; don't hand-edit the build number.
- Debug signing is ad-hoc (`CODE_SIGN_IDENTITY = "-"`); Release is `Developer ID Application` (manual style) and fails without the Developer ID cert.
- Release flow: `PASSWORD=<app-specific Apple ID password> ./release`. Script archives, exports (`ExportOptions.plist`), notarizes + staples both the app and the embedded Launcher via `xcrun notarytool`, zips, copies archive to `archives/<version>-<build>/`. Bump `CFBundleShortVersionString` in `BLEUnlock/Info.plist` for new releases.

## Architecture

- `AppDelegate.swift` — menu bar UI and lock/unlock orchestration: login password in Keychain, typed into the login screen via `CGEvent` keystrokes (requires Accessibility permission); observes display/system sleep-wake and `com.apple.screenIsUnlocked` notifications.
- `BLE.swift` — all CoreBluetooth: scanning, RSSI smoothing (vDSP mean over last 5 samples), active mode (connect + `readRSSI` every 2s) vs passive mode, proximity/signal timers. Device name/MAC resolution chain is in `Device.description`: Monterey SQLite DBs at `/Library/Bluetooth/*.ledevices*.db` (`LEDeviceInfo.swift`, string-interpolated SQL — keep the interpolation when editing) → `/Library/Preferences/com.apple.Bluetooth.plist` → Device Information service (0x180A) → iBeacon parse → UUID fallback.
- `lowlevel.c` + `BLEUnlock-Bridging-Header.h` — private APIs. `SACLockScreenImmediate()` from private `login.framework`; Now Playing pause/resume from private `MediaRemote.framework` via hand-written stub `MediaRemote.h` (only 2 functions — extend the stub to add more). Both linked by hard-coded relative paths into `System/Library/PrivateFrameworks` in the pbxproj.
- `Launcher/` — ObjC helper that relaunches the main app; embedded in `Contents/Library/LoginItems/` via CopyFiles phase; toggled with `SMLoginItemSetEnabled`.
- `Info.plist` deliberately has NO `LSUIElement` — code comment says it breaks `CBCentralManager` scanning. Dock icon is hidden at runtime via `NSApp.setActivationPolicy(.accessory)`, switched to `.regular` on system sleep so Bluetooth can re-scan.
- Not sandboxed; entitlements only allow Apple Events. Runtime needs Bluetooth + Accessibility + Keychain ("Always Allow") permissions.
- `checkUpdate.swift` — GitHub Releases API, 24h throttle, no auth.

## Conventions

- UI strings via `t()` = `NSLocalizedString`. New strings go into every `.lproj` (`Base`, `ja`, `zh-Hans`, `da`, `de`, `nb`, `sv`, `tr`) `Localizable.strings`.
- Preferences are plain `UserDefaults.standard` string keys (`device`, `lockRSSI`, `unlockRSSI`, `timeout`, `lockDelay`, `passiveMode`, `thresholdRSSI`, `wakeOnProximity`, ...) set in AppDelegate.
- Event hook `~/Library/Application Scripts/jp.sone.BLEUnlock/event` is called with arg `away|lost|unlocked|intruded` (+ RSSI) — don't break that contract.
- `apple-device-names` is a Python script regenerating the `appleDeviceNames` Swift dict from `/System/Library/CoreServices/CoreTypes.bundle/.../MobileDevices*.bundle` plists — run it, don't hand-edit the dict.
- Deployment target is 10.13: guard or avoid newer-only APIs.
