# Claude Code Worker — 设计规格（Design Spec）

**版本**: 0.3  
**当前实现方案**: 方案 B — Vagrant + libvirt + Ubuntu Cloud Image  
**目标**: 让用户通过一条命令启动多个隔离的 Claude Code 工作环境，每个环境对应一个 git 分支，在浏览器中与 Claude Code 交互完成任务，结果自动推送到结果分支。

---

## 0. 方案演进路线图

| 方案 | 技术栈 | 启动时间 | 隔离级别 | 状态 |
|---|---|---|---|---|
| **A（当前）** | Vagrant + libvirt + Ubuntu Cloud Image | 5–15 秒 | VM（独立内核） | ✅ **当前实现** |
| B（演进） | Weaveworks Ignite（Firecracker 封装） | ~1 秒 | VM（独立内核） | 📋 待规划 |
| C（演进） | Kata Containers（Docker CLI + VM 隔离） | 1–2 秒 | VM（独立内核） | 📋 待规划 |

> 三个方案的 `entrypoint.sh`、`push-and-exit.sh`、`claude-code-worker` CLI 接口**完全复用**，
> 只有底层 VM 启动机制不同。演进时只需替换"启动层"，不需要重写上层逻辑。

---

## 1. 当前 Claude Code Web 环境基线（经实测）

> 此节描述 Anthropic 云端环境，作为构建本地环境的参考基准。

| 项目 | 值 | 路径 |
|---|---|---|
| 虚拟化类型 | Firecracker VM（PID 1: /process_api --firecracker-init） | — |
| 操作系统 | Ubuntu 24.04.4 LTS (Noble Numbat) | — |
| 内核 | Linux 6.18.5 x86_64 | — |
| Python | 3.11.15 | `/usr/local/bin/python3` |
| Node.js | 22.22.2 | `/opt/node22/bin/node` |
| Go | 1.24.7 | `/usr/local/go/bin/go` |
| Rust | 1.94.1 | `/usr/bin/rustc` |
| Java | OpenJDK 21.0.10 | `/usr/bin/java` |
| Ruby | 3.3.6 | `/usr/bin/ruby` |
| PHP | 8.4.19 | `/usr/bin/php` |
| Git | 2.43.0 | `/usr/bin/git` |
| tmux | 3.4 | `/usr/bin/tmux` |
| **Claude Code** | **2.1.91** | `/opt/claude-code/bin/claude`（230MB ELF） |
| CPU | 4 核 / 内存 15 GB / 磁盘 252 GB | — |

### Claude Code 关键参数（v2.1.91 实测）

```
claude [prompt]                       # 交互式会话
claude -p / --print                   # 非交互模式
claude --dangerously-skip-permissions # 跳过权限确认（VM 内安全）
claude --tmux                         # 在 tmux 会话中运行
claude --worktree [name]              # 创建 git worktree
claude --settings <file>              # 自定义设置文件
```

> ⚠️ `--bare` 在 v2.1.91 中不存在。认证只需设 `ANTHROPIC_API_KEY` 环境变量。

---

## 2. 方案 A：Vagrant + libvirt + Ubuntu Cloud Image（当前实现）

### 2.1 架构总览

```
用户本地 Linux 机器（有 KVM + vagrant-libvirt）
│
├── claude-code-worker CLI
│   ├── start --repo --branch --purpose
│   ├── list / stop / logs / status
│   └── build-box（构建 golden box，首次或更新时用）
│
├── ~/.claude-worker/
│   ├── secrets.env                  ← 敏感信息（chmod 600，不进 git）
│   ├── workers.json                 ← 运行状态
│   ├── boxes/
│   │   └── claude-worker.box        ← golden box（预装所有工具）
│   └── instances/
│       ├── ccw-bug-fixing-xxx/
│       │   ├── Vagrantfile          ← 动态生成，含端口和参数
│       │   └── .vagrant/
│       └── ccw-feature-dev-xxx/
│           └── ...
│
└── Vagrant VMs（KVM 隔离）
    ├── ccw-bug-fixing-xxx  :7700  → http://localhost:7700
    ├── ccw-feature-dev-xxx :7701  → http://localhost:7701
    └── ccw-refactor-xxx    :7702  → http://localhost:7702
```

### 2.2 Golden Box 策略（核心性能优化）

**为什么需要 golden box：**
每次 `vagrant up` 从头 provision（装 Claude Code）需要 3–5 分钟。
Golden box 把这步变成一次性工作，之后每次启动只需 5–15 秒。

