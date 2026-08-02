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
