# Claude Code Worker — 设计规格（Design Spec）

**版本**: 0.1  
**目标**: 让用户通过一条命令启动多个隔离的 Claude Code 工作环境，每个环境对应一个 git 分支，在浏览器中与 Claude Code 交互完成任务，结果自动推送到结果分支。

---

## 1. 当前环境基线（经实测）

构建 Docker 镜像的基础参考当前运行环境：

| 项目 | 值 |
|---|---|
| 基础镜像 | Ubuntu 24.04.4 LTS (Noble Numbat) |
| 内核 | Linux 6.18.5 x86_64 |
| Python | 3.11.15（`/usr/local/bin/python3`） |
| Node.js | 22.22.2（`/opt/node22/bin/node`） |
| Go | 1.24.7（`/usr/local/go/bin/go`） |
| Rust | 1.94.1（`/root/.cargo/bin/rustc`） |
| Java | 已安装（JDK） |
| Ruby | 已安装 |
| PHP | 8.4 |
| Git | 2.43.0 |
| Docker | 29.3.1（宿主机可用） |
| Claude Code | 2.1.91（`/root/.local/bin/claude`，native installer） |
| CPU | 4 核 |
| 内存 | 15 GB |
| 磁盘 | 252 GB，可用 ~31 GB |

**关键约束**：
- 网络出站通过 Anthropic 托管代理过滤，仅允许白名单域名（PyPI、npm、GitHub、crates.io、apt.ubuntu.com 等已在白名单内）
- 宿主机已有 Docker socket 和 Docker CLI

---

## 2. 系统架构

```
宿主机 (用户的 Linux 电脑)
│
├── claude-code-worker CLI          ← 宿主机上运行的管理脚本
│   ├── start --repo --branch --purpose
│   ├── list
│   ├── stop <id>
│   ├── logs <id>
│   └── build
│
├── ~/.claude-worker/
│   ├── secrets.env                 ← API Key 等敏感信息（chmod 600，不进 git）
│   └── workers.json                ← 运行中 worker 的状态记录
│
└── Docker Containers (隔离)
    ├── ccw-bug-fixing-xxx  :7700  → 浏览器访问 http://localhost:7700
    ├── ccw-feature-dev-xxx :7701  → 浏览器访问 http://localhost:7701
    └── ccw-refactor-xxx    :7702  → 浏览器访问 http://localhost:7702
```

每个容器内部：
```
容器内
├── entrypoint.sh（初始化）
│   ├── 配置 git 凭证（SSH mount 或 HTTPS token）
│   ├── git clone --depth=50 --branch <BRANCH> <REPO>
│   ├── git checkout -b result/<PURPOSE>-<timestamp>
│   └── 启动 ttyd → 包装 claude CLI
│
└── ttyd --port 7681
    └── bash -c "cd /workspace && claude --dangerously-skip-permissions"
        ← 用户通过浏览器操作这个终端
```

---

## 3. 技术选型说明

### 3.1 为什么用 ttyd 而不是内置 web 模式

Claude Code **没有**内置 HTTP web server 模式——它是一个 TUI 二进制程序。  
`ttyd` 是 Ubuntu 24.04 apt 官方源中的 `1.7.4` 版本，`apt install ttyd` 即可安装，无需编译。它将任意终端命令包装成 WebSocket + WebGL 的浏览器终端，是最简单且已在生产环境验证的方案。

### 3.2 Claude Code 安装方式

使用官方 native installer（npm 方式已弃用）：
```bash
DISABLE_AUTOUPDATER=1 curl -fsSL https://claude.ai/install.sh | bash
```
`DISABLE_AUTOUPDATER=1` 防止容器内安装卡住，这是 Docker 场景的必要参数。

### 3.3 认证方式

- **API Key**（推荐）：通过 `ANTHROPIC_API_KEY` 环境变量传入，配合 `claude --bare` 参数强制使用 API Key 认证，跳过 OAuth 流程。
- **OAuth 登录**（备选）：需要交互式浏览器认证，不适合容器化场景。

