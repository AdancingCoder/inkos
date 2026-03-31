# Radar Agent 深度分析文档

> 本文档详细分析 InkOS 项目中 Radar Agent 的架构设计与实现细节，帮助开发者理解如何开发类似的 Agent。

## 目录

- [一、整体架构](#一整体架构)
- [二、核心组件](#二核心组件)
  - [2.1 BaseAgent 基类](#21-baseagent-基类)
  - [2.2 RadarSource 数据源接口](#22-radarsource-数据源接口)
  - [2.3 RadarAgent 核心实现](#23-radaragent-核心实现)
  - [2.4 CLI 命令层](#24-cli-命令层)
- [三、数据流](#三数据流)
- [四、数据结构定义](#四数据结构定义)
- [五、扩展指南](#五扩展指南)
- [六、设计模式总结](#六设计模式总结)

---

## 一、整体架构

Radar Agent 采用分层架构，从上到下分为 CLI 层、Pipeline 层、Agent 层和数据源层：

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI 层                                   │
│  packages/cli/src/commands/radar.ts                              │
│  命令: inkos radar scan [--json]                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Pipeline 层                                  │
│  packages/core/src/pipeline/runner.ts                            │
│  PipelineRunner.runRadar() - 编排雷达扫描流程                     │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Agent 层                                    │
│  packages/core/src/agents/radar.ts                               │
│  RadarAgent - 核心 Agent，继承 BaseAgent                          │
│  - scan() 方法执行扫描                                            │
│  - 聚合多个 RadarSource 的数据                                    │
│  - 调用 LLM 分析市场趋势                                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│  FanqieRadarSource  │         │  QidianRadarSource  │
│  番茄小说排行榜      │         │  起点中文网排行榜    │
│  - 热门榜            │         │  - 起点热榜          │
│  - 黑马榜            │         │                     │
└─────────────────────┘         └─────────────────────┘
```
完整依赖关系图

radar.ts (CLI 命令层)
│
├── commander (第三方库)
│   └── Command 类
│
├── @actalk/inkos-core (核心包)
│   │
│   ├── pipeline/runner.ts
│   │   ├── agents/radar.ts ⭐ (实际执行扫描)
│   │   │   ├── agents/radar-source.ts (数据源：起点/番茄)
│   │   │   └── agents/base.ts (Agent 基类)
│   │   ├── llm/provider.ts (调用大模型 API)
│   │   └── state/manager.ts (管理真相文件)
│   │
│   └── utils/config-loader.ts
│       └── 读取 inkos.config.json / .env
│
├── ../utils.ts (CLI 工具层)
│   ├── 加载配置 → loadProjectConfig()
│   ├── 构建配置 → buildPipelineConfig()
│   │   ├── createLLMClient()
│   │   ├── createLogger()
│   │   └── createJsonLineSink()
│   └── 日志输出 → log(), logError()
│
└── node:fs/promises, node:path (Node.js 原生)
    └── 保存 JSON 文件到 radar/scan-时间戳.json

关键调用链
1. radar.ts (你看到的文件)
   ↓
2. utils.ts (加载配置)
   ↓
3. runner.ts (初始化 PipelineRunner)
   ↓
4. radar.ts (RadarAgent.scan())
   ↓
5. radar-source.ts (调用 FanqieRadarSource, QidianRadarSource)
   ↓
6. provider.ts (调用 LLM API 分析数据)
   ↓
7. 返回 RadarResult → 保存到文件系统

### 文件结构

```
packages/
├── cli/src/commands/
│   └── radar.ts              # CLI 命令入口
└── core/src/
    ├── agents/
    │   ├── base.ts           # Agent 基类
    │   ├── radar.ts          # Radar Agent 实现
    │   └── radar-source.ts   # 数据源接口与实现
    └── pipeline/
        └── runner.ts         # Pipeline 编排器
```

---

## 二、核心组件

### 2.1 BaseAgent 基类

**文件位置**: `packages/core/src/agents/base.ts`

BaseAgent 是所有 Agent 的抽象基类，提供统一的 LLM 调用能力。

#### AgentContext 接口

```typescript
export interface AgentContext {
  readonly client: LLMClient;           // LLM 客户端实例
  readonly model: string;               // 使用的模型名称
  readonly projectRoot: string;         // 项目根目录
  readonly bookId?: string;             // 当前书籍 ID（可选）
  readonly logger?: Logger;             // 日志器（可选）
  readonly onStreamProgress?: OnStreamProgress;  // 流式输出回调（可选）
}
```

#### BaseAgent 类

```typescript
export abstract class BaseAgent {
  protected readonly ctx: AgentContext;

  constructor(ctx: AgentContext) {
    this.ctx = ctx;
  }

  // 获取日志器
  protected get log() {
    return this.ctx.logger;
  }

  // 普通 LLM 对话
  protected async chat(
    messages: ReadonlyArray<LLMMessage>,
    options?: { readonly temperature?: number; readonly maxTokens?: number },
  ): Promise<LLMResponse> {
    return chatCompletion(this.ctx.client, this.ctx.model, messages, {
      ...options,
      onStreamProgress: this.ctx.onStreamProgress,
    });
  }

  // 带网络搜索的 LLM 对话
  // - OpenAI: 使用原生 web_search_options
  // - 其他 Provider: 通过 Tavily API 搜索，注入结果到 Prompt
  protected async chatWithSearch(
    messages: ReadonlyArray<LLMMessage>,
    options?: { readonly temperature?: number; readonly maxTokens?: number },
  ): Promise<LLMResponse> {
    // OpenAI 原生支持
    if (this.ctx.client.provider === "openai") {
      return chatCompletion(this.ctx.client, this.ctx.model, messages, {
        ...options,
        webSearch: true,
        onStreamProgress: this.ctx.onStreamProgress,
      });
    }

    // 其他 Provider: 手动搜索 + 注入结果
    const lastUserMsg = [...messages].reverse().find((m) => m.role === "user");
    if (!lastUserMsg) {
      return this.chat(messages, options);
    }

    const query = lastUserMsg.content.slice(0, 200);
    const results = await searchWeb(query, 3);

    // 构建搜索上下文
    const searchContext = [
      "## Web Search Results\n",
      ...results.map((r, i) => `${i + 1}. **${r.title}**\n   ${r.url}\n   ${r.snippet}`),
    ].join("\n");

    // 注入到用户消息中
    const augmentedMessages = messages.map((m) =>
      m === lastUserMsg
        ? { ...m, content: `${searchContext}\n\n---\n\n${m.content}` }
        : m,
    );

    return this.chat(augmentedMessages, options);
  }

  // 子类必须实现的抽象属性
  abstract get name(): string;
}
```

**设计亮点**:
- 依赖注入: 通过 `AgentContext` 统一注入所有外部依赖
- 适配器模式: `chatWithSearch` 自动适配不同 Provider 的搜索能力
- 模板方法: 子类只需关注业务逻辑，LLM 调用由基类处理

---

### 2.2 RadarSource 数据源接口

**文件位置**: `packages/core/src/agents/radar-source.ts`

#### 接口定义

```typescript
// 排行榜条目
export interface RankingEntry {
  readonly title: string;     // 书名
  readonly author: string;    // 作者
  readonly category: string;  // 分类
  readonly extra: string;     // 额外信息（如榜单类型）
}

// 平台排行榜
export interface PlatformRankings {
  readonly platform: string;                    // 平台名称
  readonly entries: ReadonlyArray<RankingEntry>; // 排行榜条目
}

// 数据源接口（策略模式）
export interface RadarSource {
  readonly name: string;
  fetch(): Promise<PlatformRankings>;
}
```

#### 内置实现 1: FanqieRadarSource

```typescript
const FANQIE_RANK_TYPES = [
  { sideType: 10, label: "热门榜" },
  { sideType: 13, label: "黑马榜" },
] as const;

export class FanqieRadarSource implements RadarSource {
  readonly name = "fanqie";

  async fetch(): Promise<PlatformRankings> {
    const entries: RankingEntry[] = [];

    for (const { sideType, label } of FANQIE_RANK_TYPES) {
      try {
        // 调用番茄小说 API
        const url = `https://api-lf.fanqiesdk.com/api/novel/channel/homepage/rank/rank_list/v2/?aid=13&limit=15&offset=0&side_type=${sideType}`;
        const res = await globalThis.fetch(url, {
          headers: { "User-Agent": "Mozilla/5.0 (compatible; InkOS/0.1)" },
        });

        if (!res.ok) continue;
        const data = await res.json();
        const list = data.data?.result;

        if (!Array.isArray(list)) continue;

        for (const item of list) {
          entries.push({
            title: String(item.book_name ?? ""),
            author: String(item.author ?? ""),
            category: String(item.category ?? ""),
            extra: `[${label}]`,
          });
        }
      } catch {
        // 网络错误时跳过
      }
    }

    return { platform: "番茄小说", entries };
  }
}
```

#### 内置实现 2: QidianRadarSource

```typescript
export class QidianRadarSource implements RadarSource {
  readonly name = "qidian";

  async fetch(): Promise<PlatformRankings> {
    const entries: RankingEntry[] = [];

    try {
      // 抓取起点排行榜页面
      const url = "https://www.qidian.com/rank/";
      const res = await globalThis.fetch(url, {
        headers: {
          "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...",
        },
      });

      if (!res.ok) return { platform: "起点中文网", entries };
      const html = await res.text();

      // 正则解析书名
      const bookPattern = /<a[^>]*href="\/\/book\.qidian\.com\/info\/(\d+)"[^>]*>([^<]+)<\/a>/g;
      let match: RegExpExecArray | null;
      const seen = new Set<string>();

      while ((match = bookPattern.exec(html)) !== null) {
        const title = match[2].trim();
        if (title && !seen.has(title) && title.length > 1 && title.length < 30) {
          seen.add(title);
          entries.push({
            title,
            author: "",
            category: "",
            extra: "[起点热榜]"
          });
        }
        if (entries.length >= 20) break;
      }
    } catch {
      // 网络错误时跳过
    }

    return { platform: "起点中文网", entries };
  }
}
```

#### 内置实现 3: TextRadarSource

用于注入外部分析文本：

```typescript
export class TextRadarSource implements RadarSource {
  readonly name: string;
  private readonly text: string;

  constructor(text: string, name = "external") {
    this.name = name;
    this.text = text;
  }

  async fetch(): Promise<PlatformRankings> {
    return {
      platform: this.name,
      entries: [{
        title: this.text,
        author: "",
        category: "",
        extra: "[外部分析]"
      }],
    };
  }
}
```

---

### 2.3 RadarAgent 核心实现

**文件位置**: `packages/core/src/agents/radar.ts`

#### 输出数据结构

```typescript
export interface RadarResult {
  readonly recommendations: ReadonlyArray<RadarRecommendation>;
  readonly marketSummary: string;
  readonly timestamp: string;
}

export interface RadarRecommendation {
  readonly platform: Platform;              // 平台名
  readonly genre: Genre;                    // 题材类型
  readonly concept: string;                 // 一句话概念描述
  readonly confidence: number;              // 置信度 0.0-1.0
  readonly reasoning: string;               // 推荐理由
  readonly benchmarkTitles: ReadonlyArray<string>;  // 对标书籍
}
```

#### RadarAgent 类

```typescript
const DEFAULT_SOURCES: ReadonlyArray<RadarSource> = [
  new FanqieRadarSource(),
  new QidianRadarSource(),
];

// 格式化排行榜数据为 Prompt 文本
function formatRankingsForPrompt(rankings: ReadonlyArray<PlatformRankings>): string {
  const sections = rankings
    .filter((r) => r.entries.length > 0)
    .map((r) => {
      const lines = r.entries.map(
        (e) => `- ${e.title}${e.author ? ` (${e.author})` : ""}${e.category ? ` [${e.category}]` : ""} ${e.extra}`,
      );
      return `### ${r.platform}\n${lines.join("\n")}`;
    });

  return sections.length > 0
    ? sections.join("\n\n")
    : "（未能获取到实时排行数据，请基于你的知识分析）";
}

export class RadarAgent extends BaseAgent {
  private readonly sources: ReadonlyArray<RadarSource>;

  constructor(
    ctx: ConstructorParameters<typeof BaseAgent>[0],
    sources?: ReadonlyArray<RadarSource>,
  ) {
    super(ctx);
    this.sources = sources ?? DEFAULT_SOURCES;
  }

  get name(): string {
    return "radar";
  }

  async scan(): Promise<RadarResult> {
    // 1. 并发获取所有数据源的排行榜
    const rankings = await Promise.all(this.sources.map((s) => s.fetch()));
    const rankingsText = formatRankingsForPrompt(rankings);

    // 2. 构建 System Prompt
    const systemPrompt = `你是一个专业的网络小说市场分析师。下面是从各平台实时抓取的排行榜数据，请基于这些真实数据分析市场趋势。

## 实时排行榜数据

${rankingsText}

分析维度：
1. 从排行榜数据中识别当前热门题材和标签
2. 分析哪些类型的作品占据榜单高位
3. 发现市场空白和机会点（榜单上缺少但有潜力的方向）
4. 风险提示（榜单上过度扎堆的题材）

输出格式必须为 JSON：
{
  "recommendations": [
    {
      "platform": "平台名",
      "genre": "题材类型",
      "concept": "一句话概念描述",
      "confidence": 0.0-1.0,
      "reasoning": "推荐理由（引用具体榜单数据）",
      "benchmarkTitles": ["对标书1", "对标书2"]
    }
  ],
  "marketSummary": "整体市场概述（基于真实榜单数据）"
}

推荐数量：3-5个，按 confidence 降序排列。`;

    // 3. 调用 LLM 分析
    const response = await this.chat(
      [
        { role: "system", content: systemPrompt },
        {
          role: "user",
          content: `请基于上面的实时排行榜数据，分析当前网文市场热度，给出开书建议。`,
        },
      ],
      { temperature: 0.6, maxTokens: 4096 },
    );

    // 4. 解析结果
    return this.parseResult(response.content);
  }

  private parseResult(content: string): RadarResult {
    // 从响应中提取 JSON
    const jsonMatch = content.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      throw new Error("Radar output format error: no JSON found");
    }

    try {
      const parsed = JSON.parse(jsonMatch[0]);
      return {
        recommendations: parsed.recommendations ?? [],
        marketSummary: parsed.marketSummary ?? "",
        timestamp: new Date().toISOString(),
      };
    } catch (e) {
      throw new Error(`Radar JSON parse error: ${e}`);
    }
  }
}
```

---

### 2.4 CLI 命令层

**文件位置**: `packages/cli/src/commands/radar.ts`

```typescript
import { Command } from "commander";
import { PipelineRunner } from "@actalk/inkos-core";
import { loadConfig, buildPipelineConfig, findProjectRoot, log, logError } from "../utils.js";
import { writeFile, mkdir } from "node:fs/promises";
import { join } from "node:path";

export const radarCommand = new Command("radar")
  .description("Market intelligence");

radarCommand
  .command("scan")
  .description("Scan market for opportunities")
  .option("--json", "Output JSON")
  .action(async (opts) => {
    try {
      // 1. 加载配置
      const config = await loadConfig();
      const root = findProjectRoot();

      // 2. 创建 Pipeline 并执行扫描
      const pipeline = new PipelineRunner(buildPipelineConfig(config, root));

      if (!opts.json) log("Scanning market...");

      const result = await pipeline.runRadar();

      // 3. 保存结果到文件
      const radarDir = join(root, "radar");
      await mkdir(radarDir, { recursive: true });
      const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
      const filePath = join(radarDir, `scan-${timestamp}.json`);
      await writeFile(filePath, JSON.stringify(result, null, 2), "utf-8");

      // 4. 输出结果
      if (opts.json) {
        // JSON 格式输出（供 AI Agent 解析）
        log(JSON.stringify({ ...result, savedTo: filePath }, null, 2));
      } else {
        // 人类可读格式
        log(`\nMarket Summary:\n${result.marketSummary}\n`);
        log("Recommendations:");

        for (const rec of result.recommendations) {
          log(`  [${(rec.confidence * 100).toFixed(0)}%] ${rec.platform}/${rec.genre}`);
          log(`    Concept: ${rec.concept}`);
          log(`    Reasoning: ${rec.reasoning}`);
          log(`    Benchmarks: ${rec.benchmarkTitles.join(", ")}`);
          log("");
        }

        log(`Radar result saved to radar/scan-${timestamp}.json`);
      }
    } catch (e) {
      if (opts.json) {
        log(JSON.stringify({ error: String(e) }));
      } else {
        logError(`Radar scan failed: ${e}`);
      }
      process.exit(1);
    }
  });
```

---

## 三、数据流

完整的数据流程如下：

```
1. 用户执行命令
   $ inkos radar scan
           │
           ▼
2. CLI 层处理
   ├── 加载项目配置 (loadConfig)
   ├── 创建 PipelineRunner
   └── 调用 pipeline.runRadar()
           │
           ▼
3. Pipeline 层编排
   └── 创建 RadarAgent 并调用 scan()
           │
           ▼
4. Agent 层执行
   ├── 并发调用所有 RadarSource.fetch()
   │   ├── FanqieRadarSource → 番茄 API
   │   └── QidianRadarSource → 起点 HTML
   │           │
   │           ▼
   ├── 聚合排行榜数据
   │           │
   │           ▼
   ├── 格式化为 Prompt 文本
   │           │
   │           ▼
   ├── 调用 LLM 分析 (temperature=0.6)
   │           │
   │           ▼
   └── 解析 JSON 响应
           │
           ▼
5. 返回 RadarResult
   {
     recommendations: [...],
     marketSummary: "...",
     timestamp: "..."
   }
           │
           ▼
6. CLI 层输出
   ├── 保存到 radar/scan-{timestamp}.json
   └── 输出到终端（JSON 或人类可读格式）
```

---

## 四、数据结构定义

### 输入数据

```typescript
// 排行榜条目
interface RankingEntry {
  title: string;     // 书名
  author: string;    // 作者
  category: string;  // 分类
  extra: string;     // 额外信息
}

// 平台排行榜
interface PlatformRankings {
  platform: string;           // 平台名称
  entries: RankingEntry[];    // 排行榜条目
}
```

### 输出数据

```typescript
// 推荐条目
interface RadarRecommendation {
  platform: Platform;         // 推荐平台
  genre: Genre;               // 推荐题材
  concept: string;            // 一句话概念
  confidence: number;         // 置信度 (0.0-1.0)
  reasoning: string;          // 推荐理由
  benchmarkTitles: string[];  // 对标书籍
}

// 扫描结果
interface RadarResult {
  recommendations: RadarRecommendation[];  // 推荐列表
  marketSummary: string;                   // 市场概述
  timestamp: string;                       // 扫描时间 (ISO 8601)
}
```

### CLI 输出示例

**JSON 格式** (`--json`):

```json
{
  "recommendations": [
    {
      "platform": "番茄小说",
      "genre": "xuanhuan",
      "concept": "废柴逆袭流，主角获得神秘传承",
      "confidence": 0.85,
      "reasoning": "热门榜前10有3本类似题材，读者接受度高",
      "benchmarkTitles": ["吞天魔帝", "万古神帝"]
    }
  ],
  "marketSummary": "当前玄幻题材热度最高，都市题材次之...",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "savedTo": "radar/scan-2024-01-15T10-30-00-000Z.json"
}
```

**人类可读格式**:

```
Scanning market...

Market Summary:
当前玄幻题材热度最高，都市题材次之...

Recommendations:
  [85%] 番茄小说/xuanhuan
    Concept: 废柴逆袭流，主角获得神秘传承
    Reasoning: 热门榜前10有3本类似题材，读者接受度高
    Benchmarks: 吞天魔帝, 万古神帝

Radar result saved to radar/scan-2024-01-15T10-30-00-000Z.json
```

---

## 五、扩展指南

### 5.1 添加新的数据源

实现 `RadarSource` 接口即可：

```typescript
// 示例：豆瓣阅读数据源
export class DoubanRadarSource implements RadarSource {
  readonly name = "douban";

  async fetch(): Promise<PlatformRankings> {
    const entries: RankingEntry[] = [];

    try {
      // 抓取豆瓣阅读排行榜
      const url = "https://read.douban.com/charts";
      const res = await fetch(url);
      const html = await res.text();

      // 解析 HTML 提取书籍信息
      // ...

      return { platform: "豆瓣阅读", entries };
    } catch {
      return { platform: "豆瓣阅读", entries: [] };
    }
  }
}
```

### 5.2 注册新数据源

方式一：修改默认数据源列表

```typescript
// radar.ts
const DEFAULT_SOURCES: ReadonlyArray<RadarSource> = [
  new FanqieRadarSource(),
  new QidianRadarSource(),
  new DoubanRadarSource(),  // 新增
];
```

方式二：创建 Agent 时注入

```typescript
const radar = new RadarAgent(ctx, [
  new FanqieRadarSource(),
  new QidianRadarSource(),
  new DoubanRadarSource(),
]);
```

### 5.3 自定义 LLM 分析逻辑

继承 `RadarAgent` 并重写 `scan` 方法：

```typescript
export class CustomRadarAgent extends RadarAgent {
  async scan(): Promise<RadarResult> {
    // 自定义分析逻辑
    const rankings = await Promise.all(this.sources.map((s) => s.fetch()));

    // 使用不同的 Prompt 或分析维度
    const customPrompt = `...`;

    const response = await this.chat([
      { role: "system", content: customPrompt },
      { role: "user", content: "..." },
    ]);

    return this.parseResult(response.content);
  }
}
```

---

## 六、设计模式总结

| 设计模式 | 应用位置 | 作用 |
|----------|----------|------|
| **策略模式** | `RadarSource` 接口 | 可插拔的数据源，支持运行时切换 |
| **模板方法** | `BaseAgent.chat()` | 统一的 LLM 调用逻辑，子类复用 |
| **依赖注入** | `AgentContext` | 解耦 Agent 与外部依赖（LLM、日志等） |
| **门面模式** | `PipelineRunner` | 简化 CLI 层调用，隐藏内部复杂性 |
| **适配器模式** | `chatWithSearch()` | 适配不同 Provider 的搜索能力 |
| **工厂模式** | `DEFAULT_SOURCES` | 默认数据源的创建 |

---

## 附录：相关文件索引

| 文件路径 | 说明 |
|----------|------|
| `packages/core/src/agents/base.ts` | Agent 基类定义 |
| `packages/core/src/agents/radar.ts` | Radar Agent 实现 |
| `packages/core/src/agents/radar-source.ts` | 数据源接口与实现 |
| `packages/core/src/pipeline/runner.ts` | Pipeline 编排器 |
| `packages/cli/src/commands/radar.ts` | CLI 命令入口 |
| `packages/core/src/llm/provider.ts` | LLM 客户端抽象 |
| `packages/core/src/utils/web-search.ts` | 网络搜索工具 |