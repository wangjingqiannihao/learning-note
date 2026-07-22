# containerd 配置参数说明

基于 containerd v2.3.3 整理。v2.3.3 是 containerd 2.3 的第三个补丁版本；containerd v2.0 引入配置版本 3，containerd 1.x 使用的配置版本 2 在 v2.x 中仍受支持并会自动转换为版本 3。本文以 containerd 2.x 推荐的 `version = 3` 为主，参考 containerd 官方发布页、CRI 配置文档和配置文件手册。

## 配置文件位置

containerd 默认读取 `/etc/containerd/config.toml`。如果配置文件不存在，或启动时未通过 `--config` 指定配置文件，containerd 会使用默认配置。当前运行中的有效配置没有专门的“dump live config”命令，通常按启动参数找到正在使用的配置文件后直接打印；如果未指定 `--config`，则查看默认路径。

```bash
# 查看 systemd 中 containerd 的启动命令，确认是否通过 --config 或 -c 指定了配置文件
systemctl cat containerd

# 打印当前服务通常使用的配置文件
cat /etc/containerd/config.toml

# 如果 systemd 启动参数指定了其他配置文件，则打印该文件
cat /path/to/containerd-config.toml
```

## 完整配置示例

```toml
# containerd 2.x 推荐使用 version = 3。
# version = 3 在 containerd v2.0 引入；version = 2 仍兼容，但插件 ID 写法不同。
version = 3

# containerd 保存持久化元数据的目录，例如镜像元数据、容器元数据、快照元数据。
# 生产环境不要放在临时目录；迁移该目录前需要停止 containerd 并完整迁移数据。
root = "/var/lib/containerd"

# containerd 保存运行态数据的目录，例如 socket、运行中任务状态、临时运行信息。
# 通常放在 /run 下，系统重启后可清理。
state = "/run/containerd"

# 临时文件目录。空字符串表示使用系统默认临时目录。
temp = ""

# 动态插件目录。一般保持默认空值，除非明确使用外部插件。
plugin_dir = ""

# 禁用指定插件。Kubernetes 节点不要禁用 CRI 相关插件，否则 kubelet 无法通过 CRI 使用 containerd。
disabled_plugins = []

# 声明必须启动成功的插件；任一 required 插件不存在、初始化失败或启动失败时，containerd 会退出。
required_plugins = []

# 调整 containerd 进程的 OOM score。值越低越不容易被 OOM Killer 杀死。
# 生产节点通常保持默认，避免影响系统整体 OOM 策略。
oom_score = 0

# 引入其他配置文件。适合把运行时、镜像仓库等配置拆分为独立文件。
# 被导入配置的版本不能高于主配置；简单字段会覆盖，数组和 map 字段会追加或合并。
imports = []

[grpc]
  # containerd gRPC API 监听地址。Kubernetes 节点上 kubelet 通常通过这个 Unix socket 访问 containerd。
  address = "/run/containerd/containerd.sock"

  # TCP 监听地址。默认关闭；仅在明确需要远程访问并做好访问控制时配置。
  tcp_address = ""

  # TCP TLS CA 证书路径。仅 tcp_address 启用并需要 TLS 时配置。
  tcp_tls_ca = ""

  # TCP TLS 服务端证书路径。
  tcp_tls_cert = ""

  # TCP TLS 服务端私钥路径。
  tcp_tls_key = ""

  # Unix socket 用户 ID。0 表示 root。
  uid = 0

  # Unix socket 用户组 ID。0 表示 root。
  gid = 0

  # 每次 gRPC 发送消息的最大 bytes 数。保持默认通常即可。
  max_send_message_size = 16777216

  # 每次 gRPC 接收消息的最大 bytes 数。镜像、容器元数据较大时可按需调整。
  max_recv_message_size = 16777216

[ttrpc]
  # ttrpc socket 地址，主要供 shim 等内部组件通信使用。
  address = ""

  # ttrpc socket 用户 ID。
  uid = 0

  # ttrpc socket 用户组 ID。
  gid = 0

[debug]
  # debug socket 地址。默认关闭；排障时可临时启用。
  address = ""

  # debug socket 用户 ID。
  uid = 0

  # debug socket 用户组 ID。
  gid = 0

  # 日志级别。常用值包括 trace、debug、info、warn、error。
  # 生产环境建议使用 info 或 warn，排障时再临时调高。
  level = "info"

  # 日志输出格式。空值表示默认格式；也可按版本支持情况使用 json 等格式。
  format = ""

[metrics]
  # Prometheus metrics 监听地址。默认关闭；需要监控时配置为类似 127.0.0.1:1338。
  address = ""

  # 是否暴露 gRPC histogram 指标。开启后指标更细，但会增加指标基数和采集成本。
  grpc_histogram = false

[cgroup]
  # containerd daemon 自身所属的 cgroup 路径。
  # 这是 containerd 进程级配置，不等同于容器运行时的 SystemdCgroup。
  path = ""

[timeouts]
  # shim 清理超时时间。
  "io.containerd.timeout.shim.cleanup" = "5s"

  # shim 加载超时时间。
  "io.containerd.timeout.shim.load" = "5s"

  # shim 关闭超时时间。
  "io.containerd.timeout.shim.shutdown" = "3s"

  # 查询 task 状态的超时时间。
  "io.containerd.timeout.task.state" = "2s"

[plugins]
  [plugins."io.containerd.cri.v1.images"]
    # CRI 使用的快照器。默认 overlayfs，类似 Docker 的 overlay2。
    # 常见可选项包括 overlayfs、native、btrfs、zfs、devmapper，具体取决于内核、文件系统和插件支持。
    snapshotter = "overlayfs"

    # 是否禁用 snapshot annotations。保持默认可减少无关注解传播。
    disable_snapshot_annotations = true

    # 镜像解包后是否丢弃未解包层。开启可节省空间，但可能影响后续复用。
    discard_unpacked_layers = false

    # 每个镜像拉取时的最大并发下载数。网络和 registry 能力较强时可适当调大。
    max_concurrent_downloads = 3

    # 镜像拉取进度超时时间。超过该时间无进度可能被判定为失败。
    image_pull_progress_timeout = "5m0s"

    # 镜像拉取后是否执行同步落盘。开启可提高异常断电后的稳妥性，但会影响拉取性能。
    image_pull_with_sync_fs = false

    # 镜像统计信息采集周期，单位为秒。
    stats_collect_period = 10

    # 是否使用本地镜像拉取模式。false 表示优先使用 Transfer Service。
    use_local_image_pull = false

    [plugins."io.containerd.cri.v1.images".pinned_images]
      # Pod sandbox 使用的 pause 镜像。Kubernetes 集群中应与 kubeadm 或发行版推荐版本保持一致。
      sandbox = "registry.k8s.io/pause:3.10.2"

    [plugins."io.containerd.cri.v1.images".registry]
      # registry 主机配置目录。containerd 2.x 推荐通过 certs.d/<registry>/hosts.toml 管理镜像源、TLS 和认证入口。
      # 空字符串表示不显式指定；containerd 会按版本默认逻辑处理。
      config_path = "/etc/containerd/certs.d"

    [plugins."io.containerd.cri.v1.images".image_decryption]
      # 镜像解密 key 模型。仅使用加密镜像时需要关注。
      key_model = "node"

  [plugins."io.containerd.cri.v1.runtime"]
    # 是否启用 SELinux 支持。仅 SELinux 环境需要开启，并要求系统策略配套。
    enable_selinux = false

    # SELinux category 范围上限。
    selinux_category_range = 1024

    # 单行容器日志最大 bytes 数。超过限制会被拆分；-1 表示不限制。
    max_container_log_line_size = 16384

    # 是否禁用 cgroup 支持。Kubernetes 节点通常必须保持 false。
    disable_cgroup = false

    # 是否禁用 AppArmor 支持。没有 AppArmor 权限或不使用 AppArmor 时才考虑开启。
    disable_apparmor = false

    # 是否限制容器 OOMScoreAdj 不能低于 containerd 当前 OOMScoreAdj。
    restrict_oom_score_adj = false

    # 是否禁用 Kubernetes ProcMount 支持。仅兼容很老的 Kubernetes 版本时才需要关注。
    disable_proc_mount = false

    # 当 CRI 请求未设置 seccomp profile 时使用的 profile。空值通常映射为默认行为。
    unset_seccomp_profile = ""

    # 缺少 hugetlb controller 时是否容忍。默认 true 更兼容不同内核环境。
    tolerate_missing_hugetlb_controller = true

    # 是否禁用 hugetlb controller。
    disable_hugetlb_controller = true

    # 是否根据 Pod securityContext 设置设备文件属主。
    device_ownership_from_security_context = false

    # 是否忽略镜像中声明的 VOLUME。开启后可减少隐式挂载带来的隔离问题。
    ignore_image_defined_volumes = false

    # 是否把网络命名空间挂载放到 state 目录下。修改该项前需要清理现有容器。
    netns_mounts_under_state_dir = false

    # 是否允许非特权容器绑定低端口。containerd 2.x 默认 true。
    enable_unprivileged_ports = true

    # 是否允许非特权容器使用 ICMP。
    enable_unprivileged_icmp = true

    # 是否启用 CDI，用于把设备以规范化方式注入容器。
    enable_cdi = true

    # CDI spec 文件目录。
    cdi_spec_dirs = ["/etc/cdi", "/var/run/cdi"]

    # exec sync I/O drain 超时时间。0s 表示不额外等待。
    drain_exec_sync_io_timeout = "0s"

    # 忽略指定弃用告警。建议仅在明确接受迁移风险时使用。
    ignore_deprecation_warnings = []

    # runtime 统计信息采集周期。
    stats_collect_period = "1s"

    # runtime 统计信息保留时间。
    stats_retention_period = "2m"

    # 是否启用 CRIU 相关能力。仅需要容器 checkpoint/restore 时关注。
    enable_criu = true

    [plugins."io.containerd.cri.v1.runtime".containerd]
      # 默认 RuntimeClass 名称。未显式指定 RuntimeClass 的 Pod 会使用该 runtime。
      default_runtime_name = "runc"

      # blockio 未启用时报错是否忽略。
      ignore_blockio_not_enabled_errors = false

      # RDT 未启用时报错是否忽略。
      ignore_rdt_not_enabled_errors = false

      [plugins."io.containerd.cri.v1.runtime".containerd.runtimes]
        [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc]
          # runtime 类型。runc v2 shim 使用 io.containerd.runc.v2。
          runtime_type = "io.containerd.runc.v2"

          # runtime 二进制路径。空值表示使用默认查找逻辑。
          runtime_path = ""

          # 允许透传到 Pod sandbox 的 annotations。
          pod_annotations = []

          # 允许透传到普通容器的 annotations。
          container_annotations = []

          # 特权容器是否不自动挂载宿主机设备。
          privileged_without_host_devices = false

          # 特权容器不自动挂载设备时，是否仍允许访问全部设备。
          privileged_without_host_devices_all_devices_allowed = false

          # 是否允许容器内写 cgroup 文件系统。
          cgroup_writable = false

          # OCI runtime spec 基础模板路径。需要统一注入默认 OCI 配置时使用。
          base_runtime_spec = ""

          # 针对该 runtime 覆盖 CNI 配置目录。空值表示使用全局 CNI 配置。
          cni_conf_dir = ""

          # 针对该 runtime 覆盖最多加载的 CNI 配置数量。0 表示不覆盖。
          cni_max_conf_num = 0

          # 针对该 runtime 覆盖 snapshotter。空值表示使用 CRI images 中的 snapshotter。
          snapshotter = ""

          # sandboxer 名称。默认 podsandbox。
          sandboxer = "podsandbox"

          # 是否禁止拉取 pause 镜像。一般保持 false。
          disable_pause_image_pull = false

          # shim I/O 类型。通常保持默认空值。
          io_type = ""

          [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc.options]
            # runc 二进制名称或路径。空值表示使用默认 runc。
            BinaryName = ""

            # CRIU 镜像路径。仅 checkpoint/restore 场景使用。
            CriuImagePath = ""

            # CRIU 工作路径。仅 checkpoint/restore 场景使用。
            CriuWorkPath = ""

            # shim I/O 组 ID。
            IoGid = 0

            # shim I/O 用户 ID。
            IoUid = 0

            # 是否不为容器创建新的 session keyring。
            NoNewKeyring = false

            # runc root 目录。空值表示默认路径。
            Root = ""

            # shim 所在 cgroup。需要把 shim 放入指定 cgroup 时配置。
            ShimCgroup = ""

            # systemd 系统建议设为 true，使容器 cgroup 由 systemd 管理。
            # Kubernetes 节点上 kubelet 的 cgroupDriver 也应与这里保持一致。
            SystemdCgroup = true

    [plugins."io.containerd.cri.v1.runtime".cni]
      # 已弃用，containerd v2.1 起建议使用 bin_dirs。
      bin_dir = ""

      # CNI 插件二进制目录。
      bin_dirs = ["/opt/cni/bin"]

      # CNI 配置目录。
      conf_dir = "/etc/cni/net.d"

      # 最多加载的 CNI 配置数量。
      max_conf_num = 1

      # 是否串行执行 CNI setup。遇到插件或网络环境并发不安全时再开启。
      setup_serially = false

      # CNI 配置模板路径。需要由 CRI 动态生成 CNI 配置时使用。
      conf_template = ""

      # IP 偏好。通常保持默认空值。
      ip_pref = ""

      # 是否使用内部 loopback 配置。
      use_internal_loopback = false

  [plugins."io.containerd.grpc.v1.cri"]
    # 是否禁用 CRI TCP 服务。建议保持 true，避免暴露不必要端口。
    disable_tcp_service = true

    # exec、attach、port-forward 等流式服务监听地址。
    stream_server_address = "127.0.0.1"

    # 流式服务监听端口。0 表示自动选择端口。
    stream_server_port = "0"

    # 流式连接空闲超时时间。
    stream_idle_timeout = "4h0m0s"

    # 是否为流式服务启用 TLS。
    enable_tls_streaming = false

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      # 流式服务 TLS 证书路径。
      tls_cert_file = ""

      # 流式服务 TLS 私钥路径。
      tls_key_file = ""

  [plugins."io.containerd.transfer.v1.local"]
    # Transfer Service 本地传输插件的并发下载数。containerd 2.x 默认镜像拉取路径会使用该服务。
    max_concurrent_downloads = 3

    # 解包配置。一般保持默认；需要按平台或 snapshotter 调整解包行为时再配置。
    unpack_config = {}
```

