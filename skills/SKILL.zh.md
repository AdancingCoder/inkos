---
name: inkos
description: 自动化小说写作 CLI Agent - 用于创意小说写作、小说生成、文风仿写、章节续写/导入、EPUB 导出、AIGC 检测和同人创作。原生支持英文写作，内置 10 种英文题材配置（LitRPG、升级流、异世界、修仙、末日系统、地下城核心、浪漫奇幻、科幻、爬塔、治愈奇幻）。同时支持中文网文题材（玄幻、仙侠、都市、恐怖、其他）。多 Agent 管线、两阶段写手（创意写作 + 状态结算）、33 维度审计、Token 用量分析、创作简报输入、结构化日志（JSON Lines）、多模型路由，以及自定义 OpenAI 兼容接口支持。
version: 1.5.0
metadata: { "openclaw": { "emoji": "📖", "requires": { "bins": ["inkos", "node"], "env": [] }, "primaryEnv": "", "homepage": "https://github.com/Narcooo/inkos", "install": [{ "id": "npm", "kind": "node", "package": "@actalk/inkos", "label": "通过 npm 安装 InkOS" }] } }
---

# InkOS - 自动化小说写作 Agent

InkOS 是一个由 LLM Agent 驱动的自动化小说写作 CLI 工具。它编排 5 个 Agent 组成的管线（雷达 → 建筑师 → 写手 → 审计员 → 修订者）来生成、审计和修订小说内容，确保风格一致性和质量控制。

写手采用两阶段架构：第一阶段（创意写作，温度 0.7）生成章节正文，第二阶段（状态结算，温度 0.3）更新所有真相文件以保持长期一致性。

## 何时使用 InkOS

- **英文小说写作**：原生支持英文，内置 10 种题材配置（LitRPG、升级流、异世界等）。设置 `--lang en`
- **中文网文写作**：内置 5 种中文题材（玄幻、仙侠、都市、恐怖、其他）
- **同人创作**：从原作素材创建同人，支持 4 种模式（正典、架空、OOC、CP 向）
- **批量章节生成**：生成多章节并保持质量一致
- **导入续写**：从文本文件导入已有章节，逆向工程真相文件，继续创作
- **文风仿写**：分析并采用参考文本的写作风格
- **番外写作**：写前传/续集/番外，同时保持与正传的一致性
- **质量审计**：检测 AI 生成内容，执行 33 维度质量检查
- **题材探索**：探索市场趋势，创建自定义题材规则
- **数据分析**：追踪字数、审计通过率、问题分布

## 初始配置

### 首次配置
```bash
# 初始化项目目录（创建配置结构）
inkos init my-writing-project

# 配置 LLM 提供商（OpenAI、Anthropic 或任何 OpenAI 兼容 API）
inkos config set-global --provider openai --base-url https://api.openai.com/v1 --api-key sk-xxx --model gpt-4o
# 对于兼容/代理端点，使用 --provider custom：
# inkos config set-global --provider custom --base-url https://your-proxy.com/v1 --api-key sk-xxx --model gpt-4o
```

### 多模型路由（可选）
```bash
# 为不同 Agent 分配不同模型 — 平衡质量与成本
inkos config set-model writer claude-sonnet-4-20250514 --provider anthropic --base-url https://api.anthropic.com --api-key-env ANTHROPIC_API_KEY
inkos config set-model auditor gpt-4o --provider openai
inkos config show-models
```
未显式配置的 Agent 会回退到全局模型。

### 查看系统状态
```bash
# 检查安装和配置
inkos doctor

# 查看当前配置
inkos status
```

## 常用工作流

### 工作流 1：创建新小说

1. **初始化并创建书籍**：
   ```bash
   inkos book create --title "我的小说标题" --genre xuanhuan --chapter-words 3000
   # 或者带创作简报（你的世界观设定/想法文档）：
   inkos book create --title "我的小说标题" --genre xuanhuan --chapter-words 3000 --brief my-ideas.md
   ```
   - 题材：`xuanhuan`（玄幻）、`xianxia`（仙侠）、`urban`（都市）、`horror`（恐怖）、`other`（其他）
   - 返回 `book-id` 用于后续所有操作

