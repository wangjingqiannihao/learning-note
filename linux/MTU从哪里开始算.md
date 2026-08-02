# Linux 内核里的 MTU 是从哪里开始算起的

## 结论

Linux 内核里的 MTU 是从 IP Header 开始算的，也就是从三层网络包开始算。它不包含 Ethernet Header、FCS、前导码、帧间隙等二层或物理层开销。对普通以太网而言，`mtu 1500` 表示最大 IP 报文是 1500 字节，而不是整条以太网帧在线路上的完整长度。

以最常见的以太网 MTU 1500 为例，Linux 中显示的 `mtu 1500` 表示该接口可以承载最大 1500 字节的 IP 报文。这个 1500 字节里包含 IP Header、TCP 或 UDP Header，以及真正的应用层数据。

```text
# 以太网帧在链路上的大致结构如下。
# Linux 接口 MTU 从 IP Header 开始算，不包含前面的 Ethernet Header。

[ Ethernet Header ][ IP Header ][ TCP/UDP Header ][ Payload ][ FCS ]
                    ↑
                    MTU 从这里开始算
```

因此可以把 MTU 理解为三层视角下的最大包长。对于普通 IPv4 TCP 流量，如果没有 IP option 和 TCP option，IPv4 Header 通常是 20 字节，TCP Header 通常是 20 字节，所以 TCP MSS 通常是 1500 - 20 - 20 = 1460 字节。对于 IPv6，IPv6 Header 通常是 40 字节，所以常见 TCP MSS 是 1500 - 40 - 20 = 1440 字节。

```text
# 常见以太网 MTU 为 1500。
# IPv4 无额外 option 时，IP 头 20 字节，TCP 头 20 字节。
IPv4 TCP MSS = 1500 - 20 - 20 = 1460

# IPv6 基础头部为 40 字节，TCP 头仍按 20 字节计算。
IPv6 TCP MSS = 1500 - 40 - 20 = 1440
```

需要注意的是，二层帧在真实链路上的长度会比 MTU 更大。以普通以太网为例，Ethernet Header 通常是 14 字节，FCS 通常是 4 字节，所以一个承载 1500 字节 IP 报文的以太网帧，在线路上的帧长度通常是 1518 字节。如果只看不含 FCS 的部分，则常见说法是 1514 字节。

```text
# 普通以太网，不考虑 VLAN tag。
# 这里的 1500 是 Linux 接口 MTU，也就是 IP 报文大小。
Ethernet Header 14B + IP Packet 1500B + FCS 4B = 1518B
```

如果链路上有 VLAN tag，二层会额外增加 4 字节开销。通常情况下，这不会改变 IP 层 MTU 的含义，但链路设备需要能够承载更大的二层帧。如果是 VXLAN、GRE、IPIP 等隧道封装，外层封装会额外占用字节，因此内层 MTU 往往需要调小，否则容易出现分片或报文过大的问题。
