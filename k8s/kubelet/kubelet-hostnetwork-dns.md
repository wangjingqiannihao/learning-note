# kubelet 如何为主机网络 Pod 配置 DNS

## 核心结论

`hostNetwork: true` 只表示 Pod 使用宿主机网络命名空间，不代表 DNS 一定完全继承宿主机。Pod 的 `/etc/resolv.conf` 仍由 kubelet 根据 `dnsPolicy`、`dnsConfig` 以及 kubelet 自身的 DNS 参数生成。

如果主机网络 Pod 仍需要解析集群内 Service，推荐显式配置：

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

如果不这样配置，主机网络 Pod 很容易表现为使用节点 DNS，导致无法按预期解析集群服务名。

## kubelet 参与 DNS 配置的几个输入

kubelet 生成 Pod DNS 配置时，主要看这些信息：

| 配置项 | 作用 |
| --- | --- |
| `dnsPolicy` | 决定 Pod 使用哪类 DNS 策略 |
| `dnsConfig` | 在 `dnsPolicy: None` 或需要追加配置时，自定义 nameserver、search、options |
| `--cluster-dns` | kubelet 使用的集群 DNS Service IP，通常是 CoreDNS 的 ClusterIP |
| `--cluster-domain` | 集群 DNS 域，例如常见的 `cluster.local` |
| `--resolv-conf` | kubelet 读取节点侧 resolver 配置的来源文件 |

最终结果会体现在 Pod 内的 `/etc/resolv.conf`。

## kubelet 会不会把节点的 /etc/resolv.conf 直接挂进容器

不会。**kubelet 不是把节点上的 `/etc/resolv.conf` 以 bind mount 的方式直接挂进容器**，而是先读取节点侧 resolver 配置来源，再根据 Pod 的 `dnsPolicy`、`dnsConfig` 和 kubelet 参数生成 Pod 自己的 `/etc/resolv.conf`。

可以直接把它理解成：**kubelet 是“生成 resolv.conf”，不是“挂载 resolv.conf”**。

在主机网络 Pod 场景下，这个结论也成立。即使 `hostNetwork: true`，Pod 进入了宿主机网络命名空间，`/etc/resolv.conf` 依然是 kubelet 按策略写出来的，不是简单复用节点文件。

对应关系可以简化理解为：

| `dnsPolicy` | kubelet 最终使用的 DNS 来源 |
| --- | --- |
| `ClusterFirst` | 优先写入集群 DNS，一般指向 CoreDNS / kube-dns |
| `ClusterFirstWithHostNet` | 对主机网络 Pod 仍优先写入集群 DNS |
| `Default` | 参考节点 `resolv-conf` 来源文件，把节点 DNS 配置带入 Pod |
| `None` | 主要按 `dnsConfig` 手动生成 |

这也解释了一个常见误区：**主机网络 Pod 并不等于默认走节点 DNS**。如果策略是 `ClusterFirst` 或 `ClusterFirstWithHostNet`，kubelet 仍会按集群 DNS 逻辑生成配置；只有 `Default` 才更接近直接继承节点 DNS。

## 常见 dnsPolicy 行为

### `ClusterFirst`

这是 Pod 默认策略。普通 Pod 会优先使用集群 DNS，因此可以解析 `*.svc` 这类集群服务域名。

但对 `hostNetwork: true` 的 Pod，不建议依赖默认 `ClusterFirst`。如果它需要集群 DNS，应改用 `ClusterFirstWithHostNet`。

### `ClusterFirstWithHostNet`

这是主机网络 Pod 使用集群 DNS 的专用策略。它的含义是：Pod 虽然使用宿主机网络命名空间，但 DNS 仍按集群服务发现方式配置。

适合 DaemonSet、监控 Agent、日志采集、网络组件等既需要宿主机网络、又需要访问集群 Service 的场景。

### `Default`

Pod 继承节点 DNS 配置，行为更接近宿主机进程。适合只需要外部域名解析、不依赖集群 Service 发现的场景。

### `None`

完全由 `dnsConfig` 指定 DNS 配置。适合强定制场景，例如指定固定 nameserver、搜索域或 resolver options。

## 推荐配置示例

主机网络 Pod 需要访问集群内 Service 时：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-dns-demo
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

## 排查思路

排查主机网络 Pod DNS 问题时，优先看四点：

1. Pod 是否设置了 `hostNetwork: true`。
2. 是否显式设置了 `dnsPolicy: ClusterFirstWithHostNet`。
3. kubelet 是否正确配置了 `--cluster-dns` 和 `--cluster-domain`。
4. Pod 内 `/etc/resolv.conf` 是否符合预期。

可以直接进入 Pod 查看：

```bash
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

如果 nameserver 是集群 DNS IP，通常说明走的是集群 DNS；如果 nameserver 与节点一致，则更可能是继承了节点 DNS。

