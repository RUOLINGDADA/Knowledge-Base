---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, SFTP, 文件传输]
技术栈: [windows, openssh, sftp]
tags: [windows, openssh, sftp, 文件传输, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [资料速查] SFTP 交互式文件传输与参数

> 专题导航：[[Windows_OpenSSH_专题索引]]  
> 单次复制：[[SCP_文件复制命令与参数]]

## 用途与语法

SFTP 是运行在 SSH 加密连接上的文件传输协议。它不是传统 FTP/FTPS，也不需要额外开放 FTP 数据端口。

```text
sftp [选项] destination
destination = [user@]host[:path]
```

常用连接：

```powershell
sftp pi@192.168.137.20
sftp -P 2222 -i "$env:USERPROFILE\.ssh\id_ed25519" pi@server
sftp pi@server:/var/log
```

进入 `sftp>` 提示符后，命令才是 `get`、`put`、`ls` 等交互命令。

## 启动参数详解

以下参数与本机 OpenSSH 9.5p2 的 usage 对齐。

| 参数 | 作用 | 关键说明 |
| --- | --- | --- |
| `-4` | 仅使用 IPv4 | 避免优先连接不可用的 IPv6 地址。 |
| `-6` | 仅使用 IPv6 | 目标和网络必须支持 IPv6。 |
| `-A` | 启用 `ssh-agent` 转发 | 风险同 `ssh -A`，只对可信服务器使用。 |
| `-a` | 尝试续传 | 从已有文件末尾继续；源和部分文件不一致会产生损坏文件。 |
| `-B size` | 设置传输缓冲区字节数 | 默认 32768；增大可能减少往返但增加内存。 |
| `-b batchfile` | 从批处理文件读取命令 | `-` 表示标准输入；应配合非交互认证。 |
| `-C` | 启用 SSH 压缩 | 文本或慢链路可能受益，压缩包/视频通常无收益。 |
| `-c cipher` | 指定加密算法 | 通常保留默认；可用算法见 `ssh -Q cipher`。 |
| `-D command` | 直接连接本地 SFTP 服务程序 | 用于调试服务端程序，不经过网络 SSH 连接。 |
| `-F config` | 指定 SSH 客户端配置文件 | 读取指定配置替代默认用户配置。 |
| `-f` | 传输结束后请求服务器执行 `fsync` | 需要服务器支持 `fsync@openssh.com` 扩展。 |
| `-i identity` | 指定私钥文件 | 相当于向底层 SSH 传入身份文件。 |
| `-J destination` | 通过跳板机连接 | 相当于 `ProxyJump`，可写 `user@jump:port`。 |
| `-l limit` | 限制带宽 | 单位是 Kbit/s，不是 KB/s。 |
| `-N` | 取消批处理隐含的安静模式 | 常与 `-b` 配合，让输出保持可见。 |
| `-o option` | 传入任意 SSH 配置项 | 例如 `-o ConnectTimeout=10`；可多次使用。 |
| `-P port` | 指定服务器端口 | SFTP 使用大写 `-P`，区别于 `ssh -p`。 |
| `-p` | 保留时间和权限模式 | 对 Windows、本地文件系统和服务端权限模型的支持可能不同。 |
| `-q` | 安静模式 | 关闭进度条以及 SSH 警告、诊断输出。 |
| `-R count` | 设置并发请求数 | 默认 64；增大可能提高高延迟链路吞吐，也增加内存。 |
| `-r` | 递归传输目录 | 遍历目录时不跟随遇到的符号链接。 |
| `-S program` | 指定用于加密连接的程序 | 替换默认 `ssh`，程序必须理解 SSH 参数。 |
| `-s subsystem` | 指定远程子系统或 SFTP 服务路径 | 用于非默认服务端配置或不支持子系统的服务器。 |
| `-v` | 输出详细调试信息 | 可叠加为 `-vvv`；必须仍提供目标，例如 `sftp -v user@host`。 |
| `-X option` | 调整 SFTP 协议传输参数 | `nrequests=value` 控制并发数，`buffer=value` 控制单次读写缓冲区。 |

`-B`/`-R` 与 `-X buffer=`/`-X nrequests=` 分别提供传统参数和协议选项写法；无性能证据时保留默认。

## 交互命令速查

交互命令不区分大小写。含空格的路径要用引号包住；命令前加 `!` 可在本地执行命令。

### 帮助与退出

| 命令 | 作用 |
| --- | --- |
| `help` 或 `?` | 显示交互命令帮助 |
| `bye`、`exit`、`quit` | 关闭 SFTP 会话 |
| `version` | 显示 SFTP 协议版本 |
| `progress` | 开关进度条 |

### 查看和切换目录

| 远程命令 | 本地对应命令 | 作用 |
| --- | --- | --- |
| `pwd` | `lpwd` | 显示当前目录 |
| `cd path`、`chdir path` | `lcd path`、`lchdir path` | 切换远程/本地目录 |
| `ls [-1afhlnrSt] [path]` | `lls [options] [path]` | 列出目录；`lls` 的参数由本地命令解释 |
| `mkdir path` | `lmkdir path` | 创建目录 |

远程 `ls` 常用参数：

| 参数 | 含义 |
| --- | --- |
| `-1` | 每行一个条目 |
| `-a` | 包含点开头的隐藏文件 |
| `-f` | 不排序 |
| `-h` | 以易读单位显示大小，需配合长格式 |
| `-l` | 长格式 |
| `-n` | 长格式中的用户/组显示为数字 ID |
| `-r` | 反向排序 |
| `-S` | 按文件大小排序 |
| `-t` | 按修改时间排序 |

### 上传和下载

| 命令 | 作用 |
| --- | --- |
| `get [-afpR] remote [local]` | 下载文件或目录 |
| `put [-afpR] local [remote]` | 上传文件或目录 |
| `reget [-fpR] remote [local]` | 继续下载，等价于 `get -a` |
| `reput [-fpR] local [remote]` | 继续上传，等价于 `put -a` |
| `copy old new`、`cp old new` | 在远程服务器内复制文件；需要服务端支持对应扩展 |

传输参数：`-a` 续传，`-f` 完成后请求 `fsync`，`-p` 保留属性，`-R` 递归。交互命令的递归参数是大写 `-R`；程序启动参数递归是小写 `-r`。

### 远程文件管理

| 命令 | 作用 |
| --- | --- |
| `rm path` | 删除远程文件 |
| `rmdir path` | 删除空的远程目录 |
| `rename old new` | 重命名或移动远程路径 |
| `ln [-s] old new` | 建立硬链接；`-s` 建立符号链接 |
| `chmod mode path` | 修改远程权限模式，如 `chmod 600 file` |
| `chown owner path` | 以数字 UID 修改所有者 |
| `chgrp group path` | 以数字 GID 修改组 |
| `symlink old new` | 建立符号链接 |
| `df [-hi] [path]` | 查看远程文件系统容量；`-h` 易读，`-i` 查看 inode |

### 本地操作

| 命令 | 作用 |
| --- | --- |
| `!` | 启动本地 shell |
| `!command` | 在本地执行一条命令 |
| `lumask umask` | 设置本地文件创建掩码；Windows 语义可能受限 |

## 一次完整交互示例

```text
sftp> lpwd
sftp> lcd C:/Users/你的用户名/Downloads
sftp> pwd
sftp> cd /home/pi
sftp> put "local file.txt" "remote file.txt"
sftp> get -p report.pdf
sftp> bye
```

## 批处理

新建 `commands.sftp`：

```text
lcd C:/data/out
cd /srv/export
get report.csv
bye
```

执行：

```powershell
sftp -b .\commands.sftp -o BatchMode=yes user@server
```

- 批处理中命令失败通常会中止；在命令前加 `-` 可忽略该命令的失败，例如 `-rm old.tmp`。
- 命令前加 `@` 可禁止回显，例如 `@lcd C:/data/out`。
- 自动化必须提前配置密钥和主机信任；不要把密码写进批处理文件。

## Windows 路径与远程路径

- `lcd`、`put` 的本地路径推荐写 `C:/Users/name/file.txt`，减少反斜杠转义问题。
- `cd`、`get` 的远程路径由服务器解释；Linux 常用 `/home/user`，Windows SFTP 服务器可能显示 `/C:/Users/name` 或其他映射，以实际 `pwd` 为准。
- 通配符由 SFTP 客户端匹配；路径包含空格、`[`、`*` 等特殊字符时要引用或转义。
- SFTP 没有 `-V` 版本参数；`-v` 是调试。套件版本用 `ssh -V` 查看。
- OpenSSH 9.5p2 的交互命令清单没有传统 FTP 的 `mget`/`mput`；多文件传输使用带通配符的 `get`/`put`，或使用 `sftp -b` 批处理。

## 来源

- [OpenBSD Manual：sftp(1)](https://man.openbsd.org/sftp.1)
- [OpenBSD Manual：ssh_config(5)](https://man.openbsd.org/ssh_config.5)

## 相关笔记

- [[SCP_文件复制命令与参数]]
- [[OpenSSH_密钥工具与ssh-agent]]
- [[Windows_OpenSSH_故障排查]]
