# AgentDocks：基于 Agent Sandbox 的 Multi-Agent Sandbox MVP 决策

> 决策日期：2026 年 7 月  
> 基础项目：[agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox)  
> 参考项目：[clawfleet/ClawFleet](https://github.com/clawfleet/ClawFleet)

## 一、最终结论

本项目应当：

> **以 Agent Sandbox 为底座，在其 Kubernetes Sandbox 管理能力之上部署一套最小 Multi-Agent 系统。**

不建议以 ClawFleet 为底座，再把其 Docker 管理层整体改造成 Kubernetes。

ClawFleet 只用于借鉴：

- 一个 Agent 对应一个独立运行实例的产品表达。
- Agent 卡片、状态、资源、日志和生命周期操作界面。
- 多个 Agent 的统一管理视图。

真正的基础设施底座仍然使用 Agent Sandbox：

- Go 后端。
- Kubernetes 原生部署。
- Sandbox 创建、暂停、恢复和删除。
- CPU、内存等资源配置。
- Sandbox Pool 和超时回收。
- REST API 与现有管理界面。

E2B 兼容不是项目主线。MVP 优先使用 Agent Sandbox 自身的 API 和内部管理能力，E2B 接口仅保留为可选兼容层。

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

在这个基础上增加 Multi-Agent，只需要补充 Agent 语义和任务调度，而不用重新实现 Kubernetes 接入层。

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

## 三、为什么不以 ClawFleet 为底座

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

这不是“小改造”，而是把 ClawFleet 的底层执行系统重新写一遍。一个月内投入产出比不高，并且会重复 Agent Sandbox 已经完成的能力。

因此，ClawFleet 应作为产品和交互参考，而不是代码底座。

---

## 四、项目的最小闭环

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
Explorer 与 Tester 并行申请 Sandbox
   ↓
Kubernetes 调度两个 Pod
   ↓
保存两个 Agent 的产物
   ↓
回收前两个 Sandbox
   ↓
Reviewer 申请新 Sandbox
   ↓
生成最终报告
   ↓
回收全部资源
```

---

## 五、MVP 需要实现的能力

### 1. Agent 与 Sandbox 一一绑定

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

### 2. 最小 Agent 调度

只实现：

- 最大并发 Agent 数量。
- 先进先出等待队列。
- 上游 Agent 完成后唤醒下游 Agent。
- 不同角色配置不同 CPU 和内存。

需要区分：

- Agent 调度器决定哪个 Agent 现在运行。
- Kubernetes Scheduler 决定 Pod 放到哪个节点。

### 3. 生命周期状态

```text
PENDING
→ WAITING_FOR_RESOURCE
→ CREATING_SANDBOX
→ RUNNING
→ SUCCEEDED / FAILED / TIMED_OUT
→ RECYCLING
→ RECYCLED
```

### 4. 最小错误恢复

只实现三种：

- Sandbox 创建失败，重试一次。
- Agent 超时，终止并回收 Sandbox。
- Agent 执行失败，创建一个新的干净 Sandbox 重试一次。

### 5. 管理界面

在 Agent Sandbox 现有 UI 上增加一个任务页面，展示：

```text
Agent       状态       Sandbox       Pod       CPU/内存       重试
Explorer    Running    sandbox-a     pod-a     0.5C/512Mi     0
Tester      Success    sandbox-b     pod-b     1C/2Gi         1
Reviewer    Waiting    -             -         0.5C/512Mi     0
```

提供：

- 创建任务。
- 查看 Agent 状态。
- 查看基础运行日志。
- 手动取消任务。
- 查看最终报告。

深度 Trace、失败归因和评测由 AgentLens 项目负责，本项目不重复实现。

---

## 六、一个月内明确不做

- 不把 ClawFleet 整体迁移到 Kubernetes。
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

## 七、最终架构

```text
Web UI
  ↓
Task / AgentRun API
  ↓
Minimal Agent Scheduler
  ├── 并发限制
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

## 八、项目表述

项目不应描述为：

> 给 Agent Sandbox 增加了一个 Multi-Agent 页面。

也不应描述为：

> 把 ClawFleet 从 Docker 改成了 Kubernetes。

建议表述为：

> 基于 Agent Sandbox 构建 Kubernetes 原生的 Multi-Agent Sandbox 执行系统，为每个 Agent 动态分配独立 Sandbox Pod，并实现 Agent 级并发控制、资源配置、依赖调度、生命周期管理、失败重试和自动回收；通过多 Agent 仓库分析任务完成端到端验证。

这个项目在个人项目组合中的职责非常明确：

```text
Agent 应用项目：证明能够构建 Agent 工作流
AgentDocks：证明能够管理 Agent 的 Kubernetes Sandbox Runtime
AgentLens：证明能够观测、诊断和评测 Agent 执行过程
```
