# Mini-Agent 设计指南

> 面向 Agent 初级开发者的系统设计文档  
> 版本: 1.0  
> 更新日期: 2026-01-20

---

## 文档目标

本文档面向具备基础编程能力、希望设计开发企业级 AI Agent 系统的工程师。通过阅读本指南，您将理解：

1. **Agent 的核心概念与运行机制**
2. **Mini-Agent 的架构设计与实现细节**
3. **如何基于此设计构建自己的 Agent 系统**

---

## 一、Agent 系统核心概念

### 1.1 什么是 Agent？

Agent（智能体）是一个自主执行任务的智能系统，具备以下核心能力：

```
输入(任务) → [感知] → [决策] → [行动] → [观察] → [反馈循环] → 输出(结果)
```

**关键特性：**
- **自主性**：无需人工逐步指导，自行规划执行路径
- **工具使用能力**：可调用外部工具完成任务（Tool Calling）
- **上下文记忆**：维护对话历史和状态
- **反思与纠错**：根据执行结果调整策略

### 1.2 Agent vs. 简单 LLM 调用

| 维度 | 简单 LLM 调用 | Agent 系统 |
|-----|-------------|-----------|
| 交互方式 | 单轮问答 | 多轮循环（Agent Loop） |
| 能力边界 | 仅文本生成 | 工具调用 + 外部系统集成 |
| 状态管理 | 无状态 | 维护会话上下文 |
| 任务复杂度 | 简单查询 | 复杂多步骤任务 |
| 错误处理 | 无 | 重试、纠错机制 |

### 1.3 Agent 执行循环（Agent Loop）

```python
while not task_completed and step < max_steps:
    # 1. 感知：获取当前上下文
    context = get_current_context()
    
    # 2. 决策：LLM 生成响应（可能包含工具调用）
    response = llm.generate(context, tools)
    
    # 3. 行动：执行工具调用
    if response.has_tool_calls:
        results = execute_tools(response.tool_calls)
        context.append(results)
    else:
        # 无工具调用，任务完成
        return response.content
    
    # 4. 观察：检查结果并继续循环
    step += 1
```

---

## 二、Mini-Agent 架构设计

### 2.1 系统分层架构

```
┌─────────────────────────────────────────────────────────┐
│                  Application Layer                       │
│              (CLI / HTTP Server / ACP)                   │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                    Agent Core Layer                      │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│   │  Agent   │  │  Memory  │  │  Logger  │            │
│   │  Engine  │  │ Manager  │  │          │            │
│   └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                  Capability Layer                        │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│   │   LLM    │  │  Tools   │  │  Skills  │            │
│   │ Clients  │  │  System  │  │  System  │            │
│   └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│               Infrastructure Layer                       │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│   │  Config  │  │  Retry   │  │  Schema  │            │
│   │  System  │  │  Handler │  │          │            │
│   └──────────┘  └──────────┘  └──────────┘            │
└─────────────────────────────────────────────────────────┘
```

### 2.2 核心模块职责

| 模块 | 文件位置 | 核心职责 |
|-----|---------|---------|
| **Agent Engine** | `agent.py` | 执行 Agent Loop、管理执行流程 |
| **LLM Client** | `llm/` | 抽象多 Provider（Anthropic/OpenAI）调用 |
| **Tool System** | `tools/` | 工具定义、加载、执行 |
| **Skills System** | `tools/skill_loader.py` | 渐进式知识加载 |
| **Config System** | `config.py` | 配置管理与环境适配 |
| **Schema** | `schema/` | 统一数据结构定义 |
| **Logger** | `logger.py` | 完整执行日志记录 |

---

## 三、Agent 核心设计详解

### 3.1 Agent 执行引擎 (`agent.py`)

#### 3.1.1 核心数据结构

```python
class Agent:
    def __init__(
        self,
        llm_client: LLMClient,           # LLM 客户端
        system_prompt: str,              # 系统提示词
        tools: list[Tool],               # 可用工具列表
        max_steps: int = 50,             # 最大执行步数
        workspace_dir: str = "./workspace",  # 工作目录
        token_limit: int = 80000,        # Token 限制（触发摘要）
    ):
        self.llm = llm_client
        self.tools = {tool.name: tool for tool in tools}  # 工具字典
        self.messages: list[Message] = []  # 消息历史
        self.cancel_event: Optional[asyncio.Event] = None  # 取消信号
        self.api_total_tokens: int = 0   # API 累计 Token 用量
```

**设计要点：**
- **工具字典化**：O(1) 查找效率
- **消息历史**：完整保留上下文，支持摘要压缩
- **取消机制**：异步事件控制，支持 Ctrl+C / Esc 中断
- **Token 追踪**：实时监控，触发自动摘要

#### 3.1.2 执行循环实现

