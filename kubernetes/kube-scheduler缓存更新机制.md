# kube-scheduler 如何准确更新自身缓存

> 本文基于 Kubernetes 源码 tag `v1.36.2` 分析，对应提交为 `24e2b02af5`。关键源码位置包括 `pkg/scheduler/schedule_one.go` 和 `pkg/scheduler/eventhandlers.go`。

## 核心结论

`kube-scheduler` 的缓存不是简单地以 API Server 里的 Pod 状态为准做全量覆盖，而是把调度路径拆成两条信息流：调度器内部的假定写入和 informer 监听到的真实集群事件。前者用于让调度器在绑定真正落盘前就能避免资源超卖，后者用于把缓存修正到最终事实。

理解缓存更新的关键，是区分三个对象：

1. `SchedulingQueue` 保存待调度 Pod。
2. `Cache` 保存已假定或已确认调度到节点上的 Pod，并维护节点资源视图。
3. informer 事件来自 API Server，是调度器校准缓存的最终事实来源。

以 `v1.36.2` 源码为准，`AssumePod` 不在真正执行 bind 的函数里调用，而是在调度周期结束、绑定周期启动之前调用。准确调用链是：`scheduleOnePod` -> `schedulingCycle` -> `prepareForBindingCycle` -> `assumeAndReserve` -> `assume` -> `Cache.AssumePod`，然后才异步进入 `runBindingCycle` -> `bindingCycle` -> `bind`。

## 背景：调度器缓存为什么需要“先假定、后确认”

调度器的调度周期和绑定周期是分离的。一个 Pod 完成调度决策后，绑定操作可能还没有被 API Server 确认。如果调度器在绑定完成前不更新本地缓存，那么下一个 Pod 仍然会看到旧的节点可用资源，可能被错误地继续调度到同一个节点，造成资源超卖。

因此调度器会在选择节点之后、启动异步绑定之前，立即执行一次本地假定：先把 Pod 的 `spec.nodeName` 设置为目标节点，再把 Pod 写入 `Cache`。这个阶段的缓存代表“调度器认为它即将成功绑定”的状态，而不是 API Server 已经确认的状态。

源码对应关系如下：

```go
// 伪代码：以 pkg/scheduler/schedule_one.go 的调用链为准
func scheduleOnePod(podInfo *QueuedPodInfo) {
    // 1. 调度周期：更新 snapshot，执行过滤、打分，选出目标节点
    scheduleResult := schedulingAlgorithm(podInfo)

    // 2. 调度周期末尾：在启动绑定周期前先写本地缓存
    assumedPodInfo := prepareForBindingCycle(podInfo, scheduleResult)

    // 3. 绑定周期：异步执行 Permit / PreBind / Bind / PostBind
    go runBindingCycle(scheduleResult, assumedPodInfo)
}

func prepareForBindingCycle(podInfo *QueuedPodInfo, result ScheduleResult) *QueuedPodInfo {
    return assumeAndReserve(podInfo, result)
}

func assumeAndReserve(podInfo *QueuedPodInfo, result ScheduleResult) *QueuedPodInfo {
    assumedPodInfo := podInfo.DeepCopy()

    // assume 会先设置 assumedPodInfo.Pod.Spec.NodeName = result.SuggestedHost
    // 然后调用 Cache.AssumePod，把 Pod 作为已分配到目标节点的对象写入调度器缓存
    assume(assumedPodInfo, result.SuggestedHost)

    // Reserve 插件在 Assume 之后执行；Reserve 失败时通过 unreserveAndForget 回滚
    runReservePlugins(assumedPodInfo)
    return assumedPodInfo
}
```

`AssumePod` 的价值在于提升并发调度的正确性；informer 的价值在于把调度器的假定状态修正为集群真实状态。两者配合起来，形成“先乐观写缓存，再用真实事件收敛”的机制。

## 场景 1：未调度 Pod

未调度 Pod 指的是还没有 `.spec.nodeName` 的 Pod。调度器通过 informer 监听到这类 Pod 后，会把它放入 `SchedulingQueue`，而不是直接加入 `Cache` 的节点资源统计。

源码入口在 `pkg/scheduler/eventhandlers.go`。`addPod` 会先判断 `assignedPod(pod)`。`assignedPod` 的判断条件很简单：`len(pod.Spec.NodeName) != 0`。如果没有 `nodeName`，并且该 Pod 由当前 scheduler profile 负责，才会调用 `addPodToSchedulingQueue`。

处理逻辑可以概括为：

