# Claude Code Worker — 设计规格（Design Spec）

**版本**: 0.2（更新：加入详细环境调查结果）  
**目标**: 让用户通过一条命令启动多个隔离的 Claude Code 工作环境，每个环境对应一个 git 分支，在浏览器中与 Claude Code 交互完成任务，结果自动推送到结果分支。

---

## 1. 当前 Claude Code Web 环境基线（经实测）

> ⚠️ 注意：当前 Claude Code Web 运行环境是 **Anthropic 专有 Firecracker VM**，
> **不是** Docker 容器，Docker daemon 未运行。
> 本 spec 的 Docker 方案面向**用户本地 Linux 机器**，不是当前 Web 环境。

| 项目 | 值 | 路径 |
|---|---|---|
| 虚拟化类型 | Firecracker VM（PID 1: /process_api --firecracker-init） | — |
| 操作系统 | Ubuntu 24.04.4 LTS (Noble Numbat) | — |
| 内核 | Linux 6.18.5 x86_64 | — |
| 当前用户 | root（uid=0） | — |
| Python | 3.11.15 | `/usr/local/bin/python3` |
| Node.js | 22.22.2 | `/opt/node22/bin/node` |
| Go | 1.24.7 | `/usr/local/go/bin/go` |
| Rust | 1.94.1 | `/usr/bin/rustc` |
| Java | OpenJDK 21.0.10 | `/usr/bin/java` |
| Ruby | 3.3.6 | `/usr/bin/ruby` |
| PHP | 8.4.19 | `/usr/bin/php` |
| Git | 2.43.0 | `/usr/bin/git` |
| Docker CLI | 29.3.1（客户端，**无 daemon**） | `/usr/bin/docker` |
| tmux | 3.4 | `/usr/bin/tmux` |
| **Claude Code** | **2.1.91** | `/opt/claude-code/bin/claude`（230MB ELF 静态二进制） |
| CPU | 4 核 | — |
| 内存 | 15 GB（可用 ~14 GB） | — |
| 磁盘 | 252 GB（可用 ~30 GB） | — |

### Claude Code 命令行关键参数（v2.1.91 实测）

```
claude [prompt]                    # 交互式会话
claude -p / --print                # 非交互模式（管道友好）
claude -c / --continue             # 继续最近对话
claude --resume [session-id]       # 恢复指定会话
claude --worktree [name]           # 创建 git worktree
claude --tmux                      # 在 tmux 会话中运行
claude --dangerously-skip-permissions  # 跳过所有权限确认
claude --settings <file>           # 自定义设置文件
claude --output-format stream-json # 流式 JSON 输出
```

> ⚠️ `--bare` 参数在 v2.1.91 帮助文档中**未出现**，请勿使用。  
> API Key 认证只需设置 `ANTHROPIC_API_KEY` 环境变量即可，无需额外参数。

### apt 可用的关键包（已确认）

| 包 | 版本 | 状态 |
|---|---|---|
| ttyd | 1.7.4-1build2 | ✅ 可安装（`apt install ttyd`） |
| supervisor | 4.2.5-1ubuntu0.1 | ✅ 可安装 |
| openssh-server | 9.6p1 | ✅ 可安装 |

---

## 2. 系统架构

```
用户本地 Linux 机器（有 Docker daemon）
│
├── claude-code-worker CLI         ← 宿主机管理脚本
│   ├── start --repo --branch --purpose
│   ├── list
│   ├── stop <id>
│   ├── logs <id>
│   └── build
│
├── ~/.claude-worker/
│   ├── secrets.env                ← ANTHROPIC_API_KEY 等（chmod 600，不进 git）
│   └── workers.json               ← 运行状态记录
│
└── Docker Containers（隔离）
    ├── ccw-bug-fixing-xxx  :7700  → http://localhost:7700
    ├── ccw-feature-dev-xxx :7701  → http://localhost:7701
    └── ccw-refactor-xxx    :7702  → http://localhost:7702
```

每个容器内部：
```
容器内
├── entrypoint.sh
│   ├── 配置 git 凭证（SSH mount 或 ~/.netrc token）
│   ├── git clone --depth=50 --branch <BRANCH> <REPO>
│   ├── git checkout -b result/<PURPOSE>-<timestamp>
│   └── 启动 ttyd :7681 → 包装 claude CLI
│
└── 用户浏览器 → ttyd WebSocket → claude TUI
```

---

## 3. 技术选型说明

### 3.1 为什么用 ttyd

Claude Code 是**静态编译的 TUI 二进制**（230MB ELF），没有内置 HTTP web server。  
`ttyd 1.7.4` 在 Ubuntu 24.04 官方源中，`apt install ttyd` 直接安装，将终端包装为 WebSocket + WebGL 的浏览器界面。

