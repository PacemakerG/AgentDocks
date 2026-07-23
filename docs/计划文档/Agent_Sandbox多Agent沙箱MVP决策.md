# AgentDocks：基于 Agent Sandbox 的 Multi-Agent Sandbox MVP 决策

> 决策日期：2026 年 7 月  
> 基础项目：[agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox)  
> 产品与管理参考：[clawfleet/ClawFleet](https://github.com/clawfleet/ClawFleet)  
> 本文取代《Agent-Sandbox 复现与生产化增强计划》，其底座复现与基线验证部分已并入本文第五节；原计划的 Lease + Prometheus 增强主线不再执行。

## 一、最终结论

本项目应当：

> **以 Agent Sandbox 为 Kubernetes Sandbox 底座，在其上部署一套最小 Multi-Agent 系统；同时明确借鉴 ClawFleet 的 Agent 实例管理、生命周期操作、实时资源监控和统一管理界面。**

不建议以 ClawFleet 为底座，再把其 Docker 管理层整体改造成 Kubernetes。

两个项目的分工应当明确：

```text
ClawFleet 提供产品与管理参考
- 一个 Agent 对应一个独立运行实例
- Agent 实例列表和卡片
- 创建、启动、停止、重启、销毁
- CPU、内存实时状态
- 实时日志和生命周期事件
- 终端、桌面和运行时配置入口

Agent Sandbox 提供实际基础设施底座
- Go 控制面
- Kubernetes 原生部署
- Sandbox 创建、暂停、恢复和删除
- CPU、内存等资源配置
- Sandbox Pool 和超时回收
- Pod 生命周期和 Kubernetes 接入
```

最终不是简单把两个项目拼接，而是形成：

```text
ClawFleet 风格的 Agent 管理体验
              ↓
自己实现的最小 Agent 调度层
              ↓
Agent Sandbox 的 Sandbox 管理能力
              ↓
Kubernetes Pod 和容器资源
```

E2B 兼容不是项目主线。MVP 优先使用 Agent Sandbox 自身 API 和内部管理能力，E2B 接口只保留为可选兼容层。

---

## 二、为什么选择 Agent Sandbox 作为底座

Agent Sandbox 已经解决了本项目最难、也最符合 Agent Infra 岗位的基础问题：

```text
创建 Sandbox 请求
        ↓
Agent Sandbox 控制面
        ↓
创建 Kubernetes 工作负载
        ↓
Kubernetes 调度 Pod
        ↓
容器中运行 Agent
        ↓
超时、失败或完成后回收资源
```

在这个基础上增加 Multi-Agent，只需要补充 Agent 语义和任务调度，不用重新实现 Kubernetes 接入层。

需要新增的核心关系是：

```text
Task
├── Explorer Agent → Sandbox A → Pod A
├── Tester Agent   → Sandbox B → Pod B
└── Reviewer Agent → Sandbox C → Pod C
```

项目重点不是重新发明 Sandbox，而是实现：

- Agent 与 Sandbox 的绑定。
- 多个 Agent 的运行顺序和依赖。
- 每个 Agent 的 CPU、内存和超时配置。
- 并发额度和等待队列。
- Agent 失败后的干净沙箱重试。
- 任务完成后的自动回收。

这部分可以直接覆盖 Agent Infra 岗位中的核心能力：

- Kubernetes 实际使用。
- 容器化 Agent 执行。
- Sandbox 生命周期管理。
- 资源分配。
- 任务调度与状态管理。
- 超时、失败重试和资源回收。

---

## 三、ClawFleet 具体借鉴什么

ClawFleet 不能只被理解为一个前端参考项目。它已经实现了一套较完整的单机 Agent 实例管理体验，值得借鉴的是下面四部分。

### 3.1 Agent 实例抽象

ClawFleet 将每个 Agent 表达为一个独立实例：

```text
Agent Instance
├── runtime：OpenClaw / Hermes
├── container
├── model configuration
├── character / role
├── channel / tools
├── runtime status
└── logs / resource stats
```

AgentDocks 应采用相同的产品表达，但底层映射改为：

```text
AgentRun
├── agent_role
├── runtime_image
├── sandbox_id
├── pod_name
├── CPU / memory
├── status
├── retry_count
└── logs / events
```

也就是：

> ClawFleet 中一个 Agent 对应一个 Docker 容器；AgentDocks 中一个 AgentRun 对应一个 Agent Sandbox，再对应一个 Kubernetes Pod。

### 3.2 生命周期管理交互

借鉴 ClawFleet 的统一生命周期操作：

- 创建 Agent 实例。
- 启动。
- 停止。
- 重启。
- 删除和清理数据。
- 进入终端。
- 查看实例详情。

在 AgentDocks 中，对应为：

```text
Create AgentRun
→ Create Sandbox
→ Wait Pod Ready
→ Start Agent
→ Stop / Retry / Cancel
→ Delete Sandbox
```

### 3.3 Monitoring：资源、日志和事件

ClawFleet 已经提供三类基础运行信息：

```text
实时 CPU / 内存
实时容器日志
创建、启动、停止等生命周期事件
```

AgentDocks 的第一版应借鉴这种监控体验，但不重复实现 AgentLens 的深度 Trace。

前端至少展示：

- Agent 当前状态。
- Sandbox 和 Pod 名称。
- 申请的 CPU、内存。
- 当前 CPU、内存使用情况；如果第一版取不到实时值，先展示资源申请值和 Pod 状态。
- Sandbox 创建、启动、失败、重试和回收事件。
- Agent stdout / stderr 基础日志。

边界需要明确：

```text
AgentDocks：Sandbox、Pod、资源和生命周期监控
AgentLens：模型调用、工具调用、文件变化、任务诊断和失败归因
```

### 3.4 统一管理界面

借鉴 ClawFleet 的 Fleet 视图，不再让用户只面对一组 Sandbox ID。

建议页面层级：

```text
Task List
  ↓
Task Detail
  ├── Explorer Agent Card
  ├── Tester Agent Card
  └── Reviewer Agent Card
        ↓
Agent Detail
  ├── Sandbox / Pod
  ├── Resource
  ├── Status Events
  ├── Logs
  └── Stop / Retry / Delete
```

这会让项目从“调用 Sandbox API”变成真正可演示的 Agent Runtime 管理系统。

### 3.5 不应误解 ClawFleet 的“调度”能力

ClawFleet 主要实现的是：

> 一台机器上多个长期 Agent 容器的创建、运行、监控和管理。

它没有完整实现本项目需要的：

- 任务依赖调度。
- 等待队列。
- 最大并发额度。
- 上游完成后唤醒下游。
- 失败后创建干净沙箱重试。
- 临时 Agent 完成后的自动回收。

因此 AgentDocks 中有三层不同职责：

```text
第一层：ClawFleet 风格的实例管理和 Monitoring
- 看得见、管得住每个 Agent

第二层：自己实现的最小 Agent Scheduler
- 决定哪个 Agent 现在运行、哪个等待、何时重试

第三层：Agent Sandbox + Kubernetes
- 创建 Sandbox，并决定 Pod 最终运行在哪个节点
```

---

## 四、为什么不以 ClawFleet 为底座

ClawFleet 的当前结构是：

```text
ClawFleet
    ↓ Docker API
单台机器上的 Docker Engine
    ↓
多个长期运行的 Agent 容器
```

它适合管理一组长期在线的 Agent，例如 CTO、开发、产品等固定角色。它已经具备：

- Agent 容器创建、启动、停止和删除。
- CPU、内存和日志展示。
- 浏览器 Dashboard。
- 容器终端和桌面访问。

但它缺少：

- Kubernetes 集群调度。
- Pod 和节点状态管理。
- Kubernetes 资源声明和回收。
- 临时任务型 Sandbox 生命周期。
- 集群资源不足时的排队。
- Pod 异常后的重新创建。

如果以 ClawFleet 为底座增加 Kubernetes，实际上需要替换其核心基础设施层：

```text
原有：go-dockerclient → Docker Engine
改造：Kubernetes Client → Pod / Deployment / Service
```

同时还要重新实现：

- Kubernetes 资源模板。
- Pod 就绪检测。
- 节点与资源状态同步。
- 网络访问和服务发现。
- 超时删除和异常清理。
- Agent 到 Pod 的映射。

这不是小改造，而是把 ClawFleet 的底层执行系统重新写一遍。一个月内投入产出比不高，并且会重复 Agent Sandbox 已完成的能力。

因此：

> **ClawFleet 是产品设计、实例管理和 Monitoring 参考；Agent Sandbox 才是代码与基础设施底座。**

---

## 五、前置阶段：底座复现与基线验证

> 本节合并自原《Agent-Sandbox 复现与生产化增强计划》。在实现 Multi-Agent MVP 之前，必须先跑通并验证底座；底座指标不达标时先修底座，再建上层。

### 5.1 开发环境

- Mac 本地：Docker Desktop / OrbStack + kind 或 minikube，Kubernetes 1.26+。用于跑通安装、阅读源码、调试 API 和 Controller、验证基本生命周期。
- Linux 云服务器（Ubuntu 22.04/24.04，x86_64，8 核 16 GB，单节点 k3s 起步）：并发实验、Controller 重启、资源不足实验和长时间稳定性测试。Apple Silicon 遇到镜像架构问题时不要在本地纠缠，直接迁移到 x86_64。
- 源码修改放在上游 Fork（`agent-sandbox/agent-sandbox` → 个人 Fork）；本仓库只放部署配置、验证脚本、实验数据、复现报告和不适合合入上游的扩展组件。README 和简历必须清楚区分上游已有能力、个人实现的改动、个人完成的实验与结论。
- 云服务器部署时：必须更换默认管理 Token，不直接向公网开放 `/ui`、API 和 Kubernetes 管理端口，优先通过防火墙、VPN 或 SSH 端口转发访问。

### 5.2 阶段一：完整复现 Agent Sandbox（2～4 天）

必须实际跑通：

- UI 登录。
- 创建 Sandbox。
- 查看 Sandbox 状态。
- 访问 Shell 或 Terminal。
- 文件读写。
- 执行代码。
- 访问 Browser 或 VNC 环境。
- 暂停与恢复。
- 超时自动回收。
- 手动删除。
- REST API 调用。
- MCP 调用。
- E2B SDK 兼容调用。
- Sandbox Pool 的创建和复用。

输出 `docs/REPRODUCTION_REPORT.md`，内容必须包括：

- 操作系统、CPU 架构、Kubernetes 版本。
- 完整安装命令。
- 成功运行的功能与没有跑通的功能。
- 实际架构图。
- 关键 Controller、API、Pool 和 Sandbox 生命周期代码入口。
- 本地环境遇到的问题。
- 与 README 宣传不一致的地方。

### 5.3 阶段二：基线验证场景

每个场景都必须可以通过脚本重复执行，并给出确定性结果：

| 场景 | 需要验证的内容 |
|---|---|
| 正常生命周期 | 创建、访问、暂停、恢复、删除是否正确 |
| 超时回收 | TTL 到期后是否及时删除，状态是否一致 |
| 空闲回收 | 没有活动的 Sandbox 是否按预期回收 |
| 重复请求 | 重复创建或删除是否幂等 |
| Controller 重启 | 重启后是否继续管理已有 Sandbox |
| 并发创建 | 1、10、30 个并发请求下的成功率与延迟 |
| 资源不足 | CPU、内存不足时是否明确失败并正确清理 |
| Pool 复用 | Pool 命中率、复用延迟和脏状态问题 |
| 异常 Sandbox | Sandbox 进程或 Pod 异常退出后的状态收敛 |
| API 中断 | 客户端断开后任务是否出现泄漏 |
| 多租户 | 不同租户和 Session 是否发生状态或数据串扰 |
| Controller 多副本 | 是否出现重复回收、重复创建或竞争处理 |

统一记录：

- 请求数量、成功数量、失败数量。
- 创建延迟 p50 / p95 / p99、删除延迟、超时误差。
- 残留 Pod 数量、状态不一致数量。
- Controller 重启后的恢复时间。
- Pool 命中率。

输出：

```text
tests/scenarios/
scripts/benchmark/
docs/BASELINE_REPORT.md
```

基线验证的意义：Multi-Agent MVP 的 Agent 调度层直接建立在 Sandbox 创建、超时回收和 Pool 复用之上，这些指标就是上层并发额度、超时和重试配置的输入。

---

## 六、项目的最小闭环

上层只部署一套很小的 Multi-Agent 仓库分析系统，用于证明 Sandbox 资源确实服务于真实 Agent 任务。

用户输入一个 GitHub 仓库地址，系统创建三个 Agent：

```text
Explorer Agent
- 拉取仓库。
- 分析目录、语言和核心模块。
- 输出 architecture.md。

Tester Agent
- 拉取同一仓库的独立副本。
- 安装依赖并运行测试。
- 输出 test-report.md。

Reviewer Agent
- 等待前两个 Agent 完成。
- 读取显式传递的两个产物。
- 生成 final-report.md。
```

每个 Agent 分别运行在独立 Sandbox Pod 中，不是在同一个 Python 进程里模拟多个角色。

执行顺序：

```text
创建任务
   ↓
Explorer 与 Tester 进入调度队列
   ↓
为两个 Agent 创建独立 Sandbox
   ↓
Kubernetes 调度两个 Pod
   ↓
保存两个 Agent 的产物
   ↓
回收前两个 Sandbox
   ↓
唤醒 Reviewer
   ↓
Reviewer 申请新 Sandbox
   ↓
生成最终报告
   ↓
回收全部资源
```

---

## 七、MVP 需要实现的能力

### 6.1 Agent 与 Sandbox 一一绑定

记录：

```text
task_id
agent_run_id
agent_role
sandbox_id
pod_name
resource_request
status
retry_count
```

### 6.2 最小 Agent 调度

只实现：

- 最大并发 Agent 数量。
- 先进先出等待队列。
- 上游 Agent 完成后唤醒下游 Agent。
- 不同角色配置不同 CPU 和内存。

需要区分：

- Agent 调度器决定哪个 Agent 现在运行。
- Kubernetes Scheduler 决定 Pod 放到哪个节点。

### 6.3 生命周期状态

```text
PENDING
→ WAITING_FOR_RESOURCE
→ CREATING_SANDBOX
→ RUNNING
→ SUCCEEDED / FAILED / TIMED_OUT
→ RECYCLING
→ RECYCLED
```

### 6.4 最小错误恢复

只实现三种：

- Sandbox 创建失败，重试一次。
- Agent 超时，终止并回收 Sandbox。
- Agent 执行失败，创建一个新的干净 Sandbox 重试一次。

### 6.5 ClawFleet 风格管理界面

在 Agent Sandbox 现有 UI 上增加 Task / AgentRun 页面，借鉴 ClawFleet 的 Agent 卡片和 Fleet 视图。

展示：

```text
Agent       状态       Sandbox       Pod       CPU/内存       重试
Explorer    Running    sandbox-a     pod-a     0.5C/512Mi     0
Tester      Success    sandbox-b     pod-b     1C/2Gi         1
Reviewer    Waiting    -             -         0.5C/512Mi     0
```

提供：

- 创建任务。
- 查看 Agent 状态。
- 查看 Sandbox / Pod 映射。
- 查看基础资源信息。
- 查看实时或轮询日志。
- 查看生命周期事件。
- 手动停止、重试和删除 AgentRun。
- 查看最终报告。

深度 Trace、失败归因和评测由 AgentLens 项目负责，本项目不重复实现。

---

## 八、一个月内明确不做

- 不把 ClawFleet 整体迁移到 Kubernetes。
- 不直接复制 ClawFleet 的完整资产、角色和渠道系统。
- 不重写 Kubernetes Scheduler。
- 不实现通用 Multi-Agent 工作流平台。
- 不做复杂 DAG 编辑器。
- 不做 Agent 深度可观测和自动归因。
- 不做 Prometheus、Grafana、Loki 全套系统。
- 不做 Kubernetes Operator 和 CRD。
- 不做 gVisor、Kata、Firecracker。
- 不做多租户和多集群。
- 不做 Agentic RL、训练和自进化。

这些能力可以作为后续路线，但不进入第一个可演示版本。

---

## 九、最终架构

```text
ClawFleet-style Web UI
  ├── Task / Agent Fleet View
  ├── Agent Cards
  ├── Status / Resource
  ├── Logs / Events
  └── Start / Stop / Retry / Delete
              ↓
Task / AgentRun API
              ↓
Minimal Agent Scheduler
  ├── 并发限制
  ├── 等待队列
  ├── 依赖唤醒
  ├── 超时控制
  └── 失败重试
              ↓
Agent Sandbox
  ├── Sandbox 创建与删除
  ├── 资源配置
  ├── Pool / 生命周期
  └── Kubernetes 接入
              ↓
Kubernetes
  ├── Pod 调度
  ├── CPU / 内存限制
  ├── Pod 状态管理
  └── 异常容器处理
              ↓
Explorer / Tester / Reviewer Agent Containers
```

---

## 十、项目表述

项目不应描述为：

> 给 Agent Sandbox 增加了一个 Multi-Agent 页面。

也不应描述为：

> 把 ClawFleet 从 Docker 改成了 Kubernetes。

建议表述为：

> 基于 Agent Sandbox 构建 Kubernetes 原生的 Multi-Agent Sandbox 执行系统，为每个 Agent 动态分配独立 Sandbox Pod，并实现 Agent 级并发控制、资源配置、依赖调度、生命周期管理、失败重试和自动回收；产品层借鉴 ClawFleet 的 Agent Fleet 管理模式，提供 Agent 实例、资源状态、运行日志与生命周期事件的统一管理界面，并通过多 Agent 仓库分析任务完成端到端验证。

这个项目在个人项目组合中的职责非常明确：

```text
Agent 应用项目：证明能够构建 Agent 工作流
AgentDocks：证明能够管理 Agent 的 Kubernetes Sandbox Runtime
AgentLens：证明能够观测、诊断和评测 Agent 执行过程
```
