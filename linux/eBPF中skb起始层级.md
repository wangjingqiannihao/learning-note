# eBPF skb 是从哪一层开始的

结论：`skb` 不是从物理层开始的，而是从 Linux 内核构造出 `struct sk_buff` 之后开始存在。按网络分层理解，它大致对应二层以太网帧进入内核协议栈处理阶段之后的抽象。XDP 比 `skb` 更早，操作的是 `xdp_buff` 或原始 packet buffer；TC ingress/egress、socket filter、cgroup skb 等 hook 才是典型的 `skb` 语义。

本文记录一个常见问题：在 eBPF 网络程序里提到的 `skb`，到底是从网络协议栈的哪一层开始出现的。

## skb 的位置

`skb` 是 Linux 内核里的 `struct sk_buff`，是内核网络协议栈传递网络包时使用的核心数据结构。它不是网卡硬件上的原始 DMA buffer，也不是物理层概念，而是内核开始接管网络包之后的抽象。

收包路径可以简化理解为：

```text
物理层 / 网卡硬件
  -> 网卡驱动接收数据
  -> 驱动或内核路径构造 struct sk_buff
  -> Linux 内核网络栈开始处理 skb
  -> L2 / L3 / L4 / socket 相关逻辑继续处理
```

因此，如果问题是“eBPF skb 从哪一层开始”，可以回答为：从 Linux 内核构造 `struct sk_buff` 之后开始，一般可以认为是在二层帧进入内核网络栈处理阶段之后。

## XDP 和 skb 的区别

XDP 的位置比 `skb` 更早。XDP 程序通常运行在驱动收包路径上，面对的是 `xdp_buff` 或原始 packet buffer，而不是 `struct sk_buff`。这也是 XDP 性能高的原因之一：它可以在网络包进入完整内核网络栈之前就决定 pass、drop、redirect 或 tx。

```text
网卡驱动早期收包路径
  -> XDP hook，操作 xdp_buff，不是 skb
  -> XDP_PASS 后继续进入内核网络栈
  -> 构造或进入 skb 处理路径
  -> TC ingress 等 skb hook
```

所以，不能把所有 eBPF 网络程序都理解为操作 `skb`。XDP 程序不是 `skb` 上下文；TC、socket filter、cgroup skb 这类程序才是 `skb` 上下文。

## 常见 hook 的层次关系

不同 eBPF 挂载点看到的数据结构不同，位置也不同。

| eBPF 挂载点 | 是否是 skb 语义 | 大致位置 | 说明 |
|---|---|---|---|
| XDP | 否 | 驱动收包早期 | 操作 `xdp_buff` 或 packet buffer，通常还没有 `skb` |
| TC ingress | 是 | 包进入内核网络栈后，靠近 L2/L3 | 上下文是 `__sk_buff`，可以解析以太网头、IP 头、TCP/UDP 头 |
| TC egress | 是 | 包准备从网络设备发出前 | 适合做出方向过滤、重定向、标记等处理 |
| socket filter | 是 | socket 收包路径 | 更接近 socket 语义，仍基于 skb 抽象 |
| cgroup skb | 是 | cgroup 网络路径 | 常用于按容器、进程组维度做网络过滤 |
| kprobe 网络函数 | 取决于函数参数 | 任意内核网络函数 | 如果目标函数参数包含 `struct sk_buff *`，就可以读取 skb |

其中 TC ingress 是理解 `skb` 起点时最常用的参照点：它已经不是 XDP 的原始收包阶段，而是进入了内核 skb 网络栈。
