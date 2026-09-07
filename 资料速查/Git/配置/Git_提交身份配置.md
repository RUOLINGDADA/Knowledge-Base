---
分类: 资料速查
层级: 理解层
类型: 问题排查型
主题: [Git, 提交身份, 配置排查]
技术栈: [git]
tags: [git, 提交配置, 身份认证, 问题排查, 理解层]
状态: 已验证
创建日期: 待验证
更新日期: 2026-09-08
---

# [资料速查] Git 提交身份配置异常

> **分类**：资料速查
> **层级**：理解层
> **标签**：#git #github #版本控制 #提交配置 #理解层

> 专题导航：[[Git_专题索引]]  
> 相关笔记：[[Git_常用命令与场景]]、[[Git_提交后未同步到GitHub]]

## 问题现象

执行 `git commit` 时，Git 无法识别提交者身份，终端提示：

```text
Author identity unknown

*** Please tell me who you are.

fatal: unable to auto-detect email address
```

本次环境中的具体错误为：

```text
fatal: unable to auto-detect email address (got 'zhoujinyuan@DESKTOP-QMC7OLJ.(none)')
```

## 排查过程

1. 检查当前仓库是否已经配置用户名：

   ```bash
   git config --get user.name
   ```

2. 检查当前仓库是否已经配置邮箱：

   ```bash
   git config --get user.email
   ```

3. 两条命令均没有返回有效值，确认 Git 缺少提交身份配置。

4. Git 提交不仅需要提交内容，还需要在提交对象中写入作者姓名和邮箱，因此配置缺失会在 `commit` 阶段直接终止。

## 最终解决方案

### 方案一：配置所有仓库通用的身份

适合个人电脑上的常规开发环境：

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

例如：

```bash
git config --global user.name "Zhou Jinyuan"
git config --global user.email "you@example.com"
```

### 方案二：只配置当前仓库

适合不同项目使用不同身份，或不希望修改全局配置：

```bash
git config user.name "项目提交者姓名"
git config user.email "项目提交邮箱"
```

### 验证配置

```bash
git config --global --get user.name
git config --global --get user.email
```

如果使用仓库级配置，则执行：

```bash
git config --local --get user.name
git config --local --get user.email
```

确认能输出姓名和邮箱后，重新执行：

```bash
git commit -m "更新知识库说明"
```

## 核心代码 / 配置片段

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
git config --global --list
```

## 踩坑记录

- **坑点1：GitHub 登录用户名不等于 Git 提交身份。** GitHub 账号登录成功，并不会自动为本地 Git 配置 `user.name` 和 `user.email`。
- **坑点2：邮箱建议使用 GitHub 已验证邮箱。** 如果不希望公开真实邮箱，可以在 GitHub 的 Email settings 中使用 `noreply` 隐私邮箱，否则提交可能无法关联到 GitHub 账号或会暴露个人邮箱。
- **坑点3：`--global` 和当前仓库配置的优先级不同。** 当前仓库中的 `user.name` / `user.email` 会覆盖全局配置；遇到配置看似正确但仍报错时，先检查仓库级配置。
- **坑点4：姓名和邮箱只影响提交元数据。** 它们不会修改远程仓库权限；如果之后出现 `Permission denied` 或认证失败，那是 SSH、Token 或远程账号权限问题。
- **快速判断：** `git config --list --show-origin` 可以同时查看配置值以及配置来自哪个文件，适合排查配置冲突。

## 原理 / 因果点睛

Git 的每个提交对象都会记录作者和提交者信息，至少包括姓名和邮箱；本地没有可用配置时，Git 无法生成完整的提交对象，所以在提交创建前终止。`--global` 写入用户级配置，`--local` 写入当前仓库配置，后者优先级更高。
