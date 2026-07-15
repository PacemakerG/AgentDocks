# Agent Runtime Infra 秋招项目规划

> 项目暂定名：**AgentDock**  
> 定位：面向 Coding Agent 的安全执行、任务调度与可观测基础设施  
> 规划日期：2026 年 7 月

## 一、最终判断

这个方向值得做，而且比继续新增一个普通 Agent 应用更适合当前阶段。

目前已有项目覆盖：

- **MedAgent**：Agent 应用、RAG、多智能体工作流。
- **AgentLens（CCWhat）**：Coding Agent 的日志采集、Trace、Task 切分、回放、诊断、报告和 Dataset。
- **美团实习**：真实研发流程、平台工具和工程落地经验。

当前最明显的能力缺口不是再实现一种 Agent，而是：

> Agent 在什么环境里执行、能访问什么资源、多个任务如何并发调度、失败后如何回收和恢复。

因此，新项目应补齐 **Agent Runtime / Execution Plane**，并与 AgentLens 的 **Observability / Diagnosis Plane** 对接。

最终形成：

```text
Agent 应用
    ↓
AgentDock：安全执行、资源隔离、并发调度
    ↓
AgentLens：Trace、回放、诊断、评测、Dataset
```

这条路线能把个人定位从“Agent 应用开发”推进到：

> 具备 Coding Agent 应用、运行时隔离、任务调度、可观测诊断和评测闭环经验的 Agent Infrastructure 工程师。

但需要明确：项目的壁垒不来自“用了 Rust、Go、C++”，而来自是否真正解决了隔离、安全、并发、故障恢复、资源治理和可观测问题。

---

## 二、最适合秋招的项目

# AgentDock：面向 Coding Agent 的安全执行与调度平台

## 2.1 项目定位

AgentDock 提供一个统一运行时，让 Claude Code、Codex、OpenCode 或自定义 Agent 可以在受控环境中执行代码。

它负责：

- 创建独立 Workspace。
- 启动和管理 Agent 进程。
- 控制文件、网络和系统调用权限。
- 限制 CPU、内存、PID 和运行时间。
- 流式返回 stdout、stderr 和 PTY 输出。
- 支持任务提交、取消、超时、重试和恢复。
- 记录进程树、资源使用、网络访问、文件变化和退出原因。
- 将完整 Trace 发送给 AgentLens。

它不负责：

- 重新实现大模型。
- 再写一套 Agent 工作流框架。
- 训练模型或实现 CUDA 算子。
- 一开始就实现 Kubernetes、MicroVM 和大规模集群。

## 2.2 核心架构

```text
                   ┌──────────────────────────┐
                   │     Go Control Plane     │
Client / CLI ─────▶│                          │
                   │ Task API                 │
                   │ Scheduler                │
                   │ Worker Registry          │
                   │ Lease / Heartbeat        │
                   │ Retry / Cancellation     │
                   │ Tenant Quota             │
                   └────────────┬─────────────┘
                                │ gRPC
                   ┌────────────▼─────────────┐
                   │   Rust Sandbox Worker    │
                   │                          │
                   │ Process Supervisor       │
                   │ Filesystem Policy        │
                   │ Network Policy           │
                   │ cgroups v2               │
                   │ Timeout / Cancellation   │
                   │ PTY / Log Streaming      │
                   │ Workspace / Git Diff     │
                   └────────────┬─────────────┘
                                │ Trace Protocol
                   ┌────────────▼─────────────┐
                   │        AgentLens         │
                   │ Trace / Replay / Eval    │
                   └──────────────────────────┘
```

## 2.3 Rust Sandbox Worker

Rust 只承担真正需要系统能力的执行面。

核心能力：

1. **进程生命周期**
   - 启动 Agent 或普通命令。
   - 管理完整子进程树。
   - 支持超时、主动取消和强制终止。
   - 防止僵尸进程和孤儿进程。
   - 记录退出码、信号和失败原因。

