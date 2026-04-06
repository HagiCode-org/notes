# Current State: Hagipower

## One-Screen Summary

- 当前 Hagipower 有两种余额：
  - `PublicBalance`
  - `HeroBalances[heroId]`
- 执行完成时，`tokenCount` 会等量转成 Hagipower。
- 如果执行没有绑定 Hero，Power 进入 `PublicBalance`。
- 如果执行绑定了 Hero，Power 进入该 Hero 的专属余额。
- 当前小游戏链路只消费 `hero-bound balance`，不会消费 `PublicBalance`。
- 当前小游戏奖励会把消耗掉的 `Power(M)` 等量转成 `Stored XP(M)`。

## Current Balance Model

### Acquisition

当前 Hagipower 的来源只有一条：

- `AddFromExecutionAsync(executionMessageId, heroId, tokenCount, completedAtUtc)`

行为如下：

- `executionMessageId` 作为去重键。
- `tokenCount <= 0` 时不增加余额。
- `heroId` 为空时，余额进入 `PublicBalance`。
- `heroId` 非空时，余额进入对应 Hero 的 `HeroBalances`。

也就是说，当前设计里：

- token 明确驱动了 Hagipower 系统，同时也是当前经验公式与历史 backfill 的共同基础数值。
- Hagipower 不是额外结算出来的复杂派生值，而是非常直接地复用 token 数。

### Storage shape

当前状态快照结构包含：

- `PublicBalance`
- `HeroBalances`
- `UpdatedAtUtc`

其中 `HeroBalances` 会在输出时：

- 过滤掉空 HeroId
- 过滤掉 `<= 0` 的余额
- 按余额倒序排序

## Current Game Driver Loop

### Enablement

配置类默认值层面，小游戏驱动支持但默认不开启：

- `Enabled = false`

开发环境配置里当前已开启：

- `Enabled = true`
- `ProbeIntervalSeconds = 60`
- `AckWindowSeconds = 15`
- `MinConsumePowerM = 1`
- `MaxConsumePowerM = 3`

这意味着当前状态更准确的描述是：

- 后端主循环已经实现
- 开发环境已打开
- 但机制整体仍属于“初步接通”阶段

### Probe creation

`GameDriverGrain` 会周期性检查：

1. 驱动是否启用
2. 当前是否已有 active round
3. 当前是否存在任意 `hero-bound balance > 0`

只有满足以上条件时才会创建新的 probe round，并通过消息服务广播：

- `probeId`
- `createdAtUtc`
- `expiresAtUtc`

### Ack settlement

当前结算规则：

- 一个 round 只接受第一次有效 ack
- ack 必须在过期时间之前到达
- 晚到 ack 会被忽略
- 过期未结算的 round 直接关闭，不发奖励

### Power consumption

round 结算后，系统会：

1. 从有余额的 Hero 中随机选一个
2. 在 `min-max` 范围内随机取一个消耗值
3. 实际消耗值不超过该 Hero 当前余额
4. 如果余额被耗尽，就移除这个 Hero 的绑定项

当前消耗规则的几个特点：

- 只从 Hero 绑定余额里扣
- 不会动 `PublicBalance`
- 选中 Hero 是随机的，不是按权重、等级、活跃度或贡献结算
- 单次消耗当前很小，默认只有 `1-3M`

## Current Reward Conversion

Hagipower 游戏奖励链路如下：

1. `GameDriverGrain` 消耗某个 Hero 的绑定 Power
2. 生成 `HeroGameRewardRequest`
3. `HeroGameGrain` 把 `ConsumedPowerM` 包装为 `HeroProgressionEvent`
4. 事件以 `HagipowerGameDriver` 作为 source type 送入经验系统
5. 经验系统按等量 `M` 写入 Hero progression
6. 同时写 hero history

当前等价关系是：

- `1M consumed power = 1M stored xp`

也就是说，Hagipower 当前不是经验倍率器，而是经验的另一条直接输入通道。

## Current External Surface

### HTTP API

当前可见接口：

- `GET /api/Hero/hagipower-status`

返回：

- `PublicBalance`
- `TotalBalance`
- `UpdatedAtUtc`
- `Bindings[]`

其中每个 binding 会补充：

- `HeroId`
- `HeroName`
- `HeroIcon`
- `Balance`

### History

Hagipower 相关经验落时间线时：

- `sourceCategory = hagipower-game-driver`
- `eventType = experience-gained`
- 可能伴随 `level-up`

当前 summary 文案格式：

- `-<consumedPower>M Power -> +<appliedXp>M XP`

### Debug support

当前只有一个明显的 runtime debug 开关：

- `ProbeIntervalSecondsOverride`

这意味着当前调试能力还比较轻量，主要用于控制 probe 频率。

## Current Limitations

### Public pool has no gameplay sink

`PublicBalance` 目前会积累，但当前后端链路里没有消费它的游戏规则。

这会带来两个直接结论：

- Public pool 现在更像“账本留存”而不是有效玩法资源。
- 如果后续设计要强调共享能量池、公共事件或全服机制，当前实现还需要新增消费侧。

### Hero-bound flow is still minimal

当前 Hero 绑定 Power 的玩法逻辑比较轻：

- 来源只有 execution token
- 玩法只有 probe -> ack -> random consume -> xp reward
- 单次奖励很小
- 没有连击、倍率、衰减、保底、失败惩罚、角色差异化

### Reward is currently one-dimensional

当前 Hagipower 的唯一直接收益是经验：

- 没有掉落
- 没有道具
- 没有技能树资源
- 没有 dungeon 门票
- 没有 public/private economy 联动

## Design Implications

- 如果未来要把 Hagipower 变成“主玩法资源”，当前最缺的是 source/sink 体系。
- 如果未来要强调“绑定 Hero 的身份感”，当前随机抽取 Hero 的规则需要重新审视。
- 如果未来要引入公共资源竞争或协作，`PublicBalance` 需要从旁支账本升级为真实玩法节点。
- 如果未来要强调节奏感与反馈，现在的 `60s probe + 15s ack + 1-3M reward` 只适合最早期验证。

## Source Pointers

- `repos/hagicode-core/src/PCode.Orleans/Grains/HagipowerGrain.cs`
- `repos/hagicode-core/src/PCode.Orleans/State/HagipowerState.cs`
- `repos/hagicode-core/src/PCode.Orleans/Grains/GameDriverGrain.cs`
- `repos/hagicode-core/src/PCode.Orleans/Grains/HeroGameGrain.cs`
- `repos/hagicode-core/src/PCode.Application/HeroAppService.Hagipower.cs`
- `repos/hagicode-core/src/PCode.Application.Contracts/Dto/HagipowerStatusDto.cs`
- `repos/hagicode-core/src/PCode.Application.Contracts/Configuration/HeroSystemSettingsConfiguration.cs`
- `repos/hagicode-core/src/PCode.Web/appsettings.Development.yml`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HagipowerGrainTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/GameDriverGrainTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HeroGameGrainTests.cs`
