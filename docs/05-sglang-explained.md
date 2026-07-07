# 05 - SGLang 详解

## 一句话定义

> **SGLang 是一个高性能大模型推理框架，用 RadixAttention 和前端编程语言，让 LLM 推理更快、更灵活。**

与 vLLM 解决同样的问题（高效 LLM serving），但技术路线不同，各有优势。

---

## SGLang vs vLLM 定位

```mermaid
graph TB
    subgraph Problem["共同目标：高效 LLM 推理"]
        P1[高吞吐]
        P2[低延迟]
        P3[高 GPU 利用率]
    end

    subgraph vLLM["vLLM"]
        V1[PagedAttention]
        V2[Continuous Batching]
        V3[OpenAI API]
    end

    subgraph SGLang["SGLang"]
        S1[RadixAttention]
        S2[前端编程语言]
        S3[结构化输出优化]
    end

    Problem --> vLLM
    Problem --> SGLang
```

| | vLLM | SGLang |
|---|------|--------|
| **出品** | UC Berkeley (vLLM team) | LMSYS (Chatbot Arena 团队) |
| **核心创新** | PagedAttention | RadixAttention + SGLang 语言 |
| **强项** | 通用高吞吐 serving | 复杂 Agent 工作流、前缀共享 |
| **API** | OpenAI 兼容 | OpenAI 兼容 + 原生前端语言 |
| **适合场景** | 通用 API 服务 | Agent 多轮对话、批量推理 |

---

## 核心创新 1：RadixAttention

### 问题：Agent 场景大量重复前缀

Agent 应用中，很多请求共享相同的前缀：

```mermaid
graph TB
    subgraph Requests["Agent 请求的 System Prompt"]
        R1["System: 你是代码助手...<br/>User: 列出文件"]
        R2["System: 你是代码助手...<br/>User: 写个测试"]
        R3["System: 你是代码助手...<br/>User: 解释这段代码"]
    end

    PREFIX["共享前缀<br/>System: 你是代码助手...<br/>（完全相同）"]
    PREFIX --> R1
    PREFIX --> R2
    PREFIX --> R3
```

传统 KV Cache：每个请求独立计算和存储前缀 → **重复计算浪费 GPU**。

### 解决：基数树（Radix Tree）共享前缀

```mermaid
graph TB
    subgraph RadixTree["RadixAttention 基数树"]
        ROOT[Root]
        ROOT --> SYS["System: 你是代码助手..."]
        SYS --> U1["User: 列出文件"]
        SYS --> U2["User: 写个测试"]
        SYS --> U3["User: 解释这段代码"]
    end

    subgraph KVShared["共享 KV Cache"]
        KV_SYS["前缀 KV Cache<br/>（只算一次）"]
        KV_SYS --> KV1[请求1 独有部分]
        KV_SYS --> KV2[请求2 独有部分]
        KV_SYS --> KV3[请求3 独有部分]
    end
```

效果：
- 相同前缀只计算 **一次** KV Cache
- Agent 多轮对话场景下吞吐量提升 **2-5 倍**
- 特别适合 System Prompt 长、多用户共享的场景

---

## 核心创新 2：SGLang 前端语言

SGLang 提供了一种专门描述 LLM 应用逻辑的编程语言：

```python
import sglang as sgl

@sgl.function
def code_assistant(s, task):
  s += sgl.system("You are a helpful coding assistant.")
  s += sgl.user(f"Task: {task}")
  s += sgl.assistant(sgl.gen("response", max_tokens=1024))

# 批量执行（自动共享前缀）
results = code_assistant.run_batch([
    {"task": "list files"},
    {"task": "write tests"},
    {"task": "explain code"},
])
```

```mermaid
graph LR
    subgraph SGLang_Lang["SGLang 前端语言"]
        CODE["Python 装饰器<br/>@sgl.function"]
        CODE --> IR[中间表示 IR]
        IR --> OPT[自动优化<br/>前缀共享 / 并行]
        OPT --> ENGINE[推理引擎]
    end
```

