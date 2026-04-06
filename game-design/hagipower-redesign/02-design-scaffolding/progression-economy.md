# Design Draft: Progression And Economy

## Purpose

用于固定 `v1` 对等级、经验和 Hagipower 经济的技术边界。

## V1 Locked Rules

### Level curve

- 保持 `Lv.1-1000` 不变
- 保持现有四阶段经验预算不变
- 保持现有 progression version 和经验上限逻辑不变

### XP result range

- 新冒险的最终 XP 结果区间固定为 `-10M ~ +20M`
- 单次入场成本固定为 `1M power`
- XP 由整局冒险结果产生，不按单个事件直接发奖

### Cap policy

- 保留现有 `50M/hour`、`400M/day`
- `v1` 继续按 Hero 粒度结算
- 正向 XP 继续走现有 cap
- 负向 XP 不应受正向 cap 限制，但需要新增 signed delta 支持

### Hagipower economy

- 来源：继续来自 execution token
- 消耗点：`v1` 只有“启动单人冒险”
- 账本：`v1` 只消费 hero-bound power
- 公共池：暂不进入主玩法

## V1 Mapping Rule

推荐采用“事件分数 -> 结果档位 -> XP 区间”的映射：

| 档位 | XP 区间 |
| --- | --- |
| 惨败 | `-10 ~ -6` |
| 失利 | `-5 ~ -1` |
| 平局/空手而归 | `0 ~ +4` |
| 成功 | `+5 ~ +12` |
| 大成功 | `+13 ~ +20` |

## Reward Mix

- `v1` 只产出经验
- 暂不加入掉落、门票、材料、装备和外观

## Technical Constraint

当前后端不支持负经验，因此 `-10 ~ +20` 不能直接沿用现有 `HeroProgressionGrain`。

`v1` 推荐规则：

- 允许负经验结果
- Hero 总经验最低不低于 `0`
- Hero 不会因此掉级
- 负经验最低只能扣到当前等级起点

## Deferred

- public pool 的玩法职责
- 多 Hero 或组队经济
- 冒险失败是否返还部分 power
- 是否增加非经验奖励
