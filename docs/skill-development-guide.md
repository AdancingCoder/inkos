# Skill 开发指南

> 本指南详细介绍如何为 AI Agent（如 Claude Code、OpenClaw）开发 Skill，使你的 CLI 工具能够被 AI 智能调用。

## 目录

- [一、什么是 Skill](#一什么是-skill)
- [二、Skill 文件结构](#二skill-文件结构)
  - [2.1 文件位置](#21-文件位置)
  - [2.2 Frontmatter 元数据](#22-frontmatter-元数据)
  - [2.3 正文内容结构](#23-正文内容结构)
- [三、编写 Skill 的最佳实践](#三编写-skill-的最佳实践)
- [四、CLI 工具设计原则](#四cli-工具设计原则)
- [五、完整示例](#五完整示例)
- [六、发布与分发](#六发布与分发)
- [七、常见问题](#七常见问题)

---

## 一、什么是 Skill

**Skill** 是一种让 AI Agent 理解并调用外部 CLI 工具的标准化描述文件。它本质上是一个 Markdown 文档，包含：

1. **元数据** - 告诉 AI 这个工具是什么、何时使用
2. **使用指南** - 告诉 AI 如何正确调用命令
3. **工作流示例** - 告诉 AI 如何组合命令完成复杂任务

### Skill 的工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI Agent                                 │
│  (Claude Code / OpenClaw / 其他兼容 Agent)                        │
│                                                                  │
│  1. 读取 SKILL.md 理解工具能力                                     │
│  2. 根据用户意图选择合适的命令                                      │
│  3. 通过 exec 调用 CLI 命令                                        │
│  4. 解析 --json 输出，决定下一步                                    │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                       你的 CLI 工具                               │
│                                                                  │
│  $ my-cli command --option value --json                          │
│  {"result": "...", "status": "success"}                          │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么需要 Skill

| 传统方式 | Skill 方式 |
|----------|------------|
| AI 不知道你的工具存在 | AI 主动发现并使用 |
| 用户需要手动输入命令 | AI 自动选择正确命令 |
| 难以组合多个命令 | AI 按工作流编排 |
| 输出难以被 AI 理解 | JSON 输出易于解析 |

---

## 二、Skill 文件结构

### 2.1 文件位置

```
your-project/
├── skills/
│   └── SKILL.md          # Skill 定义文件（必须）
├── package.json
└── src/
    └── ...
```

**命名规范**:
- 文件必须命名为 `SKILL.md`（大写）
- 放在项目根目录的 `skills/` 文件夹中

### 2.2 Frontmatter 元数据

Skill 文件以 YAML frontmatter 开头，定义关键元数据：

```yaml
---
name: my-skill
description: 简短但完整的描述，说明这个 skill 做什么、何时使用。这段描述会被 AI 用来判断是否应该调用此 Skill。
version: 1.0.0
metadata: {
  "openclaw": {
    "emoji": "🔧",
    "requires": {
      "bins": ["my-cli", "node"],
      "env": ["MY_API_KEY"]
    },
    "primaryEnv": "MY_API_KEY",
    "homepage": "https://github.com/user/my-project",
    "install": [
      {
        "id": "npm",
        "kind": "node",
        "package": "@scope/my-package",
        "label": "Install via npm"
      }
    ]
  }
}
---
```

#### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | Skill 名称，用于调用和安装 |
| `description` | ✅ | 详细描述，帮助 AI 判断何时使用。越具体越好 |
| `version` | ✅ | 语义化版本号 |
| `metadata.openclaw.emoji` | ❌ | 展示用的 emoji |
| `metadata.openclaw.requires.bins` | ❌ | 依赖的可执行文件列表 |
| `metadata.openclaw.requires.env` | ❌ | 依赖的环境变量列表 |
| `metadata.openclaw.primaryEnv` | ❌ | 主要的环境变量（用于配置提示） |
| `metadata.openclaw.homepage` | ❌ | 项目主页 URL |
| `metadata.openclaw.install` | ❌ | 安装方式列表 |

#### description 编写技巧

**好的 description**:
```yaml
description: Autonomous novel writing CLI agent - use for creative fiction writing, novel generation, style imitation, chapter continuation/import, EPUB export, AIGC detection, and fan fiction.
```

**为什么好**:
- 明确说明工具类型：`CLI agent`
- 列出所有使用场景：`creative fiction writing, novel generation...`
- 帮助 AI 精准匹配用户意图

**差的 description**:
```yaml
description: A tool for writing.
```

**为什么差**:
- 太模糊，AI 难以判断何时使用
- 没有列出具体功能

### 2.3 正文内容结构

Frontmatter 之后是 Markdown 正文，推荐结构如下：

```markdown
# 工具名称 - 简短描述

一段话介绍工具的核心功能和架构。

## When to Use

明确列出适合使用此工具的场景：

- **场景1**: 具体描述
- **场景2**: 具体描述
- ...

## Initial Setup

### First Time Setup
```bash
# 初始化命令
my-cli init
my-cli config --api-key xxx
```

### View System Status
```bash
my-cli doctor
my-cli status
```

## Common Workflows

### Workflow 1: 工作流名称

1. **步骤1**:
   ```bash
   my-cli command1 --option value
   ```
   - 说明这个命令做什么
   - 返回什么

2. **步骤2**:
   ```bash
   my-cli command2 --json
   ```

### Workflow 2: 另一个工作流
...

## Command Reference Summary

| Command | Purpose | Notes |
|---------|---------|-------|
| `my-cli cmd1` | 做什么 | 选项说明 |
| `my-cli cmd2` | 做什么 | 选项说明 |

## Key Concepts

### 概念1
解释重要概念...

### 概念2
解释重要概念...

## Error Handling

### Common Issues

**"错误信息1"**
- 原因和解决方案

**"错误信息2"**
- 原因和解决方案

## Tips for Best Results

1. **技巧1**: 具体建议
2. **技巧2**: 具体建议
```

---

## 三、编写 Skill 的最佳实践

### 3.1 以工作流为中心组织内容

**推荐** ✅:
```markdown
## Workflow 1: Create a New Novel

1. Initialize and create book:
   ```bash
   inkos book create --title "My Novel" --genre xuanhuan
   ```

2. Generate initial chapters:
   ```bash
   inkos write next book-id --count 5
   ```

3. Review and approve:
   ```bash
   inkos review approve-all book-id
   ```
```

**不推荐** ❌:
```markdown
## Commands

- `inkos book create` - 创建书籍
- `inkos write next` - 写下一章
- `inkos review approve-all` - 批量通过
```

**为什么**：AI 需要理解命令的组合顺序和上下文，工作流比命令列表更有指导性。

### 3.2 提供足够的上下文

**推荐** ✅:
```markdown
```bash
inkos write next book-id --count 5 --words 3000 --context "young protagonist discovering powers"
```
- The `write next` command runs the full pipeline: draft → audit → revise
- `--context` provides guidance to the Architect and Writer agents
- Returns JSON with chapter details and quality metrics
```

**不推荐** ❌:
```markdown
```bash
inkos write next book-id --count 5
```
```

### 3.3 明确输入输出

```markdown
### Workflow 3: Fine-Grained Control

1. **Generate draft only**:
   ```bash
   inkos draft book-id --words 3000 --context "protagonist escapes" --json
   ```

2. **Audit the chapter** (33-dimension quality check):
   ```bash
   inkos audit book-id chapter-1 --json
   ```
   - Returns metrics across 33 dimensions including pacing, dialogue, world-building
```

### 3.4 覆盖边界情况

```markdown
### Book ID Auto-Detection
If your project contains only one book, most commands accept `book-id` as optional:
```bash
# Explicit
inkos write next book-123 --count 1

# Auto-detected (if only one book exists)
inkos write next --count 1
```
```

### 3.5 提供错误处理指南

```markdown
## Error Handling

### Common Issues

**"book-id not found"**
- Verify the ID with `inkos book list`
- Ensure you're in the correct project directory

**"Provider not configured"**
- Run `inkos config set-global` with valid credentials
- Check API key and base URL with `inkos doctor`
```

---

## 四、CLI 工具设计原则

要让你的 CLI 工具成为一个好的 Skill，需要遵循以下设计原则：

### 4.1 支持 `--json` 输出

这是最重要的原则。AI 需要解析命令输出来决定下一步操作。

```typescript
// 命令实现示例
command
  .option("--json", "Output JSON")
  .action(async (opts) => {
    const result = await doSomething();

    if (opts.json) {
      // 结构化输出，供 AI 解析
      console.log(JSON.stringify(result, null, 2));
    } else {
      // 人类可读输出
      console.log(`Result: ${result.summary}`);
    }
  });
```

### 4.2 原子化命令设计

每个命令应该做一件事，并且可以独立运行：

```bash
# 好的设计 - 原子化
inkos draft book-id          # 只写草稿
inkos audit book-id 31       # 只审计
inkos revise book-id 31      # 只修订

# 也提供组合命令
inkos write next book-id     # 完整管线：草稿 → 审计 → 修订
```

### 4.3 一致的错误输出

```typescript
if (opts.json) {
  console.log(JSON.stringify({ error: String(e) }));
} else {
  console.error(`Error: ${e.message}`);
}
process.exit(1);
```

### 4.4 幂等性

相同输入产生相同输出，便于 AI 重试：

```bash
# 多次运行结果一致
inkos status book-id --json
inkos status book-id --json
```

### 4.5 清晰的返回值

```typescript
interface CommandResult {
  success: boolean;
  data?: any;
  error?: string;
  nextSteps?: string[];  // 可选：建议的下一步操作
}
```

---

## 五、完整示例

### 5.1 最小 Skill 示例

```markdown
---
name: greeter
description: A simple greeter CLI - use for generating personalized greetings in various styles (formal, casual, funny).
version: 1.0.0
metadata: { "openclaw": { "emoji": "👋", "requires": { "bins": ["greeter"] }, "install": [{ "id": "npm", "kind": "node", "package": "greeter-cli" }] } }
---

# Greeter - Personalized Greeting Generator

Generate greetings in various styles.

## When to Use

- Generate formal greetings for business emails
- Create casual greetings for social media
- Make funny greetings for entertainment

## Setup

```bash
npm install -g greeter-cli
```

## Common Workflows

### Workflow 1: Generate a Greeting

```bash
greeter hello "John" --style formal --json
```

Output:
```json
{
  "greeting": "Dear Mr. John, I hope this message finds you well.",
  "style": "formal"
}
```

### Workflow 2: Batch Greetings

```bash
greeter batch --names "Alice,Bob,Charlie" --style casual --json
```

## Command Reference

| Command | Purpose |
|---------|---------|
| `greeter hello <name>` | Generate single greeting |
| `greeter batch` | Generate multiple greetings |

## Tips

1. Use `--style formal` for business contexts
2. Use `--style funny` for casual contexts
```

### 5.2 InkOS Skill 结构分析

InkOS 的 SKILL.md 是一个优秀的实战示例，包含以下特点：

| 特点 | 示例 |
|------|------|
| 详细的 description | 列出了所有功能点和支持的题材 |
| 清晰的 When to Use | 10 个具体使用场景 |
| 完整的 Setup 流程 | 初始化、配置、状态检查 |
| 11 个工作流 | 覆盖所有常见任务 |
| 命令参考表 | 快速查找 |
| 关键概念解释 | Truth Files、Two-Phase Writer 等 |
| 错误处理指南 | 常见问题及解决方案 |
| 最佳实践 | 9 条实用建议 |

---

## 六、发布与分发

### 6.1 通过 npm 发布

确保 `skills/SKILL.md` 包含在 npm 包中：

```json
// package.json
{
  "files": [
    "dist",
    "skills"
  ]
}
```

### 6.2 发布到 ClawHub

```bash
# 安装 clawhub CLI
npm install -g clawhub

# 登录
clawhub login

# 发布
clawhub publish
```

### 6.3 本地使用

用户克隆项目或通过 npm 安装后，`skills/SKILL.md` 会自动被 AI Agent 发现。

---

## 七、常见问题

### Q1: Skill 文件放在哪里？

**A**: 放在项目根目录的 `skills/SKILL.md`。这是约定俗成的位置。

### Q2: AI 如何发现我的 Skill？

**A**:
1. 用户通过 `clawhub install` 安装
2. 用户 `npm install` 你的包，Skill 随包分发
3. 用户克隆你的项目，Skill 在 `skills/` 目录

### Q3: 需要修改代码才能支持 Skill 吗？

**A**: 不需要修改业务代码。只需：
1. 确保命令支持 `--json` 输出
2. 编写 `skills/SKILL.md` 描述文件

### Q4: 如何测试 Skill 是否有效？

**A**:
1. 在 Claude Code 中使用你的工具
2. 观察 AI 是否能正确理解和调用命令
3. 根据 AI 的行为调整 SKILL.md 内容

### Q5: description 应该写多长？

**A**: 尽可能详细，但保持在一段话内。列出所有功能点和使用场景。AI 依赖这段描述来判断是否应该使用你的工具。

### Q6: 工作流应该写多少个？

**A**: 覆盖所有常见使用场景。InkOS 有 11 个工作流，覆盖了从创建书籍到导出的完整生命周期。

---

## 附录：Skill 检查清单

在发布 Skill 前，检查以下项目：

### Frontmatter

- [ ] `name` 唯一且有意义
- [ ] `description` 详细列出所有功能和场景
- [ ] `version` 遵循语义化版本
- [ ] `requires.bins` 列出所有依赖的可执行文件
- [ ] `install` 提供安装方式

### 正文

- [ ] 有 "When to Use" 部分，列出具体场景
- [ ] 有 "Initial Setup" 部分，说明如何配置
- [ ] 有多个 "Workflow" 覆盖常见任务
- [ ] 每个命令都有 `--json` 示例
- [ ] 有 "Command Reference" 快速参考表
- [ ] 有 "Error Handling" 部分
- [ ] 有 "Tips" 最佳实践建议

### CLI 工具

- [ ] 所有命令支持 `--json` 输出
- [ ] 错误信息清晰且可被解析
- [ ] 命令是原子化的，可独立运行
- [ ] 有 `doctor` 或 `status` 命令检查配置

---

## 相关资源

- [InkOS SKILL.md 示例](../skills/SKILL.md)
- [Radar Agent 分析文档](./radar-agent-analysis.md)
- [OpenClaw 官网](https://clawhub.ai)
- [Claude Code 文档](https://docs.anthropic.com/claude-code)