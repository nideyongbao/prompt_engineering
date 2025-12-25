# Fabric MCP 集成指南

> 将 Daniel Miessler 的 Fabric AI 框架作为 MCP 工具集成到 Antigravity 中

## 📋 目录

1. [概述](#概述)
2. [架构设计](#架构设计)
3. [前置条件](#前置条件)
4. [安装步骤](#安装步骤)
5. [配置 MCP](#配置-mcp)
6. [使用方式](#使用方式)
7. [可用工具列表](#可用工具列表)
8. [高级用法](#高级用法)
9. [故障排除](#故障排除)
10. [MCP 综合使用指南](#mcp-综合使用指南) ⭐ NEW

---

## 概述

### 什么是 Fabric？

[Fabric](https://github.com/danielmiessler/fabric) 是一个开源 AI 框架，提供 **140+ 预构建的 AI Prompt 模式（Patterns）**，用于解决各种任务：

- 📝 文本摘要 (`summarize`)
- 🧠 智慧提取 (`extract_wisdom`)
- 🔍 声明分析 (`analyze_claims`)
- 💻 代码分析 (`analyze_code`)
- ✍️ 写作改进 (`improve_writing`)

### 为什么集成到 Antigravity？

通过 MCP (Model Context Protocol) 集成，可以：

| 特性 | 说明 |
|------|------|
| **工具化调用** | 像调用其他 MCP 工具一样使用 Fabric Patterns |
| **Prompt 复用** | 获取优化后的 Prompt，而非直接调用 LLM |
| **Agent 接管** | 后续的分析、反思、LLM 调用由 Antigravity 控制 |
| **自动匹配** | Agent 根据用户意图自动选择最佳 Pattern |

---

## 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Antigravity Agent                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户请求 ──→ 意图识别 ──→ MCP工具调用 ──→ 获取Pattern               │
│                              │                  │                   │
│                              ↓                  ↓                   │
│                    ┌─────────────────┐    优化后的Prompt            │
│                    │ fabric-mcp-server│         │                   │
│                    └─────────────────┘          ↓                   │
│                                           Antigravity增强层          │
│                                                 │                   │
│                                                 ↓                   │
│                                           调用LLM/输出结果           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 前置条件

- ✅ Windows 10/11 或 macOS/Linux
- ✅ Node.js 18+ 已安装
- ✅ Git 已安装
- ✅ Antigravity IDE 已安装并配置

---

## 安装步骤

### 步骤 1：安装 Fabric

#### Windows (PowerShell)

```powershell
# 一键安装
iwr -useb https://raw.githubusercontent.com/danielmiessler/fabric/main/scripts/installer/install.ps1 | iex
```

#### macOS/Linux

```bash
# 一键安装
curl -fsSL https://raw.githubusercontent.com/danielmiessler/fabric/main/scripts/installer/install.sh | bash
```

#### 或使用 Docker

```bash
# 使用 Docker Hub 镜像
docker pull kayvan/fabric:latest

# 首次设置
mkdir -p $HOME/.fabric-config
docker run --rm -it -v $HOME/.fabric-config:/root/.config/fabric kayvan/fabric:latest --setup
```

### 步骤 2：配置 Fabric

运行 Fabric 设置向导：

```bash
fabric --setup
```

在设置过程中配置你的 LLM API Key（OpenAI、Anthropic、Gemini 等）。

### 步骤 3：验证 Fabric 安装

```bash
# 检查版本
fabric --version

# 列出可用的 Patterns
fabric --list
```

### 步骤 4：安装 fabric-mcp-server

```bash
# 克隆仓库
git clone https://github.com/adapoet/fabric-mcp-server.git

# 进入目录
cd fabric-mcp-server

# 安装依赖
npm install

# 构建项目
npm run build
```

### 步骤 5：验证 MCP Server 构建

```bash
# 确认 build 目录存在
ls build/index.js
```

---

## 配置 MCP

### Antigravity MCP 配置

找到 Antigravity 的 MCP 配置文件并添加 fabric-mcp-server：

**配置文件路径**（根据实际情况调整）：
- Windows: `C:\Users\<用户名>\.gemini\mcp_config.json`
- macOS: `~/Library/Application Support/Antigravity/mcp_config.json`
- Linux: `~/.config/antigravity/mcp_config.json`

**添加以下配置**：

```json
{
  "mcpServers": {
    "fabric-mcp-server": {
      "command": "node",
      "args": [
        "C:\\path\\to\\fabric-mcp-server\\build\\index.js"
      ],
      "env": {},
      "transportType": "stdio",
      "timeout": 60
    }
  }
}
```

> ⚠️ **重要**：将 `C:\\path\\to\\fabric-mcp-server` 替换为你的实际安装路径

### 路径示例

**Windows**:
```json
"args": ["C:\\Users\\YourName\\fabric-mcp-server\\build\\index.js"]
```

**macOS/Linux**:
```json
"args": ["/home/username/fabric-mcp-server/build/index.js"]
```

### 重启 Antigravity

配置完成后，重启 Antigravity 以加载新的 MCP 服务器。

---

## 使用方式

### 方式 1：直接请求使用 Fabric

```
用户: "使用 fabric 的 summarize 模式帮我总结这篇文章"

Agent 流程:
1. 识别意图 → 需要 summarize Pattern
2. 调用 fabric-mcp-server.summarize
3. 获取优化后的 System Prompt
4. 使用 Prompt 调用 LLM
5. 返回结构化总结
```

### 方式 2：让 Agent 自动选择 Pattern

```
用户: "帮我分析这段代码中的潜在问题"

Agent 流程:
1. 分析需求 → 代码分析任务
2. 自动匹配 → analyze_code Pattern
3. 调用 MCP 工具获取专业 Prompt
4. 增强 Prompt（添加上下文）
5. 调用 LLM 分析
6. 输出结果
```

### 方式 3：组合多个 Patterns

```
用户: "先用 extract_wisdom 提取这个演讲的要点，然后用 summarize 生成简短总结"

Agent 流程:
1. 调用 fabric-mcp-server.extract_wisdom
2. 处理中间结果
3. 调用 fabric-mcp-server.summarize
4. 整合输出
```

---

## 可用工具列表

### 常用 Patterns

| Pattern 名称 | 功能描述 |
|-------------|----------|
| `summarize` | 生成内容摘要 |
| `extract_wisdom` | 提取智慧、洞见、引用和建议 |
| `analyze_claims` | 分析声明的真实性和逻辑 |
| `improve_writing` | 改进写作质量 |
| `explain` | 解释复杂概念 |
| `analyze_code` | 分析代码质量和问题 |
| `create_coding_project` | 创建项目框架 |
| `create_mermaid_visualization` | 生成 Mermaid 图表 |
| `extract_article_wisdom` | 从文章中提取智慧 |
| `analyze_threat_report` | 分析安全威胁报告 |

### 查看完整列表

```bash
# 列出所有可用的 Patterns
fabric --list

# 或查看 Fabric 仓库
# https://github.com/danielmiessler/fabric/tree/main/data/patterns
```

---

## 高级用法

### 自定义 Pattern

1. 创建自定义 Pattern 目录：

```bash
mkdir -p ~/.config/fabric/patterns/my_custom_pattern
```

2. 创建 `system.md` 文件：

```markdown
# IDENTITY and PURPOSE

你是一个专业的 [角色描述]。你的任务是 [任务描述]。

# OUTPUT FORMAT

- 输出格式要求 1
- 输出格式要求 2

# OUTPUT INSTRUCTIONS

- 具体指导 1
- 具体指导 2

# INPUT:

INPUT:
```

3. 新 Pattern 会自动被 fabric-mcp-server 发现

### 结合 Antigravity Workflow

创建 `.agent/workflows/fabric-analyze.md`：

```markdown
---
description: 使用 Fabric 进行内容分析
---

## Fabric 内容分析工作流

1. 确认用户需要分析的内容类型

2. 选择合适的 Fabric Pattern:
   - 总结 → summarize
   - 提取要点 → extract_wisdom
   - 代码分析 → analyze_code
   - 逻辑分析 → analyze_claims

// turbo
3. 调用 fabric-mcp-server 获取 Pattern

4. 使用获取的 Prompt 进行 LLM 调用

5. 整理并输出结果
```

### 环境变量配置

如果需要在 MCP 服务器中传递环境变量：

```json
{
  "mcpServers": {
    "fabric-mcp-server": {
      "command": "node",
      "args": ["path/to/fabric-mcp-server/build/index.js"],
      "env": {
        "FABRIC_CONFIG_PATH": "/custom/path/to/.config/fabric",
        "DEFAULT_MODEL": "gpt-4o"
      }
    }
  }
}
```

---

## 故障排除

### 问题 1：MCP 服务器无法启动

**症状**：Antigravity 提示无法连接到 fabric-mcp-server

**解决方案**：

```bash
# 1. 检查 Node.js 版本
node --version  # 需要 18+

# 2. 重新构建项目
cd fabric-mcp-server
npm install
npm run build

# 3. 手动测试服务器
node build/index.js
```

### 问题 2：找不到 Fabric Patterns

**症状**：MCP 工具调用返回空或错误

**解决方案**：

```bash
# 1. 确认 Fabric 已正确安装
fabric --version

# 2. 检查 Patterns 目录
ls ~/.config/fabric/patterns/

# 3. 重新运行 setup
fabric --setup
```

### 问题 3：路径配置错误

**症状**：JSON 配置解析错误

**解决方案**：

- Windows 路径使用双反斜杠：`C:\\Users\\Name\\path`
- 或使用正斜杠：`C:/Users/Name/path`
- 确保 JSON 格式正确（无尾逗号）

### 问题 4：Docker 模式配置

如果使用 Docker 运行 Fabric：

```json
{
  "mcpServers": {
    "fabric-mcp-server": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "-v", "/path/to/.fabric-config:/root/.config/fabric",
        "fabric-mcp-server:latest"
      ]
    }
  }
}
```

---

## 参考链接

- [Fabric GitHub 仓库](https://github.com/danielmiessler/fabric)
- [fabric-mcp-server 仓库](https://github.com/adapoet/fabric-mcp-server)
- [Fabric REST API 文档](https://github.com/danielmiessler/fabric/blob/main/docs/rest-api.md)
- [Model Context Protocol 规范](https://modelcontextprotocol.io/)
- [Fabric Patterns 完整列表](https://github.com/danielmiessler/fabric/tree/main/data/patterns)

---

## MCP 综合使用指南

> [!NOTE]
> 以下是在 Antigravity 中可用的所有 MCP 服务器及其工具的完整列表和使用方法。

### 当前可用的 MCP 服务器

| MCP 服务器 | 状态 | 用途描述 |
|-----------|------|---------|
| **fabric-mcp-server** | ✅ 正常 | Fabric AI Patterns 工具集 |
| **tavily** | ✅ 正常 | 网络搜索、内容提取、网站爬取 |
| **rednote-MCP** | ⚠️ 需登录 | 小红书笔记搜索和内容获取 |

---

### 1. fabric-mcp-server 工具

#### `recommend_tool`
**功能**：根据任务描述推荐最合适的 Fabric Pattern

**使用示例**：
```
用户: "我需要总结一篇文章"
→ 调用 recommend_tool(input="summarize an article")
→ 返回推荐的 Pattern: summarize
```

**测试验证**（2025-12-25 15:31）：
```json
// 输入
{"input": "test if the fabric mcp server is working correctly"}

// 输出
{"output": "Recommended tool: summarize"}
```
✅ **测试通过**

#### 可用的 Fabric Patterns

所有 Fabric 的 140+ Patterns 都可以作为工具调用：

| Pattern 名称 | 功能描述 |
|-------------|----------|
| `summarize` | 生成内容摘要 |
| `extract_wisdom` | 提取智慧、洞见、引用和建议 |
| `analyze_claims` | 分析声明的真实性和逻辑 |
| `analyze_code` | 分析代码质量和问题 |
| `improve_writing` | 改进写作质量 |
| `explain` | 解释复杂概念 |
| `create_mermaid_visualization` | 生成 Mermaid 图表 |
| `create_coding_project` | 创建项目框架 |
| `extract_article_wisdom` | 从文章中提取智慧 |
| `analyze_threat_report` | 分析安全威胁报告 |

> [!TIP]
> 运行 `fabric --list` 可以查看完整的 Pattern 列表

---

### 2. tavily MCP 工具（网络搜索）

#### `tavily-search`
**功能**：强大的网络搜索工具，返回实时搜索结果

**主要参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `query` | string | 搜索查询（必需） |
| `max_results` | number | 返回结果数量（5-20） |
| `search_depth` | string | 搜索深度："basic" 或 "advanced" |
| `topic` | string | 搜索类别："general" 或 "news" |
| `include_domains` | array | 指定搜索的域名 |
| `exclude_domains` | array | 排除的域名 |
| `time_range` | string | 时间范围："day", "week", "month", "year" |
| `country` | string | 优先显示特定国家的结果 |

**使用示例**：
```
搜索最新的 AI 新闻
→ tavily-search(query="latest AI news", topic="news", max_results=10)
```

#### `tavily-extract`
**功能**：从指定 URL 提取网页内容

**主要参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `urls` | array | 要提取的 URL 列表（必需） |
| `extract_depth` | string | 提取深度："basic" 或 "advanced" |
| `format` | string | 输出格式："markdown" 或 "text" |
| `include_images` | boolean | 是否包含图片 |

**使用示例**：
```
提取 GitHub README 内容
→ tavily-extract(urls=["https://github.com/user/repo"], format="markdown")
```

#### `tavily-crawl`
**功能**：从指定 URL 开始爬取网站

**主要参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `url` | string | 起始 URL（必需） |
| `max_depth` | number | 爬取深度 |
| `max_breadth` | number | 每层最大链接数 |
| `limit` | number | 总处理链接数上限 |
| `instructions` | string | 自然语言爬取指令 |

**使用示例**：
```
爬取文档站点
→ tavily-crawl(url="https://docs.example.com", max_depth=2, limit=20)
```

#### `tavily-map`
**功能**：创建网站 URL 结构地图

**主要参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `url` | string | 起始 URL（必需） |
| `max_depth` | number | 映射深度 |
| `select_paths` | array | 路径过滤正则 |

**使用示例**：
```
映射网站结构
→ tavily-map(url="https://example.com", max_depth=2)
```

---

### 3. rednote-MCP 工具（小红书）

> [!WARNING]
> 使用前需要先调用 `login` 工具进行登录

#### `login`
**功能**：登录小红书账号

**使用方法**：
```
→ 调用 login() 获取登录二维码
→ 使用小红书 APP 扫码登录
```

#### `search_notes`
**功能**：根据关键词搜索小红书笔记

**参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `keywords` | string | 搜索关键词（必需） |
| `limit` | number | 返回结果数量限制 |

**使用示例**：
```
搜索旅游相关笔记
→ search_notes(keywords="日本旅游攻略", limit=10)
```

#### `get_note_content`
**功能**：获取指定笔记的内容

**参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `url` | string | 笔记 URL（必需） |

#### `get_note_comments`
**功能**：获取指定笔记的评论

**参数**：
| 参数 | 类型 | 描述 |
|-----|------|------|
| `url` | string | 笔记 URL（必需） |

---

### MCP 配置文件示例

完整的多 MCP 服务器配置示例：

```json
{
  "mcpServers": {
    "fabric-mcp-server": {
      "command": "node",
      "args": ["C:\\path\\to\\fabric-mcp-server\\build\\index.js"],
      "env": {},
      "transportType": "stdio",
      "timeout": 60
    },
    "tavily": {
      "command": "npx",
      "args": ["-y", "tavily-mcp@latest"],
      "env": {
        "TAVILY_API_KEY": "your-api-key"
      }
    },
    "rednote-MCP": {
      "command": "npx",
      "args": ["-y", "xiaohongshu-mcp@latest"],
      "env": {}
    }
  }
}
```

---

### 使用场景示例

#### 场景 1：研究调研
```
1. 使用 tavily-search 搜索相关资料
2. 使用 tavily-extract 提取关键内容
3. 使用 fabric recommend_tool 获取最佳分析 Pattern
4. 使用对应的 Fabric Pattern 进行内容分析
```

#### 场景 2：内容创作
```
1. 使用 rednote-MCP 搜索热门笔记获取灵感
2. 使用 fabric extract_wisdom 提取要点
3. 使用 fabric improve_writing 优化写作
```

#### 场景 3：代码分析
```
1. 使用 fabric analyze_code 分析代码问题
2. 使用 fabric create_mermaid_visualization 生成架构图
3. 使用 fabric summarize 生成文档总结
```

## 更新日志

| 日期 | 版本 | 更新内容 |
|------|------|----------|
| 2025-12-25 | 1.1 | 新增 MCP 综合使用指南、测试验证、完整工具列表 |
| 2025-12-25 | 1.0 | 初始版本 |

---

> 💡 **提示**：如有问题，可以在 Antigravity 中直接询问如何使用 Fabric 工具，Agent 会自动调用相关 MCP 功能。
