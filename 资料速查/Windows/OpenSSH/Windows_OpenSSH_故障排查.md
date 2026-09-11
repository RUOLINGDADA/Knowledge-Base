---
分类: 问题排查
层级: 理解层
类型: 问题排查型
主题: [Windows 11, OpenSSH, SSH, SFTP, 故障排查]
技术栈: [windows, powershell, openssh, networking]
tags: [windows, openssh, ssh, sftp, 故障排查, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [问题排查] Windows OpenSSH 连接与传输失败

> 专题导航：[[Windows_OpenSSH_专题索引]]

## 先判断失败层级

```text
命令存在 → DNS/IP 可达 → TCP 端口可达 → SSH 握手 → 主机密钥校验
→ 用户认证 → 远程权限/路径 → SFTP/SCP 文件操作
```

不要用“能 ping”替代 TCP 端口验证，也不要用“能登录”推断目标路径和文件权限一定正确。

## 分层检查命令

```powershell
Get-Command ssh,sftp,scp
Resolve-DnsName server.example.com
Test-NetConnection server.example.com -Port 22
ssh -vvv -o ConnectTimeout=10 user@server
sftp -vvv -o ConnectTimeout=10 user@server
```

服务端 Windows 上：

```powershell
Get-Service sshd
Get-NetTCPConnection -State Listen -LocalPort 22
sshd -t
Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 30
```

最后一条日志命令需要服务端已安装并写入该事件日志；无日志时先检查服务和日志配置。

## 常见现象与原因

| 现象 | 优先检查 | 说明 |
| --- | --- | --- |
| `ssh is not recognized` | OpenSSH Client 能力包、PATH | 安装客户端或调用 `C:\Windows\System32\OpenSSH\ssh.exe`。 |
| `usage: sftp ... destination` | 是否只执行了 `sftp -v` | `-v` 是调试，必须再给 `user@host`；程序本身已找到。 |
| `ssh -V` 有版本，`sftp -V` 报 unknown option | 参数大小写/工具差异 | `sftp`、`scp` 没有 `-V`；用 `ssh -V` 看套件版本。 |
| `Could not resolve hostname` | 主机名、DNS、代理 | 先用 IP 验证，再检查 DNS 和 `HostName`。 |
| `Connection timed out` | 路由、防火墙、端口 | `Test-NetConnection` 区分 TCP 不通和认证失败。 |
| `Connection refused` | 服务是否监听、端口是否正确 | 主机可达但目标端口没有服务接受连接。 |
| `No route to host` | 路由、VPN、网络段 | 检查本机路由和目标网段。 |
| `Host key verification failed` | `known_hosts` 中旧密钥 | 先确认服务器是否重装/换 IP，再处理旧记录；不要直接关闭校验。 |
| `Permission denied (publickey)` | 用户名、公钥路径、ACL、私钥 | `ssh -vvv` 看客户端实际尝试了哪些身份。 |
| `Permission denied (password)` | 密码认证策略、账户状态 | 服务端可能禁用密码或限制账户/组。 |
| `Too many authentication failures` | agent 中密钥过多 | 使用 `-o IdentitiesOnly=yes -i key` 或清理 `ssh-add`。 |
| SSH 能登录但 SFTP 失败 | `Subsystem sftp`、服务端路径和权限 | SFTP 需要服务端子系统，不等于远程 shell。 |
| `No such file or directory` | 当前远程目录和路径语法 | 在 SFTP 中先 `pwd`、`ls`，不要把本地 `C:` 路径当远程路径。 |
| 上传成功但应用看不到 | 目标目录、账户、权限、路径映射 | 通过同一账户 `pwd`、`ls -l` 复核。 |
| 传输很慢 | RTT、压缩、并发、限速 | 先测默认参数，再调整 `-C`、`-R`、`-X` 或 `-l`。 |
| `scp` 通配符行为异常 | OpenSSH 版本和协议模式 | 9.x 默认 SFTP；旧兼容需求才尝试 `-O`。 |

## 主机密钥冲突的安全处理

如果服务器确实重装、换机或更换 IP，先通过控制台、管理员或可信渠道取得新的指纹：

```powershell
ssh-keygen -F server.example.com
ssh-keygen -R server.example.com
ssh-keyscan -t ed25519 server.example.com
```

只有独立确认新指纹后，才将新记录加入 `known_hosts`。如果无法解释指纹变化，应视为潜在中间人攻击并停止连接。

## 密钥认证排查链

1. 客户端确认私钥存在：`Test-Path "$env:USERPROFILE\.ssh\id_ed25519"`。
2. 确认公私钥匹配：`ssh-keygen -y -f .\id_ed25519`，与 `.pub` 对比。
3. `ssh-add -l` 查看 agent 是否加载；agent 服务未运行时先启动服务。
4. 服务端确认公钥整行没有换行、没有复制多余字符。
5. 服务端检查 `AuthorizedKeysFile`、用户/组限制和 ACL。
6. 用 `ssh -vvv -o IdentitiesOnly=yes -i .\id_ed25519 user@server` 缩小变量。

## SFTP/SCP 路径排查

- SFTP 登录后先执行 `pwd`，再执行 `lpwd`，确认远程和本地当前目录不是同一个概念。
- Windows 本地路径推荐用 `C:/Users/name/file`；Linux 远程路径通常用 `/home/name/file`。
- `scp` 的 `user@host:path` 中冒号表示远程路径；本地带冒号的文件名需要改名或明确相对/绝对路径。
- 目录传输时确认是否需要递归：SFTP 启动用 `-r`，SCP 用 `-r`，SFTP 交互 `get/put` 使用 `-R`。
- SFTP 递归不跟随符号链接；SCP `-r` 会跟随符号链接，不能混用两者的安全假设。

## Windows 特有检查

- 服务操作、安装能力包、防火墙和 `%ProgramData%\ssh` 下配置通常需要管理员权限。
- PowerShell 的 `$env:USERPROFILE`、引号、反引号和通配符会先于 OpenSSH 解析；复杂命令先在本地打印最终参数。
- 多网卡、VPN、Windows ICS 或网桥环境中，目标 IP 可达性和默认路由可能不同；分别测试 `Test-NetConnection`、SSH 和互联网访问。
- 杀毒软件、企业防火墙、组策略和 WSUS 可能阻止安装、监听或密码认证；需要查看对应策略和事件日志。

## 证据记录模板

```text
客户端版本：ssh -V
目标：host / IP / 端口
TCP：Test-NetConnection 结果
SSH：ssh -vvv 的最后 20 行
SFTP：pwd、lpwd、失败命令
服务端：sshd 状态、sshd -t、OpenSSH/Operational 日志
```

记录时脱敏用户名、内网地址、私钥路径中的个人信息和日志中的凭据片段；绝不粘贴私钥内容。

## 来源

- [Microsoft Learn：OpenSSH Server 配置](https://learn.microsoft.com/windows-server/administration/openssh/openssh-server-configuration)
- [Win32-OpenSSH：Windows 实现与问题跟踪](https://github.com/PowerShell/Win32-OpenSSH)
- [OpenBSD Manual：ssh(1)](https://man.openbsd.org/ssh.1)
- [OpenBSD Manual：sftp(1)](https://man.openbsd.org/sftp.1)

## 相关笔记

- [[Windows_OpenSSH_安装验证与工具总览]]
- [[OpenSSH_密钥工具与ssh-agent]]
- [[Windows_OpenSSH_服务器配置]]
