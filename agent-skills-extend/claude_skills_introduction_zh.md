# Claude Skills 入门介绍

使用 Claude 的 Excel、PowerPoint、PDF Skills，创建文档、分析数据、自动化工作流。

**作者**：Alex Notov（@zealoushacker）  
**发布日期**：2025 年 10 月 14 日

---

学习如何使用 Claude 的 Skills 功能，通过 Excel、PowerPoint 和 PDF 生成来创建专业文档、分析数据并自动化业务工作流。

> **实际效果展示：** 你将在本文中学到的 Skills，正是驱动 Claude 文件创建能力的核心！请查看 **[Claude Creates Files](https://www.anthropic.com/news/create-files)**，了解这些 Skills 如何让 Claude 直接在 Claude.ai 中创建和编辑文档。

---

## 目录

1. [环境配置与安装](#1-环境配置与安装)
2. [理解 Skills](#2-理解-skills)
3. [发现可用 Skills](#3-发现可用-skills)
4. [快速上手：Excel](#4-快速上手excel)
5. [快速上手：PowerPoint](#5-快速上手powerpoint)
6. [快速上手：PDF](#6-快速上手pdf)
7. [常见问题排查](#7-常见问题排查)

---

## 1. 环境配置与安装

### 前置条件

开始之前，请确保你已具备：

- Python 3.8 或更高版本
- 从 [console.anthropic.com](https://console.anthropic.com/) 获取的 Anthropic API 密钥

### 环境初始化（仅首次需要）

**如果你尚未配置环境**，请按以下步骤操作：

#### 第一步：创建虚拟环境

```bash
# 进入 skills 目录
cd /path/to/claude-cookbooks/skills

# 创建虚拟环境
python -m venv venv

# 激活虚拟环境
source venv/bin/activate  # macOS/Linux
# 或
venv\Scripts\activate     # Windows
```

#### 第二步：安装依赖

```bash
# 在虚拟环境激活状态下安装依赖
pip install -r requirements.txt
```

#### 第三步：在 VSCode / Jupyter 中选择内核

**在 VSCode 中：**

1. 打开本 Notebook
2. 点击右上角的内核选择器（例如 "Python 3.11.x"）
3. 选择 "Python Environments..."
4. 选择 `./venv/bin/python` 解释器

**在 Jupyter 中：**

1. 从 Kernel 菜单 → Change Kernel
2. 选择与你的虚拟环境对应的内核

#### 第四步：配置 API 密钥

```bash
# 复制示例文件
cp .env.example .env

# 编辑 .env 文件，填入你的 API 密钥：
# ANTHROPIC_API_KEY=sk-ant-api03-...
```

### 快速安装检查

运行下方代码单元格，验证环境配置是否正确：

**如果看到 ❌ 或 ⚠️ 警告**，请在继续之前完成上述配置步骤。

**如果 anthropic SDK 版本过旧（需要 0.71.0 或更高版本）：**

```bash
pip install anthropic>=0.71.0
```

安装完成后，**重启 Jupyter 内核**以使新版本生效。

---

### API 配置

现在加载 API 密钥并初始化客户端：

**⚠️ 重要提示**：请在 skills 目录中创建 `.env` 文件：

```bash
# 复制示例文件
cp ../.env.example ../.env
```

然后编辑 `../.env`，填入你的 Anthropic API 密钥。

```python
import os
import sys
from pathlib import Path

# 将父目录添加到路径中以便导入
sys.path.insert(0, str(Path.cwd().parent))

from anthropic import Anthropic
from dotenv import load_dotenv

# 导入文件工具函数
from file_utils import (
    download_all_files,
    extract_file_ids,
    get_file_info,
    print_download_summary,
)

# 从父目录加载环境变量
load_dotenv(Path.cwd().parent / ".env")

API_KEY = os.getenv("ANTHROPIC_API_KEY")
MODEL = os.getenv("ANTHROPIC_MODEL", "claude-sonnet-4-6")

if not API_KEY:
    raise ValueError(
        "未找到 ANTHROPIC_API_KEY。请将 ../.env.example 复制为 ../.env 并填入你的 API 密钥。"
    )

# 初始化客户端
# 注意：使用 Skills 时，将在每次请求中单独添加 beta 请求头
client = Anthropic(api_key=API_KEY)

# 创建输出目录（如不存在）
OUTPUT_DIR = Path.cwd().parent / "outputs"
OUTPUT_DIR.mkdir(exist_ok=True)

print("✓ API 密钥已加载")
print(f"✓ 当前使用模型：{MODEL}")
print(f"✓ 输出目录：{OUTPUT_DIR}")
print("\n📝 注意：使用 Skills 时，将在每次请求中单独添加 beta 请求头")
```

### 测试连接

验证 API 连接是否正常工作：

```python
# 简单测试，验证 API 连接
test_response = client.messages.create(
    model=MODEL,
    max_tokens=100,
    messages=[
        {
            "role": "user",
            "content": "如果你能读到这条消息，请说 'Connection successful!'",
        }
    ],
)

print("API 测试响应：")
print(test_response.content[0].text)
print(
    f"\n✓ Token 用量：输入 {test_response.usage.input_tokens}，输出 {test_response.usage.output_tokens}"
)
```

---

## 2. 理解 Skills

### 什么是 Skills？

**Skills** 是由指令、可执行代码和资源组成的结构化包，能为 Claude 赋予特定任务的专业能力。可以将它们理解为"专业知识包"——Claude 可以动态发现并按需加载。

📖 阅读工程博客：[为 Agent 装备实战能力——Skills 详解](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

### Skills 的重要性

在了解了 MCP（模型上下文协议）和工具之后，你可能会好奇 Skills 的独特价值所在：

- **Skills 层级更高**：相比单一工具，Skills 将指令、代码和资源整合为一体
- **Skills 可组合**：多个 Skills 可以无缝协同工作
- **Skills 更高效**：渐进式加载机制让你只为实际使用的部分付费
- **Skills 内含可靠代码**：预置的辅助脚本运行稳定，节省时间、减少错误

### 核心优势

- **专家级表现**：无需漫长学习曲线，即可输出专业结果
- **经过验证的辅助脚本**：Skills 内含经过测试的可用代码，Claude 可直接调用
- **沉淀组织知识**：将公司的工作流程和最佳实践封装为可复用资产
- **成本效率高**：渐进式加载机制有效降低 Token 消耗
- **高可靠性**：预先测试的脚本减少错误，结果更加稳定一致
- **节省时间**：Claude 复用现有方案，无需从头生成代码
- **可组合性**：多个 Skills 协同工作，支撑复杂工作流

### 渐进式加载架构

Skills 采用三级加载模型：

![[prog-disc-1.png]]

1. **元数据**（名称：最多 64 字符，描述：最多 1024 字符）：Claude 获取 Skill 的名称和描述
2. **完整指令**（不超过 5k Token）：当 Skill 与当前任务相关时加载
3. **关联文件**：仅在必要时加载的附加资源

![[prog-disc-2.png]]

这种机制在按需提供专业能力的同时，保持了操作的高效性。最初，Claude 仅能看到 `SKILL.md` YAML 前置元数据中的基本信息；只有当某个 Skill 与当前任务相关时，才会加载完整内容，包括辅助脚本和关联资源。

### Skill 类型

| 类型 | 描述 | 示例 |
|------|------|------|
| **Anthropic 官方维护** | 由 Anthropic 预构建并持续维护 | `xlsx`、`pptx`、`pdf`、`docx` |
| **自定义** | 用户针对特定业务工作流自定义 | 品牌规范、财务模型 |

### Skills 概念总览

![[skills-conceptual-diagram.png]]
该图展示了以下内容：

- **Skill 目录结构**：Skills 如何通过 `SKILL.md` 及配套文件进行组织
- **YAML 前置元数据**：Claude 初始获取的元信息
- **渐进式加载**：Skills 的发现与按需加载机制
- **可组合性**：多个 Skills 在单次请求中协同工作

### Skills 与代码执行的协作方式

使用 Skills 时，必须启用**代码执行**工具。典型工作流如下：

```python
# 使用 client.beta.messages.create() 以支持 Skills
response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    container={
        "skills": [
            {"type": "anthropic", "skill_id": "xlsx", "version": "latest"}
        ]
    },
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[{"role": "user", "content": "创建一个 Excel 文件..."}],
    # 使用 betas 参数代替 extra_headers
    betas=["code-execution-2025-08-25", "files-api-2025-04-14", "skills-2025-10-02"]
)
```

**工作流程说明：**

1. Claude 接收到携带 xlsx skill 的请求
2. Claude 通过代码执行创建文件
3. 响应中包含已创建文件的 `file_id`
4. 你通过 **Files API** 下载该文件

**重要说明：Beta API**

- 必须使用 `client.beta.messages.create()`（而非 `client.messages.create()`）
- `container` 参数仅在 Beta API 中可用
- 使用 `betas` 参数启用以下 Beta 功能：
  - `code-execution-2025-08-25`：启用代码执行
  - `files-api-2025-04-14`：下载文件的必要条件
  - `skills-2025-10-02`：启用 Skills 功能

⚠️ **注意**：使用 Skills 时，请求中**必须**包含 `code_execution` 工具。

### Token 用量优化

与在提示词中手写指令相比，Skills 能显著降低 Token 消耗：

| 方案 | Token 成本 | 效果 |
|------|-----------|------|
| 手写指令 | 每次请求 5,000–10,000 Token | 效果不稳定 |
| Skills（仅元数据） | 极低（仅名称/描述） | 专家级水准 |
| Skills（完整加载） | Skill 被使用时约 5,000 Token | 专家级水准 |

**核心优势：** 你可以在提示词中预置多个 Skills，而不会导致上下文膨胀。每个 Skill 在实际被调用之前，仅占用极少的元数据开销（名称 + 描述）。

**示例**：创建一个带格式的 Excel 文件

- 不使用 Skills：需要预先提供约 8,000 Token 的 Excel 操作说明
- 使用 Skills：初始元数据开销极低，仅在调用 Excel Skill 时消耗约 5,000 Token
- **关键洞察**：98% 的节省体现在初始上下文阶段；一旦实际使用某个 Skill，完整指令才会被加载。

**额外收益：**

- Skills 内置的辅助脚本经过可靠性验证，提升了运行稳定性
- Claude 复用成熟代码模式，节省时间
- 输出结果更加一致、更具专业水准

### ⏱️ 预期生成时间

**⚠️ 重要提示**：通过 Skills 生成文档需要代码执行和文件创建，这需要一定时间。请耐心等待单元格执行完成。

**实测生成时间参考：**

- **Excel 文件**：约 2 分钟（含图表和格式设置）
- **PowerPoint 演示文稿**：约 1–2 分钟（简单的 2 页演示，含图表）
- **PDF 文档**：约 40–60 秒（简单文档）

**执行过程说明：**

- 单元格运行时将显示 `[*]` 状态
- 可能会看到 "Executing..." 状态持续 1–2 分钟
- **请勿中断单元格执行**，等待其完全完成

**💡 操作建议：**

1. **从简单示例开始**：先用最小化示例验证环境是否正常
2. **逐步提升复杂度**：确认基础功能正常后，再逐步增加功能
3. **保持耐心**：操作通常需要 40 秒至 2 分钟
4. **说明**：文档内容越复杂，耗时越长——建议示例保持精简

---

## 3. 发现可用 Skills

### 列出所有内置 Skills

下面来查看 Anthropic 官方提供的所有可用 Skills：

```python
# 列出所有可用的 Anthropic Skills
# 注意：Skills API 需要携带 skills beta 请求头
client_with_skills_beta = Anthropic(
    api_key=API_KEY, default_headers={"anthropic-beta": "skills-2025-10-02"}
)

skills_response = client_with_skills_beta.beta.skills.list(source="anthropic")

print("可用的 Anthropic 官方 Skills：")
print("=" * 80)

for skill in skills_response.data:
    print(f"\n📦 Skill ID: {skill.id}")
    print(f"   标题: {skill.display_title}")
    print(f"   最新版本: {skill.latest_version}")
    print(f"   创建时间: {skill.created_at}")

    # 获取版本详情
    try:
        version_info = client_with_skills_beta.beta.skills.versions.retrieve(
            skill_id=skill.id, version=skill.latest_version
        )
        print(f"   名称: {version_info.name}")
        print(f"   描述: {version_info.description}")
    except Exception as e:
        print(f"   （无法获取版本详情：{e}）")

print(f"\n\n✓ 共找到 {len(skills_response.data)} 个 Anthropic 官方 Skill")
```

### 理解 Skill 元数据

每个 Skill 包含以下信息：

- **skill_id**：唯一标识符（如 `xlsx`、`pptx`）
- **version**：版本号或 `"latest"`
- **name**：可读名称
- **description**：Skill 的功能描述
- **directory**：Skill 的目录结构

### 版本管理策略

- 对于 Anthropic 官方 Skills，推荐使用 `"latest"`（Anthropic 会自动更新）
- 生产环境中，可固定使用特定版本以保证稳定性
- 自定义 Skills 使用 Unix 时间戳作为版本号

---

## 4. 快速上手：Excel

### 示例：月度预算表格

我们将演示两个示例——一个简洁版和一个详细版。

#### 简洁示例（1–2 行）

```python
# 简单提示词 —— Skills 处理背后的复杂性
prompt = "创建一个包含营收数据和图表的季度销售报告 Excel 文件"
```

#### 详细示例

如需更精细的控制，可提供具体需求：

- 收入和支出类别
- 合计公式
- 基础格式设置

### 示例：月度预算表格

我们将创建一个包含以下内容的简单预算表格：

- 收入与支出类别
- 合计公式
- 基础格式设置

**⏱️ 注意**：Excel 生成通常需要 **1–2 分钟**（含图表和格式设置）。运行期间单元格将显示 `[*]`，请耐心等待！

```python
# 创建 Excel 预算表格
excel_response = client.beta.messages.create(  # 注意：使用 beta.messages 以支持 Skills
    model=MODEL,
    max_tokens=4096,
    container={"skills": [{"type": "anthropic", "skill_id": "xlsx", "version": "latest"}]},
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[
        {
            "role": "user",
            "content": """创建一个月度预算 Excel 表格，包含以下内容：

收入：
- 工资：$5,000
- 自由职业：$1,200
- 投资收益：$300

支出：
- 房租：$1,500
- 水电费：$200
- 伙食费：$600
- 交通：$300
- 娱乐：$400
- 储蓄：$1,000

要求：
1. 添加公式分别计算收入总计和支出总计
2. 添加净储蓄公式（收入 - 支出）
3. 正确格式化货币值
4. 添加一个简单的柱状图对比收入与支出
5. 使用带标题行的专业格式
""",
        }
    ],
    # 使用 betas 参数启用 Beta 功能
    betas=["code-execution-2025-08-25", "files-api-2025-04-14", "skills-2025-10-02"],
)

print("Excel 响应：")
print("=" * 80)
for content in excel_response.content:
    if content.type == "text":
        print(content.text)
    elif content.type == "tool_use":
        print(f"\n🔧 工具：{content.name}")
        if hasattr(content, "input"):
            print(f"   输入预览：{str(content.input)[:200]}...")

print("\n\n📊 Token 用量：")
print(f"   输入：{excel_response.usage.input_tokens}")
print(f"   输出：{excel_response.usage.output_tokens}")
```

### 下载 Excel 文件

提取 `file_id` 并下载生成的 Excel 文件：

```python
# 从响应中提取文件 ID
file_ids = extract_file_ids(excel_response)

if file_ids:
    print(f"✓ 找到 {len(file_ids)} 个文件\n")

    # 下载所有文件
    results = download_all_files(
        client, excel_response, output_dir=str(OUTPUT_DIR), prefix="budget_"
    )

    # 打印摘要
    print_download_summary(results)

    # 显示文件详情
    for file_id in file_ids:
        info = get_file_info(client, file_id)
        if info:
            print("\n📄 文件详情：")
            print(f"   文件名：{info['filename']}")
            print(f"   大小：{info['size'] / 1024:.1f} KB")
            print(f"   创建时间：{info['created_at']}")
else:
    print("❌ 响应中未找到文件")
    print("\n调试信息：响应内容类型：")
    for i, content in enumerate(excel_response.content):
        print(f"  {i}. {content.type}")
```

**✨ 刚才发生了什么？**

1. Claude 使用 `xlsx` Skill 创建了一个专业的 Excel 文件
2. Skill 处理了所有 Excel 相关的格式设置和公式
3. 文件在 Claude 的代码执行环境中生成
4. 我们从响应中提取了 `file_id`
5. 通过 Files API 下载了文件
6. 文件已保存至 `outputs/budget_*.xlsx`

用 Excel 打开该文件，查看生成结果！

---

## 5. 快速上手：PowerPoint

接下来，使用 `pptx` Skill 创建一份 PowerPoint 演示文稿。

### 示例：营收汇报演示

#### 简洁示例（1 行）

```python
# 最简提示词 —— Skills 负责处理细节
prompt = "创建一份包含 3 页幻灯片的 Q3 业绩执行摘要演示文稿"
```

#### 详细示例

**注意**：为减少生成时间并聚焦于核心功能演示，本示例有意保持简洁（2 页幻灯片，1 张图表）。

### 示例：简单营收演示

```python
# 创建 PowerPoint 演示文稿
pptx_response = client.beta.messages.create(
    model=MODEL,
    max_tokens=4096,
    container={"skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]},
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[
        {
            "role": "user",
            "content": """创建一份简单的 2 页 PowerPoint 演示文稿：

第 1 页：封面
- 标题："Q3 2025 业绩报告"
- 副标题："Acme Corporation"

第 2 页：营收概览
- 标题："季度营收"
- 添加一个简单的柱状图，显示：
  - Q1：$12M
  - Q2：$13M
  - Q3：$14M

使用简洁、专业的格式。
""",
        }
    ],
    betas=["code-execution-2025-08-25", "files-api-2025-04-14", "skills-2025-10-02"],
)

print("PowerPoint 响应：")
print("=" * 80)
for content in pptx_response.content:
    if content.type == "text":
        print(content.text)

print("\n\n📊 Token 用量：")
print(f"   输入：{pptx_response.usage.input_tokens}")
print(f"   输出：{pptx_response.usage.output_tokens}")
```

### 下载 PowerPoint 文件

```python
# 下载 PowerPoint 文件
file_ids = extract_file_ids(pptx_response)

if file_ids:
    results = download_all_files(
        client, pptx_response, output_dir=str(OUTPUT_DIR), prefix="q3_review_"
    )

    print_download_summary(results)

    print("\n✅ 使用 PowerPoint 或 Google Slides 打开演示文稿查看效果！")
else:
    print("❌ 响应中未找到文件")
```

---

## 6. 快速上手：PDF

**⏱️ 注意**：PDF 生成通常需要 **1–2 分钟**（简单文档）。运行期间单元格将显示 `[*]`，请耐心等待！

### 示例：PDF 文档

#### 简洁示例（1 行）

```python
# 快速生成 PDF
prompt = "为 $500 的咨询服务创建一份专业 PDF 发票"
```

#### 详细示例：收据

**注意**：本示例有意保持简洁，以确保格式整洁清晰。

### 示例：简单收据

```python
# 创建 PDF 收据
pdf_response = client.beta.messages.create(
    model=MODEL,
    max_tokens=4096,
    container={"skills": [{"type": "anthropic", "skill_id": "pdf", "version": "latest"}]},
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[
        {
            "role": "user",
            "content": """创建一份简单的 PDF 收据：

收据

Acme Corporation
日期：2025 年 1 月 15 日
收据编号：RCT-2025-001

客户：Jane Smith

商品明细：
- 商品 A：$50.00
- 商品 B：$75.00
- 商品 C：$25.00

小计：$150.00
税款（8%）：$12.00
总计：$162.00

感谢您的惠顾！

请使用简洁清晰的格式，各部分层次分明。
""",
        }
    ],
    betas=["code-execution-2025-08-25", "files-api-2025-04-14", "skills-2025-10-02"],
)

print("PDF 响应：")
print("=" * 80)
for content in pdf_response.content:
    if content.type == "text":
        print(content.text)

print("\n\n📊 Token 用量：")
print(f"   输入：{pdf_response.usage.input_tokens}")
print(f"   输出：{pdf_response.usage.output_tokens}")
```

### 下载并验证 PDF

```python
# 下载 PDF 文件
file_ids = extract_file_ids(pdf_response)

if file_ids:
    results = download_all_files(
        client, pdf_response, output_dir=str(OUTPUT_DIR), prefix="receipt_"
    )

    print_download_summary(results)

    # 验证 PDF 文件完整性
    for result in results:
        if result["success"]:
            file_path = result["output_path"]
            file_size = result["size"]

            # 基本 PDF 格式校验
            with open(file_path, "rb") as f:
                header = f.read(5)
                if header == b"%PDF-":
                    print(f"\n✅ PDF 文件有效：{file_path}")
                    print(f"   文件大小：{file_size / 1024:.1f} KB")
                else:
                    print(f"\n⚠️ 文件可能不是有效的 PDF：{file_path}")
else:
    print("❌ 响应中未找到文件")
```

---

## 7. 常见问题排查

### 问题 1：找不到 API 密钥

**错误信息：**

```
ValueError: ANTHROPIC_API_KEY not found
```

**解决方案：**

1. 确认父目录中存在 `.env` 文件
2. 检查是否已正确设置 `ANTHROPIC_API_KEY=sk-ant-api03-...`
3. 创建或修改 `.env` 文件后，重启 Jupyter 内核

---

### 问题 2：不识别 container 参数

**错误信息：**

```
TypeError: Messages.create() got an unexpected keyword argument 'container'
```

**解决方案：**  
使用 `client.beta.messages.create()` 代替 `client.messages.create()`：

```python
# ✅ 正确 —— 使用 beta.messages
response = client.beta.messages.create(
    model=MODEL,
    container={"skills": [...]},
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[...],
    betas=["code-execution-2025-08-25", "files-api-2025-04-14", "skills-2025-10-02"]
)

# ❌ 错误 —— 普通 messages 不支持 container 参数
response = client.messages.create(
    model=MODEL,
    container={"skills": [...]},  # 报错！
    messages=[...]
)
```

---

### 问题 3：Skills Beta 需要代码执行工具

**错误信息：**

```
BadRequestError: Skills beta requires the code_execution tool to be included in the request.
```

**解决方案：**  
使用 Skills 时，请求中**必须**包含 `code_execution` 工具：

```python
# ✅ 正确
response = client.beta.messages.create(
    model=MODEL,
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[...],
    betas=["...", "skills-2025-10-02"]
)

# ❌ 错误 —— 缺少 code_execution 工具
response = client.beta.messages.create(
    model=MODEL,
    messages=[...],
    betas=["...", "skills-2025-10-02"]
)
```

---

### 问题 4：响应中未找到文件

**错误信息：**

```
❌ No files found in response
```

**解决方案：**

1. 检查请求中是否包含代码执行工具
2. 验证 Skill 是否已加载（查看响应内容）
3. 确认当前任务确实需要创建文件
4. 查看响应文本中是否有错误提示信息

---

### 问题 5：文件下载失败

**错误信息：**

```
Error retrieving file: File not found
```

**解决方案：**

1. Anthropic 服务器上的文件有时效限制，请在创建后立即下载
2. 检查是否正确从响应中提取了 `file_id`
3. 确认 `betas` 列表中包含了 Files API 的 Beta 标识符

---

### Token 优化建议

1. **使用 `"latest"` 版本**：Anthropic 官方 Skills 会自动优化，推荐使用最新版本
2. **批量操作**：尽可能在一次对话中创建多个文件
3. **复用容器**：使用上次响应中的 `container.id`，避免重复加载 Skills
4. **指令清晰**：明确的指令意味着更少的迭代次数

### API 速率限制

遇到速率限制时：

- 对重试请求实施指数退避策略
- 批量处理多文件任务
- 考虑升级 API 套餐以获取更高限额

---

## 下一步

🎉 **恭喜你！** 你已经掌握了 Claude Skills 的基础用法。

### 观看 Skills 实际效果

查看官方公告，了解这些 Skills 如何驱动 Claude 的文件创建功能：

- **[Claude Creates Files](https://www.anthropic.com/news/create-files)**：了解 Skills 如何让 Claude 直接在 Claude.ai 中创建和编辑 Excel、PowerPoint 及 PDF 文件

### 继续深入学习：

- **[Notebook 2：金融应用实战](https://github.com/anthropics/claude-cookbooks/blob/main/skills/notebooks/02_skills_financial_applications.ipynb)**：基于真实金融数据的业务场景案例
- **[Notebook 3：自定义 Skills 开发](https://github.com/anthropics/claude-cookbooks/blob/main/skills/notebooks/03_skills_custom_development.ipynb)**：构建你自己的专属 Skills

### 支持文档：

- 📚 **[用 Skills 教会 Claude 你的工作方式](https://support.claude.com/en/articles/12580051-teach-claude-your-way-of-working-using-skills)**：Skills 使用用户指南
- 🛠️ **[如何通过对话让 Claude 创建 Skill](https://support.claude.com/en/articles/12599426-how-to-create-a-skill-with-claude-through-conversation)**：交互式 Skill 创建指南

### 参考资料：

- [Claude API 文档](https://docs.anthropic.com/en/api/messages)
- [Skills 官方文档](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
- [Skills 最佳实践](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [Files API 文档](https://docs.claude.com/en/api/files-content)
- [Claude 支持中心](https://support.claude.com)

### 动手练习建议：

1. 从简单的单行提示词开始，观察 Skills 的实际效果
2. 修改预算示例，增加更多支出类别
3. 用你自己的数据创建演示文稿
4. 生成一份结合文字和表格的 PDF 报告
5. 在一次请求中同时使用多个 Skills