## 镜像仓库 hosts.toml 示例

containerd 2.x 推荐把镜像仓库配置放到 `certs.d/<registry>/hosts.toml`。这种方式比把所有 mirror、TLS、认证配置堆在主 `config.toml` 中更清晰，也更方便按 registry 拆分维护。

```toml
# 文件路径示例：/etc/containerd/certs.d/docker.io/hosts.toml

# 默认 server。通常写原始 registry 地址。
server = "https://registry-1.docker.io"

[host."https://mirror.example.com"]
  # mirror 能力。pull 表示可拉取镜像；resolve 表示可解析 tag 到 digest；push 表示可推送。
  capabilities = ["pull", "resolve"]

  # 是否跳过 TLS 证书校验。仅临时测试或受控内网自签场景使用，生产环境不建议开启。
  skip_verify = false

  # CA 证书路径。私有 CA 或自签证书场景使用。
  ca = "/etc/containerd/certs.d/docker.io/ca.crt"

  # 客户端证书和私钥路径。需要双向 TLS 时使用。
  client = [["/etc/containerd/certs.d/docker.io/client.crt", "/etc/containerd/certs.d/docker.io/client.key"]]

  [host."https://mirror.example.com".header]
    # 自定义请求头。一般不需要配置；如需认证，优先使用平台支持的标准方式。
    x-custom-header = ["value"]
```

