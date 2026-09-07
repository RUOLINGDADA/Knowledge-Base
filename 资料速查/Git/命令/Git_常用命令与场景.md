---
分类: 资料速查
层级: 理解层
类型: 资料速查型
主题: [Git, 版本控制, 命令行]
技术栈: [git, github]
tags: [git, github, 版本控制, 命令行, 理解层]
状态: 已验证
创建日期: 待验证
更新日期: 2026-09-08
---

# [资料速查] Git 常用命令与场景

> **分类**：资料速查
> **层级**：理解层
> **标签**：#git #github #版本控制 #命令行 #理解层

> 专题导航：[[Git_专题索引]]  
> 相关笔记：[[Git_提交身份配置]]、[[Git_提交后未同步到GitHub]]

## 核心定义

Git 是一个分布式版本控制系统，用来记录文件变化、管理多个版本、创建分支，并在本地与远程仓库之间同步代码。

Git 的完整数据流可以记成：

```text
工作区 → git add → 暂存区 → git commit → 本地仓库 → git push → 远程仓库
远程仓库 → git fetch / git pull → 本地仓库与工作区
```

四个区域的区别：

| 区域 | 含义 | 常用查看命令 |
| --- | --- | --- |
| 工作区 | 正在编辑的实际文件 | `git diff` |
| 暂存区 | 下一次提交准备包含的内容 | `git diff --cached` |
| 本地仓库 | 已经 commit 的历史记录 | `git log` |
| 远程仓库 | GitHub 等服务器上的仓库 | `git fetch`、`git push` |

## 核心逻辑 / 工作流程

### 首次配置身份

```bash
git config --global user.name "你的姓名"
git config --global user.email "你的邮箱"
```

### 修改并提交

```bash
git status
git diff
git add README.md
git diff --cached
git commit -m "更新 README"
```

### 同步 GitHub

```bash
git pull --rebase origin main
git push origin main
```

### 推荐记忆方式

- `add`：挑选要提交的改动。
- `commit`：把暂存区内容保存为本地版本。
- `pull`：把远程更新拉下来并整合到本地。
- `push`：把本地提交上传到远程。
- `fetch`：只下载远程信息，不自动改动当前文件。
- `merge`：把两个分支的历史合并。
- `rebase`：把当前提交重新接到另一条分支的最新提交之后。

## 关键命令详解

### 1. `git config`：配置 Git

用于配置用户名、邮箱、默认分支、编辑器、别名和代理等信息。

#### 查看配置

```bash
# 查看所有配置及来源
git config --list --show-origin

# 查看当前生效的用户名和邮箱
git config --get user.name
git config --get user.email

# 分层查看配置
git config --system --list   # 系统级
git config --global --list   # 用户级
git config --local --list    # 当前仓库级
```

#### 配置提交身份

```bash
# 对所有仓库生效
git config --global user.name "Zhou Jinyuan"
git config --global user.email "you@example.com"

# 只对当前仓库生效
git config --local user.name "项目提交者"
git config --local user.email "project@example.com"
```

配置优先级通常为：仓库级 `local` > 用户级 `global` > 系统级 `system`。

#### 删除和修改配置

```bash
git config --global --unset user.name
git config --global --unset user.email
git config --global --replace-all user.email "new@example.com"
```

#### 常用行为配置

```bash
# 新仓库默认分支名
git config --global init.defaultBranch main

# pull 时默认使用 rebase
git config --global pull.rebase true

# 设置命令别名
git config --global alias.st "status --short --branch"
git config --global alias.lg "log --oneline --graph --decorate --all"
```

#### 常见问题

- `user.name` 和 `user.email` 只记录提交身份，不负责 GitHub 登录认证。
- 邮箱最好使用 GitHub 已验证邮箱，或使用 GitHub 提供的 `noreply` 隐私邮箱。
- 出现 `Author identity unknown` 时，优先检查 `git config --list --show-origin`。

### 2. `git init`：创建本地仓库

```bash
mkdir demo
cd demo
git init
```

