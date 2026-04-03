# 用 Claude Code 探究 Claude Code Web 环境

**记录日期**：2026-04-03  
**实验性质**：用工具本身研究工具——在 Claude Code Web 环境内，用 Claude Code 调查该环境的真实面貌，并据此设计一个能并行启动多个隔离 Claude Code 工作环境的工具。

---

## 起点：一个工具需求，一个合理假设

这次探索的起点是一个实际需求：构建 `claude-code-worker`，让用户可以通过一条命令，并行启动多个隔离的 Claude Code 工作环境，每个环境对应一个 git 分支，完成后自动推送结果。

最自然的假设是：Claude Code Web 运行在 Docker 容器里。毕竟 Docker 是"隔离环境"的标准方案，用于云端托管服务的概率极高。既然要构建类似的本地工具，先调查清楚 Anthropic 是怎么做的，再照葫芦画瓢，是合理的起手思路。

但调查结果出乎意料。

---

## 第一阶段：调查当前环境——打破"Docker 假设"

### 用了什么工具

在 Claude Code 内直接使用 Bash 工具执行系统探测命令：
- `cat /proc/1/cmdline` 查看 PID 1 进程
- `docker info` 和 `ls /var/run/docker.sock` 检查 Docker 状态
- `uname -a`、`cat /etc/os-release` 获取内核和系统信息
- `which claude`、`file $(which claude)` 检查 Claude Code 本身的二进制形态
- `nproc`、`free -h`、`df -h` 查看资源配额

### 发现了什么

**PID 1 不是 Docker**。`/proc/1/cmdline` 返回的是 `/process_api --firecracker-init`。

这是 Anthropic 专有的 Firecracker microVM 初始化进程。Docker daemon 根本没有运行——`/var/run/docker.sock` 不存在，`docker info` 报连接失败。

这个发现很关键。它告诉我们：Anthropic 选择了比 Docker 隔离级别更高的方案。Firecracker 是 AWS Lambda 和 Fargate 背后的 microVM 技术，每个 VM 有独立内核，安全边界远比容器 namespace 强。

完整环境数据记录如下：

| 项目 | 值 |
|---|---|
| 虚拟化 | Anthropic 专有 Firecracker microVM |
| 操作系统 | Ubuntu 24.04.4 LTS |
| 内核 | Linux 6.18.5 x86_64 |
| CPU | 4 核 |
| 内存 | 15 GB |
| 磁盘 | 252 GB（可用约 30 GB）|
| 网络 | 通过 Anthropic 托管代理，白名单域名 |

### 为什么重要

原计划是"在本地用 Docker 复刻云端环境"。现在知道云端用的是 Firecracker，而本地没有 Firecracker 工具——这意味着方案需要调整。但更重要的是，这个发现揭示了 Anthropic 的设计哲学：给每个用户会话一个独立内核隔离的 microVM，而不是一个共享内核的容器。

---

## 第二阶段：调查 Claude Code 本身

知道了"跑在什么里面"，下一个问题是"跑的是什么"。Claude Code 是一个 npm 包？一个 Node.js 脚本？还是别的什么？

### 发现了什么

```bash
$ file $(which claude)
/opt/claude-code/bin/claude: ELF 64-bit LSB executable, x86-64, statically linked
$ du -sh /opt/claude-code/bin/claude
230M
```

Claude Code 是一个 **230MB 的静态编译 ELF 二进制**。不是 npm 包，不是脚本，而是一个自包含的可执行文件。npm 安装方式已经弃用，现在的官方安装方式是 native installer：

```bash
DISABLE_AUTOUPDATER=1 curl -fsSL https://claude.ai/install.sh | bash
```

接着测试了各种参数：
- `claude --bare`：不存在
- `claude --serve`：不存在
- `claude --dangerously-skip-permissions`：存在，跳过权限确认
- `claude --tmux`：在 tmux 会话中运行
- `claude --worktree`：创建 git worktree

没有内置 web server 模式。要让 Claude Code TUI 能在浏览器里访问，需要借助 `ttyd` 这个工具——它能把终端进程包装成 WebSocket 服务。恰好 Ubuntu 24.04 官方源里有 ttyd 1.7.4。

认证方面，云端没有预设 `ANTHROPIC_API_KEY`，需要在运行时注入。支持三种方式：OAuth 订阅、API Key（环境变量注入）、以及 AWS/GCP 等云服务凭证。

### 为什么重要

静态二进制这个特点决定了部署方式。安装不依赖 Node.js 环境，但需要从 Anthropic 服务器下载。这影响 golden box 构建时间和网络要求。同时，`--dangerously-skip-permissions` 参数的存在证实了"自动化场景下跳过人工确认"是官方支持的用法，这是 worker 方案的重要基础。

---

## 第三阶段：工具链摸底——意外的丰富程度

除了 Claude Code 本身，这个 Firecracker VM 里预装了相当完整的开发工具链：

