# kube-apiserver APF实现原理

## 结论

APF（API Priority and Fairness）可以理解为 kube-apiserver 门口的“分级排队系统”。它先使用 FlowSchema 判断请求属于哪一类，再把请求送到对应的 PriorityLevel；如果该优先级还有可用 Seat，请求立即执行，否则进入队列；队列也满时返回 HTTP 429。

APF 控制的是 kube-apiserver 的并发执行能力，不是传统的每秒请求数限流。它不能让 etcd、Admission 或 LIST 本身执行得更快，但可以在过载时限制同时执行的请求，防止普通高并发流量挤占节点心跳、系统控制器和管理员请求。

APF 中最容易混淆的三个概念是：`nominalConcurrencyShares` 表示相对权重，不是固定并发数；Seat 表示请求占用的执行能力，一个请求可能占用多个 Seat；客户端并发量不等于 kube-apiserver 内部 executing 数量，因为部分请求可能正在 APF 队列中等待。

## APF解决什么问题

假设 kube-apiserver 当前最多只能稳定处理一批并发请求，某个异常控制器突然持续发送大量 LIST。如果所有请求不分类地竞争同一个执行池，节点心跳、核心控制器以及管理员操作也可能被拖慢。

APF 会把请求分级并隔离。例如系统关键请求可以进入较高优先级，普通业务请求进入默认优先级。普通请求过多时，它们先在自己的 PriorityLevel 中排队，不会立即占满 kube-apiserver 的全部执行能力。

APF 的目标不是保证每个请求等待时间完全相同，而是在容量不足时实现优先级保护和流之间的相对公平。

## 一条请求如何经过APF

一条请求到达 kube-apiserver 后，APF 的处理过程如下。

| 阶段 | 作用 | 可能结果 |
|---|---|---|
| FlowSchema 匹配 | 根据用户、用户组、Verb、资源和 Namespace 对请求分类 | 找到关联的 PriorityLevel |
| 计算 Flow | 按用户或 Namespace 区分请求流 | 避免单个用户或 Namespace 垄断队列 |
| 估算 Seat | 判断请求需要占用多少执行能力 | 普通请求通常较少，大型 LIST 可能更多 |
| 检查 PriorityLevel 额度 | 判断当前是否有足够 Seat | 有额度则立即执行 |
| 排队 | 没有足够 Seat 时进入队列 | 等待其他请求释放 Seat |
| 拒绝 | 候选队列已经达到长度上限 | 返回 HTTP 429 |
| 公平分发 | Seat 释放后选择下一条流的请求 | 请求开始执行 |
| 释放 Seat | 请求完成或相应阶段结束 | 后续请求得到执行机会 |

因此，客户端看到的请求耗时通常可以理解为：

```text
请求耗时
= APF 排队时间
+ 被放行后的执行时间
+ 网络传输和客户端处理时间
```

其中，被放行后的执行时间还可能包含认证后的对象处理、Admission、Managed Fields、storage、etcd、序列化和响应写回。

## FlowSchema负责给请求分类

FlowSchema 类似机场的旅客分流规则。它可以根据请求主体和请求内容进行匹配，包括用户、用户组、资源请求、非资源请求、API Group、资源类型、Verb 和 Namespace。

多个 FlowSchema 可能同时匹配一个请求。APF 按 `matchingPrecedence` 选择，数值越小，匹配优先级越高。找到第一个匹配项后，请求就进入该 FlowSchema 指向的 PriorityLevel。

下面是一个简化示例。参数含义直接写在配置注释中。

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: example-workload
spec:
  # 数值越小越先匹配，应避免和更高优先级规则产生意外交叉。
  matchingPrecedence: 1000

  # 请求按用户区分成不同 Flow，降低单个用户影响其他用户的概率。
  distinguisherMethod:
    type: ByUser

  # 匹配后进入名为 example-limited 的 PriorityLevel。
  priorityLevelConfiguration:
    name: example-limited

  rules:
    - subjects:
        - kind: Group
          group:
            name: system:serviceaccounts
      resourceRules:
        - apiGroups: ["*"]
          resources: ["*"]
          verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
          namespaces: ["*"]
          clusterScope: true
