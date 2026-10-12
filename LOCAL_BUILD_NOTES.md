# 本机构建修改说明（Local Build Notes）

本文件记录在 Windows 本机（NDK 30 / JDK 21 / Gradle 8.13）为构建成功所做的全部修改，
供后续维护与复现。**原始上游代码未做任何功能性改动**，以下均为构建兼容性修复。

## 构建环境

- JDK 21（Eclipse Adoptium `jdk-21.0.12.8-hotspot`）
- NDK **30.0.16248370**（必须在 `app/build.gradle` 显式指定，见下）
- CMake 3.22.1 / compileSdk 35 / AGP 8.11.1
- 仓库自带 `gradlew`（Gradle 8.13）

## 构建命令

⚠️ **必须后台执行**：单次构建 4~5 分钟，前台会因超时被 SIGTERM 中断。

```bash
cd wolvic-huawei-vr
JAVA_HOME="C:/Program Files/Eclipse Adoptium/jdk-21.0.12.8-hotspot" \
  ./gradlew assembleHvrArm64GeckoGenericDebug --console=plain
```

⚠️ **Gradle 任务名顺序是 `assemble<Platform><Abi><Backend><Store><BuildType>`**，
platform 在 abi **之前**。`README_CN.md` 里的 `assembleArm64VisionglassGeckoGenericDebug`
写法是错的（少 platform 前缀且顺序反）。已构建验证的两种：

| Flavor | 任务名 | 产物 |
|---|---|---|
| 华为 VR Glass | `assembleHvrArm64GeckoGenericDebug` | `app/build/outputs/apk/hvrArm64GeckoGeneric/debug/Wolvic-hvr-arm64-gecko-generic-debug.apk` |
| 无 SDK 回退 | `assembleNoapiArm64GeckoGenericDebug` | `app/build/outputs/apk/noapiArm64GeckoGeneric/debug/Wolvic-noapi-arm64-gecko-generic-debug.apk` |

## 修改清单（共 6 处）

### 1. `app/build.gradle` — 显式指定 NDK 版本
```gradle
android {
    ndkVersion "30.0.16248370"
}
```
原因：SDK 目录里的 `ndk/27.0.12077973` 是**空壳**（只剩 `.installer`，缺 `source.properties`），
AGP 报 `CXX1101 NDK ... did not have a source.properties`。

### 2. `KTX-Software/lib/astc-encoder/Source/cmake_core.cmake` — 抑制 fp flags 冲突告警
在**两处** fp flags（`-ffp-model=precise` + `-ffp-contract=fast/off`）之后各追加：
```cmake
$<${is_gnu_fe}:-Wno-overriding-option>
```
原因：NDK 30 的 clang 报 `overriding '-ffp-model=precise' option with '-ffp-contract=off' [-Werror,-Woverriding-option]`。
⚠️ **必须加在 `-Werror` 之后的 `target_compile_options` 内才生效**；加进 gradle 的
`cppFlags` 会被后面的 `-Werror` 反向覆盖。

### 3. `app/CMakeLists.txt` — 单 TU 抑制 tinygltf 告警
```cmake
set_source_files_properties(
    ${CMAKE_SOURCE_DIR}/../app/src/main/cpp/tinygltf/tiny_gltf.cc
    PROPERTIES COMPILE_OPTIONS "-Wno-deprecated-literal-operator")
```
原因：tinygltf 自带 `json.hpp` 的 `operator "" _json` 触发 `-Wdeprecated-literal-operator`（NDK 30 新告警）。

### 4. `app/src/main/cpp/DeviceDelegate.h` — `return nullptr` → `return std::string()`
原因：返回类型是 `std::string`，NDK 30 在 `-Werror` 下报 `-Wnonnull`。

### 5. `app/src/main/cpp/VRVideo.cpp` — `pitchRotation * yawRotation` → `PostMultiply`（3 处）
原因：**`vrb::Matrix` 没有 `operator*`**，只有 `PreMultiply` / `PostMultiply`。
语义：vrb 是列向量约定，`A.PostMultiply(B) == A * B`。
（注：VR180 pitch 功能本身是 fork 相对上游的既有改动，此处只修乘法写法。）

### 6. `app/src/openxr/cpp/OpenXRInputMappings.h` — 去掉元素类型顶层 const
```cpp
using OpenXRInputProfile = const char*;   // 原为 const char* const
```
原因：该别名被用于 `std::vector<OpenXRInputProfile>`，**顶层 const 元素类型**在新版 libc++
（NDK 30）触发 `std::allocator does not support const types` 静态断言（连带 5 处错误）。
`OpenXRButtonPath = const char* const` 仅作 struct 成员使用，无需改。

## submodule gitlink 补齐

上游用 submodule 管理 `vrb` / `tinygltf` / `KTX-Software`，但**本 fork 的 commit tree 里
不含这三个 gitlink**，导致 `git submodule update --init` 什么都不做、直接构建必然缺源码。
本次已用 `git update-index --add --cacheinfo 160000,<sha>,<path>` 补回三个 gitlink
（指向各仓库公开 master / 指定分支 HEAD），并修正 `.gitmodules` 中 KTX 路径的笔误
（原写 `app/src/KTX-Software`，实际为 `app/src/main/cpp/KTX-Software`）。

新 clone 后需执行：
```bash
git submodule update --init --recursive
```

## `third_party/hvr/` 说明

`.gitignore` 第 68 行的 `third_party` 会忽略整个目录（上游惯例：专有 SDK 不入库）。
本次为让 fork 可直接构建，已将华为 VR Glass SDK 装配内容一并提交
（`arm64-v8a/libxr_loader.so`、`armeabi-v7a/libxr_loader.so`、`hvrbridge.jar`、`include/` 4 个头文件）。

来源：华为开发者联盟公开下载的 `hvrsdk-openxr-3.5.0.79`（`sdkDemo/openXRsdk/`）。
⚠️ **这些二进制版权归华为所有**：本仓库上传仅用于本地方便构建与自测，
**对外分发前请从华为开发者联盟官方渠道获取并遵守其 SDK 许可协议**。

## `visionglass` flavor 无法构建（已知阻塞）

`assembleVisionglassArm64GeckoGenericDebug` 失败于 `compileKotlin`：
`PlatformActivity.java` 实现的 `com.huawei.usblib.OnConnectionListener` 来自
`third_party/aliceimu/*.aar`（华为 Vision Glass 的 IMU 库，原名 AliceIMU）。
该 aar **无公开来源**：仅存于 Igalia 私有仓 `wolvic-third-parties`（需成员权限）
或华为合作伙伴 SDK 门户（需 NDA）。已核查 Maven Central、华为 maven、GitHub 公开仓、
多个搜索引擎均无下载源。需该 aar 才能构建 visionglass。

## 已产出的 debug APK

- `Wolvic-hvr-arm64-gecko-generic-debug.apk`（140,069,780 B）—— 华为 VR Glass
- `Wolvic-noapi-arm64-gecko-generic-debug.apk`（134,478,105 B）—— 无 SDK 回退版

均已上传至本仓库 GitHub Release。
