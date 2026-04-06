# Technical Draft V1: Single Hero Adventure

## Status

当前文档是 `v1 技术草案`，用于把已经确定的机制决策整理成可落地的实现方向。

## Fixed Decisions

### Preserved from current system

- 等级范围保持 `Lv.1-1000` 不变。
- 经验曲线保持不变。
- 经验上限保持不变：
  - `50M / hour / hero`
  - `400M / day / hero`
- 继续保留类似现有机制的 `probe/ack` 轻交互。

### New mechanism constraints

- 先只做 `单人英雄冒险`。
- 每次冒险的初始消耗固定为 `1M Hagipower`。
- 冒险结果通过一系列随机事件决定。
- 冒险与现实时间结合，单次冒险流程通常在 `30s` 内完成。
- 冒险场景、主题风格、随机事件池按英雄等级段切换。
- 最终经验结果目标区间为 `-10M ~ +20M`。

## Design Goal

把当前偏“随机消耗一点 power 然后直接换经验”的小游戏，升级为：

- 先触发一次轻交互
- 再进入一个短时长、可观察的单人冒险
- 过程中出现若干随机事件
- 最终把冒险结果折算成经验

这样做的目标是让 `Hagipower` 更像“冒险燃料”，而不是“立即兑换经验的代币”。

## Recommended Core Loop

### Loop summary

1. Hero 累积到至少 `1M hero-bound Hagipower`
2. 系统发出 `Adventure Probe`
3. 用户 `ack` 接受这次冒险
4. 系统立即扣除 `1M power`
5. 系统基于当前参数一次性随机生成整局冒险脚本
6. `GameGrain` 按时间逐步播放脚本中的 `2-4` 个事件
7. 系统在终局步骤结算最终结果
8. 按结果发放 `-10M ~ +20M XP`
9. Hero 回到待命态，等待下一次 probe

### Why keep probe/ack

保留 probe/ack 的目的不是增加操作负担，而是保留一种“是否出发”的即时确认感：

- `probe` 负责发出“发现冒险机会”
- `ack` 负责确认“现在就出发”

这样既能保留当前机制的轻交互特征，也能让后续冒险过程有明确起点。

## State Machine

### Hero adventure states

- `idle`
  - Hero 空闲，等待下一次冒险机会。
- `probe-pending`
  - 系统已经发出冒险机会，等待用户 `ack`。
- `adventuring`
  - Hero 已出发，正在经历随机事件链。
- `resolving`
  - 事件链完成，正在计算最终结果并写入经验/history。
- `cooldown`
  - 可选短冷却态，用于防止 probe 立即连发。

### Transition rules

- `idle -> probe-pending`
  - 条件：Hero 绑定余额 `>= 1M` 且没有 active adventure。
- `probe-pending -> idle`
  - 条件：probe 过期未 ack，不扣 power。
- `probe-pending -> adventuring`
  - 条件：用户在时间窗内 ack，立即扣 `1M power` 并生成完整冒险脚本。
- `adventuring -> resolving`
  - 条件：计划中的事件链全部执行完毕，或提前触发终局事件。
- `resolving -> cooldown`
  - 条件：结算完成。
- `cooldown -> idle`
  - 条件：冷却结束。

## Time Model

### Probe window

- 推荐继续沿用当前量级：
  - `AckWindowSeconds = 15`

### Adventure duration

- 单次冒险总时长目标：
  - `5s ~ 30s`
- 推荐默认：
  - 普通副本 `8s ~ 18s`
  - 稍深探索 `18s ~ 30s`

### Event cadence

- 每次冒险推荐生成 `2-4` 个事件节点。
- 事件节点按固定时间偏移触发，例如：
  - `t+3s`
  - `t+8s`
  - `t+14s`
  - `t+22s`

这能让“现实时间参与”是可感知的，但又不会拖长到影响日常使用。

## Resolved Runtime Execution Model

### Core decision

技术实现直接确定为：

- `ack` 成功后，不是在运行中边走边随机。
- 系统会立刻基于当前参数一次性随机出整局冒险的所有步骤和最终结果。
- 这份结果不会一次性全部生效，也不会一次性全部展示。
- 后续只由 `GameGrain` 按脚本逐步播放。
- 只有播放到终局步骤时，最终经验才真正写入 progression。

### Why this is the chosen model

这个方案的优点很明确：

- 运行结果确定且可复现
- 中途不会因为额外状态波动导致结果漂移
- 播放层与结算层解耦，后续扩展事件类型容易
- 可以安全补做 history、dashboard、回放、调试与重放

### Parameter snapshot at start

脚本生成时，必须先固化一份开始快照：

- `heroId`
- `currentLevel`
- `levelBand`
- `scenePoolVersion`
- `eventPoolVersion`
- `startedAtUtc`
- `seed`
- `consumedPowerM = 1`

