# Kubernetes CPU Manager strict-cpu-reservation 源码解析

本文记录 kubelet `reservedSystemCPUs` 与 `strict-cpu-reservation` 的行为差异和源码实现。重点不是配置说明，而是从 kubelet CPU Manager 的状态构建、容器创建、CRI 传递和 cgroup 写入链路解释为什么旧行为下普通 Pod 仍可能使用 reserved CPU，以及 `strict-cpu-reservation` 如何修正这个问题。

> 说明：本文基于 Kubernetes 上游公开源码进行整理，文件路径以 Kubernetes 仓库为准。不同版本的代码细节可能略有变化，但核心链路和设计意图一致。

## 0. 参考版本

本文源码对照版本为 Kubernetes `v1.34.10` tag，对应 release commit `bd6c4ad1`。分析时已切换到该 tag 的 detached HEAD 状态确认关键代码路径，包括 `pkg/kubelet/cm/cpumanager/policy_static.go` 中 `validateState()` 的 `allCPUs.Difference(p.reservedCPUs)` 逻辑、`pkg/kubelet/cm/cpumanager/policy_options.go` 中 `StrictCPUReservationOption` 的解析，以及 `pkg/kubelet/cm/internal_container_lifecycle_linux.go` 中 `PreCreateContainer()` 写入 `containerConfig.Linux.Resources.CpusetCpus` 的链路。

`strict-cpu-reservation` 最早在 Kubernetes `v1.32` 作为 alpha 特性引入，使用时需要同时配置 `cpuManagerPolicyOptions.strict-cpu-reservation: "true"`，并开启 `CPUManagerPolicyAlphaOptions` feature gate。到后续版本后，该选项的 feature gate 阶段会随版本演进变化；本文使用 `v1.34.10` tag 做源码对照，是为了固定代码位置和字段名称，避免直接引用 master 分支造成行号或实现细节漂移。该特性的专门增强提案可以跟踪 KEP-4540：`https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/4540-strict-cpu-reservation`；CPU Manager 相关背景也可以参考 KEP-3570：`https://github.com/kubernetes/enhancements/issues/3570`。

```bash
# 示例：切换到本文参考的 Kubernetes 版本。
git fetch origin tag v1.34.10
git checkout --detach v1.34.10

# 示例：确认当前工作区正处于 v1.34.10 tag。
git describe --tags --exact-match
git rev-parse --short HEAD
```

## 1. 问题背景

`reservedSystemCPUs` 的目标是为操作系统守护进程、kubelet、容器运行时、中断处理等系统侧负载预留一组 CPU。直觉上，配置了 `reservedSystemCPUs: "0-1"` 后，用户往往会期待普通业务 Pod 不再使用 CPU 0 和 CPU 1。

但在历史实现中，这个预期并不总是成立。原因是 kubelet CPU Manager 在 `static` 策略下主要维护两类 CPU 集合：一类是分配给 Guaranteed 且整数 CPU 请求容器的独占 CPU，另一类是普通容器共享使用的 `defaultCPUSet`。旧逻辑可以让独占 CPU 分配避开 reserved CPU，但共享池 `defaultCPUSet` 仍可能包含 reserved CPU。因此 BestEffort、Burstable，以及 Guaranteed 但非整数 CPU 请求的容器，仍可能通过共享池运行到 reserved CPU 上。

`strict-cpu-reservation` 修复的正是这个共享池边界问题：开启后，kubelet 会在构建 `defaultCPUSet` 时把 `reservedCPUs` 从共享池中扣除，使没有独占 assignment 的普通容器也无法继承到 reserved CPU。

## 2. CPU Manager 相关源码入口

CPU Manager 的主入口在：

```text
# kubelet CPU Manager 主入口
pkg/kubelet/cm/cpumanager/cpu_manager.go
```

关键函数包括：

