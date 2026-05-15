# Privacy Model

This document defines exactly what Privacy Clipboard Sentinel observes, persists, and transmits, so that the app's behavior can be audited without reading the source.

## What the app stores

Only one Room entity is persisted: `ClipboardEvent`
([app/src/main/java/com/example/privacyclipboard/database/ClipboardEvent.kt](../app/src/main/java/com/example/privacyclipboard/database/ClipboardEvent.kt)).

| Field | Type | Source | Example |
|---|---|---|---|
| `id` | `Int` (autoGenerate) | Room | `1, 2, 3 …` |
| `appName` | `String` | Package name extracted from a `logcat` line | `com.example.someapp` |
| `timestamp` | `Long` | `System.currentTimeMillis()` at insertion | `1747234567890` |
| `contentType` | `String` (label) | One of two fixed labels | `"Leu seu Clipboard"` / `"Tentou ler (Bloqueado)"` |

The `contentType` value space is a closed set of two literal strings assigned
in [MonitorService.handleEvent](../app/src/main/java/com/example/privacyclipboard/MonitorService.kt#L61):

```kotlin
val historyText = if (type == "Bloqueado") "Tentou ler (Bloqueado)" else "Leu seu Clipboard"
```

It is a category label, not clipboard content.

## What the app does not store

- **Clipboard text, images, intents, or any other clipboard payload.** The app does not call `ClipboardManager.getPrimaryClip()` or read clipboard data at all. Verify with: `grep -R "ClipboardManager" app/src/main/java` (returns no matches).
- **Foreground app names beyond what appears in matched log lines.** The app does not query `UsageStatsManager`, `ActivityManager`, or accessibility services.
- **Device identifiers** (no `ANDROID_ID`, `IMEI`, advertising ID, account name).

## Network behavior

There is no network egress.

Confirmed by inspecting [gradle/libs.versions.toml](../gradle/libs.versions.toml)
and [app/build.gradle.kts](../app/build.gradle.kts): no HTTP client (Retrofit,
OkHttp, Ktor), no Firebase, no analytics SDK. The `AndroidManifest.xml` does not
declare `android.permission.INTERNET`.

## How events are detected

The detection path is:

1. `MonitorService` (a foreground service of type `dataSync`) starts
   `LogcatMonitor` on a coroutine dispatched to `Dispatchers.IO`
   ([MonitorService.kt:44](../app/src/main/java/com/example/privacyclipboard/MonitorService.kt#L44)).
2. `LogcatMonitor` spawns the platform `logcat` binary with the tag filter
   `-s ClipboardService SamsungClipboard SemClipboardController RestrictionPolicy`
   ([LogcatMonitor.kt:64](../app/src/main/java/com/example/privacyclipboard/LogcatMonitor.kt#L64)).
3. Each line is matched against two patterns
   ([LogcatMonitor.kt:82-89](../app/src/main/java/com/example/privacyclipboard/LogcatMonitor.kt#L82)):
   - `"Denying clipboard access to"` → labeled as `Bloqueado` (Blocked)
   - `"Reading clipboard"` or `"getPrimaryClip"` → labeled as `Leu` (Read)
4. The package name is extracted via regex
   ([LogcatMonitor.kt:19-20](../app/src/main/java/com/example/privacyclipboard/LogcatMonitor.kt#L19)).
5. Matches against the allow-list of system apps are dropped
   ([LogcatMonitor.kt:23-30](../app/src/main/java/com/example/privacyclipboard/LogcatMonitor.kt#L23)):
   - `com.android.systemui`
   - `com.example.privacyclipboard`
   - `com.samsung.android.honeyboard`
   - `com.google.android.inputmethod.latin`
   - `system_server`
   - `com.android.server`
6. A per-app throttle of 1000 ms suppresses duplicate emissions
   ([LogcatMonitor.kt:34-101](../app/src/main/java/com/example/privacyclipboard/LogcatMonitor.kt#L34)).
7. Surviving events are emitted into `EventBus.events`
   (a `SharedFlow<Pair<String, String>>` with `replay = 0`,
   `extraBufferCapacity = 128`, `BufferOverflow.DROP_OLDEST` —
   [EventBus.kt:11-15](../app/src/main/java/com/example/privacyclipboard/EventBus.kt#L11)).
8. The service collects, inserts a `ClipboardEvent` row, and schedules a
   grouped `InboxStyle` notification after a 1500 ms coalescing window
   ([MonitorService.kt:72-78](../app/src/main/java/com/example/privacyclipboard/MonitorService.kt#L72)).

Because the source of truth is `logcat`, the app reports what Android (and the
OEM) chose to log. It does not observe clipboard reads that produce no log
entry, and it cannot confirm whether a logged attempt actually succeeded in
copying data — only that the system emitted a line matching one of the
patterns above.

## Permissions and why each is requested

Declared in [app/src/main/AndroidManifest.xml](../app/src/main/AndroidManifest.xml):

| Permission | Why |
|---|---|
| `READ_LOGS` | Required to read system `logcat` output. Not granted automatically; the user must grant it through ADB. |
| `FOREGROUND_SERVICE` | Required to run `MonitorService` while the UI is not in the foreground. |
| `FOREGROUND_SERVICE_DATA_SYNC` | Required on Android 14+ because the service is declared with `foregroundServiceType="dataSync"`. |
| `POST_NOTIFICATIONS` | Required on Android 13+ to display the ongoing service notification and grouped alerts. |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Allows prompting the user to exempt the app from battery-optimization sleep so the service keeps reading `logcat`. |
| `RECEIVE_BOOT_COMPLETED` | Declared but no `BootCompletedReceiver` is currently wired up; left in place for a planned auto-start feature. |

The app does not request `INTERNET`.

## Open considerations

- `android:allowBackup="true"` is currently declared in the manifest
  ([AndroidManifest.xml:13](../app/src/main/AndroidManifest.xml#L13)). This
  means the Room database may be included in Android's automatic backup. A
  follow-up may set `allowBackup="false"` or define a stricter
  `data_extraction_rules.xml` once the audit history is considered sensitive
  enough to require it.
- The package name `com.example.privacyclipboard` is a scaffold default and
  is hard-coded into the ADB grant command in the README. A follow-up will
  rename the `applicationId` to something namespaced to the author.
