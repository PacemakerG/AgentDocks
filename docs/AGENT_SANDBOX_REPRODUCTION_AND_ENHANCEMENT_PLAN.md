# AgentDocks：Agent-Sandbox 复现与生产化增强计划

> 基础项目：[agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox)  
> 当前目标：先复现、验证和理解现有项目，再根据真实缺口做生产化增强。  
> 本文取代原来的 MVP 与长期实施计划。

## 一、项目判断

AgentDocks 当前不应该从零开发新的 Sandbox Runtime，也不应该先构建离线 Coding 测试集或接入 CCWhat。

第一阶段只回答三个问题：

1. Agent-Sandbox 现在真实具备哪些能力？
2. 它在生命周期管理、可靠性和可观测性上存在哪些可复现的问题？
3. 哪些问题值得通过上游贡献或独立扩展解决？

因此，当前项目路线调整为：

```text
复现 Agent-Sandbox
        ↓
建立可重复的基础设施验证场景
        ↓
定位一个真实缺口
        ↓
实现生产化增强
        ↓
通过故障与并发实验验证
        ↓
优先向上游提交 PR
```

允许最终结论是“现有项目已经足够，不需要继续独立开发”。

---

## 二、为什么选择 Agent-Sandbox

Agent-Sandbox 已经提供：

- Kubernetes 原生部署。
- Sandbox 创建、访问、暂停、恢复和删除。
- 多租户、多 Session 和长时间运行环境。
- Sandbox Pool。
- REST API。
- MCP Server。
- E2B 协议和 SDK 兼容。
- Code、Shell、Browser、Computer 等运行环境。
- 内置 Web 管理界面 `/ui`。

相较 OpenSandbox，Agent-Sandbox 的成熟度和知名度更低，但它不是空白项目，已经具备完整主流程和可演示 UI；同时其公开 TODO 仍包含明确的生产化缺口：

- Metrics 导出。
- 资源预留。
- Leader Election。
- Sandbox Snapshot。
- 基于 Lease 的空闲超时。
- Telemetry 日志上报。

这比自己设想一个全新的 Runtime 更适合当前阶段：既能学习真实系统，也存在可以验证和贡献的改进空间。

但必须明确：

> 差异化不能来自“项目知名度低”，而要来自解决了一个真实、可复现、可量化的问题。

---

## 三、当前明确不做

第一阶段不做以下内容：

- 不构建离线 Coding Agent 评测集。
- 不接入 CCWhat 或 AgentLens。
- 不重新实现容器、Namespace、cgroups 或底层 Sandbox。
- 不先设计 Rust Worker 和 Go Control Plane。
- 不做 Prompt 优化和 Agent 自进化。
- 不做多 Agent DAG。
- 不做 MicroVM、Firecracker 或跨地域调度。
- 不宣称项目已经达到企业级或生产级。

测试对象是 **Sandbox 基础设施本身**，不是 Agent 能力。

---

## 四、仓库与上游关系

建议采用两个仓库分工：

### 1. 真正的上游 Fork

在 GitHub 上 Fork：

```text
agent-sandbox/agent-sandbox
        ↓
PacemakerG/agent-sandbox
```

源码修改、Feature Branch 和向上游提交的 PR 都放在该 Fork 中。

### 2. AgentDocks

当前仓库保留为个人项目入口，存放：

- 部署配置和脚本。
- 可重复验证场景。
- Benchmark 工具。
- 故障注入脚本。
- 架构阅读笔记。
- 复现报告。
- 改进前后对比数据。
- 不适合直接合入上游的扩展组件。

不要把上游源码复制到 AgentDocks 后包装成完全自研项目。README 和简历必须清楚区分：

```text
上游已有能力
个人实现的改动
个人完成的实验与结论
```

---

## 五、开发环境

## 5.1 Mac 本地环境

Mac 可以完成第一轮复现，不需要一开始就租云服务器。

推荐结构：

```text
macOS
  ↓
Docker Desktop / OrbStack
  ↓
Linux 虚拟机
  ↓
本地 Kubernetes
  ↓
Agent-Sandbox
```

Agent-Sandbox 要求 Kubernetes 1.26 及以上。优先使用以下一种方式：

- Docker Desktop 自带 Kubernetes。
- `kind`。
- `minikube`。

本地阶段用于：

- 跑通安装。
- 阅读源码。
- 调试 API 和 Controller。
- 验证基本生命周期。
- 开发单元测试和小规模集成测试。

Apple Silicon 可能遇到部分镜像缺少 ARM64 构建的问题。遇到镜像架构、网络插件或 RuntimeClass 问题时，不要在本地环境上耗费过多时间，直接迁移到 x86_64 Linux。

## 5.2 Linux 云服务器

以下工作开始前再租服务器：

- 并发和性能实验。
- Controller 高可用。
- Kubernetes 网络策略。
- 资源限制和资源不足实验。
- 多副本 Leader Election。
- 长时间稳定性测试。
- 需要 x86_64 镜像的场景。

建议配置：

