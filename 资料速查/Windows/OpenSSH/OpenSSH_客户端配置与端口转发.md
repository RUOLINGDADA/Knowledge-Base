---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, ssh_config, 端口转发]
技术栈: [windows, openssh, ssh, networking]
tags: [windows, openssh, ssh_config, 端口转发, 理解层]
状态: 待实际验证
创建日期: 2026-09-08
---

# [资料速查] OpenSSH 客户端配置与端口转发

> 专题导航：[[Windows_OpenSSH_专题索引]]  
> 命令参数：[[SSH_远程登录命令与参数]]

## 配置文件层级

OpenSSH 客户端通常按以下顺序读取配置；同一参数第一次取得的值优先，因此更具体的 `Host` 块应放在前面：

1. `-F` 指定的配置文件（如果使用）。
2. 当前用户 `%USERPROFILE%\.ssh\config`。
3. 全局 `%ProgramData%\ssh\ssh_config`（存在时）。

本机配置示例：

```sshconfig
Host pi
    HostName 192.168.137.20
    User pi
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
```

之后可以直接：

```powershell
ssh pi
sftp pi
scp .\file.txt pi:/home/pi/
```

Windows OpenSSH 配置中的 `~` 指当前用户的 SSH 目录；也可使用绝对路径。配置文件权限过宽时，客户端或安全软件可能拒绝使用，应限制为当前用户可写。

## 常用配置指令

| 指令 | 用途 |
| --- | --- |
| `Host pattern` | 定义别名和匹配范围；`*`、`?`、`!pattern` 可用于通配。 |
| `HostName` | 实际 DNS 名称或 IP。 |
| `User` | 默认远程用户名。 |
| `Port` | 默认远程端口。 |
| `IdentityFile` | 私钥或证书路径；可多次指定。 |
| `IdentitiesOnly yes` | 只使用配置的身份，避免 agent 中密钥过多导致尝试失败。 |
| `PreferredAuthentications` | 调整认证顺序，如 `publickey,password`。 |
| `PasswordAuthentication` | 是否允许密码认证（客户端偏好，最终由服务端决定）。 |
| `StrictHostKeyChecking` | 主机密钥策略：`yes`、`accept-new`、`no`。 |
| `UserKnownHostsFile` | 指定主机密钥数据库；可写多个文件。 |
| `GlobalKnownHostsFile` | 指定全局主机密钥数据库。 |
| `ConnectTimeout` | 连接建立和初始握手超时时间。 |
| `ConnectionAttempts` | 每个目的地的连接尝试次数。 |
| `ServerAliveInterval` / `ServerAliveCountMax` | 应用层保活和无响应断开阈值。 |
| `ProxyJump` | 通过跳板机转发连接。 |
| `ProxyCommand` | 自定义代理命令；常见是 `ssh -W %h:%p jump`。 |
| `LocalForward` / `RemoteForward` / `DynamicForward` | 配置三类端口转发。 |
| `ExitOnForwardFailure` | 转发建立失败时退出。 |
| `LogLevel` | `QUIET`、`ERROR`、`INFO`、`VERBOSE`、`DEBUG1`～`DEBUG3`。 |

检查最终配置而不连接：

```powershell
ssh -G pi
ssh -G -F .\test_config user@server | Select-String 'hostname|user|port|identityfile'
```

## 三类端口转发

### 1. 本地转发 `-L`

语法：

```text
-L [bind_address:]listen_port:destination_host:destination_port
```

本机监听端口，连接后由 SSH 服务器访问目标：

```powershell
ssh -N -L 127.0.0.1:15432:db.internal:5432 user@gateway
```

此时本机应用连接 `127.0.0.1:15432`，实际到达网关可访问的 `db.internal:5432`。

### 2. 远程转发 `-R`

语法：

```text
-R [bind_address:]listen_port:destination_host:destination_port
```

远端监听端口，连接后由 SSH 客户端访问目标：

```powershell
ssh -N -R 127.0.0.1:18080:127.0.0.1:8080 user@server
```

服务端的 `AllowTcpForwarding`、`GatewayPorts` 等策略可能禁止或限制监听地址。

### 3. 动态转发 `-D`

语法：

```text
-D [bind_address:]port
```

在本机建立 SOCKS4/5 代理：

```powershell
ssh -N -D 127.0.0.1:1080 user@gateway
```

应用需要显式支持 SOCKS 代理；它不是自动接管 Windows 全部流量的 VPN。

## 转发安全参数

```powershell
ssh -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 `
  -L 127.0.0.1:15432:db.internal:5432 user@gateway
```

- 绑定 `127.0.0.1` 只让本机访问；绑定 `0.0.0.0` 会让局域网可访问，应视为开放服务并配套防火墙和认证。
- `-N` 表示不打开远程终端。
- `-f` 可在认证后转入后台，但 PowerShell/Windows 后台行为应在目标版本实际验证；诊断阶段保持前台更容易观察。
- `ExitOnForwardFailure=yes` 能避免“SSH 连接成功但转发端口没有监听”的假成功。
- `-g` 或 `GatewayPorts` 会扩大本地/远程转发监听范围，默认不应打开。

## 代理跳转

```powershell
ssh -J jumpuser@jump.example.com appuser@10.0.0.8
```

配置写法：

```sshconfig
Host internal-app
    HostName 10.0.0.8
    User appuser
    ProxyJump jumpuser@jump.example.com
```

目标主机的主机密钥仍应按目标主机名/IP 校验；跳板机只负责传输，不等于目标主机身份。

## 连接复用和控制命令

OpenSSH 的 `ControlMaster`、`ControlPath`、`ControlPersist` 可以复用已认证连接，减少重复握手。Windows 支持情况和路径限制随版本变化，启用前先用 `ssh -G` 查看配置、再做窄范围测试。

```powershell
ssh -O check -S C:/Temp/ssh-control user@server
ssh -O exit  -S C:/Temp/ssh-control user@server
```

常见 `-O` 控制命令：`check` 检查主连接，`forward` 建立转发，`cancel` 取消转发，`exit` 请求退出主连接，`stop` 停止监听新复用连接。

## 来源

- [OpenBSD Manual：ssh_config(5)](https://man.openbsd.org/ssh_config.5)
- [OpenBSD Manual：ssh(1)](https://man.openbsd.org/ssh.1)
- [Microsoft Learn：OpenSSH Server 配置](https://learn.microsoft.com/windows-server/administration/openssh/openssh_server_configuration)

## 相关笔记

- [[SSH_远程登录命令与参数]]
- [[Windows_OpenSSH_服务器配置]]
- [[Windows_OpenSSH_故障排查]]

