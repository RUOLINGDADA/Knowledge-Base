---
分类: 技术/嵌入式/树莓派
层级: 理解层
类型: 资料速查型
主题: [Raspberry Pi 5, EEPROM, USB boot, usbboot, rpiboot, 恢复]
技术栈: [raspberry-pi, linux, usb, eeprom, bootloader]
tags: [raspberry-pi, eeprom, usbboot, rpiboot, bootloader, linux, 理解层]
状态: 待验证
创建日期: 2026-09-10
来源: 用户命令笔记
---

# [技术/嵌入式/树莓派] 树莓派 5 EEPROM 的 USB 恢复与 `usbboot` 使用

> 适用目标：当 Raspberry Pi 5 无法从预期介质启动、需要更新或恢复 SPI EEPROM 中的 bootloader 时，用一台运行 Ubuntu 的主机和 USB 连接使主机通过 `rpiboot` 与树莓派通信。
>
> 本文基于用户已记录的命令整理；尚未在当前环境独立执行。`pieeprom.*` 文件的具体来源、版本及其是否适配当前硬件均需在现场确认。

## 先理解要操作的对象

Raspberry Pi 5 的启动固件位于板载 SPI EEPROM，不在 microSD 卡或 USB 系统盘中。启动时，EEPROM 中的 bootloader 按其配置寻找可启动介质；因此修复 EEPROM 是在恢复启动链路，而不是重装操作系统。

`usbboot` 项目提供的 `rpiboot` 工具可让电脑端通过 USB 与处于恢复模式的树莓派通信。`recovery5/` 中的脚本会根据 `boot.conf` 和 EEPROM 二进制文件生成待写入镜像，再由 `rpiboot` 将恢复文件发送给 Pi 5 执行。

```text
Ubuntu 主机 -- USB 数据线 --> Raspberry Pi 5（恢复模式）
       |                               |
       +-- rpiboot 发送恢复文件 --------+
                                       SPI EEPROM
```

## 风险与前置条件

- 此流程面向 **Raspberry Pi 5**。不要将 `recovery5` 或 Pi 5 的 EEPROM 镜像用于其他型号。
- `eeprom-erase/` 会擦除 EEPROM。官方将其标为测试/调试选项，并明确说明通常**不必先擦除再刷写**；擦除后 Pi 不会启动，直到通过 RPIBOOT 成功写入新镜像。默认不要执行它。
- 使用能传输数据的 USB 线；仅充电线无法枚举 USB 设备。树莓派和主机应使用可靠供电，恢复期间不要断电或拔线。
- 先确认 Raspberry Pi 官方文档对当前硬件、镜像版本和 USB 端口的要求。不同硬件版本或工具版本的恢复模式接线、按键/跳线要求可能变化。
- 操作前移除树莓派上的其他可启动介质，并记录当前 EEPROM 版本与配置（若设备还能启动）。这可避免启动到错误介质，也便于回退。
- 下文的 `/dev/sdb1` 是示例。必须通过 `lsblk`、容量和标签确认实际分区；选错设备可能读错镜像或误挂载其他磁盘。

## 一次完整恢复流程

### 1. 在 Ubuntu 主机准备 `usbboot`

在 Ubuntu 安装界面可用 `Ctrl` + `Shift` + `T` 打开终端。用户笔记提示：Windows 环境直接执行该流程可能出现问题，因此本流程以 Ubuntu 为主机环境记录。

```bash
sudo apt update
sudo apt install -y git libusb-1.0-0-dev pkg-config build-essential
git clone --recurse-submodules --shallow-submodules --depth=1 https://github.com/raspberrypi/usbboot
cd usbboot
make
```

各命令的作用：

| 命令或依赖 | 作用 | 验证方式 |
| --- | --- | --- |
| `apt update` | 刷新 APT 软件包索引 | 无明显错误退出 |
| `git` | 获取 `usbboot` 源码 | `git --version` |
| `libusb-1.0-0-dev` | 提供 `rpiboot` 访问 USB 所需的开发库 | 安装成功 |
| `pkg-config` | 让构建过程定位 `libusb` 的编译参数 | `pkg-config --modversion libusb-1.0` |
| `build-essential` | 提供 `make`、GCC 等编译工具 | `make --version` |
| `git clone --depth=1` | 浅克隆，减少下载量 | 出现 `usbboot/` 目录 |
| `make` | 编译工具 | `./rpiboot --help` 可运行 |