后续整局运行都基于这份快照，不再重新抽取。

### Generated adventure script

建议在开始时生成 `AdventureScript`：

- `runId`
- `heroId`
- `seed`
- `sceneId`
- `sceneName`
- `levelBand`
- `startedAtUtc`
- `endsAtUtc`
- `steps[]`
- `finalOutcome`

其中 `steps[]` 每一项建议至少包含：

- `stepIndex`
- `scheduledAtUtc`
- `eventType`
- `title`
- `summary`
- `scoreDelta`
- `payload`

`finalOutcome` 建议至少包含：

- `finalScore`
- `resultTier`
- `xpDelta`
- `completionSummary`

### Playback rule

运行时不是重新计算，而是按时间播放既定脚本：

1. 到达 `scheduledAtUtc`
2. `GameGrain` 取出下一步
3. 触发对应展示、history、dashboard 更新
4. 标记该步已播放
5. 全部步骤播放完成后，应用 `finalOutcome`

### Effect timing

时序上要严格区分三件事：

- `脚本已生成`
- `步骤已播放`
- `结果已生效`

`v1` 的明确规则是：

- 扣 `1M power` 发生在 `ack` 成功后立即执行
- 冒险步骤只在播放时进入用户可见面
- 最终 XP 只在最后一步播放完成后生效
- 中途步骤不直接改经验

## Adventure Content Model

### Level band -> theme mapping

推荐先采用固定等级段切内容池：

| 等级段 | 主场景主题 | 典型风格 |
| --- | --- | --- |
| `Lv.1-49` | 新手村周边 | 田野、小林地、浅层洞窟、废弃仓库 |
| `Lv.50-149` | 矿洞与洞窟 | 黑暗矿道、潮湿岩洞、地底裂隙 |
| `Lv.150-299` | 边境荒野 | 沼泽、古战场、盗匪营地、荒废驿站 |
| `Lv.300-499` | 古代遗迹 | 地下神殿、机关回廊、埋藏墓室 |
| `Lv.500-699` | 极端地带 | 熔岩裂谷、冰封洞廊、风暴高地 |
| `Lv.700-849` | 高危秘境 | 浮空废城、虚空裂口、深层禁区 |
| `Lv.850-1000` | 神话终域 | 星界迷宫、终焉地宫、传说王座遗址 |

### Dungeon composition

每个等级段先维护三层内容：

- `scene pool`
  - 决定这次冒险落在哪个主题场景。
- `event pool`
  - 决定冒险中会遇到哪些事件类型。
- `outcome flavor pool`
  - 决定结算文案、事件名称和风格差异。

### Event archetypes

推荐事件类型先固定为以下几类：

- `encounter`
  - 遭遇敌人、怪物、守卫、伏击者。
- `hazard`
  - 陷阱、落石、毒雾、坍塌、迷雾。
- `opportunity`
  - 捷径、补给点、宝箱、神秘商人、可利用地形。
- `recovery`
  - 休整、疗伤、重整装备、获得情报。
- `climax`
  - 小首领、机关房、最终事件、撤离选择。

## Resolution Model

### Hidden score

每次冒险内部维护一个 `adventure score`，初始为 `0`。

每个事件会对 score 产生修正，例如：

- 小成功：`+1`
- 大成功：`+2`
- 小失败：`-1`
- 大失败：`-2`

### Final XP mapping

最终不直接按单个事件发经验，而是按整局 score 映射到 XP 区间：

| 结果档位 | score 区间 | XP 区间 |
| --- | --- | --- |
| 惨败 | `<= -4` | `-10 ~ -6` |
| 失利 | `-3 ~ -1` | `-5 ~ -1` |
| 平局/空手而归 | `0 ~ 1` | `0 ~ +4` |
| 成功 | `2 ~ 4` | `+5 ~ +12` |
| 大成功 | `>= 5` | `+13 ~ +20` |

### Outcome tuning principle

- `1M power` 是入场成本。
- `-10 ~ +20 XP` 是结果振幅。
- 因为现有经验上限是 `50M/hour`，所以单次冒险结果不需要再做过大数值。
- 这会让体验重点落在“连续短冒险的过程”和“场景/事件变化”，而不是单次爆奖。

## Probe/Ack Interaction Design

### V1 recommendation

`v1` 只保留一次必要 ack：

- `probe` 出现时确认是否出发

冒险开始后不再要求用户多次点击，避免轻交互变成打断式操作。

### Visible probe payload

probe 建议至少包含：

- `heroId`
- `heroName`
- `requiredPowerM = 1`
- `recommendedSceneName`
- `levelBand`
- `expiresAtUtc`

这样用户在确认前能知道：

- 是谁去
- 去哪里
- 花多少 power
- 多久后失效

## Recommended Runtime Architecture