2. **并发执行**
   - 使用 Tokio 管理多个并发任务。
   - stdout、stderr 异步流式输出。
   - 为每个任务维护独立状态机。
   - 限制单 Worker 最大并发数。

3. **文件系统隔离**
   - 每个任务独立 Workspace。
   - 默认禁止读取宿主机敏感目录。
   - Workspace 内区分只读路径和可写路径。
   - 记录任务前后文件变化和 Git Diff。
   - Linux 下逐步接入 Landlock 或 Bubblewrap。

4. **资源限制**
   - 使用 cgroups v2 限制 CPU、内存、PID 和 IO。
   - 限制最大运行时间和最大输出量。
   - 检测 OOM、Fork Bomb、Disk Bomb 和无限输出。

5. **网络策略**
   - 默认关闭网络。
   - 支持域名或代理白名单。
   - 记录访问目标、是否允许以及拒绝原因。
   - 不在第一版实现复杂的透明流量代理。

6. **Trace 输出**
   - 统一记录 `task_id`、`agent_id`、`sandbox_id`、`process_id`。
   - 输出进程、命令、资源、文件和策略拒绝事件。
   - 通过结构化事件接入 AgentLens。

## 2.4 Go Control Plane

Go 负责服务治理和调度，不用于实现底层隔离。

核心能力：

- REST / gRPC API。
- Worker 注册与注销。
- Worker Heartbeat。
- Task Queue。
- 并发额度和租户配额。
- Worker Lease。
- 任务取消。
- 超时和有限次数重试。
- Worker 崩溃后的任务重新分配。
- 幂等任务提交。
- Task、Attempt、Sandbox、Artifact 状态管理。

建议状态机：

```text
PENDING
  → ASSIGNED
  → STARTING
  → RUNNING
  → SUCCEEDED / FAILED / CANCELLED / TIMED_OUT
```

任务和执行尝试必须分离：

```text
Task
 ├── Attempt 1：Worker 崩溃
 ├── Attempt 2：超时
 └── Attempt 3：成功
```

否则无法正确表达重试、迁移和故障恢复。

## 2.5 多 Agent 能力应该怎么做

不要再实现“Planner、Coder、Reviewer 在同一个 Python 进程里聊天”。

真正的 Multi-Agent Infra 应该是：

```text
Planner Sandbox
    ↓ 输出 plan.json

Executor Sandbox
    ↓ 获得 Workspace 写权限

Reviewer Sandbox
    ↓ 只读代码和 diff

Test Sandbox
    ↓ 只允许执行测试
```

每个 Agent：

- 独立进程或独立 Sandbox。
- 拥有不同权限。
- 只共享显式 Artifact。
- 使用固定输入输出 Schema。
- 可以单独重试。
- 可以单独取消。
- 所有执行记录进入同一条 Task Trace。

这个能力应作为后期扩展，而不是第一版起点。

## 2.6 必须完成的实验

项目不能只有架构图，必须提供实测数据。

| 维度 | 建议指标 |
|---|---|
| 启动性能 | Sandbox 启动延迟 p50 / p95 / p99 |
| 并发能力 | 1、10、50、100 个任务下的吞吐和延迟 |
| 资源限制 | CPU、内存、PID 限制是否生效 |
| 取消能力 | 发出取消到完整进程树退出的延迟 |
| 故障恢复 | Worker 崩溃后重新分配成功率 |
| 文件隔离 | 能否读取 `~/.ssh`、其他 Workspace 和宿主目录 |
| 网络隔离 | 默认拒绝和白名单规则是否准确 |
| 稳定性 | Fork Bomb、无限输出、超时、OOM 测试 |
| 可复现性 | 相同输入和环境能否重新执行 |
| 可观测性 | Control Plane、Worker、Process、Tool Trace 能否关联 |

---

## 三、最值得看的开源项目

## 3.1 第一优先级：直接学习和复现

