# 04 - vLLM 详解

## 一句话定义

> **vLLM 是一个高性能大模型推理引擎，通过 PagedAttention 和连续批处理，让 LLM 在生产环境中高效服务大量并发请求。**

如果说 PyTorch 是「训练框架」，vLLM 就是「推理 serving 框架」—— 专门解决「怎么让大模型又快又省地跑在生产环境」。

---

## 为什么需要推理框架？

直接用 HuggingFace `model.generate()` 做生产服务的问题：

```mermaid
graph TB
    subgraph Problems["直接推理的问题"]
        P1[KV Cache 内存浪费严重]
        P2[无法高效批处理<br/>请求长短不一]
        P3[GPU 利用率低<br/>大量空闲等待]
        P4[不支持高并发]
    end
```

vLLM 解决了这些核心问题。

---

## vLLM 架构

```mermaid
graph TB
    subgraph Client["客户端"]
        API[OpenAI 兼容 API]
    end

    subgraph vLLM["vLLM 引擎"]
        SCHED[调度器<br/>Scheduler]
        BATCH[连续批处理<br/>Continuous Batching]
        PA[PagedAttention<br/>KV Cache 管理]
        ENGINE[推理引擎<br/>Model Executor]
    end

    subgraph GPU["GPU"]
        MODEL[模型权重]
        KV[KV Cache 池]
    end

    API --> SCHED
    SCHED --> BATCH
    BATCH --> PA
    PA --> ENGINE
    ENGINE --> MODEL
    ENGINE --> KV
```

---

## 核心创新 1：PagedAttention

### 问题：KV Cache 内存浪费

LLM 推理时，每生成一个 token 都需要缓存 Key/Value 向量（KV Cache）。传统方式是连续分配：

```mermaid
graph TB
    subgraph Traditional["传统方式：连续内存"]
        direction LR
        REQ1["Request 1<br/>████████░░░░<br/>已用 8 / 分配 12"]
        REQ2["Request 2<br/>████░░░░░░░░<br/>已用 4 / 分配 12"]
        REQ3["Request 3<br/>██████████░░<br/>已用 10 / 分配 12"]
    end
```

问题：每个请求预分配最大长度的内存，实际使用往往不到一半 → **内存浪费 50-80%**。

### 解决：分页式 KV Cache

借鉴操作系统虚拟内存的分页思想：

```mermaid
graph TB
    subgraph Paged["PagedAttention：分页内存"]
        direction TB
        REQ1B["Request 1"] --> P1[Page 1]
        REQ1B --> P2[Page 2]
        REQ2B["Request 2"] --> P3[Page 3]
        REQ3B["Request 3"] --> P4[Page 4]
        REQ3B --> P5[Page 5]

        subgraph Pool["物理 KV Cache 池"]
            P1
            P2
            P3
            P4
            P5
            FREE[空闲 Page]
        end
    end
```

效果：
- 内存浪费从 **50-80%** 降到 **< 4%**
- 同样 GPU 内存可以服务 **2-4 倍** 更多请求

---

## 核心创新 2：Continuous Batching

### 问题：静态批处理效率低

```mermaid
gantt
    title 静态批处理（等待最慢请求）
    dateFormat X
    axisFormat %s

    section Request A (短)
    生成 :a1, 0, 3

    section Request B (长)
    生成 :b1, 0, 8

    section Request C (中)
    生成 :c1, 0, 5

    section GPU 等待
    空闲等待 B 完成 :crit, 3, 8
```

Request A 3 秒就完成了，但 GPU 要等最慢的 Request B（8 秒）才能处理下一批。

### 解决：连续批处理

```mermaid
gantt
    title 连续批处理（完成即退出，新请求立即加入）
    dateFormat X
    axisFormat %s

    section Request A
    生成 :a1, 0, 3

    section Request B
    生成 :b1, 0, 8

    section Request C
    生成 :c1, 0, 5

    section Request D (新)
    生成 :d1, 3, 8
```

A 完成后，新 Request D 立即加入当前 batch，GPU 不空闲。

---

## vLLM 请求处理流程

```mermaid
sequenceDiagram
    participant C as 客户端
    participant API as vLLM API Server
    participant S as Scheduler
    participant E as Model Executor
    participant GPU as GPU

    C->>API: POST /v1/chat/completions
    API->>S: 新请求入队
    S->>S: 加入当前 batch
    S->>E: 调度执行
    E->>GPU: Prefill（处理输入 prompt）
    GPU->>E: KV Cache 写入
    loop 逐 token 生成
        E->>GPU: Decode（生成下一个 token）
        GPU->>E: token + 更新 KV Cache
        E->>API: stream token
        API->>C: SSE 流式返回
    end
    E->>S: 请求完成，释放 KV Cache 页
```

---

## 部署架构

```mermaid
graph TB
    subgraph Production["生产部署"]
        LB[负载均衡]
        LB --> V1[vLLM Instance 1<br/>GPU 0-1]
        LB --> V2[vLLM Instance 2<br/>GPU 2-3]
        LB --> V3[vLLM Instance N<br/>GPU ...]

        V1 --> API1[OpenAI API :8000]
        V2 --> API2[OpenAI API :8000]
    end

    CLIENT[Agent Harness] -->|HTTP| LB
```

### 启动命令

```bash
# 单卡部署
python -m vllm.entrypoints.openai.api_server \
    --model deepseek-ai/DeepSeek-V3 \
    --tensor-parallel-size 1

# 多卡并行
python -m vllm.entrypoints.openai.api_server \
    --model deepseek-ai/DeepSeek-V3 \
    --tensor-parallel-size 8
```

Agent Harness 通过 OpenAI 兼容 API 调用 vLLM，无需特殊适配。

---

## 关键术语速查

| 术语 | 含义 |
|------|------|
| **vLLM** | 高性能 LLM 推理引擎（UC Berkeley 出品） |
| **PagedAttention** | 分页式 KV Cache 管理，大幅减少内存浪费 |
| **Continuous Batching** | 连续批处理，完成即退出、新请求立即加入 |
| **KV Cache** | 存储历史 token 的 Key/Value 向量，避免重复计算 |
| **Prefill** | 处理输入 prompt 的阶段（计算密集） |
| **Decode** | 逐 token 生成的阶段（内存密集） |
| **Tensor Parallelism** | 模型权重切分到多 GPU |

---

[← 上一章：Firecracker](03-what-is-firecracker.md) | [下一章：SGLang →](05-sglang-explained.md)