2. **生成初始章节**（例如 5 章）：
   ```bash
   inkos write next book-id --count 5 --words 3000 --context "年轻主角发现自己的力量"
   ```
   - `write next` 命令运行完整管线：草稿 → 审计 → 修订
   - `--context` 为建筑师和写手 Agent 提供指导
   - 返回包含章节详情和质量指标的 JSON

3. **审阅并通过章节**：
   ```bash
   inkos review list book-id
   inkos review approve-all book-id
   ```

4. **导出书籍**（支持 txt、md、epub）：
   ```bash
   inkos export book-id
   inkos export book-id --format epub
   ```

### 工作流 2：继续写作已有小说

1. **列出你的书籍**：
   ```bash
   inkos book list
   ```

2. **从上一章继续**：
   ```bash
   inkos write next book-id --count 3 --words 2500 --context "主角面临关键抉择"
   ```
   - InkOS 维护 7 个真相文件（世界状态、角色矩阵、情感弧线等）以保持一致性
   - 如果只有一本书，可省略 `book-id` 自动检测

3. **审阅并通过**：
   ```bash
   inkos review approve-all
   ```

### 工作流 3：导入已有章节并续写

当你有一部已有小说（或部分小说）并希望 InkOS 接续创作时使用。

1. **从单个文本文件导入**（按章节标题自动分割）：
   ```bash
   inkos import chapters book-id --from novel.txt
   ```
   - 自动按 `第X章` 模式分割
   - 自定义分割模式：`--split "Chapter\\s+\\d+"`

2. **从目录导入**独立的章节文件：
   ```bash
   inkos import chapters book-id --from ./chapters/
   ```
   - 按排序顺序读取 `.md` 和 `.txt` 文件

3. **恢复中断的导入**：
   ```bash
   inkos import chapters book-id --from novel.txt --resume-from 15
   ```

4. **从导入的章节继续写作**：
   ```bash
   inkos write next book-id --count 3
   ```
   - InkOS 从导入的章节逆向工程所有 7 个真相文件
   - 从现有文本生成风格指南
   - 新章节与导入内容保持一致

### 工作流 4：文风仿写

1. **分析参考文本**：
   ```bash
   inkos style analyze reference_text.txt
   ```
   - 检查词汇、句子结构、语气、节奏

2. **将风格导入你的书籍**：
   ```bash
   inkos style import reference_text.txt book-id --name "作者名"
   ```
   - 所有后续章节采用此风格配置
   - 风格规则成为修订者审计标准的一部分

### 工作流 5：番外/前传写作

1. **导入正传正典**：
   ```bash
   inkos import canon spinoff-book-id --from parent-book-id
   ```
   - 创建与正传世界状态、角色和事件的链接
   - 修订者强制执行正典一致性

2. **继续番外**：
   ```bash
   inkos write next spinoff-book-id --count 3 --context "第 20 章之后的平行时间线"
   ```

### 工作流 6：精细控制（草稿 → 审计 → 修订）

如果你需要对管线各阶段进行单独控制：

1. **仅生成草稿**：
   ```bash
   inkos draft book-id --words 3000 --context "主角逃脱" --json
   ```

2. **审计章节**（33 维度质量检查）：
   ```bash
   inkos audit book-id chapter-1 --json
   ```
   - 返回 33 个维度的指标，包括节奏、对话、世界构建、大纲遵循等

3. **使用特定模式修订**：
   ```bash
   inkos revise book-id chapter-1 --mode polish --json
   ```
   - 模式：`polish`（润色）、`spot-fix`（定点修复）、`rewrite`（重写）、`rework`（结构调整）、`anti-detect`（去 AI 味）

### 工作流 7：监控平台趋势

```bash
inkos radar scan
```
- 分析热门题材、套路和读者偏好
- 为建筑师的新书建议提供参考

### 工作流 8：检测 AI 生成内容

```bash
# 检测特定章节的 AIGC
inkos detect book-id

# 深度扫描所有章节
inkos detect book-id --all
```
- 使用 11 条确定性规则（零 LLM 成本）+ 可选 LLM 验证
- 返回检测置信度和问题段落

### 工作流 9：查看数据分析

