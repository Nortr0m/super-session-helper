# 票据状态机

票据是编排的最小记账单位。原型的核心不是“任务列表”，而是四类对象分开记账：决定、票据、派工、报告。混用会导致状态误判。

## 四类对象

| 对象 | 状态 | 说明 |
|---|---|---|
| Decision | accept / stop / ignore / unavailable | 准入结论，一请求一决定 |
| Ticket | open → ended | 票据寿命；结束原因 completed / merged / closed / dropped / stopped |
| Assignment | pending → running → ended | 一次派工；与票据寿命独立，可多次 |
| Report | progress / waiting / ready / blocked / failed / dropped | 执行会话的报告状态 |

Stage 是派生状态，不单独存储。按顺序取第一个匹配：

1. person/delivery：投递失败，2 秒起倍增到 1 分钟
2. person/answer：存在待用户回答的 attention
3. person/review：最新报告为 ready，等待人复核或合并
4. person/blocked 或 person/failed：此后没有新的人工消息
5. working/queued：无派工或派工 pending
6. idle：waiting，或派工已结束
7. working：派工 running

## 状态迁移

票据 open → ended 只有两条路：

- 执行侧：报告 ready 且没有待续 PR；dropped 表示无事可做；所开 PR 全部关闭或合并且无续作
- 人侧：用户显式 stop，或经准入的 stop 消息

ended → open 仅当用户消息落在该票据自己的线程且重新校验通过；不新建决定；被用户 stop 过的票据永不复活。

Assignment 结束有三种原因：retry、restart、票据结束。只有最终报告能在执行侧结束票据；progress 不能。

## 票据字段

```text
ticket {
  id, title, decisionId, mandate, channels,
  assignments, inputs, undelivered, attention,
  follows, origin, execution, answers, delivery
}

mandate {
  strength,        // explicit / inferred
  scope,           // 允许范围
  rules,           // 本次启用的规则
  policyVersion,
  actor,           // 谁授权
  receivingScope   // 适用范围
}
```

mandate 创建后不可变；扩大范围 = 新票据。input 类型包括 trigger、policy-changed、note、text、resumed、reopen，各带到达时间与已读时间。“已读”指被一步取用后又产生报告或回合结束，不等于送达。

## 谁能改

- 只有准入评估能建票
- 只有用户能撤销 mandate
- 只有执行会话能写 attention 和 report
- 宿主写 stage、undelivered、delivery
- 外部消息永不修改策略

## 幂等与去重

确定性 id，重试复用同 id 同文本：

```text
ack:<trigger>
attention-reply:<attention>
reply:<trigger|ticket>:<text>
```

- 一个 trigger 至多一票；accept 对 trigger id 幂等
- attach 仅在消息显式指向时使用（引用 PR、issue、来源消息或 ack，或“刚做的改动”且恰好一票匹配）；主题相似不算
- 错 attach 会订阅整条线程且不可撤，只在明确指向时用
- 重复票据可关闭，不重复执行

## Trae 落地

- ticket id 用短稳定串，例如 `t-<YYYYMMDD>-<序号>`；会话标题带同一 id
- 票据卡放在当前会话回复或计划中，用户可复核；不依赖不可见的内部状态
- 派工用 `create_session`，续派或追问用 `send_message_to_session`，同一票据保持同一会话
- 收报后用 `read_session` 核对；终态才更新票据，progress 不改变票据状态
- 用户 stop 时关闭对应执行会话；已 stop 的票据不再因后续消息自动重启
