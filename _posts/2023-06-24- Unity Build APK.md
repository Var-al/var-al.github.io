---
layout: post
title: Unity Build APK
date:   2023-06-22 17:48
description: Unity 打APK记录
toc: true
comments: true
categories:
 - blog
tags:
 - Unity
---

从两个角度详细说明：**1. Unity 直接打包 APK (默认方式)** 和 **2. 导出 Android Studio 项目再打包 (Export Project)**。并比较它们的异同、流程和所需设置。

## **核心概念解析**

打包过程中涉及到的几个关键组件及其作用：

1.  **JDK (Java Development Kit):**
    *   **作用:** 编译 Unity 项目中包含的 Java 代码（比如 Android Plugin 或 Unity 自动生成的部分代码）。它提供了 `javac` (Java 编译器) 等工具。
    *   **Unity 如何使用它:** Unity 需要 JDK 来将 `.java` 文件编译成 `.class` 文件（Java 字节码）。这个字节码之后会被转换为 Android 虚拟机理解的格式 (.dex)。
    *   **位置:** 通常由 Unity Hub 自动安装 (Unity 2020+ 推荐使用内置的或指定的版本)，可以在 Unity 的 `Edit -> Preferences -> External Tools` 中配置其路径。`%JAVA_HOME%` 环境变量有时也需要设置。
    *   **常见版本:** Java 8, Java 11 (Unity 2020 LTS+ 推荐), 偶尔需要 Java 17+。**关键点：Unity 版本、Android Gradle Plugin版本、Gradle 版本、JDK 版本相互之间需要兼容！** Unity 会指定可用的版本范围。

2.  **Android SDK (Software Development Kit):**
    *   **作用:** 包含了构建和运行 Android 应用所需的所有工具、库和平台代码。
    *   **核心组件:**
        *   **`build-tools`:** 包含 `aapt` (打包资源), `dx` (已废弃，现在由 `d8` 替代, 将 `.class` 转 `.dex`), `zipalign` (优化 APK) 等编译关键工具。
        *   **`platform-tools`:** 包含 `adb` (连接调试设备), `fastboot` (刷机) 等。
        *   **`platforms` (或称 `android-xx`):** 包含对应 Android API level 的系统镜像、库文件 (`android.jar`)。
        *   **`cmdline-tools`:** 新的命令行工具 (如 `sdkmanager`)。
    *   **Unity 如何使用它:** Unity 在打包时调用 SDK 中的工具来编译资源、将 Java 字节码转成 Dalvik/ART 字节码 (.dex)、签名 APK、进行 `zipalign` 等。
    *   **位置:** 可由 Unity Hub 自动下载安装，或手动配置路径 (`Edit -> Preferences -> External Tools`)。Unity 的 `Build Settings` 中选择目标 Android API Level (`Target SDK`) 后，Unity 需要找到对应 `platforms/android-xx` 目录。

3.  **NDK (Native Development Kit):**
    *   **作用:** 用于编译 C/C++ 代码为 Android 本地库 (`.so` 文件)。
    *   **Unity 如何使用它:** 当在 Player Settings 中选择 `IL2CPP` 后端编译选项，或者项目使用了需要编译的本地插件 (.a 或 .so) 时，Unity 需要 NDK 来编译 C++ 代码（IL2CPP 将 C#/IL 代码转换为 C++ 代码）。
    *   **位置:** 可由 Unity Hub 自动下载安装（强烈推荐），或手动配置路径 (`Edit -> Preferences -> External Tools`)。**版本要求非常高！** Unity 每个版本通常指定唯一兼容的 NDK 版本。用错版本几乎必然导致编译失败。