```

`distinguisherMethod` 决定如何把匹配同一 FlowSchema 的请求进一步拆成不同 Flow。常见方式是 `ByUser` 和 `ByNamespace`。如果按用户区分，不同用户更容易进入不同调度流；如果按 Namespace 区分，不同 Namespace 的请求更容易相互隔离。

## PriorityLevel负责分配执行能力

FlowSchema 完成分类后，请求进入对应的 PriorityLevelConfiguration。PriorityLevel 可以理解为不同等级的候车区。

`Exempt` 类型不参与普通 APF 并发限制，主要服务非常关键的系统请求。它不代表 CPU、内存或网络真的无限，只表示请求不会被普通 APF 队列限制。

`Limited` 类型受到并发限制。大多数请求都属于这种类型。额度不足时，请求根据配置选择排队或直接拒绝。

下面是一个简化的 Limited PriorityLevel。

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: example-limited
spec:
  type: Limited
  limited:
    # 相对权重，不表示固定允许 100 个并发请求。
    nominalConcurrencyShares: 100

    # 允许把部分暂时未使用的名义额度借给其他 PriorityLevel。
    lendablePercent: 20

    # 最多可以从其他 PriorityLevel 借入多少额外额度。
    borrowingLimitPercent: 50

    limitResponse:
      type: Queue
      queuing:
        # 该 PriorityLevel 维护的队列总数。
        queues: 64

        # 每条 Flow 从 64 个队列中选取 8 个候选队列。
        handSize: 8

        # 每个队列最多允许等待的请求数，超过后返回 HTTP 429。
        queueLengthLimit: 50
```

如果 `limitResponse.type` 使用 `Reject`，额度不足时请求不会排队，而是直接返回 HTTP 429。使用 `Queue` 时，请求只有在候选队列也已满后才被拒绝。

## nominalConcurrencyShares为什么不是固定并发数

`nominalConcurrencyShares` 是 PriorityLevel 之间分配执行能力的相对权重。

假设 kube-apiserver 可供 APF 使用的总执行能力为 600 个 Seat，三个 Limited PriorityLevel 的 shares 分别为 100、50 和 50。忽略借入借出时，它们的名义额度大致按照 50%、25% 和 25% 分配。

真实额度还会受到服务器总并发限制、其他 PriorityLevel、借入借出配置以及请求 Seat 数量影响。因此，看到 `nominalConcurrencyShares: 30`，不能解释成最多执行 30 个请求。

当某个 PriorityLevel 没有使用完自己的名义额度时，可以通过 `lendablePercent` 让出一部分。其他 PriorityLevel 可以临时借用这些空闲额度，借入上限由 `borrowingLimitPercent` 控制。这样既能保留优先级隔离，也能减少空闲执行能力浪费。

## Seat表示请求占用多少执行能力

APF 不只计算请求数量，还会估算请求需要占用多少 Seat。Seat 可以理解为执行大厅里的座位。

普通的小请求通常占用较少 Seat。返回大量对象的 LIST 需要更多内存、序列化和网络写回工作，可能占用更多 Seat。假设某个 PriorityLevel 当前有 100 个 Seat，100 个单 Seat 请求可以占满额度，20 个各占 5 Seat 的请求也可以占满额度。

因此，以下三个值不能直接画等号：

```text
客户端并发请求数
≠ APF executing 请求数
≠ 当前占用的 Seat 总数
```

客户端设置并发 1000，只表示最多维持约 1000 个在途请求。其中一部分可能正在执行，一部分可能在 APF 中排队，还有一部分可能因队列已满而收到 HTTP 429。

## 没有Seat时如何排队

APF 不会简单地把一个 PriorityLevel 的所有请求放入同一个 FIFO 队列。它维护多个队列，并使用 Shuffle Sharding 限制单条 Flow 的影响范围。

一条 Flow 会根据自身标识选出 `handSize` 个候选队列，再从候选集合中选择合适的队列。某个高流量用户通常只会持续影响自己命中的部分队列，而不容易堵住该 PriorityLevel 的全部队列。

`queues` 越多，流之间的隔离空间越大，但管理成本也会上升。`handSize` 越大，一条 Flow 可选择的队列越多，更容易找到短队列，但不同 Flow 相互重叠的概率也会增加。`queueLengthLimit` 决定单队列最多积压多少请求，达到上限后请求被拒绝。

## APF如何实现公平调度

Seat 释放后，APF 不会持续从请求最多的 Flow 中取请求。它通过 QueueSet 的公平排队机制，在不同 Flow 之间计算调度顺序，使多个 Flow 都有机会获得执行能力。

可以把它理解为多个窗口轮流叫号。某个用户即使积压了大量请求，也不能一直占用所有后续执行机会；另一个只有少量请求的用户仍然有机会被选中。

这里的公平是长期执行机会的相对公平，不保证所有请求的排队时间完全相同。请求占用的 Seat、进入时间、所在队列和当前可用额度都会影响实际等待时间。

## APF和请求实际执行的关系

APF 只决定请求何时被放行，不负责优化放行后的内部执行路径。

如果一个 CREATE 请求被放行后，etcd Txn 仍需等待 50 ms，APF 不能消除这 50 ms。如果一个一致性 LIST 被放行后需要等待 watch cache 追平目标 ResourceVersion，APF 也不能消除该等待。

APF 的价值是限制同时进入执行阶段的请求，防止更多请求继续放大 CPU、内存、goroutine、网络和 etcd 压力。