```python
async def run(self, cancel_event: Optional[asyncio.Event] = None) -> str:
    step = 0
    while step < self.max_steps:
        # ========== 检查点 1: 取消信号 ==========
        if self._check_cancelled():
            self._cleanup_incomplete_messages()
            return "Task cancelled by user."
        
        # ========== 检查点 2: Token 管理 ==========
        await self._summarize_messages()  # 超过限制时自动摘要
        
        # ========== 步骤 1: 调用 LLM ==========
        response = await self.llm.generate(
            messages=self.messages,
            tools=list(self.tools.values())
        )
        
        # 记录 Token 用量
        if response.usage:
            self.api_total_tokens = response.usage.total_tokens
        
        # ========== 步骤 2: 保存 Assistant 消息 ==========
        assistant_msg = Message(
            role="assistant",
            content=response.content,
            thinking=response.thinking,  # M2.1 扩展思考
            tool_calls=response.tool_calls,
        )
        self.messages.append(assistant_msg)
        
        # ========== 步骤 3: 判断是否完成 ==========
        if not response.tool_calls:
            return response.content  # 无工具调用，任务完成
        
        # ========== 步骤 4: 执行工具调用 ==========
        for tool_call in response.tool_calls:
            tool_name = tool_call.function.name
            arguments = tool_call.function.arguments
            
            # 工具执行
            if tool_name not in self.tools:
                result = ToolResult(
                    success=False,
                    error=f"Unknown tool: {tool_name}"
                )
            else:
                try:
                    tool = self.tools[tool_name]
                    result = await tool.execute(**arguments)
                except Exception as e:
                    result = ToolResult(
                        success=False,
                        error=f"Tool execution failed: {str(e)}"
                    )
            
            # ========== 步骤 5: 保存工具结果 ==========
            tool_msg = Message(
                role="tool",
                content=result.content if result.success else f"Error: {result.error}",
                tool_call_id=tool_call.id,
                name=tool_name,
            )
            self.messages.append(tool_msg)
        
        step += 1
    
    # 达到最大步数
    return f"Task couldn't be completed after {self.max_steps} steps."
```

**设计亮点：**

1. **多重安全检查**
   - 取消信号检查（用户中断）
   - Token 限制检查（防止上下文溢出）
   - 最大步数检查（防止无限循环）

2. **异步执行**
   - 所有 I/O 操作（LLM 调用、工具执行）均异步
   - 支持并发工具调用（可扩展）

3. **错误容错**
   - 工具执行失败不中断循环
   - 错误信息作为工具结果反馈给 LLM
   - 捕获所有异常，转为 ToolResult

4. **消息完整性**
   - 每个 Tool Call 必有对应 Tool Result
   - 取消时清理不完整消息，保持一致性

#### 3.1.3 Token 管理与自动摘要

```python
async def _summarize_messages(self):
    """当 Token 超限时自动摘要"""
    # 双重检测：本地估算 + API 报告
    estimated_tokens = self._estimate_tokens()
    should_summarize = (
        estimated_tokens > self.token_limit or 
        self.api_total_tokens > self.token_limit
    )
    
    if not should_summarize:
        return
    
    # ========== 摘要策略 ==========
    # 1. 保留所有 user 消息（用户意图不可丢失）
    # 2. 压缩每轮 assistant + tool 消息为摘要
    # 3. 结构：system → user1 → summary1 → user2 → summary2 → ...
    
    user_indices = [i for i, msg in enumerate(self.messages) 
                    if msg.role == "user" and i > 0]
    
    new_messages = [self.messages[0]]  # 保留 system prompt
    
    for i, user_idx in enumerate(user_indices):
        new_messages.append(self.messages[user_idx])  # 保留 user 消息
        
        # 确定本轮执行消息范围
        if i < len(user_indices) - 1:
            next_user_idx = user_indices[i + 1]
        else:
            next_user_idx = len(self.messages)
        
        execution_messages = self.messages[user_idx + 1 : next_user_idx]
        
        # 调用 LLM 生成摘要
        if execution_messages:
            summary_text = await self._create_summary(execution_messages, i + 1)
            new_messages.append(Message(
                role="user",
                content=f"[Assistant Execution Summary]\n\n{summary_text}"
            ))
    
    self.messages = new_messages
```

**Token 估算实现：**

```python
def _estimate_tokens(self) -> int:
    """使用 tiktoken 精确计算 Token 数"""
    try:
        encoding = tiktoken.get_encoding("cl100k_base")  # GPT-4/Claude 通用
    except Exception:
        return self._estimate_tokens_fallback()
    
    total_tokens = 0
    for msg in self.messages:
        # 文本内容
        if isinstance(msg.content, str):
            total_tokens += len(encoding.encode(msg.content))
        
        # Thinking 内容
        if msg.thinking:
            total_tokens += len(encoding.encode(msg.thinking))
        
        # Tool Calls
        if msg.tool_calls:
            total_tokens += len(encoding.encode(str(msg.tool_calls)))
        
        # 消息元数据开销（约 4 tokens）
        total_tokens += 4
    
    return total_tokens
```

**设计优势：**
- **无损用户意图**：所有 user 消息全部保留
- **压缩执行细节**：工具调用过程摘要化
- **无限长度任务**：理论上可无限执行
- **成本可控**：防止单次请求 Token 爆炸

---

### 3.2 工具系统设计 (`tools/`)

#### 3.2.1 Tool 抽象基类

