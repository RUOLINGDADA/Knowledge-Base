---
分类: 技术/网络与嵌入式
层级: 理解层
类型: 问题排查型
主题: [Windows 11, 树莓派, SSH, 网线直连, Internet Connection Sharing, 网络桥接]
技术栈: [windows, linux, raspberry-pi, ssh, tcp-ip]
tags: [windows, raspberry-pi, ssh, 网络共享, 网桥, 问题排查, 理解层]
状态: 用户已验证
创建日期: 2026-09-08
---

# [技术/网络与嵌入式] Windows 11 通过网线连接树莓派并共享网络：问题排查

## 问题现象

目标是让 Windows 11 笔记本通过 WLAN 连接互联网，再用网线连接树莓派，同时满足：

- Windows 可以通过 SSH 登录树莓派。
- 树莓派可以经过 Windows 访问互联网。

最初在 WLAN 属性中找不到“共享”选项卡。截图显示 WLAN 为“已启用，桥接的”，网络连接列表中还存在“网桥”。删除网桥、恢复独立网卡后，网络共享成功。

## 目标拓扑

```text
互联网
  |
Windows 11 WLAN（上游、能上网）
  |
Windows ICS/NAT/DHCP
  |
Windows 以太网（下游） ---- 网线 ---- 树莓派 eth0/end0
```

SSH 只依赖 Windows 与树莓派之间的局域网连通；树莓派能否访问互联网则还依赖 Windows 的 ICS 转发、DHCP、DNS 和防火墙。

## 排查过程

### 1. 先检查是否存在网络桥接

运行 `ncpa.cpl` 打开“网络连接”。如果 WLAN 或以太网的状态包含“桥接的”，并且有名为“网桥”的逻辑适配器，说明物理网卡已成为二层桥的一部分。

### 2. 为什么共享选项卡消失

Windows 的“共享”选项卡是 Internet Connection Sharing（ICS）的配置入口。ICS 需要把一个上游连接共享到一个明确的下游网卡；网络桥接则把多个网卡合并到一个二层逻辑接口，由网桥统一处理流量。作为网桥成员的 WLAN 不再是可独立绑定 ICS 下游接口的普通适配器，因此 Windows 通常隐藏或禁用该成员网卡的“共享”选项卡。

这不是“必须删除网桥才能使用 ICS”的网络协议规则，而是两种 Windows 网络角色在当前图形界面和驱动组合下不能按预期同时配置。若确实要保留桥，应改用另一块独立的 USB 网卡来承载 ICS，或改为路由器/交换机方案。

### 3. 恢复 ICS 的操作

1. 关闭 WLAN 属性窗口。
2. 在“网络连接”中右键“网桥”，选择“删除”；若显示“从网桥中删除”，按实际菜单操作。
3. 确认 WLAN 和以太网恢复为两个独立连接。
4. 右键能上网的 WLAN →“属性”→“共享”。
5. 勾选“允许其他网络用户通过此计算机的 Internet 连接来连接”。
6. “家庭网络连接”选择物理网线对应的“以太网”（Realtek PCIe GbE），不要选 `vEthernet`、WSL、Tailscale 或其他虚拟适配器。

删除前应确认网桥没有被虚拟机、Hyper-V 或其他专用网络配置依赖；删除网桥可能使这些虚拟网络暂时失效，但不会卸载物理网卡驱动。

## 关键配置与验证

### Windows 端

```powershell
ipconfig
Get-NetIPConfiguration
Get-NetAdapter
```

启用 ICS 后，以太网通常为 `192.168.137.1/24`。实际结果以 `ipconfig` 为准，不要仅凭默认值猜测。

### 树莓派端

```bash
ip -br addr
ip route
hostname -I
```

有线接口应获得与 Windows 以太网相同网段的地址，例如 `192.168.137.20/24`；路由表应有 `default via 192.168.137.1`。

### 分段测试

```bash
ping -c 4 192.168.137.1  # 测试树莓派到 Windows 下游接口
ping -c 4 8.8.8.8        # 测试经过 ICS 的三层转发
ping -c 4 www.baidu.com  # 测试 DNS 解析和外网访问
```

判断逻辑：

| 结果 | 优先怀疑 |
| --- | --- |
| 连 `192.168.137.1` 都不通 | 网线、接口状态、IP 地址、子网掩码或防火墙 |
| 网关能通，`8.8.8.8` 不通 | ICS 未生效、共享了错误的上游/下游接口、VPN 或防火墙拦截 |
| `8.8.8.8` 能通，域名不通 | DNS 配置或 DNS 转发 |
| SSH 不通但上述测试正常 | SSH 服务未启动、端口 22 被拦截、用户名或地址错误 |

## SSH 连接

Windows PowerShell：

```powershell
ssh <树莓派用户名>@<树莓派IP>
```

树莓派端确认服务：

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

新版 Raspberry Pi OS 不保证存在默认用户 `pi`，应使用安装系统时创建的用户名。

## 踩坑记录

- “家庭网络连接”不是指家庭 Wi-Fi 名称，而是 ICS 的下游接口；本场景下必须选择连接树莓派的物理以太网。
- 不要把 `vEthernet (Default Switch)`、`vEthernet (WSL)`、Tailscale 或“网桥”当作树莓派所在的下游接口，除非有意设计了对应的虚拟网络路径。
- Windows 共享启用后以太网通常自动改成 `192.168.137.1`；若仍是旧的静态地址，先恢复 IPv4 自动获取，再关闭并重新开启共享。
- VPN、安全软件和 Hyper-V/WSL 虚拟交换机可能改变路由或阻止 ICS；排障时应记录这些依赖。

## 原理 / 因果点睛

网桥是二层转发，ICS 是三层 NAT 加 DHCP/DNS 转发。当前 Windows 图形界面要求 ICS 绑定清晰的独立下游适配器，所以 WLAN 成为网桥成员后，其 ICS 配置入口会消失；解除桥接后，Windows 才能把以太网作为共享出口管理。

## 相关笔记

- [[Windows_ICS与网桥_原理与地址]]
- [[Windows与Linux_IP地址查询命令_速查]]
- [[Windows_树莓派网络共享_专题索引]]
