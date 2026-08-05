# kube-apiserver高并发Create请求分析

> 本文基于 Kubernetes `v1.34.0` 和本地 kind 单控制面集群验证，分析高并发 Namespace CREATE 请求的延迟变化，并通过 CPU、goroutine、Block Profile 和阶段级 Trace 定位请求耗时。测试请求为 `POST /api/v1/namespaces`，压测客户端通过 TLS/HTTP/2 直连 kube-apiserver。文中的压测数据和结论仅适用于该 CREATE 场景，不能直接外推到 LIST、单对象 GET、UPDATE、DELETE 或 WATCH。

## 结论

本次在本机 kind 集群中模拟高并发创建 Namespace。慢请求的主要耗时位于 **APF 排队**和 **etcd Txn 写事务**，而不是认证、鉴权、Admission、Managed Fields、响应序列化或响应写回。

并发量表示同一时刻最多维持多少个正在执行或等待返回的请求。例如 `c=200、n=2000` 表示最多同时存在 200 个请求，总共完成 2000 个请求；它不是将 2000 个请求一次性全部发出。第一批约 200 个请求会快速发出，某个请求返回后，对应执行单元继续发送下一个请求。

最慢 20 条请求的服务端墙钟耗时 P50 为 76.322 ms、P95 为 86.940 ms。其中 APF queue 的 P50 为 26.961 ms、P95 为 68.096 ms；storage outer 的 P50 为 42.741 ms、P95 为 55.611 ms；嵌套在 storage outer 内的 etcd Txn P50 为 38.578 ms、P95 为 55.180 ms。storage outer 扣除 etcd Txn 后的独占耗时很小，说明写入阶段的主要时间确实消耗在 etcd Txn 调用内部。

etcd Txn 的墙钟耗时并不等于一次磁盘写入时间。它覆盖 kube-apiserver 发起事务后，在 etcd 内经历请求等待、Raft proposal、WAL 持久化、提交与应用，直到结果返回的完整过程。高并发写入会争用同一条 Raft 写入流水线，因此即使不同 Namespace 之间没有对象冲突，也可能因排队、批量提交、磁盘同步和 CPU 调度而增加单次请求延迟。

## 测试环境

| 项目 | 配置 |
|---|---|
| 主机 | Apple M3 Pro，12 核（6 个性能核、6 个能效核），36 GB 内存 |
| Docker Desktop Linux VM | 12 个逻辑 CPU |
| kind | 单 control-plane 节点，control-plane 容器内运行 kube-apiserver、etcd、kube-controller-manager 和 kube-scheduler |
| Kubernetes | v1.34.0，linux/arm64 |
| kubectl | v1.34.1，darwin/arm64 |
| etcd | kind 控制面内的单节点 etcd |
| Admission Webhook | 未配置 |
| APF | 常规压测使用系统默认配置；阶段级 Trace 临时使用专用 FlowSchema 和 PriorityLevel |
| Audit | 未配置审计策略 |
| OpenTelemetry Tracing | 未配置；阶段数据来自临时最小源码插桩 |
| kube-apiserver CPU quota | 未设置 |
| 请求方式 | TLS/HTTP/2 直连 kube-apiserver，不经过 `kubectl proxy` |

kind 的 control-plane 是一个 Docker 容器，kube-apiserver 和 etcd 共享 Docker Desktop Linux VM 的 CPU 与存储资源。因此，本次数据反映的是该本地单控制面环境中的延迟表现，不等同于生产多节点 etcd 集群。`docker stats` 展示的是整个 control-plane 容器的资源使用量；kube-apiserver 自身 CPU 需要结合 `process_cpu_seconds_total` 和 CPU Profile 计算。

阶段级 Trace 为覆盖真实认证、鉴权和 APF handler chain，使用专用 ServiceAccount 发起请求，并临时创建专用 FlowSchema 和 PriorityLevel。测试完成后，相关 Namespace、FlowSchema、PriorityLevel、ClusterRoleBinding 和 ServiceAccount 均已删除，kube-apiserver 已恢复为 Kubernetes v1.34.0 官方镜像。

## 测试范围

测试对象为 Kubernetes v1.34.0 的 Namespace CREATE 请求，即 `POST /api/v1/namespaces`。集群运行在本机 kind 环境中，kube-apiserver 和单节点 etcd 共享 Docker Desktop Linux 虚拟机中的 CPU 与存储资源。测试环境没有额外配置 Admission Webhook。

