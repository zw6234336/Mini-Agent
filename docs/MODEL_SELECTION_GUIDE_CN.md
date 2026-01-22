# 中国区 Agent 系统大模型选型指南 (2025-2026版)

> **基于互联网评测、开发者社区反馈及实战表现的综合评估**

本文档旨在为面向中国用户的 Agent 系统提供大模型选型建议。评估依据包括逻辑推理能力、指令遵循（Tool Calling）、编程能力、长文本支持及成本效益。

## 1. 选型总览 (Executive Summary)

截至 2026 年初，国产大模型在 Agent 领域已形成“一超多强”的格局。对于构建 Agent 系统，我们推荐采用 **混合模型架构 (Hybrid Model Architecture)**，即根据 Agent 的不同思维阶段调用最擅长的模型，以达到效果与成本的最优平衡。

| Agent 阶段 | 核心需求 | 首选推荐 | 备选方案 | 理由 |
| :--- | :--- | :--- | :--- | :--- |
| **核心规划 (Brain)** | 复杂逻辑、长程推理、任务拆解 | **DeepSeek-V3** | Qwen-2.5-Max | DeepSeek 在推理深度和成本上具有极强统治力；Qwen 综合能力稳健。 |
| **工具调用 (Hands)** | 严格 JSON 格式、参数提取、编程 | **Qwen-2.5-Coder** | DeepSeek-V3 | Qwen 在 Function Calling 和代码生成上的稳定性经过了大量验证。 |
| **长文档/记忆 (Memory)** | 超长上下文、大海捞针 | **Kimi (Moonshot)** | GLM-4-Long | Kimi 在 200k+ 长文本的无损召回上依然保持领先口碑。 |
| **高速/闲聊 (Mouth)** | 极速响应、高并发、低成本 | **Doubao-Pro (豆包)** | DeepSeek-Lite | 字节豆包模型在从以 Tokens 计费转为并发计费后，性价比极高。 |

---

## 2. 详细评测与分析

### 2.1 核心大脑 (Pattern 1: The Planner)
**场景**: 用户输入“帮我策划一次去日本的旅行，并预定机票酒店”。Agent 需要将此模糊指令拆解为“搜索攻略”、“查询机票”、“比价”、“生成日程表”等子任务。

*   **DeepSeek-V3 (深度求索)**
    *   **社区评价**: "开源之光"、"价格屠夫"。DeepSeek-V3 在逻辑推理 (Math/Reasoning) 榜单上常年霸榜，甚至超越部分闭源 GPT-4o 模型。
    *   **Agent 优势**: 极强的 Chain-of-Thought (思维链) 能力，能够自我反思修正错误，非常适合做 Agent 的 System 2 (慢思考) 系统。
    *   **成本**: 极低，几乎是其他同级别模型的 1/10。

*   **Qwen-2.5-Max (通义千问)**
    *   **社区评价**: "最懂中文的通用模型"。阿里通义系列在中文语境理解和指令遵循上表现极其稳定。
    *   **Agent 优势**: 在多轮对话中不易丢失指令（Instruction Following），适合处理结构化非常复杂的 SOP。

### 2.2 工具执行 (Pattern 2: The Executor)
**场景**: Generate SQL, Write Python Script, Parse JSON for API.

*   **Qwen-2.5-Coder (通义千问代码版)**
    *   **开发者反馈**: 目前国产最强代码模型。在 HumanEval 和 MBPP 榜单上表现优异。
    *   **Why for Agent**: Agent 调用外部工具本质上是“代码生成”或“结构化数据生成”。Qwen-Coder 极少出现 JSON 格式错误，对参数类型的敏感度极高。

*   **Yi-Lightning (零一万物)**
    *   **特点**: 数学与代码能力强劲，推理速度快。
    *   **适用**: 如果你的 Agent 需要进行大量的数据分析或科学计算，Yi 是个不错的选择。

### 2.3 记忆与检索 (Pattern 3: The Reader)
**场景**: “读取这本 500 页的保险条款，告诉我如果我在滑雪时受伤能否理赔”。

*   **Kimi (Moonshot AI)**
    *   **核心优势**: 长文本处理的“守门员”。业界公认的 128k/200k 长窗口真实可用性最高的模型之一，"大海捞针" (Needle In A Haystack) 评测近乎满分。
    *   **Agent 场景**: 作为 RAG (检索增强生成) 的重排序 (Rerank) 或最终阅读器。

*   **GLM-4-Long (智谱)**
    *   **特点**: 1M (100万) 上下文支持，适合超大规模知识库的直接阅读。

### 2.4 快速交互 (Pattern 4: The Chatbot)
**场景**: 各种简单的问候、意图澄清、或是生成给用户的最终回复文案。

*   **Doubao-Pro (字节跳动)**
    *   **特点**: 响应速度极快 (TPOT 低)，且音色合成 (TTS) 能力与文本生成结合得很好。
    *   **性价比**: 针对高并发 C 端应用优化，价格极具竞争力。

---

## 3. 推荐架构配置 (Recommended Configuration)

针对本项目 (Mini-Agent)，我们要构建一个面向开发者的 coding assistant 或行业助手，推荐如下配置：

### 配置方案 A：极致性价比 (The "DeepSeek" Stack)
适合初创团队或个人开发者，成本极低，能力顶尖。

*   **Planner**: `DeepSeek-V3`
*   **Executor**: `DeepSeek-V3` (v3 代码能力已足够强)
*   **Summarizer**: `DeepSeek-V3`
*   **Cost**: < ¥10 / 百万 Tokens (混合大量输入输出)

### 配置方案 B：企业级高容错 (The "Enterprise" Stack)
适合对稳定性要求极高的金融/保险场景。

*   **Planner**: `Qwen-2.5-Max` (逻辑稳)
*   **Executor**: `Qwen-2.5-Coder` (零出错)
*   **Long Context**: `Moonshot-v1-128k` (读取合同)
*   **Cost**: 适中，但稳定性有保障。

## 4. 接入提示 (Implementation Tips)

在 `mini_agent/config/config.yaml` 或 `mcp.json` 中，建议支持多模型配置：

```yaml
llm:
  # 默认模型 (主要负责逻辑)
  default: 
    provider: "openai" # 兼容协议
    base_url: "https://api.deepseek.com"
    api_key: "${DEEPSEEK_API_KEY}"
    model: "deepseek-chat"
  
  # 代码专用模型 (可选配置)
  coder:
    provider: "openai"
    base_url: "https://dashscope.aliyuncs.com/compatible-mode/v1"
    api_key: "${QWEN_API_KEY}"
    model: "qwen-2.5-coder-32b-instruct"
```

## 5. 结论

**DeepSeek-V3 是目前中国区 Agent 开发的首选全能模型**，其低廉的价格和惊人的智力水平使其成为 Agent "思考" 过程的最佳载体。对于特定的代码生成或超长文本任务，可以引入 **Qwen-Coder** 和 **Kimi** 作为专项能力的补充。