4.  **Gradle:**
    *   **作用:** 一个强大的项目构建自动化工具。它基于 Groovy 或 Kotlin DSL 脚本 (`build.gradle`) 定义项目如何构建、依赖哪些库、如何打包、签名等。
    *   **Unity 如何使用它:**
        *   **Unity 2018.1+ 的默认 Android 构建系统：** Unity 内部集成了特定版本的 Gradle 和 Android Gradle Plugin (AGP)。它生成必要的 `build.gradle` 文件，然后**调用它自己的内置 Gradle Wrapper** 去执行整个构建过程。**这是 `Gradle (Installed)` 选项通常不需要勾选的原因！** Unity 管理一切。
        *   **`Gradle (Installed)` 选项：** 如果勾选此选项 (`Build Settings` -> `Build System` 为 `Gradle`，然后勾选 `Export Project` 下方的 `Gradle (Installed)`，或在 `Player Settings -> Publishing Settings` 中找到)，Unity 会使用系统环境变量 `PATH` 中配置的系统级 Gradle 命令 (`gradle` 或 `gradlew`) 来执行构建，**而不是**它内置的 Gradle。**除非有特殊需求（如需要使用系统 Gradle 的特定版本或插件），否则不建议手动勾选此选项并管理自己的 Gradle 环境，用 Unity 内置的就行。**
        *   **Export Project:** 导出为 Android Studio 项目时，项目会自带一个 `gradle/wrapper/gradle-wrapper.properties` 文件，里面指定了项目**需要**的 Gradle 版本。当用 Android Studio 打开项目或执行 `gradlew` 命令时，它会自动下载该指定版本的 Gradle 到本地缓存 (如果尚未下载)。Android Studio 也允许配置使用本地已安装的系统级 Gradle (`File -> Settings -> Build, Execution, Deployment -> Gradle`)。

5.  **Android Gradle Plugin (AGP):**
    *   **作用:** Gradle 的一个插件，专门用于构建 Android 应用。它定义了构建 Android APK 的特定任务 (task)，如 `:app:assembleDebug`, `:app:assembleRelease`。
    *   **Unity 如何使用它:** Unity 在生成 Gradle 项目 (无论是内部构建还是导出项目) 时，会在 `build.gradle` 文件中指定 `com.android.tools.build:gradle:版本号` 依赖。这个版本由 Unity 版本严格控制。
    *   **版本关键性:** Unity 版本会锁定兼容的 AGP 版本。如果 Unity 生成的项目指定用 AGP 7.4.x，但强制 Android Studio 使用 AGP 8.x 去构建，很可能失败。

---

## **一、Unity 直接构建 APK (Build and Run / Build)**

**目标：** 在 Unity Editor 内一键生成可直接安装到设备的 `.apk` 文件。

**流程图解：**

```
+-------------------------------------------------------------------+
|                      Unity Editor                                 |
|  [Project Files (.cs, .shader, .png, etc.) + Unity Settings]      |
|                                                                   |
|  [Build Settings]                                                 |
|  - Platform: Android                                              |
|  - Target Arch: ARMv7, ARM64, x86, x86_64                         |
|  - Min API Level: e.g., Android 8.0 (API 26)                      |
|  - Target API Level: e.g., Android 13.0 (API 33)                  |
|  - Build System: Internal or Gradle                               |
|  - Texture Compression: ETC2, ASTC, etc.                          |
|  - Compression: Default, LZ4HC, LZ4 (IL2CPP)                      |
|  - Scripting Backend: IL2CPP (推荐) 或 Mono                      |
|  - Player Settings... (签名, 包名, 图标, 权限, etc.)              |
+---------------------+---------------------------------------------+
                      | (配置检查与环境调用)
                      v
+---------------------+---------------------------------------------+
|       Unity Build Pipeline (C# / IL Code, Assets Processing)      |
|  - Compile C# -> IL (Managed dlls)                                |
|  - Preprocess & Convert Assets (Textures, Models, Audio)          |
|  - Generate APK resource package (.ap_ file)                      |
|  - Generate Java stub code (UnityPlayerActivity, etc.)           |
|  - Generate `AndroidManifest.xml` (合并项目设置与自定义)          |
|  - If IL2CPP: Run IL2CPP -> Converts IL to C++ source             |
|  - If IL2CPP: Invoke NDK to Compile C++ to .so (Native Libraries) |
+---------------------+---------------------------------------------+
                      | (调用Android工具链)
                      | (Unity manages Gradle internally or uses system)
                      v
+---------------------+---------------------------------------------+
|       Gradle (Internal / Managed by Unity)                        |
|  - Copies: Unity assets, resources, jniLibs (.so),                |
|             stub Java code, manifest to Gradle project layout     |
|  - Compiles Java stubs + Plugins: `javac` (JDK) -> .class         |
|  - Converts .class -> .dex: `d8`/`dx` (Android SDK build-tools)   |
|  - Packages: .dex + resources + assets + native libs -> .ap_?      |
|  - Creates unsigned APK                                           |
|  - Signs APK with Keystore (debug or custom)                      |
|  - Zipaligns APK for optimization: `zipalign` (Android SDK)      |
|                                                                   |
+---------------------+---------------------------------------------+
                      |
                      v
               [MyGame.apk] (输出到选择的文件夹)
```

