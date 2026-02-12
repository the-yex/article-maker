
# 📘 AI 公众号写作系统（Skill 化）设计文档

## 1. 项目背景

传统使用单一大模型生成公众号文章存在问题：
- 技术内容容易出错
- 风格不稳定
- 爆款结构不可控
- 无法复用写作经验

设计目标：基于 **多专家 Skill 化流水线**，实现：
> 技术正确性 × 爆款表达力 × 自动化文章生成  

支持两种输入模式：
1. 输入主题生成文章  
2. 输入已有文章链接进行改写与重构  

---

## 2. 系统目标

### 2.1 功能目标
- 支持输入：
  - 写作主题
  - 或已有文章 URL
- 自动生成可发布的 Markdown 文章
- 每个步骤由 Skill 独立完成，可并行或条件执行
- 可快速替换模型或更新 Prompt

### 2.2 非功能目标
- 模块化、可插拔
- 易维护、易升级
- 可扩展新专家 Skill（法律、选题、热点分析等）
- 支持 Web UI 与草稿管理

---

## 3. 输入系统

### 3.1 输入类型
| 类型 | 输入内容 | 描述 |
|------|----------|------|
| Topic Mode | 主题词/关键词 | 系统根据主题生成文章 |
| URL Mode | 文章链接 | 系统抓取文章内容并改写 |

### 3.2 输入处理 Skill
| Skill 名称 | 输入 | 输出 | 功能 |
|------------|------|------|------|
| fetch_article | URL | 原文 HTML / JSON | 抓取网页内容 |
| parse_article | 原文 HTML | 标题、正文、章节结构 | 去除广告、解析正文 |
| read_article | 结构化文本 | 核心要点、逻辑、事实 | 原文理解、摘要结构化信息 |

---

## 4. 核心 Skill 模块

### 4.1 技术专家 Skill
| Skill 名称 | 输入 | 输出 | 功能 |
|------------|------|------|------|
| tech_draft | 核心要点 | 技术初稿 Markdown | 输出技术正确的文章初稿，包含示例、注意事项 |

### 4.2 公众号专家 Skill
| Skill 名称 | 输入 | 输出 | 功能 |
|------------|------|------|------|
| style_refine | 技术初稿 | 爆款风格文章 Markdown | 改写文章风格，优化标题、痛点、案例、总结 |

### 4.3 审核专家 Skill
| Skill 名称 | 输入 | 输出 | 功能 |
|------------|------|------|------|
| review | 爆款稿 | 最终 Markdown | 校验逻辑、删除废话、风险规避 |

---

## 5. Workflow / 工作流设计

### 5.1 Topic 模式
```text
Topic
  ↓
Router (根据主题选择技术领域)
  ↓
tech_draft Skill
  ↓
style_refine Skill
  ↓
review Skill
  ↓
最终 Markdown 输出
```

### 5.2 URL 模式
```text
URL
  ↓
fetch_article Skill
  ↓
parse_article Skill
  ↓
read_article Skill
  ↓
Router
  ↓
tech_draft Skill
  ↓
style_refine Skill
  ↓
review Skill
  ↓
最终 Markdown 输出
```
> Workflow 可根据需要增加并行 Skill 或条件跳转，例如风格优化或事实校验可并行执行。

---

## 6. Task 与数据结构

```json
{
  "task_id": "uuid",
  "mode": "topic | url",
  "input": "...",
  "skills_executed": [
    {"skill": "fetch_article", "status": "success", "output": "..."},
    {"skill": "tech_draft", "status": "success", "output": "..."}
  ],
  "final_output": "Markdown content"
}
```
- 每个 Skill 执行独立记录  
- 支持重试、日志记录和错误处理  

---

## 7. 技术选型建议

| 层级 | 技术 |
|------|------|
| 编排框架 | **Eino（Go）** |
| 模型 | GPT / Claude / 通义 / 可替换 |
| 输出 | Markdown |
| 存储 | 本地或数据库存储草稿 |
| 可视化 | Web UI（阶段 3 可选） |

---

## 8. 项目结构建议

```text
/ai-writer-skill
  /input
    /fetcher
    /parser
  /orchestrator
    /router
    /workflow
  /skills
    /tech_draft
    /style_refine
    /review
    /read_article
  /storage
  main.go
```

---

## 9. 开发阶段规划

| 阶段 | 目标 |
|------|------|
| MVP | Topic 模式 + 3 个核心 Skill 输出 Markdown |
| Stage 2 | URL 模式，抓取和改写文章 |
| Stage 3 | Web UI，任务管理，自动发布，新增 Skill（选题、热点分析） |

---

## 10. 优点总结
- 模块化、可维护  
- 灵活扩展新 Skill  
- 可快速替换模型或 Prompt  
- 易与智能体平台（Claude Code、Manus 等）集成  
- 支持多任务并行与条件调度  

---

## 11. 核心思路
> Skill 化 = 把多专家流水线模块化，每个 Skill 独立、可插拔，由智能体编排调用，完成从输入 → 技术初稿 → 风格优化 → 审核 → 输出 Markdown 的全过程。
