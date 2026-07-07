# 03 - 什么是 Firecracker（微虚拟机）？

## 一句话定义

> **Firecracker 是 AWS 开源的轻量级虚拟机管理器，用硬件虚拟化实现毫秒级启动和强安全隔离。**

它是 Agent 沙箱的「终极形态」之一 —— 当 AI 生成的代码不可信时，需要在硬件级别隔离执行。

---

## 为什么 Agent 需要沙箱？

Agent 能自主执行代码、调用系统命令。如果 LLM 生成了恶意代码：

```mermaid
graph TB
    subgraph Risk["没有沙箱的风险"]
        LLM[LLM 生成代码] --> EXEC[直接在宿主机执行]
        EXEC --> R1[删除系统文件]
        EXEC --> R2[窃取环境变量/API Key]
        EXEC --> R3[发起网络攻击]
        EXEC --> R4[消耗全部 CPU/内存]
    end
```

**沙箱 = 给 AI 代码一个安全的「牢笼」，在里面运行，无法影响外部系统。**

---

## 隔离技术谱系

从轻到重，隔离技术有一个光谱：

```mermaid
graph LR
    subgraph Spectrum["隔离强度 →"]
        P[Process<br/>进程隔离]
        W[Wasm<br/>字节码沙箱]
        C[Container<br/>容器隔离]
        G[gVisor<br/>用户态内核]
        F[Firecracker<br/>微虚拟机]
    end

    P -->|弱| W --> C --> G -->|强| F

    style P fill:#ffcccc
    style W fill:#ffe0cc
    style C fill:#ffffcc
    style G fill:#ccffcc
    style F fill:#cce0ff
```

| 技术 | 隔离原理 | 启动速度 | 安全级别 | 适用场景 |
|------|----------|----------|----------|----------|
| **Process** | 子进程 + 超时 | ~1ms | 低（共享内核） | Demo、可信工具 |
| **Wasm** | 字节码 + 内存安全 | ~1ms | 中（无 syscalls） | AI 生成代码 |
| **Container** | Linux Namespace | ~100ms | 中（共享内核） | 常规服务部署 |
| **gVisor** | 用户态内核拦截 syscall | ~100ms | 高 | 不可信代码（Linux） |
| **Firecracker** | KVM 硬件虚拟化 | ~125ms | 最高 | 多租户 Agent 平台 |

---

## Firecracker 是什么？

```mermaid
graph TB
    subgraph Host["物理服务器"]
        KVM[KVM 虚拟化层]
        FC[Firecracker VMM]
        KVM --> FC

        subgraph MicroVM1["MicroVM 1"]
            KERNEL1[极简 Linux 内核]
            APP1[Agent 沙箱进程]
            KERNEL1 --> APP1
        end

        subgraph MicroVM2["MicroVM 2"]
            KERNEL2[极简 Linux 内核]
            APP2[Agent 沙箱进程]
            KERNEL2 --> APP2
        end

        FC --> MicroVM1
        FC --> MicroVM2
    end
```

Firecracker 的关键设计：

| 特点 | 说明 |
|------|------|
| **MicroVM** | 极简虚拟机，只模拟必要设备（键盘、串口、网络） |
| **KVM 加速** | 利用 CPU 硬件虚拟化，性能接近原生 |
| **125ms 启动** | 比传统 VM（数秒）快 10-100 倍 |
| **< 5MB 内存** | 每个 MicroVM 开销极小 |
| **安全优先** | 最小设备模型 = 最小攻击面 |

---

## Firecracker vs Docker 容器

```mermaid
graph TB
    subgraph Container["Docker 容器"]
        APP_C[应用] --> GLIBC_C[glibc]
        GLIBC_C --> OS_C[共享 Linux 内核]
        OS_C --> HW_C[硬件]
    end

    subgraph Firecracker["Firecracker MicroVM"]
        APP_F[应用] --> GLIBC_F[glibc]
        GLIBC_F --> KERNEL_F[独立 Linux 内核]
        KERNEL_F --> KVM_F[KVM 虚拟化]
        KVM_F --> HW_F[硬件]
    end
```

| | Docker 容器 | Firecracker MicroVM |
|---|------------|---------------------|
| **隔离级别** | 进程级（Namespace） | 硬件级（KVM） |
| **内核** | 共享宿主机内核 | 每个 VM 独立内核 |
| **启动时间** | ~100ms | ~125ms |
| **内存开销** | ~10MB | ~5MB |
| **安全模型** | 信任内核不被突破 | 即使内核被攻破也不影响宿主机 |
| **典型用途** | 微服务部署 | 多租户不可信代码执行 |

**关键区别**：容器里容器逃逸（Container Escape）可以影响宿主机；MicroVM 里即使 VM 内内核被攻破，也无法突破 KVM 虚拟化层。

---

## Agent 平台的沙箱架构

```mermaid
graph TB
  subgraph Platform["Agent Harness 平台"]
    SCHED[沙箱调度器]
    SCHED -->|低风险| PS[Process Sandbox]
    SCHED -->|代码执行| WS[Wasm Sandbox]
    SCHED -->|不可信代码| FC[Firecracker MicroVM]
  end

  subgraph K8s["Kubernetes 集群"]
    FC --> POD1[MicroVM Pod 1]
    FC --> POD2[MicroVM Pod 2]
    FC --> POD3[MicroVM Pod N]
  end

  AGENT[Agent Loop] --> SCHED
```

### 为什么不建议自研 Firecracker？

| 维度 | 自研 | 集成现有方案 |
|------|------|-------------|
| 开发周期 | 1-2 年+ | 数周 |
| 安全审计 | 需要专业团队 | AWS 已生产验证 |
| 维护成本 | 持续跟进内核 CVE | 上游维护 |
| 我们的价值 | 在 VM 层以下 | **在调度编排层** |

**正确做法**：通过 K8s RuntimeClass 集成 Firecracker/gVisor，自己专注做 Harness 层的沙箱调度、生命周期管理和 Trace。

---

## 实际集成方式

不直接调用 Firecracker API，而是通过容器生态集成：

```mermaid
graph LR
    HARNESS[Agent Harness] --> K8S[Kubernetes API]
    K8S --> RC[RuntimeClass: firecracker]
    RC --> FC_CTR[firecracker-containerd]
    FC_CTR --> MICROVM[MicroVM]
    MICROVM --> AGENT_CODE[Agent 代码执行]
```

```yaml
# Kubernetes RuntimeClass 示例
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: firecracker
handler: fc-run  # firecracker-containerd handler
```

---

## 关键术语速查

| 术语 | 含义 |
|------|------|
| **Firecracker** | AWS 开源的 MicroVM 管理器 |
| **MicroVM** | 极简虚拟机，最小设备模型 |
| **KVM** | Linux 内核虚拟化模块，硬件加速 |
| **VMM** | Virtual Machine Monitor，虚拟机监视器 |
| **Container Escape** | 容器逃逸，突破 Namespace 隔离 |
| **RuntimeClass** | K8s 选择容器运行时的机制 |
| **gVisor** | Google 的用户态内核，syscall 拦截方案 |

---

[← 上一章：MCP](02-what-is-mcp.md) | [下一章：vLLM →](04-vllm-explained.md)
