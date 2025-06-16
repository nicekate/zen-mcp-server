# 架构决策记录 (ADR)

> **记录Zen MCP Server的重要架构决策和设计理念**

## ADR-001: 选择MCP协议作为通信基础

**日期**: 2024-Q4  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
需要选择Claude Code与AI服务器之间的通信协议。

### 决策
采用Anthropic的Model Context Protocol (MCP)作为通信协议。

### 理由
- **官方支持**: Anthropic官方协议，与Claude深度集成
- **标准化**: 提供标准的工具发现和调用机制
- **扩展性**: 支持复杂的工具参数和响应格式
- **未来兼容**: 随Claude发展持续更新

### 后果
- 依赖MCP协议的稳定性和发展
- 需要遵循MCP的设计约束
- 获得与Claude的最佳集成体验

---

## ADR-002: 多AI提供商架构设计

**日期**: 2024-Q4  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
需要支持多个AI提供商(Gemini、OpenAI、X.AI等)，同时保持代码的可维护性。

### 决策
采用提供商抽象层设计，通过统一接口管理不同AI服务。

### 架构设计
```python
# 抽象基类
class BaseModelProvider(ABC):
    @abstractmethod
    def generate_content(self, prompt, model_name, **kwargs) -> ModelResponse
    
    @abstractmethod
    def get_capabilities(self, model_name) -> ModelCapabilities

# 具体实现
class GeminiModelProvider(BaseModelProvider)
class OpenAIModelProvider(BaseModelProvider)
class XAIModelProvider(BaseModelProvider)
```

### 理由
- **可扩展性**: 易于添加新的AI提供商
- **一致性**: 统一的接口简化工具开发
- **灵活性**: 每个提供商可以有特定的优化
- **可测试性**: 便于mock和单元测试

### 后果
- 需要维护多个提供商的适配代码
- 抽象层可能限制某些提供商特有功能
- 增加了系统复杂度但提高了灵活性

---

## ADR-003: Redis作为对话状态存储

**日期**: 2024-Q4  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
需要在多轮对话中保持状态，支持跨工具的上下文延续。

### 决策
使用Redis作为对话状态的持久化存储。

### 理由
- **性能**: 内存数据库，读写速度快
- **过期机制**: 自动清理过期对话，避免内存泄漏
- **数据结构**: 丰富的数据类型支持复杂状态存储
- **可靠性**: 成熟稳定的开源解决方案
- **容器化**: 易于Docker部署和管理

### 实现细节
```python
# 对话数据结构
conversation_data = {
    "conversation_id": "conv_123",
    "turns": [
        {
            "tool": "analyze",
            "model": "gemini-pro",
            "files": ["/path/to/file.py"],
            "response": "分析结果...",
            "timestamp": "2025-01-01T10:00:00Z"
        }
    ],
    "file_references": {
        "/path/to/file.py": {
            "content_hash": "abc123",
            "last_referenced": "2025-01-01T10:00:00Z"
        }
    }
}
```

### 后果
- 增加了Redis依赖
- 需要处理Redis连接失败的情况
- 获得了强大的对话延续能力

---

## ADR-004: 工具基类设计模式

**日期**: 2024-Q4  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
需要为9个不同的工具提供一致的接口和通用功能。

### 决策
采用抽象基类模式，所有工具继承自`BaseTool`。

### 设计模式
```python
class BaseTool(ABC):
    # 抽象方法 - 子类必须实现
    @abstractmethod
    def get_name(self) -> str
    
    @abstractmethod
    def get_description(self) -> str
    
    @abstractmethod
    def prepare_prompt(self, request) -> str
    
    # 通用功能 - 所有工具共享
    def validate_file_paths(self, paths: list[str]) -> None
    def handle_conversation_context(self, continuation_id: str) -> dict
    def format_response(self, content: str) -> list[TextContent]
```

### 理由
- **代码复用**: 通用功能只需实现一次
- **一致性**: 所有工具有相同的生命周期和接口
- **可维护性**: 修改通用逻辑只需改基类
- **扩展性**: 新工具只需实现抽象方法