```python
class Tool:
    """所有工具的统一接口"""
    
    @property
    def name(self) -> str:
        """工具名称（唯一标识）"""
        raise NotImplementedError
    
    @property
    def description(self) -> str:
        """工具功能描述（供 LLM 理解）"""
        raise NotImplementedError
    
    @property
    def parameters(self) -> dict[str, Any]:
        """参数 Schema（JSON Schema 格式）"""
        raise NotImplementedError
    
    async def execute(self, **kwargs) -> ToolResult:
        """执行工具（异步）"""
        raise NotImplementedError
    
    def to_schema(self) -> dict:
        """转 Anthropic 格式"""
        return {
            "name": self.name,
            "description": self.description,
            "input_schema": self.parameters,
        }
    
    def to_openai_schema(self) -> dict:
        """转 OpenAI 格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters,
            },
        }
```

**设计模式：**
- **模板方法模式**：定义统一接口，子类实现细节
- **适配器模式**：`to_schema()` / `to_openai_schema()` 适配不同 Provider
- **策略模式**：每个 Tool 是独立的执行策略

#### 3.2.2 Tool 标准化返回

```python
class ToolResult(BaseModel):
    """工具执行结果的统一结构"""
    success: bool              # 是否成功
    content: str = ""          # 成功时的返回内容
    error: str | None = None   # 失败时的错误信息
```

**设计原则：**
- **成功与失败路径分离**：LLM 可根据 success 判断
- **错误信息保留**：失败不中断循环，作为反馈继续
- **内容与错误互斥**：清晰的语义

#### 3.2.3 工具实现示例

```python
class ReadTool(Tool):
    """读取文件内容工具"""
    
    def __init__(self, workspace_dir: str):
        self.workspace = Path(workspace_dir)
    
    @property
    def name(self) -> str:
        return "read_file"
    
    @property
    def description(self) -> str:
        return """Read file contents with line range support.
        
        Use Cases:
        - Read specific lines: start_line and end_line
        - Read entire file: omit both parameters
        - Pagination: use start_line with max_lines
        """
    
    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "file_path": {
                    "type": "string",
                    "description": "Relative or absolute file path"
                },
                "start_line": {
                    "type": "integer",
                    "description": "Starting line (1-based, optional)"
                },
                "end_line": {
                    "type": "integer",
                    "description": "Ending line (inclusive, optional)"
                },
            },
            "required": ["file_path"],
        }
    
    async def execute(
        self,
        file_path: str,
        start_line: int | None = None,
        end_line: int | None = None,
    ) -> ToolResult:
        try:
            # 路径解析
            path = self._resolve_path(file_path)
            
            # 文件检查
            if not path.exists():
                return ToolResult(
                    success=False,
                    error=f"File not found: {file_path}"
                )
            
            # 读取内容
            content = path.read_text(encoding="utf-8")
            lines = content.splitlines()
            
            # 行范围处理
            if start_line is not None or end_line is not None:
                start = (start_line - 1) if start_line else 0
                end = end_line if end_line else len(lines)
                lines = lines[start:end]
            
            return ToolResult(
                success=True,
                content="\n".join(lines)
            )
            
        except Exception as e:
            return ToolResult(
                success=False,
                error=f"Failed to read file: {str(e)}"
            )
    
    def _resolve_path(self, file_path: str) -> Path:
        """解析相对路径"""
        path = Path(file_path)
        if not path.is_absolute():
            path = self.workspace / path
        return path.resolve()
```

**关键设计：**
1. **工作空间隔离**：相对路径基于 workspace_dir
2. **详细描述**：description 包含使用场景
3. **灵活参数**：可选参数支持多种使用方式
4. **异常捕获**：所有异常转为 ToolResult
5. **路径安全**：resolve() 防止路径遍历攻击

---

### 3.3 LLM 客户端抽象 (`llm/`)

#### 3.3.1 多 Provider 统一接口

```python
class LLMClientBase:
    """LLM 客户端基类"""
    
    def __init__(
        self,
        api_key: str,
        api_base: str,
        model: str,
        retry_config: RetryConfig | None = None,
    ):
        self.api_key = api_key
        self.api_base = api_base
        self.model = model
        self.retry_config = retry_config or RetryConfig()
        self.retry_callback = None
    
    async def generate(
        self,
        messages: list[Message],
        tools: list[Tool] | None = None,
    ) -> LLMResponse:
        """生成响应（抽象方法）"""
        raise NotImplementedError
```

#### 3.3.2 LLMClient 包装器

```python
class LLMClient:
    """统一 LLM 客户端，自动选择底层实现"""
    
    def __init__(
        self,
        api_key: str,
        provider: LLMProvider = LLMProvider.ANTHROPIC,
        api_base: str = "https://api.minimaxi.com",
        model: str = "MiniMax-M2.1",
        retry_config: RetryConfig | None = None,
    ):
        # ========== 端点自动适配 ==========
        api_base = api_base.rstrip("/")
        is_minimax = any(d in api_base for d in self.MINIMAX_DOMAINS)
        
        if is_minimax:
            # MiniMax API：自动追加协议后缀
            api_base = api_base.replace("/anthropic", "").replace("/v1", "")
            if provider == LLMProvider.ANTHROPIC:
                full_api_base = f"{api_base}/anthropic"
            else:
                full_api_base = f"{api_base}/v1"
        else:
            # 第三方 API：保持原样
            full_api_base = api_base
        
        # ========== 实例化具体客户端 ==========
        if provider == LLMProvider.ANTHROPIC:
            self._client = AnthropicClient(api_key, full_api_base, model, retry_config)
        elif provider == LLMProvider.OPENAI:
            self._client = OpenAIClient(api_key, full_api_base, model, retry_config)
        else:
            raise ValueError(f"Unsupported provider: {provider}")
    
    async def generate(self, messages, tools=None) -> LLMResponse:
        return await self._client.generate(messages, tools)
```