```go
// 伪代码：对应 eventhandlers.go 的 addPod 分流
func addPod(pod *Pod) {
    if assignedPod(pod) {
        addAssignedPodToCache(pod)
        return
    }

    if responsibleForPod(pod) {
        addPodToSchedulingQueue(pod)
    }
}

func assignedPod(pod *Pod) bool {
    return pod.Spec.NodeName != ""
}
```

这个场景下，缓存准确性的边界是：`SchedulingQueue` 记录“等待被调度的事实”，`Cache` 记录“已经归属到节点的事实”。未调度 Pod 只影响队列，不影响节点资源缓存。

## 场景 2：完成调度 Pod

完成调度 Pod 指的是调度器已经为 Pod 选出节点，并且绑定最终被 API Server 接受。这个过程不是一步完成的，而是经历“本地假定”和“真实确认”两个阶段。

第一阶段发生在绑定之前。`assumeAndReserve` 会复制一份 Pod，调用 `assume` 设置 `spec.nodeName`，再调用 `Cache.AssumePod`。从这一刻开始，调度器本地缓存已经认为该 Pod 占用了目标节点资源，所以后续调度周期不会继续把这部分资源当成可用资源。

第二阶段发生在绑定周期。`bindingCycle` 会等待 Permit，执行 PreBind，然后调用 `bind`。`bind` 内部执行 extender bind 或 `RunBindPlugins`。默认 binder 最终会向 API Server 写入 Binding。绑定成功后，API Server 中 Pod 会变成带 `spec.nodeName` 的 assigned pod，informer 随后触发 `addAssignedPodToCache` 或 `updateAssignedPodInCache`，缓存由 assumed 对象收敛到 API Server 里的真实对象版本。

简化流程如下：

```go
// 伪代码：完成调度 Pod 的缓存状态流转，以源码调用顺序为准
func schedulingCycle(podInfo *QueuedPodInfo) (*QueuedPodInfo, Status) {
    result := schedulingAlgorithm(podInfo)

    // 1. 绑定前先假定：设置 nodeName，并写入 Cache.AssumePod
    assumedPodInfo := assumeAndReserve(podInfo, result)

    // 2. Permit 插件也在 prepareForBindingCycle 中执行；失败会 ForgetPod 回滚
    runPermitPlugins(assumedPodInfo)
    return assumedPodInfo, nil
}

func bindingCycle(assumedPodInfo *QueuedPodInfo, result ScheduleResult) Status {
    assumedPod := assumedPodInfo.Pod

    waitOnPermit(assumedPod)
    schedulingQueue.Done(assumedPod.UID)
    runPreBindPlugins(assumedPod)

    // 3. bind 不再做 AssumePod，只负责把绑定结果写向 API Server
    status := bind(assumedPod, result.SuggestedHost)
    if !status.IsSuccess() {
        // 4. 绑定周期失败时，handleBindingCycleError 会 unreserveAndForget，最终 ForgetPod
        return status
    }

    runPostBindPlugins(assumedPod)
    return nil
}

func onPodUpdate(oldPod, newPod *Pod) {
    if assignedPod(oldPod) {
        updateAssignedPodInCache(oldPod, newPod)
    } else if assignedPod(newPod) {
        // 这个 update 代表绑定发生；AddPod 会处理 assumed pod 与真实 pod 的衔接
        addAssignedPodToCache(newPod)
        deletePodFromSchedulingQueue(oldPod, true)
    }
}
```

这个场景下，缓存的准确性来自两层保障：

1. 调度器在绑定前先 `AssumePod`，保证并发调度时资源视图不滞后。
2. informer 在绑定落盘后触发 assigned pod 的 add/update 事件，保证本地对象版本最终和 API Server 对齐。

因此，完成调度的 Pod 在缓存中会先以 assumed 形态存在，随后变成真实已确认的 assigned pod。这个设计本质上是最终一致，而不是强一致。

## 场景 3：调度完成但被 kubelet 二次否决，随后进入删除清理路径

这里讨论的“二次否决”不是调度器的 `Bind` 失败，而是 Pod 已经被绑定到某个 Node 后，目标节点上的 kubelet 在本地准入阶段发现它不能运行，于是拒绝该 Pod，并把 Pod 置为失败状态。需要注意：`rejectPod` 本身不会直接调用 API Server 删除 Pod，它只是写失败状态；真正的删除发生在 Pod 已经带有 `deletionTimestamp`、并且 kubelet 终止流程确认完成之后。

