# ADB Setup

Privacy Clipboard Sentinel reads system logs through `android.permission.READ_LOGS`. This is a signature/development-level permission that cannot be requested at runtime from a normal user-installed app — it has to be granted manually via ADB.

This document walks through the one-time setup.

## 1. Install Android SDK Platform Tools

`adb` ships inside the Android SDK Platform Tools bundle. Install it from the official Android SDK Platform Tools download page (search "Android SDK Platform Tools" on `developer.android.com`). After installation, make sure `adb` is on your `PATH`:

```
adb --version
```

If you already have Android Studio installed, Platform Tools is available under `~/Library/Android/sdk/platform-tools` (macOS), `%LOCALAPPDATA%\Android\Sdk\platform-tools` (Windows), or `~/Android/Sdk/platform-tools` (Linux).

## 2. Enable USB debugging on the device

1. Open **Settings → About phone**.
2. Tap **Build number** seven times until "You are now a developer" appears.
3. Open **Settings → System → Developer options**.
4. Enable **USB debugging**.

On Samsung devices, Developer options live under **Settings → About phone → Software information → Build number**.

## 3. Authorize the host

Connect the device via USB and run:

```
adb devices
```

The first time, the device shows an **Allow USB debugging?** dialog. Accept it (optionally tick "Always allow from this computer"). The device should now appear with status `device`.

## 4. Grant READ_LOGS to the app

With Privacy Clipboard Sentinel already installed on the device, run:

```
adb shell pm grant com.example.privacyclipboard android.permission.READ_LOGS
```

This is a one-time grant. The command succeeds silently when it works.

## 5. Verify the grant

```
adb shell dumpsys package com.example.privacyclipboard
```

Look for a line of the form:

```
android.permission.READ_LOGS: granted=true
```

On Windows PowerShell or Command Prompt, you can narrow the output with:

```
adb shell dumpsys package com.example.privacyclipboard | findstr READ_LOGS
```

On macOS/Linux, use `grep`:

```
adb shell dumpsys package com.example.privacyclipboard | grep READ_LOGS
```

## 6. Re-grant after reinstall

The grant is tied to the installed application instance. If you uninstall and reinstall the app (including most updates that go through `pm install -r` from a fresh package), you must re-run step 4. Open the app: if the warning "Permissão ADB necessária para detectar!" is visible in red under the status card, the permission is not currently granted.

## Troubleshooting

- **`Operation not allowed: Package … has not requested permission`** — make sure the app is installed and that `android.permission.READ_LOGS` is declared in the manifest of the build you installed. The current debug build declares it
  in [app/src/main/AndroidManifest.xml](../app/src/main/AndroidManifest.xml).
- **`Can't find service: package`** — `adb` is connected but the device shell cannot reach `pm`. Reboot the device and try again.
- **The app still says permission missing after grant** — fully close and reopen the app; the status is checked at `onCreate()` in [MainActivity](../app/src/main/java/com/example/privacyclipboard/MainActivity.kt).