**设计优势：**
- **Provider 透明**：业务层无需关心底层实现
- **端点智能适配**：MiniMax API 自动追加后缀
- **重试统一管理**：retry_config 统一配置
- **可扩展性**：新增 Provider 只需实现 LLMClientBase

#### 3.3.3 消息格式转换

**Anthropic 格式：**
```python
def _convert_messages(self, messages: list[Message]):
    system_message = None
    api_messages = []
    
    for msg in messages:
        if msg.role == "system":
            system_message = msg.content  # 系统消息单独提取
        
        elif msg.role == "assistant" and (msg.thinking or msg.tool_calls):
            # 构建内容块
            content_blocks = []
            if msg.thinking:
                content_blocks.append({"type": "thinking", "thinking": msg.thinking})
            if msg.content:
                content_blocks.append({"type": "text", "text": msg.content})
            if msg.tool_calls:
                for tc in msg.tool_calls:
                    content_blocks.append({
                        "type": "tool_use",
                        "id": tc.id,
                        "name": tc.function.name,
                        "input": tc.function.arguments,
                    })
            api_messages.append({"role": "assistant", "content": content_blocks})
        
        elif msg.role == "tool":
            # Tool Result 使用 user 角色
            api_messages.append({
                "role": "user",
                "content": [{
                    "type": "tool_result",
                    "tool_use_id": msg.tool_call_id,
                    "content": msg.content,
                }]
            })
    
    return system_message, api_messages
```

**OpenAI 格式：**
```python
def _convert_messages(self, messages: list[Message]):
    api_messages = []
    
    for msg in messages:
        if msg.role == "system":
            api_messages.append({"role": "system", "content": msg.content})
        
        elif msg.role == "assistant":
            assistant_msg = {"role": "assistant"}
            if msg.content:
                assistant_msg["content"] = msg.content
            if msg.tool_calls:
                assistant_msg["tool_calls"] = [
                    {
                        "id": tc.id,
                        "type": "function",
                        "function": {
                            "name": tc.function.name,
                            "arguments": json.dumps(tc.function.arguments),
                        }
                    }
                    for tc in msg.tool_calls
                ]
            # ========== 关键：保留 reasoning_details ==========
            if msg.thinking:
                assistant_msg["reasoning_details"] = [{"text": msg.thinking}]
            api_messages.append(assistant_msg)
        
        elif msg.role == "tool":
            api_messages.append({
                "role": "tool",
                "tool_call_id": msg.tool_call_id,
                "content": msg.content,
            })
    
    return None, api_messages
```

**设计要点：**
- **协议差异抽象**：不同 Provider 的格式转换封装在客户端内部
- **Thinking 支持**：M2.1 的扩展思考字段正确传递
- **Tool Result 角色差异**：Anthropic 用 user，OpenAI 用 tool

---

### 3.4 Skills 系统设计 (`tools/skill_loader.py`)

#### 3.4.1 渐进式披露机制

```
┌─────────────────────────────────────────────────────────┐
│  Level 1: Metadata Only (启动时注入 System Prompt)       │
│  - Skill Name                                            │
│  - Skill Description                                     │
│  → Agent 知道有哪些 Skill 可用                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ Agent 判断需要时调用 get_skill()
┌─────────────────────────────────────────────────────────┐
│  Level 2: Full Content (按需加载)                        │
│  - 完整 SKILL.md 内容                                     │
│  - 操作指南、步骤说明                                      │
│  → Agent 获得详细知识                                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ Skill 引用脚本/资源
┌─────────────────────────────────────────────────────────┐
│  Level 3+: Resources (Agent 使用 read_file/bash 访问)    │
│  - scripts/ 目录脚本                                      │
│  - references/ 参考文档                                   │
│  → Agent 执行具体操作                                     │
└─────────────────────────────────────────────────────────┘
```

**设计优势：**
- **减少初始 Token**：不在启动时加载所有 Skill 内容
- **按需加载**：Agent 自主判断需要哪个 Skill
- **资源延迟访问**：脚本文件不占用上下文，用时再读

#### 3.4.2 Skill 文件结构

```markdown
---
name: my-skill                    # 小写连字符命名
description: 技能功能描述           # 何时使用此 Skill
allowed-tools:                    # 预授权工具（可选）
  - bash
  - read_file
metadata:                         # 自定义元数据
  author: "name"
  version: "1.0"
---

# Skill 内容（Markdown）

## 适用场景
描述此 Skill 解决的问题...

## 操作步骤
1. 步骤一：使用 `bash` 工具执行...
2. 步骤二：使用 `read_file` 读取...

## 脚本示例
可以调用 `scripts/helper.py` 辅助处理...
```

