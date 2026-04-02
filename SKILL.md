---
name: novel-writing-workflow
description: Stateful workflow for Chinese fiction and webnovel projects. Use when the user wants to start from scratch, continue from uploaded previous text, or polish existing prose while persisting JSON config, style presets, summaries, character controls, and output files across sessions.
---

# Novel Writing Workflow

## Overview

这个 Skill 用于管理中文小说项目的完整写作流程。它支持四个落地模式：

- `start_long`
- `start_short`
- `continue`
- `polish`

它不是一次性提示词。你要把每次收集到的设定、前文摘要、人物控制、文风选择和输出文件持续写回本地目录，让后续轮次直接复用。

## Always Load

每次进入任务时，先读取这些文件：

- `config/project_profile.json`
- `config/session_state.json`
- `config/builtin_writing_rules.json`
- `config/style_library.json`

在以下情况补充读取：

- `continue` 或 `polish`：读取 `config/story_memory.json`、`config/character_memory.json`，再检查 `text/`
- 用户上传文风参考文：检查 `pre-style/`
- 用户要求续用历史输出：检查 `output/`

如果这些 JSON 已经有可用值，优先沿用，不要重复追问同一个问题，除非用户明确要改。

详细字段约定、命名规则和追问清单见 `references/workflow.md`。

## Core Rules

- 一次只问一个紧凑问题；只有高度相关的字段才允许合并提问。
- 对于 `continue` 和 `polish`，在确认 `text/` 里存在真实可读且非空的前文文件前，不进入后续流程。
- 在用户确认你的理解无误之前，不开始生成正文。
- 每次生成一个交付单元后，都要停下来征求用户修改意见。
- 所有写作都必须同时遵守：用户当轮要求、已落盘 JSON、活动文风文件、`config/builtin_writing_rules.json`。
- 不得越出用户批准的剧情范围。`polish` 模式严禁偷偷续写到用户未提供的后文。
- 使用中文全角标点；对话默认使用「」或『』。

## Startup Flow

按下面顺序执行：

1. 询问用户要 `从头开始`、`续写` 还是 `文本润色`。
2. 如果是 `从头开始`，直接跳到“Shared Intake”。
3. 如果是 `续写` 或 `文本润色`，询问用户是否已将前文上传到 `text/`。
4. 如果没有上传，明确要求用户把前文放到 `text/`，然后重新询问是否已上传完成。
5. 只有当 `text/` 中至少存在一个可读取的非空文本文件时，才允许继续。把文件路径写入 `config/project_profile.json` 和 `config/session_state.json`。

## Shared Intake

完成模式选择后，按以下逻辑补齐信息：

1. 如缺失，询问书本基础信息，至少覆盖：书名或暂定名、类型、题材标签、当前进度。
2. 询问写作人称。
3. 询问是否有特定写作要求。
4. 询问写作字数。
5. 如果当前模式是 `从头开始`，必须继续询问：
   - 每章字数
   - 当前需要书写的章节数
6. 如果当前模式是 `从头开始`，并且用户给出的章节数为 `1` 且字数大于 `10000`，必须追问是否进入短篇模式。
7. 立刻把当前收集到的信息写回：
   - `config/project_profile.json`
   - `config/session_state.json`

模式判定规则：

- `从头开始` 且用户确认短篇模式：`start_short`
- `从头开始` 且其余情况：`start_long`
- `续写`：`continue`
- `文本润色`：`polish`

## Mode: start_long

1. 询问用户文风。
2. 提供 `config/style_library.json` 中的预设文风供选择；或者让用户把参考文章放进 `pre-style/`。
3. 如果用户上传了参考文章，读取并抽取稳定文风特征，生成新的提示词文件到 `style/`，并更新 `config/style_library.json`。
4. 询问写作背景。
5. 询问用户现有想法。
6. 继续追问关键写作细节，直到你可以稳定写出第一章。
7. 用清晰摘要复述你理解的写作方向，并询问是否开始写作。
8. 如果用户确认开始，按“章”输出，每次只交付一章。
9. 每章输出后都要等待用户审核和修改意见，再决定是否继续下一章。
10. 每章保存为单独的 `txt` 文件到 `output/`。

