---
分类: 资料速查/网络
层级: 理解层
类型: 资料速查型
主题: [IP 地址, 网卡, 路由, ARP, DNS, Windows, Raspberry Pi]
技术栈: [windows, powershell, linux, iproute2, raspberry-pi]
tags: [ip地址, 网卡, 路由, arp, dns, windows, linux, 资料速查, 理解层]
状态: 待验证
创建日期: 2026-09-08
---

# [资料速查/网络] Windows 与 Linux IP 地址查询命令

## 使用目标

排查“Windows 通过网线给树莓派共享网络”时，按这个顺序查：

```text
接口状态 → 本机地址 → 邻居/ARP → 路由 → 网关连通 → DNS → 外网路径
```

## Windows 11：系统自带命令

以下命令通常随 Windows 11 自带，不需要额外安装。

| 目的 | 命令 | 重点看什么 |
| --- | --- | --- |
| 查看所有接口地址 | `ipconfig` | 以太网 IPv4、子网掩码、默认网关 |
| 查看完整 DHCP/DNS 信息 | `ipconfig /all` | DHCP、DNS、物理地址、租约 |
| PowerShell 查看接口配置 | `Get-NetIPConfiguration` | IPv4、网关、DNS、接口别名 |
| 查询 IPv4/IPv6 地址 | `Get-NetIPAddress` | `IPAddress`、`InterfaceAlias`、前缀长度 |
| 查询路由表 | `Get-NetRoute` 或 `route print` | `0.0.0.0/0` 默认路由及接口 |
| 查看网卡状态 | `Get-NetAdapter` | `Up/Down`、物理网卡还是虚拟网卡 |
| 查看局域网邻居 | `Get-NetNeighbor` 或 `arp -a` | 树莓派 IP 与 MAC 是否出现 |
| 测试网关/外网 | `ping 192.168.137.1` | 是否能到 Windows 下游接口 |
| 测试端口 22 | `Test-NetConnection <树莓派IP> -Port 22` | `TcpTestSucceeded` |
| 测试 DNS | `nslookup www.baidu.com` | DNS 服务器和解析结果 |
| 查看路径 | `tracert 8.8.8.8` | 流量经过哪些网关 |

常用组合：

```powershell
ipconfig /all
Get-NetAdapter
Get-NetRoute -AddressFamily IPv4
Get-NetNeighbor -AddressFamily IPv4
Test-NetConnection <树莓派IP> -Port 22
```

## Raspberry Pi：通常自带或默认预装

Raspberry Pi OS 基于 Debian。最小系统和精简镜像的预装内容可能不同，以下命令优先使用。

| 命令 | 常见来源 | 用途 |
| --- | --- | --- |
| `ip -br addr` | `iproute2`，Raspberry Pi OS 通常已有 | 简洁查看接口和地址 |
| `ip addr` | `iproute2` | 查看详细 IPv4/IPv6、前缀和接口状态 |
| `ip route` | `iproute2` | 查看默认网关和路由 |
| `ip neigh` | `iproute2` | 查看 ARP/邻居缓存 |
| `hostname -I` | `hostname`，通常已有 | 快速输出本机 IP |
| `ping -c 4 <地址>` | `iputils-ping`，通常已有 | 测试连通性 |
| `cat /etc/resolv.conf` | `cat`，通常已有 | 查看当前 DNS 配置 |
| `resolvectl status` | systemd-resolved 环境可用 | 查看 DNS 状态，非所有系统启用 |
| `nmcli device status` | NetworkManager 安装时可用 | 查看设备、连接和管理状态 |

树莓派网络共享的最小检查：

```bash
ip -br addr
ip route
ping -c 4 192.168.137.1
ping -c 4 8.8.8.8
```

## 需要安装的软件包

下面是 Raspberry Pi OS/Debian 常见包名；安装前可先用 `command -v <命令>` 判断。不同发行版的包名可能不同。

| 命令 | Debian/Raspberry Pi OS 包 | 用途 |
| --- | --- | --- |
| `ifconfig`、`arp` | `net-tools` | 传统接口和 ARP 查询；新系统优先用 `ip` |
| `dig`、`host` | `dnsutils` | 详细 DNS 查询 |
| `traceroute` | `traceroute` | 查看三层路径 |
| `ethtool` | `ethtool` | 查看网卡链路、速率、双工和驱动 |
| `nmap` | `nmap` | 扫描局域网主机或检查端口，需谨慎使用 |

安装示例：

```bash
sudo apt update
sudo apt install dnsutils traceroute ethtool
```

只有确实需要兼容旧教程时再安装 `net-tools`；日常地址和路由查询使用 `ip` 更推荐。

## 命令输出的快速解释

| 观察 | 结论或下一步 |
| --- | --- |
| 有线接口显示 `DOWN` | 检查网线、接口、驱动或 `nmcli device connect` |
| 没有 `192.168.137.x`（或实际 ICS 网段） | DHCP 未成功；检查 ICS 下游接口和树莓派连接 |
| 没有 `default via ...` | 没有默认网关，外网一定不可达 |
| 网关可达、公共 IP 不可达 | ICS/NAT、VPN、Windows 防火墙或上游连接问题 |
| 公共 IP 可达、域名不可达 | DNS 配置问题 |
| `ip neigh` 显示 `INCOMPLETE` | ARP 请求未得到响应，优先查二层链路和同网段配置 |

## 常见命令的兼容性边界

- `ip`、`ip route`、`ip neigh` 是现代 Linux 首选，但其实现来自 `iproute2`，极精简镜像可能需要安装。
- `ifconfig` 和 `arp` 属于旧的 `net-tools`，不应把“命令不存在”误判为网卡故障。
- `nmcli` 只在 NetworkManager 管理网络时有意义；使用 dhcpcd、systemd-networkd 或其他方案的系统应查对应服务。
- `resolv.conf` 可能是指向 systemd-resolved 或 NetworkManager 的符号链接，直接编辑不一定持久；先确认网络管理器。
- `nmap` 能发现设备和端口，但扫描他人网络前必须获得授权。

## 相关笔记

- [[Windows_树莓派网线SSH与网络共享_问题排查]]
- [[Windows_ICS与网桥_原理与地址]]
- [[Windows_树莓派网络共享_专题索引]]