#### 3.4.3 路径自动转换

```python
def _process_skill_paths(self, content: str, skill_dir: Path) -> str:
    """将相对路径转为绝对路径，方便 Agent 访问"""
    
    # 模式 1: 目录路径（scripts/, references/, assets/）
    def replace_dir_path(match):
        prefix = match.group(1)  # "python " 或 "`"
        rel_path = match.group(2)
        abs_path = skill_dir / rel_path
        return f"{prefix}{abs_path}" if abs_path.exists() else match.group(0)
    
    pattern = r"(python\s+|`)((?:scripts|references|assets)/[^\s`\)]+)"
    content = re.sub(pattern, replace_dir_path, content)
    
    # 模式 2: Markdown 文档引用
    def replace_doc_path(match):
        prefix = match.group(1)
        filename = match.group(2)
        abs_path = skill_dir / filename
        return f"{prefix}`{abs_path}` (use read_file to access)" \
               if abs_path.exists() else match.group(0)
    
    pattern = r"(see|read|refer to)\s+([a-zA-Z0-9_-]+\.(?:md|txt))"
    content = re.sub(pattern, replace_doc_path, content, flags=re.IGNORECASE)
    
    return content
```

**设计意图：**
- **Agent 无需关心路径**：Skill 中引用的资源自动转为绝对路径
- **提示增强**：在路径后添加 "use read_file to access" 提示
- **跨工作目录支持**：无论 Agent 在哪个目录执行，都能访问 Skill 资源

---

### 3.5 配置系统设计 (`config.py`)

#### 3.5.1 配置分层结构

```python
class Config(BaseModel):
    """根配置类"""
    llm: LLMConfig        # LLM 相关配置
    agent: AgentConfig    # Agent 行为配置
    tools: ToolsConfig    # 工具开关配置

class LLMConfig(BaseModel):
    api_key: str
    api_base: str
    model: str
    provider: str              # "anthropic" | "openai"
    retry: RetryConfig         # 重试配置

class AgentConfig(BaseModel):
    max_steps: int = 50
    workspace_dir: str = "./workspace"
    system_prompt_path: str = "system_prompt.md"

class ToolsConfig(BaseModel):
    enable_file_tools: bool = True
    enable_bash: bool = True
    enable_skills: bool = True
    enable_mcp: bool = True
    mcp: MCPConfig             # MCP 超时配置
```

#### 3.5.2 配置文件搜索策略

```python
@classmethod
def find_config_file(cls, filename: str) -> Path | None:
    """优先级搜索配置文件"""
    
    # 优先级 1: 开发模式 - mini_agent/config/{filename}
    dev_config = Path.cwd() / "mini_agent" / "config" / filename
    if dev_config.exists():
        return dev_config
    
    # 优先级 2: 用户配置 - ~/.mini-agent/config/{filename}
    user_config = Path.home() / ".mini-agent" / "config" / filename
    if user_config.exists():
        return user_config
    
    # 优先级 3: 包安装目录
    package_config = cls.get_package_dir() / "config" / filename
    if package_config.exists():
        return package_config
    
    return None
```

**设计优势：**
- **开发友好**：开发时直接修改 `mini_agent/config/`
- **用户可定制**：用户配置在 `~/.mini-agent/config/` 不被覆盖
- **包自带默认**：安装后自带默认配置

#### 3.5.3 环境变量覆盖（扩展设计）

```python
class BusinessConfig(BaseModel):
    """业务配置（支持环境变量）"""
    rag_api_base: str
    rag_api_key: str
    product_api_base: str
    
    @classmethod
    def from_env(cls, config_data: dict):
        """从环境变量覆盖配置"""
        env = os.getenv("MINIAGENT_ENV", "public")
        
        if env == "private":
            rag_base = os.getenv("PRIVATE_RAG_API_BASE") or config_data["rag_api_base"]
        else:
            rag_base = os.getenv("PUBLIC_RAG_API_BASE") or config_data["rag_api_base"]
        
        return cls(
            rag_api_base=rag_base,
            rag_api_key=os.getenv("RAG_API_KEY", config_data["rag_api_key"]),
            product_api_base=os.getenv("PRODUCT_API_BASE", config_data["product_api_base"]),
        )
