# HalideAndroidCamera2Example

This is a fork from [HelloAndroidCamera2](https://github.com/halide/Halide/tree/master/apps/HelloAndroidCamera2).

It uses Android Camera2 API to capture a YUV stream, applies an edge detector and draws the result directly on the Java SurfaceView through ANativeWindow.


This fork is compatible with latest Android Studio C++ CMake toolchain.

# Setup

In `gradle.properties` set
```
halideDir=A:/dev/cv/halide-build
hostClang=C:/data-ssd/dev/cv/llvm-install/bin/clang++.exe
```
`halideDir` points to your Halide release build.

`hostClang` points to the clang executable.


# Linking Halide Pipelines
`app/src/main/cpp/hl-pipelines/CMakeLists.txt` generates and links the Halide pipelines to the native JNI library.

It expects all `*.cpp` files in the same folder to be generators with a call to `HALIDE_REGISTER_GENERATOR(...)`.


# Supported Android ABIs
* `arm64-v8a`
* `x86_64`