**详细流程步骤：**

1.  **Unity 配置 (项目级 & 构建前):**
    *   **安装必要组件:** 通过 Unity Hub 安装目标版本的 Unity Editor。
    *   **安装 Android 支持:** 在 Unity Hub 中为该版本安装 **Android Build Support** (它会顺带安装兼容的 OpenJDK、Android SDK & NDK，推荐方式！)。
    *   **设置 Android SDK/NDK/JDK 路径:**
        *   打开 Unity Editor。
        *   转到 `Edit -> Preferences -> External Tools`。
        *   **如果 Unity Hub 安装了组件：** 这些路径通常会自动设置好（路径如 `%USERPROFILE%\AppData\Local\UnityHub\Editor\<Version>\Editor\Data\PlaybackEngines\AndroidPlayer\` 的子目录）。**优先使用这个自动安装的路径！**
        *   **如果手动安装或使用已有路径：** 点击对应字段旁边的 `Browse` (...)，选择的 JDK (如果非自动), Android SDK, NDK 目录。**强烈建议通过 Hub 安装以避免冲突和兼容性问题。**
    *   **Player Settings:** (最重要！)
        *   打开 `Edit -> Project Settings -> Player`。
        *   **Icon:** 设置各种尺寸的应用图标。
        *   **Resolution and Presentation:** 设置默认方向、全屏等。
        *   **Splash Image (Pro Only):** 设置启动画面。
        *   **Other Settings:**
            *   **Identification:**
                *   `Package Name`: Android 应用的唯一标识符 (e.g., `com.YourCompany.YourGame`)。
                *   `Version`: 应用版本号 (e.g., `1.0.0`)。
                *   `Bundle Version Code`: 内部整数版本号 (必须随更新递增)。
            *   **Configuration:**
                *   `Scripting Backend`: **IL2CPP** (推荐，性能好，支持 64位, 反编译难) 或 **Mono** (旧版兼容快)。
                *   `API Compatibility Level`: 一般为 `.NET Standard 2.1` (通用性较好) 或 `.NET Framework`。
                *   `Target Architectures`: 勾选需要支持的 CPU 架构 (ARMv7, ARM64)。
            *   **Optimization:** `Strip Engine Code` (减小包体，推荐开启，测试充分)。
        *   **Publishing Settings:**
            *   `Keystore Manager`: **极其重要！** 创建或选择用于签名的密钥库文件 (.keystore或.jks)。打正式包 (Release) **必须**使用自定义证书，Debug 包使用默认临时证书。
                *   记录好 `Keystore` 路径、`Keystore Password`, `Key Alias`, `Key Password`! 丢失或遗忘将**无法更新应用**。
            *   `Minify`: (ProGuard/R8) 可选，用于混淆和优化 Release 包代码，能减小包体但可能引入问题，需谨慎配置。
            *   `Split Application Binary`: 分包 (APK + OBB)，可规避 Play Store 100MB APK 限制。现已不强制要求（App Bundle 更好）。

2.  **Build Settings:**
    *   打开 `File -> Build Settings...`.
    *   平台选择 `Android`，点击 `Switch Platform`。
    *   检查并配置 (可选):
        *   `Texture Compression`: 根据目标设备选择 (ETC2 通用性好，ASTC 效果好但需设备支持)。
        *   `Create Symbols.zip`: 仅 IL2CPP Debug 包有用，用于 C++ 崩溃分析。
        *   `Compression Method`: 包内资源压缩方式。
        *   `Build System`: `Gradle` (默认、现代、支持 App Bundle), `Internal` (旧版，快但过时，不支持新特性)。
        *   `Export Project`: **取消勾选！** (这是直接打包的关键)。
        *   `Run Device`: 如果设备已连接并开启 USB 调试，选设备名可直接构建并安装到设备。
        *   `Development Build`: 调试用，包含 `Development` 标记和脚本调试符号。
        *   `Autoconnect Profiler` / `Deep Profiling Support`: 调试分析用。
    *   点击 `Build` 或 `Build And Run`.

3.  **Unity 构建执行:**
    *   Unity Editor 会执行流程图描述的 **Unity Build Pipeline** 部分。这一步时间较长，处理代码编译、资源转换、场景烘焙等。
    *   Unity 会初始化或验证配置的工具链环境 (JDK, SDK, NDK)。

4.  **Unity 调用 Gradle:**
    *   完成自身的资源处理、C#编译和（如果是 IL2CPP）C++代码生成/编译后，Unity 会：
        *   将所有必要文件（资产、资源、编译好的库、AndroidManifest.xml、Unity Player Java Stub 代码、的或插件的 Java 代码）复制到一个临时目录，并将其组织成一个符合 Android Gradle 项目结构（`src/main`）的目录。
        *   **内部 Gradle:** Unity 使用其**内置捆绑**的 Gradle Wrapper (`gradlew` 命令) 来构建这个临时的 Gradle 项目。它使用了 Unity 控制的 **Android Gradle Plugin (AGP)** 和 **Gradle 版本**。通常不需要关心。
        *   **系统 Gradle:** 如果强制勾选了 `Gradle (Installed)` 并配置了系统 PATH，Unity 会调用系统的 `gradle` 命令。这时需要确保系统 Gradle 版本、AGP 版本（由 Unity 的 `build.gradle` 模板指定）完全兼容，否则**大概率失败**。**不推荐**新手使用。
    *   Gradle 任务链启动，调用 JDK (`javac`) 编译 Java 代码，调用 Android SDK 工具 (`d8`, `aapt2`, `zipalign`, `apksigner`) 将 `.class` 转换为 `.dex`、打包资源、合并最终 APK 组件、签名和对齐。

5.  **输出 APK:**
    *   经过上述步骤，最终生成的 `.apk` 文件会输出到指定的目录。
    *   如果勾选了 `Build And Run` 并且设备已连接且选择正确，APK 会自动安装到设备并启动。

---

## **二、导出 Android Studio 项目后再打包 (Export Project)**

**目标：** Unity 将项目导出为一个标准的 Android Gradle 项目结构，然后开发者可以在 Android Studio 中打开、修改配置、添加原生代码或插件、使用 Gradle 命令行或 Android Studio GUI 最终构建 APK/AAB。

**流程图解：**

```
+-------------------------------------------------------------------+
|                      Unity Editor                                 |
|  [Project Files (.cs, .shader, .png, etc.) + Unity Settings]      |
|                                                                   |
|  [Build Settings]                                                 |
|  - Platform: Android                                              |
|  - Export Project: [勾选]                                         |
|  - 其他设置与直接打包类似 (Player Settings, Build Options)        |
+---------------------+---------------------------------------------+
                      | (配置完成)
                      v