## Mode: start_short

1. 文风、参考文、背景、想法、追问流程与 `start_long` 相同。
2. 在用户确认方向后，按约 `500-1000` 字的自然段落块输出。
3. 每个输出块都要征求用户审核和修改意见。
4. 把整篇短篇持续汇总到一个 `txt` 文件中，保存到 `output/`。

## Mode: continue

1. 先阅读 `text/` 中的前文。
2. 如果你判断前文明显过长，不要直接硬读到底，先询问用户是否需要前文压缩。
3. 如果用户选择需要压缩，按章节总结，保留剧情主题脉络、角色状态和未回收线索，并写入 `config/story_memory.json`。
4. 如果用户选择不压缩，再读取完成本轮写作所需的正文范围。
5. 首次初始化时，生成人物形象控制模块，写入 `config/character_memory.json`。
6. 之后默认沿用已有摘要和人物模块；只有源文件明显变更或用户要求刷新时才重建。
7. 询问用户现有想法。
8. 继续追问写作细节，直到本轮续写目标、场景边界和情绪方向足够清楚。
9. 复述你的理解并征求确认。
10. 用户确认后，按约 `500-1000` 字的连续文本块输出；保持章节边界清楚，不要跨到用户未批准的下一章。
11. 每个输出块都要等待审核。
12. 续写内容保存到 `output/` 的 `txt` 文件中。

## Mode: polish

1. 先阅读 `text/` 中的前文。
2. 如果前文明显过长，先询问用户是否需要前文压缩。
3. 如需压缩，按章节总结并写入 `config/story_memory.json`。
4. 如不压缩，则读取完成本轮润色所需的正文范围。
5. 首次初始化时，生成人物形象控制模块，写入 `config/character_memory.json`。
6. 之后默认沿用已有摘要和人物模块；只有源文件明显变更或用户要求刷新时才重建。
7. 询问用户已经写完、并且本轮希望你处理的部分。
8. 询问用户现有想法。
9. 继续追问润色细节，直到范围、力度、保留策略和禁止项都明确。
10. 复述你的理解并征求确认。
11. 用户确认后，按约 `500-1000` 字的输出块给出润色后的完整正文。
12. 每个输出块都要等待审核。
13. 润色内容保存到 `output/` 的 `txt` 文件中。

## Follow-Up Questions

当信息还不够时，继续追问。优先补齐这些缺口：

- 主角是谁，当前视角是谁
- 当前章节或片段的目标是什么
- 冲突点、情绪点、悬念点是什么
- 哪些人物必须出场，哪些人物不能出场
- 这一轮允许推进到哪里，不能越过哪里
- 是否有必须保留的台词、动作、设定、伏笔
- 是否有明确禁区，例如不能新增情节、不能改口吻、不能改节奏

只有当你已经清楚“人物、目标、冲突、边界、语气、交付单位”时，才进入写作。

## Persistence Rules

- 首次询问后要把稳定设定写入 `config/project_profile.json`。
- 每轮当前状态写入 `config/session_state.json`。
- 前文摘要写入 `config/story_memory.json`。
- 人物控制模块写入 `config/character_memory.json`。
- 内置写作要求固定读取 `config/builtin_writing_rules.json`。
- 预设和自定义文风索引写入 `config/style_library.json`。

如文件缺失，则按现有结构补建；如字段已存在，则增量更新，不要随意清空旧值。

## Output Rules

- 长篇从头开始：每章一个文件。
- 短篇从头开始：整篇一个文件，持续覆写或追加。
- 续写：每轮续写一个文件，必要时再累积总稿。
- 润色：每轮润色一个文件，输出必须是完整处理后的文本。

文件命名、前文过长的判断口径、建议追问项见 `references/workflow.md`。
