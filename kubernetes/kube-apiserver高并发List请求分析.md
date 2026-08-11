# kube-apiserver高并发List请求分析

> 本文基于 Kubernetes `v1.34.0` 和本地 kind 单控制面集群验证，分析持续 Namespace LIST 请求从低并发到过载过程中的延迟变化，并结合 CPU、goroutine 和 Block Profile 定位执行资源消耗。测试请求为 `GET /api/v1/namespaces`，压测程序使用 kubeconfig 客户端证书，通过 TLS/HTTP/2 直连 kube-apiserver。文中的压测数据和结论仅适用于该 LIST 场景，不能直接外推到单对象 GET、CREATE、UPDATE、DELETE 或 WATCH。

## 结论

低并发时，Namespace LIST 的 kube-apiserver 服务端平均耗时约为 `1.5~2.1 ms`。使用 `kubectl` 观察到的约 `40 ms` 端到端耗时大部分来自进程启动、读取 kubeconfig、连接初始化、TLS 和客户端编解码，不能直接归因于 kube-apiserver。

并发提升后，延迟增长并非由单一组件引起。100 并发时 APF 几乎没有排队，服务端平均耗时已经从约 `1.5 ms` 增长到约 `8 ms`，说明请求执行阶段首先出现 CPU、调度、内存分配和网络写回竞争。500 并发时 APF 开始明显排队；1000 并发时队列达到容量并返回 HTTP 429。

Profile 显示，高并发一致性 LIST 的主要放大路径是 watch cache 新鲜度等待。其原因是，一致性 LIST 使用的目标 ResourceVersion 对应 etcd 全局 revision；即使没有 Namespace 变更，Node Lease、选主 Lease、Pod 状态、Event 等其他资源的后台写入仍会持续推进全局 revision。Namespace watch cache 需要通过事件或 watch progress notification 证明自己已观察到目标 revision，并确认期间没有遗漏 Namespace 事件；cache 尚未追平时，请求会停留在 `watchCache.waitUntilFreshAndBlock`。采样时约 2.4 万个 goroutine 等待该路径，同时内存分配速率接近 `486 MB/s`，kube-apiserver 平均消耗约 `6.44` 个 CPU 核。APF 的 `queueSet` 核心锁是最明显的 Mutex 竞争点，但总阻塞时间仍远小于 watch cache 的 channel 等待。

因此，分析单次慢请求时必须区分客户端端到端时间、APF 排队时间、kube-apiserver 执行时间和响应写回时间；分析高并发退化时，则需要同时观察 APF、CPU、GC、goroutine、watch cache 一致性等待和存储进度协同。

## 测试环境

| 项目 | 配置 |
|---|---|
| 主机 | Apple M3 Pro，12 核（6 个性能核、6 个能效核），36 GB 内存 |
| Docker Desktop Linux VM | 12 个逻辑 CPU |
| kind | 单 control-plane 节点 |
| Kubernetes | v1.34.0，linux/arm64 |
| kubectl | v1.34.1，darwin/arm64 |
| Admission Webhook | 未配置 |
| APF | 系统默认配置 |
| Audit | 未配置 |
| OpenTelemetry Tracing | 未配置 |
| kube-apiserver CPU quota | 未设置 |

kind 的 control-plane 是一个 Docker 容器，容器中同时运行 kube-apiserver、etcd、kube-controller-manager 和 kube-scheduler。因此 `docker stats` 展示的是整个 control-plane 容器的资源使用量；kube-apiserver 自身 CPU 需要结合 `process_cpu_seconds_total` 和 CPU Profile 计算。

## 并发量的含义

并发量表示同一时刻最多维持多少个正在执行或等待返回的请求，不表示所有请求会在完全相同的时刻到达。例如 `c=500、n=10000` 表示压测程序启动 500 个执行单元，最多同时存在约 500 个在途请求，整轮累计发送 10000 个请求。开始时第一批约 500 个请求会快速发出；某个请求返回后，对应执行单元立即发送下一个请求，持续滚动直到完成目标请求数。

