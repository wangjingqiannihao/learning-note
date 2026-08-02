# kubelet 推荐使用参数和版本

## 版本说明

本文基于 Kubernetes 源码 tag `v1.36.2` 整理。

## 配置方式

kubelet 参数推荐通过配置文件管理，而不是把大量参数堆在 systemd 启动命令里。启动时使用 `--config` 指向配置文件；如果命令行参数与配置文件管理同一项配置，命令行参数会覆盖配置文件。

推荐配置文件如下：

```yaml
# kubelet 配置文件使用官方 kubelet config API。
# 建议通过 --config 指向该文件，把 kubelet 主要行为收敛到配置文件中，便于审计、变更和回滚。
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# 推荐与容器运行时使用一致的 cgroup driver。kubelet 与容器运行时的 cgroup driver 必须保持一致。
cgroupDriver: systemd

# 禁止匿名访问 kubelet，减少 kubelet API 暴露带来的攻击面。
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true

# 使用 Webhook 授权，让 kubelet API 请求交给 apiserver 授权链处理。
authorization:
  mode: Webhook

# 开启 kubelet 客户端证书轮转，降低长期证书泄露风险，减少人工维护成本。
rotateCertificates: true

# 让 kubelet 服务端证书通过 CSR 流程签发，避免长期使用自签证书。
serverTLSBootstrap: true

# 镜像拉取可以并发执行，适合镜像仓库、节点网络和磁盘 IO 能力足够的场景。
serializeImagePulls: false

# 为系统进程和 Kubernetes 组件预留资源，避免业务 Pod 把节点资源打满后影响 kubelet、容器运行时和系统进程。
systemReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"

# 硬驱逐阈值不能直接省略；不配置时 kubelet 会使用默认阈值。
# 为避免节点压力触发 kubelet 直接驱逐业务 Pod，这里把阈值配置到接近 0 的水位。
# 如果需要做节点保护，建议优先通过监控告警、资源预留、容量水位和调度约束来治理。
evictionHard:
  memory.available: "1Mi"
  nodefs.available: "1%"
  imagefs.available: "1%"
  nodefs.inodesFree: "1%"
  imagefs.inodesFree: "1%"
```
