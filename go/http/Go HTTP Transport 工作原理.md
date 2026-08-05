# Go HTTP Transport 工作原理

## 结论

Go 标准库 `net/http` 中的 `http.Transport` 不是一条具体的网络连接，而是 HTTP 客户端的底层传输管理器。它负责建立连接、TLS 握手、发送请求、接收响应，以及维护和复用连接池。

同一个 `Transport` 可以访问不同地址，因为它会按照代理、协议和目标地址等信息，把连接划分到不同的连接池中。不同地址共享的是同一个连接管理器及其配置，通常不是同一条 TCP 连接。

应用程序应长期复用 `http.Client` 和 `http.Transport`，并在请求结束后读取、关闭 `Response.Body`。频繁创建 Transport 会导致连接池无法复用，增加 DNS、TCP 和 TLS 握手开销。

## Client 与 Transport 的职责

`http.Client` 负责较上层的 HTTP 调用策略，例如重定向、Cookie 和整个请求的超时控制；`http.Transport` 负责实际的网络传输，例如 DNS 解析、TCP 建连、TLS 握手、代理、连接复用和 HTTP/2 多路复用。

```go
package main

import (
    "net"
    "net/http"
    "time"
)

func newHTTPClient() *http.Client {
    transport := &http.Transport{
        // 限制所有目标地址合计保留的空闲连接数。
        MaxIdleConns: 200,

        // 限制单个目标地址保留的空闲连接数。
        MaxIdleConnsPerHost: 50,

        // 限制单个目标地址的连接总数，包括使用中、建连中和空闲连接。
        MaxConnsPerHost: 100,

        // 空闲连接超过该时间后会被关闭。
        IdleConnTimeout: 90 * time.Second,

        // 限制 TLS 握手时间。
        TLSHandshakeTimeout: 10 * time.Second,

        // 请求发送完成后，限制等待响应头的时间。
        ResponseHeaderTimeout: 10 * time.Second,

        DialContext: (&net.Dialer{
            // 限制建立 TCP 连接的时间。
            Timeout: 5 * time.Second,

            // 配置 TCP Keep-Alive 探测间隔。
            KeepAlive: 30 * time.Second,
        }).DialContext,
    }

    return &http.Client{
        // Client 可以长期共享这个 Transport。
        Transport: transport,

        // 限制一次完整 HTTP 请求的总时间。
        Timeout: 30 * time.Second,
    }
}
```

Transport 实现了 `http.RoundTripper` 接口，其核心入口是 `RoundTrip`：

```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

`RoundTrip` 接收一个请求，完成一次请求与响应的底层交互，并返回 `http.Response`。它只负责单次往返，不负责重定向等 Client 层策略。

## 一次请求的执行过程

调用 `client.Do(req)` 后，请求会交给 Transport 的 `RoundTrip` 方法。Transport 首先根据请求计算连接匹配信息，然后尝试获取可用连接。

如果连接池中存在匹配的空闲连接，Transport 会优先复用；如果没有，它会执行 DNS 解析和 TCP 建连。HTTPS 请求还需要进行 TLS 握手。获得连接后，Transport 将请求头和请求体写入连接，并读取服务器返回的响应头。

当 `client.Do` 返回时，响应头通常已经读取完成，但响应体不一定已经全部下载。响应体会随着调用方读取 `Response.Body` 而继续传输。

```go
resp, err := client.Get("https://example.com/data")
if err != nil {
    return err
}
defer resp.Body.Close()

// 读取响应体时，底层网络数据可能仍在持续到达。
body, err := io.ReadAll(resp.Body)
if err != nil {
    return err
}
```

## 为什么不同地址可以使用同一个 Transport

Transport 是连接管理器，不是单条连接。它内部能够同时维护多个目标地址对应的多组连接。例如，一个 Transport 可以同时管理 `api.example.com:443`、`storage.example.com:443` 和 `127.0.0.1:8080` 的连接。

Transport 会根据请求生成连接池的匹配键。其概念可以简化为：

```go
type connectionKey struct {
    proxy  string // HTTP 代理信息
    scheme string // http 或 https
    addr   string // 目标主机和端口
}
```

实际实现细节比这个结构更复杂，但核心思想一致：代理、协议或目标地址不同的请求，通常会被划分到不同的连接分组中。

下面两个请求可以共享同一个 Transport：

```go
client := newHTTPClient()