| 参数 | 含义 |
|---|---|
| `c=500` | 同一时刻最多维持约 500 个正在执行或等待返回的 LIST 请求 |
| `n=10000` | 整轮测试累计发送 10000 个请求，不是一次性同时发送 10000 个请求 |
| `duration=45s` | Profile 场景不限制固定请求数，由各执行单元在 45 秒内持续循环发送请求 |
| RPS | 每秒实际完成的请求数，不等于并发量 |
| 请求延迟 | 单个请求从发出到返回，或服务端从接收到处理完成的墙钟时间 |

并发量也不等于 kube-apiserver 内部实际执行数量。请求到达后先经过 APF；当可执行额度已满时，剩余请求会进入 APF 队列，队列也满后返回 HTTP 429。因此设置 `c=1000` 只表示客户端最多维持约 1000 个在途请求，不表示 kube-apiserver 会同时执行 1000 个请求。

## 性能测试数据如何获得

本次 LIST 性能数据不是通过 Jaeger 采集。测试环境没有配置 OpenTelemetry Tracing，也没有部署 Jaeger。数据来自三部分：压测客户端记录的端到端结果、kube-apiserver `/metrics` 的服务端指标，以及 `/debug/pprof` 采集的 CPU、goroutine 和 Block Profile。

### 客户端延迟、状态码和 RPS

压测程序通过 TLS/HTTP/2 直连 kube-apiserver，并复用连接。每个执行单元在请求发出前记录 `time.Now()`，读取并关闭响应体后使用 `time.Since()` 得到端到端墙钟时间，同时记录 HTTP 状态码。压测结束后，将成功请求的耗时排序并计算 P50、P95 和 P99；HTTP 429 单独计数；成功 RPS 使用成功请求数除以压测持续时间计算。

```go
startedAt := time.Now()
resp, err := client.Do(req)
latency := time.Since(startedAt)

// 必须读取并关闭响应体，HTTP/2 连接才能稳定复用。
if resp != nil {
    _, _ = io.Copy(io.Discard, resp.Body)
    _ = resp.Body.Close()
}

// 记录 latency、resp.StatusCode，并在测试结束后计算分位数和 RPS。
```

客户端耗时包含客户端调度、请求传输、kube-apiserver 处理、响应传输和响应体读取，因此不能直接当作 kube-apiserver 服务端处理时间。

### kube-apiserver 服务端指标

每轮压测前后各保存一次 kube-apiserver `/metrics`，通过计数器增量计算该轮数据。采集命令如下，两个命令分别在压测前和压测后执行。

```bash
kubectl get --raw='/metrics' > metrics-before.txt
```

```bash
kubectl get --raw='/metrics' > metrics-after.txt
```

Namespace LIST 的服务端平均耗时通过 `apiserver_request_duration_seconds_sum` 与 `apiserver_request_duration_seconds_count` 的前后增量计算：

```text
服务端平均耗时
= Δapiserver_request_duration_seconds_sum
  / Δapiserver_request_duration_seconds_count
```

APF 平均排队时间使用 `apiserver_flowcontrol_request_wait_duration_seconds_sum` 和对应 count 的增量计算。`apiserver_flowcontrol_current_inqueue_requests` 与 `apiserver_flowcontrol_current_executing_requests` 是瞬时 Gauge，需要在压测期间定时抓取，不能只比较压测前后值。本轮表格中的最大排队数来自压测窗口内的周期采样最大值。

进程 CPU 使用量通过 `process_cpu_seconds_total` 的前后增量除以压测持续时间计算，结果表示该窗口平均使用的 CPU 核数。内存分配速率使用 `go_memstats_alloc_bytes_total` 的增量除以持续时间计算；`go_memstats_heap_alloc_bytes` 用于观察当前堆内存，而不是累计分配量。

