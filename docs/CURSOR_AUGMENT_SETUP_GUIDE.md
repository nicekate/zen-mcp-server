# Zen MCP 服务器在 Cursor 编辑器和 Augment 中的配置和使用指南

## 📋 目录

1. [项目概述](#1-项目概述)
2. [在 Cursor 编辑器中的配置](#2-在-cursor-编辑器中的配置)
3. [在 Augment 中的配置](#3-在-augment-中的配置)
4. [基本使用方法](#4-基本使用方法)
5. [高级功能和工作流](#5-高级功能和工作流)
6. [故障排除](#6-故障排除)

---

## 1. 项目概述

### 1.1 什么是 Zen MCP 服务器

Zen MCP 服务器是一个基于 Model Context Protocol (MCP) 的多模型AI协作服务器，核心理念是 **"One Context. Many Minds"** - 一个上下文，多个AI大脑协作。

**主要特性：**
- 🤖 **多模型协作**：Claude + Gemini + O3 + GROK + OpenRouter + Ollama 等
- 🔄 **AI对话串联**：支持跨工具、跨模型的连续对话
- 🛠️ **丰富的工具集**：代码分析、审查、调试、重构等10+专业工具
- 🧠 **智能模型选择**：自动为不同任务选择最适合的AI模型
- 💾 **上下文复活**：即使Claude上下文重置，对话仍可继续

### 1.2 支持的AI模型

| 模型 | 上下文窗口 | 特点 | 最佳用途 |
|------|------------|------|----------|
| **Gemini Pro** | 1M tokens | 深度推理+思考模式 | 复杂架构分析、深度代码审查 |
| **Gemini Flash** | 1M tokens | 超快响应 | 快速分析、格式检查 |
| **OpenAI O3** | 200K tokens | 强逻辑推理 | 逻辑问题、代码生成 |
| **OpenAI O3-mini** | 200K tokens | 平衡性能/速度 | 中等复杂度任务 |
| **GROK-3** | 131K tokens | 高级推理 | 复杂分析 |
| **本地模型** | 可变 | 隐私保护 | 敏感代码、成本控制 |

---

## 2. 在 Cursor 编辑器中的配置

### 2.1 前置要求

- **Docker Desktop** 已安装并运行
- **Git** 已安装
- **至少一个AI模型的API密钥**（Gemini、OpenAI、OpenRouter等）

### 2.2 快速安装步骤

#### 步骤1：克隆项目

```bash
# 克隆到您喜欢的位置
git clone https://github.com/BeehiveInnovations/zen-mcp-server.git
cd zen-mcp-server
```

#### 步骤2：一键设置

```bash
# 一键设置（包含Redis用于AI对话）
./run-server.sh
```

这个脚本会：
- 构建Docker镜像
- 创建 `.env` 文件
- 启动Redis服务
- 启动MCP服务器
- 自动添加到Claude Code CLI

#### 步骤3：配置API密钥

```bash
# 编辑 .env 文件添加您的API密钥
nano .env
```

```bash
# 至少需要配置一个API密钥
GEMINI_API_KEY=your-gemini-api-key-here    # Google Gemini模型
OPENAI_API_KEY=your-openai-api-key-here    # OpenAI O3模型
OPENROUTER_API_KEY=your-openrouter-key     # OpenRouter多模型访问

# 本地模型配置（如Ollama）
CUSTOM_API_URL=http://host.docker.internal:11434/v1  # 注意：使用host.docker.internal而非localhost
CUSTOM_API_KEY=                                      # Ollama无需密钥
CUSTOM_MODEL_NAME=llama3.2                          # 默认模型

# 工作空间配置（自动配置）
WORKSPACE_ROOT=/Users/your-username
```

#### 步骤4：重启服务器

```bash
# 修改.env后必须重启
./run-server.sh
```

### 2.3 在 Cursor 中使用

#### 方法1：通过Claude Code CLI

```bash
# 在项目目录中启动Claude会话
claude
```

#### 方法2：通过Claude Desktop配置

如果您使用Claude Desktop，需要配置 `claude_desktop_config.json`：

1. **打开Claude Desktop设置**
   - 打开Claude Desktop
   - 进入 **Settings** → **Developer** → **Edit Config**

2. **添加配置**

```json
{
  "mcpServers": {
    "zen": {
      "command": "docker",
      "args": [
        "exec",
        "-i",
        "zen-mcp-server",
        "python",
        "server.py"
      ]
    }
  }
}
```

3. **重启Claude Desktop**

---

## 3. 在 Augment 中的配置

### 3.1 Augment环境配置

由于Augment已经内置了强大的代码上下文引擎，Zen MCP服务器可以作为补充工具来提供多模型AI协作能力。

#### 配置步骤：

1. **确保Docker环境**
   ```bash
   # 检查Docker是否运行
   docker ps
   
   # 应该看到这些容器：
   # - zen-mcp-server
   # - zen-mcp-redis
   ```

2. **验证服务器状态**
   ```bash
   # 检查服务器日志
   docker exec zen-mcp-server tail -n 100 /tmp/mcp_server.log
   ```

3. **在Augment中调用**
   
   由于Augment有自己的工具调用机制，您可以通过以下方式集成：

   ```python
   # 示例：在Augment环境中调用Zen MCP工具
   import subprocess
   import json
   
   def call_zen_mcp_tool(tool_name, arguments):
       """调用Zen MCP工具的辅助函数"""
       cmd = [
           "docker", "exec", "-i", "zen-mcp-server", 
           "python", "-c", 
           f"""
   import asyncio
   from server import handle_call_tool
   result = asyncio.run(handle_call_tool('{tool_name}', {json.dumps(arguments)}))
   print(result[0].text if result else 'No result')
   """
       ]
       result = subprocess.run(cmd, capture_output=True, text=True)
       return result.stdout.strip()
   ```

---

## 4. 基本使用方法

### 4.1 可用工具概览

| 工具 | 功能 | 最佳使用场景 |
|------|------|-------------|
| `chat` | 协作思考 | 头脑风暴、获取第二意见 |
| `thinkdeep` | 深度推理 | 复杂架构决策、边缘案例分析 |
| `codereview` | 代码审查 | 安全漏洞、性能问题检测 |
| `precommit` | 提交前验证 | Git变更验证、回归检测 |
| `debug` | 调试助手 | 根因分析、错误追踪 |
| `analyze` | 文件分析 | 代码理解、架构分析 |
| `refactor` | 代码重构 | 代码分解、现代化改造 |
| `tracer` | 调用流分析 | 依赖追踪、执行流程图 |
| `testgen` | 测试生成 | 全面测试套件、边缘案例 |

### 4.2 基本使用示例

#### 自动模型选择（推荐）

```
"用zen分析这个认证模块的安全性"
→ Claude自动选择Gemini Pro进行深度安全分析

"使用zen快速检查代码格式"  
→ Claude自动选择Flash进行快速格式检查

"让zen调试这个测试失败的问题"
→ Claude自动选择O3进行逻辑调试
```

#### 指定特定模型

```
"使用flash快速分析main.py的结构"
"让pro深度思考这个架构设计"
"用o3调试这个逻辑错误"
"使用local-llama进行本地代码分析"
```

### 4.3 多模型协作示例

```
"首先用pro分析认证系统的架构，然后让o3验证逻辑，最后用flash检查代码风格"
```

这会触发：
1. Gemini Pro 进行深度架构分析
2. O3 验证逻辑正确性  
3. Flash 进行快速风格检查
4. 所有分析结果在同一对话线程中保持上下文

---

## 5. 高级功能和工作流

### 5.1 AI对话串联

**跨工具对话继续：**

```
1. Claude: "分析 /src/auth.py 的安全问题"
   → 自动模式：Claude选择Gemini Pro进行深度安全分析
   → Pro分析并发现漏洞，提供continuation_id

2. Claude: "彻底审查认证逻辑"
   → 使用相同continuation_id，Claude选择O3进行逻辑分析
   → O3看到之前Pro的分析，提供逻辑聚焦的审查

3. Claude: "调试认证测试失败"
   → 相同continuation_id，Claude继续使用O3进行调试
   → O3提供有针对性的调试，包含所有之前分析的上下文
```

### 5.2 思考模式配置

对于Gemini模型，可以控制思考深度：

```bash
# 在.env中配置默认思考模式
DEFAULT_THINKING_MODE_THINKDEEP=high  # minimal/low/medium/high/max
```

**使用示例：**
```
"使用max思考模式让pro深度分析这个复杂的并发问题"
"用minimal模式快速检查语法"
```

### 5.3 大型提示处理

服务器自动处理超过MCP 25K token限制的大型提示：

1. 检测大型提示（默认>50K字符）
2. 要求Claude保存为文件
3. 直接读取文件内容到模型的大上下文窗口
4. 保留完整MCP容量用于响应

### 5.4 上下文复活功能

**革命性特性：** 即使Claude的上下文重置，对话仍可继续！

```
# Claude上下文重置后
"继续与o3讨论之前的架构方案"
→ O3从Redis中恢复完整对话历史
→ 向Claude提供上下文摘要，重新激活理解
```

---

## 6. 故障排除

### 6.1 常见问题

#### 问题1：容器无法启动

```bash
# 检查Docker状态
docker ps

# 重启容器
docker stop zen-mcp-server zen-mcp-redis
./run-server.sh

# 查看容器日志
docker logs zen-mcp-server
```

#### 问题2：API密钥问题

```bash
# 检查环境变量
docker exec zen-mcp-server env | grep API

# 重新配置.env后重启
./run-server.sh
```

#### 问题3：工具调用失败

```bash
# 查看详细日志
docker exec zen-mcp-server tail -f /tmp/mcp_server.log

# 查看工具活动日志
docker exec zen-mcp-server tail -f /tmp/mcp_activity.log
```

### 6.2 性能优化

#### 模型选择优化

```bash
# 配置自动模式以获得最佳性能
DEFAULT_MODEL=auto

# 为不同工具配置默认思考模式
DEFAULT_THINKING_MODE_THINKDEEP=medium  # 平衡性能和质量
```

#### 本地模型配置

```bash
# 使用Ollama进行成本控制
CUSTOM_API_URL=http://host.docker.internal:11434/v1
CUSTOM_MODEL_NAME=llama3.2
```

### 6.3 监控和日志

```bash
# 实时监控工具活动
python log_monitor.py

# 查看Redis对话状态
docker exec zen-mcp-redis redis-cli keys "*conversation*"

# 检查服务器健康状态
docker exec zen-mcp-server python -c "import server; print('Server OK')"
```

---

## 🎯 快速开始检查清单

- [ ] Docker Desktop已安装并运行
- [ ] 已克隆项目：`git clone https://github.com/BeehiveInnovations/zen-mcp-server.git`
- [ ] 已运行设置脚本：`./run-server.sh`
- [ ] 已配置至少一个API密钥在`.env`文件中
- [ ] 已重启服务器：`./run-server.sh`
- [ ] 在Cursor中可以使用`claude`命令或已配置Claude Desktop
- [ ] 测试基本功能：`"用zen分析这个文件"`

---

## 📚 更多资源

- **[完整技术文档](TECHNICAL_DOCUMENTATION.md)** - 深入的技术细节
- **[高级使用指南](advanced-usage.md)** - 高级配置和工作流
- **[故障排除指南](troubleshooting.md)** - 详细的问题解决方案
- **[自定义模型配置](custom_models.md)** - 添加新的AI模型
- **[工具开发指南](adding_tools.md)** - 创建自定义工具

---

## 🚀 总结

通过这个详细的配置指南，您应该能够在Cursor编辑器和Augment中成功配置和使用Zen MCP服务器，享受多模型AI协作带来的强大开发体验！

**核心优势：**
- 🎯 **一键安装**：Docker化部署，5分钟即可开始使用
- 🧠 **智能协作**：多个AI模型协同工作，各展所长
- 🔄 **无缝集成**：与现有开发工具完美配合
- 💾 **持久记忆**：对话上下文跨会话保持
- 🛠️ **专业工具**：涵盖开发全流程的专业AI工具

开始您的多模型AI协作开发之旅吧！