用户原始笔记还包含以下 Git URL 重写规则：

```bash
git config --global url."https://gh-proxy.com/https://github.com/".insteadOf "https://github.com/"
```

它会把该用户后续所有 `https://github.com/` Git 请求改为经 `gh-proxy.com` 访问。仅在网络环境确有需要、且信任该代理时使用；它不是 `usbboot` 的必需步骤。临时网络问题优先检查连接或改用可信镜像。若需撤销：

```bash
git config --global --unset url."https://gh-proxy.com/https://github.com/".insteadOf
```

### 2. 准备要写入的 EEPROM 镜像

当前 `usbboot` 的官方 `recovery5/` 目录默认包含适用于 Pi 5 的稳定版 EEPROM 二进制文件。先按目录内脚本生成恢复镜像；需要修改启动顺序等配置时，编辑 `boot.conf` 后再执行脚本。

```bash
cd ~/usbboot/recovery5
./update-pieeprom.sh
ls -lh pieeprom.bin bootcode5.bin
```

- `update-pieeprom.sh` 根据当前目录中的 `boot.conf`、`pieeprom.original.bin` 等文件生成 `pieeprom.bin`。
- `bootcode5.bin` 实际是 Pi 5 的 `recovery.bin`，由 RPIBOOT 流程加载。
- 镜像来源和版本应可追溯。若使用自定义 `pieeprom.original.bin`，必须确认它适用于 Raspberry Pi 5，并在操作前保存来源链接或校验和。