### CPU、goroutine 和阻塞数据

pprof 必须在压测运行期间采集，否则只能看到空闲状态。CPU Profile 使用固定时间窗口；goroutine、Block 和 Mutex Profile 在同一压力窗口内抓取快照。

```bash
kubectl get --raw='/debug/pprof/profile?seconds=20' > cpu.pprof
```

```bash
kubectl get --raw='/debug/pprof/goroutine' > goroutine.pprof
```

```bash
kubectl get --raw='/debug/pprof/block' > block.pprof
```

```bash
kubectl get --raw='/debug/pprof/mutex' > mutex.pprof
```

采集后使用 `go tool pprof` 分析函数 CPU 时间、goroutine 停留位置和累计阻塞时间。本文中 `watchCache.waitUntilFreshAndBlock` 的等待 goroutine 数量和阻塞占比来自 goroutine Profile 与 Block Profile，而不是客户端延迟或 Jaeger Span。

整个采集链路可以表示为：

```text
压测客户端记录每次请求耗时和状态码
→ 压测前后抓取 /metrics 并计算 Counter 增量
→ 压测期间周期采样 APF Gauge
→ 同一压力窗口采集 pprof
→ 关联客户端延迟、服务端指标和代码调用栈
```

## 为什么不能只看 kubectl 耗时

串行创建 30 个 Namespace 时，客户端观测结果如下。

| 指标 | 耗时 |
|---|---:|
| 最小值 | 37.0 ms |
| P50 | 41.1 ms |
| P95 | 45.1 ms |
| 最大值 | 65.4 ms |
| 平均值 | 42.0 ms |

同一阶段，`apiserver_request_duration_seconds` 显示 Namespace POST 的服务端平均耗时约为 `1.98 ms`。两者测量边界不同：

```text
kubectl 端到端耗时
= kubectl 进程启动
+ kubeconfig 读取
+ 客户端初始化
+ 连接与 TLS
+ 请求传输
+ kube-apiserver 处理
+ 响应传输和客户端解码
```

通过常驻客户端复用连接后，Namespace LIST 的 P50 约为 `1.91 ms`，与服务端指标的约 `2.11 ms` 基本一致。这说明诊断 kube-apiserver 时应优先使用服务端指标、审计阶段注解或 Trace，而不是直接把 `time kubectl ...` 当作服务端耗时。

## 绕过 kubectl proxy 的直连压测

为了排除代理转发干扰，压测程序直接读取 kubeconfig 中的 API Server 地址、CA、客户端证书和客户端私钥，并在内存中构建 TLS 配置。证书不写入临时文件，请求使用 HTTP/2 复用连接。

下面是核心实现。代码省略了统计分位数部分，但保留了 kubeconfig 证书加载和并发请求逻辑。

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "encoding/base64"
    "fmt"
    "io"
    "net/http"
    "os/exec"
    "strings"
    "sync"
    "sync/atomic"
    "time"
)

// kubeValue 通过 kubectl 读取当前 context 对应的 kubeconfig 字段。
// --raw 用于返回证书原始 Base64 数据，而不是脱敏后的内容。
func kubeValue(path string) string {
    out, err := exec.Command("kubectl", "config", "view", "--minify", "--raw", "-o", "jsonpath="+path).Output()
    if err != nil {
        panic(err)
    }
    return strings.TrimSpace(string(out))
}

func decodeKubeData(path string) []byte {
    value, err := base64.StdEncoding.DecodeString(kubeValue(path))
    if err != nil {
        panic(err)
    }
    return value
}