### 3.4 Git 凭证

两种方式，自动检测：
- **SSH**（推荐）：将宿主机 `~/.ssh` bind-mount 进容器（只读）
- **HTTPS Token**：通过 `GIT_TOKEN` 环境变量传入，容器内写入 `~/.netrc`

---

## 4. 文件结构

```
.claude-worker/                     ← 放在项目根目录或独立仓库
├── Dockerfile                      # 容器镜像定义
├── claude-code-worker              # 宿主机 CLI（Python 脚本）
├── install.sh                      # 一键安装脚本
├── SPEC.md                         # 本文档
└── container/
    ├── entrypoint.sh               # 容器内初始化脚本
    └── push-and-exit.sh            # 容器内自动 commit+push 脚本
```

---

## 5. Dockerfile

```dockerfile
FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive
ENV DISABLE_AUTOUPDATER=1

# 基础工具
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

# 确保 claude 在 PATH 中
ENV PATH="/root/.local/bin:$PATH"

# 容器内脚本
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

# 必要参数（由 docker run --env 传入）
: "${WORKER_REPO_URL:?需要 WORKER_REPO_URL}"
: "${WORKER_BRANCH:?需要 WORKER_BRANCH}"
: "${WORKER_PURPOSE:?需要 WORKER_PURPOSE}"
: "${WORKER_ID:?需要 WORKER_ID}"
: "${ANTHROPIC_API_KEY:?需要 ANTHROPIC_API_KEY}"

RESULT_BRANCH="result/${WORKER_PURPOSE}-$(date +%Y%m%d-%H%M%S)"
TTYD_PORT="${TTYD_PORT:-7681}"

# ── 1. Git 凭证配置 ──────────────────────────────────────────────
if [[ -n "${GIT_TOKEN:-}" ]]; then
    GIT_HOST=$(echo "$WORKER_REPO_URL" | sed -E 's|https?://([^/]+)/.*|\1|')
    printf "machine %s\n  login x-access-token\n  password %s\n" \
        "$GIT_HOST" "$GIT_TOKEN" > /root/.netrc
    chmod 600 /root/.netrc
fi

# SSH bind-mount 时修正权限
[[ -d /root/.ssh ]] && chmod 700 /root/.ssh 2>/dev/null || true

# ── 2. 克隆并切换分支 ────────────────────────────────────────────
echo "[worker] 克隆仓库: $WORKER_REPO_URL @ $WORKER_BRANCH"
git clone --depth=50 --branch "$WORKER_BRANCH" "$WORKER_REPO_URL" /workspace
cd /workspace

git checkout -b "$RESULT_BRANCH"
echo "$RESULT_BRANCH" > /tmp/result_branch_name

git config user.email "claude-worker@local"
git config user.name "Claude Worker ($WORKER_PURPOSE)"

# ── 3. 写入任务上下文到 CLAUDE.md ───────────────────────────────
cat > /workspace/CLAUDE.md << EOF
# Worker 上下文

- **任务**: $WORKER_PURPOSE
- **源分支**: $WORKER_BRANCH
- **结果分支**: $RESULT_BRANCH
- **Worker ID**: $WORKER_ID

完成任务后，提交代码并关闭浏览器标签即可（系统会自动 push）。
或手动执行: push-and-exit.sh
EOF

echo "[worker] 就绪，启动 ttyd on :$TTYD_PORT"
echo "[worker] 结果分支: $RESULT_BRANCH"

# ── 4. 启动 ttyd 包装 claude ─────────────────────────────────────
TTYD_ARGS=(--port "$TTYD_PORT" --writable --once --ping-interval 30)
[[ -n "${TTYD_CREDENTIAL:-}" ]] && TTYD_ARGS+=(--credential "$TTYD_CREDENTIAL")

ttyd "${TTYD_ARGS[@]}" \
    bash -c "cd /workspace && exec claude --dangerously-skip-permissions --bare"

# ttyd 退出后自动 push
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

echo "[push] 会话结束，推送分支: $RESULT_BRANCH"

# 有未提交变更则自动 commit
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
    # 备份到 /output（如果有 bind-mount）
    [[ -d /output ]] && tar czf /output/workspace-backup.tar.gz /workspace \
        && echo "[push] 紧急备份已保存到 /output/workspace-backup.tar.gz"
    exit 1
fi
```