在当前目录创建 `.git` 目录。不要在已经存在 Git 仓库的子目录中重复执行，除非确实需要嵌套仓库。

```bash
git init -b main
```

创建仓库并直接指定初始分支为 `main`。

### 3. `git clone`：复制远程仓库

```bash
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git
```

常用参数：

```bash
# 指定本地目录名
git clone URL local-folder

# 只克隆最近的提交，节省时间和空间
git clone --depth 1 URL

# 克隆指定分支
git clone --branch dev URL
```

### 4. `git status`：查看状态

```bash
git status
git status --short
git status --short --branch
```

短格式含义：

| 状态 | 含义 |
| --- | --- |
| `M file` | 工作区修改，尚未暂存 |
| `M  file` | 已暂存修改 |
| `?? file` | 未跟踪的新文件 |
| `A  file` | 新文件已暂存 |
| `D  file` | 文件已删除并暂存 |
| `R  old -> new` | 文件已重命名 |
| `ahead N` | 本地有 N 个提交尚未 push |
| `behind N` | 远程有 N 个提交尚未 pull |

### 5. `git add`：加入暂存区

```bash
# 添加指定文件
git add README.md

# 添加指定目录
git add src/

# 添加当前目录全部变化
git add .

# 交互式选择部分修改
git add -p
```

撤销暂存但保留文件修改：

```bash
git restore --staged README.md
```

旧版本 Git 也常用：

```bash
git reset HEAD README.md
```

### 6. `git diff`：查看差异

```bash
# 工作区相对暂存区的修改
git diff

# 暂存区相对最近一次 commit 的修改
git diff --cached
git diff --staged

# 某个文件的修改
git diff -- README.md

# 两个提交之间的差异
git diff commit1 commit2

# 比较当前分支和远程主分支
git diff main origin/main

# 只看文件统计
git diff --stat
```

常见用法：

```bash
# 查看某次提交改了什么
git show --stat <commit-id>
git show <commit-id>

# 查看两个分支各自多了什么
git diff main...feature
```

`A...B` 会比较两个分支共同祖先与 `B` 的差异，适合查看某个功能分支相对主分支引入的改动。

### 7. `git commit`：创建本地提交

```bash
git commit -m "更新 README"
```

常用参数：

```bash
# 暂存已跟踪文件的修改并提交，新文件仍需 git add
git commit -am "修复配置"

# 修改最近一次提交信息
git commit --amend -m "新的提交信息"

# 把当前暂存内容补充到上一次提交
git commit --amend --no-edit

# 创建空提交，常用于触发 CI
git commit --allow-empty -m "触发构建"
```

注意：已经推送到公共远程仓库的提交，不要随意 `--amend`，因为它会改变提交 ID，可能导致协作者历史冲突。

### 8. `git log`：查看提交历史

```bash
git log
git log --oneline
git log --oneline --graph --decorate --all
git log -5
```

按文件查看：

```bash
git log -- README.md
git log -p -- README.md
git log --follow -- README.md
```

按作者、时间或提交信息筛选：

```bash
git log --author="Zhou"
git log --since="2026-01-01"
git log --grep="README"
```

查看某个提交：

```bash
git show <commit-id>
git show --name-only <commit-id>
git show HEAD~1
```

### 9. `git restore`：恢复文件或取消暂存

```bash
# 丢弃工作区对文件的修改，恢复到暂存区版本
git restore README.md

# 将文件恢复到最近一次提交
git restore --source=HEAD -- README.md

# 取消暂存，保留工作区修改
git restore --staged README.md

# 同时取消暂存并丢弃工作区修改
git restore --staged --worktree README.md
```

最后一个命令会删除未提交的修改，执行前必须确认内容不再需要。

### 10. `git reset`：移动 HEAD 或撤销提交

```bash
# 撤销 commit，保留修改在暂存区
git reset --soft HEAD~1

# 撤销 commit，保留修改在工作区
git reset --mixed HEAD~1

# 撤销 commit，并丢弃相关修改
git reset --hard HEAD~1
```

常见场景：

