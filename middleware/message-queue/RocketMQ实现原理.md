# RocketMQ 实现原理

## 结论

RocketMQ 的核心设计可以概括为：**NameServer 管路由，Broker 管数据，CommitLog 保存消息正文，ConsumeQueue 和 IndexFile 提供不同用途的索引。** 消息写入时顺序追加到 CommitLog；消费时先查 ConsumeQueue，再按物理偏移量读取 CommitLog。

| 关键点 | 结论 |
| --- | --- |
| 路由 | NameServer 保存 Topic 路由和 Broker 地址，不保存消息正文 |
| 存储 | Broker 是数据存储核心；CommitLog 是完整消息正文的唯一持久化文件 |
| 消费 | Consumer 先读取 ConsumeQueue，再根据物理偏移量从 CommitLog 取出正文 |
| 可靠性 | 默认语义是 At Least Once，可能重复投递，消费端必须实现幂等 |
| 顺序 | 同一业务键路由到同一个 MessageQueue，可保证局部顺序，但不保证全局顺序 |

## 整体架构

![RocketMQ 整体架构与消息存储、消费链路](images/rocketmq-architecture.png)

RocketMQ 由 NameServer、Broker、Producer 和 Consumer 四类核心角色组成。

- **NameServer**：维护 Topic 与 Broker 之间的路由关系，为客户端提供服务发现，不保存消息正文。
- **Broker**：接收消息、持久化数据并向消费者投递消息，是整个系统的数据存储核心。
- **Producer**：从 NameServer 获取 Topic 路由，将消息发送到对应 Broker。
- **Consumer**：从 Broker 拉取消息，执行业务消费逻辑并提交消费进度。

NameServer 不参与消息正文的存储和转发，因此 NameServer 故障不会直接导致已经在 Broker 落盘的消息丢失。

## Broker 存储目录

Broker 的消息数据默认存放在 `storePathRootDir` 指定的目录中，常见默认位置是 `${user.home}/store`。

```text
store/
├── commitlog/                 # 完整消息正文
├── consumequeue/              # 按 Topic + QueueID 组织的消费索引
├── index/                     # 按 Message Key、消息 ID 查询的哈希索引
├── checkpoint                 # 刷盘时间点等检查点信息
├── abort                      # 判断 Broker 上一次是否异常退出
└── config/
    ├── consumerOffset.json    # 消费组的消费进度
    ├── topics.json            # Topic 配置
    └── delayOffset.json       # 延迟消息相关消费进度
```

## CommitLog：消息正文的唯一存储

CommitLog 采用所有 Topic 混写、顺序追加的方式保存消息。Broker 收到消息后，不会为每个 Topic 单独维护正文文件，而是把消息依次追加到 CommitLog。顺序写比随机写更适合磁盘吞吐，也降低了大量小文件带来的管理成本。

每条消息写入后都会获得一个物理偏移量。后续建立 ConsumeQueue 和 IndexFile 时，都会通过这个偏移量关联回 CommitLog。ConsumeQueue 和 IndexFile 都不是消息正文的副本，真正的完整消息只存在于 CommitLog。

## ConsumeQueue：正常消费的主索引

ConsumeQueue 按 `Topic + QueueID` 组织。每个索引条目主要保存以下内容：

- CommitLog 物理偏移量
- 消息长度
- Tag 哈希值

正常消费链路如下：

1. Consumer 根据消费进度读取 ConsumeQueue。
2. ConsumeQueue 返回 CommitLog 物理偏移量和消息长度。
3. Broker 根据物理偏移量从 CommitLog 读取完整消息正文。
4. Broker 将消息返回给 Consumer。
5. Consumer 完成业务处理后提交消费进度。

因此，ConsumeQueue 决定“应该读哪条消息”，CommitLog 负责“取出这条消息的完整内容”。

## IndexFile：面向检索和排查的辅助索引

IndexFile 使用哈希索引记录 Message Key、消息 ID 与 CommitLog 物理偏移量之间的关系，适合按业务 Key 或消息 ID 查询消息。

IndexFile 主要用于问题排查、消息追踪和管理查询，不是正常消费链路的必经路径。ConsumeQueue 面向按队列顺序消费，IndexFile 面向按 Key 检索。即使消息没有设置业务 Key，也不影响 Consumer 通过 ConsumeQueue 正常消费。