```bash
inkos analytics book-id --json
# 简写别名
inkos stats book-id --json
```
- 总章节数、字数、平均每章字数
- 审计通过率和主要问题类别
- 问题最多的章节、状态分布
- **Token 用量统计**：总 prompt/completion tokens、每章平均 tokens、近期趋势

### 工作流 10：写英文小说

```bash
# 创建英文 LitRPG 小说（语言从题材自动检测）
inkos book create --title "The Last Delver" --genre litrpg --chapter-words 3000

# 或显式设置语言
inkos book create --title "My Novel" --genre other --lang en

# 将英文设为所有项目的默认语言
inkos config set-global --lang en
```
- 10 种英文题材：litrpg、progression、isekai、cultivation、system-apocalypse、dungeon-core、romantasy、sci-fi、tower-climber、cozy
- 每种题材有专门的节奏规则、疲劳词表（如 "delve"、"tapestry"、"testament"）和审计维度
- 使用 `inkos genre list` 查看所有可用题材

### 工作流 11：同人创作

```bash
# 从原作素材创建同人
inkos fanfic init --title "我的同人" --from source-novel.txt --mode canon

# 模式：canon（正典延续）、au（架空世界）、ooc（性格重塑）、cp（CP 向）
inkos fanfic init --title "如果" --from source.txt --mode au --genre other
```
- 自动导入和分析原作素材
- 同人专属审计维度和信息边界控制
- 确保新内容与原作正典保持一致（或在 au/ooc 模式下有意偏离）

## 高级：自然语言 Agent 模式

用于灵活的对话式请求：

```bash
inkos agent "写一部都市题材的小说，主角是一个年轻律师，第一章三千字"
```
- Agent 解释自然语言并调用适当的命令
- 适用于复杂的多步骤请求

## 核心概念

### Book ID 自动检测
如果你的项目只有一本书，大多数命令的 `book-id` 参数是可选的。可以省略以简化操作：
```bash
# 显式指定
inkos write next book-123 --count 1

# 自动检测（如果只有一本书）
inkos write next --count 1
```

### --json 标志
所有内容生成命令都支持 `--json` 以获取结构化输出。对于程序化使用至关重要：
```bash
inkos draft book-id --words 3000 --context "指导" --json
```

### 真相文件（长期记忆）
InkOS 为每本书维护 7 个文件以保持连贯性：
- **世界状态**：地图、地点、科技水平、魔法体系
- **角色矩阵**：姓名、关系、弧线、动机
- **资源账本**：世界内物品、金钱、力量等级
- **章节摘要**：事件、进展、伏笔
- **支线进度板**：活跃和休眠的支线、钩子
- **情感弧线**：角色情感进展
- **待处理钩子**：未解决的悬念和对读者的承诺

所有 Agent 都引用这些文件以保持长期一致性。在 `import chapters` 期间，这些文件通过 ChapterAnalyzerAgent 从现有内容逆向工程。

### 两阶段写手架构
写手 Agent 分两个阶段运行：
- **第一阶段（创意）**：以温度 0.7 生成章节文本，用于创意表达。仅输出章节标题和内容。
- **第二阶段（结算）**：以温度 0.3 更新所有真相文件，用于精确状态追踪。确保世界状态、角色弧线和情节钩子保持一致。

这种分离允许写作时的创意自由，同时保持严格的连续性追踪。

### 上下文指导
`--context` 参数为写手和建筑师提供方向性提示：
```bash
inkos write next book-id --count 2 --context "主角发现背叛，必须决定是否信任导师"
```
- 上下文是可选的，但强烈建议用于叙事连贯性
- 支持英文和中文

## 题材管理

### 查看内置题材
```bash
inkos genre list
inkos genre show xuanhuan
```

### 创建自定义题材
```bash
inkos genre create --name "my-genre" --rules "规则1,规则2,规则3"
```

### 复制并修改现有题材
```bash
inkos genre copy xuanhuan --name "dark-xuanhuan" --rules "更黑暗的基调，更多暴力"
```

## 命令参考摘要