### 3.2 Claude Code 安装方式

```bash
# 正确方式（native installer，v2.1.91）
DISABLE_AUTOUPDATER=1 curl -fsSL https://claude.ai/install.sh | bash
# 安装后位于 /root/.local/bin/claude 或 /opt/claude-code/bin/claude

# ❌ 已弃用，勿用
npm install -g @anthropic-ai/claude-code
```

`DISABLE_AUTOUPDATER=1` 是 Docker 场景**必要参数**，否则安装会卡住。

### 3.3 认证

只需设置环境变量，无需交互登录：
```bash
ANTHROPIC_API_KEY=sk-ant-xxxx
```
通过 `docker run --env-file` 传入，不出现在命令行历史中。

### 3.4 Git 凭证（两种，自动检测）

- **SSH**（推荐）：`~/.ssh` bind-mount 只读进容器
- **HTTPS Token**：`GIT_TOKEN` 环境变量 → 容器内写入 `~/.netrc`

---

## 4. 文件结构

```
.claude-worker/
├── Dockerfile
├── claude-code-worker              # 宿主机 CLI（Python）
├── install.sh
├── SPEC.md
└── container/
    ├── entrypoint.sh               # 容器内初始化
    └── push-and-exit.sh            # 容器内自动 commit + push
```

---

## 5. Dockerfile

```dockerfile
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive
ENV DISABLE_AUTOUPDATER=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl wget ca-certificates gnupg2 \
    git git-lfs \
    build-essential make cmake \
    python3 python3-pip python3-venv \
    nodejs npm \
    ttyd \
    vim less jq tmux \
    && rm -rf /var/lib/apt/lists/*

# Claude Code（native installer）
RUN curl -fsSL https://claude.ai/install.sh | bash

# claude 可能安装到 ~/.local/bin 或 /opt/claude-code/bin
ENV PATH="/root/.local/bin:/opt/claude-code/bin:$PATH"

COPY container/entrypoint.sh /usr/local/bin/entrypoint.sh
COPY container/push-and-exit.sh /usr/local/bin/push-and-exit.sh
RUN chmod +x /usr/local/bin/entrypoint.sh /usr/local/bin/push-and-exit.sh

WORKDIR /workspace
EXPOSE 7681

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

---

## 6. container/entrypoint.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${WORKER_REPO_URL:?需要 WORKER_REPO_URL}"
: "${WORKER_BRANCH:?需要 WORKER_BRANCH}"
: "${WORKER_PURPOSE:?需要 WORKER_PURPOSE}"
: "${WORKER_ID:?需要 WORKER_ID}"
: "${ANTHROPIC_API_KEY:?需要 ANTHROPIC_API_KEY}"

RESULT_BRANCH="result/${WORKER_PURPOSE}-$(date +%Y%m%d-%H%M%S)"
TTYD_PORT="${TTYD_PORT:-7681}"

# git 凭证
if [[ -n "${GIT_TOKEN:-}" ]]; then
    GIT_HOST=$(echo "$WORKER_REPO_URL" | sed -E 's|https?://([^/]+)/.*|\1|')
    printf "machine %s\n  login x-access-token\n  password %s\n" \
        "$GIT_HOST" "$GIT_TOKEN" > /root/.netrc
    chmod 600 /root/.netrc
fi
[[ -d /root/.ssh ]] && chmod 700 /root/.ssh 2>/dev/null || true

# 克隆并切换分支
git clone --depth=50 --branch "$WORKER_BRANCH" "$WORKER_REPO_URL" /workspace
cd /workspace
git checkout -b "$RESULT_BRANCH"
echo "$RESULT_BRANCH" > /tmp/result_branch_name
git config user.email "claude-worker@local"
git config user.name "Claude Worker ($WORKER_PURPOSE)"

# 写入任务上下文
cat > /workspace/CLAUDE.md << EOF
# Worker 上下文
- **任务**: $WORKER_PURPOSE
- **源分支**: $WORKER_BRANCH
- **结果分支**: $RESULT_BRANCH
- **Worker ID**: $WORKER_ID

完成后关闭浏览器标签，系统会自动 commit + push。
EOF

echo "[worker] 就绪，启动 ttyd :$TTYD_PORT，结果分支: $RESULT_BRANCH"

TTYD_ARGS=(--port "$TTYD_PORT" --writable --once --ping-interval 30)
[[ -n "${TTYD_CREDENTIAL:-}" ]] && TTYD_ARGS+=(--credential "$TTYD_CREDENTIAL")

# 启动 ttyd，会话结束后自动 push
ttyd "${TTYD_ARGS[@]}" \
    bash -c "cd /workspace && exec claude --dangerously-skip-permissions"
exec /usr/local/bin/push-and-exit.sh
```