```go
// 创建 CPU Manager，根据 cpuManagerPolicy 选择 none 或 static policy。
func NewManager(...)

// 启动 CPU Manager，初始化 checkpoint state，并调用 policy.Start()。
func (m *manager) Start(...)

// Pod/Container admission 阶段调用，用于为符合条件的容器分配独占 CPU。
func (m *manager) Allocate(...)

// 周期性对已运行容器的 cpuset 做纠偏。
func (m *manager) reconcileState(...)

// 容器创建前由 internal lifecycle hook 调用，用于返回容器应使用的 cpuset。
func (m *manager) GetCPUAffinity(podUID, containerName string) cpuset.CPUSet
```

当 kubelet 配置为 `cpuManagerPolicy: static` 时，`NewManager()` 会读取 `nodeAllocatableReservation[v1.ResourceCPU]`，计算需要预留的 CPU 数量，然后创建 static policy。

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager.go
// 作用：static policy 初始化时，把 kube/system reserved CPU 数量转换为 numReservedCPUs。
case PolicyStatic:
    reservedCPUs, ok := nodeAllocatableReservation[v1.ResourceCPU]
    if !ok {
        return nil, fmt.Errorf("[cpumanager] unable to determine reserved CPU resources for static policy")
    }

    // fractional CPU 不能独占分配，因此向上取整。
    reservedCPUsFloat := float64(reservedCPUs.MilliValue()) / 1000
    numReservedCPUs := int(math.Ceil(reservedCPUsFloat))

    // specificCPUs 对应显式指定的 reservedSystemCPUs。
    policy, err = NewStaticPolicy(logger, topo, numReservedCPUs, specificCPUs, affinity, cpuPolicyOptions)
```

static policy 的实现位于：

```text
# static CPU Manager policy
pkg/kubelet/cm/cpumanager/policy_static.go
```

其核心结构如下：

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：记录 CPU 拓扑、reserved CPU、policy options 等 static policy 状态。
type staticPolicy struct {
    // 节点 CPU 拓扑信息。
    topology *topology.CPUTopology

    // 不允许用于独占 CPU 分配的 reserved CPU 集合。
    reservedCPUs cpuset.CPUSet

    // reservedCPUs 所在物理核的 sibling CPU 超集，用于 SMT 相关策略。
    reservedPhysicalCPUs cpuset.CPUSet

    // Topology Manager affinity store，用于按 NUMA/socket hint 分配 CPU。
    affinity topologymanager.Store

    // init container 复用 CPU 时使用的临时集合。
    cpusToReuse map[string]cpuset.CPUSet

    // CPU Manager static policy options，包括 strict-cpu-reservation。
    options StaticPolicyOptions

    // 每个物理核包含的逻辑 CPU 数量。
    cpuGroupSize int
}
```

`NewStaticPolicy()` 会根据显式配置或自动选择结果计算 `reservedCPUs`。

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：如果用户显式配置 reservedSystemCPUs，则直接作为 reserved set；否则按拓扑选择。
allCPUs := topology.CPUDetails.CPUs()
var reserved cpuset.CPUSet
if reservedCPUs.Size() > 0 {
    // reservedCPUs 来自 reservedSystemCPUs / --reserved-cpus。
    reserved = reservedCPUs
} else {
    // 没有显式 CPU 列表时，根据 numReservedCPUs 从拓扑中选取。
    reserved, _ = policy.takeByTopology(logger, allCPUs, numReservedCPUs)
}

policy.reservedCPUs = reserved
```

CPU Manager 启动时，`manager.Start()` 会创建 checkpoint state 并调用 policy 的 `Start()`。

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager.go
// 作用：加载 /var/lib/kubelet/cpu_manager_state，并启动 policy。
stateImpl, err := state.NewCheckpointState(logger, m.stateFileDirectory, cpuManagerStateFileName, m.policy.Name(), m.containerMap)
m.state = stateImpl

err = m.policy.Start(logger, m.state)
```