分析慢请求时，需要把总耗时拆成两个主要部分：

```text
服务端请求耗时
≈ APF queue wait
+ APF 放行后的执行耗时
```

APF queue wait 很长，说明执行额度长期被占用。此时还需要继续分析已被放行的请求为什么长时间占用 Seat，例如 CPU 饱和、watch cache 新鲜度等待、Admission Webhook、etcd Txn 或响应写回。

## 与高并发CREATE测试的关系

在 Namespace CREATE 压测中，`c=200` 时服务端平均耗时约为 34.021 ms，其中 APF 平均排队约为 13.013 ms，被放行后的处理时间约为 21.008 ms。

阶段级 Trace 中，最慢 20 条请求的 APF queue P50 为 26.961 ms、P95 为 68.096 ms。这表示部分尾部请求仅在 APF 队列里就可能等待约 68 ms，尚未计算后续 storage 和 etcd Txn。

同一批慢请求中也存在 APF queue 很短、etcd Txn 很长的情况。因此，不能看到服务端总耗时较长就直接判断 APF 是唯一瓶颈，必须对同一请求分别记录排队阶段和放行后的执行阶段。

## 与高并发LIST测试的关系

在 Namespace LIST 压测中，100 并发时 APF 几乎没有排队，但服务端平均耗时已经从低并发的约 1.54 ms 增长到约 8.06 ms。这说明请求执行阶段的 CPU、调度、内存分配、watch cache 和响应写回竞争先于 APF 排队出现。

500 并发时 APF 平均等待约 20.17 ms，最大排队请求数达到 171；1000 并发时平均等待约 110.97 ms，最大排队数达到 300，并有 6171 个请求因 queue full 返回 HTTP 429。

因此，APF 是过载后的保护和调度机制，不一定是延迟最初增长的原因。分析时应同时观察 APF 指标和请求执行路径。

## 如何观测APF

诊断 APF 时，最直接的指标如下。

| 指标 | 含义 | 使用方式 |
|---|---|---|
| `apiserver_flowcontrol_current_inqueue_requests` | 当前正在队列中等待的请求数 | 压测期间周期采样，观察峰值 |
| `apiserver_flowcontrol_current_executing_requests` | 当前已经被放行的请求数 | 结合 PriorityLevel 和 FlowSchema 标签观察 |
| `apiserver_flowcontrol_request_wait_duration_seconds` | 请求在 APF 队列中的等待时间 | 使用 histogram 分位数或 sum/count 增量 |
| `apiserver_flowcontrol_rejected_requests_total` | APF 拒绝的请求数量 | 重点关注 `queue-full` 等原因 |
| `apiserver_flowcontrol_nominal_limit_seats` | PriorityLevel 的名义 Seat 数 | 判断名义执行额度 |
| `apiserver_flowcontrol_demand_seats` | PriorityLevel 对 Seat 的需求 | 判断需求是否长期超过额度 |

`current_inqueue_requests` 和 `current_executing_requests` 是瞬时 Gauge。压测前后查看时，它们可能都为 0，因此必须在压力窗口内持续采样。排队耗时和拒绝请求数属于累计指标，分析单轮压测时需要计算前后增量。

可以在压测期间直接读取 kube-apiserver 指标：

```bash
kubectl get --raw='/metrics'
```

定位问题时，可以先判断 inqueue 是否持续增长、executing 是否达到稳定上限，再观察 wait duration 和 rejected requests。如果 APF 排队明显，还要结合 CPU Profile、goroutine Profile、Block Profile、etcd 指标和阶段级 Trace，分析哪些已执行请求长期占用 Seat。

## 配置APF时需要注意什么

FlowSchema 的匹配范围不应过宽，否则重要请求可能被错误归入普通优先级，普通请求也可能意外进入高优先级。修改前应先确认现有 FlowSchema 的 `matchingPrecedence` 和匹配交集。

PriorityLevel 的 shares 不宜直接按照期望并发数填写，因为它表达的是相对权重。配置时应同时考虑 kube-apiserver 总执行能力、其他 PriorityLevel 的 shares、请求 Seat 数量和借入借出关系。

队列容量过大虽然可以减少 HTTP 429，但会把过载转换成长时间排队，使客户端更晚才得到失败结果。队列容量过小则可能在短时突发时过早拒绝请求。应结合客户端超时、重试策略和系统可接受延迟进行设置。

高优先级不等于无限容量。大量请求进入高优先级后，同样会消耗 CPU、内存、网络和后端存储。`Exempt` 应只用于确实不能被普通 APF 阻塞的关键请求。

修改 APF 配置后，应同时验证关键请求延迟、普通请求吞吐、APF 排队时间、HTTP 429、CPU、内存和 etcd 延迟，避免只优化某一个指标而把压力转移到其他组件。