---

## 7. container/push-and-exit.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

cd /workspace 2>/dev/null || { echo "[push] 无 /workspace，跳过"; exit 0; }

RESULT_BRANCH=$(cat /tmp/result_branch_name 2>/dev/null \
    || git rev-parse --abbrev-ref HEAD)

echo "[push] 会话结束，推送: $RESULT_BRANCH"

if git status --porcelain | grep -q .; then
    git add -A
    git commit -m "chore: auto-commit by claude-worker

Worker: ${WORKER_ID:-unknown}
Purpose: ${WORKER_PURPOSE:-unknown}
Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
fi

if git push origin "$RESULT_BRANCH"; then
    echo "[push] 成功: $RESULT_BRANCH"
    echo "pushed:$RESULT_BRANCH" > /tmp/push_status
else
    echo "[push] 失败！"
    echo "failed" > /tmp/push_status
    [[ -d /output ]] && tar czf /output/workspace-backup.tar.gz /workspace \
        && echo "[push] 备份已保存到 /output/workspace-backup.tar.gz"
    exit 1
fi
```

---

## 8. claude-code-worker CLI 接口

```
用法:
  claude-code-worker start --repo <url> --branch <branch> --purpose <purpose> [选项]
  claude-code-worker list
  claude-code-worker stop <id> [--force]
  claude-code-worker logs <id> [-f]
  claude-code-worker status <id>
  claude-code-worker build [--force]

# 兼容简写（与 start 等价）:
  claude-code-worker --repo <url> --branch <branch> --purpose <purpose>

start 选项:
  --ttyd-password  ttyd Basic Auth（格式: password 或 user:password）
  --backup-dir     push 失败时的本地备份目录
  --no-ssh         不挂载 ~/.ssh
  --memory         容器内存限制（默认 4g）
  --cpus           CPU 限制（默认 2.0）
```

---

## 9. 配置文件

**`~/.claude-worker/secrets.env`**（chmod 600，不进 git）：
```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxx

# HTTPS push 时使用（与 SSH 二选一）
# GIT_TOKEN=ghp_xxxx
```

---

## 10. 端口规划

| 端口范围 | 绑定地址 | 用途 |
|---|---|---|
| 7700–7799 | 127.0.0.1 | worker ttyd（自动分配，最多 100 个并行） |

---

## 11. 安全模型

| 风险 | 缓解措施 |
|---|---|
| API Key 泄露 | `--env-file`（不出现在命令行），`secrets.env` chmod 600 |
| 容器逃逸 | 不挂载 Docker socket，无特权模式 |
| 端口暴露 | 全部绑定 `127.0.0.1`，非 `0.0.0.0` |
| 代码丢失 | push 失败时自动 tar 备份到 /output |
| 资源失控 | `--memory 4g --cpus 2.0` 默认限制 |

---

## 12. 完整使用流程

```bash
# 安装
cd .claude-worker && chmod +x install.sh && ./install.sh
vim ~/.claude-worker/secrets.env   # 填入 ANTHROPIC_API_KEY

# 启动
claude-code-worker --repo git@github.com:org/repo.git \
                   --branch release/4.0.0 \
                   --purpose bug-fixing
# → 🚀 就绪! http://localhost:7700

# 并行第二个
claude-code-worker --repo git@github.com:org/repo.git \
                   --branch main \
                   --purpose feature-login
# → 🚀 就绪! http://localhost:7701

claude-code-worker list
claude-code-worker stop ccw-bug-fixing-20240403120000
```

---

## 13. 待确认事项（需在用户本地机器实测）

- [ ] `claude --dangerously-skip-permissions` 在容器内首次运行时是否会提示 OAuth？（若会，需先在 Dockerfile 内 pre-auth）
- [ ] `ttyd --once` 连接断开后是否立即触发进程退出？（影响自动 push 时机）
- [ ] native installer 在 Ubuntu 24.04 Docker 容器内安装路径：`/root/.local/bin/claude` 还是 `/opt/claude-code/bin/claude`？
- [ ] 网络代理环境变量（`HTTP_PROXY` 等）是否需要透传进容器？

---

## 14. 备注：当前 Firecracker 环境替代方案

当前 Claude Code Web 环境无 Docker daemon，若需在此环境测试并行，可用：
- **tmux** (`/usr/bin/tmux 3.4`，已安装）创建多个 tmux session
- **ttyd**（`apt install ttyd`）对外暴露各 session
- 每个 tmux session 对应一个 git worktree

但这种方式隔离性弱（共享同一文件系统和进程空间），仅适合轻量测试。
