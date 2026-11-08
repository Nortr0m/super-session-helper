# 原宿主工具到 Trae 工具的映射

本技能不依赖 DeepSeek Harness，也不读写 `~/.dsh`。原型的宿主专用工具全部改写为 Trae 会话能力；下表给出语义对应，名称以当前会话实际暴露的工具列表为准。

## 能力对照

| 原工具 | 语义 | Trae 对应 |
|---|---|---|
| orchestrator_read | 分页读票据、线程、设置、PR，跟 next_offset | `list_sessions` + `read_session`（文件读取用当前会话的文件工具） |
| orchestrator_route | 批量准入，kind=accept/attach/ignore/stop，逐项独立成败 | `create_session`（accept）、`send_message_to_session`（attach/stop 通知） |
| orchestrator_eval | 装载候选窗口，首 5 条加前后各 5 条 | `read_session` 读取目标会话最近消息 |
| orchestrator_eval_read | 按消息 id、图片 id、游标、路径读更多 | `read_session` + 文件读取；不伪造 id |
| orchestrator_eval_decide | 落决定：方向、规则评估、范围、命名 | 判定后 `create_session`，并把判定写入回复或计划 |
| orchestrator_reply | purpose=answer/result/attention | `ReplyUser` |
| orchestrator_ack | 不超过 100 词，回原消息，按 trigger 去重 | `ReplyUser`（同在编排线程时）；执行会话内部用 `send_message_to_session` |
| orchestrator_report | state、summary、human_action、question、pr、reply_to | `send_message_to_session`（回报编排者），再由编排者 `ReplyUser` |
| orchestrator_ticket_retry | 仅 failed/blocked/投递失败可重试，同会话同目录同分支 | `send_message_to_session` 发重试指令 |
| orchestrator_ticket_request_status | 只向 idle 且末次 waiting 的执行会话要状态 | `send_message_to_session` |
| orchestrator_ticket_note | 不超过 100 词纯事实，不带授权 | `send_message_to_session`，标注 note |
| orchestrator_trigger_handback | 不超过 300 字理由，每条一次 | `send_message_to_session` 回编排者，或 `ReplyUser` |
| orchestrator_trigger_requeue | 重排评估；执行会话持有则拒绝 | 编排者自维护待评队列 |
| orchestrator_thread_unsubscribe / unrelated / follow | 订阅增删；只给上下文 | 编排者自维护订阅表 + `send_message_to_session` |
| orchestrator_settings_save / policy_sync / prompts_save | 仅凭显式人类配置请求，带 revision | 无对应；改配置走 Trae 设置，不在技能内自动执行 |

## 工具可用性自检

执行技能前先确认当前会话的工具列表：

1. 有 `create_session` 与 `send_message_to_session`：按标准流程做多会话编排。
2. 缺少 `create_session` 或 `send_message_to_session`：不冒充多会话；退回串行执行，每一步结束用 `ReplyUser` 汇报。
3. 缺少 `list_sessions` 或 `read_session`：不猜测执行结果；要求执行会话把终态写回可读产物，或在 `ReplyUser` 中说明无法观测。
4. 缺少 `Schedule`：不用定时唤醒；改为事件驱动，或结束回合等用户下次触发。

降级选择必须在首条汇报中写明，例如“当前会话未提供 `create_session`，已退回串行执行并用 `ReplyUser` 汇报”。

## 纪律

- 不调用不存在的工具，不伪造工具名
- 不假设工具会返回未声明的字段
- 工具调用失败按重试策略处理，不静默跳过
- 不使用任何 `orchestrator_*`、`dsh`、`tmux`、`pnpm` 专用指令；这些只出现在本映射表作为来源说明
