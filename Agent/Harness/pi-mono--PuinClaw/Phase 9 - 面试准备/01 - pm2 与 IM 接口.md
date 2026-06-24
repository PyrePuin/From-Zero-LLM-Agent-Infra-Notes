---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - 面试
  - pm2
  - im
  - deployment
aliases:
  - pm2
  - IM 接口
---

# 01 - pm2 与 IM 接口

> [!note]
> 让 PuinClaw 从"本地 CLI"升级到"长期在线 + 远程可用"的两件套。
> **pm2** 解决"怎么长期跑",**IM 接口**解决"怎么远程交互"。

## 1. pm2 是什么

```
pm2 = Process Manager 2(Node.js 进程管理器)

核心能力:
   ┌────────────────────────────────────────────┐
   │ ① 守护进程 — 关掉终端,agent 继续跑        │
   │ ② 自动重启 — 崩了自动拉起                  │
   │ ③ 日志管理 — stdout/stderr 自动归档        │
   │ ④ 启动队列 — 开机自启                       │
   │ ⑤ 多实例 — 多个 agent 并行                 │
   │ ⑥ 监控面板 — pm2 monit 看资源占用          │
   └────────────────────────────────────────────┘
```

### 跟 tmux 的区别

| | tmux | pm2 |
|---|---|---|
| 本质 | 终端复用器 | 进程管理器 |
| 崩了怎么办 | 进程死掉 | **自动重启** |
| 开机自启 | ❌ | ✅ `pm2 startup` |
| 日志 | ❌ | ✅ `pm2 logs` |
| 监控 | ❌ | ✅ `pm2 monit` |
| 集群 | ❌ | ✅ `pm2 scale` |

→ tmux 是"会话保持",pm2 是"生产级守护"。

## 2. 怎么用 pm2 跑 PuinClaw

### 最小配置(ecosystem.config.js)

```javascript
module.exports = {
  apps: [{
    name: "puinclaw",
    script: "./packages/mom/dist/main.js",
    args: "goal --id default",
    cwd: "/path/to/pi-mono",
    env: {
      ANTHROPIC_API_KEY: "sk-ant-...",
      NODE_ENV: "production"
    },
    max_memory_restart: "1G",    // 内存超 1G 重启
    autorestart: true,
    watch: false,                 // 文件变化不重启
    exp_backoff_restart_delay: 100
  }]
};
```

### 常用命令

```bash
pm2 start ecosystem.config.js    # 启动
pm2 list                          # 看所有进程
pm2 logs puinclaw                 # 看日志
pm2 restart puinclaw              # 重启
pm2 stop puinclaw                 # 停止
pm2 delete puinclaw               # 删除
pm2 monit                          # 监控面板
pm2 startup                        # 开机自启
pm2 save                           # 保存当前进程列表
```

## 3. IM 接口是什么

```
IM = Instant Messaging(即时通讯)

常见 IM 平台 + API 形式:
   ┌──────────────────┬──────────────────────────────────┐
   │ 平台             │ API 特点                         │
   ├──────────────────┼──────────────────────────────────┤
   │ Telegram         │ Bot API 最简单,长轮询/webhook   │
   │ Discord          │ Slash Command + Gateway          │
   │ Slack            │ Bolt SDK,企业场景                │
   │ 微信             │ 限制多,通常用 Server酱/企业微信  │
   │ 飞书             │ OpenAPI 完整,国内企业           │
   │ Lark/Teams       │ 国际版本                         │
   └──────────────────┴──────────────────────────────────┘

推荐:Telegram(最简单),Slack(企业),飞书(国内)
```

### IM 接口的本质

```
你的 agent                  IM 平台                  用户
───────────                ─────────                ────
                            ┌────────────┐
agent ←── Webhook ──────── │ IM Server  │ ←── 用户发消息
       ─── API 调用 ──────→ │            │ ──→ 推给用户
                            └────────────┘
```

- **入站**:IM 把用户消息推给你的 agent(webhook 或长轮询)
- **出站**:agent 调 IM API 把回复推给用户

## 4. PuinClaw 接入 Telegram 示例

### 最小代码

