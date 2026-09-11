---
分类: 资料速查
层级: 了解层
类型: 资料速查型
主题: [Windows 11, OpenSSH Server, sshd, SFTP 服务端]
技术栈: [windows, powershell, openssh, sshd]
tags: [windows, openssh, sshd, sftp, 服务端, 了解层]
状态: 待实际验证
创建日期: 2026-09-08
---

# [资料速查] Windows OpenSSH Server 配置

> 专题导航：[[Windows_OpenSSH_专题索引]]  
> 客户端验证：[[Windows_OpenSSH_安装验证与工具总览]]

## 适用场景

本笔记用于让其他设备通过 SSH/SFTP 连接 Windows 11。只在需要“别人登录这台电脑”时安装服务器；主动连接树莓派、Linux 或其他服务器只需要客户端。

## 安装并启动 `sshd`

在管理员 PowerShell 中：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

安装后检查：

```powershell
Get-Service sshd
Get-NetTCPConnection -State Listen -LocalPort 22
```

本机当前只验证到客户端；`sshd` 服务端尚未安装或连接验证，执行前应根据命令输出确认状态。

## 防火墙规则

如果安装程序没有创建规则，管理员 PowerShell 可创建 TCP 22 入站规则：

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

更安全的做法是限制远程地址、网络配置文件和端口来源，而不是对所有网络开放：

```powershell
New-NetFirewallRule -Name sshd-LAN -DisplayName 'OpenSSH LAN' `
  -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22 `
  -Profile Private -RemoteAddress 192.168.137.0/24
```

规则名称必须唯一；创建前先用 `Get-NetFirewallRule -Name sshd*` 检查，避免重复规则。

## 服务端文件位置

| 内容 | 默认位置 |
| --- | --- |
| 主配置 | `%ProgramData%\ssh\sshd_config` |
| 主机密钥 | `%ProgramData%\ssh\ssh_host_*` |
| 服务程序 | `%SystemRoot%\System32\OpenSSH\sshd.exe` |
| SFTP 服务程序 | `%SystemRoot%\System32\OpenSSH\sftp-server.exe` |
| 普通用户授权公钥 | `C:\Users\用户名\.ssh\authorized_keys` |
| 管理员组授权公钥（默认配置常见） | `%ProgramData%\ssh\administrators_authorized_keys` |
| 服务日志 | Windows 事件查看器 `OpenSSH/Operational`；也可按配置写文件 |

Windows 账户名、管理员组和 ACL 会影响授权文件路径。必须以实际 `sshd_config` 的 `AuthorizedKeysFile`、`Match Group administrators` 等配置为准。

## 修改配置的安全流程

1. 备份 `%ProgramData%\ssh\sshd_config`。
2. 用管理员权限编辑配置，只改必要项。
3. 先验证语法：

```powershell
sshd -t
```

4. 确认现有会话仍可用，再重启服务：

```powershell
Restart-Service sshd
```

不要在唯一远程会话中直接修改端口、认证或管理员公钥配置后立即断开；应保留一个已验证的会话用于回滚。

## 常用 `sshd_config` 指令

| 指令 | 作用 |
| --- | --- |
| `Port 22` | 监听端口；改端口后防火墙和客户端 `-p`/`-P` 必须同步。 |
| `ListenAddress` | 指定监听地址；多网卡或只允许局域网时使用。 |
| `PasswordAuthentication` | 是否允许密码认证。 |
| `PubkeyAuthentication` | 是否允许公钥认证。 |
| `AuthorizedKeysFile` | 授权公钥文件位置。 |
| `PermitEmptyPasswords` | 是否允许空密码；应保持 `no`。 |
| `AllowUsers` / `AllowGroups` | 白名单限制可登录账户或组。 |
| `DenyUsers` / `DenyGroups` | 黑名单禁止账户或组。 |
| `Subsystem sftp` | 指定 SFTP 子系统程序；Windows 默认通常指向 `sftp-server.exe`。 |
| `ChrootDirectory` | 将用户限制在目录树中；Windows 路径与 ACL 需要单独验证。 |
| `AllowTcpForwarding` | 控制端口转发权限。 |
| `X11Forwarding` | X11 转发；Windows 常规场景通常关闭。 |
| `LogLevel` | 服务端日志详细程度。 |

改动后用 `sshd -T` 输出生效配置，用 `sshd -t` 只做语法检查。

## `sshd` 命令行参数

服务由 Windows 服务管理器启动时通常不需要直接手工调用 `sshd`。以下参数适合安装、语法检查和前台调试；命令行参数会覆盖配置文件中的对应值：

