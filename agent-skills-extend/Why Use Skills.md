# 原文：[[reading_material_1_II]]
# Skills架构图
![[agent-skills-architecture.png]]
## Claude 访问 Skill 内容的方式

当一个 **Skill** 被触发时，Claude 会通过 **bash** 命令从文件系统中读取 `SKILL.md`，并将其中的指令载入上下文窗口（context window）。如果这些指令引用了其他文件（例如 `FORMS.md` 或数据库架构图），Claude 也会通过额外的 bash 命令读取这些文件。当指令涉及可执行脚本时，Claude 会通过 bash 运行它们并获取输出结果（**脚本代码本身永远不会进入上下文**）。

---

## 这种架构带来的优势

### 1. 按需文件访问

Claude 仅读取执行特定任务所需的文件。一个 Skill 可以包含数十个参考文件，但如果你的任务只需要“销售架构图”，Claude 就只加载那一个文件。其余文件保留在文件系统中，**消耗零 Token**。

### 2. 高效的脚本执行

当 Claude 运行 `validate_form.py` 时，该脚本的代码内容从不载入上下文窗口。只有脚本的输出（如“验证通过”或特定的错误信息）会消耗 Token。这使得脚本比让 Claude 现场生成等效代码要高效得多。

### 3. 捆绑内容无实际上限

由于文件在被访问前不占用上下文空间，因此 Skill 可以包含详尽的 API 文档、大型数据集、丰富的示例或任何你需要的参考材料。**对于未被使用的捆绑内容，不存在上下文惩罚（Context Penalty）**。

---

> **总结：** 这种基于文件系统的模型是实现“渐进式披露（Progressive Disclosure）”的核心。Claude 就像你在查阅入职指南的特定章节一样导航你的 Skill，精确获取每项任务所需的内容。

# Example：Loading a PDF Processing Skill
### Claude 如何加载并使用 PDF 处理 Skill：

- **启动阶段**：系统提示词（System Prompt）包含：**PDF 处理** —— 从 PDF 文件中提取文本和表格、填写表单、合并文档。
    
- **用户请求**：“提取这份 PDF 中的文本并进行摘要。”
    
- **Claude 调用**：执行 `bash: read pdf-skill/SKILL.md` → 将指令载入上下文。
    
- **Claude 判断**：当前不需要表单填写功能，因此不会读取 `FORMS.md`。
    
- **Claude 执行**：利用 `SKILL.md` 中的指令完成任务。
## Example Workflow
![[agent-skills-context-window.png]]
### 示意图展示了：

- **默认状态**：已预载系统提示词（System Prompt）和 Skill 元数据。
    
- **触发 Skill**：Claude 通过 bash 命令读取 `SKILL.md` 来触发该 Skill。
    
- **按需读取（可选）**：Claude 根据需要读取额外的捆绑文件（如 `FORMS.md`）。
    
- **执行任务**：Claude 继续完成后续任务。 

---

> **核心逻辑：** 这种动态加载机制确保只有与当前任务相关的 Skill 内容才会占用上下文窗口。

# Skill架构
**每个Skill都需要一个包含 YAML 前置元数据的 SKILL.md 文件**
```yaml
---
name: your-skill-name
description: Brief description of what this Skill does and when to use it
---

# Your Skill Name

## Instructions
[Clear, step-by-step guidance for Claude to follow]

## Examples
[Concrete examples of using this Skill]
```
### 必填字段：`name`（名称）与 `description`（描述）

**字段具体要求：**

- **`name`：**
    - 最多 **64** 个字符。
    - 仅限使用**小写字母**、**数字**和**连字符**（`-`）。
    - 不得包含 XML 标签。
    - 不得包含保留字：`anthropic`、`claude`。
- **`description`：**
    - 不能为空。
    - 最多 **1024** 个字符。
    - 不得包含 XML 标签。
    - **描述**应同时包含该 Skill 的**功能**以及 Claude 应该在**何时**使用它。