本次分析关注单次请求为什么变慢，不使用吞吐量替代延迟结论。RPS 表示每秒完成的请求数，并发量表示同一时刻最多存在的请求数，两者不是同一个概念。高并发下可以同时出现吞吐量上升和单请求耗时上升。

## 并发量的含义

本次压测不是只使用一个并发值，而是先执行并发阶梯，再在 `c=200` 时采集 Profile，最后单独执行 `c=160` 的阶段级 Trace。各轮请求均创建名称唯一的 Namespace。

| 测试阶段 | 并发量 | 请求总数 | 用途 |
|---|---:|---:|---|
| 低并发基线 | 1 | 20 | 获取单请求延迟基线 |
| 并发阶梯 | 20 | 200 | 观察并发增加后的初始变化 |
| 并发阶梯 | 50 | 500 | 观察中等并发下的延迟 |
| 并发阶梯 | 100 | 1000 | 观察 APF 排队前后的变化 |
| Profile 压测 | 200 | 2000 | 采集 CPU、goroutine 和 Block Profile |
| 阶段级 Trace | 160 | 160 | 采集同一请求内各处理阶段的墙钟时间 |

以 `c=200、n=2000` 为例，压测程序会启动 200 个执行单元。开始时第一批请求会在很短时间内发出，但不能保证它们在完全相同的微秒到达。每个执行单元等待当前请求返回后再发送下一个请求，持续滚动，直到累计完成 2000 次 CREATE。阶段级 Trace 的最终统计则来自修正 etcd Txn 插桩后的 160 个并发请求；测试过程中的两轮 Trace 和控制请求共创建 321 个 Namespace，最终分析只使用后一轮完整的 160 条记录。

| 参数 | 含义 |
|---|---|
| `c=200` | 同一时刻最多维持约 200 个正在执行或等待返回的请求 |
| `n=2000` | 整轮测试累计完成 2000 个请求 |
| RPS | 每秒实际完成的请求数，不等于并发量 |
| 请求延迟 | 单个请求从发出到返回，或服务端从接收到处理完成的墙钟时间 |

当并发量超过 kube-apiserver 当前可立即执行的能力时，部分请求先进入 APF 队列。请求被 APF 放行后，还要执行对象处理并等待 etcd 完成真实写入。因此并发提升后，请求延迟会同时受到 APF 排队和存储写入排队影响。

## 并发阶梯压测结果

各轮请求均创建名称唯一的 Namespace，成功请求返回 HTTP 201。P50、P95 和 P99 是客户端复用 TLS/HTTP/2 连接后观测到的端到端墙钟时间。

| 并发 | 请求数 | 成功 RPS | P50 | P95 | P99 |
|---:|---:|---:|---:|---:|---:|
| 1 | 20 | 391.5 | 2.046 ms | 3.226 ms | 10.739 ms |
| 20 | 200 | 1109.1 | 14.917 ms | 40.567 ms | 42.355 ms |
| 50 | 500 | 2552.2 | 14.615 ms | 53.461 ms | 54.191 ms |
| 100 | 1000 | 3807.3 | 18.059 ms | 79.054 ms | 79.742 ms |
| 200 | 2000 | 3893.0 | 35.293 ms | 163.329 ms | 187.168 ms |

并发 1 的样本只有 20 条，按本次分位数取值方法，P99 实际落在最大值 10.739 ms 上，统计稳定性有限；并发 20 以上的样本更适合观察 P99 变化。

从并发 1 提升到 100 时，成功 RPS 从 391.5 增长到 3807.3，但 P95 也从 3.226 ms 增长到 79.054 ms。并发从 100 继续提升到 200 后，RPS 仅从 3807.3 增长到 3893.0，增幅约 2.3%；与此同时，P50 从 18.059 ms 增长到 35.293 ms，P95 从 79.054 ms 增长到 163.329 ms。这说明系统已经接近该环境下的写入吞吐上限，新增并发主要转化为排队和单请求等待，而没有带来相同比例的吞吐提升。

并发 20 与并发 50 的 P50 分别为 14.917 ms 和 14.615 ms，少量倒挂属于样本和运行抖动，不能据此判断并发 50 更快。P95 从 40.567 ms 增长到 53.461 ms，更能反映尾部等待正在增加。

`c=200、n=2000` 时，`apiserver_request_duration_seconds` 中 Namespace POST 的增量总耗时为 68.0415 s，对应服务端平均耗时 34.021 ms；APF queue wait 的增量总耗时为 26.040195 s，对应平均排队约 13.013 ms。按平均值估算，请求被 APF 放行后的处理时间约为 21.008 ms。这个平均拆分用于观察整批请求，后续阶段级 Trace 则用于定位单条慢请求内部的具体耗时。

