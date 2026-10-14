# 内容系统与数据规范

> 当前内容：五职业、19 个剧情章节、30 类事故、20 条以上操作、系统抢修/代码追凶/交互沙箱、无尽与每日模式、简体中文和英文。

本文定义 `ROOT//PANIC` 的关卡、服务、事故、指令和升级如何以数据驱动方式组织。目标是让新增内容尽量只修改配置，不修改游戏引擎。

## 1. 内容设计原则

1. **数据与逻辑分离**：内容描述放在 `src/data/`，规则计算放在 `src/core/`。
2. **组合优先**：通过标签、条件和效果组合内容，避免为单个事故编写特殊逻辑。
3. **可预测但不单调**：玩家应能理解风险来源，同时无法仅靠背答案通关。
4. **每个选择都有代价**：强力操作需要消耗时间、预算、稳定性或其他资源。
5. **文本简洁专业**：像真实监控系统，但保留适度幽默和戏剧性。

## 2. 文件规划

```text
src/data/
├── services.ts             # 服务定义
├── incidents.ts            # 事故定义
├── actions.ts              # 操作指令
├── upgrades.ts             # 局内升级与职业成长
├── scenarios.ts            # 19 章与任务类型路由
├── interactionMissions.ts  # 节点、边、卡片、目标区和答案
└── debugMissions.ts        # 代码、提交、证据和补丁
```

所有内容项必须有稳定且唯一的 `id`。存档和统计只记录 `id`，不保存展示名称。

### 2.1 任务类型

| 类型 | 数据重点 | 规则引擎 |
| --- | --- | --- |
| `operations` | 时长、威胁、预算、最低 SLA、事故目标 | `engine.ts` |
| `debug` | 文件、提交、线索、候选补丁、测试门禁 | `debugEngine.ts` |
| `interactive` | `kind`、节点/边或卡片/目标区、初始状态和正确解 | `interactionEngine.ts` |

交互任务的 `kind` 支持 `connect`、`isolate`、`classify`、`sequence`、`allocate`、`switchboard`、`calibrate`。其中资源调度使用带 CPU/内存容量的目标区，开关矩阵和信号校准使用统一 `controls` 配置；新增同类关卡应优先增加数据，不应复制 DOM 事件或验证逻辑。

## 3. 命名与标签

### 3.1 ID 规范

- 使用小写英文和点号：`incident.db.deadlock`
- 类型作为前缀：`service.api_gateway`、`action.restart`
- ID 一经发布不随展示名称变化
- 删除内容时保留迁移映射，避免旧存档失效

### 3.2 推荐标签

| 分类 | 标签示例 | 用途 |
| --- | --- | --- |
| 服务类型 | `edge`、`compute`、`storage`、`data` | 事故目标筛选 |
| 故障类型 | `latency`、`capacity`、`security`、`dependency` | 抗性与升级匹配 |
| 操作方式 | `diagnose`、`mitigate`、`repair`、`rollback` | 构筑与教程 |
| 风险等级 | `low`、`medium`、`high`、`critical` | UI 和音效反馈 |
| 环境 | `prod`、`staging`、`legacy` | 场景规则 |

## 4. 服务数据

```js
export const services = [
  {
    id: 'service.api_gateway',
    nameKey: 'service.api_gateway.name',
    descriptionKey: 'service.api_gateway.description',
    tags: ['edge', 'compute'],
    maxHealth: 100,
    baseLoad: 24,
    capacity: 80,
    criticality: 1.2,
    dependencies: ['service.auth'],
    resistances: {
      security: 0.1,
      capacity: -0.1
    }
  }
];
```

服务不直接包含运行时状态。生命值、负载和状态效果在游戏状态中创建，以便同一服务模板被多个场景复用。

### 首批服务

