# 来源与许可

本仓库的方法论提炼自以下开源仓库，仅借鉴设计思想与作业规范，未复制其源代码，也未安装或运行该项目。

- 仓库：https://github.com/deepseek-ai/dsh-superagent-prototype
- 提交：`8c32a9dc5d72c033816c6a8110b37f40cf94fba4`（2026-10-31）
- 许可证：MIT

原项目自述（README 原文）：

> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software.

## 改写说明

- 只提炼机制：分层提示、准入与审批分级、票据状态机、单目标盯守、汇报与节流、沟通协议。
- 原型的宿主专用工具（`orchestrator_read`、`orchestrator_ack`、`orchestrator_report` 等）已全部改写为 Trae 会话工具，见 `references/trae-tool-mapping.md`。
- 不依赖 DeepSeek Harness，不读写 `~/.dsh`，不要求 pnpm、tmux 或任何该项目运行时。
- 原仓库中的平台适配细节（GitHub、飞书 API）未纳入；需要时按同样结构另写适配层。

## MIT License

```text
MIT License

Copyright (c) 2026 DeepSeek Harness Team

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
