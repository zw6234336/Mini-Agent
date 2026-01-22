# 行业系统历史接口 MCP 对接设计方案

本文档旨在提供一套标准化的技术方案，将现有的行业系统（如保险核心系统、CRM、业绩系统等）通过 **MCP (Model Context Protocol)** 封装为 Agent 可直接调用的标准化服务。

## 1. 背景与目标

当前业务环境中存在大量成熟的历史项目接口（Legacy APIs），包括但不限于：
*   **保单管理系统**: 保单查询、条款查询、理赔进度。
*   **业绩管理系统**: 个人业绩、团队报表、佣金查询。
*   **客户管理系统 (CRM)**: 客户档案、跟进记录、画像分析。

**目标**: 避免通过硬编码方式将这些接口逐一集成到 Agent 中，而是通过构建 **MCP Server** 中间层，实现接口的标准化暴露。Agent 无需关心底层 HTTP/SOAP 协议细节，只需通过自然语言意图调用 MCP Tools。

---

## 2. 总体架构设计

架构采用 **适配器模式 (Adapter Pattern)**，MCP Server 作为 Agent 与历史 APIs 之间的翻译层。

```mermaid
graph LR
    subgraph "Mini-Agent (Client)"
        Brain[LLM 核心]
        MCPClient[MCP 客户端]
    end

    subgraph "MCP Gateway / Server"
        Router[请求路由]
        
        subgraph "Policy Adapter"
            T_Policy[Tool: 保单查询]
            R_Policy[Resource: 保单文档]
        end
        
        subgraph "CRM Adapter"
            T_CRM[Tool: 客户管理]
        end
        
        subgraph "Perf Adapter"
            T_Perf[Tool: 业绩查询]
        end
    end

    subgraph "Legacy Systems"
        CoreSys[保险核心系统 (SOAP/XML)]
        CRMSys[CRM 系统 (REST/JSON)]
        DataSys[数据仓库 (SQL/JDBC)]
    end

    Brain -->|Tool Call| MCPClient
    MCPClient -->|JSON-RPC| Router
    
    T_Policy -->|HTTP| CoreSys
    T_CRM -->|REST| CRMSys
    T_Perf -->|SQL| DataSys
```

---

## 3. 接口与工具设计 (Schema Design)

我们将业务能力划分为三个核心 MCP Server 模块（或合并为一个 Server 的不同 Namespaces）。

### 3.1 保单服务 (Policy Service)

**Namespace**: `insurance.policy`

| 工具名称 (Tool Name) | 描述 (Description) | 关键参数 (Args) | 对应原接口 |
| :--- | :--- | :--- | :--- |
| `get_policy_detail` | 根据保单号查询保单详细信息，包含险种、保额、生效日等。 | `policy_id` (str, required) | `/api/v1/policy/detail` |
| `list_customer_policies` | 查询指定客户名下的所有有效保单列表。 | `customer_id` (str, required), `status` (enum: active, lapsed) | `/api/v1/policy/list` |
| `check_claim_status` | 查询理赔案件的当前审核进度。 | `case_id` (str) | `/api/claims/status` |

**工具定义示例 (JSON Schema)**:
```json
{
  "name": "get_policy_detail",
  "description": "Retrieve comprehensive details of an insurance policy by its ID.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "policy_id": { "type": "string", "description": "The unique policy number (e.g., POL-2024-001)" }
    },
    "required": ["policy_id"]
  }
}
```

### 3.2 客户管理 (CRM Service)

**Namespace**: `insurance.crm`

| 工具名称 (Tool Name) | 描述 (Description) | 关键参数 (Args) | 对应原接口 |
| :--- | :--- | :--- | :--- |
| `search_customer` | 通过姓名、手机号或证件号模糊搜索客户。 | `keyword` (str), `search_type` (default: 'fuzzy') | `/api/crm/search` |
| `get_customer_persona` | 获取客户画像，包含标签、家庭结构、虽然购买力评级。 | `customer_id` (str) | `/api/crm/persona` |
| `add_follow_up_record` | 添加一条客户跟进记录（访谈纪要）。 | `customer_id` (str), `content` (str), `date` (str) | `/api/crm/followup/add` |

### 3.3 业绩查询 (Performance Service)

**Namespace**: `insurance.performance`

| 工具名称 (Tool Name) | 描述 (Description) | 关键参数 (Args) | 对应原接口 |
| :--- | :--- | :--- | :--- |
| `get_my_kpi` | 查询当前登录代理人的核心KPI（FYC, 件数, 达成率）。 | `month` (str, YYYY-MM), `indicator` (optional list) | `/api/perf/personal` |
| `get_team_report` | (主管权限) 查询直辖团队的业绩报表。 | `team_id` (str), `period` (str) | `/api/perf/team/summary` |

---