+---------------------+---------------------------------------------+
|       Unity Build Pipeline (大部分相同)                           |
|  - Compile C# -> IL (Managed dlls)                                |
|  - Preprocess & Convert Assets (Textures, Models, Audio)          |
|  - Generate APK resource package (.ap_ file) *注意，资源格式不同*|
|  - Generate Java stub code                                       |
|  - Generate `AndroidManifest.xml`                                 |
|  - If IL2CPP: Run IL2CPP, Invoke NDK -> .so                       |
+---------------------+---------------------------------------------+
                      | (不调用JDK/SDK工具链打包)
                      | (输出完整项目结构)
                      v
+---------------------+---------------------------------------------+
|     Exported Android Studio Project Folder                        |
|  ├── gradle/                <-- Gradle Wrapper (指定了Gradle版本) |
|  ├── gradlew & gradlew.bat  <-- Wrapper启动脚本                   |
|  ├── settings.gradle        <-- 定义项目根                        |
|  ├── build.gradle           <-- 项目级构建配置 (顶层)             |
|  ├── gradle.properties      <-- Gradle属性                        |
|  ├── local.properties        <-- SDK, NDK *路径* (由Unity生成)    |
|  │                                                               |
|  └── [UnityProjectName]/    <-- 模块目录 (通常是 'launcher')      |
|      ├── build.gradle        <-- *模块级*构建配置 (核心,定义插件、依赖、buildTypes)
|      ├── src/main/           <-- 核心Android项目结构              |
|      │   ├── AndroidManifest.xml                                 |
|      │   ├── assets/         <-- Unity的Assets文件夹内容          |
|      │   ├── java/           <-- Unity生成的Java代码 + 的插件Java代码
|      │   ├── jniLibs/        <-- IL2CPP生成的.so或插件的本地库     |
|      │   └── res/            <-- Unity Player所需资源 (已处理)    |
|      └── libs/               <-- 可能包含的jar/aar库              |
+-------------------------------------------------------------------+
                      | (开发者在此项目上工作)
                      v