---

## 8. claude-code-worker CLI 接口

```
用法:
  claude-code-worker start --repo <url> --branch <branch> --purpose <purpose> [选项]
  claude-code-worker list
  claude-code-worker stop <worker-id> [--force]
  claude-code-worker logs <worker-id> [-f]
  claude-code-worker status <worker-id>
  claude-code-worker build [--force]

兼容简写（第一个参数为 --xxx 时自动视为 start）:
  claude-code-worker --repo <url> --branch <branch> --purpose <purpose>

start 选项:
  --repo           Git 仓库 URL（SSH 或 HTTPS）
  --branch         要检出的分支
  --purpose        任务描述（用于命名结果分支，如 bug-fixing, feature-login）
  --ttyd-password  ttyd Basic Auth 密码（格式: password 或 user:password）
  --backup-dir     push 失败时的本地备份目录
  --no-ssh         不挂载宿主机 ~/.ssh
  --memory         容器内存限制（默认 4g）
  --cpus           容器 CPU 限制（默认 2.0）
```

---

## 9. 配置文件

**`~/.claude-worker/secrets.env`**（chmod 600，不进 git）：
```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxx

# HTTPS push 时需要（与 SSH 二选一）
# GIT_TOKEN=ghp_xxxx
```

---

## 10. 完整使用流程

```bash
# 首次安装
cd .claude-worker && chmod +x install.sh && ./install.sh
vim ~/.claude-worker/secrets.env    # 填入 ANTHROPIC_API_KEY

# 启动 worker（按题目示例格式）
claude-code-worker --repo git@github.com:myorg/myrepo.git \
                   --branch release/4.0.0 \
                   --purpose bug-fixing
# → 输出: 🚀 Worker 就绪! 打开 http://localhost:7700

# 并行启动第二个
claude-code-worker --repo git@github.com:myorg/myrepo.git \
                   --branch main \
                   --purpose feature-login
# → 输出: 🚀 Worker 就绪! 打开 http://localhost:7701

# 查看所有 worker
claude-code-worker list

# 完成后停止（会自动触发 push）
claude-code-worker stop ccw-bug-fixing-20240403120000
```

---

## 11. 端口规划

| 端口范围 | 用途 |
|---|---|
| 7700–7799 | worker ttyd 端口（自动分配，只绑定 127.0.0.1） |

最多支持 100 个并行 worker。所有端口仅绑定 localhost，不对外网暴露。

---

## 12. 安全模型

| 风险 | 缓解措施 |
|---|---|
| API Key 泄露 | `--env-file`（不出现在 CLI 历史），`secrets.env` chmod 600 |
| 容器逃逸 | 不挂载 Docker socket，不给特权模式 |
| 端口暴露 | 所有端口绑定 `127.0.0.1`，非 `0.0.0.0` |
| 代码意外丢失 | push 失败时自动 tar 备份到 /output |
| 容器资源失控 | `--memory 4g --cpus 2.0` 默认限制 |

---

## 13. 待确认事项

- [ ] `claude --bare` 参数是否在 2.1.91 版本可用（需实测）
- [ ] ttyd `--once` 行为确认：连接断开后是否触发进程退出（影响自动 push）
- [ ] 宿主机 Docker socket 权限（当前用户是否在 docker 组）
- [ ] 网络代理配置是否需要透传进容器（`HTTP_PROXY` 等）
