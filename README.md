# Privacy Clipboard Sentinel

Android privacy audit tool that monitors system log events related to clipboard access attempts.

## Status

Portfolio prototype. The app is designed for local privacy auditing on a personal device and requires ADB-granted log access. The in-app UI is in Brazilian Portuguese; this README is in English.

## Overview

Privacy Clipboard Sentinel reads Android system log lines (`logcat`) that match known clipboard-related patterns, attributes each match to a package name, and records the events locally for review. It is intended to improve visibility into clipboard-related activity on a single device without requiring root.

The app does not bypass Android's clipboard protections and does not read clipboard content. It observes only what the system chooses to log; matches indicate that the operating system emitted a clipboard-related log line for a given package, not that clipboard data was actually obtained by that package.

## Key Features

- Foreground service of type `dataSync` that keeps the monitor alive while the UI is in the background.
- `logcat` tag-filtered reader (`-s ClipboardService SamsungClipboard SemClipboardController RestrictionPolicy`).
- Regex-based extraction of `Denying clipboard access to …` (labeled `Blocked`) and `Reading clipboard` / `getPrimaryClip` (labeled `Read`).
- Per-app throttle of 1 second to coalesce log bursts from the same package.
- Allow-list that suppresses noise from common system packages (`com.android.systemui`, IME packages, `system_server`, etc.).
- Local persistence via Room (`privacy_logs_db`), storing only `appName`, `timestamp`, and a category label — never clipboard content.
- Jetpack Compose UI with a status toggle, animated top-apps bar chart, and recent-history list.
- Grouped `InboxStyle` notifications with a 1.5 s coalescing window so a burst of detections produces one alert instead of many.

## Screenshots

Screenshots are stored in [docs/assets/screenshots/](docs/assets/screenshots/). They will be added once captured on a target device. The README intentionally avoids broken image links until real screenshots exist.

## How It Works

1. `MonitorService` (a foreground service of type `dataSync`) starts on the user toggle and survives backgrounding.
2. `LogcatMonitor` spawns the platform `logcat` binary through `Runtime.getRuntime` with a tag filter restricted to clipboard-related sources.
3. Each line is matched against two patterns. `Denying clipboard access to …` is recorded as `Blocked`; lines containing `Reading clipboard` or `getPrimaryClip` are recorded as `Read`.
4. The package name is extracted with a regex and tested against an allow-list of system packages.
5. Surviving events pass through a per-app throttle (1000 ms) and are emitted into a `SharedFlow` event bus with a 128-slot buffer and `DROP_OLDEST` overflow policy.
6. The service collects events, inserts a `ClipboardEvent` row in Room, and schedules a grouped notification after a 1500 ms coalescing window.
7. The Compose UI observes the Room DAO as a `Flow`, updating the recent-history list and the top-apps statistics card in real time.

The full data path is documented at [docs/privacy-model.md](docs/privacy-model.md).

## Tech Stack

| Area | Technology |
|---|---|
| Language | Kotlin 2.0.21 |
| UI | Jetpack Compose (BOM 2024.09.00), Material3 |
| Persistence | Room 2.6.1 (KSP) |
| Async / events | Coroutines, `SharedFlow` |
| Background | Foreground Service (`FOREGROUND_SERVICE_TYPE_DATA_SYNC`) |
| Monitoring | `logcat` via `Runtime` |
| Build | Android Gradle Plugin 8.13.2 |

## Requirements

- Android device running **API 36+** (current `minSdk`).
- Android Studio.
- ADB installed and authorized for the device.
- USB debugging enabled.
- `READ_LOGS` permission granted manually through ADB (see below).

The high `minSdk` is a real limitation of this build and is tracked in the roadmap.

## Installation

1. Clone the repository:
   ```
   git clone https://github.com/Sitr3n01/privacy_clipboard_for_android.git
   ```
2. Open the project in Android Studio.
3. Build and run on a physical device that meets the requirements above.
4. Grant `READ_LOGS` via ADB (next section).
5. Open the app and toggle the "Sentinela" switch to start monitoring.

## Required Permission

```
adb shell pm grant com.example.privacyclipboard android.permission.READ_LOGS
```

Detailed ADB setup, verification, and troubleshooting steps are in [docs/adb-setup.md](docs/adb-setup.md).

## Privacy and Security Notes

- The app is designed for local audit visibility on the user's own device.
- The app does not upload events, history, or telemetry. No HTTP client (Retrofit / OkHttp / Ktor), no Firebase, no analytics SDK is included. The manifest does not declare `android.permission.INTERNET`.
- Clipboard content is not read or stored. Each `ClipboardEvent` row contains an autoincrement `id`, the detected `appName`, a `timestamp`, and a fixed-string `contentType` label (`"Leu seu Clipboard"` or `"Tentou ler (Bloqueado)"`).
- The audit database is the Room file `privacy_logs_db`, kept in app-private storage.
- `android:allowBackup="true"` is currently set in the manifest, which means the audit database may be included in Android's automatic backup. This is an open design decision tracked in the privacy model.
- Behavior depends on what the operating system and OEM emit through `logcat`; the app cannot detect events that produce no log entry.

See [docs/privacy-model.md](docs/privacy-model.md) for the full data contract.

## Limitations

- Requires ADB-granted `READ_LOGS`. The app is not installable as a self-contained consumer build.
- Current `minSdk = 36`; older Android versions cannot install this build.
- Detection depends on the OS and OEM emitting recognizable log entries. Log formats and tag names can change across Android versions and across vendors.
- The current tag set leans toward Samsung-style sources (`SamsungClipboard`, `SemClipboardController`); coverage on other OEMs is limited and not formally tested.
- The app records that a log line was emitted, not that the targeted app successfully obtained clipboard data.
- The app does not bypass Android's clipboard restrictions or any other system protection.
- This is a portfolio prototype, not a hardened consumer product.

## Development

```
./gradlew assembleDebug
./gradlew test
```

## Project Structure

```
app/
  src/main/
    AndroidManifest.xml
    java/com/example/privacyclipboard/
      MainActivity.kt
      MonitorService.kt
      LogcatMonitor.kt
      EventBus.kt
      PrivacyPill.kt
      database/
        AppDatabase.kt
        ClipboardDao.kt
        ClipboardEvent.kt
      ui/theme/
        Color.kt
        Theme.kt
        Type.kt
docs/
  adb-setup.md
  privacy-model.md
  assets/screenshots/
```

## Roadmap

- Rename `applicationId` from the scaffold-default `com.example.privacyclipboard` to an author-namespaced identifier.
- Add an in-app onboarding flow that walks the user through ADB setup (currently only a warning text is shown when the permission is missing).
- Broaden tag coverage beyond Samsung-style sources; test on non-Samsung OEMs.
- Lower `minSdk` to widen device compatibility.
- Export a shareable privacy report (CSV / JSON) from the history.
- Per-app filtering and a configurable allow-list in the UI.
- Explore Shizuku as an alternative to ADB for users who cannot or do not want to keep a development host connected.
- Unit tests for the regex parsers in `LogcatMonitor`.
- Decide on `android:allowBackup` policy for the audit database.

## My Role

This is a solo portfolio project focused on Android system-level observability, Kotlin architecture (foreground service + `SharedFlow` bus + Room), and Jetpack Compose UI. The goal of the codebase is to demonstrate a complete, honest privacy-auditing path from `logcat` to a reactive UI on a constrained, real-world platform.

## License

Released under the MIT License. See [LICENSE](LICENSE) for the full text.
