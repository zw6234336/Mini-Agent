# Mini-Agent 业务能力接入开发文档

> 版本: 1.0  
> 更新日期: 2026-01-20  
> 状态: 待补充业务接口信息

---

## 一、项目概述

### 1.1 目标
将公司业务能力接入 Mini-Agent 框架，为移动端提供 Streamable HTTP 服务。

### 1.2 核心能力
| 能力 | 实现方式 | 状态 |
|------|---------|------|
| 意图识别 | 自定义 Tool（LLM + 提示词） | 待开发 |
| RAG 查询 | 自定义 Tool（API 封装） | 待开发 |
| 产品编码映射 | 自定义 Tool（API 封装） | 待开发 |
| 老服务接入 | MCP 服务配置 | 待配置 |
| 移动端服务 | FastAPI + SSE | 待开发 |

### 1.3 部署环境
- 私有云端点: `${PRIVATE_API_BASE}` (待补充)
- 公有云端点: `${PUBLIC_API_BASE}` (待补充)
- 环境切换: `MINIAGENT_ENV=private|public`

---

## 二、架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                      移动端 App                              │
└─────────────────────────────────────────────────────────────┘
                              │ HTTP + SSE
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 统一网关（认证/限流）                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Mini-Agent HTTP Server                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  /api/v1/sessions    - 会话管理                      │    │
│  │  /api/v1/prompt      - 流式提示响应                  │    │
│  │  /health, /ready     - 健康检查                      │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Agent 核心引擎                            │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐   │
│  │ IntentTool│ │RagQueryTool│ │ProductCode│ │  MCP服务   │   │
│  │ (意图识别) │ │ (RAG查询)  │ │  Tool     │ │ (老服务)   │   │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌──────────┐        ┌──────────┐        ┌──────────┐
    │  LLM API │        │  RAG API │        │ 业务 API  │
    │ (MiniMax)│        │          │        │  (MCP)   │
    └──────────┘        └──────────┘        └──────────┘
```

---

## 三、业务工具详细设计

### 3.1 意图识别工具 (IntentTool)

**功能**: 识别用户输入的业务意图，返回意图分类结果。

**实现方式**: LLM + 提示词模板

#### 3.1.1 提示词模板

**System Prompt**: (待补充)
```
<!-- TODO: 请提供意图识别的 System Prompt -->

```

**User Prompt 模板**: (待补充)
```
<!-- TODO: 请提供 User Prompt 模板，使用 {user_input} 作为用户输入占位符 -->

```

#### 3.1.2 意图分类列表

| 意图代码 | 意图名称 | 描述 | 后续动作 |
|---------|---------|------|---------|
| <!-- TODO --> | <!-- TODO --> | <!-- TODO --> | <!-- TODO --> |
| | | | |
| | | | |

#### 3.1.3 工具定义

```python
# mini_agent/tools/business/intent_tool.py

class IntentTool(Tool):
    name = "recognize_intent"
    description = "识别用户输入的业务意图，返回意图分类结果"
    
    parameters = {
        "type": "object",
        "properties": {
            "user_input": {
                "type": "string",
                "description": "用户的原始输入文本"
            }
        },
        "required": ["user_input"]
    }
    
    # 返回格式
    # {
    #     "intent_code": "xxx",
    #     "intent_name": "xxx", 
    #     "confidence": 0.95,
    #     "entities": {...}  # 提取的实体（如有）
    # }
```

---

### 3.2 RAG 查询工具 (RagQueryTool)

**功能**: 基于意图和用户问题，从知识库检索相关信息并生成回答。

#### 3.2.1 API 接口信息

**接口地址**: (待补充)
- 私有云: `${PRIVATE_RAG_API_BASE}/...`
- 公有云: `${PUBLIC_RAG_API_BASE}/...`

**请求方式**: (待补充) `POST` / `GET`

**请求头**: (待补充)
```json
{
    "Authorization": "Bearer ${RAG_API_KEY}",
    "Content-Type": "application/json"
}
```

**请求参数**: (待补充)
```json
{
    // TODO: 请提供 RAG API 请求参数结构
}
```

**响应格式**: (待补充)
```json
{
    // TODO: 请提供 RAG API 响应格式
}
```

#### 3.2.2 工具定义

```python
# mini_agent/tools/business/rag_tool.py