```

---

### 3.6 MCP 协议集成 (`tools/mcp_loader.py`)

#### 3.6.1 MCP 概念

**MCP (Model Context Protocol)** 是一种标准化协议，用于 AI 应用与外部工具/数据源通信。

```
┌─────────────┐              ┌─────────────┐
│  AI Agent   │ ◄──MCP──► │  MCP Server │
│ (MCP Client)│              │  (Tool)     │
└─────────────┘              └─────────────┘
```

**传输方式：**
- **STDIO**：本地进程通信（`npx`, `uvx` 等）
- **SSE**：Server-Sent Events（单向推送）
- **Streamable HTTP**：双向 HTTP 流式通信

#### 3.6.2 MCP 工具包装

```python
class MCPTool(Tool):
    """MCP 工具的 Tool 接口适配器"""
    
    def __init__(
        self,
        name: str,
        description: str,
        parameters: dict,
        session: ClientSession,
        execute_timeout: float,
    ):
        self._name = name
        self._description = description
        self._parameters = parameters
        self._session = session
        self._execute_timeout = execute_timeout
    
    async def execute(self, **kwargs) -> ToolResult:
        """调用 MCP 服务的工具"""
        try:
            async with asyncio.timeout(self._execute_timeout):
                result = await self._session.call_tool(self._name, arguments=kwargs)
            
            # 提取 MCP 响应内容
            content_parts = []
            for item in result.content:
                if hasattr(item, "text"):
                    content_parts.append(item.text)
            
            return ToolResult(
                success=not result.isError,
                content="\n".join(content_parts)
            )
        except TimeoutError:
            return ToolResult(
                success=False,
                error=f"MCP tool timeout after {self._execute_timeout}s"
            )
```

**设计要点：**
- **适配器模式**：将 MCP 协议适配为 Tool 接口
- **超时保护**：防止 MCP 服务卡死
- **错误处理**：网络异常转为 ToolResult

#### 3.6.3 动态工具加载

```python
async def load_mcp_tools_async(config_path: str) -> list[Tool]:
    """从配置文件加载所有 MCP 工具"""
    
    with open(config_path) as f:
        config = json.load(f)
    
    all_tools = []
    for server_name, server_config in config["mcpServers"].items():
        if server_config.get("disabled"):
            continue
        
        # 创建连接
        connection = MCPServerConnection(
            name=server_name,
            connection_type=_determine_connection_type(server_config),
            url=server_config.get("url"),
            command=server_config.get("command"),
            ...
        )
        
        # 连接并获取工具列表
        success = await connection.connect()
        if success:
            all_tools.extend(connection.tools)
    
    return all_tools
```

---

## 四、HTTP 服务层设计（扩展）

### 4.1 服务架构

```
移动端 App
    │
    ▼ HTTPS + SSE
┌─────────────────────────────────────┐
│      FastAPI HTTP Server            │
│  ┌──────────────────────────────┐   │
│  │  /api/v1/sessions (POST)     │   │  ← 创建会话
│  │  /api/v1/prompt (POST, SSE)  │   │  ← 流式对话
│  │  /health, /ready              │   │  ← 健康检查
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│      Session Manager                │
│  dict[session_id, Agent]            │
└─────────────────────────────────────┘
```

### 4.2 流式响应实现

```python
from sse_starlette.sse import EventSourceResponse

@app.post("/api/v1/prompt")
async def prompt(request: PromptRequest) -> EventSourceResponse:
    """流式返回 Agent 响应"""
    
    # 获取会话
    agent = session_manager.get(request.session_id)
    
    async def event_generator():
        """SSE 事件生成器"""
        try:
            agent.add_user_message(request.message)
            
            # 流式执行 Agent
            async for event in agent.run_streaming():
                # 将 AgentEvent 转为 SSE 事件
                if event.type == "thinking":
                    yield {
                        "event": "thinking",
                        "data": json.dumps({"content": event.content})
                    }
                elif event.type == "tool_call_start":
                    yield {
                        "event": "tool_call_start",
                        "data": json.dumps({
                            "tool": event.tool_name,
                            "args": event.arguments
                        })
                    }
                elif event.type == "content_delta":
                    yield {
                        "event": "content_delta",
                        "data": json.dumps({"delta": event.delta})
                    }
                elif event.type == "complete":
                    yield {
                        "event": "complete",
                        "data": json.dumps({"content": event.content})
                    }
        except Exception as e:
            yield {
                "event": "error",
                "data": json.dumps({"error": str(e)})
            }
    
    return EventSourceResponse(event_generator())
```

### 4.3 Agent 流式事件扩展

```python
# 在 agent.py 中新增
async def run_streaming(self) -> AsyncGenerator[AgentEvent, None]:
    """流式执行 Agent，yield 中间事件"""
    
    step = 0
    while step < self.max_steps:
        # LLM 调用前
        yield AgentEvent(type="step_start", step=step)
        
        # 调用 LLM
        response = await self.llm.generate(self.messages, list(self.tools.values()))
        
        # Thinking 事件
        if response.thinking:
            yield AgentEvent(type="thinking", content=response.thinking)
        
        # 保存消息
        self.messages.append(Message(...))
        
        # 无工具调用，完成
        if not response.tool_calls:
            yield AgentEvent(type="complete", content=response.content)
            return
        
        # 工具调用事件
        for tool_call in response.tool_calls:
            yield AgentEvent(
                type="tool_call_start",
                tool_name=tool_call.function.name,
                arguments=tool_call.function.arguments
            )
            
            # 执行工具
            result = await self.tools[tool_call.function.name].execute(...)
            
            yield AgentEvent(
                type="tool_call_end",
                tool_name=tool_call.function.name,
                result=result.content
            )
            
            self.messages.append(Message(role="tool", ...))
        
        step += 1
