---
name: remote-ssh-github-auto
version: 1.0.0
author: ZJLi2013
description: 通用远端 SSH 与 GitHub 认证修复流程。优先使用 SSH Agent Forwarding（本地私钥不落地远端）实现远端 GitHub 拉取；也支持远端独立 key 方案与 pull 故障修复。
allowed-tools: [Shell]
---

# 远端 SSH + GitHub 认证修复（通用版）

## 适用场景

- 需要从本机 SSH 到远端 Linux 机器执行命令
- 远端 `git pull` 失败（常见于 HTTPS 凭证问题）
- 希望统一改为 `git@github.com:...` 方式拉取
- 需要在多个远端节点复用同一套 GitHub 身份（推荐 agent forwarding）

## 先决条件（最少）

- 本机可 SSH 到远端：`ssh <user>@<host>`
- 远端可访问 GitHub
- GitHub 账号对目标仓库有读取权限

## 节点清单与仓库路径（通用化建议）

- 节点清单统一存放于项目级配置目录（git-ignored，不入库）：
  - 项目级：`.cursor/configs/gpu_nodes.list`（首选，多 skill 共享）
  - 全局备选：`~/.cursor/configs/gpu_nodes.list`
- 首次使用：从模板复制并填入真实主机名：
  ```bash
  cp .cursor/configs/gpu_nodes.list.example .cursor/configs/gpu_nodes.list
  # 编辑 gpu_nodes.list，填入实际 SSH host alias
  ```
- 读取节点列表示例（跳过注释行）：
  ```bash
  nodes=$(grep -v '^\s*#' .cursor/configs/gpu_nodes.list | grep -v '^\s*$')
  for host in $nodes; do
    ssh -A <user>@$host "<command>"
  done
  ```
- skill 执行时把节点当作输入参数，不依赖固定仓库内文件。
- 远端仓库路径必须可配置，使用占位符：
  - `remote_repo_dir`（例如：`~/github/code-autorun`）
  - 不要在 skill 中写死某个 repo 的绝对路径。

---

## 推荐方案：SSH Agent Forwarding（默认优先）

### 原理

- 私钥只保存在本地机器
- SSH 登录远端时转发本地 agent，让远端临时使用本地 key 访问 GitHub
- 不需要在每个节点单独生成/维护 GitHub key

### Step A：本地启动并装载 SSH key

> Windows 首次启用 `ssh-agent` 通常需要“管理员 PowerShell”。

```powershell
# 管理员 PowerShell：启用并启动 OpenSSH agent
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent

# 普通终端可执行：加载本地私钥（按你的路径调整）
ssh-add C:\Users\<you>\ssh_keys\id_ed25519
ssh-add -l
```

注意：

- `ssh-add` 与 `ssh-add -l` 必须分两行执行，不要拼在一行末尾。
- 若出现 `Error connecting to agent: No such file or directory`，说明 agent 未启动或当前会话未连接到 agent。

无管理员权限时可用临时方案（Git Bash）：

```bash
eval "$(ssh-agent -s)"
ssh-add /c/Users/<you>/ssh_keys/id_ed25519
ssh-add -l
```

### Step B：启用转发并登录远端

一次性命令方式：

```bash
ssh -A <user>@<host>
```

或在本机 `~/.ssh/config` 中为目标主机开启：

```sshconfig
Host <host>
  HostName <host>
  User <user>
  IdentityFile <local_private_key_path>
  ForwardAgent yes
```

### Step C：远端验证 agent 与 GitHub

```bash
ssh-add -l
ssh -T git@github.com
```

若验证通过，再在远端仓库执行：

```bash
cd <repo_dir>
git remote set-url origin git@github.com:<owner>/<repo>.git
git pull --ff-only
```

建议将 `<repo_dir>` 作为运行参数传入，例如：

```bash
ssh -A <user>@<host> "cd <remote_repo_dir> && git pull --ff-only"
```

---

## 备选方案：远端独立 GitHub SSH Key（仅在 forwarding 不可用时）

## Step 1：检查本机 SSH（Windows 常见）

```powershell
# 仅当出现 “UNPROTECTED PRIVATE KEY FILE” 时执行
cmd /c "icacls %USERPROFILE%\.ssh\id_ed25519 /reset"
cmd /c "icacls %USERPROFILE%\.ssh\id_ed25519 /inheritance:r"
cmd /c "icacls %USERPROFILE%\.ssh\id_ed25519 /grant:r %USERNAME%:F"
cmd /c "icacls %USERPROFILE%\.ssh\id_ed25519 /remove:g Users Everyone"
```

验证：

```powershell
ssh -o BatchMode=yes <user>@<host> "whoami && hostname"
```

---

## Step 2：在远端准备 GitHub SSH Key

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
test -f ~/.ssh/id_ed25519 || ssh-keygen -t ed25519 -C "<github_email>" -f ~/.ssh/id_ed25519 -N ""
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts
cat ~/.ssh/id_ed25519.pub
```

把输出的公钥添加到 GitHub：

- Settings -> SSH and GPG keys -> New SSH key

验证远端到 GitHub：

```bash
ssh -T git@github.com
```

---

## Step 3：切换仓库 remote 到 SSH

```bash
cd <repo_dir>
git remote -v
git remote set-url origin git@github.com:<owner>/<repo>.git
git remote -v
```

拉取：

```bash
git pull --ff-only
```

---

## Step 4：快速验证

```bash
whoami
pwd
git rev-parse --abbrev-ref HEAD
git log -1 --oneline
```

---

## 常见错误与处理

- `Permission denied (publickey)`（使用 forwarding 时）
  - 本地 `ssh-add -l` 无 key，先 `ssh-add <private_key>`
  - SSH 未开启转发，改用 `ssh -A` 或配置 `ForwardAgent yes`
- `Could not open a connection to your authentication agent`
  - 本地 `ssh-agent` 未启动
- 远端 `ssh-add -l` 看不到 key
  - forwarding 未生效，检查本机 SSH config 与登录命令
- `Permission denied (publickey)`
  - 公钥未加到 GitHub，或远端使用了错误私钥
- `Host key verification failed`
  - 没有 `github.com` 的 `known_hosts` 记录，重新执行 `ssh-keyscan`
- `could not read Username for 'https://github.com'`
  - remote 仍是 HTTPS，改为 SSH URL

---

## 回滚到 HTTPS（兜底）

```bash
git remote set-url origin https://github.com/<owner>/<repo>.git
git remote -v
```

如果必须走 HTTPS，请使用 GitHub PAT（不要使用账号密码）。