```text
系统：Ubuntu 22.04 或 24.04，x86_64
CPU：推荐 8 核
内存：推荐 16 GB
磁盘：80～100 GB SSD
Kubernetes：单节点 k3s 起步
```

第一阶段不需要多节点集群。

部署到云服务器时：

- 必须更换默认管理 Token。
- 不直接向公网开放 `/ui`、API 和 Kubernetes 管理端口。
- 优先通过防火墙、VPN 或 SSH 端口转发访问。

---

## 六、阶段一：完整复现 Agent-Sandbox

目标周期：2～4 天。

### 6.1 安装和启动

完成：

1. 准备 Kubernetes 1.26+。
2. Fork 并 Clone 上游项目。
3. 按官方方式部署。
4. 确认所有核心 Pod 正常。
5. 通过 Port Forward 或 Ingress 打开 `/ui`。

基础命令：

```bash
kubectl create namespace agent-sandbox
kubectl apply -n agent-sandbox -f install.yaml
kubectl get pods -n agent-sandbox
kubectl get svc -n agent-sandbox
```

不要预先假设 Service 名称。根据实际部署结果执行 Port Forward。

### 6.2 跑通主流程

必须实际完成：

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

### 6.3 输出

新增：

```text
docs/REPRODUCTION_REPORT.md
```

内容必须包括：

- 操作系统、CPU 架构、Kubernetes 版本。
- 完整安装命令。
- 成功运行的功能。
- 没有跑通的功能。
- 实际架构图。
- 关键 Controller、API、Pool 和 Sandbox 生命周期代码入口。
- 本地环境遇到的问题。
- 与 README 宣传不一致的地方。

阶段一只要求事实，不提出大规模重构方案。

---

## 七、阶段二：建立基础设施验证场景

这里不叫“离线测试集”，而叫 **可重复基础设施场景**。

每个场景都必须可以通过脚本重复执行，并给出确定性结果。

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

第一版统一记录：

- 请求数量。
- 成功数量。
- 失败数量。
- 创建延迟 p50 / p95 / p99。
- 删除延迟。
- 超时误差。
- 残留 Pod 数量。
- 状态不一致数量。
- Controller 重启后的恢复时间。
- Pool 命中率。

输出：

```text
tests/scenarios/
scripts/benchmark/
docs/BASELINE_REPORT.md
```

阶段二结束后才能决定具体增强方向。

---

## 八、第一条推荐增强主线

推荐优先选择：

> **基于 Kubernetes Lease 的生命周期治理 + Prometheus 可观测性。**

这条主线优先于重写 Runtime、增加新 Agent 或开发复杂前端。

原因：

- 上游 TODO 已明确列出。
- 属于真实的 Kubernetes 控制面问题。
- 可以通过 Controller 重启、并发和超时实验验证。
- 能同时体现系统设计、云原生、可靠性和可观测性能力。
- 工作量可控制，不需要先构建 Agent 评测体系。

## 8.1 Lease 生命周期治理

目标：用 Kubernetes Lease 或等价的持久化机制记录 Sandbox 活跃状态，避免仅依赖单进程内存、定时扫描或易丢失事件。

需要解决：

- 活跃状态如何续期。
- 谁负责续期。
- Controller 重启后如何恢复判断。
- 多 Controller 如何避免重复回收。
- 删除操作如何幂等。
- 续期请求乱序或延迟时如何处理。
- 长时间无活动时如何确定过期。

建议状态：

```text
CREATING
  → RUNNING
  → IDLE
  → EXPIRED
  → DELETING
  → DELETED
```

验收：

- Controller 重启不会让所有计时重新开始。
- 多副本不会重复删除同一个 Sandbox。
- 超时误差有明确上限。
- 删除失败可以重试并最终收敛。
- 重复续期和重复删除保持幂等。

## 8.2 Prometheus Metrics

至少增加：

```text
agent_sandbox_create_total
agent_sandbox_create_failures_total
agent_sandbox_delete_total
agent_sandbox_active
agent_sandbox_pool_size
agent_sandbox_pool_hits_total
agent_sandbox_expired_total
agent_sandbox_reconcile_total
agent_sandbox_reconcile_errors_total
agent_sandbox_create_duration_seconds
agent_sandbox_delete_duration_seconds
agent_sandbox_recovery_duration_seconds
```

要求：

- Counter、Gauge 和 Histogram 语义正确。
- 标签数量受控，禁止把 Sandbox ID、用户 ID 直接作为高基数标签。
- Metrics 能说明生命周期问题，而不是只显示进程 CPU 和内存。
- 提供最小 Grafana Dashboard 或 Prometheus 查询示例。

## 8.3 改进验证

改进前后使用同一组基础设施场景，比较：

- Controller 重启后的状态恢复时间。
- 超时回收误差。
- 多副本重复操作次数。
- 残留资源数量。
- 并发创建失败率。
- Reconcile 错误率。

没有前后对比数据，不能声称改进有效。

---

## 九、后续候选方向

完成第一条主线后，再根据真实结果选择一个方向，不同时铺开。

