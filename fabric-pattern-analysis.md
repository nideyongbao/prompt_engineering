---
description: 使用 Fabric Pattern 框架进行文档/代码分析（自动推荐 + 中文输出）
---

# Fabric Pattern 智能分析工作流

> 本工作流结合 Fabric MCP 自动推荐 + Pattern Prompt + Agent 智能的混合分析模式

---

## 工作流步骤

### 步骤 1：调用 Fabric MCP 自动推荐 Pattern

// turbo
调用 MCP 工具自动分析用户意图并推荐最合适的 Pattern：

```
mcp_fabric-mcp-server_recommend_tool(input="<用户的分析需求描述>")
```

返回推荐的 Pattern 名称（如 `summarize`、`analyze_paper`、`review_design` 等）。

**可用的 200+ Patterns 分类参考：**

| 类别 | 常用 Patterns |
|------|--------------|
| 摘要类 | `summarize`, `create_summary`, `summarize_paper` |
| 分析类 | `analyze_paper`, `analyze_claims`, `analyze_code`, `analyze_prose` |
| 评审类 | `review_design`, `review_code`, `improve_writing` |
| 提取类 | `extract_wisdom`, `extract_ideas`, `extract_insights` |
| 创建类 | `create_mermaid_visualization`, `create_report_finding` |

### 步骤 2：读取 Pattern 的完整 Prompt（中文模式）

// turbo
根据推荐的 Pattern 名称，读取对应的系统提示词，并使用 `-g=zh` 参数启用中文输出：

**方式 A：直接读取 Pattern 文件**
```powershell
Get-Content "$env:USERPROFILE\.config\fabric\patterns\<pattern_name>\system.md" -Raw
```

**方式 B：使用 Fabric CLI 获取带中文指令的 Prompt**
```powershell
fabric -p <pattern_name> -g=zh --dry-run
```

Fabric 的 `-g=zh` 参数会自动在 Prompt 末尾添加中文输出指令：
> "All output, titles generated as part of executing the instructions, is written ONLY in the zh language."

Pattern 文件路径：`C:\Users\<用户名>\.config\fabric\patterns\<pattern_name>\system.md`

### 步骤 3：混合分析模式执行

使用 Pattern Prompt + 中文输出指令 + Agent 智能进行分析：

**Part 1 - Fabric Pattern 标准输出（中文）**
- 严格按照 Pattern 定义的输出章节
- 使用中文表述（由 `-g=zh` 保证）
- 保持 Pattern 的格式要求

**Part 2 - Agent 增强建议（中文）**
- 具体的修改建议（带行号、文件名）
- 可操作的下一步行动
- 优先级排序表
- 与当前项目上下文相关的洞察

### 步骤 4：输出最终报告

格式化输出分析结果，包含两部分：

```markdown
# [内容名称] 分析报告

> 使用 Fabric `<pattern_name>` Pattern + Agent 混合分析（中文模式）

---

## 第一部分：Fabric Pattern 标准分析

[按 Pattern 框架输出的中文分析内容]

---

## 第二部分：Agent 增强建议

### 具体可操作项（带行号）
| 行号 | 问题 | 修复方案 |
|------|------|----------|
| ... | ... | ... |

### 优先级排序
| 优先级 | 修改项 | 工作量 |
|--------|--------|--------|
| 🔴 高 | ... | ... |

### 总结
[核心优势和关键改进点]
```

---

## Fabric 中文输出参数说明

Fabric 原生支持 `-g` / `--language` 参数指定输出语言：

```bash
# 中文输出
fabric -p summarize -g=zh < input.txt

# 其他语言示例
fabric -p summarize -g=en < input.txt  # 英文
fabric -p summarize -g=pt-BR < input.txt  # 巴西葡萄牙语
```

**工作原理：** `-g=zh` 会在 System Prompt 末尾自动添加：
> "All output, titles generated as part of executing the instructions, is written ONLY in the zh language."

这是 Fabric 的原生功能，无需翻译 Pattern 文件。

---

## 使用示例

**示例 1：不指定 Pattern（自动推荐 + 中文输出）**
```
/fabric-pattern-analysis 分析这个技术文档
```

Agent 流程：
1. 调用 MCP `recommend_tool` → 返回推荐的 Pattern
2. 读取 Pattern Prompt + 中文输出指令
3. 执行混合分析
4. 输出中文报告

**示例 2：指定 Pattern**
```
/fabric-pattern-analysis summarize 总结这篇文章的要点
```

Agent 流程：
1. 直接使用指定的 `summarize` Pattern
2. 添加 `-g=zh` 中文输出指令
3. 执行后续步骤

---

## 注意事项

1. **中文输出**：使用 Fabric 原生的 `-g=zh` 参数，无需手动翻译 Pattern
2. **Pattern 路径**：确保 Fabric 已安装且 Pattern 文件存在于 `~/.config/fabric/patterns/`
3. **混合模式**：Pattern 提供结构框架，Agent 提供具体可操作的建议
4. **Fabric CLI 路径**：`C:\Users\18810\.local\bin\fabric.exe`
