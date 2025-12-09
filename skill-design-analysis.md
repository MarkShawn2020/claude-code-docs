# Claude Code Skill 设计机制深度分析

## 一、核心设计理念

### 1.1 定位：模型自主调用的能力扩展

Skill 的核心设计理念是 **model-invoked**（模型自主调用），与 Slash Commands 的 **user-invoked**（用户显式调用）形成鲜明对比：

```
┌─────────────────────────────────────────────────────────────┐
│                    调用方式对比                              │
├─────────────────────────────────────────────────────────────┤
│  Slash Commands:  用户 → /command → Claude 执行            │
│  Agent Skills:    用户请求 → Claude 判断 → 自动加载 Skill   │
└─────────────────────────────────────────────────────────────┘
```

这种设计反映了一个核心观点：**Claude 应该像专家一样自主识别何时需要特定领域知识**，而非被动等待用户指定工具。

### 1.2 设计哲学：Progressive Disclosure（渐进式披露）

Skill 采用三层加载系统管理上下文：

```
Layer 1: Metadata (name + description)    ~100 words    [始终在上下文]
    ↓ 触发条件匹配
Layer 2: SKILL.md body                    <5k words     [按需加载]
    ↓ Claude 判断需要
Layer 3: Bundled resources                Unlimited*    [按需加载]
```

*脚本可以直接执行而无需读入上下文窗口

这种设计解决了 LLM 的核心限制：**上下文窗口有限**。通过分层加载，避免了将所有可能需要的知识预先塞入 prompt。

---

## 二、架构解析

### 2.1 Skill 目录结构

```
skill-name/
├── SKILL.md                    # [必需] 入口文件
│   ├── YAML frontmatter        # 元数据：name, description, allowed-tools
│   └── Markdown instructions   # 核心指令
└── [可选资源]
    ├── scripts/                # 可执行脚本 (Python/Bash)
    ├── references/             # 参考文档（需时加载）
    └── assets/                 # 输出资源（模板、图片等）
```

### 2.2 发现机制（Discovery）

Skill 从三个位置被发现：

| 位置 | 路径 | 用途 |
|------|------|------|
| Personal | `~/.claude/skills/` | 个人跨项目 |
| Project | `.claude/skills/` | 团队共享（git） |
| Plugin | `~/.claude/plugins/*/skills/` | 插件捆绑 |

**发现算法的核心**是 `description` 字段的语义匹配：

```yaml
# 差的 description（太模糊）
description: Helps with documents

# 好的 description（具体触发词）
description: Extract text and tables from PDF files, fill forms, merge documents.
             Use when working with PDF files or when the user mentions PDFs, forms,
             or document extraction.
```

### 2.3 allowed-tools 权限控制

```yaml
allowed-tools: Read, Grep, Glob
```

这个字段实现了 **最小权限原则**：

- 当 Skill 激活时，Claude 只能使用指定工具
- 未指定则遵循标准权限模型（需用户确认）
- 用于创建"只读"或"受限"能力

---

## 三、与其他扩展机制的关系

### 3.1 完整的扩展体系

```
┌──────────────────────────────────────────────────────────────────┐
│                   Claude Code 扩展体系                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Slash Commands ──────────────────────────────────────────────── │
│  │ 单文件 .md | 用户显式调用 | 简单提示模板                       │
│                                                                  │
│  Agent Skills ───────────────────────────────────────────────────│
│  │ 目录结构 | 模型自主调用 | 复杂能力+资源                        │
│                                                                  │
│  Hooks ──────────────────────────────────────────────────────────│
│  │ 事件驱动 | 自动触发 | 工作流自动化                             │
│                                                                  │
│  MCP Servers ────────────────────────────────────────────────────│
│  │ 外部服务 | 工具集成 | API 接口                                 │
│                                                                  │
│  Plugins ────────────────────────────────────────────────────────│
│  │ 打包分发 | 组合以上所有 | 团队/社区共享                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Skill vs Slash Command 详细对比

| 维度 | Slash Command | Agent Skill |
|------|---------------|-------------|
| 触发方式 | `/command` 显式调用 | 语义匹配自动触发 |
| 文件结构 | 单个 `.md` 文件 | 目录 + `SKILL.md` + 资源 |
| 复杂度 | 简单提示片段 | 完整工作流 |
| 资源支持 | 无 | scripts/, references/, assets/ |
| 权限控制 | 无 | `allowed-tools` 字段 |
| 适用场景 | 重复性简单指令 | 领域专业能力 |

---

## 四、设计模式分析

### 4.1 "Onboarding Guide" 模式

Skill 本质上是给另一个 Claude 实例的"入职指南"：

```markdown
# 设计思想
Skill = 将领域专家的程序性知识编码为可被 AI 消费的格式