从 `kube-scheduler` 视角看，绑定一旦成功写入 API Server，Pod 就已经有 `.spec.nodeName`。调度器的 `eventhandlers.go` 里 `assignedPod(pod)` 只通过 `len(pod.Spec.NodeName) != 0` 判断 Pod 是否已分配到节点，不会因为 kubelet 本地准入失败就把它重新当成未调度 Pod。因此，kubelet 的二次否决不会让原 Pod 回到 `SchedulingQueue`，而是先表现为已绑定 Pod 的状态更新；只有当该 Pod 后续进入删除流程时，调度器才会再看到 delete 事件。

kubelet 侧的关键路径在 `pkg/kubelet/kubelet.go`。`HandlePodAdditions` 收到被分配到本节点的 Pod 后，会先把 Pod 加入本地 `podManager`，然后调用 `allocationManager.AddPod` 做本地准入。如果本地准入失败，kubelet 会调用 `rejectPod`。`rejectPod` 会记录事件，并通过 `statusManager.SetPodStatus` 把 Pod 状态设置为 `Failed`，Reason 和 Message 说明拒绝原因。

简化流程如下：

```go
// 伪代码：kubelet 收到已绑定 Pod 后的本地准入路径
func HandlePodAdditions(pods []*Pod) {
    for _, pod := range pods {
        // kubelet 先把 API Server 下发的 Pod 加入本地期望状态
        podManager.AddPod(pod)

        // 本地准入失败时，kubelet 不会让该 Pod 继续创建容器
        ok, reason, message := allocationManager.AddPod(GetActivePods(), pod)
        if !ok {
            rejectPod(pod, reason, message)
            continue
        }

        // 只有通过本地准入后，才会进入 pod worker 创建流程
        podWorkers.UpdatePod(SyncPodCreate, pod)
    }
}

func rejectPod(pod *Pod, reason, message string) {
    recordEvent(pod, reason, message)
    statusManager.SetPodStatus(pod, PodStatus{
        Phase:   Failed,
        Reason:  reason,
        Message: "Pod was rejected: " + message,
    })
}
```

Pod 被 kubelet 标记为 `Failed` 后，源码里还有两段删除相关逻辑，但都不是 `rejectPod` 当场直接删除。第一段在 `status_manager.go` 的 `syncPod`：它会先 patch Pod status，然后只有当 `canBeDeleted` 返回 true 时才调用 `CoreV1().Pods(...).Delete(...)`；而 `canBeDeleted` 明确要求 Pod 已经有 `deletionTimestamp`，并且本地终止流程已经完成。第二段在 `kubelet_pods.go` 的 `HandlePodCleanups`：它会把已经是终态、已经有 `deletionTimestamp`、且不在 pod worker 中的 Pod 通过 `SyncPodKill` 交给 pod worker 走强制删除清理路径。源码注释里还特别举例：这类 Pod 包括 kubelet admission 阶段被拒绝、从未启动过的 Pod。

所以更准确的说法是：kubelet 本地准入失败时会先 `rejectPod` 写 `Failed` 状态，不会立刻删除 API Server 里的 Pod 对象；如果该 Pod 随后被标记删除，kubelet 的 cleanup/status manager 才会推进最终删除。以 Job、ReplicaSet、Deployment 等控制器管理的 Pod 为例，控制器会观察到旧 Pod 失败或删除，并按期望副本数创建新的替代 Pod；新的 Pod 没有 `.spec.nodeName`，所以会重新进入调度器的 `SchedulingQueue`。同时，调度器会通过 informer 观察到旧 Pod 的 update/delete 事件：update 阶段调用 `Cache.UpdatePod` 同步状态，delete 阶段调用 `Cache.RemovePod` 从节点缓存中释放资源。

对应调度器侧可以概括为：

```go
// 伪代码：调度器只按 API Server 事件修正缓存
func updatePod(oldPod, newPod *Pod) {
    if assignedPod(oldPod) {
        // kubelet 写回 Failed 状态时，调度器更新已分配 Pod 的缓存对象
        updateAssignedPodInCache(oldPod, newPod)
        return
    }

    if assignedPod(newPod) {
        // 从未分配变成已分配，说明绑定结果被观察到
        addAssignedPodToCache(newPod)
        deletePodFromSchedulingQueue(oldPod, true)
        return
    }

    updatePodInSchedulingQueue(oldPod, newPod)
}

func deletePod(pod *Pod) {
    if assignedPod(pod) {
        // kubelet/控制器删除旧 Pod 后，调度器从节点缓存中移除该 Pod
        deleteAssignedPodFromCache(pod)
        return
    }

    deletePodFromSchedulingQueue(pod, false)
}
```