| 命令 | 用途 | 备注 |
|------|------|------|
| `inkos init [name]` | 初始化项目 | 一次性设置 |
| `inkos book create` | 创建新书 | 返回 book-id。`--brief <file>`、`--lang en/zh`、`--genre litrpg/progression/...` |
| `inkos book list` | 列出所有书籍 | 显示 ID、状态 |
| `inkos write next` | 完整管线（草稿→审计→修订） | 主要工作流命令 |
| `inkos draft` | 仅生成草稿 | 无审计/修订 |
| `inkos audit` | 33 维度质量检查 | 独立评估 |
| `inkos revise` | 修订章节 | 模式：polish/spot-fix/rewrite/rework/anti-detect |
| `inkos agent` | 自然语言接口 | 灵活请求 |
| `inkos style analyze` | 分析参考文本 | 提取风格配置 |
| `inkos style import` | 将风格应用到书籍 | 使风格永久化 |
| `inkos import canon` | 链接番外到正传 | 用于前传/续集 |
| `inkos import chapters` | 导入已有章节 | 逆向工程真相文件以续写 |
| `inkos detect` | AIGC 检测 | 标记 AI 生成的段落 |
| `inkos export` | 导出完成的书籍 | 格式：txt、md、epub |
| `inkos analytics` / `inkos stats` | 查看书籍统计 | 字数、审计率、token 用量 |
| `inkos radar scan` | 平台趋势分析 | 为新书想法提供参考 |
| `inkos config set-global` | 配置 LLM 提供商 | OpenAI/Anthropic/custom（任何 OpenAI 兼容） |
| `inkos config set-model <agent> <model>` | 为特定 agent 设置模型覆盖 | `--provider`、`--base-url`、`--api-key-env` 用于多提供商路由 |
| `inkos config show-models` | 显示当前模型路由 | 查看每个 agent 的模型分配 |
| `inkos doctor` | 诊断问题 | 检查安装 |
| `inkos update` | 更新到最新版本 | 自我更新 |
| `inkos up/down` | 守护进程模式 | 后台处理。日志写入 `inkos.log`（JSON Lines）。`-q` 静默模式 |
| `inkos review list/approve-all` | 管理章节审批 | 质量门控 |
| `inkos fanfic init` | 从原作素材创建同人 | `--from <file>`、`--mode canon/au/ooc/cp` |
| `inkos genre list` | 列出所有可用题材 | 显示英文和中文题材及默认语言 |

## 错误处理

### 常见问题

**"book-id not found"**
- 使用 `inkos book list` 验证 ID
- 确保在正确的项目目录中

**"Provider not configured"**
- 使用有效凭据运行 `inkos config set-global`
- 使用 `inkos doctor` 检查 API key 和 base URL

**"Context invalid"**
- 确保 `--context` 是字符串（多词时用引号包裹）
- 上下文可以是英文或中文

**"Audit failed"**
- 检查章节是否有编码问题
- 确保 chapter-words 与实际字数匹配
- 尝试使用 `--mode rewrite` 运行 `inkos revise`

**"Book already has chapters"（导入时）**
- 使用 `--resume-from <n>` 追加到现有章节
- 或先删除现有章节

### 运行守护进程模式

用于长时间运行的操作：
```bash
# 启动后台守护进程
inkos up

# 停止守护进程
inkos down

# 守护进程自动处理排队的章节
```

## 最佳实践

1. **提供丰富的上下文**：`--context` 中的指导越多，叙事越连贯
2. **先导入风格**：如果要仿写某作者，在生成前先运行 `inkos style import`
3. **先导入章节**：对于已有小说，使用 `inkos import chapters` 引导真相文件后再续写
4. **定期审阅**：使用 `inkos review` 尽早发现问题
5. **监控审计**：检查 `inkos audit` 指标以了解质量瓶颈
6. **战略性使用番外**：写前传/续集前先导入正典
7. **批量生成**：一起生成多章（连续性更好）
8. **检查分析**：使用 `inkos analytics` 追踪质量趋势
9. **频繁导出**：使用 `inkos export` 保持备份

## 支持与资源

- **主页**：https://github.com/Narcooo/inkos
- **配置**：`inkos init` 后存储在项目根目录
- **真相文件**：位于每本书的 `.inkos/` 目录
- **日志**：检查 `inkos doctor` 的输出进行故障排除