| 参数 | 作用 |
| --- | --- |
| `-4` / `-6` | 仅监听 IPv4 / IPv6。 |
| `-C connection_spec` | 在 `-T`/`-G` 测试时提供连接条件，如 `user=pi,addr=192.168.1.10,lport=22`，用于匹配 `Match`。 |
| `-c host_certificate_file` | 指定服务器主机证书；必须与对应 `-h` 主机私钥匹配。 |
| `-D` | 前台运行，不脱离当前进程；便于观察服务输出。 |
| `-d` | 单连接调试模式；最多 `-ddd`，不会派生子进程。 |
| `-E log_file` | 将调试日志追加到文件。 |
| `-e` | 将调试日志写到标准错误。 |
| `-f config_file` | 指定服务端配置文件。 |
| `-G` | 解析并打印有效配置后退出；可结合 `-C`。 |
| `-g seconds` | 设置认证宽限时间；默认通常为 120 秒，`0` 表示不限制。 |
| `-h host_key_file` | 指定服务器主机私钥；非系统账号运行时常需要显式指定。 |
| `-i` | 从 inetd 方式运行；Windows 服务场景通常不用。 |
| `-o option` | 在命令行覆盖一个 `sshd_config` 选项，如 `-o PasswordAuthentication=no`。 |
| `-p port` | 指定监听端口；默认 22。这里是服务端 `sshd` 的小写 `-p`。 |
| `-q` | 安静模式，不写系统日志。 |
| `-T` | 扩展测试模式：检查配置、输出有效配置并退出。 |
| `-t` | 只检查配置语法和主机密钥是否正常。 |
| `-u len` | 限制记录中的远程主机名长度；`-u0` 强制使用数字地址并可避免部分 DNS 查询。 |
| `-V` | 显示 `sshd` 版本并退出。 |

Windows 服务启动参数可能由服务注册项或 `sshd_config` 生成，不要只修改手工测试命令就假设系统服务也使用了同样端口。

## `sftp-server` 服务端参数

`sftp-server.exe` 通常由 `Subsystem sftp` 调用，不应在普通 PowerShell 会话中直接运行。需要限制 SFTP 能力时可在 `sshd_config` 的 `Subsystem` 行传入参数：

| 参数 | 作用 |
| --- | --- |
| `-d start_directory` | 设置用户进入 SFTP 后的起始目录；支持 `%%`、`%d`、`%u` 替换。 |
| `-e` | 将日志写到标准错误，便于调试。 |
| `-f facility` | 指定日志设施，如 `AUTH`、`DAEMON`、`LOCAL0`～`LOCAL7`。 |
| `-h` | 显示 usage。 |
| `-l level` | 设置日志级别：`QUIET`、`ERROR`、`INFO`、`VERBOSE`、`DEBUG1`～`DEBUG3`。 |
| `-P requests` | 禁止逗号分隔的 SFTP 请求列表。 |
| `-p requests` | 只允许逗号分隔的 SFTP 请求列表；隐含请求也必须列出。 |
| `-Q requests` | 查询可限制的协议请求名称。 |
| `-R` | 只读模式，禁止写文件及其他改变文件系统的操作。 |
| `-u umask` | 为新建文件和目录设置显式 umask。 |

示例（语法和路径需按 Windows 实际安装位置调整）：

```text
Subsystem sftp sftp-server.exe -R -l INFO
```

## 连接 Windows 服务器

```powershell
ssh Windows用户名@Windows电脑IP
sftp Windows用户名@Windows电脑IP
```

使用非默认端口：

```powershell
ssh -p 2222 Windows用户名@Windows电脑IP
sftp -P 2222 Windows用户名@Windows电脑IP
```

Windows 账户包含域名时，目标字符串、PowerShell 引号和服务端账户解析可能互相影响；优先用 `whoami` 确认登录名，并按实际错误日志调整。

## 公钥登录与权限

1. 客户端生成密钥，见 [[OpenSSH_密钥工具与ssh-agent]]。
2. 将 `.pub` 文件的一整行放到服务端授权文件。
3. 确认授权文件只允许目标用户或系统管理员写入。
4. 使用 `ssh -vvv` 检查客户端是否发送了预期公钥。

Windows 管理员账号常使用 `%ProgramData%\ssh\administrators_authorized_keys`；普通账号使用用户目录下 `.ssh\authorized_keys`。如果服务端拒绝密钥，先检查 `sshd_config` 和 ACL，不要盲目重复生成密钥。

## 卸载与回滚

停止服务并取消开机启动：

```powershell
Stop-Service sshd
Set-Service sshd -StartupType Disabled
```

确认不再需要后再卸载能力包：

```powershell
Remove-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

卸载前备份配置、公钥和主机密钥；删除主机密钥会导致客户端下次连接看到指纹变化。

## 来源

- [Microsoft Learn：安装 OpenSSH Server](https://learn.microsoft.com/windows-server/administration/openssh/openssh_install_firstuse)
- [Microsoft Learn：OpenSSH Server 配置](https://learn.microsoft.com/windows-server/administration/openssh/openssh-server-configuration)
- [Microsoft Learn：OpenSSH Server 故障排查](https://learn.microsoft.com/windows-server/administration/openssh/openssh_server_troubleshoot)

## 相关笔记

- [[OpenSSH_密钥工具与ssh-agent]]
- [[Windows_OpenSSH_故障排查]]