func main() {
    const concurrency = 500
    const duration = 45 * time.Second

    // 从 kubeconfig 内存加载客户端证书和私钥。
    cert, err := tls.X509KeyPair(
        decodeKubeData("{.users[0].user.client-certificate-data}"),
        decodeKubeData("{.users[0].user.client-key-data}"),
    )
    if err != nil {
        panic(err)
    }

    // 使用 kubeconfig 中的集群 CA 验证 API Server 证书。
    roots := x509.NewCertPool()
    if !roots.AppendCertsFromPEM(decodeKubeData("{.clusters[0].cluster.certificate-authority-data}")) {
        panic("invalid kubeconfig CA")
    }

    transport := &http.Transport{
        TLSClientConfig: &tls.Config{
            Certificates: []tls.Certificate{cert},
            RootCAs:      roots,
            MinVersion:   tls.VersionTLS12,
        },
        // 自定义 TLSClientConfig 时显式启用 HTTP/2。
        ForceAttemptHTTP2:   true,
        MaxIdleConns:        concurrency * 2,
        MaxIdleConnsPerHost: concurrency * 2,
        MaxConnsPerHost:     concurrency * 2,
    }
    client := &http.Client{Transport: transport, Timeout: 15 * time.Second}
    url := kubeValue("{.clusters[0].cluster.server}") + "/api/v1/namespaces"
    deadline := time.Now().Add(duration)

    var ok, rejected, failed int64
    var wg sync.WaitGroup
    for i := 0; i < concurrency; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for time.Now().Before(deadline) {
                resp, err := client.Get(url)
                if err != nil {
                    atomic.AddInt64(&failed, 1)
                    continue
                }
                _, readErr := io.Copy(io.Discard, resp.Body)
                resp.Body.Close()

                switch {
                case readErr != nil:
                    atomic.AddInt64(&failed, 1)
                case resp.StatusCode == http.StatusOK:
                    atomic.AddInt64(&ok, 1)
                case resp.StatusCode == http.StatusTooManyRequests:
                    atomic.AddInt64(&rejected, 1)
                default:
                    atomic.AddInt64(&failed, 1)
                }
            }
        }()
    }
    wg.Wait()
    fmt.Printf("ok=%d 429=%d failed=%d\n", ok, rejected, failed)
}
```

直连低并发基线为：客户端 P50 `1.89 ms`，服务端平均 `1.54 ms`，APF 平均等待 `0.03 ms`。这说明直连测量链路正常。

## HTTP GET 与 Kubernetes LIST 语义

本轮压测虽然使用 HTTP GET 方法，但请求路径为：

```http
GET /api/v1/namespaces
```

它在 Kubernetes API 中对应 `verb=LIST`，并不是读取单个对象的 `verb=GET`。单对象 GET 的路径应类似 `GET /api/v1/namespaces/default`。

LIST 除了认证、鉴权、APF、路由和响应写回，还需要确认一致性 ResourceVersion、等待 watch cache 达到要求的新鲜度、遍历对象、构造 `NamespaceList` 并序列化整个列表。单对象 GET 同样存在认证、鉴权、APF、缓存读取、序列化和 TLS 的固定开销，但不需要构造对象列表，通常负载更轻。

| 处理阶段 | 单对象 GET | LIST |
|---|---|---|
| x509 认证、RBAC 和 APF | 有 | 有 |
| watch cache 一致性检查 | 可能有 | 有，影响更明显 |
| 返回对象数量 | 1 | 0 到 N |
| 列表遍历与构造 | 无 | 有 |
| 对象复制和转换 | 少 | 随对象数量增长 |
| 响应序列化 | 单对象 | 整个列表 |

### 没有创建 Namespace 时 ResourceVersion 为什么仍会变化

需要区分 Namespace 对象自身的 `metadata.resourceVersion` 与一致性 LIST 使用的目标 ResourceVersion。单个 Namespace 的 `metadata.resourceVersion` 只有在该对象发生变化时才会更新；但 etcd revision 是整个 Kubernetes 存储的全局单调递增版本，并不按资源类型分别计数。

即使没有创建或修改 Namespace，集群后台仍会持续更新其他资源，例如 Node 状态、Lease、EndpointSlice、Event、Pod 状态以及控制器的选主 Lease。任意存储写入都会推进 etcd 的全局 revision。

一致性 LIST 的简化过程如下：

```text
请求到达
→ 从 etcd 获取当时的全局 current revision，记为目标 RV
→ 检查 Namespace watch cache 已观察到的 RV
→ cache RV 小于目标 RV 时等待缓存追平
→ 满足 cache RV 大于等于目标 RV 后返回列表
```

对于某一次请求，已经取得的目标 RV 不会在等待过程中不断改变；变化的是后续新请求取得的目标值。由于 etcd 的全局 revision 持续前进，每个新 LIST 请求可能得到比前一个请求更大的目标 RV。

Namespace watch cache 只会直接收到 Namespace 相关事件。当全局 revision 因其他资源写入而推进、但 Namespace 没有变化时，它需要依赖 etcd watch progress notification 等机制确认自己已经观察到该 revision 且没有遗漏 Namespace 事件。高并发一致性 LIST 会同时等待这种“已追平”的证明，因此可能出现大量 goroutine 停留在 `watchCache.waitUntilFreshAndBlock`。

这并不表示 Namespace 一直在变化，而是表示一致性读取必须证明：在目标全局 revision 之前，没有尚未被 Namespace watch cache 观察到的相关事件。

## 并发阶梯压测结果

| 并发 | 请求数 | 成功 | 429 | 成功 RPS | P50 | P95 | P99 | 服务端平均 | APF 平均等待 | 最大排队 |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 100 | 100 | 0 | 368 | 1.89 ms | 5.49 ms | 12.70 ms | 1.54 ms | 0.03 ms | 0 |
| 100 | 5,000 | 5,000 | 0 | 4,582 | 17.30 ms | 52.23 ms | 70.39 ms | 8.06 ms | 0.59 ms | 0 |
| 500 | 10,000 | 10,000 | 0 | 3,791 | 94.12 ms | 421.90 ms | 545.11 ms | 82.88 ms | 20.17 ms | 171 |
| 1000 | 15,000 | 8,829 | 6,171 | 2,306 | 228.99 ms | 1159.68 ms | 1424.42 ms | 228.65 ms | 110.97 ms | 300 |

100 并发时 APF 尚未形成明显队列，但服务端平均耗时已经增长到 `8.06 ms`，说明执行阶段先出现资源竞争。500 并发时，APF 平均等待达到 `20.17 ms`；1000 并发时，队列达到容量，后续请求以 `reason="queue-full"` 返回 HTTP 429。

`nominalConcurrencyShares` 不是固定并发数。APF 会根据服务端总并发额度、各 PriorityLevel 的权重、借入借出额度和请求 seats 动态计算可执行数量。本轮观测到 `global-default` 的 executing 在约 330 附近停止增长，剩余请求进入队列。

可以使用 Little 定律验证在途请求规模：

```text
平均在途请求数 ≈ 吞吐量 × 服务端平均耗时
```

500 并发时：

```text
3791 × 0.08288 ≈ 314
```

该结果与最大 executing 约 330 基本一致，说明请求执行额度已经接近上限。

## 为什么并发提高后耗时非线性增长

当系统利用率接近 100% 时，延迟不会按并发量线性增长。简化的排队模型可以写为：

```text
平均响应时间 ≈ 单请求服务时间 / (1 - 系统利用率)
```

| 系统利用率 | 单请求服务时间为 2 ms 时的理论响应时间 |
|---:|---:|
| 50% | 4 ms |
| 80% | 10 ms |
| 90% | 20 ms |
| 95% | 40 ms |
| 99% | 200 ms |

### 墙钟时间与 CPU 时间

墙钟时间（wall-clock time）是从外部观察一段过程实际经过的时间，也就是从请求开始到请求结束，时钟向前走过了多久。它包含真正占用 CPU 执行的时间，也包含排队、调度、锁、channel、网络和 I/O 等等待时间。

CPU 时间只统计进程实际在 CPU 上运行的时间，不包括请求没有获得 CPU 或正在等待其他资源的时间。两者可以表示为：

```text
墙钟时间
= CPU 执行时间
+ 等待 CPU 调度的时间
+ APF 排队时间
+ 锁和 channel 等待时间
+ 网络与 I/O 等待时间
```

例如，一个请求真正需要的 CPU 工作量为 2 ms。在低并发时，它可能连续获得 CPU，墙钟时间接近 2 ms；在高并发时，它可能被多次调度出去，并等待 APF、watch cache、GC 或 Socket 写入。即使总 CPU 工作量仍接近 2 ms，墙钟时间也可能增长到几十甚至几百毫秒。

| 场景 | CPU 时间 | 额外等待 | 墙钟时间 |
|---|---:|---:|---:|
| 低并发，立即执行 | 2 ms | 约 0 ms | 约 2 ms |
| 高并发，等待调度和共享资源 | 2 ms | 48 ms | 约 50 ms |
| 过载，叠加 APF 排队 | 2 ms | 198 ms | 约 200 ms |

因此，看到请求延迟从 2 ms 增长到 100 ms，并不表示业务代码的计算量一定增长了 50 倍。更常见的情况是 CPU 工作量只增加了一部分，而大部分墙钟时间消耗在等待上。CPU Profile 用于分析 CPU 时间花在哪些函数，Block Profile、goroutine Profile 和 APF 指标则用于解释墙钟时间中的等待部分。

高并发下，一个请求即使只需要少量 CPU 时间，也可能在执行过程中多次被调度出去，等待其他 goroutine、GC、Socket 写入或共享状态。最终墙钟时间会远高于实际 CPU 时间。达到 APF 执行额度后，请求还会增加额外队列等待。

## Profile 采集方法

测试集群可在 kube-apiserver 静态 Pod 的 command 中加入：

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # 仅建议在测试环境临时启用，用于采集 channel、select、Cond 和锁等待。
    - --contention-profiling=true
```

