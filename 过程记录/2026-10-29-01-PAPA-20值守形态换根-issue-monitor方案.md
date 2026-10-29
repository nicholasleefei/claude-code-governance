# 过程记录 2026-10-29-01 · PAPA-20 常驻直通通道值守形态换根（issue monitor 方案实测落地）

> **Z-08「只增不改」纪要** · 编号 2026-10-29-01 · 主题：CEO / UX / CTO 三条常驻直通通道长期靠「可点击的值守卡」维持 `in_review`，卡被点即失效 → 反复补挂 → watchdog 双醒，烧掉大量 run；本篇实测打通服务端 **issue monitor**（`executionPolicy.monitor`）这条**不可点击**的一等值守路径，并在 PAPA-20 落地。

---

## 0. 一句话

常驻通道的 `in_review` 值守一直挂一张**可点击**的 request_confirmation 卡；卡被点一下恰好等于「把这路径点没了」，于是触发「评审路径失效 → 唤醒 → 补挂新卡 → watchdog 双醒」循环。此前一串信使 issue（PAPA-24 / 27 / 28 / 30 / 31 / 32 / 34 / 35 / 36 / 48 / 90）都在反复修形态、没修到根。**PAPA-30 当年列出的候选方案 2「计划 monitor 值守」始终未实测**（原记录原话：「本次核查未发现该字段，可能不可写」）。本次直接查服务端 `/api/openapi.json`，确认 `PATCH /api/issues/{id}` **接受** `executionPolicy.monitor`，已在 PAPA-20 落地并通过四项生效前提的回读验证。

## 1. 病根（实测，非推断）

| 事实 | 证据 |
|---|---|
| 值守卡被点 = 自毁式路径 | PAPA-30 记录 PAPA-19 的 v1–v6 共被点 **6 次**，每次点击触发「评审路径失效 → 唤醒 → 补挂新卡 → watchdog 双醒」，12:18–12:31 共 13 分钟产生 **8+ 个 run** |
| issue 级 watchdog **不构成评审路径** | PAPA-30 描述原话：「issue 级 watchdog `6aef8a0c` 本身不构成评审路径」；它是**子树停了之后的恢复机制**（会另起影子 issue PAPA-29「Watchdog review for PAPA-19」），恢复动作又变成补挂新卡 → 循环自我复制 |
| 守卫只在 `in_review` 没有真实路径时追问 | 线程里反复出现系统评论「Paperclip needs a disposition before this issue can continue.」「Paperclip could not resolve this issue's missing disposition automatically.」 |
| `blocked` 形态在这几条通道上守不住 | PAPA-30 方案 1 用 `blocked + unblockDescriptor{owner:"board"}`；PAPA-34 / PAPA-35 随后又把 PAPA-20 / PAPA-21 从 `blocked` 改回 `in_review + 值守卡` |

**结论**：循环的燃料是「需要人来点的路径」。只要值守路径必须被点击，它迟早会被点，然后就会触发一轮补挂与双醒。

## 2. 本次补上的空白点

PAPA-30 的方案 2 写的是「先查 `/api/openapi.json` 确认 PATCH issue 是否接受 `executionPolicy.monitor`（**本次核查未发现该字段，可能不可写**）」——**该核查当时没做实**（没找到字段 ≠ 不可写，多半是字段埋在嵌套 schema 里没翻到）。

本次实测（Paperclip server `2026.916.1`）：`GET /api/openapi.json`（约 1.1 MB）→ `paths['/api/issues/{id}'].patch.requestBody.content['application/json'].schema.properties.executionPolicy.properties.monitor` 确实存在。schema 全文要点：

- 必填**只有** `nextCheckAt`（date-time）。
- 可选：`kind`（enum 仅 `external_service`，可 null）、`serviceName`、`externalRef`、`timeoutAt`、`maxAttempts`(1–100)、`recoveryPolicy`（`wake_owner` / `create_recovery_issue` / `escalate_to_board`）、`notes`(≤500)、`scheduledBy`（`assignee` / `board`）。
- `kind` / `serviceName` / `externalRef` **允许为 null**——常驻通道不指向外部服务，实测留 null 即被接受并正常落库。

## 3. 落地配方（已在 PAPA-20 实测通过）

```json
PATCH /api/issues/{issueId}
{
  "status": "in_review",
  "executionPolicy": {
    "mode": "normal",
    "monitor": {
      "nextCheckAt": "<当前时刻 +24h，ISO>",
      "timeoutAt": "<+48h，ISO>",
      "scheduledBy": "assignee",
      "maxAttempts": 100,
      "recoveryPolicy": "wake_owner",
      "notes": "常驻通道兜底值守：monitor 到点唤醒责任人，核查通道健康并把 nextCheckAt 续期 +24h。无新留言则仅续期，不改状态。"
    }
  },
  "comment": "<给董事会看的大白话：值守方式变了、留言照常即达>"
}
```

