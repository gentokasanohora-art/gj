# PokéFinder – Android APK Build Guide

This document explains how to build an Android APK from the PokéFinder
source using either **GitHub Actions** (recommended) or a local machine.

---

## GitHub Actions (automated, no local setup needed)

The workflow file `.github/workflows/android.yaml` runs automatically on every
push to `master` / `feature/**` branches and on pull requests.

### Steps
1. Fork or push this repository to GitHub.
2. Go to **Actions → CI - Android APK** and trigger a run (or push a commit).
3. After the job succeeds, download the APK from the **Artifacts** section of
   the run summary.

### Release signing (optional)
To produce a signed release APK on version tags (`v*.*.*`):

1. Generate a keystore:
   ```bash
   keytool -genkeypair -v \
     -keystore pokefinder.jks \
     -keyalg RSA -keysize 2048 \
     -validity 10000 \
     -alias pokefinder
   ```
2. Base64-encode it:
   ```bash
   base64 pokefinder.jks > pokefinder.jks.b64
   ```
3. Add these **repository secrets** (Settings → Secrets → Actions):
   | Secret name         | Value                                    |
   |---------------------|------------------------------------------|
   | `KEYSTORE_BASE64`   | Contents of `pokefinder.jks.b64`         |
   | `KEYSTORE_PASSWORD` | Keystore password you chose              |
   | `KEY_ALIAS`         | `pokefinder` (or your alias)             |
   | `KEY_PASSWORD`      | Key password you chose                   |
4. Push a tag: `git tag v4.4.2 && git push --tags`
   The signed APK is attached to the GitHub Release automatically.

---

## Local build prerequisites

| Tool              | Version                  |
|-------------------|--------------------------|
| CMake             | ≥ 3.31                   |
| Ninja             | any recent               |
| Python            | ≥ 3.14                   |
| Java (JDK)        | 17                       |
| Android NDK       | 26.3.11579264            |
| Android SDK       | platform 34, tools 34.0.0|
| Qt6 for Android   | 6.10.1 `android_arm64_v8a` |
| Qt6 for desktop   | 6.10.1 (host tools)      |

### Configure & build

```bash
# Point to your Qt Android installation
QT_ANDROID=/path/to/Qt/6.10.1/android_arm64_v8a
QT_HOST=/path/to/Qt/6.10.1/gcc_64        # or macos / win host

$QT_ANDROID/bin/qt-cmake \
  -B build-android \
  -G Ninja \
  -DANDROID_PLATFORM=android-26 \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_SDK_ROOT=$ANDROID_HOME \
  -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME \
  -DQT_HOST_PATH=$QT_HOST \
  -DCMAKE_BUILD_TYPE=Release \
  -DTEST=OFF \
  -DENABLE_SIMD=OFF \
  .

cmake --build build-android --parallel
cmake --build build-android --target apk
```

The resulting APK is located at:
```
build-android/android-build/build/outputs/apk/debug/android-build-debug.apk
```

---

## Notes

- **Minimum Android version:** API 26 (Android 8.0 Oreo) – required for C++23
  features used by the core library.
- **Supported ABIs:** `arm64-v8a` (the workflow builds this by default; you can
  add `armeabi-v7a` and `x86_64` by repeating the configure + build steps with
  `-DANDROID_ABI=<abi>`).
- **Qt Widgets on Android:** Qt's widget layer works on Android via a fullscreen
  `QScreen`; the UI is not natively "Material" but it is fully functional.
- **SIMD:** Disabled for Android (`-DENABLE_SIMD=OFF`) because the SIMD paths
  use x86 intrinsics. ARM NEON support can be added separately.