用户原始笔记采用了“用 Raspberry Pi Imager 将 EEPROM 恢复镜像烧录到另一只 USB 存储设备，再从中复制 `pieeprom.*` 到 `recovery5/`”的方式。此方式不是当前 `usbboot` README 的默认流程，且仅复制文件后可能遗漏生成镜像所需的配置或脚本步骤。必须沿用该来源时，至少先确认真实分区和文件：

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb  # 用实际确认的分区名替换
find /mnt/usb -maxdepth 2 -type f -iname 'pieeprom.*'
```

`/dev/sdb1` 只是示例，必须用 `lsblk` 的容量、标签和文件系统确认。若找不到文件，或不清楚其是否应替换 `recovery5/` 中的哪个文件，停止操作并回到官方 `update-pieeprom.sh` 流程。完成检查后，用 `cd ~ && sudo umount /mnt/usb` 卸载该介质。

### 3. 让树莓派进入恢复模式并检查 USB 枚举

Raspberry Pi 5 当前官方步骤为：先断开 Pi 5 的 USB-C 线，确保电源已移除（仅执行 `sudo shutdown now` 不够）；按住电源按钮；保持按住的同时，用 USB-C 数据线把 RPIBOOT 主机连接到 Pi 5。不要只依赖端口外观判断，执行下列命令确认主机是否看到了新设备：

```bash
lsusb
dmesg -w
```

若 `rpiboot` 一直等待且 `lsusb` 没有新增设备，优先检查：USB 线是否支持数据、接入端口是否正确、恢复模式是否按官方要求设置、是否实际给 Pi 供电，以及是否被 USB 集线器/虚拟机 USB 透传限制。

### 4. 默认直接写入恢复镜像

设备必须已经按上一步进入恢复模式并连接。从 `recovery5` 目录执行；每个阶段完成后的 LED 状态和终端输出以当前 `usbboot` 版本及官方说明为准。

```bash
cd ~/usbboot/recovery5
sudo ../rpiboot -d .
```

`rpiboot -d .` 将当前 `recovery5` 目录作为提供给目标板的文件目录。工具完成且未报告 USB 传输错误，只说明主机端传输完成；仍须在断电重启后验证实际启动结果。

若 Pi 退出恢复模式、USB 设备消失或命令超时，应重新断电，并按官方方式重新进入恢复模式后重试。不要把“主机不再枚举设备”直接当作修复成功，应完成恢复镜像写入并实际验证启动。

> [!warning] 不要把擦除当作常规步骤
> 用户原始命令中的 `sudo ./rpiboot -d eeprom-erase` 可执行 EEPROM 芯片擦除，但官方说明指出刷写前没有必要手动擦除。只有在明确的测试/调试要求、已准备可用恢复镜像且理解后果时才考虑它；若确实执行，随后必须重新进入 RPIBOOT 模式并完成本节的 `recovery5` 写入流程。

### 5. 断电重启并验证结果

1. 等待 `rpiboot` 命令完成，确认没有 USB、权限或找不到文件的错误。
2. 关闭目标板电源，不要仅重启 Ubuntu 主机。
3. 移除恢复模式所需的跳线、按钮状态或 USB 连接，恢复正常启动接线。
4. 接入已知可启动的系统介质，通电观察启动行为。
5. 若可启动 Raspberry Pi OS，查询 EEPROM 状态和版本：

```bash
vcgencmd bootloader_version
sudo rpi-eeprom-update
```

`vcgencmd` 和 `rpi-eeprom-update` 需要系统已启动且安装了对应 Raspberry Pi 工具；命令不存在不等于 EEPROM 恢复失败。最终的成功标准是：设备以预期的启动顺序稳定启动，且系统中查询到的 bootloader 版本与所使用镜像相符。

## 原始命令与改进点对照

| 原始记录 | 整理后的处理 | 原因 |
| --- | --- | --- |
| `git submodule update --init --depth=1` 在 `cd usbboot` 之前 | 改为 `git clone --recurse-submodules --shallow-submodules --depth=1` | 原命令不在仓库目录中执行不会初始化该仓库的子模块；当前上游推荐递归浅克隆 |
| `cp /mnt/usb/pieeprom.* ~/usbboot/recovery5/` | 改为优先运行 `recovery5/update-pieeprom.sh` | 当前上游流程由脚本按 `boot.conf` 和原始二进制生成镜像，手动复制可能遗漏必要上下文 |
| 直接执行 `rpiboot` | 在前面补充恢复模式、线缆和 USB 枚举检查 | 工具卡住时，最常见原因是主机根本未识别目标板 |
| 先执行 `eeprom-erase` 再恢复 | 改为默认直接写入 `recovery5` | 官方明确表示刷写前无需手动擦除；擦除会增加设备无法启动的时间和失败风险 |

## 常见故障速查

| 现象 | 常见原因 | 优先动作 |
| --- | --- | --- |
| `make` 找不到 `libusb` 头文件或链接失败 | 未安装开发包，或包索引过期 | 重新执行 `sudo apt update` 与依赖安装命令 |
| `rpiboot` 报权限不足 | 当前用户无 USB 设备访问权限 | 仅对该命令使用 `sudo`，不要整段终端长期以 root 运行 |
| `rpiboot` 持续等待设备 | 不是数据线、端口/恢复模式错误、目标板未供电 | 用 `lsusb`/`dmesg -w` 验证主机 USB 枚举 |
| `update-pieeprom.sh` 失败或找不到 `pieeprom.bin` | 子模块未初始化、脚本执行目录错误，或恢复文件不完整 | 在 `usbboot/recovery5` 目录执行脚本；检查克隆时已包含子模块 |
| 写入完成仍无法启动 | 镜像不匹配、启动介质故障、恢复流程未实际完成或启动配置不符 | 用已知可用介质复测；可启动后查询 bootloader 版本；再核对官方恢复文档 |
| GitHub 克隆失败 | 网络/DNS/代理限制 | 先验证网络；只有信任代理且确有需要时再配置 Git URL 重写 |

## 使用边界与资料入口

- 本文记录的是主机端通过 USB 恢复 Pi 5 EEPROM 的路径，不覆盖 SD 卡系统镜像烧录、普通启动顺序调整或硬件级故障维修。
- EEPROM 更新和完整擦除的风险不同。设备还能正常启动时，优先通过 Raspberry Pi OS 的 `rpi-eeprom-update` 和官方建议的更新路径处理；`eeprom-erase` 是测试/调试选项，非正常更新或恢复的前置步骤。
- `rpiboot` 的目录结构、支持的硬件与恢复模式细节可能随版本变化。执行前应以 [raspberrypi/usbboot](https://github.com/raspberrypi/usbboot) 仓库的当前说明和 [Raspberry Pi 官方文档](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html) 为准。

## 相关笔记

- [[树莓派_专题索引]]
- [[Windows_树莓派网络共享_专题索引]]：树莓派正常启动后的 Windows 有线网络共享与 SSH 接入。