class RagQueryTool(Tool):
    name = "query_knowledge_base"
    description = "从知识库检索信息回答用户问题"
    
    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "用户的问题"
            },
            "intent": {
                "type": "string",
                "description": "识别出的意图代码（可选）"
            }
            # TODO: 根据实际 API 补充其他参数
        },
        "required": ["query"]
    }
```

---

### 3.3 产品编码工具 (ProductCodeTool)

**功能**: 将产品名称/别名映射为标准产品编码。

#### 3.3.1 API 接口信息

**接口地址**: (待补充)
- 私有云: `${PRIVATE_PRODUCT_API_BASE}/...`
- 公有云: `${PUBLIC_PRODUCT_API_BASE}/...`

**请求方式**: (待补充)

**请求参数**: (待补充)
```json
{
    // TODO: 请提供产品编码 API 请求参数
}
```

**响应格式**: (待补充)
```json
{
    // TODO: 请提供产品编码 API 响应格式
}
```

#### 3.3.2 工具定义

```python
# mini_agent/tools/business/product_tool.py

class ProductCodeTool(Tool):
    name = "get_product_code"
    description = "将产品名称或别名转换为标准产品编码"
    
    parameters = {
        "type": "object",
        "properties": {
            "product_name": {
                "type": "string",
                "description": "产品名称或别名"
            }
        },
        "required": ["product_name"]
    }
```

---

### 3.4 其他小工具 (待补充)

| 工具名称 | 功能描述 | API 地址 | 状态 |
|---------|---------|---------|------|
| <!-- TODO --> | <!-- TODO --> | <!-- TODO --> | 待补充 |
| | | | |

---

## 四、MCP 服务配置

### 4.1 已有 MCP 服务列表

| 服务名称 | 功能描述 | 端点地址 | 提供的工具 |
|---------|---------|---------|-----------|
| <!-- TODO --> | <!-- TODO --> | <!-- TODO --> | <!-- TODO --> |
| | | | |

### 4.2 MCP 配置模板

```json
// mini_agent/config/mcp-business.json
{
    "mcpServers": {
        "service_name_1": {
            "description": "服务描述",
            "type": "streamable_http",
            "url": "${MCP_SERVICE_1_URL}",
            "headers": {
                "Authorization": "Bearer ${MCP_SERVICE_1_TOKEN}"
            },
            "connect_timeout": 10.0,
            "execute_timeout": 60.0
        }
        // TODO: 补充更多服务
    }
}
```

---

## 五、业务引导 Skill

### 5.1 Skill 定义

```markdown
<!-- mini_agent/skills/business-guide/SKILL.md -->
---
name: business-guide
description: 公司业务场景处理指南，帮助理解何时使用意图识别、RAG查询、产品编码等工具
allowed-tools:
  - recognize_intent
  - query_knowledge_base
  - get_product_code
---

# 业务处理指南

## 适用场景
<!-- TODO: 描述业务场景 -->

## 工具使用流程

### 1. 意图识别
当用户提出业务相关问题时，首先调用 `recognize_intent` 识别意图：
<!-- TODO: 补充具体指引 -->

### 2. RAG 知识查询
根据识别的意图，调用 `query_knowledge_base` 查询相关信息：
<!-- TODO: 补充具体指引 -->

### 3. 产品编码转换
当涉及具体产品时，使用 `get_product_code` 获取标准编码：
<!-- TODO: 补充具体指引 -->

