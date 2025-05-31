# Agent Zero 项目核心架构与开发指南

## 🧠 系统架构概述

Agent Zero 是一个基于提示（prompt）驱动的智能代理框架，采用模块化和可扩展的设计。其核心运行依赖于 Docker 容器环境，确保跨平台一致性与安全性。

### 架构层级：

1. **用户层**：通过 Web UI 或 CLI 与代理交互。
2. **代理层**：支持任务分解与多级协作。
3. **工具层**：提供多种功能模块（如浏览器操作、代码执行等）。
4. **记忆层**：支持长期记忆存储与检索。
5. **知识库**：支持本地文件导入以增强回答能力。
6. **扩展机制**：允许开发者添加新功能。
7. **仪器模块**：自定义脚本用于执行特定任务。

### 目录结构概览:
| 目录 | 描述 |
|------|------|
| `/docker` | Docker相关配置文件 |
| `/docs` | 项目文档 |
| `/instruments` | 自定义脚本和工具 |
| `/knowledge` | 知识库相关文件 |
| `/lib` | 浏览器相关脚本 |
| `/prompts` | 系统提示模板 |
| `/python` | Python 脚本，包含 API、扩展和工具 |
| `/webui` | Web 用户界面相关文件 |
| `/work_dir` | 工作目录用于临时文件处理 |

## 🔧 核心组件说明

### 1. Agent（代理）
- 负责接收指令、推理决策、调用工具完成任务。

### 2. Tools（工具）
- 常用工具包括浏览器操作、代码执行、搜索引擎集成等。

### 3. History & Context（上下文与历史）
- 分段压缩策略优化长对话上下文窗口占用。

### 4. Prompts（提示）
- 控制代理行为逻辑，可定制化替换默认提示文件。

### 5. Knowledge（知识）
- 支持多种格式外部知识文件导入。

### 6. Extensions（扩展）
- 非侵入式增强代理功能。

### 7. Instruments（仪器）
- 自定义脚本用于执行特定任务。

## 🚀 快速启动方式（Docker）
```bash
# 启动服务
docker pull frdel/agent-zero-run
docker run -p 50001:80 frdel/agent-zero-run

# 访问地址
http://localhost:50001
```

## 📌 注意事项

1. **安全提示**：代理可能执行危险操作，请始终在隔离环境中运行。
2. **行为可控性**：所有行为由提示文件控制。

## 📚 如何阅读源码？推荐入手点

### 🧩 1. 代理核心逻辑
- **路径**: `python/agent.py`
- **作用**: 核心代理类，负责接收输入、决策、调用工具、维护状态等。

### ⚙️ 2. 工具系统
- **路径**: `python/tools/`
- **重点文件**:
  - `code_execution_tool.py`: 代码执行工具
  - `search_engine.py`: 搜索引擎集成
  - `memory_save.py` / `memory_load.py`: 记忆读写工具
  - `behaviour_adjustment.py`: 动态行为调整工具

### 📜 3. 上下文与历史管理
- **路径**: `python/helpers/history.py`
- **作用**: 处理对话历史的压缩、摘要生成、上下文优化。

### 💬 4. 提示系统
- **路径**: `prompts/default/`
- **重点文件**:
  - `agent.system.main.*.md`: 主要行为逻辑控制
  - `agent.system.tool.*.md`: 工具提示模板
  - `fw.code.*.md`: 代码执行相关的系统提示

### 🗂️ 5. 知识库与RAG
- **路径**: `python/helpers/knowledge_import.py`
- **作用**: 知识文件的导入、索引与检索。

### 🛠️ 6. 仪器模块
- **路径**: `instruments/`
- **作用**: 自定义脚本存放处，用于执行特定任务。

### 🧪 7. 扩展模块
- **路径**: `python/extensions/`
- **作用**: 非侵入式功能扩展。

## 📎 推荐学习路径

如果你是新手，建议按照以下顺序逐步了解源码：

1. **先看文档**：从 `docs/architecture.md` 开始，理解整体架构。
2. **入口文件**：查看 `run_ui.py` 和 `agent.py`，了解启动流程。
3. **工具系统**：研究 `tools/` 中的关键工具实现。
4. **历史与上下文**：深入 `history.py` 和压缩逻辑。
5. **提示系统**：查看 `prompts/default/` 中的提示文件。
6. **扩展与仪器**：探索 `extensions/` 和 `instruments/` 模块。

## 📦 示例核心代码摘录

### 1. 代理核心逻辑 (`python/agent.py`)
```python
class Agent:
    def __init__(self, config):
        self.config = config
        self.memory = Memory()
        self.tools = ToolManager(self)
    
    def handle_input(self, user_input):
        # 解析输入并决定调用哪个工具
        tool_name = self._decide_tool(user_input)
        tool = self.tools.get(tool_name)
        return tool.execute(user_input)

    def _decide_tool(self, input_text):
        # 使用提示工程来判断该调用哪个工具
        ...
```

### 2. 历史压缩与摘要 (`python/helpers/history.py`)
```python
async def compress_attention(self) -> bool:
    if len(self.messages) > 2:
        cnt_to_sum = math.ceil((len(self.messages) - 2) * TOPIC_COMPRESS_RATIO)
        msg_to_sum = self.messages[1 : cnt_to_sum + 1]
        summary = await self.summarize_messages(msg_to_sum)
        sum_msg_content = self.history.agent.parse_prompt("fw.msg_summary.md", summary=summary)
        sum_msg = Message(False, sum_msg_content)
        self.messages[1 : cnt_to_sum + 1] = [sum_msg]
        return True
    return False
```

### 3. 工具调用 (`python/tools/code_execution_tool.py`)
```python
class CodeExecution(Tool):
    async def execute(self, **kwargs):
        runtime = self.args.get("runtime", "").lower().strip()
        session = int(self.args.get("session", 0))

        if runtime == "python":
            response = await self.execute_python_code(code=self.args["code"], session=session)
        elif runtime == "nodejs":
            response = await self.execute_nodejs_code(code=self.args["code"], session=session)
        elif runtime == "terminal":
            response = await self.execute_terminal_command(command=self.args["code"], session=session)
        ...