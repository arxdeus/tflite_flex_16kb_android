# TFLite Flex Delegate for Android (16KB Page Size)

CI repository for building `libtensorflowlite_flex.so` with 16KB page size support for Android.

## Why

Starting with Android 15, arm64 devices may use 16KB page size instead of 4KB.
Google Play [requires](https://developer.android.com/guide/practices/page-sizes) 16KB compatibility for new apps.

Pre-built `.so` files from TensorFlow do not support 16KB page alignment.
This repository builds the flex delegate with the correct linker flags:
- `-Wl,-z,max-page-size=16384`
- `-Wl,-z,common-page-size=16384`

## Architectures

| ABI | Description |
|-----|-------------|
| `arm64-v8a` | 64-bit ARM (modern devices) |
| `armeabi-v7a` | 32-bit ARM (legacy devices) |

## Running the Build

**Manual:** Actions → *Build TFLite Flex Delegate for Android* → Run workflow

You can specify a TensorFlow git ref (branch, tag, or SHA). Defaults to `master`.

**Automatic:** triggers on changes to `patches/` or the workflow file.

## Artifacts

After a successful build, the following artifacts are available in Actions:

| Artifact | Contents |
|----------|----------|
| `libtensorflowlite_flex-arm64-v8a` | `.so` for arm64 |
| `libtensorflowlite_flex-armeabi-v7a` | `.so` for armv7 |
| `tflite-flex-delegate-android-jniLibs` | Ready-to-use `jniLibs/` directory |

## Replacing `.so` in a project using `flutter_litert_flex`

The [`flutter_litert_flex`](https://github.com/hugocornellier/flutter_litert_flex) package
pulls in `org.tensorflow:tensorflow-lite-select-tf-ops` via Maven, which bundles its own
`libtensorflowlite_flex.so` **without** 16KB page alignment. To replace it with our build:

### Step 1 — Add the custom `.so` files

Download the `tflite-flex-delegate-android-jniLibs` artifact and copy into your **app**:

```
your_app/android/app/src/main/jniLibs/
├── arm64-v8a/libtensorflowlite_flex.so
└── armeabi-v7a/libtensorflowlite_flex.so
```

### Step 2 — Exclude the Maven native library

In your **app-level** `android/app/build.gradle.kts`, exclude the native `.so` bundled
inside the Maven AAR so it doesn't conflict with our custom build:

```kotlin
android {
    // ...

    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a")
        }
    }

    packaging {
        jniLibs {
            // Prefer our custom .so from jniLibs/ over the one inside the Maven AAR
            pickFirsts += listOf(
                "lib/arm64-v8a/libtensorflowlite_flex.so",
                "lib/armeabi-v7a/libtensorflowlite_flex.so",
            )
        }
    }
}
```

> **Note:** `pickFirsts` tells Gradle to use the first match when the same `.so` appears in
> both `jniLibs/` and the AAR. Files in `jniLibs/` take priority, so our 16KB-aligned
> build will be used.

If you use Groovy `build.gradle` instead of Kotlin DSL:

```groovy
android {
    defaultConfig {
        ndk {
            abiFilters 'arm64-v8a', 'armeabi-v7a'
        }
    }

    packagingOptions {
        pickFirst 'lib/arm64-v8a/libtensorflowlite_flex.so'
        pickFirst 'lib/armeabi-v7a/libtensorflowlite_flex.so'
    }
}
```

### Step 3 — Verify

Build the APK and confirm the correct `.so` is bundled:

```bash
cd your_app
flutter build apk
# Check the .so inside the APK
unzip -l build/app/outputs/flutter-apk/app-release.apk | grep libtensorflowlite_flex
```

You should see one entry per ABI. To verify 16KB alignment:

```bash
unzip -o build/app/outputs/flutter-apk/app-release.apk lib/arm64-v8a/libtensorflowlite_flex.so -d /tmp
readelf -l /tmp/lib/arm64-v8a/libtensorflowlite_flex.so | grep -i align
```

The LOAD segment alignment should show `0x4000` (16384).

### How it works

The `flutter_litert_flex` plugin declares a Maven dependency on
`org.tensorflow:tensorflow-lite-select-tf-ops`, which provides:
- **Java class** `org.tensorflow.lite.flex.FlexDelegate` — still needed, kept as-is
- **Native library** `libtensorflowlite_flex.so` — replaced by our 16KB-aligned build

The `pickFirsts` directive resolves the duplicate: Gradle picks our `jniLibs/` copy
and discards the one from the AAR.

## Patches

### `fix-android-flex-delegate-build.patch`

Fixes **duplicate symbol** linker errors when building the flex delegate for Android.

**Root cause:** `portable_tensorflow_lib_lite` unconditionally depends on `onednn_env_vars`
(Intel MKL — useless on ARM), which pulls in `//tensorflow/core:framework` (the desktop
framework library). On Android, `framework_internal_impl` gets linked via `if_static()`,
duplicating symbols already present in `portable_tensorflow_lib_lite`
(`graph_def_util.cc`, `kernel_def_builder.cc`, `graph_to_functiondef.cc`, etc.).

**Fix:**
- Guard `onednn_env_vars` behind `if_not_mobile()`
- Add explicit `protobuf` and `executor` deps that were previously transitive through `framework_internal_impl`

## Versions

| Component | Version |
|-----------|---------|
| Android NDK | r28c (28.2.13676358) |
| Android SDK | API 36 |
| Build Tools | 36.1.0 |
| Bazel | 7.7.0 (from TF `.bazelversion`) |
| Python | 3.12 (hermetic) |

## Repository Structure

```
├── .github/workflows/
│   └── build-flex-delegate-android.yml   # GitHub Actions workflow
├── patches/
│   └── fix-android-flex-delegate-build.patch
└── README.md
```