```javascript
import TelegramBot from "node-telegram-bot-api";
import { runGoal } from "@mariozechner/pi-mom";

const bot = new TelegramBot(process.env.TELEGRAM_BOT_TOKEN, { polling: true });

bot.onText(/\/goal (.+)/, async (msg, match) => {
  const chatId = msg.chat.id;
  const goalText = match[1];
  
  bot.sendMessage(chatId, `🎯 收到目标: ${goalText},开始执行...`);
  
  // 调 PuinClaw 的 runGoal
  await runGoal("./goals/user-goal.md", {
    onTurn: (state) => {
      // 每 turn 给用户汇报进度
      bot.sendMessage(
        chatId,
        `Turn ${state.turn} | $${state.cost_usd.toFixed(4)} | ${state.last_action}`
      );
    }
  });
  
  bot.sendMessage(chatId, "✅ 目标完成!");
});

bot.onText(/\/pause/, async (msg) => {
  // 调 PuinClaw 的 abort 接口
});
```

### 接入流程

```
1. @BotFather 创建 bot,拿到 token
2. agent 进程跑在 pm2 里
3. agent 用长轮询(polling)或 webhook 收消息
4. 收到 /goal 命令 → 启动 PuinClaw runGoal
5. 每个 turn 通过 Telegram API 推送进度
6. /pause /resume /abort 通过 pm2 signal 或 agent 接口控制
```

## 5. 完整架构图

```
┌──────────────────────────────────────────────────────────┐
│ Telegram 用户                                            │
│   │                                                      │
│   │ /goal "测试覆盖率达到 90%"                          │
│   ▼                                                      │
│ Telegram Bot API  ←──── polling ────┐                   │
│   │                                  │                   │
│   │ webhook                          │                   │
│   ▼                                  │                   │
│ ┌─────────────────────────────────┐  │                   │
│ │ pm2 守护进程                    │  │                   │
│ │  ┌───────────────────────────┐  │  │                   │
│ │  │ PuinClaw (Node.js)        │  │  │                   │
│ │  │  ├─ Telegram adapter      │──┘  │                   │
│ │  │  ├─ Goal runner           │     │                   │
│ │  │  ├─ pi-agent-core         │     │                   │
│ │  │  ├─ pi-ai (Anthropic)     │     │                   │
│ │  │  └─ tools(bash/read/...) │     │                   │
│ │  └───────────────────────────┘  │                     │
│ └─────────────────────────────────┘                     │
│   │                                                      │
│   │ 进度推送                                              │
│   ▼                                                      │
│ Telegram → 用户看到 "Turn 5 | $0.12 | wrote tests"     │
└──────────────────────────────────────────────────────────┘
```

## 6. 面试要点

### 必答清单

```
Q1: pm2 是什么?跟 tmux/nohup 区别?
A: 进程管理器。比 tmux 多了自动重启/日志/监控/开机自启。

Q2: 怎么让 agent 在崩了之后自动恢复?
A: pm2 autorestart + 状态持久化(PuinClaw 的 state.json)。
   崩了 → pm2 拉起 → 加载 state.json → 从上次 turn 继续。

Q3: IM 接口本质是什么?
A: 双向通信。入站 webhook/polling 收消息,出站 API 调用发消息。

Q4: Telegram vs Slack 怎么选?
A: Telegram 简单(polling 即可),Slack 企业级(需要 OAuth + webhook)。

Q5: agent 长跑怎么避免单点故障?
A: pm2 守护 + 进程崩溃自动重启 + 状态外置(state.json) +
   可选多实例 + 反向代理 + 监控告警。
```

### 加分点

```
• 知道 pm2 cluster mode(Node.js 多进程)
• 知道 pm2 + systemd 的关系
• 提到 IM 接入的安全考虑(token 管理、用户鉴权)
• 提到消息队列削峰(高并发 IM 消息)
• 提到 webhook vs polling 的取舍
```

## 7. 跟之前学的对照

| 学过的 | 这里的对应 |
|---|---|
| [[04 - Channels]](claw0) | IM 接口就是 Channels 的实现 |
| [[05 - Gateway & Routing]] | Telegram/Slack 是 Gateway |
| [[09 - Resilience]] | pm2 autorestart = graceful shutdown |
| pi-mono packages/mom | 已经做了 goal runner,加 IM 即可 |

## 8. PuinClaw 实操清单

```
□ npm install -g pm2
□ 写 ecosystem.config.js
□ pm2 start + pm2 save + pm2 startup
□ @BotFather 申请 Telegram bot token
□ 实现 Telegram adapter(收 /goal 命令,推送进度)
□ 测试崩溃恢复:pm2 restart 后能不能 resume
```

_Generated for PuinClaw 面试准备, 2026-06-25_
