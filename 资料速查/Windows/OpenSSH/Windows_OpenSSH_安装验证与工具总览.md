---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, 安装, 工具总览]
技术栈: [windows, powershell, openssh]
tags: [windows, openssh, powershell, 命令行, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [资料速查] Windows OpenSSH 安装验证与工具总览

> 专题导航：[[Windows_OpenSSH_专题索引]]

## 用途

Windows 11 将 OpenSSH 客户端和服务器作为“可选功能”提供。客户端负责主动连接其他主机；服务器端 `sshd` 只有在需要让其他设备登录本机时才安装。

## 查看安装状态

在管理员 PowerShell 中查看两个能力包：

```powershell
Get-WindowsCapability -Online |
  Where-Object Name -Like 'OpenSSH*'
```

常见名称：

| 能力包 | 用途 |
| --- | --- |
| `OpenSSH.Client~~~~0.0.1.0` | 提供 `ssh`、`sftp`、`scp` 和密钥工具 |
| `OpenSSH.Server~~~~0.0.1.0` | 提供 `sshd`、`sftp-server` 等服务端程序 |

安装客户端或服务器：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

卸载时将 `Add-WindowsCapability` 换成 `Remove-WindowsCapability`。安装/卸载系统能力需要管理员权限，企业策略或 WSUS 配置可能阻止在线获取组件。

## 客户端工具

| 程序 | 主要用途 | 对应笔记 |
| --- | --- | --- |
| `ssh.exe` | 远程登录、执行命令、端口转发 | [[SSH_远程登录命令与参数]] |
| `sftp.exe` | 在加密连接中交互或批量传输文件 | [[SFTP_交互式文件传输与参数]] |
| `scp.exe` | 用一条命令复制文件或目录 | [[SCP_文件复制命令与参数]] |
| `ssh-keygen.exe` | 生成、转换、检查密钥和证书 | [[OpenSSH_密钥工具与ssh-agent]] |
| `ssh-add.exe` | 将私钥加入认证代理 | [[OpenSSH_密钥工具与ssh-agent]] |
| `ssh-agent.exe` | 缓存已解锁的私钥 | [[OpenSSH_密钥工具与ssh-agent]] |
| `ssh-keyscan.exe` | 批量获取服务器公开的主机密钥 | [[OpenSSH_密钥工具与ssh-agent]] |

`ssh-pkcs11-helper.exe` 和 `ssh-sk-helper.exe` 是 OpenSSH 为 PKCS#11/FIDO 安全密钥启动的内部辅助程序，没有日常用户命令语法，不应直接调用。

服务器能力包还会提供 `sshd.exe` 和 `sftp-server.exe`，见 [[Windows_OpenSSH_服务器配置]]。

## 程序和配置位置

| 内容 | Windows 默认位置 |
| --- | --- |
| 可执行文件 | `C:\Windows\System32\OpenSSH\` |
| 当前用户客户端配置 | `%USERPROFILE%\.ssh\config` |
| 当前用户已知主机 | `%USERPROFILE%\.ssh\known_hosts` |
| 当前用户默认密钥 | `%USERPROFILE%\.ssh\id_*`、`id_*.pub` |
| 全局客户端配置 | `%ProgramData%\ssh\ssh_config`（存在时） |
| 服务器配置 | `%ProgramData%\ssh\sshd_config` |
| 服务器主机密钥 | `%ProgramData%\ssh\ssh_host_*` |

OpenSSH 配置中的路径优先写 `/`；PowerShell 命令中的 Windows 路径可以继续写 `\`。带空格的路径要加引号。

## 最小验证流程

### 1. 确认可执行文件

```powershell
Get-Command ssh,sftp,scp,ssh-keygen,ssh-add,ssh-keyscan
ssh -V
```

本机验证结果为：

```text
OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
```

### 2. 验证目标端口

```powershell
Test-NetConnection 服务器地址 -Port 22
```

`TcpTestSucceeded : True` 只证明 TCP 端口可达，不证明用户名、密码或密钥正确。

### 3. 发起连接

```powershell
ssh 用户名@服务器地址
sftp 用户名@服务器地址
```

首次连接应先核对服务器指纹，再输入 `yes` 接受；接受后记录写入 `known_hosts`。

## 版本与帮助的反直觉点

- `ssh -V` 是显示版本，使用大写 `V`。
- `ssh -v`、`sftp -v` 和 `scp -v` 是显示连接调试信息。
- `sftp -v` 仍然缺少必填的 `destination`，所以只会输出 usage。正确调试写法是 `sftp -v 用户名@服务器`。
- 本机 9.5p2 的 `sftp` 和 `scp` 不支持 `-V`。它们和 `ssh.exe` 属于同一套 Windows OpenSSH，可以用 `ssh -V` 查看套件版本，或查看可执行文件属性。
- `ssh -?`、`sftp -h` 等帮助开关并不统一；无效参数通常也会打印 usage。完整语义应查看对应手册，而不是只看一行 usage。

## PowerShell 路径与参数解析

- 本地相对路径用 `.` 表示当前目录，例如 `scp .\a.txt user@host:/tmp/`。
- PowerShell 会展开 `$变量`，远程命令中包含 `$` 时要正确引用。
- 远程规格中的冒号用于分隔主机和路径，例如 `user@host:/var/log/a.log`；本地文件名如果含冒号会被当成远程规格。
- IPv6 地址放在方括号中，例如 `user@[2001:db8::10]`；URI 语法可写 `ssh://user@[2001:db8::10]:2222`。

## 验证与边界

- 本笔记在 Windows 11、本机 OpenSSH 9.5p2 上验证了程序位置、版本和 usage。
- OpenSSH 会随 Windows 更新而升级；新增或删除的参数以 `ssh -V` 对应版本的手册为准。
- Windows 自带客户端不等于目标服务器已经开启 SSH；服务端必须监听端口且防火墙允许连接。

## 来源

- [Microsoft Learn：Windows 中的 OpenSSH 概述](https://learn.microsoft.com/windows-server/administration/openssh/openssh-overview)
- [Microsoft Learn：开始使用 OpenSSH Server](https://learn.microsoft.com/windows-server/administration/openssh/openssh_install_firstuse)
- [Win32-OpenSSH Wiki](https://github.com/PowerShell/Win32-OpenSSH/wiki)

## 相关笔记

- [[SSH_远程登录命令与参数]]
- [[Windows_OpenSSH_故障排查]]