## 消息从生产到消费的完整链路

### 发送阶段

Producer 从 NameServer 获取 Topic 路由，根据队列选择策略确定 MessageQueue，并将消息发送到对应 Broker。

### 存储阶段

Broker 将完整消息顺序追加到 CommitLog，随后生成对应的 ConsumeQueue 索引。如果消息携带可索引的 Key，还会建立 IndexFile 索引。

### 消费阶段

Consumer 从 Broker 拉取指定逻辑位点的消息。Broker 查询 ConsumeQueue 得到 CommitLog 物理偏移量，再读取正文并返回。Consumer 业务处理成功后提交消费进度；处理失败时，消息会按重试机制重新投递。

## 投递语义与消费幂等

RocketMQ 默认提供 **At Least Once** 语义，即消息至少会被投递一次，但不能保证绝对不重复。

网络超时、消费完成后进度提交失败、Broker 重投等情况，都可能让同一条消息再次到达消费端。因此，可靠消费不能只依赖消息中间件，业务处理必须具备幂等性。

| 方案 | 实现方式 | 适用场景 |
| --- | --- | --- |
| 数据库唯一索引 + 本地事务 | 以业务唯一键建立唯一约束，并把防重记录与业务更新放在同一事务中 | 对一致性要求高的核心业务 |
| 业务状态机条件更新 | 更新时附带前置状态条件，例如仅允许“待支付”变为“已支付” | 订单、工单等状态驱动业务 |
| Redis `SET NX EX` | 以业务唯一键抢占处理标记，并设置过期时间 | 允许一定误差、追求低延迟的场景；不能替代强一致事务 |

数据库唯一索引配合本地事务通常是更可靠的防重方式。Redis 方案需要考虑缓存丢失、过期时间设置不合理以及业务处理与标记写入不原子等问题。

## 重试与死信队列

消费失败后，RocketMQ 会把消息送入消费组对应的重试 Topic：

```text
%RETRY%consumer-group-name
```

系统按照重试策略再次投递。达到最大重试次数后，消息进入死信队列：

```text
%DLQ%consumer-group-name
```

死信消息不会自动恢复正常消费。生产环境通常需要配套监控、告警、原因分析和人工或自动补偿机制。重新投递死信消息前仍要保证业务幂等，否则补偿操作可能再次产生副作用。

## 事务消息

事务消息用于协调本地事务与消息发送，典型流程如下：

1. Producer 向 Broker 发送半消息。
2. Broker 保存半消息，但暂不向 Consumer 投递。
3. Producer 执行本地事务。
4. 本地事务成功后向 Broker 提交 Commit，失败则提交 Rollback。
5. Commit 后消息可被 Consumer 消费；Rollback 后消息被丢弃。
6. 如果 Broker 长时间未收到明确结果，会向 Producer 发起事务状态回查。

Producer 需要根据本地事务记录返回最终状态。事务消息解决的是消息发送与本地事务之间的一致性问题，但本地事务回查逻辑仍需可靠，消费端仍需实现幂等。

## 顺序消息

RocketMQ 的顺序保证建立在 MessageQueue 维度。Producer 需要把同一业务键的消息稳定路由到同一个 MessageQueue，例如按订单 ID 选择队列；Consumer 再以顺序方式消费该队列，才能保证同一业务键内的消息顺序。

这种方式提供的是局部顺序，不是全局顺序。全局顺序通常意味着所有消息进入单一队列，会明显限制并发能力。实际设计时应先确定需要保持顺序的最小业务粒度，再以该业务键进行队列路由。

## 常见理解误区

**NameServer 不保存消息。** NameServer 只维护路由与 Broker 信息，消息正文始终由 Broker 持久化。

**ConsumeQueue 不保存完整消息。** 它只保存定位 CommitLog 所需的轻量索引。

**IndexFile 不是消费主路径。** 它主要用于按 Key 或消息 ID 查询和排查。

**消费成功不等于永不重复。** 只要消费结果与进度提交之间存在故障窗口，就需要业务幂等兜底。

**顺序消息不等于全局有序。** 默认只能保证同一 MessageQueue 内的顺序。