对于 static policy，`Start()` 会进入 `validateState()`，这正是 `defaultCPUSet` 初始化和校验的核心位置。

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：启动 static policy，并校验或初始化 CPU Manager state。
func (p *staticPolicy) Start(logger klog.Logger, s state.State) error {
    if err := p.validateState(logger, s); err != nil {
        return err
    }
    p.initializeMetrics(logger, s)
    return nil
}
```

## 3. defaultCPUSet 的构建流程

CPU Manager state 的内存结构在：

```text
# CPU Manager state 内存实现
pkg/kubelet/cm/cpumanager/state/state_mem.go
```

核心字段是 `assignments` 和 `defaultCPUSet`。

```go
// 文件：pkg/kubelet/cm/cpumanager/state/state_mem.go
// 作用：保存容器独占 CPU assignment，以及共享 CPU 池 defaultCPUSet。
type stateMemory struct {
    assignments    ContainerCPUAssignments
    baselines      ContainerCPUBaselines
    podAssignments PodCPUAssignments
    defaultCPUSet  cpuset.CPUSet
}
```

`assignments` 表示已经分配给特定容器的独占 CPU；`defaultCPUSet` 表示没有独占 CPU assignment 的容器共享使用的 CPU 池。关键函数是 `GetCPUSetOrDefault()`。

```go
// 文件：pkg/kubelet/cm/cpumanager/state/state_mem.go
// 作用：如果容器有独占 CPU assignment，则返回该 assignment；否则返回共享池 defaultCPUSet。
func (s *stateMemory) GetCPUSetOrDefault(podUID string, containerName string) cpuset.CPUSet {
    if res, ok := s.GetCPUSet(podUID, containerName); ok {
        return res
    }
    return s.GetDefaultCPUSet()
}
```

这段代码解释了普通 Pod 为什么会受 `defaultCPUSet` 影响。BestEffort、Burstable、非整数 CPU 请求的 Guaranteed 容器通常不会写入 `assignments`，所以最终会继承 `defaultCPUSet`。

`defaultCPUSet` 的初始化发生在 `policy_static.go` 的 `validateState()` 中。

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：初始化或校验 CPU Manager state，其中 defaultCPUSet 是关键共享池。
func (p *staticPolicy) validateState(logger klog.Logger, s state.State) error {
    tmpAssignments := s.GetCPUAssignments()
    tmpDefaultCPUset := s.GetDefaultCPUSet()

    // 从 CPU 拓扑中获取节点所有 CPU。
    allCPUs := p.topology.CPUDetails.CPUs()

    // 开启 strict-cpu-reservation 时，从共享池候选集合中扣除 reserved CPU。
    if p.options.StrictCPUReservation {
        allCPUs = allCPUs.Difference(p.reservedCPUs)
    }

    // checkpoint 中没有 defaultCPUSet 时，说明状态需要初始化。
    if tmpDefaultCPUset.IsEmpty() {
        if len(tmpAssignments) != 0 {
            return fmt.Errorf("default cpuset cannot be empty")
        }

        // 用计算后的 allCPUs 初始化 defaultCPUSet。
        s.SetDefaultCPUSet(allCPUs)
        logger.Info("Static policy initialized", "defaultCPUSet", allCPUs)
        return nil
    }

    // 后续逻辑用于校验已有 checkpoint。
    return nil
}
```

因此，`defaultCPUSet` 的初始化流程可以概括为：

1. `topology.Discover()` 发现节点 CPU 拓扑。
2. `NewStaticPolicy()` 计算 `reservedCPUs`。
3. `manager.Start()` 加载 CPU Manager checkpoint。
4. `staticPolicy.Start()` 调用 `validateState()`。
5. 如果 checkpoint 里 `defaultCPUSet` 为空，则用 `allCPUs` 初始化。
6. 如果开启 strict，则 `allCPUs` 已经提前执行过 `Difference(p.reservedCPUs)`。

## 4. strict-cpu-reservation 的 diff 级说明

