# Novel Writing Workflow

`novel-writing-workflow` 是一个面向中文网文、小说与长篇项目的 Agent Skill。它把“从头开始写作、续写、文本润色”三类工作统一进一个可持久化的流程里，并把项目状态、文风预设、前文摘要、人物控制和输出文件都保存在本地目录中，方便跨会话续用。

## 适用场景

- 从零开始搭建一本小说的写作流程
- 基于前文继续写作，并保持剧情脉络和人物一致性
- 对已有正文进行润色，而不是只给修改建议
- 在 Agent 中长期维护同一本书的配置、文风和人物控制

## 核心能力

- 支持四种模式：
  - `start_long`
  - `start_short`
  - `continue`
  - `polish`
- 首次运行会收集项目设定，并写入 `config/`
- 后续运行默认复用已有设定，只追问缺失或被修改的部分
- 对 `continue` 和 `polish` 模式支持前文摘要与人物控制模块
- 内置 5 套预设文风，也支持上传参考文自动抽取文风提示词
- 每次输出后都会停下来等待用户审核

## 目录结构

```text
novel-writing-workflow/
├── SKILL.md
├── README.md
├── LICENSE
├── .gitignore
├── agents/
│   └── openai.yaml
├── references/
│   └── workflow.md
├── config/
│   ├── builtin_writing_rules.json
│   ├── character_memory.json
│   ├── project_profile.json
│   ├── session_state.json
│   ├── story_memory.json
│   └── style_library.json
├── style/
│   ├── 01-简洁写实.txt
│   ├── 02-冷峻克制.txt
│   ├── 03-细腻抒情.txt
│   ├── 04-镜头感强.txt
│   └── 05-网文节奏.txt
├── pre-style/
├── text/
└── output/
```

## 在 Goose 中使用

Goose 会在项目目录或全局目录中发现 Skills。推荐把本仓库作为“项目级 Skill”使用，避免不同书稿的状态互相污染。

推荐目录：

```text
your-book-project/
└── .agents/
    └── skills/
        └── novel-writing-workflow/
            ├── SKILL.md
            ├── agents/
            ├── config/
            ├── style/
            ├── pre-style/
            ├── text/
            └── output/
```

使用步骤：

1. 把本仓库内容放到 `.agents/skills/novel-writing-workflow/`
2. 确保 Agent 已启用 Skills 能力，并能读取项目目录
3. 把前文放到 `text/`
4. 把文风参考文放到 `pre-style/`
5. 在 Agent 会话中明确调用：

```text
Use the novel-writing-workflow skill.
我想续写这本书。
```

## 工作流程概览

### 1. 模式选择

Skill 会先询问用户选择：

- 从头开始
- 续写
- 文本润色

如果是续写或润色，Skill 会要求用户先把前文放到 `text/`，并确认文件真实可读。

### 2. 共享信息收集

无论哪种模式，都会尽量收集并持久化这些信息：

- 书名或暂定名
- 类型和题材标签
- 当前进度
- 写作人称
- 特定要求
- 本轮字数目标

若是从头开始，还会继续询问：

- 每章字数
- 当前要写的章节数

如果章节数为 `1` 且字数大于 `10000`，会追问是否进入短篇模式。

### 3. 模式细分

#### 从头开始-长篇

- 选择预设文风，或上传参考文抽取文风
- 收集背景、想法和写作细节
- 先向用户复述理解
- 用户确认后按章输出
- 每章单独保存到 `output/`

#### 从头开始-短篇

- 流程与长篇相同
- 改为按约 `500-1000` 字的块输出
- 整篇持续保存到 `output/`

#### 续写

- 读取前文
- 若前文过长，先询问是否做压缩摘要
- 首次初始化时生成人物控制模块
- 追问本轮续写边界与细节
- 用户确认后按约 `500-1000` 字输出

#### 润色

- 读取前文
- 若前文过长，先询问是否做压缩摘要
- 首次初始化时生成人物控制模块
- 询问要处理的具体片段与润色边界
- 输出完整润色后的正文，而不是只给建议

## 配置文件说明

- `config/project_profile.json`
  - 项目总设定和跨会话状态
- `config/session_state.json`
  - 当前轮任务状态
- `config/story_memory.json`
  - 前文摘要与剧情脉络
- `config/character_memory.json`
  - 人物性格、行为模式和关系控制
- `config/builtin_writing_rules.json`
  - 内置写作要求
- `config/style_library.json`
  - 预设和自定义文风索引

## 文风系统

仓库内预置了 5 种风格：

- 简洁写实
- 冷峻克制
- 细腻抒情
- 镜头感强
- 网文节奏

如果用户上传参考文到 `pre-style/`，Skill 可以分析参考文，提炼叙述距离、句式密度、对话比例、描写偏好和节奏特征，再生成新的文风提示词文件到 `style/`。

## 内置写作约束

默认规则已经内置在 `config/builtin_writing_rules.json`，包括但不限于：

- 保持人物一致，不跳脱
- 优化对话、动作、描写的流畅度
- 尽量不删减内容，可适度扩充有效内容
- 不堆砌辞藻，避免 AI 味描写
- 使用中文全角标点和「」『』对话格式
- 不脱离前文情节脉络
- 输出完整正文，不只给修改意见

## 输出约定

- 长篇从头开始：每章一个文件
- 短篇从头开始：整篇一个文件
- 续写：每轮一个文件
- 润色：每轮一个文件

默认命名格式见 `references/workflow.md`。

## 许可证

本项目采用 `CC BY-NC-SA 4.0` 许可证，详见 [LICENSE](LICENSE)。