```

---

## 五、设计自己的 Agent 系统

### 5.1 最小可行实现（MVP）

**核心组件清单：**

1. **Agent 执行引擎** - 必需
   - Agent Loop 主循环
   - 消息历史管理
   - 工具调用协调

2. **LLM 客户端** - 必需
   - 至少支持一个 Provider
   - 消息格式转换
   - Tool Calling 支持

3. **工具系统** - 必需
   - Tool 基类定义
   - 至少 2-3 个基础工具（如 read_file, bash）
   - ToolResult 统一返回

4. **配置管理** - 推荐
   - API Key 配置
   - 工具开关

5. **错误处理** - 推荐
   - 工具执行异常捕获
   - 重试机制（可选）

**MVP 代码结构：**
```
my_agent/
├── agent.py           # Agent 执行引擎
├── llm_client.py      # LLM 客户端
├── tools/
│   ├── base.py        # Tool 基类
│   ├── read_tool.py
│   └── bash_tool.py
├── config.yaml        # 配置文件
└── main.py            # 入口
```

### 5.2 进阶功能扩展

**功能清单（按优先级）：**

| 功能 | 优先级 | 复杂度 | 价值 |
|-----|-------|-------|------|
| Token 管理 + 摘要 | ⭐⭐⭐ | 中 | 支持长任务 |
| 多 Provider 支持 | ⭐⭐⭐ | 低 | 灵活切换 LLM |
| 取消机制 | ⭐⭐ | 低 | 用户体验 |
| 流式响应 | ⭐⭐ | 中 | 实时反馈 |
| Skills 系统 | ⭐⭐ | 中 | 知识管理 |
| MCP 集成 | ⭐ | 高 | 标准化工具 |
| HTTP 服务 | ⭐ | 中 | 移动端支持 |
| 日志系统 | ⭐⭐⭐ | 低 | Debug 必备 |

### 5.3 企业级扩展建议

#### 5.3.1 安全性增强

```python
# 工具权限控制
class ToolPermission:
    def __init__(self, allowed_tools: set[str]):
        self.allowed_tools = allowed_tools
    
    def check(self, tool_name: str) -> bool:
        return tool_name in self.allowed_tools

# Agent 初始化
agent = Agent(
    llm_client=llm,
    tools=tools,
    tool_permission=ToolPermission({"read_file", "query_db"})  # 白名单
)
```

#### 5.3.2 可观测性

```python
# 分布式追踪
import opentelemetry

class Agent:
    async def run(self):
        with tracer.start_as_current_span("agent_run") as span:
            span.set_attribute("task", task_description)
            
            for step in range(self.max_steps):
                with tracer.start_span("llm_call") as llm_span:
                    response = await self.llm.generate(...)
                    llm_span.set_attribute("tokens", response.usage.total_tokens)
```

#### 5.3.3 会话持久化

```python
# Redis 会话存储
class RedisSessionManager:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    async def save_session(self, session_id: str, agent: Agent):
        """序列化 Agent 状态到 Redis"""
        state = {
            "messages": [msg.model_dump() for msg in agent.messages],
            "api_total_tokens": agent.api_total_tokens,
        }
        await self.redis.set(f"session:{session_id}", json.dumps(state))
    
    async def load_session(self, session_id: str) -> Agent:
        """从 Redis 恢复 Agent"""
        state = json.loads(await self.redis.get(f"session:{session_id}"))
        agent = Agent(...)
        agent.messages = [Message(**msg) for msg in state["messages"]]
        agent.api_total_tokens = state["api_total_tokens"]
        return agent
```

#### 5.3.4 成本控制

```python
class CostTracker:
    """Token 成本追踪"""
    
    PRICING = {
        "MiniMax-M2.1": {"input": 0.01, "output": 0.03},  # 元/1K tokens
    }
    
    def calculate_cost(self, model: str, usage: TokenUsage) -> float:
        pricing = self.PRICING[model]
        input_cost = (usage.prompt_tokens / 1000) * pricing["input"]
        output_cost = (usage.completion_tokens / 1000) * pricing["output"]
        return input_cost + output_cost

# 在 Agent 中集成
class Agent:
    def __init__(self, ..., cost_tracker: CostTracker):
        self.cost_tracker = cost_tracker
        self.total_cost = 0.0
    
    async def run(self):
        ...
        if response.usage:
            cost = self.cost_tracker.calculate_cost(self.model, response.usage)
            self.total_cost += cost
            
            # 成本限制
            if self.total_cost > self.max_cost:
                raise CostLimitExceededError(...)