### Reuse

可以复用的现有能力：

- `HagipowerGrain`
  - 保存 hero-bound power 余额。
- `GameDriverGrain`
  - 继续担任 probe 调度器。
- `probe/ack` 消息下发逻辑
  - 继续作为轻交互入口。
- `HeroGameGrain`
  - 直接演进为运行态 `GameGrain`，负责脚本生成、逐步播放和最终结算。

### Determined runtime ownership

本草案直接确定：

- `GameDriverGrain`
  - 只负责判断是否发出 probe、维护 ack 窗口、在 ack 后启动游戏运行。
- `HeroGameGrain`
  - 作为 `GameGrain` 使用，负责生成并持久化整局 `AdventureScript`，然后逐步播放，最后把最终 XP 写入 progression。

这里不再引入额外的 `AdventureDirectorGrain`。

### New pieces

建议新增以下概念：

- `AdventureCatalog`
  - 场景池、等级段、事件池、结果文案配置。
- `AdventureScript`
  - 开始时一次性生成的整局脚本。
- `AdventureRunSnapshot`
  - 一次冒险当前运行态的快照 DTO。
- `AdventureEventRecord`
  - 冒险过程中发生的事件记录。
- `AdventureOutcomeResult`
  - 最终经验、事件摘要、场景信息、持续时间。

### Responsibility split

- `GameDriverGrain`
  - 判断是否生成 probe
  - 管理 ack 窗口
  - 在 ack 后委派 `HeroGameGrain` 启动 run
- `HeroGameGrain`
  - 读取开始参数
  - 一次性生成整局 `AdventureScript`
  - 持久化脚本和当前播放进度
  - 按时间逐步播放事件
  - 在终局步骤应用最终 XP delta
  - 写入 progression、history 和可见状态

## Backend Constraints

### Important: negative XP is not supported today

当前后端 progression 逻辑只接受非负经验：

- `HeroProgressionGrain` 会把负值归零
- 当前 `ApplyCappedStoredExperience` 也是面向正向增量设计

因此，如果要支持你指定的 `-10 ~ +20 XP`，必须新增能力。

### Recommended implementation rule for negative XP

推荐 `v1` 采用：

- 允许冒险结果产生负经验
- 负经验可以扣减当前 Hero 的总经验
- 但总经验不能低于 `0`
- `v1` 不允许掉级
  - 即负经验最低只能把经验扣到“当前等级起点”

这样可以同时满足：

- 有失败成本
- 不破坏“等级成长整体保持不变”的目标
- 不引入掉级带来的强挫败与额外兼容复杂度

### Required progression changes

为了支持上述规则，至少需要：

- `HeroProgressionEvent` 支持 signed delta
- `HeroProgressionGrain` 区分正向奖励与负向惩罚
- `HeroProgressionPersistenceService` 支持负向 history 记录
- `Hero.UpdateProgression(...)` 前增加“不可掉级”的 clamp 逻辑

## History And Feedback

### New history event suggestions

推荐新增：

- `adventure-started`
- `adventure-event`
- `adventure-completed`

当前如果不想新增 event type，也至少需要在 payload 中补足：

- `runId`
- 场景名
- 等级段
- 冒险耗时
- 事件链摘要
- `stepIndex`
- 最终 score
- 最终 XP delta

### Recommended dashboard data

实时看板建议新增：

- `adventureStatus`
  - idle / probe-pending / adventuring / resolving
- `adventureSceneName`
- `adventureEndsAtUtc`
- `lastAdventureResult`
- `lastAdventureXpDelta`

## V1 Scope Boundary

### Included

- 单 Hero 冒险
- 固定 `1M power` 入场
- `probe/ack` 出发确认
- 真实时间推进
- 按等级段切场景与事件池
- 最终 XP 结算

### Excluded for now

- 组队
- 公共池消费玩法
- 装备掉落
- 道具系统
- 多次交互式事件分支
- 赛季/排行榜联动

## Implementation Sequence

### Phase 1

- 保留当前 `probe/ack`
- 把 `1M power -> 冒险 run -> 正向 XP` 路线先接通
- 先不开放负经验，只记录“失败但不扣经验”的假结算日志

### Phase 2

- 补 signed XP 能力
- 开启 `-10 ~ +20 XP` 的完整结果区间
- 增加 history 与 dashboard 字段

### Phase 3

- 扩展等级段内容池
- 扩展事件风格与副本主题
- 视需要增加 public pool 玩法

## Remaining Risks

- 负经验与“不掉级”并存时，需要明确 current-level clamp 规则。
- 如果 probe 过于频繁，用户会感到被打断；如果过慢，又感受不到系统存在。
- 如果副本场景只是换皮文案，机制会重新退化成“随机换经验”。
- 如果场景主题很多但事件类型太少，内容会很快被看穿。
