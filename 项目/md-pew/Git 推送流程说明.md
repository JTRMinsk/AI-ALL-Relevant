# Git Push Walkthrough / Git 推送流程说明

本文记录 md-pew 第一次上传到 GitHub 的实际过程。它不是完整 Git 教程，而是一次真实任务的复盘：做了哪些判断，跑了哪些命令，每条命令是什么意思，以及遇到网络问题时怎么定位。

This document records the first md-pew upload to GitHub. It is not a full Git tutorial; it is a walkthrough of the actual task: decisions made, commands run, what each command means, and how the network issue was diagnosed.

## 最终结果 / Final Result

- 本地分支：`main`
- Local branch: `main`
- 远程仓库：`https://github.com/JTRMinsk/md-pew.git`
- Remote repository: `https://github.com/JTRMinsk/md-pew.git`
- 首次提交：`942c294 Initial md-pew project`
- Initial commit: `942c294 Initial md-pew project`
- 推送结果：`main` 已上传并跟踪 `origin/main`
- Push result: `main` was uploaded and now tracks `origin/main`

## 决策过程 / Decision Process

### 1. 先确认仓库状态

Before changing anything, the local repository state was checked.

The goal was to understand whether the project was already a Git repository, whether it had commits, which branch was active, and whether a remote repository was configured.

这样做可以避免误推到错误分支、错误远程，或者把未确认的文件直接上传。

### 2. 先做本地提交，再推送远程

GitHub push 的对象是 Git commit，不是普通文件夹复制。

The project had to become a local commit first. After that, the commit could be pushed to GitHub.

This keeps the upload reproducible: GitHub receives a precise snapshot with a commit message and author metadata.

### 3. 作者信息只设置在当前仓库

The commit author was configured in this repository only:

```powershell
git config user.name "Salim Sehng"
git config user.email "salimsheng93@gmail.com"
```

This records who authored the commit. It is not the same thing as GitHub login.

这不会把 GitHub 密码或 token 写进仓库，也不会影响其他项目的全局 Git 配置。

### 4. 不在对话里处理 GitHub token

GitHub push authentication should be handled by Git Credential Manager, browser login, SSH keys, or a Personal Access Token entered locally.

The token should not be pasted into chat.

本机 Git 已配置：

```text
credential.helper = manager
```

That means HTTPS push can use Git Credential Manager when the network reaches GitHub.

### 5. 远程地址由项目名推断

The user provided the GitHub account page:

```text
https://github.com/JTRMinsk/
```

The project name is `md-pew`, so the inferred repository URL was:

```text
https://github.com/JTRMinsk/md-pew.git
```

This URL was added as `origin`.

### 6. 网络失败先按网络问题处理

The first push attempts failed before authentication:

```text
Failed to connect to github.com port 443
```

Because the error happened at the network layer, it was not treated as a wrong username, wrong password, or missing token problem.

After the network mode was changed, connectivity checks succeeded, and the next push worked.

## 命令记录 / Commands Run

### 查看仓库状态 / Check Repository Status

```powershell
git status --short --branch
```

Shows the current branch and a compact list of changed files.

用于确认当前在 `main` 分支，并看到当时文件都还未提交。

```powershell
git remote -v
```

Lists configured remotes.

用于确认一开始没有远程仓库，后来确认 `origin` 指向 GitHub。

```powershell
git branch --show-current
```

Prints the active branch name.

用于确认当前分支是 `main`。

```powershell
git log --oneline -5
```

Shows recent commits in compact form.

当时这个命令失败了，因为仓库还没有任何提交。这反而确认了项目需要创建首次提交。

### 检查文件和语法 / Check Files And Syntax

```powershell
node -e "const fs=require('fs'); const html=fs.readFileSync('md-pew.html','utf8'); const m=html.match(/<script>([\s\S]*)<\/script>/); if(!m) throw new Error('script tag not found'); new Function(m[1]); console.log('script syntax ok');"
```

Extracts the JavaScript inside `md-pew.html` and checks whether it parses.

This was a lightweight sanity check before committing the project.

```powershell
Get-ChildItem -Recurse -Force | Where-Object { -not $_.PSIsContainer -and $_.FullName -notmatch '\\.git\\' } | Select-Object FullName,Length
```

Lists project files outside `.git`.

用于确认即将提交的是项目文件，没有明显的构建产物或临时文件。

### 暂存文件 / Stage Files

```powershell
git add -A
```

Stages all current files, including new files, modified files, and deletions.

第一次运行时遇到了 `.git/index.lock` 权限问题；获得文件系统权限后重新运行成功。

### 配置提交作者 / Configure Commit Author

```powershell
git config user.name "Salim Sehng"
git config user.email "salimsheng93@gmail.com"
```

Sets the local repository commit author.

This is commit metadata, not GitHub authentication.

### 创建首次提交 / Create Initial Commit

```powershell
git commit -m "Initial md-pew project"
```

Creates the first Git commit from the staged project files.

The resulting commit was:

```text
942c294 Initial md-pew project
```

### 添加远程仓库 / Add Remote Repository

```powershell
git remote add origin https://github.com/JTRMinsk/md-pew.git
```

Adds the GitHub repository URL under the conventional remote name `origin`.

After this, Git knows where to push.

### 推送到 GitHub / Push To GitHub

```powershell
git push -u origin main
```

Pushes local `main` to `origin/main`.

The `-u` option sets upstream tracking, so future pushes can usually be shortened to:

```powershell
git push
```

The early push attempts failed because the command environment could not connect to `github.com:443`.

### 诊断网络 / Diagnose Network

```powershell
Test-NetConnection github.com -Port 443
```

Checks whether the machine can open a TCP connection to GitHub over HTTPS.

After the network mode changed, this succeeded:

```text
TcpTestSucceeded : True
```

```powershell
git ls-remote https://github.com/JTRMinsk/md-pew.git
```

Checks whether Git can reach the remote repository.

The command succeeded with no refs printed, which is expected for a newly created empty repository.

### 成功推送 / Successful Push

```powershell
git push -u origin main
```

Final result:

```text
branch 'main' set up to track 'origin/main'.
To https://github.com/JTRMinsk/md-pew.git
 * [new branch]      main -> main
```

This means the local `main` branch was uploaded to GitHub and linked to `origin/main`.

## 之后如何继续 / Future Workflow

After future edits:

```powershell
git status --short --branch
git add -A
git commit -m "Describe the change"
git push
```

建议每次提交前先确认：

- `CHANGELOG.md` 已按中英双语更新。
- `README.md` 如涉及功能变化也已同步。
- `md-pew.html` 的 JavaScript 语法检查通过。

Recommended before each commit:

- Update `CHANGELOG.md` bilingually.
- Update `README.md` when user-facing behavior changes.
- Run the lightweight JavaScript syntax check for `md-pew.html`.