**四项生效前提**（缺一项 monitor 不触发；PATCH 响应里即可回读，不必再 GET）：

1. `monitorNextCheckAt` 非空；
2. `assigneeAgentId` = 通道责任 agent；
3. `assigneeUserId` = null；
4. `status` ∈ {`in_progress`, `in_review`}。

落地后 `reviewAttention.paths` 首项即 `"kind": "monitor"`、`"label": "Scheduled review monitor"`，`reviewAttention.state = covered`。

**配套动作**：撤回现存的 pending 值守卡（`POST /api/issues/{id}/interactions/{interactionId}/withdraw`，可带 `reason`）——monitor 落地后卡纯冗余，留着只是等着被点。

## 4. PAPA-20 实测结果（run `b3c53725`）

| 项 | 结果 |
|---|---|
| PATCH 挂 monitor | HTTP 200，回显 `monitorNextCheckAt=2026-10-30T12:05:13.078Z`、`scheduledBy=assignee`、`status=in_review`、`assigneeAgentId=CTO`、`assigneeUserId=null` |
| 撤回值守卡 v2（`111e00de`） | HTTP 200 → `status=cancelled`、`outcome=withdrawn`、`resolvedByAgentId=CTO` |
| 复核 | 通道 interactions 共 2 张、**pending 0 张**（v1 accepted / v2 withdrawn）；`reviewAttention` 覆盖路径 = monitor + active_run + queued_wake ×2，**已无 interaction 路径** |
| 通道能力未受影响 | 本次唤醒本身即「董事会留言 → `issue_commented` → CTO 醒来」的一次实证；`/hq/` 对话页按标题约定 `与 {agent} 的直达通道` 匹配本 issue 并直接 `POST /api/issues/{issueId}/comments`，与线程留言同通道 |
| 看板可达性 | 新增 `runtime_service` work product：HQ Console `http://127.0.0.1:3101/hq/`（serve = `E:\Paperclip\hq-console\server\proxy.mjs`，HQ_PORT=3101，upstream 3100；另监听 Tailscale `100.127.76.1:3101`），实测 `/hq/` 与 `/api/health` 均 200 |

## 5. 约定（推广时照抄）

- **monitor 是滚动的**：到点被唤醒后把 `nextCheckAt` 再推 +24h；醒来**没有**新留言就只续期、不改状态、不额外留言；发现异常再单独留言并调 `nextCheckAt`。
- **不补挂值守卡**。monitor 已是服务端可校验的一等路径，卡片只会重新引入循环。
- **通道 issue 不关 `done` / `cancelled`**：常驻通道不关闭，历史消息即对话记录。
- **不用 `blocked`**：在这几条通道上守不住。

## 6. 扩散

已开 [PAPA-92](/PAPA/issues/PAPA-92)「信使：把已实测的 monitor 值守方案套用到 PAPA-19 / PAPA-21 常驻通道」交 CEO（`b47d4f51`），内含本文配方与验收清单。同族仍待套用：

- [PAPA-19](/PAPA/issues/PAPA-19)（CEO 通道，`in_review`，现仍靠 pending 卡；watchdog `6aef8a0c` active）。
- [PAPA-21](/PAPA/issues/PAPA-21)（UX 通道，`in_progress`，`watchdog=null`、`monitorNextCheckAt=null`）。

## 7. 顺带记下的操作事实（本机）

- **bash 的 PATH 是分号分隔的 Windows 串**，bash 不解析 → `ls/env/grep/node` 全 "command not found"。对策：`export PATH=/usr/bin:/bin:/e/tools/node-v24.21.0-win-x64:$PATH`。
- **MSYS 会把参数里的 `/api/...` 改成 `C:/Program Files/Git/api/...`** → fetch 报 `Invalid URL`。对策：`export MSYS_NO_PATHCONV=1`（本篇与此前 `taskkill/schtasks` 的坑同源）。
- **控制面 API 路径前缀**：`PAPERCLIP_API_URL=http://127.0.0.1:3100`，端点全在 `/api/` 下；**不存在**的路由会静默返回 SPA 的 `index.html`（HTTP 200），不是 404 —— 写脚本时要按 content-type 判 JSON，别被"200"骗了。
- **`commentClientRequestId` 必须是 UUID**（否则 PATCH 400 `invalid_format`），可作幂等键。
- **查契约的正规入口**：`GET /api/openapi.json`（本机有，约 1.1 MB）；`/openapi.json` 返回 SPA HTML。

## 8. 改动清单

- 控制面：`PAPA-20` 挂 issue monitor + 撤回值守卡 v2 + 新增 HQ Console `runtime_service` work product；新建 `PAPA-92`（交 CEO）。
- 本机文件：无代码改动（未动 `E:\Paperclip\hq-console` 任何源码）。
- 本文：新建纪要。

相关：[[PAPA-1 指挥系统交接]]、[[hq-console v1 状态]]