修改静态 Pod 文件后，kubelet 会自动重建 kube-apiserver。应等待健康检查恢复后再压测：

```bash
kubectl get --raw='/readyz'
```

在压测窗口中采集 20 秒 CPU Profile：

```bash
kubectl get --raw='/debug/pprof/profile?seconds=20' > cpu.pprof
```

采集 goroutine 和 Block Profile：

```bash
kubectl get --raw='/debug/pprof/goroutine' > goroutine.pprof
kubectl get --raw='/debug/pprof/block' > block.pprof
kubectl get --raw='/debug/pprof/mutex' > mutex.pprof
```

查看热点：

```bash
go tool pprof -top -nodecount=40 cpu.pprof
go tool pprof -top -cum -nodecount=40 block.pprof
go tool pprof -top -cum -sample_index=goroutine goroutine.pprof
```

Block Profile 的 delay 是所有 goroutine 等待时间的累加，可能远大于压测墙钟时间。不能把 `123 goroutine-hours` 理解为某个请求等待了 123 小时。

## CPU 与内存分配结果

开启 contention profiling 后，以 500 并发持续压测 45 秒：

| 指标 | 结果 |
|---|---:|
| 成功请求 | 295,833 |
| HTTP 429 | 20,950 |
| 其他失败 | 0 |
| 成功吞吐 | 6,571 RPS |
| 成功请求平均耗时 | 75.14 ms |
| kube-apiserver 平均 CPU | 约 6.44 核 |
| control-plane 容器峰值 CPU | 约 9.43 核 |
| 内存分配速率 | 约 486 MB/s |