### 后果
- 基类变更可能影响所有工具
- 需要仔细设计抽象接口
- 获得了高度一致和可维护的工具系统

---

## ADR-005: Docker容器化部署策略

**日期**: 2024-Q4  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
需要简化部署过程，确保环境一致性。

### 决策
采用Docker Compose进行多容器编排部署。

### 容器架构
```yaml
services:
  zen-mcp:          # 主服务容器
    - MCP Server
    - 工具和提供商
    - 业务逻辑
    
  redis:            # 数据存储容器
    - 对话状态存储
    - LRU过期策略
    
  log-monitor:      # 日志监控容器
    - 日志聚合
    - 实时监控
```

### 理由
- **环境一致性**: 开发、测试、生产环境完全一致
- **简化部署**: 一键启动所有服务
- **服务隔离**: 每个服务独立运行，故障隔离
- **资源管理**: 容器级别的资源限制和监控
- **可移植性**: 跨平台部署支持

### 后果
- 增加了Docker的学习成本
- 需要管理容器生命周期
- 获得了简单可靠的部署方案

---

## ADR-006: 智能模型选择机制

**日期**: 2025-Q1  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
不同AI模型有不同的优势，需要为不同任务自动选择最佳模型。

### 决策
实现基于任务特征的智能模型选择算法。

### 选择策略
```python
def select_model(task_type: str, complexity: str, user_preference: str) -> str:
    if user_preference != "auto":
        return user_preference
        
    if task_type == "quick_analysis":
        return "flash"
    elif task_type == "deep_thinking":
        return "pro" if complexity == "high" else "o3-mini"
    elif task_type == "logical_reasoning":
        return "o3"
    elif task_type == "local_privacy":
        return "custom"
    else:
        return "flash"  # 默认快速模型
```

### 理由
- **性能优化**: 为任务选择最适合的模型
- **成本控制**: 避免过度使用昂贵模型
- **用户体验**: 自动化选择减少用户决策负担
- **灵活性**: 用户仍可手动指定模型

### 后果
- 需要维护模型能力映射表
- 选择算法需要持续优化
- 获得了智能化的模型使用体验

---

## ADR-007: 文件安全和访问控制

**日期**: 2025-Q1  
**状态**: 已采用  
**决策者**: 项目团队

### 背景
AI工具需要访问用户文件，但必须确保安全性。

### 决策
实现多层文件安全控制机制。

### 安全措施
1. **路径验证**: 只允许绝对路径，防止路径遍历
2. **沙盒限制**: 容器内只读挂载工作空间
3. **范围检查**: 限制访问用户主目录范围内
4. **类型过滤**: 只处理已知安全的文件类型

```python
def validate_file_path(path: str) -> bool:
    # 1. 必须是绝对路径
    if not os.path.isabs(path):
        return False
        
    # 2. 路径规范化，防止 ../ 攻击
    normalized = os.path.normpath(path)
    
    # 3. 必须在工作空间范围内
    workspace_root = os.getenv("WORKSPACE_ROOT")
    if not normalized.startswith(workspace_root):
        return False
        
    return True
```

### 理由
- **安全第一**: 防止恶意文件访问
- **用户信任**: 确保用户数据安全
- **合规要求**: 满足企业安全标准
- **风险控制**: 最小化潜在安全风险

### 后果
- 限制了某些高级文件操作
- 增加了路径验证的开销
- 获得了可信赖的文件访问机制

---

## 设计原则总结

### 1. 安全优先
- 所有文件访问都经过安全验证
- 容器化隔离和只读挂载
- API密钥环境变量隔离

### 2. 可扩展性
- 插件化的工具架构
- 抽象的提供商接口
- 模块化的系统设计

### 3. 用户体验
- 自动化的模型选择
- 智能的对话延续
- 简化的部署流程

### 4. 性能优化
- Redis缓存和LRU策略
- 异步处理和并发支持
- 智能的文件处理

### 5. 可维护性
- 清晰的代码结构
- 完善的测试覆盖
- 详细的文档说明

---

*这些架构决策指导了Zen MCP Server的设计和实现，确保系统的安全性、可扩展性和用户体验。*