| 服务 | 定位 | 主要风险 |
| --- | --- | --- |
| API Gateway | 外部流量入口 | 流量洪峰、证书、限流 |
| Auth Service | 身份认证 | Token、依赖超时、安全事件 |
| Order Service | 核心业务 | 线程池、发布故障、雪崩 |
| Redis Cache | 缓存层 | 内存、热点 Key、穿透 |
| Primary DB | 主数据库 | 慢查询、锁、连接耗尽 |
| Message Queue | 异步链路 | 消息堆积、重复消费 |

## 5. 事故数据

```js
export const incidents = [
  {
    id: 'incident.db.deadlock',
    nameKey: 'incident.db.deadlock.name',
    descriptionKey: 'incident.db.deadlock.description',
    severity: 3,
    targetQuery: {
      anyTags: ['data']
    },
    duration: 24,
    tickEffects: [
      { type: 'healthDelta', value: -1.5 },
      { type: 'loadMultiplier', value: 1.2 }
    ],
    counters: ['action.kill_session', 'action.rollback_release'],
    escalatesTo: 'incident.db.connection_exhausted',
    escalationAfter: 12,
    weight: 8,
    minimumThreat: 2
  }
];
```

### 事故字段约束

- `severity`：1 至 4，分别对应提示、警告、严重、致命。
- `duration`：以逻辑秒为单位；`null` 表示必须人工解除。
- `weight`：同一候选池中的随机权重，不代表概率百分比。
- `tickEffects`：只允许使用引擎已注册的标准效果。
- `counters`：推荐操作，不强制唯一解。
- `escalatesTo`：事故未处理时可能产生的后续事故。

### 首批事故池

1. CPU 持续飙升
2. 内存泄漏
3. 磁盘空间不足
4. 数据库慢查询
5. 数据库死锁
6. 连接池耗尽
7. Redis 热点 Key
8. 缓存穿透
9. 消息队列积压
10. 上游接口超时
11. 错误发布
12. 证书即将过期
13. 可疑登录洪峰
14. DNS 解析异常
15. 监控误报
16. 区域网络抖动

## 6. 操作指令数据

```js
export const actions = [
  {
    id: 'action.restart_service',
    nameKey: 'action.restart_service.name',
    tags: ['mitigate', 'repair'],
    targetQuery: { anyTags: ['compute'] },
    cost: { focus: 12 },
    cooldown: 14,
    castTime: 2.5,
    effects: [
      { type: 'removeStatusByTag', tag: 'runtime' },
      { type: 'healthDelta', value: 18 },
      { type: 'temporaryCapacityMultiplier', value: 0.65, duration: 5 }
    ],
    risk: {
      chance: 0.08,
      effect: { type: 'healthDelta', value: -10 }
    }
  }
];
```

### 操作类别

| 类别 | 示例 | 典型代价 |
| --- | --- | --- |
| 诊断 | 查看日志、链路追踪、查询指标 | 时间、专注度 |
| 缓解 | 限流、扩容、降级、切流 | 预算、服务能力 |
| 修复 | 重启、清理、杀会话、刷新缓存 | 冷却、短时风险 |
| 回滚 | 回滚发布、恢复配置、数据恢复 | 时间、变更额度 |
| 自动化 | 自动扩容、告警聚合、故障转移 | 高前置成本 |

当前版本提供 20 条以上操作。扩充时至少保持 4 条通用操作，并保证每类服务都有诊断与处置路径。

## 7. 升级数据

```js
export const upgrades = [
  {
    id: 'upgrade.observability.trace_sampling',
    nameKey: 'upgrade.observability.trace_sampling.name',
    category: 'observability',
    rarity: 'uncommon',
    maxLevel: 3,
    prerequisites: [],
    modifiers: [
      { stat: 'diagnose.castTime', operation: 'multiply', value: 0.85 }
    ]
  }
];
```

升级分为可观测性、可靠性、自动化、资源效率和应急能力五个方向。升级应改变决策方式，而不只是增加数值。

## 8. 场景数据