+---------------------+---------------------------------------------+
|                Android Studio (or Command Line)                   |
|  - Open project: Select exported folder.                          |
|  - *Verify/Sync Gradle:* AS下载所需Gradle版本、插件、依赖库.      |
|  - *Modify:* (可选)                                              |
|       * Edit `build.gradle` (module): 添加依赖, 配置签名,         |
|           buildTypes (debug/release), minify (ProGuard/R8)规则.   |
|       * Edit `AndroidManifest.xml`: 添加/修改权限、功能、活动属性|
|       * Add Native (C++/Java/Kotlin) code/modules/plugins.       |
|  - Build: `Build -> Build Bundle(s)/APK(s)` 或使用 Gradle任务.    |
|                                                                   |
| +----------------- Gradle Build (using local/wrapper) ----------+|
| | (与Unity直接打包的内部Gradle步骤 *高度相似*):                    ||
| | - Compile Java/Kotlin: `javac`/`kotlinc` (JDK) -> .class      ||
| | - Convert to .dex: `d8` (Android SDK build-tools)              ||
| | - Package resources: `aapt2`                                   ||
| | - Merge assets, native libs, manifest, .dex, etc. into APK/AAB||
| | - Sign APK/AAB: `apksigner`                                    ||
| | - Zipalign (if APK): `zipalign`                               ||
| +----------------+------------------------------------------------+|
|                  |                                                |
|                  v                                                |
|        [MyGame.apk / MyGame.aab] (输出到模块的 `build/outputs` 目录)
+-------------------------------------------------------------------+
```

**详细流程步骤：**

1.  **Unity 配置 (与直接打包大部分相同):**
    *   同样需要安装 Unity + Android Build Support (含 JDK/SDK/NDK)。
    *   同样需要在 Player Settings 中配置包名、版本号、图标、启动画面、脚本后端、目标架构等。
    *   **关键差异：**
        *   在 `Build Settings` 窗口中，**必须勾选 `Export Project` 选项！**
        *   配置 `Keystore` **不是必须在这里完成的**！因为签名通常在 Android Studio 的模块级 `build.gradle` 文件中配置（推荐方式，更符合原生开发习惯）。可以在 Unity 的 Player Settings 里配，也可以不配（留空）后面在 Android Studio 里配。
        *   对于`Minify`和`Split Application Binary`等设置，Unity 会生成相应配置，但也可以在 Android Studio 的 `build.gradle` 中更精确地控制。

2.  **导出项目:**
    *   点击 `Build Settings` 中的 `Export` 按钮（不是 Build!）。
    *   选择一个空的文件夹或指定目录作为导出位置。
    *   Unity 会执行其构建管线的**核心部分**（C#编译、资源处理、代码生成等），但**不会调用 `d8`, `aapt2` 等 Android 工具打包最终 APK 资源**。
    *   相反，它会生成一个完整的 **Android Gradle 项目结构**（如上流程图所示）。特别注意 `local.properties` 文件，它包含了 Android SDK 和 NDK 在 Unity 导出时的**绝对路径**（Windows, Mac, Linux 路径不同）。这个文件对后续在 Android Studio 中构建**至关重要**。

3.  **在 Android Studio (AS) 中操作:**
    *   打开 Android Studio。
    *   选择 `Open` -> 导航到上一步导出的**项目根目录** (包含 `settings.gradle`, `gradle` 文件夹的目录)。
    *   **Gradle Sync:**
        *   第一次打开时，AS 会检测到项目使用 Gradle Wrapper。它会解析 `gradle/wrapper/gradle-wrapper.properties` 文件，查找项目所需的 Gradle 版本号 (e.g., `distributionUrl=https\://services.gradle.org/distributions/gradle-7.6-bin.zip`)。
        *   AS 会检查本地缓存（通常在 `%USERPROFILE%\.gradle\wrapper\dists`）是否有这个 Gradle 版本。**如果没有，会从服务器下载。**
        *   同时，AS 会下载项目 `build.gradle` 文件中指定的 **Android Gradle Plugin (AGP)** 版本和其他依赖库 (如 `com.android.tools.build:gradle:版本号`)。
        *   这个同步过程需要稳定的网络连接，完成后状态栏应提示 "Gradle project sync completed"。
    *   **验证/修复项目:**
        *   **检查 JDK:** AS 有自己的 JDK 设置 (`File -> Project Structure -> SDK Location`)。确保它使用的是兼容的 JDK (通常推荐 Android Studio 自带的 JDK 或系统兼容版本)。
        *   **检查 NDK & SDK:** 正常情况下，AS 会读取 `local.properties` 文件中的 `sdk.dir` 和 `ndk.dir`（由 Unity 生成），并使用这些路径。确保路径有效且包含了所需的 SDK Platform、Build-Tools 和 NDK。如果 `local.properties` 内容不正确或缺失，**需要手动创建或编辑它** (在项目根目录下)。格式如：
            ```
            sdk.dir = C:\\Path\\To\\Your\\Android\\Sdk
            ndk.dir = C:\\Path\\To\\Your\\Android\\Ndk
            ```
        *   **查看模块级 `build.gradle`:** 进入 `[UnityProjectName]` (通常是 `launcher`) 模块，打开 `build.gradle` (`Module: [UnityProjectName].main`)。检查 AGP 版本、`minSdk`、`targetSdk`、`versionCode`、`versionName`。**最重要的配置步骤在此：**
            *   **签名配置 (`signingConfigs`):** 在这里配置的 Release 签名密钥。**强烈推荐在此配置**，而不是依赖 Unity 的 Player Settings 里设置的签名文件路径。示例：
                ```groovy
                android {
                    signingConfigs {
                        release {
                            storeFile file('your_keystore.jks') // 相对或绝对路径
                            storePassword 'your_keystore_password'
                            keyAlias 'your_key_alias'
                            keyPassword 'your_key_password'
                        }
                    }
                    buildTypes {
                        release {
                            signingConfig signingConfigs.release
                            minifyEnabled true // 启用代码混淆/优化
                            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-unity.txt' // Unity提供的基础规则 + 自定义规则
                        }
                    }
                }
                ```
            *   **构建类型 (`buildTypes`):** 定义 debug 和 release（及可自定义的其他类型）。可以在不同类型中启用/禁用 minify (`minifyEnabled`)、应用不同的签名配置、设置不同的优化选项等。
            *   **依赖项 (`dependencies`):** Unity 会自动添加对 `unityLibrary` 模块（如果导出了多个模块）和 Unity 运行时库的依赖。可以在这里添加自己的原生库依赖 (`implementation` / `api`)。
        *   **修改 `AndroidManifest.xml`:** 如果需要添加新的权限、服务、活动属性、元数据等，修改 `src/main/AndroidManifest.xml` 文件。注意不要破坏 UnityPlayerActivity 的设置。
        *   **添加原生代码 (可选):** 可以在 `src/main/java` 或 `src/main/kotlin` 添加 Java/Kotlin 代码，在 `src/main/cpp` 添加原生 C/C++ 代码（需配置 CMake/ndk-build）。可以添加原生模块或插件。

