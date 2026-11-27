# 双音频输出（Dual Audio 简体中文汉化版）

把 Android 的媒体声音**同时输出到 USB 和蓝牙（A2DP）**，例如同时用一个 USB 耳机和一副蓝牙耳机。
任何应用的声音（音乐、视频、游戏）都适用。

不需要 root：它借助 [Shizuku](https://shizuku.rikka.app/)（或 adb）以 Android shell 身份的权限运行。

> **本仓库是什么**
> - 上游原项目：[xTudoS/dual-audio](https://github.com/xTudoS/dual-audio)（作者 xTudoS）。
> - 本仓库是它的**简体中文汉化版**，基于上游提交 `46157b9`，只做界面与状态文本的汉化，
>   **音频路由逻辑没有任何改动**。
> - 许可证与上游保持一致：**GNU GPL v3.0 或更高版本**（GPL-3.0-or-later），见 [`LICENSE`](LICENSE)。
> - 英文原版说明见 [`README.en.md`](README.en.md)。
>
> **与上游的差异**
> 1. `app/src/main/res/values/strings.xml` 改为中文，并新增 `values-en/strings.xml` 作为英文回退；
> 2. `MainActivity.kt`、`AudioRouter.java`、`AudioRoutingUserService.java` 里给用户看的提示文本汉化；
> 3. `activity_main.xml` 中按钮加上 `android:textAllCaps="false"`，避免中文按钮被强制大写；
> 4. `gradle.properties` 中 Termux 构建所需的三行配置改为注释形式（在手机上编译时手动打开）。
>
> `tools/audio-dup` 命令行工具的终端输出仍是英文（未改）。
> 若发现汉化有误，欢迎提 issue 或 PR。

## 工作原理

Android 12 引入了[组合音频设备路由](https://source.android.com/docs/core/audio/combined-audio-routing)：
`setPreferredDevicesForStrategy()` 允许对同一条策略指定**多个**设备。Dual Audio 把媒体策略设为
`[USB, BLUETOOTH_A2DP]`，`AudioPolicyManager` 就会切换到复制输出（`DuplicatingThread`），
把混音同时送到两个设备。

该 API 属于 `@SystemApi`，需要 `MODIFY_AUDIO_ROUTING` 权限，普通应用拿不到；而 Android 的
**shell 用户（uid 2000）**拥有该权限，所以路由代码以 shell 身份运行，有两种方式：

| 版本 | 以 shell 身份运行的方式 | 需要电脑吗 |
|---|---|---|
| **App**（推荐） | [Shizuku](https://shizuku.rikka.app/) | 不需要，只要无线调试 |
| **adb CLI** | `adb shell app_process` | 需要，每次启用都要 |

## 环境要求

- **App：** Android 14 及以上。**adb CLI：** Android 12 及以上。
- 一个 USB 音频设备和一个蓝牙 A2DP 耳机，且同时连接。
- 已开启开发者选项（App 需要无线调试，adb CLI 需要 USB 或无线调试）。

## 安装应用

本汉化版仓库不提供官方签名的 APK，请自行编译（见[从源码构建](#从源码构建)），
或使用上游的 [Releases](https://github.com/xTudoS/dual-audio/releases) 页面安装原版后再对照汉化。

1. 把编译出的 `app-debug.apk`（或你自行签名的 release APK）传到手机上打开安装，
   Android 询问时允许浏览器/文件管理器「安装未知应用」。
2. 安装 **Shizuku**：[Play 商店](https://play.google.com/store/apps/details?id=moe.shizuku.privileged.api)
   或 [GitHub](https://github.com/RikkaApps/Shizuku/releases)。

### 每次重启后启动 Shizuku

1. 手机连上 Wi-Fi。无线调试只能在 Wi-Fi 网络下使用。
2. *开发者选项 → 无线调试*：打开。
3. 打开 Shizuku → **通过无线调试启动**。首次需点「配对」，把
   *使用配对码配对设备* 里的配对码填进 Shizuku 的通知里。

Shizuku 会一直运行到下次重启。连着电脑时，也可以用下面的命令启动：

```bash
adb shell "$(adb shell pm path moe.shizuku.privileged.api | sed 's/package://;s#/base.apk##' | tr -d '\r')/lib/arm64/libshizuku.so"
```

### 使用方法

1. 打开 **双音频输出** → **授予 Shizuku 权限** → *始终允许*。
2. 两个设备都连接后，点 **开启 USB + 蓝牙同时输出**。
3. **关闭（恢复普通音频路由）** 可关闭该功能。

开启期间，一个小后台服务（通过 Shizuku 启动）每 3 秒检查一次；当两个设备都已连接、但路由被重置
（例如耳机重连后）时，会**自动重新应用路由**。关掉 App 也依然有效，直到你点「关闭」、
Shizuku 停止或手机重启。

## adb 命令行（免安装 App）

想从电脑上快速试一下，可以不装 App。需要 Android SDK（`platforms`、`build-tools`、`platform-tools`）
和 JDK 17+：

```bash
cd tools/audio-dup
./run.sh status   # 已连接的输出设备与当前路由
./run.sh set      # 把媒体声音路由到 USB + 蓝牙
./run.sh clear    # 恢复正常路由
```

脚本会通过 `$ANDROID_HOME` 查找 SDK，macOS 上也会找 `~/Library/Android/sdk`。
与 App 不同，这里没有任何自动重放：设备断开或手机重启后，需要重新执行 `set`。

## 验证是否生效

```bash
adb shell dumpsys media.audio_policy | grep -A1 -E 'Device role per product|STRATEGY_MEDIA'
```

路由开启后，`STRATEGY_MEDIA` 选中的设备会同时包含 `AUDIO_DEVICE_OUT_BLUETOOTH_A2DP` 和
`AUDIO_DEVICE_OUT_USB_HEADSET`；有声音播放时，*Outputs* 中会出现一个包含这两个设备的复制输出
（duplicated output），并显示 `Global active count: 1`。

## 已知限制

- **不同步：** 蓝牙比 USB 慢约 150–300 ms，Android 在复制输出上不做补偿。
- **重采样：** 混音以蓝牙输出的格式运行，再转换成 USB 设备的格式。使用高解析蓝牙模式
  （如 24 bit / 96 kHz）时会多出一次重采样；全程都是未压缩 PCM。把
  *开发者选项 → 蓝牙音频采样率* 设为 48 kHz 可以避免。哪个输出是主输出由 Android 在蓝牙连接时
  决定，这个 API 改不了。
- **音量：** 每个设备有各自的音量曲线，音量键对两者的影响可能不同。
- **厂商音效**（如三星 Dolby Atmos、SoundAlive、360 Audio）在复制输出上的表现可能不同。
- **非官方方案：** 依赖 Android 隐藏 API，Android 或厂商更新后可能失效。
  上游在三星 One UI + Android 16 上测试通过。

## 从源码构建

```bash
./gradlew :app:installDebug
```

- debug 构建与 release APK 使用不同的签名密钥，Android 不允许互相覆盖安装；
  切换时先卸载，再重新授予 Shizuku 权限。
- **修改路由服务后务必在 `app/build.gradle.kts` 里递增 `versionCode`**：该服务是 Shizuku 守护进程，
  Shizuku 只在版本号变化时才替换它。
- GitHub Actions 会在每次推送时构建（`.github/workflows/build.yml`），并为 `v*` 标签发布签名 APK
  （`.github/workflows/release.yml`）。

### 在手机上的 Termux 里构建（aarch64）

```bash
pkg install openjdk-21 aapt2 gradle
# 打开 gradle.properties 底部被注释的三行 Termux 配置，再执行：
./gradlew :app:assembleDebug
```

### 项目结构

```
app/src/main/
  aidl/com/byxnd/dualaudio/routing/IAudioRoutingService.aidl   App ↔ shell 服务的接口约定
  java/com/byxnd/dualaudio/
    MainActivity.kt                         界面；连接 Shizuku 与路由服务
    routing/AudioRouter.java                路由逻辑（与 CLI 共用）
    routing/AudioRoutingUserService.java    Shizuku 服务（以 shell 运行，自动重放）
tools/audio-dup/
  src/Poc.java                              CLI：status | set | clear
  run.sh                                    用 javac/d8 构建并通过 app_process 运行
```

## 许可证

Copyright (C) 2026 xTudoS
汉化部分（本仓库新增/修改的文本与文档）同样以 GPL-3.0-or-later 发布。

本项目以 **GNU General Public License v3.0 或更高版本**（`GPL-3.0-or-later`）发布，
与上游原项目一致；全文见 [`LICENSE`](LICENSE)。简单来说：

- 你可以使用、研究、fork 和修改这份代码。
- 如果你**分发**它或修改后的版本（APK、应用商店里的分支等），必须以同样的许可证分发，
  并提供对应的源代码。
- 不提供任何担保。

App 中打包的第三方库（MIT 与 Apache 2.0，均与 GPL-3.0 兼容）列在
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md)，App 内「开源许可」页面会展示这两个文件。
