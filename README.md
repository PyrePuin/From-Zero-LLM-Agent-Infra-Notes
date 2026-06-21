# From Zero --- LLM Agent Infra Notes

从零拆解 LLM Agent 的工程结构。

> [!note]
> 本仓库名暗示了"LLM 基础设施全景"，但目前**只填充了 Harness 这一层**。其他方向（训练 / 推理 / RAG / 评测 …）暂未启动。

## 当前内容

```
Agent/
  Harness/                      ← Harness（外壳）层：包在模型外面的工程代码
    Learn-Claude-Code/          ← Phase 1-6：CLI Agent 骨架（shareAI-lab/learn-claude-code）
      README.md                 ← 进入这一层看细节
      Phase 1 - 基础机制/
      Phase 2 - 上下文治理/
      Phase 3 - 记忆与恢复/
      Phase 4 - 长时间任务/
      Phase 5 - 多智能体/
      Phase 6 - 扩展与组装/
      对话精华 QA/
    Claw-Theory/                ← Phase 7+：产品级常驻 Agent（shareAI-lab/claw0）
      README.md
      Phase 7 - 常驻 Agent/
      对话精华 QA/
```

## 阅读顺序

1. 先读 [Agent/Harness/Learn-Claude-Code/README.md](Agent/Harness/Learn-Claude-Code/README.md)：了解 Phase 1-6 骨架。
2. 每个 Phase 先读 `00 - 综合总结.md`：建立"这一组课为什么放一起"的整体感。
3. 再按编号顺序读单课笔记。
4. 卡住时翻 `对话精华 QA`。
5. Phase 1-6 完成后进 [Agent/Harness/Claw-Theory/README.md](Agent/Harness/Claw-Theory/README.md)：从 CLI 推进到常驻服务。

## 分支说明

- `v2`（推荐）：2026-06 重构版，统一笔记结构、规范化 mermaid。
- `main`：旧版，结构较松散，保留作对照。