4.  **构建 APK/AAB:**
    *   **在 Android Studio 中:**
        *   确保 Build Variant 正确 (`Build -> Select Build Variant...`，选择 `release` 或带签名的 debug)。
        *   点击菜单 `Build -> Build Bundle(s) / APK(s) -> Build APK` 或 `Build Bundle(s)`。输出文件在 `[module]/build/outputs/apk/release` 或 `.../bundle/release`。
    *   **使用命令行 (Gradle Wrapper):**
        *   在终端/命令行中，导航到导出的项目**根目录**（包含 `gradlew` 文件）。
        *   执行命令：
            *   `./gradlew assembleDebug` (Debug APK) - macOS/Linux
            *   `gradlew.bat assembleDebug` (Debug APK) - Windows
            *   `./gradlew assembleRelease` (Unsigned Release APK)
            *   `./gradlew bundleRelease` (Unsigned Release AAB / App Bundle)
            *   如果在 `build.gradle` 中配置了 `signingConfigs` (如上所示)，并且构建类型设置为使用该配置 (`signingConfig signingConfigs.release`)，那么 `assembleRelease` 和 `bundleRelease` **会输出带签名的包**！

---

## **三、两种方式的核心对比**

| 特性                          | Unity 直接打包 (Build APK)                                          | 导出项目到 Android Studio 再打包 (Export Project)                                                                           |
|:----------------------------|:----------------------------------------------------------------|:--------------------------------------------------------------------------------------------------------------------|
| **主要目的**                    | 快速生成最终 APK，无需额外工具链知识，适合纯 Unity 内容项目                             | 需要深度定制原生层 (Java/Kotlin/C++)、修改 AGP 配置、添加原生插件、构建多风味 App Bundle、集成原生服务或使用 Android Studio 强大的调试分析工具。                   |
| **构建执行者**                   | Unity Editor (调用内部或系统 Gradle/JDK/SDK)                           | 开发者使用 Android Studio (或命令行) 调用项目中的 Gradle Wrapper (使用本地或缓存 Gradle)                                                  |
| **Gradle 控制权**              | Unity 控制 AGP 版本、Gradle 版本。开发者有限配置。                              | **开发者完全控制** AGP 版本（可通过 Gradle 脚本指定，但要与Unity兼容）、Gradle 版本（通过 Wrapper 文件）。可以在 `build.gradle` 中精细配置几乎所有原生构建选项，添加插件和任务。 |
| **签名管理**                    | 通常在 Unity Player Settings 中配置 Keystore。                         | **推荐在模块 `build.gradle` 中配置**，更灵活、更符合原生开发流程和安全实践。                                                                    |
| **代码/资源混淆**                 | 在 Unity Player Settings 中配置 `Minify`（开启/关闭）。                    | 在 AS 模块 `build.gradle` 中配置 `minifyEnabled` 和自定义 ProGuard/R8 规则，控制粒度更细。                                              |
| **原生代码集成**                  | 相对麻烦，只能添加预编译好的 .jar/.aar/.so 插件。需处理JNI接口。                       | **最强大之处！** 可以直接在 AS 项目中添加、修改和调试 Java/Kotlin/C++ 源代码，方便实现复杂原生功能或深度集成第三方SDK。可使用 AS 的原生调试器。                            |
| **App Bundle 生成**           | 支持（通过 `Build System = Gradle` 和勾选 `Google Android App Bundle`）。 | **更自然、功能更全的支持。** 在 Android Studio 中构建 Bundle 是标准流程，可以方便地配置 Dynamic Feature Modules。                                 |
| **多风味构建**                   | 支持有限。                                                           | **原生支持。** 在 AS 中可以方便地为不同渠道、支付方式、A/B测试等配置风味（Flavors）及其组合（Build Variants）。                                            |
| **项目结构**                    | 临时生成，构建完即丢弃或覆盖。无持久性。                                            | 导出为完整、持久的 Android Gradle 项目目录。可以被版本控制、重复构建、深度修改。                                                                    |
| **构建速度**                    | 相对较快（流程集成度高，无需手动同步）。                                            | **首次构建/同步可能慢（需下载Gradle/AGP/依赖），** 后续增量构建速度正常。                                                                       |
| **适用场景**                    | 90% 以上的标准 Unity Android 项目，无特殊原生开发需求。                           | 需要原生功能扩展（如复杂推送、支付、ARCore深度集成）、自定义AGP配置、集成C++跨平台库、发布到不同渠道需不同配置、生成 App Bundle 或调试原生层问题。                               |
| **`Gradle (Installed)` 角色** | **通常不勾选**（用Unity内置）。勾选则用系统Gradle，需版本严格兼容。                       | **不适用**（这个选项只在 Unity 直接调用 Gradle 构建时有效）。导出后，构建由导出的项目自身（`gradlew` 或 AS）管理。                                           |

