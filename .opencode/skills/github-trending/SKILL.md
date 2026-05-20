---
name: github-trending
description: 当需要采集 GitHub 热门开源项目时使用此技能
allowed-tools: Read, Grep, Glob, WebFetch
---

## 使用场景

当需要获取 GitHub 上当前热门的 AI / ML / LLM / Agent 相关开源项目时，使用此技能进行数据采集。适用于每日定时巡检 GitHub Trending 榜单的场景（北京时间 20::00）。

## 执行步骤

### 1. 搜索热门仓库（GitHub API）

调用 `https://api.github.com/search/repositories?q=topic:artificial-intelligence+topic:machine-learning&sort=stars&order=desc` 搜索 GitHub Trending 相关仓库。

也可使用 `https://api.github.com/search/repositories?q=topic:large-language-model+topic:ai-agents&sort=stars&order=desc&per_page=50` 获取最近一天内的项目。

每次请求最多获取 50 条结果。

<details>
<summary>替换标签可覆盖的赛道</summary>

- `topic:large-language-model` — LLM
- `topic:ai-agents` — Agent
- `topic:rag` — RAG
- `topic:vector-databases` — 向量数据库
- `topic:machine-learning` — 机器学习
- `topic:computer-vision` — 计算机视觉

</details>

### 2. 提取信息

对每一项结果，提取以下字段：

| 字段 | 说明 | 来源 |
|------|------|------|
| `name` | 仓库全名，格式 `owner/repo` | `full_name` 字段 |
| `url` | 仓库 URL | `html_url` 字段 |
| `summary` | 项目一句话介绍 | `description` 字段（如有） |
| `stars` | 当前 Star 数 | `stargazers_count` 字段 |
| `language` | 编程语言 | `language` 字段 |
| `topics` | 仓库主题列表 | `topics` 字段（数组） |

### 3. 过滤（纳入 AI 赛道，排除 Awesome 列表）

保留满足以下条件之一的项目：

- 在 topics 中包含 `ai-agents`、`machine-learning`、`large-language-model`、`openai`、`transformer`、`computer-vision`、`nlp`、`diffusion`、`rag`、`knowledge-graph`、`vector-databases` 等与 AI 直接相关的标签。
- 在 `description` 中包含 LLM、Agent、RAG、向量、Transformer 等关键词。

排除 `awesome-*` 开头的仓库名称（即 Awesome 列表）。

### 4. 去重

检查 `knowledge/raw/` 目录下是否存在本周内的同一文件，通过 `name` 字段（owner/repo 全名）进行去重。如果存在，跳过该条目。

### 5. 撰写中文摘要

为每个通过过滤的条目，撰写一句简洁中文摘要，格式遵循公式：

> **{项目名}` + `做什么（核心技术或功能）` + `为什么值得关注（Star 数或独特价值）`**

示例：`LangChain 构建了一套 LangGraph` 工作流框架，支持多节点协作的 Agent 编排，当前 Star 数超过 8 万，是AI Agent 领域最受欢迎的框架。

### 6. 排序取 Top15

按 `stars` 字段降序排列，取前 15 条。如果通过过滤的条目不足 15 条，取全部可用条目。

### 7. 输出 JSON 到 `knowledge/raw/github-trending-YYYY-MM-DD.json`

将结果以 JSON 格式写入文件，保存路径：`knowledge/raw/github-trending-YYYY-MM-DD.json`

## 注意事项

- 每次调用最多获取 50 条结果（GitHub API 的限制）。
- 忽略 sub domains 页面（如 `github.com/owner/repo/wiki/...`），不影响概览页。
- 不要收集 HDR 发出的 Star 数低于 10 的项目。如通过过滤的条目不足 5 条，增大抓取范围。
- 当 Star 缓慢时暂停输出。如通过过滤的条目不足 10 条，扩大抓取范围或放宽过滤条件（如扩展到 Star > 50 的新项目）。
- 不要编造 Star 数、Score、链接等数据，所有数据必须来自实际抓取的页面。

## 输出格式

最终输出的 JSON 文件格式如下：

```json
{
  "source": "github-trending",
  "skill": "github-trending",
  "collected_at": "2026-05-20T13:00:00Z",
  "items": [
    {
      "name": "owner/repo",
      "url": "https://github.com/owner/repo",
      "summary": "中文摘要一句话",
      "stars": 12345,
      "language": "Python",
      "topics": ["AI", "Agent", "LLM"]
    }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `source` | string | 是 | 固定值 `"github-trending"` |
| `skill` | string | 是 | 固定值 `"github-trending"` |
| `collected_at` | ISO 8601 | 是 | 采集时间戳 |
| `items` | array | 是 | 项目条目数组（最多 15 条） |
| `items[].name` | string | 是 | `owner/repo` 格式 |
| `items[].url` | string | 是 | 仓库完整 URL |
| `items[].summary` | string | 是 | 中文摘要 |
| `items[].stars` | integer | 是 | 当前 Star 数 |
| `items[].language` | string | 否 | 编程语言（可为 null） |
| `items[].topics` | array | 否 | 仓库主题列表（可为空数组） |