# 目标
将 Claude 从通用代理 → 特定领域的专业代理
```

这解释了为什么 SKILL.md 的写作风格强调：
- **祈使句/动词形式**（而非第二人称）
- 包含 Claude 本身不会自然知道的**程序性知识**
- 提供**具体例子和错误恢复策略**

### 4.2 资源分类策略

```
┌─────────────────────────────────────────────────────────────┐
│  scripts/     →  执行层（token 高效，确定性输出）            │
│  references/  →  知识层（按需加载，信息源）                  │
│  assets/      →  输出层（模板/素材，不进入上下文）           │
└─────────────────────────────────────────────────────────────┘
```

**决策树**：
- 代码会被重复编写？→ `scripts/`
- 信息需要被参考？→ `references/`
- 文件用于输出？→ `assets/`

### 4.3 实际案例剖析

以官方 `pdf` skill 为例：

```yaml
---
name: pdf
description: Comprehensive PDF manipulation toolkit for extracting text
             and tables, creating new PDFs, merging/splitting documents,
             and handling forms. When Claude needs to fill in a PDF form
             or programmatically process, generate, or analyze PDF
             documents at scale.
---
```

**设计要点**：
1. description 包含动作词（extracting, creating, merging, filling）
2. 明确触发条件（"When Claude needs to..."）
3. SKILL.md 只包含核心指令，详细内容在 `reference.md` 和 `forms.md`
4. 提供代码模板而非完整实现

---

## 五、系统集成机制

### 5.1 在 System Prompt 中的呈现

Skill 被暴露为一个可调用的工具，Claude 通过 `skill: "skill-name"` 来激活：

```xml
<function>
  "name": "Skill"
  "description": "Execute a skill within the main conversation..."
</function>
```

### 5.2 激活流程

```
1. Claude 分析用户请求
2. 检查 available_skills 列表（在 system prompt 中）
3. 匹配 description 与用户意图
4. 调用 Skill 工具: skill: "pdf"
5. SKILL.md 内容被注入上下文
6. Claude 按照 SKILL.md 指令执行
```

---

## 六、最佳实践与陷阱

### 6.1 Description 写作原则

```yaml
# ✗ 错误
description: For data analysis

# ✓ 正确
description: Analyze sales data in Excel files and CRM exports.
             Use for sales reports, pipeline analysis, and revenue tracking.
```

**关键要素**：
- 具体的动作（analyze, extract, generate）
- 明确的对象（Excel files, CRM exports）
- 触发场景（"Use when...", "Use for..."）

### 6.2 避免的问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Skill 不被触发 | description 太模糊 | 添加具体触发词 |
| 多 Skill 冲突 | description 重叠 | 使用独特的领域词汇 |
| 上下文爆炸 | SKILL.md 太长 | 拆分到 references/ |
| 脚本权限问题 | 未设置执行权限 | `chmod +x scripts/*.py` |

### 6.3 SKILL.md 长度控制

```
推荐: SKILL.md < 5000 words
原则: 核心流程在 SKILL.md，详细参考在 references/
```

---

## 七、与 Plugin 系统的整合

### 7.1 分发机制

```
开发者创建 Skill
    ↓
打包为 Plugin（包含 skills/ 目录）
    ↓
发布到 Marketplace
    ↓
用户安装 Plugin
    ↓
Skills 自动可用
```

### 7.2 Plugin 中的 Skill 结构

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── my-skill/
│       ├── SKILL.md
│       └── scripts/
└── commands/
```

---

## 八、设计局限与改进空间

### 8.1 当前局限

1. **语义匹配不透明**：用户无法知道 Claude 为何选择/未选择某个 Skill
2. **无版本依赖管理**：Skill 之间无法声明依赖关系
3. **无运行时参数**：不像 MCP 工具可以传递参数
4. **调试困难**：需要 `claude --debug` 查看加载错误

### 8.2 潜在改进方向

- **显式触发选项**：允许用户强制使用某 Skill
- **Skill 组合**：声明式的 Skill 编排
- **条件激活**：基于文件类型、项目类型的自动激活
- **监控面板**：可视化 Skill 使用频率和效果

---

## 九、总结

Claude Code 的 Skill 机制体现了一个核心设计理念：

> **将 AI Agent 的能力扩展从"工具调用"提升到"专业知识注入"**

它不仅仅是给 Claude 更多工具，而是给它特定领域的**程序性知识**和**决策框架**。这种设计使得：

1. **用户体验更自然**：无需记忆命令，自然语言即可触发
2. **知识可复用**：团队专业知识被编码和共享
3. **上下文高效**：按需加载，避免 prompt 膨胀
4. **权限可控**：`allowed-tools` 实现最小权限

Skill 系统是 Claude Code 从"AI 助手"向"AI 专家团队"演进的关键基础设施。

---

## 附录：参考资源

- [Agent Skills 官方文档](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Plugins 文档](https://docs.anthropic.com/en/docs/claude-code/plugins)
- [Slash Commands 文档](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Anthropic 工程博客：Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