## 示例对话
<!-- TODO: 提供典型对话示例 -->
```

---

## 六、配置项清单

### 6.1 环境变量

| 变量名 | 描述 | 私有云值 | 公有云值 |
|-------|------|---------|---------|
| `MINIAGENT_ENV` | 环境标识 | `private` | `public` |
| `RAG_API_BASE` | RAG 服务地址 | <!-- TODO --> | <!-- TODO --> |
| `RAG_API_KEY` | RAG 服务密钥 | <!-- TODO --> | <!-- TODO --> |
| `PRODUCT_API_BASE` | 产品服务地址 | <!-- TODO --> | <!-- TODO --> |
| `PRODUCT_API_KEY` | 产品服务密钥 | <!-- TODO --> | <!-- TODO --> |
| `MCP_SERVICE_1_URL` | MCP 服务1地址 | <!-- TODO --> | <!-- TODO --> |
| <!-- TODO --> | <!-- TODO --> | <!-- TODO --> | <!-- TODO --> |

### 6.2 配置文件扩展

```yaml
# config.yaml 新增业务配置段
business:
  intent:
    prompt_path: "prompts/intent_system.md"  # 意图识别提示词路径
  rag:
    api_base: "${RAG_API_BASE}"
    api_key: "${RAG_API_KEY}"
    timeout: 30
  product:
    api_base: "${PRODUCT_API_BASE}"
    api_key: "${PRODUCT_API_KEY}"
    timeout: 10
```

---

## 七、HTTP API 设计

### 7.1 端点列表

| 方法 | 路径 | 描述 |
|-----|------|------|
| `GET` | `/health` | 健康检查（存活探针） |
| `GET` | `/ready` | 就绪检查（就绪探针） |
| `POST` | `/api/v1/sessions` | 创建会话 |
| `DELETE` | `/api/v1/sessions/{session_id}` | 删除会话 |
| `POST` | `/api/v1/prompt` | 发送提示（SSE 流式响应） |

### 7.2 SSE 事件类型

| 事件类型 | 描述 | 数据结构 |
|---------|------|---------|
| `thinking` | Agent 思考过程 | `{"content": "..."}` |
| `tool_call_start` | 开始工具调用 | `{"tool": "...", "args": {...}}` |
| `tool_call_end` | 工具调用完成 | `{"tool": "...", "result": "..."}` |
| `content_delta` | 内容增量输出 | `{"delta": "..."}` |
| `complete` | 响应完成 | `{"content": "..."}` |
| `error` | 错误 | `{"error": "...", "code": "..."}` |

---

## 八、实施步骤与进度

| 阶段 | 任务 | 负责人 | 状态 | 备注 |
|-----|------|-------|------|------|
| 1 | 补充业务接口信息 | - | ⏳ 进行中 | |
| 2 | 开发 IntentTool | - | ⏸️ 待开始 | 依赖提示词 |
| 3 | 开发 RagQueryTool | - | ⏸️ 待开始 | 依赖 API 信息 |
| 4 | 开发 ProductCodeTool | - | ⏸️ 待开始 | 依赖 API 信息 |
| 5 | 配置 MCP 服务 | - | ⏸️ 待开始 | 依赖服务列表 |
| 6 | 创建业务 Skill | - | ⏸️ 待开始 | |
| 7 | 开发 HTTP 服务层 | - | ⏸️ 待开始 | |
| 8 | 集成测试 | - | ⏸️ 待开始 | |
| 9 | 部署配置 | - | ⏸️ 待开始 | |

---

## 九、待补充信息清单

请按以下顺序提供信息：

- [ ] **意图识别提示词** (System Prompt + User Prompt 模板)
- [ ] **意图分类列表** (意图代码、名称、描述)
- [ ] **RAG API 接口文档** (地址、参数、响应格式)
- [ ] **产品编码 API 接口文档** (地址、参数、响应格式)
- [ ] **其他小工具 API** (如有)
- [ ] **MCP 服务列表** (服务名、端点、提供的工具)
- [ ] **环境配置** (私有云/公有云各服务地址)
- [ ] **业务场景示例** (用于 Skill 编写)

---

*文档持续更新中，请逐步提供上述信息以完善开发细节。*