### 方向 A：Leader Election 与控制面高可用

进入条件：

- 单副本生命周期逻辑已经稳定。
- 多副本确实出现重复处理或竞争。
- Lease 机制已经可以支撑主节点选举。

验证：

- 主 Controller 被杀死后的切换时间。
- 切换期间请求失败率。
- 是否发生重复创建或删除。
- 新主节点是否恢复已有状态。

### 方向 B：资源预留与准入控制

进入条件：

- 已复现资源不足导致的创建失败或资源争抢。
- Sandbox Pool 与动态创建存在明确容量问题。

验证：

- 资源预留准确度。
- 过量提交时的拒绝行为。
- 不同租户之间的公平性。
- Pool 命中率和资源浪费之间的关系。

### 方向 C：Snapshot 与恢复

进入条件：

- 暂停恢复不能满足真实状态保存需求。
- 已确认底层存储与运行环境支持可实现的快照方案。

不要仅为了补齐 TODO 实现一个没有真实恢复语义的“打包目录”。

### 方向 D：Telemetry 与审计

进入条件：

- 已明确哪些生命周期和安全事件需要长期保存。
- Metrics 无法满足问题定位和审计需求。

后续与 AgentLens 的集成也只能在该阶段重新评估，不提前绑定。

---

## 十、前端策略

Agent-Sandbox 已有管理 UI，第一阶段不重新开发前端。

只在以下情况修改 UI：

- 新增的生命周期状态无法展示。
- 需要展示 Pool、Lease 或 Controller 健康状态。
- 需要为真实故障提供诊断入口。
- 上游 UI 存在可复现 Bug。

第一版可观测性优先使用 Prometheus 和 Grafana，不在现有 UI 中重复实现完整监控系统。

---

## 十一、提交与贡献方式

每项改动遵循：

```text
复现问题
  → 编写失败测试
  → 提出最小设计
  → 实现修复
  → 运行回归与并发实验
  → 更新文档
  → 向上游提交 PR
```

PR 必须说明：

- 问题如何复现。
- 现有实现为什么不足。
- 修改了哪些行为。
- 是否影响兼容性。
- 如何测试。
- 性能或可靠性数据。

优先将通用能力贡献到上游。只有与个人项目强相关、上游明确不接受的能力，才保留在 AgentDocks 中。

---

## 十二、建议时间安排

### 第 1 周：复现和源码理解

- Mac 本地部署。
- 跑通 UI、REST、MCP、E2B SDK。
- 理解 Sandbox、Pool、Template 和 Timeout 主流程。
- 输出复现报告。

### 第 2 周：基线验证

- 编写生命周期场景脚本。
- 测试超时、空闲回收、重启、重复请求和并发创建。
- 输出基线报告。
- 确认第一条真实缺口。

### 第 3 周：实现一项生产化增强

- 优先实现 Lease 生命周期治理或 Metrics。
- 补充单元测试和集成测试。
- 在 Linux 云服务器上进行正式实验。

### 第 4 周：验证和上游贡献

- 完成改进前后对比。
- 补充 Grafana Dashboard 或查询示例。
- 完善设计文档。
- 提交上游 PR。
- 录制五分钟演示。

如果第二周没有发现值得开发的缺口，就停止独立实现，转为研究和贡献上游已有 Issue。

---

## 十三、MVP 验收标准

当前 MVP 不是“自己做出一套 Sandbox 平台”，而是完成以下内容：

- [ ] 在 Mac 本地完整部署 Agent-Sandbox。
- [ ] 跑通 UI、REST、MCP 和 E2B SDK 主流程。
- [ ] 输出可复现的安装与架构报告。
- [ ] 建立至少 8 个基础设施验证场景。
- [ ] 在 x86_64 Linux 上完成正式实验。
- [ ] 找到一个有证据的生命周期、可靠性或可观测性缺口。
- [ ] 为该问题编写失败测试。
- [ ] 完成一项生产化增强。
- [ ] 提供修改前后的量化数据。
- [ ] 向上游提交 Issue 或 PR。
- [ ] 文档清楚区分上游能力和个人贡献。

以下内容不属于当前 MVP：

- Coding Agent 离线测试集。
- CCWhat 集成。
- Rust Sandbox 自研。
- Go 调度平台自研。
- 多 Agent 工作流。
- 新的复杂管理前端。

---

## 十四、最终项目叙事

不要描述成：

> 从零实现企业级 Agent Sandbox 平台。

建议描述为：

> 基于 Kubernetes 原生开源 Agent Sandbox 平台完成源码级复现与生产化增强，围绕 Sandbox 生命周期治理、空闲回收和控制面可观测性建立可重复故障场景；通过 Lease、Prometheus Metrics 和幂等 Reconcile 改进 Controller 重启及多副本环境下的状态一致性，并以并发、超时和故障实验验证可靠性收益。

项目价值取决于：

```text
真实复现
> 找到真实问题
> 修复可以验证
> 改动可以贡献
> 架构能够解释
> 功能数量和技术栈数量
```
