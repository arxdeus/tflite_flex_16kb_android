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

## Flutter Integration

1. Download the `tflite-flex-delegate-android-jniLibs` artifact
2. Copy into your project:

```
android/app/src/main/jniLibs/
├── arm64-v8a/libtensorflowlite_flex.so
└── armeabi-v7a/libtensorflowlite_flex.so
```

3. In `android/app/build.gradle`:

```groovy
android {
    defaultConfig {
        ndk {
            abiFilters 'arm64-v8a', 'armeabi-v7a'
        }
    }
}
```

4. In Dart:

```dart
import 'package:tflite_flutter/tflite_flutter.dart';

final interpreter = await Interpreter.fromAsset(
  'model.tflite',
  options: InterpreterOptions()..addDelegate(FlexDelegate()),
);
```

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
