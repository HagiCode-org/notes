# Design Draft: Reward And Feedback

## Purpose

用于规划 `v1` 冒险机制落地后用户能看到什么，以及后端需要暴露哪些数据。

## V1 User-Facing Surfaces

- Hero 卡片
- 实时 dashboard
- History timeline
- 战报
- 系统通知
- 探针/小游戏提示

## V1 Moment-to-Moment Feedback

- 获得冒险机会时
  - 显示 Hero、场景、消耗 `1M power`、剩余确认时间
- 冒险进行中
  - 显示场景名、当前事件摘要、预计结束时间
  - 这些事件来自开局已生成的脚本，但按播放进度逐步公开
- 成功结算时
  - 显示结果档位、总耗时、最终 XP 变化
- 失败结算时
  - 显示失败原因摘要与负经验结果
- probe 过期时
  - 显示“机会错过，本次未消耗 power”

## Information Clarity

- 明确区分 `hero-bound power`
- 明确说明本次入场固定消耗 `1M`
- 明确说明结果是“冒险结算 XP”，不是“1M power 直接换 1M xp”
- 如果出现负经验，要明确说明这是冒险失败结果，不是系统异常

## Backend Observability

### Suggested DTO fields

- `adventureStatus`
- `adventureSceneName`
- `adventureLevelBand`
- `adventureEndsAtUtc`
- `lastAdventureResult`
- `lastAdventureXpDelta`

### Suggested history payload fields

- `sceneName`
- `levelBand`
- `durationSeconds`
- `eventChain`
- `finalScore`
- `xpDelta`
- `consumedPowerM`

### Suggested new history events

- `adventure-started`
- `adventure-event`
- `adventure-completed`

### Suggested debug knobs

- `ProbeIntervalSecondsOverride`
- `AdventureDurationScale`
- `ForcedSceneId`
- `ForcedOutcomeBand`

## Anti-Confusion Rules

- 不把 `power cost` 和 `xp result` 写成固定兑换关系
- 不把“事件过程中的临时表现”直接显示成最终经验
- 不把“脚本已生成”误显示成“结果已生效”
- 如果负经验被 clamp 到当前等级起点，需要明确写成“已触及本级最低经验”

## Interaction Rule

- 保留前端主动 `ack`
- 不追加中途多次确认
- `v1` 仍然是轻交互，不做复杂分支选择 UI
