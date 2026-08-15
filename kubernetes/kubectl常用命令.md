# kubectl 常用命令

## 日志

### 查看 Pod 日志

```bash
kubectl logs <pod名称> -n <namespace>
kubectl logs <pod名称> -c <容器名> -n <namespace>
```

### 查看上一个容器退出时的日志

当容器发生重启时，可用 `--previous`（`-p`）参数查看上一次容器实例的日志：

```bash
kubectl logs <pod名称> --previous -n <namespace>
kubectl logs <pod名称> -c <容器名> --previous -n <namespace>
```

## 动态调整控制平面组件日志级别

在控制平面节点上向组件安全端口发送请求，可动态调整日志级别，不会修改启动参数，也无需重启组件。`kube-scheduler` 默认安全端口为 `10259`，`kube-controller-manager` 默认安全端口为 `10257`。Kubernetes 上游 `klog` 的默认 verbosity 是 `v=0`，不是 `v=2`；但很多 Kubernetes 发行版会在 `kube-scheduler` 的启动参数中显式配置 `--v=2`，实际默认级别应以组件启动参数为准。

```bash
# 将 kube-scheduler 日志级别设置为 v=4
curl -k -X PUT https://127.0.0.1:10259/debug/flags/v -d "4"

# 将 kube-controller-manager 日志级别设置为 v=4
curl -k -X PUT https://127.0.0.1:10257/debug/flags/v -d "4"
```

数值越高，日志越详细。修改后立即生效；排查完成后应恢复原日志级别，避免持续产生大量日志。