`strict-cpu-reservation` 的选项定义在：

```text
# CPU Manager policy options
pkg/kubelet/cm/cpumanager/policy_options.go
```

关键定义如下：

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_options.go
// 作用：定义 CPU Manager static policy 支持的选项名称。
const (
    StrictCPUReservationOption string = "strict-cpu-reservation"
)

// 作用：解析后的 static policy option 结构。
type StaticPolicyOptions struct {
    // 开启后，从 available/default CPU 集合中移除 reserved CPU。
    StrictCPUReservation bool
}
```

解析逻辑如下：

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_options.go
// 作用：把用户配置的 strict-cpu-reservation=true 解析为布尔字段。
case StrictCPUReservationOption:
    optValue, err := strconv.ParseBool(value)
    if err != nil {
        return opts, fmt.Errorf("bad value for option %q: %w", name, err)
    }
    opts.StrictCPUReservation = optValue
```

从 diff 视角看，最核心的变化就是在 `validateState()` 中初始化 `defaultCPUSet` 前多了一个差集操作。

```diff
 func (p *staticPolicy) validateState(logger klog.Logger, s state.State) error {
     tmpAssignments := s.GetCPUAssignments()
     tmpDefaultCPUset := s.GetDefaultCPUSet()

     allCPUs := p.topology.CPUDetails.CPUs()
+    if p.options.StrictCPUReservation {
+        allCPUs = allCPUs.Difference(p.reservedCPUs)
+    }

     if tmpDefaultCPUset.IsEmpty() {
         if len(tmpAssignments) != 0 {
             return fmt.Errorf("default cpuset cannot be empty")
         }
         s.SetDefaultCPUSet(allCPUs)
         logger.Info("Static policy initialized", "defaultCPUSet", allCPUs)
         return nil
     }
 }
```

假设节点 CPU 是 `0-7`，配置如下：

```yaml
# kubelet 配置示例：开启 static CPU Manager，并预留 CPU 0-1。
cpuManagerPolicy: static
reservedSystemCPUs: "0-1"
cpuManagerPolicyOptions:
  # 开启后，defaultCPUSet 会排除 reservedSystemCPUs。
  strict-cpu-reservation: "true"
```

那么开启前后的共享池变化如下：

```text
# 未开启 strict-cpu-reservation 时：
# defaultCPUSet 可能仍包含 reserved CPU。
allCPUs       = 0-7
reservedCPUs  = 0-1
defaultCPUSet = 0-7

# 开启 strict-cpu-reservation 后：
# defaultCPUSet 会扣除 reserved CPU。
allCPUs       = 0-7
reservedCPUs  = 0-1
defaultCPUSet = 2-7
```

checkpoint 已经存在时，校验规则也发生了反转。旧逻辑要求 `reservedCPUs` 必须包含在 `defaultCPUSet` 中；strict 逻辑要求 `reservedCPUs` 不能出现在 `defaultCPUSet` 中。

```diff
- if !p.reservedCPUs.Intersection(tmpDefaultCPUset).Equals(p.reservedCPUs) {
-     return fmt.Errorf("not all reserved cpus are present in defaultCpuSet")
- }

+ if p.options.StrictCPUReservation {
+     if !p.reservedCPUs.Intersection(tmpDefaultCPUset).IsEmpty() {
+         return fmt.Errorf("some of strictly reserved cpus are present in defaultCpuSet")
+     }
+ } else {
+     if !p.reservedCPUs.Intersection(tmpDefaultCPUset).Equals(p.reservedCPUs) {
+         return fmt.Errorf("not all reserved cpus are present in defaultCpuSet")
+     }
+ }
```

这段校验揭示了旧行为的根因：旧逻辑把 reserved CPU 仍视为共享池的一部分，只是在独占 CPU 分配时额外排除；strict 逻辑则把 reserved CPU 从共享池不变量中直接移除。

