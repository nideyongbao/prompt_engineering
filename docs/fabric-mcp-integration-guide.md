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

## 更新日志

| 日期 | 版本 | 更新内容 |
|------|------|----------|
| 2025-12-25 | 1.0 | 初始版本 |

---

> 💡 **提示**：如有问题，可以在 Antigravity 中直接询问如何使用 Fabric 工具，Agent 会自动调用相关 MCP 功能。