## 4. MCP 工具 vs 技能 (MCP Tools vs Skills)

在实现过程中，一个常见的问题是：**“每一个 MCP 接口调用都需要包装成 Skill 吗？”** 答案是否定的。必须区分 **原子能力 (Atomic Capability)** 与 **复合流程 (Composite Workflow)**。

### 4.1 决策矩阵 (Decision Matrix)

| 场景特点 | 推荐方式 | 解释 |
| :--- | :--- | :--- |
| **单一查询/操作** | **直接调用 MCP** | 例如“查询保单详情”。这是一个原子操作，LLM 可以直接理解并调用 `get_policy_detail`。 |
| **复杂业务流程 (SOP)** | **封装为 Skill** | 例如“为客户制定家庭保障计划”。这涉及查询已有保单、分析缺口、生成建议书等多个步骤。 |
| **需严格遵守规范** | **封装为 Skill** | 例如“处理理赔审核”，必须按顺序检查 A、B、C 三个条件，不能跳过。 |
| **跨系统联动** | **封装为 Skill** | 例如“客户迁居”，需要同时更新 CRM 地址和保单邮寄地址。 |

### 4.2 示例对比

#### 场景 A: 简单查询 (直接使用 MCP)
*   **用户**: "查一下张三今年的业绩。"
*   **Agent**: 识别意图 -> 直接调用 `insurance.performance.get_my_kpi(name="张三")` -> 返回结果。
*   **结论**: 无需 Skill，LLM 足够智能来处理。

#### 场景 B: 复杂业务 (使用 Skill)
*   **用户**: "帮我对张三进行一次年度保单检视。"
*   **Agent**: 识别意图 -> 调用 `skills/annual_policy_review`。
*   **Skill 内部逻辑 (SKILL.md)**:
    1.  调用 `insurance.crm.search_customer` 获取 ID。
    2.  调用 `insurance.policy.list_customer_policies` 获取所有保单。
    3.  调用 `insurance.crm.get_customer_persona` 获取家庭结构。
    4.  让 LLM 对比保单覆盖范围与家庭需求。
    5.  生成检视报告并发送邮件。
*   **结论**: 必须封装为 Skill，否则 Agent 容易在多步推理中迷失或遗漏步骤。

---

## 5. 数据安全与鉴权 (Security & Auth)

Agent 调用后端接口时，必须确保数据安全和权限合规。

### 5.1 鉴权透传方案
Agent 本身作为“代理人助手”，继承当前操作用户的身份。

1.  **Environment Variables**: 在启动 MCP Server 时，注入系统级服务账号（如 `CRM_API_TOKEN`）。
2.  **Request Context**: (高级) 如果 Agent 支持多用户，需在 MCP 请求头中携带用户的 `Auth-Token`，MCP Server 解析 Token 并以该用户身份调用后端接口。

### 5.2 数据脱敏
**原则**: MCP Server 必须在返回数据给 Agent 之前进行脱敏处理。
*   **身份证/手机号**: 自动掩码处理 (e.g., `138****0000`).
*   **敏感字段**: 过滤掉非业务必要的底层字段（如数据库主键、内部版本号）。

---

## 6. 实施路线图

### 第一阶段：搭建 MCP Server 骨架
推荐使用 `python-mcp` 或 `mcp-node-sdk` 搭建独立服务。

**代码骨架 (Python FastMCP)**:
```python
from mcp.server.fastmcp import FastMCP
import httpx

# 初始化 Server
mcp = FastMCP("Insurance-Integrator")

# 配置后端地址
LEGACY_API_BASE = "http://internal-api-gateway.corp/v1"

@mcp.tool()
async def get_policy_detail(policy_id: str) -> str:
    """根据保单号查询保单详细信息"""
    async with httpx.AsyncClient() as client:
        # 这里进行接口转换
        resp = await client.get(f"{LEGACY_API_BASE}/policy/{policy_id}")
        data = resp.json()
        
        # 数据清洗与格式化
        return f"保单 {data['pol_no']} (险种: {data['product_name']}) 状态: {data['status']}"

if __name__ == "__main__":
    mcp.run()
```

### 第二阶段：Agent 配置集成
在 `mini_agent/config/mcp.json` 中添加该 Server 的连接配置：

```json
{
  "mcpServers": {
    "legacy-system": {
      "command": "python",
      "args": ["/path/to/insurance_mcp_server.py"],
      "env": {
        "API_KEY": "xxx-internal-key"
      }
    }
  }
}
```

### 第三阶段：测试与调优
1.  **单元测试**: 验证 MCP Server 能正确解析参数并调用 Mock 的后端接口。
2.  **Agent 联调**: 使用自然语言指令（如“帮我查一下张三名下所有的有效保单，看看哪张快过保了”）测试 Agent 是否能自主拆解任务并调用对应的工具。
