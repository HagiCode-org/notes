# Current State: Level And Experience

## One-Screen Summary

- 等级区间为 `Lv.1` 到 `Lv.1000`。
- 当前曲线版本为 `v2`。
- 当前默认配置下，目标是按 `400M/day * 365 days = 146000M` 的总经验预算到达满级。
- 当前执行事件的真实经验来源只有 `totalTokens`，不是 `baseExperience` 或 `toolExperience`。
- 普通执行经验的实时增量上报链路当前未在源码中找到；已确认接通的是“历史 execution statistics backfill”和“Hagipower 奖励写入”。
- 当前经验采用双层单位：
  - `raw experience`: 直接等于执行的 `totalTokens`
  - `stored experience (M)`: 每 `1_000_000 raw` 折算为 `1M`
- 当前按 Hero 单独限流：
  - `50M / hour`
  - `400M / day`
- 小于 `1M` 的经验余数会跨事件累计。
- 但如果命中小时/天上限，溢出的余数不会保留到未来窗口。

## Current Configuration

### Core constants

| 项目 | 当前值 |
| --- | ---: |
| 最低等级 | `1` |
| 最高等级 | `1000` |
| 每日目标经验 | `400M` |
| 满级目标天数 | `365` |
| 满级总经验 | `146000M` |
| 存储经验换算 | `1M = 1_000_000 raw xp` |
| 每小时经验上限 | `50M` |
| 每日经验上限 | `400M` |
| 进度版本 | `2` |

### Current stages

| Stage | 等级区间 | Budget Share | Weight Exponent | 当前阶段预算 |
| --- | --- | ---: | ---: | ---: |
| Rookie Sprint | `1-100` | `3%` | `1.15` | `4380M` |
| Growth Run | `101-300` | `12%` | `1.2` | `17520M` |
| Veteran Climb | `301-700` | `35%` | `1.28` | `51100M` |
| Legend Marathon | `701-1000` | `50%` | `1.35` | `73000M` |

这意味着当前曲线是明确的“前快后慢”：

- 早期升级成本低，便于快速起量。
- 中后期成本持续抬升。
- `Lv.700+` 吃掉整条曲线一半预算。

## Current Milestones

下面的里程碑值基于当前默认配置推导，单位均为 `M stored xp`。

| 等级 | 累计经验 | 升到本级新增经验 |
| --- | ---: | ---: |
| `Lv.1` | `0` | `0` |
| `Lv.2` | `1` | `1` |
| `Lv.10` | `36` | `7` |
| `Lv.25` | `232` | `19` |
| `Lv.50` | `998` | `42` |
| `Lv.100` | `4380` | `93` |
| `Lv.150` | `5262` | `37` |
| `Lv.200` | `8271` | `83` |
| `Lv.300` | `21900` | `190` |
| `Lv.500` | `32568` | `120` |
| `Lv.700` | `73000` | `289` |
| `Lv.850` | `87468` | `224` |
| `Lv.900` | `101295` | `329` |
| `Lv.950` | `120655` | `444` |
| `Lv.1000` | `146000` | `568` |

## Current Reward Model

### Execution -> XP conversion

从公式层面看，普通执行经验的核心逻辑非常直接：

1. `GrantedRawExperience = totalTokens`
2. `GrantedExperience = floor(totalTokens / 1_000_000)`
3. 不足 `1M` 的部分作为 remainder 挂在 Hero 的 pending 状态中

虽然事件对象里还保留了以下字段：

- `BaseExperience = 40`
- `ToolExperience = min(toolCallCount * 4, 180)`
- `TokenExperience = floor(totalTokens / 1_000_000)`

但当前真正用于经验公式的是 `GrantedRawExperience`，而它目前只等于 `totalTokens`。换句话说：

- `BaseExperience` 目前更像元数据。
- `ToolExperience` 目前更像元数据。
- 真实经验公式并没有把基础奖励和工具调用奖励加进最终账本。

### Current wiring status

当前源码里可以确认两件事：

- 普通执行会稳定写入 `HeroExecutionStatistic`
- Hagipower 会稳定从执行统计中拿到 `TotalTokens`

但当前没有找到普通执行完成后调用 `IHeroProgressionService.ReportExecutionAsync(...)` 的源码入口。当前可确认的 progression 写入来源是：

- 启动时基于历史 `HeroExecutionStatistic` 的 backfill
- Hagipower 游戏奖励通过 `HeroGameGrain -> HeroProgressionGrain` 的实时写入