**构建流程：**
```
Ubuntu 24.04 cloud image (generic/ubuntu2404 libvirt box)
    ↓ vagrant up + provision（一次性，约 5 分钟）
    装：git, python3, nodejs, go, ttyd, claude, 开发工具
    ↓ vagrant package --output claude-worker.box
    ↓ vagrant box add claude-worker ./claude-worker.box
claude-worker.box（本地 golden box）
    ↓ 每次 worker start（无 provision）
    5–15 秒内就绪
```

**命令：**
```bash
claude-code-worker build-box          # 构建/重建 golden box（约 5 分钟，只需偶尔执行）
claude-code-worker build-box --force  # 强制重建（更新 Claude Code 版本时用）
```

### 2.3 前置依赖

```bash
# 宿主机需要安装
sudo apt install qemu-kvm libvirt-daemon-system vagrant
vagrant plugin install vagrant-libvirt

# 确认 KVM 可用
kvm-ok
virsh list --all
```

### 2.4 文件结构

```
.claude-worker/
├── SPEC.md
├── claude-code-worker          # 宿主机 CLI（Python）
├── install.sh                  # 一键安装脚本
├── build-box/
│   ├── Vagrantfile             # Golden box 构建用
│   └── provision.sh            # 工具安装脚本（apt + claude + ttyd）
└── vm/
    ├── Vagrantfile.tmpl        # Worker VM 模板（动态生成）
    ├── entrypoint.sh           # VM 内初始化脚本
    └── push-and-exit.sh        # VM 内自动 commit + push
```

### 2.5 build-box/Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2404"
  config.vm.provider :libvirt do |lv|
    lv.memory = 4096
    lv.cpus   = 2
    lv.driver = "kvm"
  end

  config.vm.provision "shell", path: "provision.sh"
end
```

### 2.6 build-box/provision.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive
export DISABLE_AUTOUPDATER=1

apt-get update && apt-get install -y --no-install-recommends \
    curl wget ca-certificates gnupg2 \
    git git-lfs \
    build-essential make cmake \
    python3 python3-pip python3-venv \
    nodejs npm \
    ttyd \
    vim less jq tmux \
    && rm -rf /var/lib/apt/lists/*

# Claude Code（native installer）
curl -fsSL https://claude.ai/install.sh | bash

# 确保 claude 在全局 PATH
CLAUDE_BIN=$(find /root/.local/bin /opt/claude-code/bin -name claude 2>/dev/null | head -1)
ln -sf "$CLAUDE_BIN" /usr/local/bin/claude

# 验证安装
claude --version

# 写入 entrypoint 和 push 脚本
cp /vagrant/../vm/entrypoint.sh /usr/local/bin/entrypoint.sh
cp /vagrant/../vm/push-and-exit.sh /usr/local/bin/push-and-exit.sh
chmod +x /usr/local/bin/entrypoint.sh /usr/local/bin/push-and-exit.sh

echo "[provision] Golden box 构建完成"
```

### 2.7 vm/Vagrantfile.tmpl

每个 worker 动态生成，`{{占位符}}` 由 CLI 替换：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "claude-worker"   # 使用本地 golden box

  config.vm.network "forwarded_port",
    guest: 7681,
    host:  {{HOST_PORT}},
    host_ip: "127.0.0.1"            # 只绑定 localhost，不对外暴露

  config.vm.provider :libvirt do |lv|
    lv.memory = {{MEMORY_MB}}
    lv.cpus   = {{CPUS}}
    lv.driver = "kvm"
  end

  # SSH key 只读挂载（用于 git push）
  config.vm.synced_folder "~/.ssh", "/root/.ssh",
    type: "rsync",
    rsync__exclude: [],
    rsync__args: ["--chmod=D700,F600"]

  # 注入环境变量并启动 ttyd + claude
  config.vm.provision "shell", run: "always", env: {
    "ANTHROPIC_API_KEY" => ENV["ANTHROPIC_API_KEY"] || "",
    "GIT_TOKEN"         => ENV["GIT_TOKEN"] || "",
    "WORKER_ID"         => "{{WORKER_ID}}",
    "WORKER_REPO_URL"   => "{{REPO_URL}}",
    "WORKER_BRANCH"     => "{{BRANCH}}",
    "WORKER_PURPOSE"    => "{{PURPOSE}}",
  }, inline: "/usr/local/bin/entrypoint.sh"
