<img width="800" height="480" alt="11111" src="https://github.com/user-attachments/assets/5469d5ac-ca85-4619-8fd3-6577e1117d90" />



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

## 一键安装做什么

后台安装固定版本 **Shairport Sync 5.5.2 + NQPTP 1.2.8**，核对 Git 提交后，以低权限构建账户编译。安装构建依赖时不会运行全系统 `apt upgrade`，也不允许 APT 删除已有软件包。

- 程序目录：`/opt/airplay-control-receiver/`。
- 服务：`shairport-sync.service` 和 `nqptp.service`，安装成功后启用开机启动。
- 新配置按所选设备支持的双声道 PCM 格式与 44.1/48 kHz 组合生成；设备不支持或被占用时明确报错，不强行重采样。
- 首次生成配置默认关闭音量处理和 DSP，保留本项目的连续数字静音方案。
- 已有配置完整保留；已有自定义 systemd drop-in 或符号链接配置需要人工确认，不自动覆盖。
- 安装失败会恢复提交前的接收器配置、程序和服务单元；已安装的系统依赖和构建缓存会保留。
- 安装在 systemd 后台执行，关闭窗口或退出终端进度显示不会中止。失败可查看日志后再次安装。

安装日志：`/var/lib/airplay-control/install.log`；阶段状态：`/var/lib/airplay-control/install.json`；接收器安装备份：`/var/backups/airplay-control/receiver-<时间>/`。

不自动修改防火墙或停止 PipeWire 等其他音频服务。USB 声卡若被其他程序独占，先释放设备后重试。已有较旧的 Shairport Sync 配置可能与 5.5.2 不兼容，启动失败时保留原配置并显示日志。

## 纯终端模式

SSH、Raspberry Pi OS Lite 或无显示器环境可使用全部配置功能。`airplay-control` 在没有 DISPLAY / WAYLAND_DISPLAY 时默认打印当前状态；有桌面时打开图形界面。

```bash
# 显示配置、服务、当前输出状态及设备信息
airplay-control status
airplay-control status --watch

# 枚举真实 ALSA 播放硬件及采样率能力
airplay-control devices

# 一键安装（多设备时使用 devices 显示的 ID）
airplay-control install --device 'hw:CARD=Audio,DEV=0' --name '客厅音响' --wait

# 修改名称 / 音量控制 / 音频输出，并立即应用
airplay-control set --name '书房音响'
airplay-control set --volume off
airplay-control set --volume on
airplay-control set --device 'hw:CARD=USB,DEV=0'
airplay-control set --name '客厅' --volume off --device 'hw:CARD=Audio,DEV=0'

# 启停接收器与诊断
airplay-control start
airplay-control stop
airplay-control restart
airplay-control logs

# 交互式控制台，进入后输入 help 或上述子命令，exit 退出
airplay-control console

# JSON 输出 / 强制图形模式 / 演示模式
airplay-control --json status
airplay-control --json devices
airplay-control gui
airplay-control --demo
airplay-control --demo-setup
```

上面的 `hw:CARD=...` 是示例，请以当前设备列表为准。只有一个播放设备时，`install` 可以省略 `--device`。`--wait` 中按 Ctrl+C 只退出进度显示。终端中的写入、安装、启停操作会请求 sudo 授权；不保存密码。`stop` 停止接收器，NQPTP 时钟服务保留运行。

仅安装终端版（含 32 位系统）：解压源码包，进入项目目录后执行：

```bash
sudo bash scripts/install.sh --cli-only
airplay-control console
```

该模式在本机编译小型 ALSA/libconfig 辅助程序，不编译 Flutter、不安装图形依赖。

## 设备采样率显示

设备列表查询 ALSA 硬件双声道能力，显示 8–768 kHz 范围内常见支持值，并非承诺枚举所有整数采样率。ALSA 的采样率上下限保留在 JSON 输出中；范围中不一定每个采样率都可用。

正在播放的 USB 设备不能直接打开时，使用内核 USB 播放描述符作为参考，并标注「USB 描述符」。不会混入录音采样率。无可用描述符时显示「待确认」，不会停止播放来查询。具体采样率与样本格式组合在应用或首次安装时验证，查询不会发送测试音。

顶部显示的是当前硬件运行采样率，可能是连续数字静音，不等同于正在播放音乐。关闭音量控制表示忽略手机音量指令，不等同于整个 AirPlay 链路逐位一致。

## 800×480 显示

使用紧凑的两栏布局，名称、音量和输出选择在同一屏；设备较多时只滚动设备列表，底部操作栏固定。小屏幕启动时最大化，保留桌面窗口关闭能力。更窄窗口采用上下排列。界面截图位于 `screenshots/preview-800x480.png` 和 `screenshots/setup-800x480.png`。

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
