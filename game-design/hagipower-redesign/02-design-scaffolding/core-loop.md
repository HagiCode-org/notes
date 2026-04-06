# Design Draft: Core Loop

## Purpose

用于定义新的基于 Hagipower 的主循环。当前文件已经根据 `v1` 决策填写。

## V1 Summary

- 核心定位：`Hagipower` 是短时冒险燃料，不是立即兑换经验的代币。
- 核心循环：`积累 hero-bound power -> probe -> ack -> 生成整局脚本 -> 30s 内逐步播放 -> 结果结算 -> 获得或损失经验`
- 交互强度：保留一次 `probe/ack` 轻交互，不做多次中断式确认。
- 冒险范围：先只做 `单人英雄冒险`。

## V1 Core Loop

1. Hero 绑定 Hagipower 达到 `1M`
2. 系统发出一次冒险 `probe`
3. 用户在窗口期内 `ack`
4. 系统扣除 `1M power`
5. 立即随机生成本局完整冒险脚本
6. `GameGrain` 在 `5s ~ 30s` 内逐步播放 `2-4` 个事件
7. 系统在最终步骤产出结果
8. 最终结算 `-10M ~ +20M XP`

## V1 Loop States

- 待机
- 冒险机会待确认
- 冒险进行中
- 结果结算中
- 可选短冷却

## Ownership Model

- `v1` 只消费 `hero-bound Hagipower`
- `PublicBalance` 暂不参与冒险玩法
- Hero 之间暂不转移资源

## Failure Model

- probe 超时：不扣 power，回到待机
- 冒险失败：允许出现负经验结果
- `v1` 推荐不允许掉级，负经验最低扣到当前等级起点
- 随机性在开局时一次性确定，中途不再重新 roll

## Decision Notes

- 保留 probe/ack：已确认
- 随机选 Hero：否，`v1` 是单 Hero 冒险，由收到 probe 的该 Hero 自己出发
- 单次冒险耗时：通常 `30s` 内
- 运行方式：`ack` 后立即生成完整脚本，由 `GameGrain` 逐步播放
