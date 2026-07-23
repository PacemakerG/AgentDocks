# Linux、Shell 与 Kubernetes 基础概念

## 1. 终端、Shell 与 Linux 内核

一条命令的基本执行过程：

```text
键盘输入
→ 终端接收字符
→ Shell 解析命令
→ Linux 内核启动程序
→ 程序执行并输出结果
→ 终端显示结果
```

- **终端（Terminal）**：负责接收键盘输入和显示输出。
- **Shell**：命令解释器，例如 Bash，负责解析命令。
- **Linux 内核**：负责创建进程、分配 CPU 和内存、访问文件和网络。

---

## 2. stdin 与命令行参数

`stdin` 是标准输入，是程序接收输入的一条通道。

在交互式终端中：

```text
键盘
→ 终端
→ Bash 的 stdin
```

例如：

```bash
python run.py
```

其中：

- `python`：要运行的程序。
- `run.py`：传给 Python 的命令行参数。
- `run.py` 不是 stdin。

下面这种情况才是把内容传给程序的 stdin：

```bash
echo "hello" | python run.py
```

---

## 3. PATH 与命令查找

`PATH` 是一组用于查找可执行程序的目录，不是程序接口。

查看 PATH：

```bash
echo $PATH
```

可能得到：

```text
/usr/local/bin:/usr/bin:/bin
```

当输入：

```bash
kubectl
```

Shell 会依次查找：

```text
/usr/local/bin/kubectl
/usr/bin/kubectl
/bin/kubectl
```

找到第一个同名的可执行文件后，请求 Linux 内核启动它。

查看命令实际位置：

```bash
which kubectl
type kubectl
```

如果命令中已经写了路径，例如：

```bash
./run.sh
```

Shell 会直接执行当前目录中的文件，不再搜索 PATH。

---

## 4. `python run.py` 的执行过程

```text
Bash 找到 Python 可执行程序
→ Linux 内核启动 Python 进程
→ Python 读取 run.py
→ 解析 Python 语法
→ 编译成 Python 字节码
→ Python 虚拟机执行字节码
```

Shell 只负责启动 Python，不负责理解 Python 代码。

---

## 5. `kubectl get nodes` 的执行过程

```text
Bash 找到 kubectl
→ Linux 内核启动 kubectl
→ kubectl 解析 get nodes
→ kubectl 请求 Kubernetes API Server
→ API Server 返回节点数据
→ kubectl 格式化并显示结果
```

其中：

- `kubectl`：操作 Kubernetes 的命令行客户端。
- `get`：查询资源。
- `nodes`：查询集群节点。
- API Server：Kubernetes 的统一操作入口。

`kubectl` 不直接操作底层容器，而是通过 Kubernetes API 发出请求。

---

## 6. K3s 与 Kubernetes

- **Kubernetes（K8s）**：容器集群管理系统。
- **K3s**：轻量化的 Kubernetes 发行版。
- K3s 保留了 Kubernetes 的核心 API、Pod、Deployment、Service 和调度能力。

当前单节点环境：

```text
腾讯云服务器
├── Kubernetes 控制面
├── Kubernetes 工作节点
├── containerd 容器运行时
└── 业务 Pod
```

同一台服务器既负责管理集群，也负责运行 Pod。

---

## 7. Docker 容器的隔离方式

Docker 主要依赖 Linux 内核的两类机制。

### Namespace

用于隔离：

- 进程
- 网络
- 文件系统
- 主机名
- 用户
- 进程间通信

### Cgroup

用于限制：

- CPU
- 内存
- 磁盘 I/O
- 进程数量

容器和虚拟机的区别：

```text
虚拟机：拥有独立操作系统内核
容器：共享宿主机 Linux 内核
```

k3d 使用多个 Docker 容器模拟多个 Kubernetes 节点，但这些节点仍然共享同一台物理机器。

---

## 8. SSH 公钥认证

SSH 密钥由一对文件组成：

```text
~/.ssh/id_ed25519       私钥
~/.ssh/id_ed25519.pub   公钥
```

- 私钥保存在本地 Mac，必须保密。
- 公钥可以复制到远端服务器。
- 服务器将公钥保存到：

```text
~/.ssh/authorized_keys
```

登录过程：

```text
Mac 使用私钥证明身份
→ 服务器使用保存的公钥验证
→ 验证成功
→ 允许登录
```

服务器保存的是公钥，而不是 Mac 设备本身。

---

## 9. 整体关系

```text
Shell 负责解释命令
PATH 帮助 Shell 查找程序
Linux 内核负责运行程序
程序负责完成具体功能
kubectl 再通过 API 操作 Kubernetes
```