| 模式 | `defaultCPUSet` 不变量 | 普通 Pod 行为 |
| --- | --- | --- |
| 未开启 strict | `reservedCPUs` 必须包含在 `defaultCPUSet` 中 | BestEffort、Burstable 等共享池容器可能使用 reserved CPU |
| 开启 strict | `reservedCPUs` 不能出现在 `defaultCPUSet` 中 | 没有独占 assignment 的普通容器继承裁剪后的共享池 |

独占 CPU 分配链路仍然走 `allocateForAdd()` 和 `allocateCPUs()`。当容器满足 Guaranteed 且整数 CPU 请求时，`allocateForAdd()` 会为该容器写入独占 assignment。

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：为符合条件的 Guaranteed 容器分配独占 CPU。
numCPUs := p.guaranteedCPUs(logger, pod, container)
if numCPUs == 0 {
    // 不满足独占 CPU 条件，容器继续使用 defaultCPUSet。
    return nil
}

// 根据 Topology Manager hint 和当前可用 CPU 分配独占 CPU。
cpuAllocation, err := p.allocateCPUs(logger, s, numCPUs, hint.NUMANodeAffinity, p.cpusToReuse[string(pod.UID)])

// 把独占 CPU assignment 写入 state。
s.SetCPUSet(string(pod.UID), container.Name, cpuAllocation.CPUs)
```

`allocateCPUs()` 分配完成后，会把分配出去的 CPU 从共享池移除。

```go
// 文件：pkg/kubelet/cm/cpumanager/policy_static.go
// 作用：独占 CPU 分配完成后，从 defaultCPUSet 中移除这些 CPU。
s.SetDefaultCPUSet(s.GetDefaultCPUSet().Difference(result.CPUs))
```

因此，`strict-cpu-reservation` 并没有替换独占 CPU 分配算法，而是改变了共享 CPU 池的初始边界。旧逻辑下，独占 CPU 分配已经可以避开 reserved CPU；真正的问题在于普通容器继承的 `defaultCPUSet` 仍包含 reserved CPU。

## 5. 容器创建时 cpuset 的完整链路

容器创建入口在：

```text
# kubelet runtime manager 容器创建逻辑
pkg/kubelet/kuberuntime/kuberuntime_container.go
```

创建容器前，kubelet 会先生成 `runtimeapi.ContainerConfig`，再调用 internal lifecycle hook，最后通过 CRI 创建容器。

```go
// 文件：pkg/kubelet/kuberuntime/kuberuntime_container.go
// 作用：生成容器配置、执行 PreCreateContainer hook，并调用 CRI CreateContainer。
containerConfig, cleanupAction, err := m.generateContainerConfig(...)

// CPU Manager 的 cpuset 注入发生在这个 hook 中。
err = m.internalLifecycle.PreCreateContainer(logger, pod, container, containerConfig)

// 带有 CpusetCpus 的 ContainerConfig 会通过 CRI 发送给容器运行时。
containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)
```

Linux 下 CPU Manager 注入 cpuset 的代码在：

```text
# Linux 平台 internal lifecycle hook
pkg/kubelet/cm/internal_container_lifecycle_linux.go
```

核心逻辑如下：

```go
// 文件：pkg/kubelet/cm/internal_container_lifecycle_linux.go
// 作用：容器创建前，从 CPU Manager 获取 cpuset，并写入 CRI ContainerConfig。
func (i *internalContainerLifecycleImpl) PreCreateContainer(logger klog.Logger, pod *v1.Pod, container *v1.Container, containerConfig *runtimeapi.ContainerConfig) error {
    if i.cpuManager != nil {
        // 对独占容器返回 assignment；对普通容器返回 defaultCPUSet。
        allocatedCPUs := i.cpuManager.GetCPUAffinity(string(pod.UID), container.Name)
        if !allocatedCPUs.IsEmpty() {
            // 这里是 kubelet 写入 CRI cpuset 字段的关键点。
            containerConfig.Linux.Resources.CpusetCpus = allocatedCPUs.String()
        }
    }
    return nil
}
```

`GetCPUAffinity()` 位于 `cpu_manager.go`。

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager.go
// 作用：返回容器应使用的 CPU affinity。
func (m *manager) GetCPUAffinity(podUID, containerName string) cpuset.CPUSet {
    return m.state.GetCPUSetOrDefault(podUID, containerName)
}
```

