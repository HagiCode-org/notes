# Open Questions

## Before Detailed Design

### Experience

- 负经验是否允许真正扣减 Hero 总经验？
- 负经验是否允许导致掉级？当前草案建议“不掉级”。
- 负经验是否应该受正向 `50M/hour`、`400M/day` cap 影响？当前草案建议“不受影响”。

### Hagipower

- `PublicBalance` 的真实玩法职责是什么？
- `v1` 之后是否还保留固定 `1M` 入场，还是按等级段扩展 power 成本？
- `PublicBalance` 是否会在 `v2` 进入公共副本或全局事件？

### Player experience

- 玩家是否应该看到完整事件链，还是只看摘要？
- 失败时的负经验是否需要额外补偿文案或保护机制？
- 未来是否要增加“中途事件选择”，还是继续保持纯观察式冒险？

### Implementation

- history event type 是新增枚举，还是先复用现有事件类型并扩展 payload？
- `HeroGameGrain` 的脚本与播放状态是否单独落库，还是先沿用 Orleans state 即可？
