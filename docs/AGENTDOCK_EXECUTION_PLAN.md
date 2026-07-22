# AgentDock 执行阶段

> 只列阶段、目标、产物、验收标准，不谈架构、选型、开源参考。

---

## 阶段 0：建立最小执行协议

定义 Task 请求/事件 JSON Schema。

**产物：**
- `proto/task.proto` — Task 请求、事件类型定义
- 事件类型：`process_started` / `stdout` / `stderr` / `resource_sample` / `file_changed` / `policy_denied` / `process_exited`

**验收：**
- JSON Schema 可以描述一次完整的 Agent 执行过程

---

## 阶段 1：Rust 单机 Worker

在一台 Linux 机器上可靠运行和回收 Agent 任务。

**能力：**
- Tokio 并发执行多任务
- 完整进程树管理（启动、监控、杀死）
- stdout / stderr 流式输出
- 超时自动终止 + 主动取消
- 独立 Workspace 隔离 + Git Diff
- cgroups v2：CPU、内存、PID、IO 限制
- 基础文件权限（禁止读取 `~/.ssh` 等敏感目录）
- 安全/压力测试（Fork Bomb、OOM、无限输出）

**CLI：**
```bash
agentdock run --workspace ./repo --memory 2g --cpu 1 --network deny --timeout 300 -- codex exec "fix tests"
```

**产物：**
- `worker/` — Rust Workspace
- `tests/security/` / `tests/failure/` / `tests/benchmark/`

**验收：**
- 取消后无残留子进程
- 超时确定性退出
- 无法读取宿主机 SSH Key
- Fork Bomb 不拖垮宿主机
- 多任务独立运行、独立返回日志
- 输出完整 Git Diff 和资源统计

---

## 阶段 2：Go Control Plane

从"能运行一个任务"升级到"能管理一批任务"。

**能力：**
- Task API（REST / gRPC）
- Worker 注册/注销 + Heartbeat + Lease
- Task Queue + 并发额度 + 租户配额
- Worker 崩溃后任务重新分配
- 幂等提交 + 超时/取消 + 有限重试
- SQLite 状态存储

**状态机：**
```
PENDING → ASSIGNED → STARTING → RUNNING → SUCCEEDED / FAILED / CANCELLED / TIMED_OUT
```
Task 与 Attempt 分离：一个 Task 可以有多次 Attempt（Worker 崩溃、超时重试）。

**产物：**
- `control-plane/` — Go Module

**验收：**
- Worker 崩溃后任务自动迁移到其他 Worker
- 取消请求幂等
- 配额超限时排队而非直接失败

---

## 阶段 3：接入 AgentLens

统一标识串联执行和观测。

**标识体系：**
```
session_id → task_id → attempt_id → agent_id → sandbox_id → process_id → tool_call_id
```

**AgentLens 新增：**
- Sandbox 生命周期
- 任务调度/重试过程
- 进程树 + CPU/内存/PID 曲线
- 文件变化 + 网络访问 + Policy Denial
- 退出原因 + Worker 崩溃与任务迁移

**产物：**
- AgentLens 侧新增 Sandbox/Scheduling View

**验收：**
- 一次 Agent 执行的全链路 Trace 在 AgentLens 中可视化

---

## 阶段 4：多 Agent DAG（可选）

**能力：**
- Planner / Executor / Reviewer / Tester 四个角色
- 每个 Agent 独立 Sandbox + 独立权限
- Artifact 显式传递（不共享 Workspace）
- 节点级重试 + 独立取消

**执行流程：**
```
Planner Sandbox → 输出 plan.json
                     ↓
Executor Sandbox → 输出 diff
                     ↓
Reviewer Sandbox（只读代码+diff）+ Tester Sandbox（只允许跑测试）
```

**验收：**
- DAG 中任一节点失败可单独重试，不污染其他节点 Sandbox

---

## 前置依赖：Agent Sandbox 底座复现与基线验证

> 阶段 0-4 开始前，必须先跑通并验证底座。

### P0：复现 Agent Sandbox（2-4 天）

跑通：UI 登录、创建/暂停/恢复/删除 Sandbox、Shell/Terminal、文件读写、代码执行、超时回收、REST API、MCP、Pool 创建复用。

**产物：** `docs/REPRODUCTION_REPORT.md`

### P1：基线验证（12 个场景）

| # | 场景 | 验证点 |
|---|---|---|
| 1 | 正常生命周期 | 创建→暂停→恢复→删除 |
| 2 | 超时回收 | TTL 到期后及时删除 |
| 3 | 空闲回收 | 无活动 Sandbox 按时回收 |
| 4 | 重复请求 | 创建/删除幂等 |
| 5 | Controller 重启 | 重启后继续管理已有 Sandbox |
| 6 | 并发创建 | 1/10/30 并发成功率与延迟 |
| 7 | 资源不足 | CPU/内存不足时明确失败+清理 |
| 8 | Pool 复用 | 命中率、复用延迟、脏状态 |
| 9 | 异常 Sandbox | Pod 异常退出后状态收敛 |
| 10 | API 中断 | 客户端断开后无泄漏 |
| 11 | 多租户 | 不同租户无状态/数据串扰 |
| 12 | Controller 多副本 | 无重复回收/创建/竞争 |

每个场景记录：请求数、成功/失败数、p50/p95/p99 延迟、残留 Pod、状态不一致数。

**产物：** `tests/scenarios/` / `scripts/benchmark/` / `docs/BASELINE_REPORT.md`

---

## 推荐执行顺序

```
P0 复现底座 → P1 基线验证 → 阶段 0 协议 → 阶段 1 Rust Worker → 阶段 2 Go 控制面 → 阶段 3 AgentLens 接入 → （阶段 4 多 Agent）
```

每个阶段完成后必须有：可运行代码 + 测试 + Benchmark 数据 + 故障案例。