```bash
# 撤销误 add 的文件，不删除文件内容
git reset HEAD -- README.md

# 本地提交尚未 push，想重新整理最近两次提交
git reset --soft HEAD~2
```

`--hard` 可能永久丢失未保存内容，不要用于不确定的文件。

### 11. `git branch`：管理分支

```bash
# 查看本地分支
git branch

# 查看所有本地和远程分支
git branch -a

# 查看分支及其跟踪关系
git branch -vv

# 创建分支
git branch feature/login

# 删除已合并分支
git branch -d feature/login

# 强制删除未合并分支
git branch -D feature/login

# 重命名当前分支
git branch -m new-name
```

### 12. `git switch`：切换和创建分支

```bash
# 切换已有分支
git switch main

# 创建并切换到新分支
git switch -c feature/readme

# 从远程分支创建本地跟踪分支
git switch --track origin/feature/readme
```

旧命令：

```bash
git checkout main
git checkout -b feature/readme
```

`switch` 更专注于分支切换，建议新项目优先使用它。

### 13. `git merge`：合并分支

```bash
git switch main
git pull origin main
git merge feature/readme
```

常用参数：

```bash
# 只允许快进合并
git merge --ff-only feature/readme

# 即使可以快进也创建合并提交
git merge --no-ff feature/readme

# 发生冲突后中止合并
git merge --abort
```

冲突处理流程：

```bash
git status
# 手动编辑冲突文件并删除 <<<<<<< ======= >>>>>>> 标记
git add 冲突文件
git commit
```

### 14. `git rebase`：整理提交历史

```bash
# 将当前分支接到 main 最新提交之后
git switch feature/readme
git rebase main
```

交互式整理最近提交：

```bash
git rebase -i HEAD~3
```

冲突处理：

```bash
git status
# 解决冲突后
git add 冲突文件
git rebase --continue

# 放弃本次 rebase
git rebase --abort
```

不要对已经被多人使用的公共分支随意 rebase，因为它会重写历史。

### 15. `git remote`：管理远程仓库

```bash
# 查看远程地址
git remote -v

# 查看远程详细信息
git remote show origin

# 添加远程仓库
git remote add origin https://github.com/user/repo.git

# 修改远程地址
git remote set-url origin https://github.com/user/new-repo.git

# 删除远程别名
git remote remove origin
```

`origin` 只是默认远程名称，不是固定关键字，可以替换成其他名称。

### 16. `git fetch`：获取远程更新但不合并

```bash
git fetch origin
git fetch --all --prune
```

- 下载远程的新提交和分支引用。
- 不会直接修改当前工作区文件。
- 适合在合并前先检查远程变化。

```bash
git fetch origin
git log --oneline main..origin/main
git diff main origin/main
```

### 17. `git pull`：拉取并整合远程更新

```bash
git pull origin main
git pull
```

`git pull` 通常相当于：

```bash
git fetch
git merge
```

使用 rebase：

```bash
git pull --rebase origin main
```

只允许快进：

```bash
git pull --ff-only origin main
```

常见误区：

- `pull` 是远程到本地，不是本地到远程。
- 本地显示 `ahead 1` 时，应该执行 `git push`。
- `Already up to date` 只说明远程没有需要拉取的新提交，不代表本地提交已经上传。

### 18. `git push`：上传本地提交

#### 基本语法

```bash
git push <远程名> <本地分支>:<远程分支>
```

最常用：

```bash
git push origin main
```

如果当前分支已经设置 upstream：

```bash
git push
```

#### 首次推送并建立跟踪关系

```bash
git push -u origin main
```

`-u` 等价于 `--set-upstream`，之后在该分支可以直接使用 `git push` 和 `git pull`。

#### 推送当前分支

```bash
git push origin HEAD
```

推送到指定名称的远程分支：

```bash
git push origin HEAD:feature/readme
```

#### 推送标签

```bash
git push origin v1.0.0

git push origin --tags
```

#### 删除远程分支或标签

```bash
git push origin --delete feature/old

git push origin --delete v1.0.0
```

#### 安全强制推送

```bash
git push --force-with-lease origin feature/readme
```