## Kubernetes 节点常用最小配置

```toml
# containerd 2.x Kubernetes 节点常用最小配置。
version = 3

[plugins."io.containerd.cri.v1.images"]
  # 默认使用 overlayfs；大多数 Linux Kubernetes 节点适用。
  snapshotter = "overlayfs"

  [plugins."io.containerd.cri.v1.images".pinned_images]
    # pause 镜像版本应和 Kubernetes 发行版或 kubeadm 推荐值保持一致。
    sandbox = "registry.k8s.io/pause:3.10.2"

  [plugins."io.containerd.cri.v1.images".registry]
    # 使用 certs.d 目录维护每个 registry 的 hosts.toml。
    config_path = "/etc/containerd/certs.d"

[plugins."io.containerd.cri.v1.runtime".containerd]
  # 默认 runtime。
  default_runtime_name = "runc"

[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc]
  # runc v2 shim。
  runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc.options]
  # systemd 系统建议开启，并确保 kubelet cgroupDriver 也为 systemd。
  SystemdCgroup = true
```

## 常用检查命令

```bash
# 查看 containerd 版本
containerd --version

# 校验默认配置生成是否正常
containerd config default > /tmp/containerd-default.toml

# 查看 CRI 插件是否正常工作
crictl info

# 查看 containerd 插件状态
ctr plugins ls

# 查看 containerd 服务状态
systemctl status containerd

# 修改配置后重启服务
systemctl restart containerd
```

## 参考资料

- [containerd v2.3.3 Release](https://github.com/containerd/containerd/releases/tag/v2.3.3)
- [containerd CRI Plugin Config Guide](https://containerd.io/docs/2.3/cri/config/)
- [containerd config.toml manual](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md)
- [containerd Versioning and Release](https://containerd.io/releases/)