**共同点和关键配置：**

*   **Unity Player Settings 是基础：** 无论哪种方式，包名 (`Package Name`)、应用名、图标、启动画面、设备权限、目标 API Level (`Target SDK`, `Minimum SDK`)、脚本后端 (IL2CPP/Mono)、目标架构 (ARMv7, ARM64)、Bundle Version Code 这些核心设置**都必须在 Unity 的 Player Settings 中配置好**。这些设置直接影响 Unity 代码的编译和资源处理阶段。
*   **Keystore 极其重要：** 对于正式发布 (Release) **都必须**使用自定义 Keystore 签名。丢失 Keystore 密码或 Alias 密码意味着无法更新该签名的应用。
*   **JDK / Android SDK / NDK 需要：** 两者都依赖这些工具链，虽然管理方式不同（Unity内自动管理 vs 导出项目通过路径指向或AS管理）。
*   **版本兼容性是噩梦之源:** Unity版本 <-> JDK版本 <-> Android SDK/Build-Tools 版本 <-> Gradle 版本 <-> AGP 版本 <-> NDK 版本 (如果是IL2CPP) **必须严格兼容**！Unity Hub 自动安装配套版本是最好的避免方案。导出项目时，AS下载的Gradle和AGP版本由Unity生成的Wrapper文件控制（通常是兼容的），但如果AS强制升级AGP或修改Gradle版本，很容易导致失败。
*   **资源处理核心相同：** Unity 在两种方式下都需要执行关键的资源转换和优化过程（纹理压缩等）。

