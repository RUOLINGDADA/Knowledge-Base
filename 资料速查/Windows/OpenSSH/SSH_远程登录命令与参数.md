---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, SSH, 远程登录]
技术栈: [windows, openssh, ssh]
tags: [windows, openssh, ssh, 远程登录, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [资料速查] SSH 远程登录命令与参数

> 专题导航：[[Windows_OpenSSH_专题索引]]  
> 配置复用：[[OpenSSH_客户端配置与端口转发]]

## 用途与语法

`ssh` 在加密通道中登录远程系统、执行远程命令或建立端口转发。

```text
ssh [选项] destination [command [argument ...]]
destination = [user@]hostname
```

也支持 URI：`ssh://[user@]hostname[:port]`。

## 常用示例

```powershell
ssh pi@192.168.137.20
ssh -p 2222 -i "$env:USERPROFILE\.ssh\id_ed25519" pi@server
ssh pi@server -- uname -a
```

退出交互会话可执行远程端的 `exit`，或按 `Ctrl+D`。连接失去响应时，在本地新行输入 `~.` 强制断开；`~?` 查看转义命令。

## 全部命令行参数

以下参数与本机 OpenSSH 9.5p2 的 usage 对齐。

| 参数 | 作用 | 关键说明 |
| --- | --- | --- |
| `-4` | 仅使用 IPv4 | DNS 同时返回 IPv4/IPv6 时用于固定地址族。 |
| `-6` | 仅使用 IPv6 | 目标和本地网络必须支持 IPv6。 |
| `-A` | 启用认证代理转发 | 远端可借用本机 agent 发起认证；只对可信服务器启用。 |
| `-a` | 禁用认证代理转发 | 覆盖配置文件中的 `ForwardAgent yes`。 |
| `-B interface` | 指定本地出站网络接口 | 对应 `BindInterface`，如 Wi-Fi、以太网或 VPN 接口。 |
| `-b address` | 指定本地源 IP | 多网卡主机上选择连接所用的源地址。 |
| `-C` | 启用压缩 | 慢链路上的文本可能受益；已压缩文件和高速网络通常收益很小。 |
| `-c cipher_spec` | 指定加密算法列表 | 可用算法由 `ssh -Q cipher` 查询；一般保留默认协商。 |
| `-D [bind:]port` | 建立本地动态转发 | 创建 SOCKS4/5 代理，例如 `-D 127.0.0.1:1080`。 |
| `-E log_file` | 将调试日志追加到文件 | 适合保留排错证据；日志可能包含主机名、用户名等信息。 |
| `-e char` | 设置交互转义字符 | 默认 `~`；写 `none` 可禁用。只在新行开头识别。 |
| `-F configfile` | 指定客户端配置文件 | 指定后不读取默认的用户配置文件。 |
| `-f` | 认证后转入后台 | 隐含 `-n`；常和端口转发及 `-N` 配合。 |
| `-G` | 输出最终生效配置后退出 | 不连接服务器；非常适合检查别名和覆盖关系。 |
| `-g` | 允许其他主机访问本地转发端口 | 对应 `GatewayPorts yes`；会扩大暴露面。 |
| `-I pkcs11` | 指定 PKCS#11 共享库 | 用于智能卡/HSM；Windows 提供者兼容性需单独验证。 |
| `-i identity_file` | 指定私钥或证书文件 | 可多次使用；不会自动改变服务器授权。 |
| `-J destination` | 通过跳板机连接 | 是 `ProxyJump` 的命令行写法，可用逗号串联多跳。 |
| `-K` | 启用 GSSAPI 认证及凭据转发 | 主要用于 Kerberos/域环境；目标和客户端均需配置。 |
| `-k` | 禁用 GSSAPI 凭据转发 | 覆盖已启用的 GSSAPI 凭据转发。 |
| `-L spec` | 建立本地端口转发 | 本地监听，流量经服务器访问目标，详见端口转发笔记。 |
| `-l user` | 指定远程用户名 | 等价于目标中的 `user@host`。 |
| `-M` | 将连接置为复用主连接 | 依赖 `ControlPath`；Windows 上连接复用能力应按版本验证。 |
| `-m mac_spec` | 指定 MAC 算法列表 | 用 `ssh -Q mac` 查询；通常无需手动修改。 |
| `-N` | 不执行远程命令 | 专用于端口转发或仅保持连接。 |
| `-n` | 将标准输入重定向为空 | 防止后台 SSH 读取当前终端输入。 |
| `-O ctl_cmd` | 控制复用主连接 | 常见命令有 `check`、`forward`、`cancel`、`exit`、`stop`。 |
| `-o option` | 传入一个 `ssh_config` 选项 | 格式如 `-o ConnectTimeout=10`；可多次使用。 |
| `-P tag` | 指定配置标签 | 供 `ssh_config` 的 `Match tagged` 选择条件使用，不是端口。 |
| `-p port` | 指定服务器端口 | SSH 使用小写 `-p`；默认端口是 22。 |
| `-Q query` | 查询算法和能力 | 如 `cipher`、`mac`、`key`、`key-sig`、`kex`、`protocol-version`。 |
| `-q` | 安静模式 | 抑制警告和诊断信息，不适合排错。 |
| `-R spec` | 建立远程端口转发 | 远端监听，流量经客户端访问目标；受服务器策略限制。 |
| `-S ctl_path` | 指定复用控制套接字路径 | 写 `none` 禁用连接共享；Windows 支持度需按版本验证。 |
| `-s` | 请求远程子系统 | 命令参数被当作子系统名，例如 `sftp`。日常 SFTP 直接用 `sftp`。 |
| `-T` | 禁止分配伪终端 | 适合自动化和纯命令输出。 |
| `-t` | 强制分配伪终端 | 可重复为 `-tt`，即使本地没有终端也强制分配。 |
| `-V` | 显示版本并退出 | 本机结果为 OpenSSH_for_Windows 9.5p2。 |
| `-v` | 输出调试信息 | 最多叠加到 `-vvv`；级别越高越详细。 |
| `-W host:port` | 将标准输入/输出转发到目标 | 常用于 `ProxyCommand`，隐含 `-N -T`。 |
| `-w local[:remote]` | 请求隧道设备转发 | 依赖两端 tun 支持和管理员配置，Windows 场景通常不适用。 |
| `-X` | 启用受限制的 X11 转发 | 需要本地 X Server 和服务端支持；Windows 默认没有 X Server。 |
| `-x` | 禁用 X11 转发 | 覆盖配置中的 X11 转发。 |
| `-Y` | 启用可信 X11 转发 | 远程程序获得更高 X11 权限，风险高于 `-X`。 |
| `-y` | 通过系统日志设施发送日志 | 该行为源自 Unix OpenSSH；Windows 日志落点需按构建验证。 |

## 常用 `-o` 选项

| 写法 | 用途 |
| --- | --- |
| `-o ConnectTimeout=10` | 连接阶段最多等待 10 秒 |
| `-o ConnectionAttempts=2` | 限制连接尝试次数 |
| `-o BatchMode=yes` | 禁止密码和确认提示，适合自动化 |
| `-o IdentitiesOnly=yes` | 只尝试显式配置的密钥 |
| `-o StrictHostKeyChecking=accept-new` | 自动接受新主机，但拒绝已记录主机密钥的变化 |
| `-o ServerAliveInterval=30` | 空闲 30 秒后发送应用层保活消息 |
| `-o ServerAliveCountMax=3` | 连续 3 次无响应后断开 |
| `-o ExitOnForwardFailure=yes` | 端口转发建立失败时直接退出 |

不建议日常使用 `StrictHostKeyChecking=no`，因为它会削弱主机身份校验。

## 跳板机和端口转发示例

```powershell
ssh -J jumpuser@jump.example.com appuser@10.0.0.8
ssh -N -L 127.0.0.1:5433:db.internal:5432 user@gateway
ssh -N -D 127.0.0.1:1080 user@gateway
```

完整方向说明见 [[OpenSSH_客户端配置与端口转发#三类端口转发]]。

## 远程命令与退出码

```powershell
ssh user@server -- 'df -h /'
$LASTEXITCODE
```

- 未提供远程命令时，`ssh` 启动登录会话。
- 提供命令时，SSH 返回远程命令的退出码。
- 连接本身发生错误时，`ssh` 通常返回 `255`。
- PowerShell 会先处理本地引号和变量；复杂脚本更适合上传脚本文件后执行。

## 安全边界与踩坑

- 首次连接显示的是服务器主机密钥指纹，不是用户公钥；必须通过可信渠道核对。
- `-A` 会让被入侵的中间服务器借用代理进行认证，不要对不可信跳板机启用。
- `-L`、`-R`、`-D` 的监听地址决定端口是否只对本机开放；优先绑定 `127.0.0.1`。
- 修改算法选项可能导致降级或兼容问题；只有确认旧设备限制后才临时启用旧算法。
- `-p` 是 SSH 端口，而 SFTP/SCP 使用大写 `-P` 指定端口。

## 来源

- [OpenBSD Manual：ssh(1)](https://man.openbsd.org/ssh.1)
- [OpenBSD Manual：ssh_config(5)](https://man.openbsd.org/ssh_config.5)
- [Microsoft Learn：Windows 中的 OpenSSH](https://learn.microsoft.com/windows-server/administration/openssh/openssh-overview)

## 相关笔记

- [[OpenSSH_客户端配置与端口转发]]
- [[OpenSSH_密钥工具与ssh-agent]]
- [[Windows_OpenSSH_故障排查]]
