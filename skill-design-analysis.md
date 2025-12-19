---
title: Claude Code Skill 设计机制深度分析
---

最近我们对 claude code 的 skills 能力做了一些深度调研，并开发了一个在 claude 里调用 nano-banana-pro 生成图片的 skill（类似工作流），并提供 slash command 封装。

![](http://cdn.cs-magic.cn/picgo/20251210165357950.png?imageslim%7CimageMogr2/format/jpeg/size-limit/1000k!)

![我们的nano-banana-pro生图skill支持图片打开和ascii渲染两种模式](http://cdn.cs-magic.cn/picgo/b1a707f048e6ddd3ef1d0d1e35f0b147.png?imageslim%7CimageMogr2/format/jpeg/size-limit/1000k!)

在这个过程中我们发现，**基于 skill 的单元开发模式（然后对外暴露 skill 接口、command 接口、被 agent 调用等）可能是一种最佳实践**。

基于这些发现，我们很希望通过几篇文章将这个理念进一步推广，以下是本期的第二篇文章：**Claude Code Skill 设计机制深度分析**。在这之前，我们还做过一次技术沙龙，欢迎移步阅读：[十问 Agent Skills：一场围绕 AI 编码新范式的深度研讨](https://mp.weixin.qq.com/s/GItOOuUEZuPD_R-ISdwVaA)。

以及这是我们整理的一些权威内容，希望对你的入门学习有所帮助：

- [Introducing Agent Skills | Claude](https://claude.com/blog/skills)
- [Agent Skills 官方文档](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Plugins 文档](https://docs.anthropic.com/en/docs/claude-code/plugins)
- [Slash Commands 文档](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Anthropic 工程博客：Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [claude-code-docs/docs/skills.md](https://github.com/ericbuess/claude-code-docs/blob/db7f17b264db7be3bf3c469bd8c2acce2fc024a3/docs/skills.md?plain=1)

## 一、核心设计理念

### 1.1 定位：模型自主调用的能力扩展

Skill 的核心设计理念是 **model-invoked**（模型自主调用），与 Slash Commands 的 **user-invoked**（用户显式调用）形成鲜明对比：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380530385-iw9axk.png)

这种设计反映了一个核心观点：**Claude 应该像专家一样自主识别何时需要特定领域知识**，而非被动等待用户指定工具。

### 1.2 设计哲学：Progressive Disclosure（渐进式披露）

Skill 采用三层加载系统管理上下文：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380580465-9it9e7.png)

\*脚本可以直接执行而无需读入上下文窗口

这种设计解决了 LLM 的核心限制：**上下文窗口有限**。通过分层加载，避免了将所有可能需要的知识预先塞入 prompt。

---

## 二、架构解析

### 2.1 Skill 目录结构

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380583424-ynlhxv.png)

### 2.2 发现机制（Discovery）

Skill 从三个位置被发现：

| 位置     | 路径                          | 用途            |
| -------- | ----------------------------- | --------------- |
| Personal | `~/.claude/skills/`           | 个人跨项目      |
| Project  | `.claude/skills/`             | 团队共享（git） |
| Plugin   | `~/.claude/plugins/*/skills/` | 插件捆绑        |

**发现算法的核心**是 `description` 字段的语义匹配：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380585689-jq7zya.png)

### 2.3 allowed-tools 权限控制

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380590003-f3egjs.png)

这个字段实现了 **最小权限原则**：

- 当 Skill 激活时，Claude 只能使用指定工具
- 未指定则遵循标准权限模型（需用户确认）
- 用于创建"只读"或"受限"能力

---

## 三、与其他扩展机制的关系

### 3.1 完整的扩展体系

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380591935-0e0e10.png)

### 3.2 Skill vs Slash Command 详细对比

| 维度     | Slash Command       | Agent Skill                    |
| -------- | ------------------- | ------------------------------ |
| 触发方式 | `/command` 显式调用 | 语义匹配自动触发               |
| 文件结构 | 单个 `.md` 文件     | 目录 + `SKILL.md` + 资源       |
| 复杂度   | 简单提示片段        | 完整工作流                     |
| 资源支持 | 无                  | scripts/, references/, assets/ |
| 权限控制 | 无                  | `allowed-tools` 字段           |
| 适用场景 | 重复性简单指令      | 领域专业能力                   |

---

## 四、设计模式分析

### 4.1 "Onboarding Guide" 模式

Skill 本质上是给另一个 Claude 实例的"入职指南"：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380597324-6pgrnw.png)

这解释了为什么 SKILL.md 的写作风格强调：

- **祈使句/动词形式**（而非第二人称）
- 包含 Claude 本身不会自然知道的**程序性知识**
- 提供**具体例子和错误恢复策略**

### 4.2 资源分类策略

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380598733-fxbrmd.png)

**决策树**：

- 代码会被重复编写？→ `scripts/`
- 信息需要被参考？→ `references/`
- 文件用于输出？→ `assets/`

### 4.3 实际案例剖析

以官方 `pdf` skill 为例：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380600536-77apmz.png)

**设计要点**：

1. description 包含动作词（extracting, creating, merging, filling）
2. 明确触发条件（"When Claude needs to..."）
3. SKILL.md 只包含核心指令，详细内容在 `reference.md` 和 `forms.md`
4. 提供代码模板而非完整实现

---

## 五、系统集成机制

### 5.1 在 System Prompt 中的呈现

Skill 被暴露为一个可调用的工具，Claude 通过 `skill: "skill-name"` 来激活：

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380602316-9vwy7m.png)

### 5.2 激活流程

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380603676-k3iku6.png)

---

## 六、最佳实践与陷阱

### 6.1 Description 写作原则

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380605587-m8mi7m.png)

**关键要素**：

- 具体的动作（analyze, extract, generate）
- 明确的对象（Excel files, CRM exports）
- 触发场景（"Use when...", "Use for..."）

### 6.2 避免的问题

| 问题           | 原因               | 解决方案                |
| -------------- | ------------------ | ----------------------- |
| Skill 不被触发 | description 太模糊 | 添加具体触发词          |
| 多 Skill 冲突  | description 重叠   | 使用独特的领域词汇      |
| 上下文爆炸     | SKILL.md 太长      | 拆分到 references/      |
| 脚本权限问题   | 未设置执行权限     | `chmod +x scripts/*.py` |

### 6.3 SKILL.md 长度控制

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380607778-0hmsbb.png)

---

## 七、与 Plugin 系统的整合

### 7.1 分发机制

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380609013-9iuhrs.png)

### 7.2 Plugin 中的 Skill 结构

![](http://cdn.cs-magic.cn/lovpen/codeblock-1765380610140-vcsek9.png)

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