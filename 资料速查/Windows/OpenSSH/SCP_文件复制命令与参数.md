---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, SCP, 文件复制]
技术栈: [windows, openssh, scp, sftp]
tags: [windows, openssh, scp, 文件传输, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [资料速查] SCP 文件复制命令与参数

> 专题导航：[[Windows_OpenSSH_专题索引]]  
> 交互传输：[[SFTP_交互式文件传输与参数]]

## 用途与语法

`scp` 用一条命令在本地和远程之间复制文件。OpenSSH 9.x 默认用 SFTP 协议完成传输；`-O` 才强制使用旧 SCP 协议。

```text
scp [选项] source ... target
远程路径 = [user@]host:[path]
```

## 常用示例

```powershell
# 上传
scp .\report.pdf pi@192.168.137.20:/home/pi/

# 下载到当前目录
scp pi@192.168.137.20:/home/pi/result.zip .

# 递归上传目录；服务器端口为 2222
scp -r -P 2222 .\project pi@server:/srv/data/
```

多个源文件可以复制到一个目录目标：

```powershell
scp .\a.txt .\b.txt user@server:/tmp/
```

## 全部命令行参数

以下参数与本机 OpenSSH 9.5p2 的 usage 对齐。

| 参数 | 作用 | 关键说明 |
| --- | --- | --- |
| `-3` | 两个远程主机之间通过本机转发 | OpenSSH 9.x 中这是默认方式；旧 SCP 协议下第二台主机不能交互询问密码。 |
| `-4` | 仅使用 IPv4 | 传递给底层 SSH。 |
| `-6` | 仅使用 IPv6 | 传递给底层 SSH。 |
| `-A` | 启用认证代理转发 | 只对可信远程主机使用。 |
| `-B` | 批处理模式 | 禁止询问密码或私钥口令；认证失败直接退出。 |
| `-C` | 启用压缩 | 传递 `-C` 给 SSH。 |
| `-c cipher` | 指定加密算法 | 通常保持默认协商。 |
| `-D path` | 指定本地 SFTP 服务程序路径 | 直接连接该程序进行调试，不通过 SSH 网络传输。 |
| `-F config` | 指定 SSH 客户端配置文件 | 适合使用独立自动化配置。 |
| `-i identity` | 指定私钥文件 | 可结合 `-o IdentitiesOnly=yes` 避免尝试其他密钥。 |
| `-J destination` | 通过跳板机 | 相当于 `ProxyJump`。 |
| `-l limit` | 限制带宽 | 单位为 Kbit/s。 |
| `-O` | 强制使用旧 SCP 协议 | 仅为旧服务器、旧通配符或旧 `~` 展开兼容；日常不用。 |
| `-o option` | 传入 SSH 配置项 | 如 `-o ConnectTimeout=10`，可多次使用。 |
| `-P port` | 指定服务器端口 | SCP 使用大写 `-P`，因为小写 `-p` 已有其他含义。 |
| `-p` | 保留修改时间、访问时间和权限模式 | Windows 文件系统不能完整表达 Unix 所有权限。 |
| `-q` | 安静模式 | 关闭进度条及 SSH 警告、诊断输出。 |
| `-R` | 远程到远程时从源主机直接连接目标 | 源主机上的 `scp` 必须能无交互认证目标主机。 |
| `-r` | 递归复制目录 | 会跟随遍历中遇到的符号链接，可能复制链接目标。 |
| `-s` | 强制使用 SFTP 协议 | 在本机 9.5p2 中可用；SFTP 已是默认值，通常无需显式指定。 |
| `-S program` | 指定加密连接程序 | 替换默认 `ssh`；程序必须理解 SSH 参数。 |
| `-T` | 禁用严格文件名检查 | 兼容特殊通配符时使用，但会信任服务器返回的文件名，降低安全性。 |
| `-v` | 输出详细调试信息 | 同时让 `scp` 和底层 `ssh` 输出调试信息。 |
| `-X option` | 调整底层 SFTP 传输参数 | 支持 `nrequests=value`、`buffer=value`。只对 SFTP 模式有效。 |

## 远程到远程复制

```powershell
# 默认：数据经过本机
scp user1@host1:/data/a.bin user2@host2:/backup/

# 源主机直接连接目标主机
scp -R user1@host1:/data/a.bin user2@host2:/backup/
```

使用 `-R` 时，源主机必须能够解析并访问目标主机，也必须自行完成认证。因此跳板、DNS 和密钥位置都可能不同于本机。

## Windows 和 PowerShell 路径规则

- 本地路径存在歧义时写成 `./file`、`.\file` 或绝对路径。
- 远程规格用冒号分隔主机和路径，例如 `user@host:/tmp/a.txt`。
- 含空格的整个参数加引号，例如 `scp ".\my file.txt" "user@host:/tmp/my file.txt"`。
- 远程路径的通配符和引号会同时受 PowerShell、SCP/SFTP 及远端语义影响；批量复杂操作优先使用 SFTP 交互/批处理。
- IPv6 主机地址用方括号，例如 `user@[2001:db8::10]:/tmp/a.txt`。
- 目标路径结尾是否有 `/` 会影响“复制到目录”还是“使用该名字”，执行前确认目标目录存在。

## `scp` 与 `sftp` 的选择

| 场景 | 推荐 |
| --- | --- |
| 已知单个源和目标，一次完成 | `scp` |
| 需要浏览目录、连续操作 | `sftp` |
| 需要脚本中执行多条文件命令 | `sftp -b` |
| 需要旧服务器兼容 | 先尝试默认 SFTP；确认必要后才用 `scp -O` |

## 风险与踩坑

- `scp -r` 会跟随符号链接，复制巨大目录或循环结构前要检查源目录；SFTP 的递归遍历则不跟随符号链接。
- `-T` 会关闭服务器返回文件名的严格校验，不应作为常规“修复”选项。
- `scp` 没有 `-V`；`-v` 是调试。套件版本用 `ssh -V` 查看。
- 旧教程常说 SCP 一定使用 SCP 协议；从 OpenSSH 9.0 起默认已经切换到 SFTP。

## 来源

- [OpenBSD Manual：scp(1)](https://man.openbsd.org/scp.1)
- [OpenSSH 9.0 Release Notes](https://www.openssh.com/txt/release-9.0)

## 相关笔记

- [[SFTP_交互式文件传输与参数]]
- [[OpenSSH_客户端配置与端口转发]]
- [[Windows_OpenSSH_故障排查]]

