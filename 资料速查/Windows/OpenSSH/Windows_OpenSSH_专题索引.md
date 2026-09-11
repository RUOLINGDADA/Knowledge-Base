---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Windows 11, OpenSSH, 专题索引]
技术栈: [windows, openssh, ssh, sftp]
tags: [windows, openssh, ssh, sftp, 理解层]
状态: 持续维护
创建日期: 2026-09-08
---

# [资料速查] Windows OpenSSH 专题索引

## 快速开始

- [[Windows_OpenSSH_安装验证与工具总览]]：安装客户端/服务器、检查版本、认识程序和配置文件位置。
- [[SSH_远程登录命令与参数]]：远程登录、执行命令、跳板机、端口转发及全部命令行参数。
- [[SFTP_交互式文件传输与参数]]：SFTP 启动参数、交互命令、批处理和断点续传。
- [[SCP_文件复制命令与参数]]：本地与远程复制、远程互传、递归复制及全部参数。

## 认证与配置

- [[OpenSSH_密钥工具与ssh-agent]]：`ssh-keygen`、`ssh-add`、`ssh-agent`、`ssh-keyscan` 的用途和参数。
- [[OpenSSH_客户端配置与端口转发]]：`~/.ssh/config`、连接复用、代理跳转和三类转发。

## 服务与排错

- [[Windows_OpenSSH_服务器配置]]：安装和管理 `sshd`、公钥位置、防火墙、服务端参数。
- [[Windows_OpenSSH_故障排查]]：连接、认证、主机密钥、传输和 Windows 权限问题。

## 工具关系

```text
ssh ─────── 远程终端、远程命令、隧道
├─ sftp ─── 交互式/批量文件传输
├─ scp ──── 单次文件复制（OpenSSH 9.x 默认使用 SFTP）
└─ ssh-keygen / ssh-add / ssh-agent / ssh-keyscan ── 密钥与信任管理
```

## 推荐阅读顺序

1. 先按 [[Windows_OpenSSH_安装验证与工具总览#最小验证流程|最小验证流程]] 确认环境。
2. 登录服务器时看 [[SSH_远程登录命令与参数]]。
3. 传文件时按使用场景选择 [[SFTP_交互式文件传输与参数]] 或 [[SCP_文件复制命令与参数]]。
4. 经常连接时配置 [[OpenSSH_密钥工具与ssh-agent]] 和 [[OpenSSH_客户端配置与端口转发]]。
5. 要让 Windows 接受连接时看 [[Windows_OpenSSH_服务器配置]]；失败时看 [[Windows_OpenSSH_故障排查]]。

## 当前验证范围

- 已在 Windows 11 上确认客户端为 `OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2`，并核对 `ssh`、`sftp`、`scp`、`ssh-keygen` 和 `ssh-keyscan` 的本机 usage。
- 用户已实际运行 `sftp -v` 并进入参数提示；这说明程序存在，但因为缺少 `destination`，尚未建立 SFTP 连接。
- 本机 `ssh-agent` 服务存在但为“已停止/已禁用”；`sshd` 尚未安装，因此服务端步骤标记为待实际连接验证。

