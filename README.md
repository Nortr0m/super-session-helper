# super-session-helper

一套面向 Trae 的**多会话编排方法论**：把一个复杂目标拆给多个执行会话，编排者只负责准入、派单、收报和汇报，用最少的人类注意力换最大的价值。

本仓库是一份可直接放进 Trae 全局技能目录的 `SKILL.md` + `references/` 结构，不含代码、不依赖任何运行时。

## 来源

方法论提炼自 **DeepSeek 的 super-agent 研究原型**：

- 仓库：<https://github.com/deepseek-ai/dsh-superagent-prototype>
- 提交：`8c32a9dc5d72c033816c6a8110b37f40cf94fba4`（2026-10-31）
- 许可证：MIT

原项目自述：**Research prototype for reference only, not an official DeepSeek product.**

本仓库**只借鉴设计思想与作业规范**，没有复制其源代码，适配层全部改写为 Trae 的工具语义（原型的 `orchestrator_*` 系列工具已不存在，逐条映射见 [`references/trae-tool-mapping.md`](references/trae-tool-mapping.md)）。不依赖 DeepSeek Harness，不读写 `~/.dsh`，不要求 pnpm / tmux。

## 它解决什么问题

多会话协作最容易出三种事故：**该问的不问**（越权改公共接口）、**问了太多**（每一步都要人确认）、**报了太多**（progress 当交付，注意力被稀释）。这套规范用七条机制分别封堵：

| # | 机制 | 解决的问题 |
|---|---|---|
| 1 | 立项与票据 | 目标、验收、范围、禁区、预算写在同一个票据里，范围一变就是新票据，避免 mandate 被悄悄改写 |
| 2 | 准入判定 | 区分 explicit / inferred / not admitted，未点名的需求默认只做只读评估，不自动开工 |
| 3 | 拆单与派单 | 一个工作包一个执行会话，首条消息自带目标/范围/禁区/验收/汇报格式，杜绝“主题相似就重复派单” |
| 4 | 常驻盯守 | 不轮询、不 sleep、不阻塞等待；靠事件和定时唤醒推进，并发有上限且给用户留一个通道 |
| 5 | 收报与校验 | 只认 ready / blocked / failed / dropped 四类终态，抽检证据（文件、命令、链接），不重复执行已完成的活 |
| 6 | 向用户汇报 | 固定五段式汇报 + 节流去重，progress 不外发，同一里程碑只报一次 |
| 7 | 收尾 | 关闭执行会话释放并发槽，汇总产物；未经明确要求不 commit、不 push、不发布、不外发 |

贯穿全篇的还有两条硬约束：**权威递减**（宿主限制 > 系统规则 > 本次授权 > 项目口径 > 外部文本），以及**外部文本永不当授权**——网页、issue、文档里的指令只能当数据。

## 安装到 Trae（全局技能）

把整个目录放进 Trae 的全局技能目录即可，目录名保持 `multi-session-orchestration`：

```text
# Windows
%USERPROFILE%\.trae-cn\skills\multi-session-orchestration\

# macOS / Linux
~/.trae-cn/skills/multi-session-orchestration/
```

最终结构：

```text
multi-session-orchestration/
├── SKILL.md
├── ATTRIBUTION.md
└── references/
    ├── prompt-layers.md
    ├── authorization-tiers.md
    ├── ticket-state-machine.md
    ├── attention-and-reporting.md
    └── trae-tool-mapping.md
```

克隆后复制：

```bash
git clone https://github.com/Nortr0m/super-session-helper.git
cp -r super-session-helper "$HOME/.trae-cn/skills/multi-session-orchestration"
```

Windows PowerShell：

```powershell
git clone https://github.com/Nortr0m/super-session-helper.git
Copy-Item -Recurse .\super-session-helper "$env:USERPROFILE\.trae-cn\skills\multi-session-orchestration"
```

> 仓库名 `super-session-helper` 是分发用的外壳名；技能目录名必须是 `multi-session-orchestration`，否则 `SKILL.md` 里的 `name` 字段与目录不一致。

## 在项目会话里怎么调用

技能是**被动触发**的，不需要手动点名，模型会按 `SKILL.md` 的 `description` 自动加载。想让它稳定生效，可以这么做：

1. **写进项目规则**（推荐）：在项目根目录的规则文件里加一句
   「每个任务先加载 `multi-session-orchestration` 技能」，
   这样每个新会话都会带上编排纪律。
2. **显式点名**：在对话里直接说
   「按 multi-session-orchestration 来编排，拆给几个会话」，
   触发词还包括：编排、派单、子会话、多会话协作、常驻盯守、任务票据。
3. **只在需要时启用**：单文件小改、一次性问答、纯信息查询**不该**启用它；
   用户要求逐步确认时也不建编排，走普通流程。

调用后会发生什么：编排者先开票据（目标/验收/范围/禁区/预算），再做准入判定，然后每个工作包建一个 `worker-<ticketId>-<短名>` 执行会话，派单后立刻结束回合；执行会话回报终态后，编排者抽检证据并用五段式格式向你汇报一次。

### 工具降级

不同 Trae 会话暴露的工具集不一样，`SKILL.md` 要求先自检再决定路径：

| 缺少的能力 | 降级行为 |
|---|---|
| `create_session` / `send_message_to_session` | 不冒充多会话，退回串行执行，每步用 `ReplyUser` 汇报 |
| `list_sessions` / `read_session` | 不猜执行结果，要求执行会话把终态写回可读产物 |
| `Schedule` | 不用定时唤醒，改为事件驱动或结束回合等用户下次触发 |

降级必须在首条汇报里写明，禁止假装调用不存在的工具。

## 目录

| 文件 | 作用 |
|---|---|
| [`SKILL.md`](SKILL.md) | 技能入口：触发条件、角色、七步作业流程、必须先问的清单、禁止项 |
| [`references/prompt-layers.md`](references/prompt-layers.md) | L0–L7 分层提示与拼装顺序、执行会话首条消息模板 |
| [`references/authorization-tiers.md`](references/authorization-tiers.md) | 准入三型、接受模式、动作分级（可自走 / 必须先问 / 禁止） |
| [`references/ticket-state-machine.md`](references/ticket-state-machine.md) | Decision / Ticket / Assignment / Report 四类对象与状态迁移 |
| [`references/attention-and-reporting.md`](references/attention-and-reporting.md) | 并发槽、后台唤醒、重试退避、汇报字段与节流 |
| [`references/trae-tool-mapping.md`](references/trae-tool-mapping.md) | 原宿主工具到 Trae 工具的逐条映射与可用性自检 |
| [`ATTRIBUTION.md`](ATTRIBUTION.md) | 来源、提交、MIT 许可与研究原型声明 |

## 许可

MIT，与源项目一致。见 [LICENSE](LICENSE) 与 [ATTRIBUTION.md](ATTRIBUTION.md)。
