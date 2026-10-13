# capacitor-firebase Android listener leak reproduction

Minimal reproduction for `@capacitor-firebase/authentication` and `@capacitor-firebase/app-check` on Android: the listeners they register on the process-wide `FirebaseAuth` / `FirebaseAppCheck` instances are never removed, so every same-process `MainActivity` recreation leaves the previous Activity and WebView reachable.

Created from [capawesome-team/.capacitor-app](https://github.com/capawesome-team/.capacitor-app), plus the Android platform and these packages:

- `@capacitor/android`, `@capacitor/core`, `@capacitor/cli` 8.5.3
- `@capacitor-firebase/authentication` 8.5.2
- `@capacitor-firebase/app-check` 8.5.2

The app does not call any plugin method; loading the plugins is enough.

`android/app/google-services.json` contains **placeholder values only** (`demo-listener-leak`). They are enough for the Firebase SDK to initialize locally, and no network access to Firebase is needed for the reproduction.

## Steps

```sh
npm install
npm run build
npx cap sync android
npx cap run android      # or build/install android/app debug from Android Studio
```

Keep the app in the foreground, then trigger an in-process `MainActivity` recreation several times. `fontScale` is not in the template's `android:configChanges`, so a font-scale change relaunches the activity inside the same process:

```sh
adb shell settings put system font_scale 1.1
adb shell settings put system font_scale 1.0
adb shell settings put system font_scale 1.1
adb shell settings put system font_scale 1.0
adb shell dumpsys meminfo com.example.listenerleak | grep -E "Activities|WebViews"
```

Force a GC and count again (debug builds only):

```sh
adb shell am dumpheap -g com.example.listenerleak /data/local/tmp/repro.hprof
adb shell rm /data/local/tmp/repro.hprof
adb shell dumpsys meminfo com.example.listenerleak | grep -E "Activities|WebViews"
```

Restore your font scale afterwards (`adb shell settings put system font_scale 1.0`).

## Result

Observed on an Android 16 (API 36) emulator:

| | Start | After 4 relaunches | After forced GC |
|---|---|---|---|
| 8.5.2 | 1 / 1 | 5 / 5 | **5 / 5** (leaked) |
| 8.5.2 + listeners removed in `handleOnDestroy()` | 1 / 1 | 2 / 2 | **1 / 1** |

(`Activities / WebViews` from `dumpsys meminfo`.)

The "Developer options → Don't keep activities" setting, or a low-memory activity destroy while the app is in the background, goes through the same destroy / same-process recreate path.