因此，容器创建时的 cpuset 决策路径如下：

```text
# 创建容器时的 cpuset 决策路径。
PreCreateContainer()
  -> cpuManager.GetCPUAffinity(podUID, containerName)
  -> state.GetCPUSetOrDefault(podUID, containerName)
      -> 如果容器有独占 assignment，返回 assignments[podUID][containerName]
      -> 如果容器没有独占 assignment，返回 defaultCPUSet
  -> containerConfig.Linux.Resources.CpusetCpus = cpuset.String()
```

CRI 字段定义在：

```text
# CRI API 生成代码
staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.pb.go
```

对应结构如下：

```go
// 文件：staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.pb.go
// 作用：CRI LinuxContainerResources 中的 cpuset 字段定义。
type LinuxContainerResources struct {
    // 限制容器允许运行的逻辑 CPU 集合；空字符串表示未指定。
    CpusetCpus string `protobuf:"bytes,6,opt,name=cpuset_cpus,json=cpusetCpus,proto3" json:"cpuset_cpus,omitempty"`

    // 限制容器允许使用的 NUMA memory node 集合；空字符串表示未指定。
    CpusetMems string `protobuf:"bytes,7,opt,name=cpuset_mems,json=cpusetMems,proto3" json:"cpuset_mems,omitempty"`
}
```

所以 kubelet 传给 CRI runtime 的结构是：

```text
# CRI 创建容器请求中的 cpuset 字段路径。
runtime.v1.ContainerConfig
  -> Linux
    -> Resources
      -> CpusetCpus
```

runtime 收到 `CreateContainer` 请求后，会把 `CpusetCpus` 转换到 OCI/cgroup 资源配置，最终写入容器 cgroup。Kubernetes 仓库中 vendored 的 OCI cgroups 库能看到最终文件写入语义。

```go
// 文件：vendor/github.com/opencontainers/cgroups/fs2/cpuset.go
// 作用：cgroup v2 下把 OCI resources 中的 CpusetCpus 写入 cpuset.cpus。
if r.CpusetCpus != "" {
    if err := cgroups.WriteFile(dirPath, "cpuset.cpus", r.CpusetCpus); err != nil {
        return err
    }
}
```

对于 cgroup v1，类似逻辑位于 `vendor/github.com/opencontainers/cgroups/fs/cpuset.go`，最终也会写入 cpuset 控制文件。

### cpuset.cpus 文件含义

`cpuset.cpus` 是 cgroup cpuset 子系统的核心控制文件，用来定义该 cgroup 内进程允许运行的 CPU 集合。它的内容格式通常是逗号分隔的 CPU 编号或范围，例如 `0-3`、`2,4,6` 或 `0-1,4-7`。当容器运行时把 kubelet 传来的 `LinuxContainerResources.CpusetCpus` 写入容器 cgroup 后，内核调度器会基于这个 cpuset 约束该 cgroup 内进程可运行的 CPU。

```text
# cgroup v1 示例路径：cpuset 控制器独立挂载。
/sys/fs/cgroup/cpuset/<pod-or-container-cgroup>/cpuset.cpus

# cgroup v2 示例路径：统一层级下的 cpuset 控制文件。
/sys/fs/cgroup/<pod-or-container-cgroup>/cpuset.cpus
```