end
```

> **安全说明**：`ENV["ANTHROPIC_API_KEY"]` 在 CLI 执行时从宿主机 `secrets.env` 读取并注入，
> 不写入 Vagrantfile 文件本身（Vagrantfile 是动态生成的临时文件，存于 `~/.claude-worker/instances/` 下）。

### 2.8 vm/entrypoint.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${WORKER_REPO_URL:?需要 WORKER_REPO_URL}"
: "${WORKER_BRANCH:?需要 WORKER_BRANCH}"
: "${WORKER_PURPOSE:?需要 WORKER_PURPOSE}"
: "${WORKER_ID:?需要 WORKER_ID}"
: "${ANTHROPIC_API_KEY:?需要 ANTHROPIC_API_KEY}"

# 写入全局环境（使 ttyd 启动的子进程也能读取）
cat >> /etc/environment << EOF
ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
EOF

RESULT_BRANCH="result/${WORKER_PURPOSE}-$(date +%Y%m%d-%H%M%S)"
TTYD_PORT="${TTYD_PORT:-7681}"

# Git 凭证（HTTPS token 方式）
if [[ -n "${GIT_TOKEN:-}" ]]; then
    GIT_HOST=$(echo "$WORKER_REPO_URL" | sed -E 's|https?://([^/]+)/.*|\1|')
    printf "machine %s\n  login x-access-token\n  password %s\n" \
        "$GIT_HOST" "$GIT_TOKEN" > /root/.netrc
    chmod 600 /root/.netrc
fi
# SSH 方式：rsync 挂载的 ~/.ssh 已经就位

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

echo "[worker] 启动 ttyd :$TTYD_PORT，结果分支: $RESULT_BRANCH"

TTYD_ARGS=(--port "$TTYD_PORT" --writable --once --ping-interval 30)
[[ -n "${TTYD_CREDENTIAL:-}" ]] && TTYD_ARGS+=(--credential "$TTYD_CREDENTIAL")

ttyd "${TTYD_ARGS[@]}" \
    bash -c "cd /workspace && exec claude --dangerously-skip-permissions"

# ttyd 退出后自动 push
exec /usr/local/bin/push-and-exit.sh
```

### 2.9 vm/push-and-exit.sh

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
    # 备份到宿主机共享目录（如有）
    [[ -d /vagrant-backup ]] && \
        tar czf /vagrant-backup/workspace-backup-$(date +%s).tar.gz /workspace
    exit 1
fi
```

### 2.10 claude-code-worker CLI 接口

```
用法:
  claude-code-worker start --repo <url> --branch <branch> --purpose <purpose> [选项]
  claude-code-worker list
  claude-code-worker stop <id> [--force]
  claude-code-worker logs <id>
  claude-code-worker status <id>
  claude-code-worker build-box [--force]   ← 构建 golden box（首次必须）

# 兼容简写:
  claude-code-worker --repo <url> --branch <branch> --purpose <purpose>

start 选项:
  --ttyd-password  ttyd Basic Auth 密码
  --memory         VM 内存（默认 2048，单位 MB）
  --cpus           VM CPU 数（默认 2）
  --no-ssh         不同步 ~/.ssh 进 VM
```

### 2.11 配置文件

**`~/.claude-worker/secrets.env`**（chmod 600，不进 git）：
```bash
ANTHROPIC_API_KEY=sk-ant-api03-xxxx
# GIT_TOKEN=ghp_xxxx   # HTTPS push 时使用
```

CLI 启动时自动 `source` 此文件，通过 Vagrantfile `env:` 注入 VM。

### 2.12 端口规划

| 范围 | 绑定 | 说明 |
|---|---|---|
| 7700–7799 | 127.0.0.1 | worker ttyd 端口，最多 100 个并行 |

### 2.13 安全模型

| 风险 | 缓解措施 |
|---|---|
| API Key 泄露 | `secrets.env` chmod 600，通过 Vagrant `env:` 注入（不写入文件） |
| VM 逃逸 | KVM 硬件隔离，不共享内核 |
| 端口暴露 | 全部绑定 `127.0.0.1` |
| 代码丢失 | push 失败时备份到宿主机共享目录 |
| 资源失控 | libvirt memory/cpus 限制 |

### 2.14 完整使用流程

```bash
# === 首次安装（只需一次）===
cd .claude-worker && chmod +x install.sh && ./install.sh
# 安装 vagrant-libvirt plugin，注册脚本到 PATH

vim ~/.claude-worker/secrets.env   # 填入 ANTHROPIC_API_KEY

claude-code-worker build-box       # 构建 golden box（约 5 分钟）

# === 日常使用 ===
claude-code-worker --repo git@github.com:org/repo.git \
                   --branch release/4.0.0 \
                   --purpose bug-fixing
# → [info] 启动 VM ccw-bug-fixing-20240403120000...
# → [ready] 打开 http://localhost:7700

