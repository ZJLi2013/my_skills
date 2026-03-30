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
  ```
- 读取节点列表示例（跳过注释行）：
  ```bash
  nodes=$(grep -v '^\s*#' .cursor/configs/gpu_nodes.list | grep -v '^\s*$')
  for host in $nodes; do
    ssh -A <user>@$host "<command>"
  done
  ```

---

## 推荐方案：SSH Agent Forwarding（默认优先）

### Step A：本地启动并装载 SSH key

```powershell
ssh-add C:\Users\<you>\ssh_keys\id_ed25519
ssh-add -l
```

### Step B：启用转发并登录远端

```bash
ssh -A <user>@<host>
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