因此，当前“普通执行经验”更准确的描述是：

- 经验规则已经定义
- 历史重算已经实现
- 实时增量接线暂未在当前源码中显式出现

### Current cap behavior

每个 Hero 都有独立经验窗口：

- 小时窗口：最多 `50M`
- 日窗口：最多 `400M`

当前实现的关键行为：

- 同一 Hero 的 caps 互相独立于其他 Hero。
- remainder 可以跨事件累计，只要还没触顶。
- 如果本次累计请求超过可发上限，超出的 raw xp 会被直接丢弃，而不是延后到下一小时或下一天。

这个特性对后续设计非常重要，因为它意味着当前系统更偏向“硬截断”而不是“排队发放”。

## Current Persistence And Lifecycle

### Confirmed write paths

当前可以确认的 progression 写入路径有两条：

#### Path A: startup backfill for normal executions

1. 读取历史 `HeroExecutionStatistic`
2. 按当前 `totalTokens -> stored xp` 规则重算
3. 应用到 Hero progression 字段

#### Path B: live write for Hagipower rewards

1. `GameDriverGrain` 结算 probe 奖励
2. `HeroGameGrain` 构造 `HeroProgressionEvent`
3. 发送给 `HeroProgressionGrain`
4. Grain 做去重、cap 处理、remainder 处理
5. 达到阈值或定时器触发后 flush 到持久层
6. 更新 `Hero.TotalExperience`、`CurrentLevel` 等字段
7. 写入 hero history 事件

### Current flush behavior

- Grain flush 阈值：`50` 个 pending 事件
- Grain flush 定时：每 `30s`
- 已处理的 `ExecutionMessageId` 会保留最多 `30 days`
- 去重缓存上限：`20_000`

### Current backfill behavior

系统初始化时会根据历史执行统计做一次 backfill：

- 读取历史 execution statistics
- 重新按当前规则计算每个 Hero 的 stored xp
- 回写到 Hero 的 progression 字段
- progression version 已应用后不重复 backfill

这表示当前经验系统不是纯增量逻辑，还包含一次“历史重算 -> 回填”的启动阶段。

## Current Player-Facing Data

Hero 当前对外暴露的进度字段包括：

- `CurrentLevel`
- `TotalExperience`
- `CurrentLevelStartExperience`
- `NextLevelExperience`
- `ExperienceProgressPercent`
- `RemainingExperienceToNextLevel`
- `LastExperienceGain`
- `LastExperienceGainAtUtc`

这些字段会出现在：

- `HeroDto`
- 地牢成员/可选 Hero DTO
- battle report DTO
- realtime dashboard DTO

## Current History Recording

经验系统目前会写两类关键时间线事件：

- `experience-gained`
- `level-up`

其中：

- 普通执行奖励的 `sourceCategory = hero-progression`
- Hagipower 游戏奖励的 `sourceCategory = hagipower-game-driver`

Hagipower 奖励的 summary 目前格式是：

- `-xM Power -> +yM XP`

## Current Constraints Worth Carrying Into Redesign

- 当前经验单位已经默认以 `M` 作为对外展示压缩单位。
- 当前等级曲线已经是明确的四阶段预算模型，不是随意散点表。
- 当前曲线锚点实际上固定为 `Lv.1000` 总经验，不是动态随全服最高等级变化。
- 当前 caps 是 per-hero 粒度，不是全局共享经济阀门。
- 当前 overflow 处理是“丢弃”，这会直接影响玩家对浪费感、刷本节奏、补偿机制的设计。
- 普通执行经验目前更像“已定义规则 + 已支持 backfill”，而不是“已明确接通的实时增长通道”。

## Source Pointers

- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionCalculator.cs`
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionConfiguration.cs`
- `repos/hagicode-core/src/PCode.Orleans/Grains/HeroProgressionGrain.cs`
- `repos/hagicode-core/src/PCode.Application/HeroProgressionPersistenceService.cs`
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/Entities/Hero.cs`
- `repos/hagicode-core/src/PCode.ClaudeHelper/AI/AIService.cs`
- `repos/hagicode-core/src/PCode.Orleans/Grains/ExecutorStatisticsTracker.cs`
- `repos/hagicode-core/src/PCode.Orleans/Grains/ClaudeCodeGrain.cs`
- `repos/hagicode-core/tests/PCode.Application.Tests/HeroProgressionCalculatorTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HeroProgressionGrainTests.cs`