在 cgroup v2 下，需要区分 `cpuset.cpus` 和 `cpuset.cpus.effective`。`cpuset.cpus` 表示当前 cgroup 配置的期望 CPU 集合；`cpuset.cpus.effective` 表示最终实际生效的 CPU 集合，它会受到父 cgroup cpuset 配置约束。也就是说，即使某个子 cgroup 的 `cpuset.cpus` 写入了 `2-7`，如果父 cgroup 实际只允许 `4-7`，那么子 cgroup 的 `cpuset.cpus.effective` 也只能是 `4-7`。

```bash
# 示例：查看容器 cgroup 配置的期望 CPU 集合。
cat /sys/fs/cgroup/<container-cgroup-path>/cpuset.cpus

# 示例：在 cgroup v2 下查看容器最终实际生效的 CPU 集合。
cat /sys/fs/cgroup/<container-cgroup-path>/cpuset.cpus.effective
```

因此，在验证 `strict-cpu-reservation` 是否真正生效时，不能只看 kubelet checkpoint，也不能只看进程瞬时运行在哪个 CPU 上。更可靠的做法是查看容器 cgroup 的 `cpuset.cpus.effective`，确认其中不包含 `reservedSystemCPUs` 配置的 CPU。

完整链路可以概括为：

```text
# kubelet 到 cgroup 的完整 cpuset 链路。
CPU Manager state
  -> state.defaultCPUSet / assignments
  -> manager.GetCPUAffinity()
  -> internalContainerLifecycleImpl.PreCreateContainer()
  -> containerConfig.Linux.Resources.CpusetCpus
  -> RuntimeService.CreateContainer()
  -> CRI LinuxContainerResources.CpusetCpus
  -> container runtime 转换为 OCI/cgroup 配置
  -> 写入容器 cgroup 的 cpuset.cpus
```

## 6. reconcile 链路

除了创建容器时注入 cpuset，CPU Manager 还有周期性 reconcile 逻辑，用于纠正已经运行容器的 cpuset。`manager.Start()` 中会启动周期任务。

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager.go
// 作用：static policy 启动后，周期性执行 reconcileState()。
go wait.Until(func() { m.reconcileState(ctx) }, m.reconcilePeriod, wait.NeverStop)
```

`reconcileState()` 会遍历 active pods 和容器，重新计算期望 cpuset，并与上次更新状态比较。如果不同，就调用 `updateContainerCPUSet()`。

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager.go
// 作用：为运行中容器重新计算期望 cpuset，并在发生变化时更新 runtime。
cset := m.state.GetCPUSetOrDefault(string(pod.UID), container.Name)
lcset := m.lastUpdateState.GetCPUSetOrDefault(string(pod.UID), container.Name)
if !cset.Equals(lcset) {
    err = m.updateContainerCPUSet(ctx, containerID, cset)
    if err != nil {
        // 更新失败时记录错误，等待下一轮 reconcile。
        continue
    }
    m.lastUpdateState.SetCPUSet(string(pod.UID), container.Name, cset)
}
```

Linux 下 `updateContainerCPUSet()` 位于：

```text
# Linux/非 Windows 平台 CPUSet 更新逻辑
pkg/kubelet/cm/cpumanager/cpu_manager_others.go
```

实现如下：

```go
// 文件：pkg/kubelet/cm/cpumanager/cpu_manager_others.go
// 作用：通过 CRI UpdateContainerResources 更新运行中容器的 CpusetCpus。
func (m *manager) updateContainerCPUSet(ctx context.Context, containerID string, cpus cpuset.CPUSet) error {
    return m.containerRuntime.UpdateContainerResources(
        ctx,
        containerID,
        &runtimeapi.ContainerResources{
            Linux: &runtimeapi.LinuxContainerResources{
                CpusetCpus: cpus.String(),
            },
        })
}
```

CRI client 侧会把它封装成 `UpdateContainerResourcesRequest`。

```go
// 文件：staging/src/k8s.io/cri-client/pkg/remote_runtime.go
// 作用：向 CRI runtime 发送 UpdateContainerResources 请求。
_, err := r.runtimeClient.UpdateContainerResources(ctx, &runtimeapi.UpdateContainerResourcesRequest{
    ContainerId: containerID,
    Linux:       resources.GetLinux(),
    Windows:     resources.GetWindows(),
})
```