// 两个请求共享 Transport 的配置和连接调度能力。
respA, err := client.Get("https://a.example.com/users")
respB, err := client.Get("https://b.example.com/orders")
```

它们对应不同的目标地址，因此通常使用不同的底层连接：

| 请求地址 | Transport | 连接分组 | 是否通常复用同一条 TCP 连接 |
| --- | --- | --- | --- |
| `https://a.example.com/users` | 相同 | `a.example.com:443` | 否 |
| `https://b.example.com/orders` | 相同 | `b.example.com:443` | 否 |

同一个地址下的不同路径通常属于相同的连接分组：

| 请求地址 | 连接分组 | 是否可以复用连接 |
| --- | --- | --- |
| `https://a.example.com/users` | `a.example.com:443` | 可以 |
| `https://a.example.com/orders` | `a.example.com:443` | 可以 |

因此，“复用 Transport”和“复用底层连接”是两个不同概念：复用 Transport 表示多个请求共享连接管理器；复用连接表示请求实际使用已有的 TCP 或 HTTP/2 连接。

## 连接池的工作方式

Transport 会保存请求结束后仍可使用的连接。后续请求到达时，它会优先选择与目标地址匹配的空闲连接，从而避免重复执行 DNS 解析、TCP 建连和 TLS 握手。

对于 HTTP/1.1，一条连接同一时刻通常只处理一个请求。请求完成后，连接回到空闲连接池，等待后续请求复用。并发请求较多时，Transport 可能为同一个目标地址建立多条连接。

对于 HTTP/2，一条 TCP 连接可以包含多个并发 Stream。多个请求能够同时复用同一条 HTTP/2 连接，因此不需要像 HTTP/1.1 那样通过大量 TCP 连接支撑并发。

在证书、DNS 和服务器能力等条件满足时，HTTP/2 还可能进行连接合并，使不同域名复用一条连接。这属于协议优化行为，不能作为普通请求的默认假设。通常应按“不同目标地址对应不同连接分组”理解。

## 为什么必须关闭 Response.Body

响应体与底层连接的生命周期密切相关。对于 HTTP/1.1，通常只有响应体读取完成并关闭后，Transport 才能确认连接可以安全回到空闲连接池。

```go
resp, err := client.Get("https://example.com/data")
if err != nil {
    return err
}
defer resp.Body.Close()

// 完整读取响应体，有利于底层连接被继续复用。
_, err = io.Copy(io.Discard, resp.Body)
return err
```

如果长期不关闭响应体，可能导致连接无法复用、文件描述符增加、并发请求持续创建新连接，最终出现连接资源或本地临时端口耗尽。

如果业务不需要响应内容，也应该根据场景消费响应体并将其关闭。仅调用 `Close` 是否能继续复用连接，与协议版本、响应状态和剩余响应体等条件有关，因此不要依赖未读取响应体时一定能够复用连接。

## 常见参数的区别

| 参数 | 作用 |
| --- | --- |
| `MaxIdleConns` | 所有目标地址合计最多保留的空闲连接数 |
| `MaxIdleConnsPerHost` | 单个目标地址最多保留的空闲连接数 |
| `MaxConnsPerHost` | 单个目标地址允许存在的连接总数 |
| `IdleConnTimeout` | 空闲连接最多保留的时间 |
| `TLSHandshakeTimeout` | TLS 握手的最长等待时间 |
| `ResponseHeaderTimeout` | 请求发送后等待响应头的最长时间 |
| `http.Client.Timeout` | 一次完整 HTTP 请求的总时限 |

这里的 Host 更接近 Transport 内部用于连接管理的目标分组，不应简单理解为 URL 中不带端口的域名。

## 使用建议

Transport 的连接池属于 Transport 实例本身，因此不要为每次请求创建新的 Transport。推荐在应用初始化阶段创建一次 Client 和 Transport，然后让多个 goroutine 长期复用。

`http.Transport` 支持并发使用，但开始处理请求后，不应再并发修改其配置字段。如果不同业务需要不同的代理、TLS 配置、连接上限或超时策略，可以分别创建 Transport；如果只是请求地址不同，通常没有必要拆分。

排查连接数异常时，应重点检查 Client 或 Transport 是否被频繁创建、`Response.Body` 是否正确关闭、响应体是否未读取完成、连接池参数是否过小，以及服务端是否主动关闭 Keep-Alive 连接。