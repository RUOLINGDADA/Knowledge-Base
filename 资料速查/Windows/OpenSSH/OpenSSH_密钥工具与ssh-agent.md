---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, SSH 密钥, ssh-agent]
技术栈: [windows, powershell, openssh]
tags: [windows, openssh, ssh密钥, ssh-agent, 理解层]
状态: 部分验证
创建日期: 2026-09-08
---

# [资料速查] OpenSSH 密钥工具与 ssh-agent

> 专题导航：[[Windows_OpenSSH_专题索引]]

## 工具关系

```text
ssh-keygen 生成私钥和公钥
      ↓
私钥留在客户端 ── ssh-add → ssh-agent 临时保管已解锁密钥
公钥放到服务器 authorized_keys
ssh-keyscan 获取服务器主机公钥 → known_hosts 用于核对服务器身份
```

用户密钥证明“我是谁”，主机密钥证明“服务器是谁”，两者不能互换。

## 生成推荐密钥

```powershell
ssh-keygen -t ed25519 -a 100 -C "设备或用途说明"
```

默认生成：

- `%USERPROFILE%\.ssh\id_ed25519`：私钥，不能发送给任何人。
- `%USERPROFILE%\.ssh\id_ed25519.pub`：公钥，可加入服务器授权列表。

推荐给私钥设置口令。自动化需要无交互时，优先使用受限服务账号、agent 或安全的秘密管理方案，不要随意创建无口令的高权限密钥。

## `ssh-keygen` 常用生成参数

| 参数 | 作用 | 说明 |
| --- | --- | --- |
| `-t type` | 密钥类型 | 推荐 `ed25519`；兼容旧系统时可用 `rsa`。`dsa` 已过时。`*-sk` 需要安全密钥硬件。 |
| `-a rounds` | 私钥口令 KDF 轮数 | 数值越高，破解更慢，解锁也更慢；只影响私钥文件保护。 |
| `-b bits` | 密钥位数 | RSA 常用 3072/4096；Ed25519 位数固定，无需设置。 |
| `-C comment` | 公钥注释 | 用于识别设备、人员或用途，不参与认证。 |
| `-f file` | 输出文件 | 避免覆盖已有默认密钥；存在同名文件时会提示。 |
| `-N passphrase` | 指定新口令 | 命令行会暴露在历史/进程信息中，交互输入更安全。 |
| `-m format` | 密钥格式 | 私钥默认 OpenSSH；导入/导出时常见 `PEM`、`PKCS8`、`RFC4716`。 |
| `-O option` | 密钥/证书附加选项 | 含义随操作模式变化，主要用于 FIDO、安全密钥和证书。 |
| `-w provider` | FIDO 安全密钥提供者 | `*-sk` 密钥使用；硬件与库需兼容。 |
| `-Z cipher` | 加密私钥的算法 | 一般使用默认值。 |
| `-q` | 安静输出 | 减少生成过程信息。 |

## `ssh-keygen` 管理与检查模式

| 命令/参数 | 用途 |
| --- | --- |
| `ssh-keygen -p -f key` | 修改私钥口令；`-P` 旧口令、`-N` 新口令、`-a` 新 KDF 轮数 |
| `ssh-keygen -y -f private_key` | 从私钥重新输出公钥 |
| `ssh-keygen -l -f key_or_pub` | 显示指纹；`-E sha256|md5` 选哈希，`-v` 显示随机艺术图 |
| `ssh-keygen -B -f key` | 显示 Bubble Babble 指纹 |
| `ssh-keygen -c -C comment -f key` | 修改密钥注释 |
| `ssh-keygen -i/-e -m format -f file` | 导入/导出其他公钥格式 |
| `ssh-keygen -F host -f known_hosts` | 在 known_hosts 中查找主机 |
| `ssh-keygen -R host -f known_hosts` | 删除指定主机的旧记录 |
| `ssh-keygen -H -f known_hosts` | 哈希 known_hosts 中未哈希的主机名 |
| `ssh-keygen -r host -f public_key` | 输出 SSHFP DNS 记录；`-g` 使用通用格式 |
| `ssh-keygen -K -w provider` | 下载 FIDO 验证器中的驻留密钥 |
| `ssh-keygen -A` | 生成缺失的服务端主机密钥；主要由管理员用于 `sshd` |
| `ssh-keygen -L -f certificate` | 显示 OpenSSH 证书内容 |

### 高级模式

| 模式 | 用途与关键参数 |
| --- | --- |
| `-M generate` / `-M screen` | 生成或筛选 Diffie-Hellman moduli；配合 `-O` 调参，日常用户不需要。 |
| `-I id -s ca_key ... file` | 用 CA 私钥签发 OpenSSH 证书；`-h` 主机证书、`-U` CA 在 agent、`-n` principals、`-V` 有效期、`-z` 序列号。 |
| `-k -f krl ...` | 创建密钥撤销列表 KRL；`-u` 更新、`-s` 指定 CA、`-z` KRL 版本。 |
| `-Q -f krl [file]` | 查询密钥是否被 KRL 撤销；此处 `-Q` 与 `ssh -Q` 含义不同。 |
| `-Y sign/verify/...` | 对普通数据进行 SSH 签名验证；关键项为命名空间 `-n`、身份 `-I`、允许签名者文件 `-f` 和签名文件 `-s`。 |