reconcile 链路说明 CPU Manager 并不只在容器创建时设置一次 cpuset。对于已经运行的容器，如果 kubelet state 中期望 cpuset 发生变化，kubelet 会通过 CRI 更新容器资源，使 runtime 侧 cgroup 尽量回到期望状态。

## 7. 配置和验证建议

典型配置如下：

```yaml
# kubelet 配置示例：启用 static CPU Manager，并开启严格 CPU 预留。
cpuManagerPolicy: static

# 为系统侧负载预留 CPU 0 和 CPU 1。
reservedSystemCPUs: "0-1"

cpuManagerPolicyOptions:
  # 开启后，普通共享 Pod 继承的 defaultCPUSet 会排除 CPU 0 和 CPU 1。
  strict-cpu-reservation: "true"
```

从旧行为切换到 strict 行为时，需要注意 checkpoint 不变量变化。旧 checkpoint 里可能保存了包含 reserved CPU 的 `defaultCPUSet`；开启 strict 后，`validateState()` 会拒绝这种状态。因此建议在节点维护窗口内 drain 节点，并清理旧的 CPU Manager state 文件。

```bash
# 示例：查看 CPU Manager checkpoint，确认 defaultCPUSet 是否仍包含 reserved CPU。
cat /var/lib/kubelet/cpu_manager_state

# 示例：查看目标容器 cgroup 中实际写入的 cpuset。
# <container-cgroup-path> 需要替换为实际容器 cgroup 路径。
cat /sys/fs/cgroup/<container-cgroup-path>/cpuset.cpus

# 示例：查看目标进程的 CPU affinity。
# <pid> 需要替换为容器内或宿主机上目标进程的 PID。
taskset -cp <pid>
```

如果 `reservedSystemCPUs: "0-1"` 且 strict 配置生效，普通共享 Pod 的 `cpuset.cpus` 不应包含 `0-1`，而应类似 `2-7`。

## 8. 边界和注意事项

`strict-cpu-reservation` 解决的是 Pod 容器侧不再使用 reserved CPU 的问题。它不会自动把系统进程、中断、kubelet 或容器运行时迁移到 reserved CPU 上。如果目标是完整隔离系统侧和业务侧 CPU，还需要配合系统层配置，例如 systemd `CPUAffinity`、`tuned`、`irqbalance`、`isolcpus` 或宿主机 cpuset/cgroup 配置。

另外，`strict-cpu-reservation` 的核心实现点不在调度器，也不在 CRI runtime 对 reserved CPU 的特殊识别。它只是在 kubelet CPU Manager static policy 初始化 `defaultCPUSet` 时改变共享池边界。这个边界随后通过 `GetCPUSetOrDefault()`、`PreCreateContainer()` 和 CRI `LinuxContainerResources.CpusetCpus` 自然传递到容器 cgroup。

## 9. 总结

旧版本中，`reservedSystemCPUs` 已经能影响独占 CPU 分配，但普通 Pod 仍可能使用 reserved CPU，因为共享池 `defaultCPUSet` 仍包含 reserved CPU。`strict-cpu-reservation` 的关键修复是：在 `staticPolicy.validateState()` 中初始化 `defaultCPUSet` 时执行 `allCPUs.Difference(p.reservedCPUs)`，并把 checkpoint 校验不变量从“reserved CPU 必须在 defaultCPUSet 中”改为“reserved CPU 不能在 defaultCPUSet 中”。

最终效果是，独占 Pod 和普通共享 Pod 都会从不含 reserved CPU 的 CPU 集合中获得 cpuset。该 cpuset 会在容器创建前写入 `containerConfig.Linux.Resources.CpusetCpus`，通过 CRI 传给容器运行时，并落到容器 cgroup 的 `cpuset.cpus` 文件中。