20.11 秒 CPU Profile 累计采样 `123.43 CPU 秒`，对应约 `6.14` 个 CPU 核。结合 `process_cpu_seconds_total` 的压测前后增量，45 秒窗口平均约使用 `6.44` 核。

CPU 热点主要包括：

| 路径 | 观察结果 |
|---|---|
| `runtime.mallocgc` | 累计约 15.77%，大量临时对象分配 |
| `runtime.scanobject` | 累计约 9.80%，GC 扫描明显 |
| `crypto/x509.Certificate.Verify` | 客户端证书链和签名验证占用明显 |
| `crypto/rsa.VerifyPKCS1v15` | x509 请求认证中的 RSA 校验 |
| `crypto/tls.Conn.Write` | TLS 响应加密与写回 |
| `go-restful.CurlyRouter` | API 路由匹配 |
| JSON 编码 | 响应对象序列化与 Compact |

45 秒内新增分配约 `21.85 GB`，平均接近 `486 MB/s`。这会增加 GC 扫描、写屏障、对象查找和 goroutine stack 扫描成本。

## watch cache 一致性等待

Block Profile 的累计 delay 约为 `123.96 goroutine-hours`，其中约 `84.69%` 落在：

```text
watchCache.waitUntilFreshAndBlock.func2
→ runtime.chanrecv1
```

