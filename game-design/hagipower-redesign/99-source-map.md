# Source Map

## Progression Core

- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionCalculator.cs`
  - 等级范围、经验换算、阶段预算、阈值表、小时/天 cap。
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionConfiguration.cs`
  - 当前默认 progression 配置。
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionEvent.cs`
  - progression event 结构与 `HeroProgressionSourceType`。
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroProgressionSummary.cs`
  - 对外暴露的等级摘要字段。

## Progression Runtime

- `repos/hagicode-core/src/PCode.Orleans/Grains/HeroProgressionGrain.cs`
  - 去重、remainder、cap、flush、初始化 backfill。
- `repos/hagicode-core/src/PCode.Orleans/State/HeroProgressionState.cs`
  - Grain 持久状态。
- `repos/hagicode-core/src/PCode.Application/HeroProgressionPersistenceService.cs`
  - Hero 更新与 history 写入。
- `repos/hagicode-core/src/PCode.Application/HeroProgressionService.cs`
  - 应用层入口。

## Execution Statistics Feed

- `repos/hagicode-core/src/PCode.ClaudeHelper/AI/AIService.cs`
  - 普通执行统计落库并实时写入 Hagipower。
- `repos/hagicode-core/src/PCode.Orleans/Grains/ExecutorStatisticsTracker.cs`
  - 流式执行统计落库并实时写入 Hagipower。
- `repos/hagicode-core/src/PCode.Orleans/Grains/ClaudeCodeGrain.cs`
  - ClaudeCode 相关执行统计落库并实时写入 Hagipower。

## Hero Exposure

- `repos/hagicode-core/src/PCode.DomainServices.Contracts/Entities/Hero.cs`
  - Hero 实体里实际持久化的 progression 字段。
- `repos/hagicode-core/src/PCode.Application/HeroProgressionDtoMapper.cs`
  - progression 字段如何映射到 DTO。
- `repos/hagicode-core/src/PCode.Application.Contracts/Dto/HeroDto.cs`
  - Hero 对外等级/经验字段。
- `repos/hagicode-core/src/PCode.Application.Contracts/Dto/HeroRealtimeDashboardDto.cs`
  - 实时看板卡片字段。
- `repos/hagicode-core/src/PCode.Application/HeroRealtimeDashboardComposer.cs`
  - Dashboard 聚合器。

## History

- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroHistoryEventType.cs`
  - 当前事件类型枚举。
- `repos/hagicode-core/src/PCode.DomainServices.Contracts/HeroHistorySourceCategories.cs`
  - 当前 sourceCategory 常量。
- `repos/hagicode-core/docs/hero-history.md`
  - Hero history 存储方式与查询契约。

## Hagipower Core

- `repos/hagicode-core/src/PCode.Orleans/Grains/HagipowerGrain.cs`
  - 执行转 Power、hero/public 双账本、probe 消耗。
- `repos/hagicode-core/src/PCode.Orleans/State/HagipowerState.cs`
  - Hagipower 持久状态与快照输出。
- `repos/hagicode-core/src/PCode.Orleans.IGrains/HagipowerContracts.cs`
  - Hagipower 快照合同。
- `repos/hagicode-core/src/PCode.Application/HagipowerService.cs`
  - 应用层写入入口。

## Hagipower Game Driver

- `repos/hagicode-core/src/PCode.Orleans/Grains/GameDriverGrain.cs`
  - probe round 生命周期、ack 结算、reward dispatch。
- `repos/hagicode-core/src/PCode.Orleans/Grains/HeroGameGrain.cs`
  - 把 `ConsumedPowerM` 转成 progression event。
- `repos/hagicode-core/src/PCode.Application.Contracts/Configuration/HeroSystemSettingsConfiguration.cs`
  - `HagipowerGameDriverConfiguration` 配置定义。
- `repos/hagicode-core/src/PCode.Web/appsettings.Development.yml`
  - 开发环境当前启用值。
- `repos/hagicode-core/src/PCode.Orleans.IGrains/HagipowerGameDriverDebugSettingsSnapshot.cs`
  - debug override 结构。

## Hagipower API Surface

- `repos/hagicode-core/src/PCode.Application/HeroAppService.Hagipower.cs`
  - 状态聚合逻辑。
- `repos/hagicode-core/src/PCode.Application.Contracts/Dto/HagipowerStatusDto.cs`
  - 状态接口输出字段。
- `repos/hagicode-core/src/PCode.HttpApi/Controllers/HeroController.Hagipower.cs`
  - `GET /api/Hero/hagipower-status`

## Tests Worth Re-Reading Before Redesign

- `repos/hagicode-core/tests/PCode.Application.Tests/HeroProgressionCalculatorTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HeroProgressionGrainTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HagipowerGrainTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/GameDriverGrainTests.cs`
- `repos/hagicode-core/tests/PCode.Orleans.Tests/HeroGameGrainTests.cs`
