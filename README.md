# AirPlay 控制台

为 Raspberry Pi 开发的 Flutter Linux 桌面配置程序，界面为中文。

- 修改 AirPlay 接收器名称。
- 开启或关闭手机音量控制。关闭时对应 `ignore_volume_control = "yes"`。
- 枚举本机 ALSA 播放硬件，选择 USB 声卡、HDMI 或内置输出。
- 显示接收器、NQPTP 时钟服务以及当前硬件采样率状态，每 5 秒刷新。
- 应用前备份；设备占用、格式不兼容或服务启动失败时恢复原配置。

## 适用环境

64 位 Raspberry Pi OS / Debian 13 桌面，已有支持 AirPlay 2 的 Shairport Sync 和 NQPTP。程序管理 `/etc/shairport-sync.conf`，服务名为 `shairport-sync.service` 和 `nqptp.service`。服务必须实际使用这个配置文件，且不能用命令行参数覆盖这三个设置。

本程序是配置界面，不包含 AirPlay 接收引擎或自动安装脚本。原 192.168.0.15 上已经配置的 Shairport Sync 5.5.2 是目标环境。没有接收引擎的电脑可以使用 `--demo` 演示界面。

## 运行安装包

树莓派使用 **arm64** 安装包；WSL / Intel、AMD Linux 电脑使用 **x64** 安装包。不能互换。

```bash
sudo apt-get install python3 libconfig11 libasound2t64 libgtk-3-0t64 liblzma5 libstdc++6 pkexec polkitd fonts-noto-cjk
# 解压对应架构的安装包后进入目录
sudo bash scripts/install.sh
airplay-control
```

从桌面菜单也可打开「AirPlay 控制台」。应用设置时，桌面的 Polkit 授权窗口会请求管理员认证。**界面以普通用户运行，不要 sudo 启动 Flutter 界面**。桌面需要正在运行的 Polkit 认证代理；Raspberry Pi OS 桌面通常已经配置。

安装过程不会修改 AirPlay 配置或重启接收器。若运行用户不能枚举声卡，请检查 `/dev/snd` 权限和 `audio` 组；更改组后重新登录。避免同时让其他播放器或 PipeWire 独占所选硬件。

```bash
# 无硬件演示；名称、音量和设备修改只保存在本次进程内存中
airplay-control --demo
# 只读诊断
/usr/local/lib/airplay-control/airplay-helper status
```

## WSL 开发与编译

可以使用 WSL2 + WSLg / X11 开发、热重载和预览 Linux 界面。WSL 的 Flutter SDK 必须与 Windows SDK 分开，避免 Linux/Windows 缓存互相覆盖。当前开发机使用：

```bash
export PATH="$HOME/tools/flutter-3.41.6/bin:$PATH"
cd /mnt/c/Users/Gary/Desktop/raspiaudio/airplay_control
flutter pub get
flutter run -d linux -- --demo
```

从源码构建，先在对应 Linux 系统安装：

```bash
sudo apt-get install clang cmake ninja-build pkg-config libgtk-3-dev liblzma-dev libstdc++-14-dev libconfig-dev libasound2-dev fonts-noto-cjk
bash scripts/build.sh
```

脚本运行后端测试、Flutter 静态检查和交互测试，编译 release，并生成 `dist/airplay-control-linux-<架构>.tar.gz`。Flutter SDK 需要 Dart >= 3.11，开发时使用 Flutter 3.41.6。

**x64 WSL 的普通 `flutter build linux` 产物不能直接在 ARM64 树莓派运行。** 本项目在树莓派本机编译 ARM64 版本；可将源码同步过去运行同一构建脚本。无显示器仍可编译，运行 GUI 时需要桌面会话。

## 音频行为

程序只改三项：`general.name`、`general.ignore_volume_control`、`alsa.output_device`。原有采样率列表、格式、同步策略、DSP、数字静音保活等配置值均保留。libconfig 会重新排版配置并移除注释，备份保留原始文件的全部内容。

- 关闭音量控制只表示忽略音量指令，并不自动关闭现有 DSP，也不能保证整个 AirPlay 链路逐位一致。
- 不增加音量滑块、重采样或 DSP；原项目里的 44.1/48 kHz 和数字静音修复继续由接收器负责。
- 当前硬件采样率来自 ALSA `hw_params`。连续静音也会令设备处于工作状态，因此它不等于「正在播放音乐」。
- 切换设备时验证现有双声道格式和全部配置采样率，不发送测试音。如果设备不支持，则提示错误并恢复。
- 输出格式为 `auto` 时兼容性探测使用 S16_LE；未配置采样率时探测 44.1/48 kHz。正式接收引擎仍按原配置工作。
- 保存会短暂中断播放，随后在手机重新选择接收器。热插拔只刷新设备列表，不会自动切换输出。

## 备份与恢复

备份目录：`/var/backups/airplay-control/`，每份文件仅 root 可读。应用操作有互斥锁，并检测配置是否被其他程序修改，避免旧界面覆盖新设置。遇到版本冲突可先记下草稿，再点「撤销修改」载入刷新后的设置。

手动恢复某份备份：

```bash
sudo cp /var/backups/airplay-control/shairport-sync.<时间>.conf /etc/shairport-sync.conf
sudo systemctl restart shairport-sync.service
```

启动检查覆盖 Shairport Sync 和 NQPTP 在应用后的短时间内保持运行；不能替代真实手机连接、暂停/恢复、切歌和 USB 输出测试。严重设备或系统故障可能使旧服务也无法恢复，界面会显示恢复失败和备份路径。

## 实现与测试

`lib/` 是 Flutter 界面，`backend/airplay_helper.py` 负责本机读写事务，`backend/config_bridge.c` 使用 libconfig 和 ALSA 解析配置与枚举设备。没有网络监听端口，也不保存 SSH 或管理员密码。读取以普通用户执行，写入通过限定到此辅助程序的 Polkit 策略授权。

```bash
python3 -m unittest discover -s backend/tests -v
flutter analyze
flutter test
# 生成真实 Flutter 组件的界面预览（需要 Noto CJK 字体）
flutter test tool/capture_preview.dart
```

参考：[Flutter Linux 开发环境](https://docs.flutter.dev/platform-integration/linux/setup)、[Flutter 支持平台](https://docs.flutter.dev/reference/supported-platforms)、[Shairport Sync](https://github.com/mikebrady/shairport-sync)。