采集 goroutine Profile 时：

| 指标 | 数量 |
|---|---:|
| 总 goroutine | 27,341 |
| 等待 `watchCache.waitUntilFreshAndBlock` | 24,149 |
| 占比 | 88.33% |

高并发一致性 LIST 会要求 watch cache 达到满足请求一致性的 ResourceVersion。如果缓存尚未追平，每个请求会进入等待路径。大量临时 goroutine 会增加 channel 等待、调度、Context 对象、stack 内存和 GC 扫描开销。

压测结束 15 秒后，goroutine 从 27,341 恢复到约 2,371，APF inqueue 恢复为 0。因此本轮观察到的是压力窗口内的临时积压，不是永久 goroutine 泄漏。

Profile 同时出现 `etcd3.store.GetCurrentResourceVersion`、`etcd3.store.GetList` 和 gRPC `RecvMsg` 路径。这说明使用 watch cache 不代表请求完全不与 etcd 协同。一致性 LIST 仍可能需要查询当前 ResourceVersion、等待缓存追平，或在特定条件下通过存储层读取。

## APF 锁竞争

Block Profile 共采样到约 `2,704,334` 次 contention：

| 类型 | 次数占比 |
|---|---:|
| `runtime.selectgo` | 82.59% |
| `sync.Mutex.Lock` | 9.50% |
| `runtime.chanrecv1` | 4.89% |
| `runtime.chansend1` | 1.64% |
| `sync.Cond.Wait` | 1.16% |

`sync.Mutex.Lock` 共出现约 256,808 次 contention，其中大部分来自 APF queueSet：

| APF 路径 | 采样次数 |
|---|---:|
| `queueSet.lockAndSyncTime` | 205,882 |
| `request.Finish` | 178,381 |
| `configController.startRequest` | 约 74,000 |
| `queueSet.StartRequest` | 约 74,000 |
| `finishRequestAndDispatchAsMuchAsPossible` | 约 71,000 |

每个请求进入 APF 时需要更新队列状态和虚拟时间，完成时需要归还执行额度并调度下一个请求。在每秒数千次请求和大量 429 的情况下，queueSet 核心锁会成为高频共享临界区。

不过，APF 路径约占总 Block Delay 的 `1.35%`。它是最明显的 Mutex 竞争点，但总体阻塞时间仍显著小于 watch cache 新鲜度等待。

独立的 Mutex Profile 在本轮没有获得有效样本，但 Block Profile 已记录 `sync.Mutex.Lock` contention。实际分析应以 Block Profile 为准，不能因为 `/debug/pprof/mutex` 为空就断言没有锁竞争。

## 延迟形成链路

本轮高并发退化可以归纳为以下过程：

