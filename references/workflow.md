# Workflow Notes

## Directory Responsibilities

- `config/project_profile.json`
  - 跨会话持久化的总设定
  - 记录模式、文风、书本信息、前文来源、输出计数器
- `config/session_state.json`
  - 当前轮任务状态
  - 记录当轮字数目标、当前步骤、活动文件、用户临时想法
- `config/story_memory.json`
  - 前文摘要
  - 分章总结、剧情主线、未回收伏笔、时间线
- `config/character_memory.json`
  - 人物形象控制模块
  - 角色性格、说话习惯、行为模式、关系约束
- `config/builtin_writing_rules.json`
  - 默认内置写作约束
- `config/style_library.json`
  - 预设文风清单和自定义文风索引
- `style/`
  - 预设或分析后生成的文风提示词文本
- `pre-style/`
  - 用户上传的文风参考文章
- `text/`
  - 用户上传的前文
- `output/`
  - 最终正文输出

## First Run vs Later Runs

首次运行时：

1. 读取全部 JSON。
2. 如果 `project_profile.initialized` 为 `false`，按启动流程完整收集首轮关键信息。
3. 生成或补齐所需的摘要、人物模块、文风信息。
4. 写回所有相关 JSON，并把 `initialized` 改为 `true`。

后续运行时：

1. 先复用已有设定。
2. 只追问缺失字段或用户主动变更的字段。
3. 只有在源文件变化、模式变化或用户要求刷新时，才重建摘要、人物模块和文风分析。

## Acceptable Source Files

优先接受这些可读文本文件：

- `.txt`
- `.md`
- `.markdown`

若用户放入其他格式，只有在你能实际读取并确认内容有效时才算“上传完成”。

“真实可用”的最低标准：

- 文件非空
- 能读取到正文内容，不是乱码或占位文本
- 内容与本项目写作有关

## Long Context Heuristic

遇到 `continue` 或 `polish` 时，如果前文明显过长，不要浪费上下文窗口硬读到底。

可视为“明显过长”的经验条件：

- 单文件大于约 `40000` 字
- 或多个章节总量已经超过一次性稳定处理范围
- 或用户上传了多份长文，且本轮只需要延续局部情节

这时先询问：

`前文较长。要不要先按章节压缩成摘要，再进入本轮写作？`

如果用户同意压缩：

1. 先做全局摘要。
2. 再做分章摘要。
3. 标出人物状态、关系变化、关键设定、未解决线索。
4. 写入 `config/story_memory.json`。

如果用户拒绝压缩：

1. 只读取支撑本轮任务所需的正文范围。
2. 仍要在 `session_state` 里记录你实际参考了哪些文件。

## Character Control Module

人物模块至少包含这些维度：

- 角色名
- 身份与当前处境
- 核心性格
- 说话方式
- 常见行为模式
- 对其他关键人物的关系态度
- 不能违背的行为边界

把这些内容写入 `config/character_memory.json`，供后续所有轮次复用。

## Style Workflow

用户选择文风时，提供两条路：

1. 直接从 `config/style_library.json` 的预设中选
2. 上传参考文到 `pre-style/`

如果用户选择参考文：

1. 阅读参考文，不要照抄内容。
2. 提炼这些稳定特征：
   - 句长和句式密度
   - 叙述距离
   - 对话比例
   - 描写偏好
   - 节奏快慢
   - 常见禁区
3. 生成一个新的文风提示词文件，命名为：
   - `style/custom-YYYYMMDD-slug.txt`
4. 在 `config/style_library.json` 的 `custom_styles` 中登记：
   - `id`
   - `label`
   - `file`
   - `source_files`
   - `description`

## Question Checklist

### Shared Intake

- 书名或暂定名
- 类型与题材
- 当前进度
- 写作人称
- 特定要求
- 本轮字数目标

### start_long / start_short

除了 shared intake，再确认：

- 世界观或时代背景
- 主角与核心配角
- 主线冲突
- 当前章节目标
- 卖点与读者期待
- 禁止出现的设定和情节

### continue

除了 shared intake，再确认：

- 本轮要续写到哪里
- 必须衔接的最近剧情点
- 必须延续的人物情绪或关系状态
- 是否允许轻度扩写
- 哪些伏笔要继续保留

### polish

除了 shared intake，再确认：

- 本轮润色范围
- 是否允许扩写
- 是否允许重组段落
- 必须保留的台词、动作、结构
- 本轮是否只改文风，不改剧情

## Stop Condition Before Writing

在开始正文前，必须确认这六件事已经清楚：

1. 主视角与人称
2. 当前写作目标
3. 本轮边界
4. 人物约束
5. 文风约束
6. 交付单位

如果任何一项仍然模糊，继续追问。

## Output Naming Convention

统一使用零填充序号：

- `output/start-long-chapter-001.txt`
- `output/start-short-draft-001.txt`
- `output/continue-001.txt`
- `output/polish-001.txt`

如需写入多轮结果，继续递增计数器，并把已生成文件写回 JSON。

## Review Loop

每次交付输出后都执行：

1. 提醒用户当前交付单元已完成。
2. 请求用户给出修改意见。
3. 若用户要求改动，先按意见修订当前单元。
4. 修订完成后再询问是否继续下一单元。

没有用户确认，不要自作主张进入下一章或下一段。