对比普通 API 调用：

| | 普通 API 调用 | SGLang 前端语言 |
|---|-------------|----------------|
| 前缀共享 | 手动实现 | 自动（RadixAttention） |
| 批量推理 | 手动 batch | `run_batch()` 自动优化 |
| 多步逻辑 | 多次 API 调用 | 一个 function 描述完整流程 |
| 结构化输出 | 手动解析 | 原生支持 JSON/regex 约束 |

---

## SGLang 架构

```mermaid
graph TB
    subgraph Frontend["前端层"]
        PY[Python API<br/>@sgl.function]
        OAI[OpenAI 兼容 API]
    end

    subgraph Engine["SGLang Runtime"]
        RADIX[RadixAttention<br/>前缀树 KV Cache]
        SCHED[调度器]
        EXEC[Model Executor]
    end

    subgraph Hardware["硬件"]
        GPU[GPU 集群]
    end

    PY --> RADIX
    OAI --> SCHED
    RADIX --> SCHED
    SCHED --> EXEC
    EXEC --> GPU
```

---

## 请求处理流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant S as SGLang Server
    participant RT as RadixAttention
    participant GPU as GPU

    C->>S: 请求（含 system prompt + user message）
    S->>RT: 查找前缀树
    RT->>RT: 命中共享前缀？<br/>复用已有 KV Cache
    RT->>GPU: 只计算新增部分（Prefix Cache Miss 部分）
    GPU->>S: 生成 token
    S->>C: 流式返回

    Note over RT: 新请求到来
    C->>S: 另一个请求（相同 system prompt）
    S->>RT: 查找前缀树
    RT->>RT: 命中！直接复用 KV Cache
    RT->>GPU: 只计算 user message 部分
```

---

## 选型建议

```mermaid
graph TD
    START{你的场景？} -->|通用 API 服务<br/>OpenAI 兼容| VLLM[vLLM]
    START -->|Agent 多轮对话<br/>大量共享前缀| SGLANG[SGLang]
    START -->|复杂 LLM 工作流<br/>多步推理| SGLANG
    START -->|最大生态兼容<br/>最多部署案例| VLLM[vLLM]
    START -->|前缀共享是瓶颈<br/>批量 Agent 任务| SGLANG
```

| 场景 | 推荐 | 原因 |
|------|------|------|
| 通用 Chatbot API | vLLM | 生态成熟、部署案例多 |
| Agent 平台（多用户共享 System Prompt） | SGLang | RadixAttention 前缀共享 |
| 复杂多步 LLM 工作流 | SGLang | 前端语言描述逻辑 |
| 快速上线、OpenAI 兼容 | 两者都行 | 都支持 OpenAI API |
| 极致吞吐量 benchmark | 看具体场景 | 建议都测试 |

---

## 与 Agent Harness 的关系

```mermaid
graph TB
    USER[用户] --> HARNESS[Agent Harness<br/>Loop / Tools / MCP / Sandbox]
    HARNESS -->|OpenAI API| INFERENCE{推理引擎}
    INFERENCE --> VLLM[vLLM]
    INFERENCE --> SGLANG[SGLang]
    VLLM --> GPU[GPU 集群]
    SGLANG --> GPU
```

Agent Harness 通过标准 OpenAI API 调用推理引擎，**底层用 vLLM 还是 SGLang 可以灵活切换**。

---

## 关键术语速查

| 术语 | 含义 |
|------|------|
| **SGLang** | 高性能 LLM 推理框架（LMSYS 出品） |
| **RadixAttention** | 基于基数树的前缀 KV Cache 共享 |
| **Radix Tree** | 基数树，高效存储和查找共享前缀 |
| **Prefix Caching** | 前缀缓存，相同 prompt 前缀只算一次 |
| **SGLang 语言** | 描述 LLM 应用逻辑的 Python DSL |
| **run_batch** | 批量执行，自动优化前缀共享和并行 |

---

[← 上一章：vLLM](04-vllm-explained.md) | [返回目录 →](../README.md)