```
Python 3.11.15   /usr/local/bin/python3
Node.js 22.22.2  /opt/node22/bin/node
Go 1.24.7        /usr/local/go/bin/go
Rust 1.94.1      /usr/bin/rustc
Java OpenJDK 21  /usr/bin/java
Ruby 3.3.6       /usr/bin/ruby
PHP 8.4.19       /usr/bin/php
Git 2.43.0       /usr/bin/git
tmux 3.4         /usr/bin/tmux
```

这个发现确认了一件事：复刻这个环境并不需要特别定制，Ubuntu 24.04 + 标准包 + native Claude installer，基本可以还原。

---

## 第四阶段：架构演变——从 Docker 到 Vagrant

### 初始方案（被否定）

最初打算用 Docker 容器作为 worker 隔离方案。发现 Docker daemon 不可用后，需要重新选型。

### 用户本地环境的实际情况

用户的本地 Linux 机器已有：
- KVM（硬件虚拟化）
- Vagrant
- vagrant-libvirt plugin
- libvirt daemon

这些工具组合在一起，可以提供和 Firecracker 接近的隔离级别（独立内核的 KVM VM），启动时间在 5–15 秒之间——对于一个"开个工作环境做一两小时任务"的场景，这个延迟完全可以接受。

**方案 A（当前实现）** 就此确定：Vagrant + libvirt + Ubuntu Cloud Image，用 `ttyd` 包装 Claude Code TUI 暴露浏览器终端。

### 演进路线图（规划中）

同时规划了两条演进路径，在需要更快启动速度时迁移：

- **方案 B**：Weaveworks Ignite（Firecracker 封装），~1 秒启动，接近云端体验
- **方案 C**：Kata Containers（Docker CLI + VM 隔离），1–2 秒启动，适合团队场景

三个方案的 `entrypoint.sh`、`push-and-exit.sh` 和 CLI 接口完全复用，只替换底层启动机制。这个设计决策是在探索过程中逐渐清晰的——先把上层逻辑固定，留出底层可替换的接缝。

---

## 第五阶段：关键设计决策

架构确定后，一系列具体设计问题需要逐一讨论：

### 认证：三层优先级

最终设计了三层 API Key 获取策略，优先级从高到低：
1. 通过 URL 动态获取（`SECRET_URL` + `SECRET_TOKEN`），不落磁盘
2. 挂载目录文件（`/secrets/anthropic_api_key`）
3. 环境变量兜底（`ANTHROPIC_API_KEY`）

这个设计考虑了不同安全敏感程度的使用场景——个人开发者可以用环境变量，有安全要求的场景可以接 secret server。

### VM 方案 vs tmux+ttyd

讨论过一个轻量级替代方案：不起 VM，直接用 tmux + ttyd 在宿主机上开多个终端会话。但这方案没有隔离——多个 Claude Code 实例共享文件系统和环境变量，一个 worker 的 `rm -rf` 操作可能影响其他 worker。VM 方案的 5–15 秒启动时间换来的是真正的隔离，这个代价值得。

### Golden Box：把 provision 变成一次性工作

每次 `vagrant up` 从头安装 Claude Code 需要 3–5 分钟（主要是下载 230MB 二进制）。Golden box 策略把这变成一次性构建：

```
构建一次 golden box（5 分钟）
    ↓
之后每次 worker start 只需 5–15 秒
```

`claude-code-worker build-box` 命令负责这个一次性工作，更新 Claude Code 版本时用 `--force` 重建。

### 审核流程：VM 和 PR 生命周期解耦

最终确定的审核流程：
- Worker 完成任务后，`push-and-exit.sh` 自动 commit + push 到结果分支
- VM push 完成后**立即销毁**——VM 生命周期到此结束
- 需要 AI 审核时，用 `claude-code-worker review --branch result/xxx` 起一个新的 review worker
- 人工审核独立进行，不依赖 VM 是否存活

这个"VM 是一次性工具"的设计，比"VM 长期运行等待审核"简单得多，也避免了资源浪费。

### Box 更新：不做版本共存

更新 Claude Code 版本时，采用最简单的策略：停所有 worker → 重建 golden box → 继续。不维护多个版本的 box。理由是：这是个人开发工具，worker 生命周期短（几小时），更新 Claude Code 版本时统一升级没有问题。

### 默认 repo：不需要

每次 `start` 都必须指定 `--repo`，没有"默认仓库"的概念。这个决策避免了配置管理的复杂性，也防止了"忘记指定 repo 结果在错误仓库里工作"的人为错误。

---

## 总结：用工具研究工具的收获

这次探索有一个有趣的元层面：用 Claude Code（工具）在 Claude Code Web 环境（运行环境）内调查该环境，然后用调查结果设计一个本地版本的类似工具。

最重要的发现是那个打破初始假设的时刻——PID 1 不是 Docker，而是 Firecracker。这一个发现推翻了最初的方案，把架构选型从"用 Docker 复刻"转向了"用 KVM VM 提供同等隔离"。这种"先调查再设计"的过程，比凭直觉选方案更可靠——哪怕调查结果会推翻你的初始假设，尤其是当调查结果会推翻你的初始假设时。

最终的 SPEC.md 是这次探索的直接产物，记录了基于实测数据做出的每一个架构决策。