## 阶段级 Trace 方法

kube-apiserver 自带的指标和 Profile 可以判断系统存在排队或 CPU、锁、I/O 等压力，但不能直接把同一条 CREATE 请求完整拆分为认证、鉴权、APF、Admission、Managed Fields、storage、etcd Txn、序列化和响应写回。为获得各阶段墙钟时间，本次基于 Kubernetes v1.34.0 做了临时的最小源码插桩。

每个请求携带唯一的 `X-Stage-Trace-ID`。服务端将该 ID 放入请求 Context，各阶段从同一个 Context 写入开始时间和持续时间。请求完成后，kube-apiserver 输出一条结构化日志。客户端同时记录响应中的 `Audit-ID`，用于把客户端请求、服务端阶段记录和审计标识关联起来。

阶段持续时间使用 `time.Since(startedAt)` 计算。Go 的 `time.Time` 在同一进程内携带 monotonic clock 信息，因此适合记录墙钟区间，避免系统时间校准影响持续时间。阶段之间可能嵌套，统计时需要对时间区间取并集，不能直接把 storage outer 和 etcd Txn 相加。

### 统计数据如何获得

本次阶段数据不是通过 Jaeger 采集。测试环境没有配置 OpenTelemetry Tracing，也没有部署 Jaeger。这里使用的是临时源码插桩和 kube-apiserver 结构化日志，借用了 Span 的阶段划分思路，但没有把数据导出到分布式追踪后端。

服务端在每个目标请求结束时输出一条以 `STAGE_TRACE` 标识的 JSON 日志，内容包括 Trace ID、请求开始时间、服务端总耗时，以及 authentication、authorization、APF queue、Admission、Managed Fields、storage outer、etcd Txn、serialization 和 response write 等阶段区间。客户端同时记录 Trace ID、响应中的 Audit ID、HTTP 状态和客户端墙钟时间。

采集完成后，从 kube-apiserver 日志提取 `STAGE_TRACE` JSON，与客户端记录按 Trace ID 关联，形成结构化 JSONL 数据。最终一轮 160 个并发 CREATE 全部返回 HTTP 201，并获得 160 条可关联的阶段记录。统计程序按服务端 `request_duration_ns` 从高到低排序，选择最慢 20 条请求，再分别计算各阶段的 P50 和 P95。对于 storage outer 与 etcd Txn 等嵌套区间，先合并时间区间再计算未解释时间，避免重复统计。

数据链路如下：

```text
压测客户端发送 X-Stage-Trace-ID
→ kube-apiserver 各阶段记录墙钟区间
→ 请求结束时输出 STAGE_TRACE JSON 日志
→ 客户端记录 Trace ID、Audit ID、状态码和端到端耗时
→ 提取并关联服务端与客户端记录
→ 选择最慢 20 条请求并计算各阶段 P50/P95
```

如果使用 Jaeger，实现方式会改为在相同函数边界创建 OpenTelemetry child span，并配置 kube-apiserver 将 Trace 导出到 OpenTelemetry Collector 或 Jaeger；本次验证没有走这条链路。

### 在请求 Context 中保存阶段记录

下面代码展示了插桩的核心结构。正式实现还需要通过互斥锁保护同一请求的阶段数组，并在请求结束时复制快照，避免并发读写产生数据竞争。

```go
// StageTiming 表示同一请求中的一个墙钟时间区间。
type StageTiming struct {
    Stage      string `json:"stage"`
    Start      string `json:"start"`
    DurationNS int64  `json:"duration_ns"`
}

// RecordStage 从阶段开始时间计算持续时间，并写入请求级记录器。
func RecordStage(ctx context.Context, stage string, startedAt time.Time) {
    RecordStageDuration(ctx, stage, startedAt, time.Since(startedAt))
}

func RecordStageDuration(
    ctx context.Context,
    stage string,
    startedAt time.Time,
    duration time.Duration,
) {
    tracker, ok := LatencyTrackersFrom(ctx)
    if !ok {
        return
    }

    tracker.stageMu.Lock()
    defer tracker.stageMu.Unlock()

    tracker.StageTimings = append(tracker.StageTimings, StageTiming{
        Stage:      stage,
        Start:      startedAt.UTC().Format(time.RFC3339Nano),
        DurationNS: duration.Nanoseconds(),
    })
}
```