## 将公钥安装到服务器

Linux/Unix 用户通常把一整行 `.pub` 内容追加到：

```text
~/.ssh/authorized_keys
```

Windows OpenSSH Server 普通用户通常使用：

```text
C:\Users\用户名\.ssh\authorized_keys
```

Windows 管理员组账号的默认配置通常改用：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

最终路径和权限由服务器的 `sshd_config` 决定，详见 [[Windows_OpenSSH_服务器配置]]。Windows 自带客户端没有 `ssh-copy-id`；可登录后手工追加公钥，且不要覆盖已有授权。

## Windows `ssh-agent`

管理员 PowerShell 启用并启动服务：

```powershell
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
ssh-add "$env:USERPROFILE\.ssh\id_ed25519"
```

本机已确认 `ssh-agent` 服务存在，但当前为 `Stopped/Disabled`，因此直接运行 `ssh-add` 会出现 `Error connecting to agent`。

## `ssh-add` 参数

| 参数 | 作用 |
| --- | --- |
| `ssh-add [file ...]` | 将指定私钥加入 agent；无文件时尝试默认身份文件 |
| `-C` | 加载或删除时只处理证书，跳过普通密钥 |
| `-k` | 加载或删除时只处理普通私钥，跳过证书 |
| `-l` | 列出已加载密钥的指纹 |
| `-L` | 列出已加载公钥完整内容 |
| `-d file` | 从 agent 删除指定密钥 |
| `-D` | 删除 agent 中全部密钥 |
| `-t life` | 设置密钥在 agent 中的存活时间，如 `1h` |
| `-c` | 每次使用密钥前要求用户确认 |
| `-x` / `-X` | 锁定/解锁 agent |
| `-E hash` | 指纹哈希算法，常用 `sha256` 或 `md5` |
| `-q` | 成功时安静输出 |
| `-v` | 输出调试信息；最多叠加到 `-vvv` |
| `-T pubkey` | 测试与给定公钥对应的私钥能否使用 |
| `-K` | 从 FIDO 验证器加载驻留密钥 |
| `-S provider` | 指定 FIDO 提供者 |
| `-s provider` / `-e provider` | 添加/移除 PKCS#11 提供者 |
| `-H hostkey_file`、`-h destination` | 约束密钥可用于哪些目的地主机；可重复指定跳转路径 |

Windows 版具体支持项以本机 `ssh-add` 构建为准；agent 未启动时，即使是查询帮助也可能先报无法连接。

## `ssh-agent` 参数

| 参数 | 作用 |
| --- | --- |
| `-a bind_address` | 指定 agent Unix-domain 套接字地址 |
| `-D` | 前台运行 |
| `-d` | 调试模式且不派生后台进程 |
| `-s` / `-c` | 输出 Bourne shell / C shell 环境设置命令 |
| `-t life` | 设置加入密钥的默认最长生存时间 |
| `-O option` | 高级限制，如远程 PKCS#11/FIDO 加载策略 |
| `-P providers` | 允许的 PKCS#11/FIDO 提供者路径模式 |

Windows 通常通过系统服务管理 agent，不需要手工运行 `ssh-agent -s` 并导入 Unix shell 环境变量。

## `ssh-keyscan` 参数

```powershell
ssh-keyscan -T 5 -t ed25519 server.example.com
```

| 参数 | 作用 |
| --- | --- |
| `-4` / `-6` | 仅使用 IPv4 / IPv6 |
| `-c` | 请求主机证书而不是普通主机密钥 |
| `-D` | 以 SSHFP DNS 记录格式输出；重复为 `-DD` 输出通用格式 |
| `-f file` | 从文件读取主机；`-` 表示标准输入 |
| `-H` | 哈希输出中的主机名和地址 |
| `-O option` | 设置扫描选项；支持 `hashalg=sha1|sha256` |
| `-p port` | 指定远程端口；这里是小写 `-p` |
| `-T seconds` | 设置连接和读取超时 |
| `-t types` | 指定密钥类型列表，如 `ed25519,rsa,ecdsa` |
| `-v` | 调试输出 |

> [!warning]
> `ssh-keyscan` 只抓取对端声称的主机密钥，不会证明它属于正确服务器。未经独立核对就把结果写入 `known_hosts`，仍可能遭遇中间人攻击。

核对后再追加：

```powershell
ssh-keyscan -H server.example.com |
  Add-Content "$env:USERPROFILE\.ssh\known_hosts"
```

## 来源

- [OpenBSD Manual：ssh-keygen(1)](https://man.openbsd.org/ssh-keygen.1)
- [OpenBSD Manual：ssh-add(1)](https://man.openbsd.org/ssh-add.1)
- [OpenBSD Manual：ssh-agent(1)](https://man.openbsd.org/ssh-agent.1)
- [OpenBSD Manual：ssh-keyscan(1)](https://man.openbsd.org/ssh-keyscan.1)
- [Microsoft Learn：OpenSSH 密钥管理](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement)

## 相关笔记

- [[OpenSSH_客户端配置与端口转发]]
- [[Windows_OpenSSH_服务器配置]]
- [[Windows_OpenSSH_故障排查]]
