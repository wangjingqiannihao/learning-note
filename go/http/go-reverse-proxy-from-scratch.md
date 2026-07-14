# Go 自己实现 Reverse Proxy

Reverse Proxy 的核心作用，是在服务端接收客户端请求，再由代理程序把请求转发给上游服务，并将上游返回的响应再转回给客户端。对调用方来说，它访问的是代理地址；对上游来说，请求像是由代理发起的。实际项目里，反向代理通常会承担统一入口、路由转发、鉴权、灰度、限流、观测以及隐藏内部服务地址等职责。

自己实现一个简化版 Reverse Proxy，可以把它拆成四件事来看。第一，复制并改写入站请求，准备发给目标服务。第二，决定目标地址，也就是把原始请求的 `Scheme`、`Host`、`Path` 等信息改成上游服务需要的样子。第三，补充或透传常见代理头，比如 `X-Forwarded-For`、`X-Forwarded-Host` 和 `X-Forwarded-Proto`，这样上游可以知道真实来源。第四，把上游响应的状态码、响应头和响应体原样写回给客户端。

如果只看数据流，过程其实很直接：客户端先请求代理服务，代理服务基于原请求构造一个新的出站请求，然后用 `http.Transport` 发到上游，再把上游返回的结果回写给客户端。这里最容易出错的点，通常有三个：一是请求头复制不完整，导致鉴权、Cookie 或内容类型丢失；二是 URL 拼接不正确，尤其是 `Path` 和 `RawQuery` 处理不当；三是没有正确处理超时、连接复用和错误返回。

下面给一个完整可运行的示例。这个示例不依赖 `httputil.ReverseProxy`，而是直接使用标准库 `net/http` 手动实现转发逻辑，适合理解原理。

## 完整示例

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net"
    "net/http"
    "net/url"
    "strings"
    "time"
)

type ReverseProxy struct {
    target    *url.URL
    transport http.RoundTripper
}

func NewReverseProxy(rawTarget string) (*ReverseProxy, error) {
    target, err := url.Parse(rawTarget)
    if err != nil {
        return nil, err
    }

    transport := &http.Transport{
        Proxy: http.ProxyFromEnvironment,
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        ForceAttemptHTTP2:     true,
        MaxIdleConns:          100,
        IdleConnTimeout:       90 * time.Second,
        TLSHandshakeTimeout:   5 * time.Second,
        ExpectContinueTimeout: 1 * time.Second,
    }

    return &ReverseProxy{
        target:    target,
        transport: transport,
    }, nil
}

func (p *ReverseProxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    outReq := r.Clone(context.Background())
    rewriteRequestURL(outReq, p.target)
    copyHeaders(outReq.Header, r.Header)
    setForwardHeaders(outReq, r)

    resp, err := p.transport.RoundTrip(outReq)
    if err != nil {
        http.Error(w, fmt.Sprintf("proxy request failed: %v", err), http.StatusBadGateway)
        return
    }
    defer resp.Body.Close()

    copyHeaders(w.Header(), resp.Header)
    w.WriteHeader(resp.StatusCode)

    if _, err := io.Copy(w, resp.Body); err != nil {
        log.Printf("copy response body failed: %v", err)
    }
}

func rewriteRequestURL(outReq *http.Request, target *url.URL) {
    outReq.URL.Scheme = target.Scheme
    outReq.URL.Host = target.Host
    outReq.Host = target.Host

    targetPath := strings.TrimSuffix(target.Path, "/")
    requestPath := strings.TrimPrefix(outReq.URL.Path, "/")

    switch {
    case targetPath == "":
        outReq.URL.Path = "/" + requestPath
    case requestPath == "":
        outReq.URL.Path = targetPath
    default:
        outReq.URL.Path = targetPath + "/" + requestPath
    }

    if outReq.URL.Path == "" {
        outReq.URL.Path = "/"
    }
}

func setForwardHeaders(outReq *http.Request, inReq *http.Request) {
    clientIP, _, err := net.SplitHostPort(inReq.RemoteAddr)
    if err == nil {
        prior := inReq.Header.Get("X-Forwarded-For")
        if prior != "" {
            clientIP = prior + ", " + clientIP
        }
        outReq.Header.Set("X-Forwarded-For", clientIP)
    }

    outReq.Header.Set("X-Forwarded-Host", inReq.Host)
    if inReq.TLS != nil {
        outReq.Header.Set("X-Forwarded-Proto", "https")
    } else {
        outReq.Header.Set("X-Forwarded-Proto", "http")
    }

    outReq.RequestURI = ""
}

func copyHeaders(dst, src http.Header) {
    for k, vv := range src {
        dst.Del(k)
        for _, v := range vv {
            dst.Add(k, v)
        }
    }
}

func main() {
    proxy, err := NewReverseProxy("http://127.0.0.1:9090")
    if err != nil {
        log.Fatalf("create proxy failed: %v", err)
    }

    mux := http.NewServeMux()
    mux.Handle("/", proxy)

    log.Println("reverse proxy listening on :8080, upstream is http://127.0.0.1:9090")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatalf("proxy server failed: %v", err)
    }
}
```

## 怎么运行

先启动一个上游服务，例如：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "upstream response, path=%s\n", r.URL.Path)
    })

    log.Println("upstream listening on :9090")
    log.Fatal(http.ListenAndServe(":9090", nil))
}
```

分别保存为 `upstream.go` 和 `proxy.go` 后，可以直接运行：

```bash
go run upstream.go
```

```bash
go run proxy.go
```

然后访问：

```bash
curl http://127.0.0.1:8080/hello
```

如果一切正常，会看到上游服务返回的内容，但入口地址是代理服务的 `:8080`。

## 这个实现的关键点

这个版本的实现，重点是把标准库里隐藏的几个动作手动展开了。`r.Clone` 用来基于原请求复制出一个新的请求对象，避免直接修改入站请求。`rewriteRequestURL` 负责把请求改写到目标上游。`RoundTrip` 负责真正把请求发出去。最后通过 `io.Copy` 把响应体流式写回客户端，这样不会一次性把整个响应读进内存。

从工程角度看，这个版本已经能说明 Reverse Proxy 的基本机制，但还不是生产级实现。真实场景里通常还会继续补充这些能力：比如更细的超时控制、重试策略、连接池参数调优、WebSocket 或 SSE 透传、对跳转和压缩头的兼容、按路由转发到不同上游，以及更完整的错误日志和观测埋点。

## 和 `httputil.ReverseProxy` 的关系

标准库里的 `httputil.ReverseProxy`，本质上也是在做同一件事，只是把请求改写、Header 处理、错误处理、响应拷贝这些细节都封装好了。自己手写一次的价值，不是为了替代标准库，而是为了真正理解它内部在做什么。理解了这版最小实现，再去看 `httputil.ReverseProxy` 的 `Director`、`Rewrite`、`Transport`、`ModifyResponse` 等扩展点，会更容易把它用对。