`--force-with-lease` 会在远程分支没有被别人更新时才覆盖，通常比 `--force` 安全。

```bash
# 高风险：直接覆盖远程分支
git push --force origin feature/readme
```

不要在共享的 `main` / `master` 分支上随意使用强制推送。

#### 推送前检查

```bash
git status --short --branch
git log --oneline origin/main..HEAD
git diff origin/main...HEAD
git push origin main
```

状态判断：

| 状态 | 含义 | 下一步 |
| --- | --- | --- |
| `ahead 1` | 本地多一个提交 | `git push` |
| `behind 1` | 远程多一个提交 | `git pull` 或先 `fetch` |
| `ahead 1, behind 1` | 双方都有新提交 | 先整合，再 push |
| 无 ahead/behind | 本地和远程提交一致 | 无需同步 |

### 19. `git tag`：标记版本

```bash
# 查看标签
git tag

# 创建轻量标签
git tag v1.0.0

# 创建带说明的标签
git tag -a v1.0.0 -m "第一个稳定版本"

# 查看标签信息
git show v1.0.0

# 删除本地标签
git tag -d v1.0.0
```

### 20. `git stash`：临时保存未提交修改

```bash
# 保存当前修改
git stash push -m "临时保存 README 修改"

# 包含未跟踪文件
git stash push -u -m "保存全部修改"

# 查看 stash 列表
git stash list

# 恢复最近一次 stash，保留 stash 记录
git stash apply

# 恢复并删除对应 stash 记录
git stash pop

# 删除某条 stash
git stash drop stash@{0}

# 清空全部 stash
git stash clear
```

### 21. `git rm` 和 `git mv`：删除与移动

```bash
# 删除文件并加入暂存区
git rm old.md

# 只从 Git 跟踪中移除，保留本地文件
git rm --cached config.local

# 移动或重命名
git mv old-name.md new-name.md
```

### 22. `.gitignore`：忽略不应提交的文件

`.gitignore` 不是命令，而是仓库根目录中的规则文件：

```gitignore
# Python 缓存
__pycache__/
*.pyc

# 环境配置
.env
*.local

# 构建产物
build/
dist/

# 编辑器目录
.vscode/
.idea/
```

已经被 Git 跟踪的文件不会因为后来加入 `.gitignore` 自动消失，需要：

```bash
git rm --cached filename
```

### 23. `git clean`：清理未跟踪文件

```bash
# 先预览将删除的内容
git clean -n

# 删除未跟踪文件
git clean -f

# 同时删除未跟踪目录
git clean -fd
```

`git clean` 可能删除未保存文件，优先使用 `-n` 预览，确认无误后再执行。

### 24. `git cherry-pick`：复制指定提交

```bash
git switch main
git cherry-pick <commit-id>
```

冲突处理：

```bash
git add 冲突文件
git cherry-pick --continue

git cherry-pick --abort
```

适合把某个分支中的单个修复提交移植到当前分支。

### 25. `git revert`：用新提交撤销旧提交

```bash
git revert <commit-id>
```

与 `reset` 的区别：

- `revert` 保留历史，并创建一个反向修改的新提交，适合已经推送的公共分支。
- `reset` 移动分支指针，可能重写本地历史，适合尚未推送的提交整理。

### 26. `git blame`：查看每一行的最后修改者

```bash
git blame README.md
git blame -L 10,30 README.md
```

适合追踪一行配置或代码是什么时候、由哪个提交修改的。

### 27. `git bisect`：二分定位引入 Bug 的提交

```bash
git bisect start
git bisect bad
git bisect good <已知正常的提交>
```

Git 会切换到中间提交，测试后标记：

```bash
git bisect good
git bisect bad
```

结束：

```bash
git bisect reset
```

### 28. `git reflog`：找回丢失的提交引用

```bash
git reflog
```

当误执行 `reset`、`rebase` 或切错分支时，可以从 reflog 找到旧的 HEAD：

```bash
git reset --hard <reflog中的commit-id>
```

### 29. `git archive`：导出指定版本