```js
export const scenarios = [
  {
    id: 'scenario.black_friday_eve',
    nameKey: 'scenario.black_friday_eve.name',
    seedMode: 'daily',
    duration: 420,
    services: [
      'service.api_gateway',
      'service.auth',
      'service.order',
      'service.redis',
      'service.primary_db',
      'service.message_queue'
    ],
    phases: [
      { at: 0, threat: 1, spawnInterval: [18, 26] },
      { at: 120, threat: 2, spawnInterval: [13, 20] },
      { at: 270, threat: 3, spawnInterval: [9, 16] },
      { at: 390, threat: 4, spawnInterval: [6, 11] }
    ],
    victory: { type: 'survive' },
    modifiers: ['modifier.traffic_growth']
  }
];
```

场景负责定义节奏，不直接决定每次具体出现的事故。引擎根据威胁等级、服务状态、标签、冷却和随机种子构建候选池。

## 9. 平衡规则

### 9.1 服务风险值

```text
risk = incidentSeverity × serviceCriticality × dependencyPressure
```

该数值主要用于 AI 事故导演和 UI 排序，不直接作为伤害值。

### 9.2 事故生成约束

- 同一服务默认最多同时存在 3 个事故。
- 同一事故在冷却结束前不可重复生成。
- 不生成当前完全无法处理的事故组合。
- 连续两次致命事故之间必须保留反应窗口。
- 玩家长期无压力时逐步提高生成权重，而不是突然提高伤害。

### 9.3 难度调整维度

优先调整：

1. 事故出现频率
2. 同时事故数量
3. 事故链长度
4. 资源恢复速度
5. 容错时间

最后才调整纯伤害数值，以避免产生不透明的“数值墙”。

## 10. 文本与叙事风格

- 系统信息：简短、准确、可扫描。
- 事故描述：先说明现象，再给可调查线索。
- 角色对白：体现 IT 工作语境，但不依赖晦涩梗。
- 幽默来源于情境，例如“这不是 Bug，是未记录的容灾演练”。
- 不使用真实企业、产品事故或可识别个人作为负面素材。

示例：

```text
[P1] 订单服务错误率突破 18%
变更时间线显示 3 分钟前完成了一次灰度发布。
```

## 11. 本地化

所有展示文本使用语言键：

```js
export const zhCN = {
  'incident.db.deadlock.name': '数据库死锁',
  'incident.db.deadlock.description': '事务相互等待，写入吞吐正在下降。'
};
```

规则：

- 简体中文与英文同时维护；新增展示文本必须提供两种语言。
- 数值、快捷键和单位使用插值参数。
- 不在模板字符串中拼接语序相关的文本。
- UI 测试需覆盖中文长文本和英文长单词。

## 12. 内容制作流程

1. 在设计表中定义目标体验和反制手段。
2. 添加内容配置和本地化文本。
3. 运行数据校验，检查 ID、引用、字段和循环依赖。
4. 使用固定种子完成至少三轮模拟。
5. 检查是否存在唯一解、死局或无法理解的失败。
6. 记录平衡调整原因，避免反复修改同一参数。

## 13. 自动校验要求

构建或测试时必须检查：

- ID 是否唯一
- 语言键是否完整
- 引用的服务、事故和操作是否存在
- 数值是否在允许范围内
- 事故链是否形成不可退出的循环
- 是否存在没有任何可用反制手段的事故
- 场景是否至少包含一个胜利条件
- 存档版本与内容版本是否兼容

## 14. 内容验收清单

每条新内容合入前确认：

- 玩家能从画面和文本理解发生了什么
- 至少有两种合理应对方式，或明确标注为教学内容
- 成功与失败都有清晰反馈
- 与现有内容组合时不会造成必败局
- 手机端无需悬停即可查看全部信息
- 文本不会溢出主要组件
- 固定种子下可稳定复现

## 15. 内容包边界

核心引擎只识别标准 Schema、规则效果和语言键。官方或社区内容包可以注册服务、事故、操作、升级、场景、对话、角色外观与主题资源。

内容包不得执行任意脚本；自定义效果必须引用引擎白名单中的规则类型，以保持静态部署、离线运行与社区扩展的安全边界。