| 项目 | 重点 | 应该借鉴什么 |
|---|---|---|
| [Sandlock](https://github.com/multikernel/sandlock) | Rust、Landlock、seccomp、进程级沙箱 | Rust Sandbox Worker、权限策略、Supervisor、轻量隔离 |
| [Anthropic Sandbox Runtime](https://github.com/anthropic-experimental/sandbox-runtime) | 文件和网络权限控制 | 面向 Agent 的 Policy API、安全默认值、违规事件 |
| [SWE-ReX](https://github.com/SWE-agent/SWE-ReX) | Agent 远程执行接口、并发 Shell | Runtime 抽象、交互式 Shell、不同执行后端统一接口 |
| [E2B Infra](https://github.com/e2b-dev/infra) | AI 代码执行云基础设施 | Control Plane、Compute Plane、Sandbox 生命周期、Worker 管理 |
| [OpenAI Codex](https://github.com/openai/codex) | Rust Coding Agent | 命令执行、审批、Sandbox、协议和客户端架构 |

### 推荐阅读顺序

1. **Anthropic Sandbox Runtime**
   - 先理解一个面向 Agent 的沙箱产品应该暴露什么配置和权限。
   - 重点不是照抄 TypeScript，而是学习安全边界和使用体验。

2. **SWE-ReX**
   - 理解 Agent 和执行环境之间应该如何解耦。
   - 重点关注 Shell Session、交互式命令、并发执行和多后端接口。

3. **Sandlock**
   - 深入 Rust、Landlock、seccomp、Supervisor 和进程级隔离。
   - 适合作为 Rust Worker 的主要技术参考。

4. **OpenAI Codex**
   - 阅读成熟 Coding Agent 如何组织 Rust 工程。
   - 重点看命令执行、审批策略、Sandbox 和事件协议，不要试图完整复现 Codex。

5. **E2B Infra**
   - 学习完整平台如何拆分控制面和计算面。
   - 第一版不要直接复现 Firecracker 和云基础设施。

## 3.2 第二优先级：理解底层边界

| 项目 | 重点 | 是否直接复现 |
|---|---|---|
| [youki](https://github.com/youki-dev/youki) | Rust OCI Runtime、Namespace、cgroups、rootless | 阅读核心模块，不完整复现 |
| [gVisor](https://github.com/google/gvisor) | Go 用户态内核、系统调用隔离 | 理解安全边界，不作为秋招主项目 |
| [Firecracker](https://github.com/firecracker-microvm/firecracker) | Rust MicroVM、KVM、多租户强隔离 | 只学习架构，暂不复现 |
| [Kata Containers](https://github.com/kata-containers/kata-containers) | VM 隔离与容器接口结合 | 作为后续执行后端参考 |
| [Daytona](https://github.com/daytonaio/daytona) | Interface / Control / Compute Plane | 学习平台拆分、SDK 和 Sandbox 生命周期 |

## 3.3 推荐论文

- [Sandlock: Confining AI Agent Code with Unprivileged Linux Primitives](https://arxiv.org/abs/2605.26298)
  - 重点看轻量进程沙箱、安全策略拆分和性能实验。
- 后续需要研究大规模 Coding Agent 训练或评测环境时，再补充 SWE Sandbox、并行评测和 Agent 调度论文。
- 秋招前优先把开源代码和真实实验做出来，不要让论文调研替代工程实现。

---

## 四、实际实施顺序

## 阶段 0：建立最小执行协议

先定义协议，不写复杂平台。

请求：

```json
{
  "task_id": "task-001",
  "command": ["codex", "exec", "fix the failing tests"],
  "workspace": "/workspace/repo",
  "timeout_seconds": 300,
  "cpu_limit": 1,
  "memory_limit_mb": 2048,
  "network_policy": "deny"
}
```

事件：

```json
{
  "task_id": "task-001",
  "sandbox_id": "sandbox-001",
  "type": "process_started",
  "timestamp": "...",
  "payload": {}
}
```

第一版只需要：

- `process_started`
- `stdout`
- `stderr`
- `resource_sample`
- `file_changed`
- `policy_denied`
- `process_exited`

## 阶段 1：Rust 单机 Worker

目标：在一台 Linux 机器上可靠地运行和回收 Agent 任务。

完成：

- Rust CLI。
- Tokio 并发。
- 进程树管理。
- stdout / stderr 流式输出。
- 超时和取消。
- Workspace 隔离。
- Git Diff。
- cgroups v2。
- 基础文件权限限制。
- 安全和压力测试。

CLI 示例：

```bash
agentdock run \
  --workspace ./repo \
  --memory 2g \
  --cpu 1 \
  --network deny \
  --timeout 300 \
  -- codex exec "修复失败测试"
```

验收标准：

- 任务取消后没有残留子进程。
- 超时任务可以确定性退出。
- 无法读取宿主机 SSH Key。
- Fork Bomb 不会拖垮宿主机。
- 可以同时运行多个任务并独立返回日志。
- 能输出完整 Git Diff 和资源统计。

## 阶段 2：Go Control Plane

目标：从“能运行一个任务”升级到“能管理一批任务”。

完成：

- Task API。
- Worker Registry。
- Queue。
- Lease。
- Heartbeat。
- Retry。
- Cancellation。
- Quota。
- 幂等提交。
- Worker Crash Recovery。
- PostgreSQL 或 SQLite 状态存储。

第一版不需要：

- Kubernetes Operator。
- 跨地域调度。
- 复杂优先级算法。
- 一致性协议。
- 自研消息队列。

## 阶段 3：接入 AgentLens

统一标识：

```text
session_id
task_id
attempt_id
agent_id
sandbox_id
process_id
tool_call_id
```

AgentLens 新增展示：

- Sandbox 生命周期。
- 任务调度和重试过程。
- 进程树。
- CPU、内存、PID 曲线。
- 文件变化。
- 网络访问。
- Policy Denial。
- 退出原因。
- Worker 崩溃与任务迁移。

这一步会把两个项目连成完整故事：

```text
AgentDock 负责执行和治理
AgentLens 负责观测、诊断和评测
```

## 阶段 4：多 Agent DAG

最后再实现：

- Planner。
- Executor。
- Reviewer。
- Tester。
- 权限隔离。
- Artifact 传递。
- 节点级重试。
- DAG 回放。

如果时间不足，这一阶段可以不做。单 Agent 安全执行、调度和 AgentLens 接入已经足以形成完整秋招项目。

---

## 五、技术选型

## 5.1 推荐组合

| 模块 | 技术 |
|---|---|
| Sandbox Worker | Rust |
| 异步运行时 | Tokio |
| Worker API | tonic gRPC 或 Axum |
| Linux 系统接口 | `nix`、Landlock、seccomp、cgroups v2 |
| Control Plane | Go |
| 服务通信 | gRPC |
| 元数据存储 | 第一版 SQLite，后续 PostgreSQL |
| 队列 | 第一版数据库队列或内存队列，后续再考虑 NATS / Redis |
| Trace 协议 | JSONL 起步，后续兼容 OpenTelemetry |
| Agent Adapter | Python |
| 诊断和 Viewer | AgentLens |

## 5.2 为什么使用 Rust

Rust 用在：

- 进程生命周期。
- 系统调用。
- 异步 IO。
- 文件和网络权限。
- 资源限制。
- 长时间运行的 Worker。

这能形成真实的系统编程经验，而不是为了简历增加一种语言。

## 5.3 为什么使用 Go

Go 用在：

- API Server。
- Worker 管理。
- 并发任务调度。
- Heartbeat。
- Lease。
- Retry。
- 服务状态和资源配额。

Go 的价值是快速建立稳定控制面，不需要拿它重写 Rust 已经负责的底层执行逻辑。

## 5.4 为什么暂时不使用 C++

目前没有一个必须使用 C++ 才能解决的核心问题。

强行加入 C++ 会导致：

- 项目语言过多。
- 构建和部署复杂。
- 学习成本扩大。
- 核心功能完成度下降。

只有在后续需要以下能力时再考虑 C/C++：

- eBPF 用户态组件。
- 高性能网络代理。
- 特定 VMM 或系统组件。
- 与已有 C/C++ Runtime 进行 FFI。

## 5.5 开发环境限制

底层能力依赖 Linux：

- cgroups v2。
- Landlock。
- seccomp。
- Namespace。
- Bubblewrap。
- eBPF。

因此：

- macOS 只作为开发客户端。
- 使用 Linux 云主机、远程服务器或本地 Linux VM 开发 Worker。
- 不要为了同时支持 macOS 和 Windows，削弱第一版的 Linux 能力。
- 第一版明确声明：**Linux Worker，跨平台 Client。**

---

## 六、仓库结构建议

```text
agentdock/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SECURITY_MODEL.md
│   ├── TRACE_PROTOCOL.md
│   ├── BENCHMARK.md
│   └── ROADMAP.md
├── proto/
│   └── agentdock.proto
├── worker/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       ├── supervisor/
│       ├── sandbox/
│       ├── cgroup/
│       ├── workspace/
│       ├── network/
│       └── trace/
├── control-plane/
│   ├── go.mod
│   └── internal/
│       ├── api/
│       ├── scheduler/
│       ├── worker/
│       ├── lease/
│       ├── task/
│       └── store/
├── sdk/
│   └── python/
├── tests/
│   ├── security/
│   ├── failure/
│   └── benchmark/
└── examples/
    ├── codex/
    ├── claude-code/
    └── multi-agent/
```

不要先把空目录全部建出来。目录结构应随着实现逐步形成，避免只有脚手架没有能力。

---

## 七、项目的秋招叙事

简历中不要写成：

> 使用 Rust 和 Go 开发 Agent 沙箱平台。

更好的表达是：

> 设计并实现面向 Coding Agent 的安全执行与调度平台，将 Agent 任务运行在资源受限的独立 Workspace 中；基于 Rust/Tokio 管理并发进程、超时取消和完整进程树回收，结合 cgroups v2 与文件/网络策略实现资源和权限治理。

> 使用 Go 构建任务控制面，实现 Worker 注册、Heartbeat、Lease、幂等提交、失败重试和崩溃恢复；通过统一 Trace Protocol 将 Sandbox、Process、资源和文件事件接入 AgentLens，形成执行、观测、诊断和回放闭环。

面试时应能回答：

- cgroups 为什么不等于安全沙箱？
- Landlock、seccomp、Namespace 分别限制什么？
- 如何杀死 Agent 创建的完整进程树？
- 为什么 Task 和 Attempt 必须分离？
- Worker Lease 如何防止任务永久占用？
- Worker 崩溃后如何避免任务重复执行？
- 取消请求如何做到幂等？
- 如何限制无限 stdout？
- 网络白名单如何防止绕过？
- 如何把调度 Trace 和 Agent Trace 关联起来？
- 100 个任务并发时瓶颈在哪里？

这些问题能回答清楚，项目才真正产生壁垒。

---

## 八、最终执行建议

1. 新建独立仓库 `AgentDock`，不要直接塞进 AgentLens。
2. 第一阶段只实现 Linux 单机 Rust Worker。
3. 先证明进程管理、资源限制和安全策略真实有效。
4. 再增加 Go Control Plane。
5. 然后接入 AgentLens。
6. 最后根据时间决定是否增加 Multi-Agent DAG。
7. 暂时不引入 C++、Kubernetes、Firecracker 和复杂分布式调度。
8. 每个阶段必须同时提供测试、Benchmark、故障案例和设计文档。

最重要的优先级是：

```text
真实可运行
> 安全边界明确
> 故障可以恢复
> 指标可以验证
> 架构可以解释
> 技术栈数量
```

这个项目真正新增的不是一个新的 Agent，而是此前项目没有的能力：

> **让 Agent 能够被安全、稳定、并发、可恢复地运行，并且将执行证据完整交给 AgentLens。**
