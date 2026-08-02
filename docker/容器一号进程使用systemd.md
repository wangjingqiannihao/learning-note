# 为什么容器 1 号进程是 systemd 而不是业务进程

## 结论

容器的 1 号进程不一定必须是业务进程。让 `systemd` 作为 PID 1，是为了让它承担 init 进程职责，负责信号处理、子进程回收、服务启动顺序、异常重启和统一停止。业务进程则作为 `systemd` 管理的服务运行。

这种方式适合多进程、强依赖、传统服务迁移或需要完整 init 语义的容器。如果容器只运行一个简单业务进程，并且业务进程能正确处理退出信号和子进程回收，直接让业务进程作为 PID 1 更简单。

## 容器 PID 1 的特殊性

Linux 中的 PID 1 不是普通进程。它需要接收容器停止信号、回收孤儿进程和僵尸进程，并决定容器生命周期。

容器运行时会把入口命令作为容器内的 PID 1。这个进程退出后，容器通常也会退出。因此，PID 1 是否可靠，会直接影响容器能否优雅停止、正确清理进程，以及故障后是否能按预期恢复。

## 业务进程直接作为 PID 1 的问题

很多业务进程是按普通进程设计的，没有专门适配 PID 1 语义，常见问题包括：

1. 信号处理不完整。容器停止时，运行时通常向 PID 1 发送 `SIGTERM`。如果业务进程没有正确处理，可能无法优雅退出。
2. 子进程回收不可靠。业务进程如果启动脚本、插件或子任务，但没有调用 `wait()` 回收，容易留下僵尸进程。
3. 多进程缺少统一管理。主服务、日志进程、定时任务、辅助进程如果都在一个容器内，业务进程通常无法可靠管理它们的启动顺序、重启策略和停止流程。
4. 传统服务迁移成本高。部分软件依赖 `systemd` unit、`systemctl`、`journald` 等机制，没有 init 系统时需要额外改造。

## systemd 作为 PID 1 的作用

`systemd` 作为容器内 PID 1 时，容器内部服务通过 unit 文件管理。容器运行时只管理 `systemd`，业务服务由 `systemd` 启动、停止和重启。

```text
# 容器内进程关系示意
PID 1: systemd
  ├─ business.service    # 业务主进程
  ├─ log-agent.service   # 可选：日志或辅助进程
  └─ timer.service       # 可选：定时任务
```

简化的 service 示例：

```ini
# /etc/systemd/system/business.service
[Unit]
# 服务描述，方便查看状态
Description=Business Service

# 表示业务服务在网络就绪后启动
After=network-online.target
Wants=network-online.target

[Service]
# 前台运行业务进程
Type=simple

# 业务启动命令
ExecStart=/usr/local/bin/business-server --config /etc/business/config.yaml

# 停止服务时先发送 SIGTERM
KillSignal=SIGTERM

# 业务异常退出时自动重启
Restart=on-failure

# 重启前等待 5 秒，避免频繁重启
RestartSec=5s

[Install]
# 进入 multi-user.target 时启动该服务
WantedBy=multi-user.target
```

## 这样做的好处

### 统一管理进程生命周期

`systemd` 可以统一管理容器内服务的启动、停止、重启和状态。业务进程不需要自己实现 supervisor 逻辑，也不需要用复杂 shell 脚本管理多个进程。

### 正确承担 PID 1 职责

`systemd` 天然适合作为 init 进程，能处理停止信号、接管孤儿进程并回收僵尸进程，避免普通业务进程不适配 PID 1 带来的隐藏问题。

### 支持服务依赖编排

如果容器内有多个服务，`systemd` 可以通过 `After=`、`Before=`、`Requires=`、`Wants=` 描述启动顺序和依赖关系，比在入口脚本中手动串联更清晰。

### 复用传统运维方式

容器内使用 `systemd` 后，可以复用已有 unit 文件和排障命令：

```bash
# 查看业务服务状态
systemctl status business.service

# 查看业务服务日志
journalctl -u business.service

# 重启业务服务
systemctl restart business.service
```

这适合把虚拟机或物理机上的传统服务迁移到容器环境。

### 退出和故障恢复更可控

业务进程异常退出后，`systemd` 可以按策略自动重启；关键服务持续失败时，也可以让容器最终退出，再由外部编排系统重建。相比简单的 `while true` 脚本，这种方式更明确，也更容易排障。

## 代价和适用边界

使用 `systemd` 会让容器更接近完整 Linux 系统，镜像更大，启动链路更复杂，也可能需要额外挂载 cgroup 或配置更高权限。容器内运行多个服务后，排障边界也会比单进程容器更复杂。

适合使用 `systemd` 的场景包括：容器内有多个长期进程，服务之间有启动依赖，需要复用已有 systemd unit，业务进程会派生子进程但自身没有可靠回收逻辑，或容器定位更接近轻量系统环境。

不适合默认使用 `systemd` 的场景包括：容器只运行一个简单业务进程，业务进程已经能正确处理 `SIGTERM` 和优雅退出，外部编排系统已经负责重启和健康检查，并且希望镜像和运行权限尽量保持最小。
