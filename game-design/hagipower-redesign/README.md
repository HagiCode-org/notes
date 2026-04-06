# Hagipower Redesign

## Audience

面向后续要继续做数值、系统、交互和实现联动设计的产品与研发同学。

## Goal

在开始新的 Hagipower 游戏机制设计前，先把当前后端已经落地的等级、经验、Hagipower 事实整理成可复用笔记，并搭出后续可以持续填充的文档骨架。

## Scope

本目录当前包含：

- 当前等级与经验机制的确认版摘要。
- 当前 Hagipower 积累、消耗、奖励转换链路的确认版摘要。
- 后续详细设计的工作流占位。
- 关键源码入口映射。

本目录当前不包含：

- 前端视觉稿或交互稿。
- 已确定的新机制方案。
- 生产环境数值结论。

## Current Snapshot

- 等级范围是 `Lv.1` 到 `Lv.1000`。
- 当前默认曲线下，升到 `Lv.1000` 需要 `146000M` 累计经验。
- 当前执行奖励的实际经验来源是 `totalTokens`，`1_000_000 raw xp = 1M stored xp`。
- 普通执行经验的公式与 backfill 已经存在，但当前源码里未看到普通执行实时写入 progression grain 的调用链。
- 每个 Hero 当前有独立经验上限：`50M/小时`、`400M/天`。
- Hagipower 当前按执行 `tokenCount` 累积，但只有 `hero-bound balance` 会进入现有小游戏消耗链路。
- Hagipower 小游戏驱动默认配置支持 `1-3M` 的随机消耗并将其等量转成存储经验。

## Folder Structure

- `01-current-state/level-and-experience.md`
  - 当前等级曲线、经验换算、经验写入与历史记录行为。
- `01-current-state/hagipower.md`
  - 当前 Hagipower 余额模型、probe 机制、奖励转换与暴露接口。
- `02-design-scaffolding/core-loop.md`
  - 后续设计主循环骨架。
- `02-design-scaffolding/technical-draft-v1.md`
  - 当前已经确认约束下的 v1 技术草案。
- `02-design-scaffolding/progression-economy.md`
  - 后续等级/经济/数值设计骨架。
- `02-design-scaffolding/reward-feedback.md`
  - 后续反馈、展示与可观测性设计骨架。
- `03-open-questions.md`
  - 进入精细设计前的关键问题清单。
- `99-source-map.md`
  - 与当前实现对应的源码/文档索引。

## Usage Rule

后续新增内容时，先判断是“确认事实”还是“设计提案”：

- 如果是从 `hagicode-core` 读出来的已实现行为，写到 `01-current-state/`。
- 如果是准备讨论、评估、权衡的新方案，写到 `02-design-scaffolding/` 或 `03-open-questions.md`。