### 复用已有阶段计时点

认证、鉴权、APF、序列化和响应写回原本已经有延迟跟踪入口。最小改动是在这些入口中额外记录请求级阶段，不需要重写原有处理流程。

```go
// APF 已经计算出排队耗时，直接将该区间加入请求级 Trace。
func TrackAPFQueueWaitLatency(ctx context.Context, d time.Duration) {
    RecordStageDuration(ctx, "apf-queue", time.Now().Add(-d), d)

    if tracker, ok := LatencyTrackersFrom(ctx); ok {
        tracker.APFQueueWaitTracker.TrackDuration(d)
    }
}

// 认证和鉴权使用相同方式记录 authentication、authorization。
// 序列化和响应写回分别记录 serialization、response-write。
```

### 在 CREATE 路径增加缺失阶段

Admission 和 Managed Fields 没有满足本次关联要求的完整请求级区间，因此在 CREATE handler 的调用边界增加计时。计时只包围目标函数，避免把相邻逻辑错误归入该阶段。

```go
// 记录 Managed Fields 更新耗时。
managedFieldsStartedAt := time.Now()
obj = scope.FieldManager.UpdateNoErrors(
    liveObj,
    obj,
    managerOrUserAgent(options.FieldManager, req.UserAgent()),
)
request.RecordStage(ctx, "managed-fields", managedFieldsStartedAt)

// 记录 Mutating Admission 耗时。
admissionStartedAt := time.Now()
err := mutatingAdmission.Admit(ctx, admissionAttributes, scope)
request.RecordStage(ctx, "admission-mutating", admissionStartedAt)
```

Validating Admission 采用同样方式，在 validation 回调外包一层计时函数，记录为 `admission-validating`。分析时，将同一请求内互不重叠的 mutating 和 validating 区间合并为 Admission 总耗时。

### 分开记录 storage outer 和 etcd Txn

`storage-outer` 包围 `e.Storage.Create`，用于观察 kube-apiserver 整个存储调用。`etcd-txn` 只包围实际事务调用。两者是包含关系，不应相加。

```go
// kube-apiserver 的外层 Storage.Create。
storageStartedAt := time.Now()
storageErr := e.Storage.Create(ctx, key, obj, out, ttl, dryRun)
request.RecordStage(ctx, "storage-outer", storageStartedAt)
```

Kubernetes v1.34.0 的 CREATE 路径通过 `OptimisticPut` 发起事务，没有经过旧的 `clientv3.KV.Txn().Commit()` 包装。因此仅在旧 Txn wrapper 增加 Span 会漏掉本次 CREATE，必须在实际调用的 `OptimisticPut` 边界记录。

```go
// Kubernetes v1.34.0 CREATE 实际经过 OptimisticPut。
startTime := time.Now()
txnResp, err := s.client.Kubernetes.OptimisticPut(
    ctx,
    preparedKey,
    newData,
    0,
    kubernetes.PutOptions{LeaseID: lease},
)
request.RecordStage(ctx, "etcd-txn", startTime)
```

这里的“增加 Span”实际是增加一个可关联的请求级墙钟区间。如果集群已经接入 OpenTelemetry，也可以在相同函数边界创建真正的 tracing child span；关键点仍然是 Span 必须从当前请求 Context 派生，并覆盖实际执行的 `OptimisticPut` 调用，而不是只修改未经过的旧事务封装。

### 输出请求级结构化记录

请求结束后，仅对带有测试 Trace ID 的 Namespace CREATE 输出结构化日志，避免把临时诊断扩散到所有请求。

```go
// 仅记录本次诊断请求，降低日志量和插桩影响。
if requestInfo.Verb == "create" &&
    requestInfo.Resource == "namespaces" &&
    strings.HasPrefix(traceID, "nst-") {
    id, startedAt, stages, ok := request.StageTimingSnapshot(req.Context())
    if ok {
        record := stageTraceRecord{
            TraceID:         id,
            RequestStart:    startedAt.UTC().Format(time.RFC3339Nano),
            RequestDuration: time.Since(startedAt).Nanoseconds(),
            Verb:            requestInfo.Verb,
            Resource:        requestInfo.Resource,
            Name:            requestInfo.Name,
            Stages:          stages,
        }
        data, _ := json.Marshal(record)
        klog.Infof("STAGE_TRACE %s", data)
    }
}
```

## 阶段耗时结果