```text
高并发一致性 LIST
→ 每个请求执行 x509 认证、路由、APF 分类和缓存读取
→ 请求查询或确认当前 ResourceVersion
→ watch cache 未达到一致性要求时进入新鲜度等待
→ 大量 goroutine 同时等待 channel
→ 临时对象和 goroutine stack 增加
→ 分配速率、GC 和调度成本上升
→ TLS/HTTP2 响应写回继续消耗 CPU
→ APF Start/Finish 高频竞争 queueSet 核心锁
→ executing 达到并发额度
→ 后续请求排队或返回 HTTP 429
```

这里的 APF 是过载后的保护机制，而不是最初导致延迟增长的唯一原因。100 并发、APF 尚未排队时，执行时间已经明显增加；500 并发以后，APF 排队和 queueSet 锁竞争进一步放大尾延迟。

## 单次慢请求排查顺序

| 现象 | 优先检查 |
|---|---|
| `apf-queue-wait` 明显升高 | FlowSchema、PriorityLevel、executing 和 inqueue |
| Webhook Span 明显升高 | Webhook 服务、DNS、TLS、网络和 `timeoutSeconds` |
| etcd Span 很短但根 Span 很长 | 认证、鉴权、准入、转换、序列化、审计和运行时调度 |
| Trace 父 Span 长、子 Span 都短、CPU 高 | CPU Profile、GC、序列化和证书认证 |
| Trace 父 Span 长、CPU 不高 | goroutine、Block Profile、网络写回和外部等待 |
| LIST 高并发出现大量 goroutine | watch cache 新鲜度和一致性读取路径 |
| 固定接近 1 秒、3 秒或 10 秒 | 超时、重试、DNS、TLS 建连或代理退避 |

## 相关上游优化

| Issue / PR | 方向 | 与本文关系 |
|---|---|---|
| Kubernetes #27667 | API Server、Controller 和 Client 性能总纲 | 涵盖 Protobuf、转换、缓存、锁和 GC |
| Kubernetes #119799 | APF 排队超过固定阈值 | 对应请求排队与取消传播 |
| Kubernetes PR #120222 | APF 使用请求 Context 管理排队生命周期 | 避免取消请求继续占用队列 |
| Kubernetes #79209 | 连接请求与底层 Trace | 解决根 Span 与存储、Webhook Trace 割裂 |
| Kubernetes #76219 | SSA Protobuf 序列化性能 | 深层对象和 managedFields 场景相关 |
| Kubernetes PR #77355 | 反向 Protobuf marshaling | 减少重复 Size 计算 |
| Kubernetes #110146 | JSON Watch Event 重复编码 | 高 Watch 数量下减少 CPU |
| Kubernetes PR #120300 | 缓存已编码 Watch Event | 多 Watcher 复用序列化结果 |
| Kubernetes #69540 | Admission Webhook 指标基数过高 | 降低指标内存与 GC 压力 |
| Kubernetes PR #69895 | 移除 Admission 指标资源标签 | 以较低维度换取可控开销 |

## 使用 Profile 时的注意事项

`--contention-profiling=true` 会引入额外采样开销，建议只在测试环境或短时间诊断窗口启用。采集完成后应恢复原始静态 Pod 配置，并再次确认 `/readyz`、APF 队列和 goroutine 数已经恢复。

CPU Profile 中的 cumulative 百分比存在调用链重叠，不能把多个筛选结果直接相加。Heap Profile 的 `alloc_space` 默认包含进程启动以来的累计分配，分析单轮压测时应同时读取 `go_memstats_alloc_bytes_total` 的前后增量。Block Profile 的 delay 是所有 goroutine 等待时间之和，必须结合 contentions 次数和 goroutine Profile 判断。

对于生产环境，应先用低采样率 Trace 和指标确定异常时间窗口，再在可控条件下短时间采集 pprof，避免长期开放高开销诊断能力。