这个场景的重点是：调度器不会主动“撤销绑定并重调度原 Pod”。原 Pod 已经绑定到节点，kubelet 本地否决后先把它置为失败；后续如果 Pod 被删除，调度器只负责根据 informer 的 delete 事件修正缓存。真正重新调度的是控制器创建出来的新 Pod，而不是被 kubelet 否决的旧 Pod。

## kube-scheduler 在哪些情况下调度可能不准确

`kube-scheduler` 的决策基于本地缓存和调度周期内的 snapshot，它追求的是高吞吐下的最终一致，而不是对集群瞬时状态的强一致。因此，以下情况下调度结果可能和节点真实可运行状态存在偏差：

1. **Assume 到绑定确认之间的时间窗口。** 调度器在绑定前先 `Cache.AssumePod`，这会让本地缓存提前认为 Pod 已经占用目标节点资源。如果随后 Reserve、Permit、PreBind 或 Bind 失败，缓存需要通过 `ForgetPod` 回滚。在回滚前，缓存会短暂比真实集群“多算”一个 Pod。
2. **绑定成功但 kubelet 本地准入失败。** 调度器只确认 Pod 是否绑定到 Node，不负责最终容器能否在该节点启动。kubelet 可能因为本地资源、特性门控、OS/安全策略、设备或运行时约束拒绝 Pod。此时调度器曾经做出的绑定从 API Server 角度是成功的，但从“能否真正运行”角度看是不准确的。
3. **节点状态在调度周期内快速变化。** 调度器使用的是 informer 缓存和调度周期开始时的 snapshot。如果节点资源、Pod 数量、污点、标签、设备状态在调度过程中发生变化，调度器可能基于稍旧的视图选出节点。
4. **kubelet 本地状态没有完全上报给 API Server。** 有些约束只存在于节点本地，或者上报存在延迟，例如临时资源压力、运行时异常、设备插件状态变化、镜像/卷相关失败。调度器如果无法及时从 Node/Pod 状态中看到这些信息，就可能做出对 kubelet 来说不可接受的选择。
5. **扩展插件与 kubelet 准入逻辑不完全等价。** 调度器插件负责在调度阶段近似判断 Pod 是否适合节点，但 kubelet 还有自己的 admission handlers。两边检查的时机、数据来源和覆盖范围不同，只要插件没有完整模拟 kubelet 本地准入，就可能出现“调度器认为可以，kubelet 拒绝”的情况。
6. **外部组件改动了 Pod 或 Node。** 例如其他组件绑定 Pod、修改 Node 状态、删除 assumed pod、或控制器快速删除重建 Pod。调度器最终会通过 informer 事件收敛，但事件到达前本地缓存可能短暂不准确。

这些不准确通常不是缓存 bug，而是 Kubernetes 调度模型的边界：调度器负责基于可见集群状态做最佳努力的节点选择，kubelet 负责在节点本地做最终准入和运行。两者之间通过 API Server 状态和 informer 事件做最终一致的校准。

## 为什么不会长期不一致

调度器缓存允许短时间不一致，但通过以下机制避免长期错误：

1. 绑定前先假定。`assume` 设置 `spec.nodeName` 并调用 `Cache.AssumePod`，让后续调度周期立即看到资源占用。
2. 调度周期失败会回滚。Reserve 或 Permit 失败时，会通过 `unreserveAndForget` 调用 `Cache.ForgetPod`。
3. 绑定周期失败会回滚。PreBind、Bind 等阶段失败时，`handleBindingCycleError` 会调用 `unreserveAndForget`，最终 `ForgetPod`。
4. informer 事件持续校准。API Server 中 Pod 的新增、更新、删除都会反馈到调度器，assigned pod 的 add/update/delete 分别对应 `Cache.AddPod`、`Cache.UpdatePod`、`Cache.RemovePod`。
5. assumed pod 被删除会特殊处理。如果 assumed pod 在绑定完成前被删除，`handleAssumedPodDeletion` 会通过 `Cache.RemoveAssumedPod` 立即释放它占用的缓存资源。

可以把这个机制理解成一条状态链：

```text
# 状态链：调度器缓存从待调度事实走向集群事实
未调度 Pod
  -> addPodToSchedulingQueue
  -> schedulingAlgorithm 选出目标节点
  -> assumeAndReserve
  -> assume 设置 Pod.Spec.NodeName
  -> Cache.AssumePod 写入调度器本地缓存
  -> go runBindingCycle 异步绑定
  -> bind 写入 API Server
  -> informer 观察到 assigned pod
  -> Cache.AddPod 或 Cache.UpdatePod 将 assumed 状态收敛到真实对象
  -> Pod 删除时 Cache.RemovePod 释放资源
```