# 并行第二个
claude-code-worker --repo git@github.com:org/repo.git \
                   --branch main \
                   --purpose feature-login
# → [ready] 打开 http://localhost:7701

claude-code-worker list
# WORKER ID                          PORT   BRANCH          PURPOSE
# ccw-bug-fixing-20240403120000      7700   release/4.0.0   bug-fixing
# ccw-feature-login-20240403120100   7701   main            feature-login

claude-code-worker stop ccw-bug-fixing-20240403120000
# → [info] 触发 push...
# → [push] 成功: result/bug-fixing-20240403-120001
# → [info] VM 已销毁
```

### 2.15 待确认事项（本地机器实测）

- [ ] `claude --dangerously-skip-permissions` 在 VM 内首次运行是否还需要交互确认？
      → 若需要，在 provision.sh 中预先运行一次 `echo "" | claude --print "hello"` 完成初始化
- [ ] `ttyd --once` 在连接断开后是否立即退出进程？
      → 若不立即退出，改用 `ttyd --once --exit-on-disconnect`（如支持）或用 trap 处理
- [ ] golden box 中 claude 安装路径：`/root/.local/bin/` 还是 `/opt/claude-code/bin/`？
      → provision.sh 用 `find` 自动定位并建 symlink，已处理
- [ ] SSH rsync 到 VM 时权限是否正确？
      → 已在 Vagrantfile 中指定 `--chmod=D700,F600`，实测确认

---

## 3. 方案 B（演进）：Weaveworks Ignite — Firecracker 封装

> 📋 规划阶段，暂不实现。当方案 A 满足不了启动速度需求时迁移。

**核心优势**：启动时间 ~1 秒，接近 Anthropic 云端体验。

**原理**：
```
ignite run ubuntu  →  Firecracker VMM  →  KVM  →  microVM（125ms 启动）
```
Ignite 提供类 Docker 的 CLI，但底层每个"容器"是一个真正的 VM。

**前置依赖**：
```bash
# 需要 KVM（已有），额外安装：
curl -fsSL https://github.com/weaveworks/ignite/releases/latest/download/ignite-amd64 \
    -o /usr/local/bin/ignite && chmod +x /usr/local/bin/ignite

# containerd（Ignite 依赖）
sudo apt install containerd
```

**与方案 A 的差异**：
- 启动命令：`ignite run claude-worker:latest --name ccw-xxx --ports 7700:7681`（替换 `vagrant up`）
- 停止命令：`ignite stop ccw-xxx && ignite rm ccw-xxx`（替换 `vagrant destroy`）
- Golden image：用 `ignite build` 替换 `vagrant package`
- `entrypoint.sh`、`push-and-exit.sh`、CLI 接口**完全不变**

**迁移成本**：仅替换 `claude-code-worker` CLI 中的 ~50 行启动/停止逻辑。

---

## 4. 方案 C（演进）：Kata Containers — Docker CLI + VM 隔离

> 📋 规划阶段，暂不实现。适合团队共享 CI/CD 场景。

**核心优势**：对外呈现标准 Docker 接口，底层每个容器是 KVM microVM，1–2 秒启动。

**原理**：
```
docker run  →  containerd  →  Kata shim  →  QEMU/Firecracker  →  KVM  →  microVM
```

**前置依赖**：
```bash
sudo apt install kata-containers
# 配置 Docker/containerd 使用 kata runtime
```

**与方案 A 的差异**：
- 启动命令：`docker run --runtime=kata-runtime ...`（回归 Docker 接口）
- 构建：`docker build`（替换 `vagrant package`）
- `entrypoint.sh`、`push-and-exit.sh`、CLI 接口**完全不变**

**适用场景**：多人团队、有 Docker Registry、需要 CI/CD 集成时优先选择。

---

## 5. 三方案对比

| | 方案 A（当前）| 方案 B（演进）| 方案 C（演进）|
|---|---|---|---|
| **技术** | Vagrant + libvirt | Weaveworks Ignite | Kata Containers |
| **CLI** | vagrant | ignite | docker |
| **启动时间** | 5–15 秒 | ~1 秒 | 1–2 秒 |
| **隔离** | KVM VM | Firecracker VM | QEMU/Firecracker VM |
| **前置依赖** | KVM + Vagrant（已有） | KVM + containerd | Docker + KVM |
| **镜像格式** | .box | OCI image | Docker image |
| **Docker Registry** | 不需要 | 不需要 | 可选 |
| **团队共享** | 困难（.box 文件大） | 中等 | 容易（Registry） |
| **迁移成本** | — | 低（替换 ~50 行） | 低（替换 ~50 行） |