```bash
git archive --format=zip --output=release.zip HEAD
git archive --format=tar.gz --output=release.tar.gz v1.0.0
```

适合导出不包含 `.git` 目录的源码压缩包。

## Git 命令分类与自助查询

Git 命令数量很多，日常开发不需要一次记住全部命令。可以先掌握上面的核心命令，再使用帮助系统查询特殊场景。

```bash
# 查看 Git 总帮助
git help

# 查看所有可用命令
git help -a

# 查看某个命令的帮助
git help push
git push --help

# 查看简短用法
git push -h
```

常见命令可以按用途分为：

| 类别 | 命令 |
| --- | --- |
| 创建仓库 | `init`、`clone` |
| 配置与检查 | `config`、`status`、`version` |
| 文件与提交 | `add`、`commit`、`restore`、`reset`、`rm`、`mv` |
| 查看历史 | `log`、`show`、`diff`、`blame`、`reflog` |
| 分支操作 | `branch`、`switch`、`merge`、`rebase`、`cherry-pick` |
| 远程同步 | `remote`、`fetch`、`pull`、`push` |
| 版本发布 | `tag`、`archive` |
| 临时与清理 | `stash`、`clean` |
| 历史修复 | `revert`、`bisect` |
| 高级维护 | `gc`、`fsck`、`repack`、`filter-repo` |

`git gc`、`git fsck` 等命令会影响本地仓库维护，不建议在不了解作用时直接使用；涉及历史批量改写时，应先备份仓库。

## 常见场景命令组合

### 场景一：首次把本地项目上传到 GitHub

```bash
git init -b main
git add .
git commit -m "初始化项目"
git remote add origin https://github.com/user/repo.git
git push -u origin main
```

### 场景二：日常修改并同步

```bash
git status
git diff
git add .
git diff --cached
git commit -m "描述本次修改"
git pull --rebase origin main
git push origin main
```

### 场景三：只想查看远程有什么更新

```bash
git fetch origin
git log --oneline main..origin/main
git diff main origin/main
```

### 场景四：本地有提交，GitHub 没有变化

```bash
git status --short --branch
# 如果看到 ahead N
git push origin main
```

### 场景五：本地和远程都产生了提交

```bash
git fetch origin
git status --short --branch
git pull --rebase origin main
# 解决冲突后
git add 冲突文件
git rebase --continue
git push origin main
```

### 场景六：撤销尚未推送的最近一次提交

```bash
git reset --soft HEAD~1
```

修改或补充文件后重新：

```bash
git add .
git commit -m "整理后的提交"
```

### 场景七：撤销已经推送的错误提交

```bash
git revert <错误提交ID>
git push origin main
```

不要优先使用 `reset --hard` 和 `push --force`，除非明确知道会重写远程历史。

## 常见误区

- `git commit` 只提交到本地，不会自动上传 GitHub。
- `git pull` 是拉取，不是推送；本地领先远程时要执行 `git push`。
- `git add .` 只负责放入暂存区，不等于已经 commit。
- `git diff` 默认不显示已经暂存的内容，查看暂存区要用 `git diff --cached`。
- 修改 `user.name` 和 `user.email` 不会解决 GitHub 权限、Token 或 SSH 认证问题。
- `git fetch` 不会自动把远程文件合并到当前分支；需要后续 `merge`、`rebase` 或 `pull`。
- `git reset --hard`、`git clean -f`、`git push --force` 都可能造成不可逆损失。
- `.gitignore` 对已被跟踪的文件不生效，需要使用 `git rm --cached` 清除跟踪状态。
- 分支名、远程名和提交 ID 都区分上下文；执行命令前先用 `git status`、`git branch -vv`、`git remote -v` 确认当前对象。

## 原理 / 因果点睛

Git 的核心不是直接同步文件，而是保存提交对象和分支指针。`commit` 在本地生成一组不可变的历史对象，`push` 将这些对象和分支指针发送到远程，`pull` 则先获取远程对象，再通过合并或变基让当前分支包含远程历史。命令显示的 `ahead`、`behind`，本质上是在比较两个分支指针之间的提交图关系。