```

---

## 六、常见问题与最佳实践

### 6.1 消息历史管理

**问题：** 消息历史过长导致 Token 超限

**解决方案：**
1. **固定窗口**：只保留最近 N 轮对话
   ```python
   if len(self.messages) > self.max_messages:
       self.messages = [self.messages[0]] + self.messages[-self.max_messages:]
   ```

2. **自动摘要**：压缩历史（Mini-Agent 采用）
3. **滑动窗口 + 关键信息保留**：保留重要的 user 消息

### 6.2 工具调用失败处理

**问题：** 工具执行失败导致任务中断

**解决方案：**
1. **错误作为反馈**：将错误信息作为 ToolResult 返回给 LLM
   ```python
   except Exception as e:
       return ToolResult(success=False, error=str(e))
   ```

2. **重试机制**：在 Agent 层实现工具调用重试
   ```python
   for retry in range(max_retries):
       result = await tool.execute(**args)
       if result.success:
           break
   ```

3. **降级策略**：提供备用工具
   ```python
   if primary_tool_failed:
       result = await fallback_tool.execute(**args)
   ```

### 6.3 LLM 幻觉处理

**问题：** LLM 捏造工具名称或参数

**解决方案：**
1. **工具名称验证**
   ```python
   if tool_name not in self.tools:
       return ToolResult(success=False, error=f"Unknown tool: {tool_name}")
   ```

2. **参数 Schema 验证**
   ```python
   from jsonschema import validate
   
   try:
       validate(instance=arguments, schema=tool.parameters)
   except ValidationError as e:
       return ToolResult(success=False, error=f"Invalid arguments: {e}")
   ```

3. **System Prompt 明确工具列表**
   ```
   Available tools:
   - read_file: Read file contents (parameters: file_path, start_line, end_line)
   - bash: Execute shell command (parameters: command)
   ```

### 6.4 循环调用检测

**问题：** Agent 陷入重复调用同一工具的死循环

**解决方案：**
1. **步数限制**：`max_steps` 硬性终止
2. **重复检测**：检测连续相同的工具调用
   ```python
   def _detect_loop(self):
       recent_calls = [
           (msg.tool_calls[0].function.name, msg.tool_calls[0].function.arguments)
           for msg in self.messages[-3:] 
           if msg.role == "assistant" and msg.tool_calls
       ]
       if len(recent_calls) == 3 and recent_calls[0] == recent_calls[1] == recent_calls[2]:
           raise LoopDetectedError("Agent is stuck in a loop")
   ```

3. **Prompt 引导**：在 System Prompt 中强调避免重复

---

## 七、总结与展望

### 7.1 Mini-Agent 核心设计优势

| 设计点 | 优势 | 适用场景 |
|-------|------|---------|
| **Agent Loop** | 自主任务执行 | 复杂多步骤任务 |
| **Token 自动摘要** | 无限长度支持 | 长对话、大型项目 |
| **工具系统抽象** | 易扩展、统一接口 | 企业系统集成 |
| **多 Provider 支持** | 灵活切换 LLM | 成本优化、备份 |
| **Skills 渐进披露** | 节省 Token | 知识密集型场景 |
| **MCP 协议集成** | 标准化工具调用 | 跨平台能力复用 |
| **流式响应** | 实时反馈 | 移动端、Web 应用 |

### 7.2 构建企业 Agent 的关键决策

1. **选择 LLM Provider**
   - MiniMax M2.1：高性价比、长上下文、中文优化
   - Anthropic Claude：思维链强、工具调用稳定
   - OpenAI GPT-4：生态成熟、兼容性好

2. **工具设计策略**
   - **原子化**：每个工具职责单一
   - **幂等性**：相同输入产生相同输出
   - **明确边界**：通过 description 明确功能范围

3. **系统提示词设计**
   - **角色定位**：明确 Agent 身份和能力
   - **工具列表**：详细说明每个工具的用法
   - **约束条件**：安全规则、隐私保护
   - **示例对话**：few-shot 引导

4. **可观测性建设**
   - **完整日志**：请求/响应/工具调用全记录
   - **性能指标**：Token 用量、响应时间、成功率
   - **错误追踪**：异常堆栈、失败原因分类

### 7.3 未来演进方向

1. **多 Agent 协作**
   - Master-Worker 模式
   - Agent 间通信协议
   - 任务分解与结果聚合

2. **记忆系统增强**
   - 向量数据库集成
   - 长期记忆 vs. 短期记忆
   - 记忆检索与更新

3. **规划能力强化**
   - ReAct 模式（推理 + 行动）
   - Tree of Thoughts（思维树搜索）
   - 自我反思与纠错

4. **安全与合规**
   - 敏感信息过滤
   - 操作审计日志
   - 权限细粒度控制

---

## 八、参考资源

### 8.1 相关论文

- **ReAct**: [https://arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)
- **Tool Learning**: [https://arxiv.org/abs/2304.08354](https://arxiv.org/abs/2304.08354)
- **Agent Prompting**: [https://arxiv.org/abs/2305.14687](https://arxiv.org/abs/2305.14687)

### 8.2 开源项目

- **LangChain**: [https://github.com/langchain-ai/langchain](https://github.com/langchain-ai/langchain)
- **AutoGPT**: [https://github.com/Significant-Gravitas/AutoGPT](https://github.com/Significant-Gravitas/AutoGPT)
- **Model Context Protocol**: [https://modelcontextprotocol.io](https://modelcontextprotocol.io)

### 8.3 Mini-Agent 文档

- **开发指南**: [DEVELOPMENT_GUIDE.md](DEVELOPMENT_GUIDE.md)
- **生产部署**: [PRODUCTION_GUIDE.md](PRODUCTION_GUIDE.md)
- **业务集成**: [BUSINESS_INTEGRATION.md](BUSINESS_INTEGRATION.md)

---

*本文档持续更新，欢迎贡献您的实践经验和改进建议。*
