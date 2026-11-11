# 2026-11-11-05 · 工作台助手接入本机 Claude Code CLI（Agent SDK）

> 只增不改。用户纠正：工作台助手不是关键词匹配，也不是方舟 LLM，
> 必须走 **Claude Agent SDK → 本机 Claude Code CLI**（MVP 已验证）。

## 一、架构

```
对话口 POST /messages
  ├─ 命中技能 → 建任务（确定性，不变）
  └─ 未命中 → agent_jobs 入队 → 本机 Agent 节点取活
              → ClaudeSDKClient（本机 claude.exe）→ 写回回答
              ↳ 节点离线：退方舟 LLM → 再退关键词文案
```

- 平台只排活、只收结果；本机**只出不进**（同算力节点约定）
- 鉴权：`X-Compute-Key`（COMPUTE_NODE_KEY）
- 节点：`agent-node/worker.py`，参数对齐 MVP：
  `system_prompt=claude_code preset + append`、`allowed_tools=[]`、
  `can_use_tool` 只放行 Read/Glob/Grep/LS/NotebookRead/TodoWrite、
  `max_turns=8`、`max_budget_usd=0.5`、`include_partial_messages=False`

## 二、改动

- 新增 `infra/initdb/06_agent_jobs.sql`（已执行）
- 新增 `backend/app/api/v1/agent_jobs.py`（next/result/get + enqueue/wait）
- `messages.py`：未命中技能时优先 Agent，等待最多 90s
- 新增 `agent-node/worker.py` + README
- 前端标注回复来源（Claude Code / 方舟 / 关键词）

## 三、验证

- 本机 SDK 与 bundled `claude.exe` 已确认安装
- 实测用户提问返回 `reply_source=agent` 的完整中文回答（自我介绍 + 能力清单）

## 四、清单

153 项：133 ✅ / 8 🟡 / 12 ⛔（新增 P16）。