**新手建议：**

1.  **优先使用 Unity 直接打包 (Build APK)！** 它简单、快捷、Unity帮管理大部分复杂性（Gradle、签名配置）。对于大多数游戏项目，这是首选。
2.  当不需要做原生开发，但又需要打正式包上架时：
    *   在 Unity Player Settings (`Publishing Settings`) 中**务必配置好 Keystore** (创建并记住密码!)。可以使用界面下方的 `Keystore Manager`。
    *   使用 `Build System = Gradle`。
    *   确保 `Export Project` **没有勾选**。
    *   点击 `Build` 生成签名好的 `release.apk`。
3.  **只有在明确需要以下功能时，才选择导出项目到 Android Studio：**
    *   需要在 Android Studio 里写自定义的原生 Java/Kotlin/C++ 代码（不仅仅是引用预编译好的库）。
    *   需要精确控制 Gradle 构建流程（比如添加特定插件、自定义 flavor 维度）。
    *   需要生成 App Bundle (虽然Unity直接也能做，但AS更灵活)。
    *   需要深度调试或修改 Unity 生成的 AndroidManifest.xml 或 Gradle 配置。
    *   项目结构复杂，需要在AS中进行原生集成开发。

**环境问题排查提示 (当Build失败时)：**

*   **仔细阅读错误信息！** Unity Console 或 Android Studio 的 `Build` 输出窗口会给出明确的错误栈。
*   **检查版本兼容性：** 最常见的错误来源！JDK? SDK Build-Tools? NDK? (看错误提示需要的版本) VS Unity 支持的版本 (`Edit -> Preferences -> External Tools` 有推荐范围，Unity 版本发行说明有详细要求)。
*   **检查 Keystore:** 路径、密码、Alias、密码是否正确？Release 包必须签名。
*   **看日志文件:**
    *   Unity 直接打包失败：查看 `Editor.log` (位置各OS不同，Unity菜单 `Help` 下有找Log的选项) 或 Console 中的详细错误。
    *   Export Project + AS 构建失败：查看 AS 底部的 `Build` 输出标签页，里面有详细的 Gradle 任务执行日志。`Gradle Console` 视图更原始。

**重要区别总结：**

*   用 `Export Project` 选项导出到 Android Studio 的核心目的是 **获得对 Android 原生层（Java/Kotlin/C++ 代码和 Android 构建配置）的完全控制权和开发环境**。
*   直接 `Build APK` 则是一种在 Unity 内部完成的、封装的、“傻瓜式”的构建方式，牺牲灵活性换取便捷性。