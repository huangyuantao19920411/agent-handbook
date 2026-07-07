# 01 - 什么是 Agent（智能体）？

## 一句话定义

> **Agent = 能自主感知环境、做出决策、执行动作的大模型应用。**

普通 Chatbot 只会「聊天」，Agent 能「做事」。

---

## 核心公式

```mermaid
graph LR
    M[Model<br/>大模型] --> A[Agent]
    H[Harness<br/>工程层] --> A
    style A fill:#4a90d9,color:#fff
    style M fill:#e8f4fd
    style H fill:#e8f4fd
```

**Model + Harness = Agent**

| 组件 | 是什么 | 举例 |
|------|--------|------|
| **Model** | 大语言模型，负责「思考」 | DeepSeek-V3、GPT-4、Claude |
| **Harness** | 模型之外让 Agent 跑起来的全部工程 | Agent Loop、工具调用、记忆、沙箱 |
| **Agent** | 两者组合后的完整智能体 | 代码助手、研究 Agent、客服机器人 |

---

## Agent vs Chatbot

```mermaid
graph TB
    subgraph Chatbot["普通 Chatbot"]
        U1[用户提问] --> LLM1[LLM 生成回答]
        LLM1 --> R1[返回文本]
    end

    subgraph Agent["Agent 智能体"]
        U2[用户任务] --> LOOP[Agent Loop]
        LOOP --> THINK[LLM 推理]
        THINK -->|需要工具| TOOL[调用工具]
        TOOL --> OBS[获取结果]
        OBS --> LOOP
        THINK -->|任务完成| DONE[返回结果]
    end
```

| | Chatbot | Agent |
|---|---------|-------|
| 交互模式 | 单轮问答 | 多轮自主循环 |
| 能力边界 | 只能生成文本 | 能调用工具、执行代码、操作外部系统 |
| 典型场景 | 聊天、问答 | 写代码、做研究、自动化流程 |

---

## Agent Loop（智能体循环）

Agent 的核心运行机制是 **Loop（循环）**：

```mermaid
flowchart TD
    START([用户输入任务]) --> REASON[LLM 推理<br/>我该怎么做？]
    REASON --> DECIDE{需要工具吗？}
    DECIDE -->|是| ACT[执行工具<br/>搜索 / 写代码 / 调 API]
    ACT --> OBSERVE[获取工具返回结果]
    OBSERVE --> REASON
    DECIDE -->|否，任务完成| ANSWER[返回最终答案]
    ANSWER --> END([结束])
```

这个循环叫做 **ReAct**（Reasoning + Acting）：

1. **Reason** — LLM 分析当前状态，决定下一步
2. **Act** — 调用工具执行动作
3. **Observe** — 把工具结果喂回 LLM
4. 重复，直到任务完成

---

## Harness 包含什么？

```mermaid
graph TB
    subgraph Harness["Harness 工程层"]
        LOOP[Agent Loop<br/>循环调度]
        TOOL[Tool Use<br/>工具调用]
        CTX[Context Engineering<br/>上下文管理]
        MEM[Memory<br/>短期 + 长期记忆]
        PLAN[Planning<br/>任务规划]
        MULTI[Multi-Agent<br/>多智能体协作]
        TRACE[Trace<br/>执行轨迹记录]
        SANDBOX[Sandbox<br/>安全沙箱]
    end

    LOOP --> TOOL
    LOOP --> CTX
    LOOP --> MEM
    LOOP --> PLAN
    PLAN --> MULTI
    LOOP --> TRACE
    TOOL --> SANDBOX
```

每一层解决一个具体问题：

| 模块 | 解决什么问题 |
|------|-------------|
| **Agent Loop** | 怎么让 LLM 反复推理直到任务完成？ |
| **Tool Use** | 怎么让 LLM 调用外部 API、执行代码？ |
| **Context Engineering** | 上下文窗口有限，怎么管理长对话？ |
| **Memory** | 跨会话怎么记住用户偏好和历史？ |
| **Planning** | 复杂任务怎么拆解成步骤？ |
| **Multi-Agent** | 多个 Agent 怎么协作？ |
| **Trace** | 怎么记录和回放 Agent 的每一步？ |
| **Sandbox** | 怎么安全执行 AI 生成的不可信代码？ |

---

## 一个具体例子：代码 Agent

用户说：「帮我列出当前目录的文件，然后写一个 README」

```mermaid
sequenceDiagram
    participant U as 用户
    participant H as Harness
    participant LLM as DeepSeek
    participant FS as 文件系统工具

    U->>H: 列出目录并写 README
    H->>LLM: 用户任务 + 可用工具列表
    LLM->>H: 调用 list_dir(".")
    H->>FS: list_dir(".")
    FS->>H: ["src/", "Cargo.toml", ...]
    H->>LLM: 工具返回: 文件列表
    LLM->>H: 调用 write_file("README.md", "...")
    H->>FS: write_file(...)
    FS->>H: 写入成功
    H->>LLM: 工具返回: 成功
    LLM->>H: 任务完成，README 已创建
    H->>U: README.md 已生成
```

---

## 常见 Agent 产品

| 产品 | 类型 | 特点 |
|------|------|------|
| **Claude Code** | 代码 Agent | 终端内自主写代码、跑测试 |
| **Cursor** | 代码 Agent | IDE 内 Vibe Coding |
| **Manus** | 通用 Agent | 浏览器自动化、多步任务 |
| **OpenClaw** | 生活助理 | 日程、邮件、个人事务 |

---

## 关键术语速查

| 术语 | 含义 |
|------|------|
| **Agent Loop** | 推理 → 行动 → 观察 的循环 |
| **Tool Use** | LLM 调用外部工具的能力 |
| **Function Calling** | OpenAI 格式的工具调用 API |
| **MCP** | 标准化的工具协议（见下一章） |
| **Harness** | Model 之外的全部工程层 |
| **Context Engineering** | 上下文窗口的管理与优化 |
| **ReAct** | Reason + Act 的 Agent 推理模式 |

---

[下一章：什么是 MCP？ →](02-what-is-mcp.md)