最终捕获 160 条可通过 Trace ID 与 Audit ID 关联的 CREATE 阶段记录，并按服务端整请求墙钟时间降序选择最慢 20 条。单位为毫秒。

| 阶段 | P50 | P95 | 判断 |
|---|---:|---:|---|
| 服务端整请求 | 76.322 | 86.940 | 慢请求总体耗时 |
| authentication | 0.011 | 0.070 | 不是主要瓶颈 |
| authorization | 0.012 | 0.053 | 不是主要瓶颈 |
| APF queue | 26.961 | 68.096 | 主要耗时点之一 |
| Admission 总计 | 0.350 | 3.449 | 不是主要瓶颈 |
| managed-fields | 0.069 | 3.411 | 不是主要瓶颈 |
| storage outer | 42.741 | 55.611 | 主要耗时位于存储路径 |
| etcd Txn | 38.578 | 55.180 | storage 内的主要耗时点 |
| serialization | 0.016 | 1.394 | 不是主要瓶颈 |
| response-write | 0.000459 | 0.000773 | 不是主要瓶颈 |

`storage outer` 包含 `etcd Txn`。扣除嵌套 Txn 后，storage outer 的独占耗时 P50 为 0.028 ms、P95 为 1.155 ms。因此不能把 42.741 ms 和 38.578 ms 相加；正确理解是 storage outer 的大部分时间都在等待 etcd Txn。

同一批慢请求中同时存在两类情况。一类主要慢在 APF queue，请求尚未进入实际执行阶段；另一类 APF queue 很短，但 storage/etcd Txn 较长，说明请求放行后在写事务路径等待。阶段级 Trace 的价值在于可以区分这两类慢请求，而不是只看到一个服务端总耗时。

## etcd Txn 为什么耗时较长

etcd Txn 记录的是 kube-apiserver 从调用 `OptimisticPut` 开始，到 etcd 返回事务结果为止的墙钟时间。它不仅包含 MVCC 数据修改，也包含 gRPC 和 Go 调度等待、Raft proposal 排队、WAL 持久化、Raft commit、backend apply 以及响应返回。

Namespace 名称虽然不同，但写请求仍然要进入同一个 etcd Raft 日志并按顺序提交。200 个并发 CREATE 快速到达时，不同对象之间没有业务数据冲突，也仍然会争用同一条写入流水线。etcd 可以通过批处理提高整体吞吐量，但单个请求可能等待当前批次提交和应用，因此会出现吞吐量较高、单请求延迟同时上升的现象。

本次是单节点 kind 环境，不存在等待远端多数派副本网络确认的问题，但仍然存在本地 Raft proposal、WAL 持久化和 backend apply。etcd 与 kube-apiserver 还共享 Docker Desktop 虚拟机中的 CPU 和容器存储层，高并发下的 CPU 调度及存储同步抖动也会计入 Txn 墙钟时间。因此 38～55 ms 不能直接解释为一次 `fsync` 耗时 38～55 ms。

如果需要继续拆分 etcd Txn，应同步观察 `wal_fsync_duration_seconds`、`backend_commit_duration_seconds`、`proposals_pending`、`proposals_committed_total` 和 `proposals_applied_total`。如果 WAL fsync 与 backend commit 都很短，而 Txn 仍然较长，排查重点应转向 proposal 排队、批处理和 CPU 调度，而不是直接判断为磁盘慢。

## 未解释时间

本次将“未解释时间”定义为服务端整请求区间减去所有已记录阶段区间的时间并集。最慢 20 条请求中，未解释时间 P50 为 6.230 ms、P95 为 33.539 ms。它可能包含 handler 路由、请求体读取、解码与转换、对象元数据处理、策略钩子，以及 Go 调度抢占造成的阶段间空档。

未解释时间不是将各阶段数值简单相加后的残差。storage outer 包含 etcd Txn，serialization 也可能包含 response write，必须先合并重叠区间再计算。客户端墙钟还包含连接、TLS、客户端线程调度和响应读取，因此也不能直接参与服务端阶段加总。

## 插桩使用边界

该源码插桩用于临时诊断，不应未经评估直接作为长期生产改动。阶段日志应限制到指定请求，避免高并发下产生大量日志；阶段数组应设置合理上限；请求头中的 Trace ID 只用于关联，不能直接信任为安全标识；完成验证后应恢复官方 kube-apiserver。

本次验证结束后已删除测试 Namespace 和临时 APF、鉴权对象，kube-apiserver 已恢复为 Kubernetes v1.34.0 官方镜像，`/readyz` 检查通过。