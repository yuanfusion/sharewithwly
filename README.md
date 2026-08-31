# Godot 8 km × 8 km ASCII 开放世界

## 技术设计、实施路线、验收标准与 Agent 构建提示词

> 目标引擎：Godot 4.x  
> 推荐语言：静态类型 GDScript  
> 游戏形态：二维俯视角、字符/ASCII 风格、程序生成开放世界  
> 核心策略：确定性生成、区块流式加载、差异存档、分层 NPC 模拟

---

## 目录

1. [项目目标](#1-项目目标)
2. [关键设计决策](#2-关键设计决策)
3. [规模与性能预算](#3-规模与性能预算)
4. [项目目录结构](#4-项目目录结构)
5. [坐标系统](#5-坐标系统)
6. [数据模型](#6-数据模型)
7. [ASCII 渲染系统](#7-ascii-渲染系统)
8. [程序化世界生成](#8-程序化世界生成)
9. [区块流式加载](#9-区块流式加载)
10. [碰撞、视野与寻路](#10-碰撞视野与寻路)
11. [NPC 模拟](#11-npc-模拟)
12. [时间与离线模拟](#12-时间与离线模拟)
13. [存档系统](#13-存档系统)
14. [多线程边界](#14-多线程边界)
15. [调试工具](#15-调试工具)
16. [测试与验收标准](#16-测试与验收标准)
17. [按顺序实施的开发阶段](#17-按顺序实施的开发阶段)
18. [交给编码 Agent 的总控 Prompt](#18-交给编码-agent-的总控-prompt)
19. [分阶段 Agent Prompt](#19-分阶段-agent-prompt)
20. [最终完成清单](#20-最终完成清单)

---

# 1. 项目目标

构建一个可运行的 Godot ASCII 开放世界原型，具备以下能力：

- 世界逻辑尺寸为 `8000 × 8000` 个地块。
- 默认 `1 个地块 = 1 米`，对应 `8 km × 8 km`。
- 世界由整数地块 ID 表示，ASCII 字符只是显示层。
- 世界由统一种子确定性生成。
- 地图以 `64 × 64` 地块区块流式加载。
- 任何时刻只加载玩家附近的区块。
- 支持海洋、平原、森林、沙漠、沼泽、山地等生物群系。
- 支持河流、道路、聚落和局部兴趣点。
- 支持玩家移动、碰撞、视野和探索。
- 支持世界种子、玩家状态、全局状态和区块差异存档。
- 支持活跃、简化、统计三级 NPC 模拟。
- 支持离开区块后的时间推进和重新加载。
- 提供确定性、存档、区块边界和 NPC 调度测试。

## 1.1 第一版非目标

第一版暂不追求：

- 一次性生成并保存全部 6400 万个地块。
- 每个地块创建一个 Node。
- 每个 NPC 永久保持一个场景节点。
- 对整个世界执行逐格 A*。
- 完整生态、政治和经济仿真。
- 网络多人同步。
- 完整地下多层世界。
- 复杂天气流体模拟。

这些能力应建立在稳定的分块和数据架构之上，后续增量加入。

---

# 2. 关键设计决策

## 2.1 基础参数

```gdscript
const WORLD_SIZE_CELLS := Vector2i(8000, 8000)
const CHUNK_SIZE := 64
const ACTIVE_RADIUS := 2
const PRELOAD_RADIUS := 3
const MACRO_CELL_SIZE := 32
```

说明：

- 每个区块包含 `64 × 64 = 4096` 个地块。
- 世界约有 `125 × 125 = 15625` 个区块。
- `ACTIVE_RADIUS = 2` 表示玩家附近最多保持 `5 × 5` 个活跃区块。
- `PRELOAD_RADIUS = 3` 可提前生成外围区块，减少移动卡顿。
- 宏观地图尺寸约为 `250 × 250`。

## 2.2 必须遵守的原则

1. **数据与显示分离**  
   地形保存数字 ID，渲染器负责把 ID 转换为字符和颜色。

2. **生成与存档分离**  
   原始世界由种子生成，存档只记录差异。

3. **逻辑与节点分离**  
   未激活的 NPC 和区块只是数据，不应存在于场景树。

4. **生成顺序不得影响结果**  
   区块先生成或后生成，结果必须完全相同。

5. **禁止每格一个 Node**  
   只能使用批量瓦片、MultiMesh 或自定义绘制。

6. **禁止每帧扫描全部 NPC**  
   使用 `next_update_time`、优先队列或时间轮调度。

7. **主线程只负责场景树和最终渲染提交**  
   工作线程仅处理纯数据计算。

---

# 3. 规模与性能预算

## 3.1 地形数据

完整世界有：

```text
8000 × 8000 = 64,000,000 个地块
```

如果每个地块仅使用 1 字节：

```text
约 64 MB
```

但如果使用 Dictionary、Object 或 Node 表示每格，内存会大幅膨胀。因此：

- 运行时区块地形使用 `PackedByteArray`。
- 需要额外信息时使用平行数组。
- 不应为每个地块创建对象。

## 3.2 建议性能目标

这些是工程目标，不是绝对保证，最终以目标硬件实测为准：

- 正常移动时保持稳定 60 FPS。
- 主线程区块应用时间尽量低于每帧 2 ms。
- 初始可玩区域在数秒内出现。
- 活跃区块通常不超过 25 个。
- 预加载区块通常不超过 49 个。
- 完整模拟 NPC 目标为 50～300 名。
- 简化模拟 NPC 可达到数千名。
- 遥远区域使用聚落统计，不逐个高频更新。
- 自动存档不产生明显帧卡顿。

---

# 4. 项目目录结构

建议创建以下结构：

```text
res://
├─ project.godot
├─ assets/
│  ├─ fonts/
│  ├─ glyphs/
│  └─ palettes/
├─ scenes/
│  ├─ main/
│  │  └─ main.tscn
│  ├─ world/
│  │  ├─ world.tscn
│  │  └─ chunk_view.tscn
│  ├─ actors/
│  │  ├─ player.tscn
│  │  └─ npc_view.tscn
│  └─ ui/
│     ├─ hud.tscn
│     └─ debug_overlay.tscn
├─ scripts/
│  ├─ core/
│  │  ├─ constants.gd
│  │  ├─ event_bus.gd
│  │  ├─ game_clock.gd
│  │  └─ game_state.gd
│  ├─ data/
│  │  ├─ tile_def.gd
│  │  ├─ chunk_data.gd
│  │  ├─ macro_world_data.gd
│  │  ├─ npc_data.gd
│  │  └─ settlement_data.gd
│  ├─ generation/
│  │  ├─ seed_util.gd
│  │  ├─ macro_world_generator.gd
│  │  ├─ biome_generator.gd
│  │  ├─ hydrology_generator.gd
│  │  ├─ settlement_generator.gd
│  │  ├─ road_generator.gd
│  │  ├─ poi_generator.gd
│  │  └─ chunk_generator.gd
│  ├─ world/
│  │  ├─ world_manager.gd
│  │  ├─ chunk_streamer.gd
│  │  ├─ chunk_repository.gd
│  │  └─ chunk_view.gd
│  ├─ rendering/
│  │  ├─ glyph_catalog.gd
│  │  └─ ascii_renderer.gd
│  ├─ actors/
│  │  ├─ player_controller.gd
│  │  └─ actor_view.gd
│  ├─ navigation/
│  │  ├─ local_pathfinder.gd
│  │  ├─ region_graph.gd
│  │  └─ route_service.gd
│  ├─ npc/
│  │  ├─ npc_manager.gd
│  │  ├─ npc_scheduler.gd
│  │  ├─ npc_materializer.gd
│  │  ├─ utility_ai.gd
│  │  ├─ action_executor.gd
│  │  └─ settlement_simulator.gd
│  ├─ save/
│  │  ├─ save_manager.gd
│  │  ├─ save_manifest.gd
│  │  ├─ chunk_delta.gd
│  │  └─ save_migrations.gd
│  ├─ debug/
│  │  ├─ debug_overlay.gd
│  │  └─ world_debug_draw.gd
│  └─ tests/
│     ├─ test_runner.gd
│     ├─ test_seed_determinism.gd
│     ├─ test_chunk_borders.gd
│     ├─ test_save_roundtrip.gd
│     └─ test_npc_scheduler.gd
└─ tests/
   └─ test_main.tscn
```

建议注册以下 Autoload：

```text
EventBus
GameClock
GameState
SaveManager
```

`WorldManager` 和 `NPCManager` 可以先作为 `world.tscn` 的子节点，避免过度使用全局单例。

---

# 5. 坐标系统

必须明确区分以下坐标：

| 坐标 | 类型 | 用途 |
|---|---|---|
| 世界地块坐标 | `Vector2i` | 逻辑位置，范围约 0～7999 |
| 区块坐标 | `Vector2i` | 指定地块属于哪个区块 |
| 区块内坐标 | `Vector2i` | 范围 0～63 |
| 屏幕/像素坐标 | `Vector2` | 仅用于显示 |
| 宏观坐标 | `Vector2i` | 生物群系、城市、河流规划 |

## 5.1 坐标转换

必须正确处理负数坐标，不能直接依赖截断除法。

```gdscript
static func world_to_chunk(cell: Vector2i) -> Vector2i:
    return Vector2i(
        floori(float(cell.x) / CHUNK_SIZE),
        floori(float(cell.y) / CHUNK_SIZE)
    )

static func world_to_local(cell: Vector2i) -> Vector2i:
    return Vector2i(
        posmod(cell.x, CHUNK_SIZE),
        posmod(cell.y, CHUNK_SIZE)
    )

static func local_to_index(local: Vector2i) -> int:
    return local.y * CHUNK_SIZE + local.x
```

即使第一版世界限制在正坐标内，也应测试负坐标转换，避免未来扩展时出现错误。

---

# 6. 数据模型

## 6.1 地块 ID

```gdscript
enum TileId {
    VOID,
    DEEP_WATER,
    SHALLOW_WATER,
    SAND,
    GRASS,
    FOREST_FLOOR,
    SWAMP,
    ROCK,
    SNOW,
    ROAD,
    WALL,
    FLOOR,
    DOOR_CLOSED,
    DOOR_OPEN
}
```

不要把 `"."`、`"#"`、`"~"` 直接作为世界数据。

## 6.2 地块定义

每种地块需要：

```gdscript
class_name TileDef
extends Resource

@export var id: int
@export var display_name: String
@export var glyph: String
@export var foreground: Color
@export var background: Color
@export var walkable: bool
@export var transparent: bool
@export var movement_cost: float = 1.0
```

地块定义是静态配置，地块实例只保存紧凑 ID。

## 6.3 区块数据

```gdscript
class_name ChunkData
extends RefCounted

var coord: Vector2i
var tiles: PackedByteArray
var flags: PackedByteArray
var biome_ids: PackedByteArray
var generated := false
var dirty := false
var last_access_tick: int
```

第一版可以只保存 `tiles`。确认需要时再增加并行数组。

## 6.4 NPC 数据

```gdscript
class_name NPCData
extends RefCounted

var id: int
var archetype_id: int
var world_cell: Vector2i
var settlement_id: int
var home_id: int
var workplace_id: int

var health: float = 100.0
var hunger: float
var fatigue: float
var morale: float
var danger: float

var current_action: int
var action_target_cell: Vector2i
var action_target_entity_id: int
var action_finish_time: int
var next_update_time: int
var last_update_time: int

var simulation_level: int
var persistent := false
```

存档中避免直接保存 Node、Callable、信号连接或场景实例。

---

# 7. ASCII 渲染系统

## 7.1 第一版方案

第一版使用：

```text
TileMapLayer + 字符图集
```

每个瓦片对应一个字符图像。字符图集需要：

- 固定宽度字体。
- 统一单元格尺寸，例如 `16 × 16`。
- 最近邻过滤，避免像素模糊。
- 字符前景色可通过替代瓦片、顶点颜色或 Shader 控制。

## 7.2 渲染边界

渲染器只接收：

```text
需要显示的区块数据
地块 ID → 字符图块映射
实体 ID → 字符图块映射
```

世界生成器不能直接操作 `TileMapLayer`。

## 7.3 图层顺序

建议拆为：

```text
TerrainLayer
FeatureLayer
ItemLayer
ActorLayer
EffectLayer
FogLayer
```

第一版可只实现：

```text
TerrainLayer
ActorLayer
FogLayer
```

## 7.4 优化路线

只有在性能分析证明 TileMapLayer 成为瓶颈时，才升级：

```text
TileMapLayer
    ↓
MultiMeshInstance2D
    ↓
自定义 CanvasItem / Shader 字符缓冲
```

禁止在没有测量数据的情况下提前重写渲染器。

---

# 8. 程序化世界生成

推荐使用分层生成，而不是一个噪声函数决定所有内容。

## 8.1 生成阶段

```text
世界种子
  ↓
宏观海拔
  ↓
海洋和陆地
  ↓
温度与湿度
  ↓
生物群系
  ↓
河流
  ↓
聚落
  ↓
道路
  ↓
兴趣点
  ↓
区块局部细节
```

## 8.2 种子规则

必须区分各生成层：

```gdscript
enum SeedLayer {
    MACRO_HEIGHT,
    TEMPERATURE,
    MOISTURE,
    HYDROLOGY,
    SETTLEMENTS,
    ROADS,
    CHUNK_DETAILS,
    VEGETATION,
    POI,
    NPC_SPAWN
}
```

建议使用稳定的坐标混合函数：

```gdscript
static func mix_seed(
    world_seed: int,
    coord: Vector2i,
    layer: int
) -> int:
    var value := world_seed
    value ^= coord.x * 73856093
    value ^= coord.y * 19349663
    value ^= layer * 83492791
    return value
```

要求：

- 相同输入必须得到相同输出。
- 不依赖区块加载顺序。
- 各生成层使用独立种子。
- 生成算法改动时提升 `generator_version`。

## 8.3 宏观地图

宏观地图建议 `250 × 250`：

```text
每个宏观格代表 32 × 32 个世界地块
```

每格保存：

```text
海拔
温度
湿度
生物群系
水流方向
河流水量
聚落归属
道路信息
```

宏观地图体积较小，可以在新建世界时一次性生成并缓存。

## 8.4 海拔

建议组合三层噪声：

```text
大陆尺度低频噪声
+ 地形起伏中频噪声
+ 山脉脊状噪声
```

然后执行：

- 海拔归一化。
- 海平面阈值。
- 可选的边缘海洋衰减。
- 少量平滑。
- 保留局部变化。

## 8.5 温度

温度建议由以下部分组成：

```text
纬度基础温度
+ 温度噪声
- 海拔降温
```

## 8.6 湿度

湿度建议由以下部分组成：

```text
湿度噪声
+ 邻近海洋/河流加成
- 高山背风惩罚（后续可选）
```

## 8.7 生物群系

生物群系由规则表决定：

```text
低海拔                       → 海洋
接近海平面且邻近陆地         → 海滩
高海拔                       → 山地/雪山
高温低湿                     → 沙漠
高温高湿                     → 沼泽/雨林
中温中湿                     → 草原/森林
低温                         → 苔原
```

不要直接把一个噪声值映射为生物群系。

## 8.8 河流

建议在宏观地图生成：

1. 从高海拔、高降水区域选择源头。
2. 每一步流向最低邻格。
3. 记录流量累积。
4. 到达海洋、湖泊或已有河流时结束。
5. 对死洼地执行填洼或允许形成湖泊。
6. 区块生成时把宏观河道细化为地块级河道。

河流必须先于聚落和道路生成。

## 8.9 聚落

候选位置根据以下条件评分：

```text
邻近淡水
地形平坦
可耕地比例
距离其他聚落适中
避开深海和陡峭山地
靠近资源
```

生成后为每个聚落保存稳定 ID：

```gdscript
var settlement_id: int
var center: Vector2i
var settlement_type: int
var population: int
var wealth: float
var food_stock: float
```

## 8.10 道路

道路生成流程：

1. 构建聚落节点图。
2. 优先连接相邻聚落。
3. 使用最小生成树保证主要聚落连通。
4. 增加少量额外边，避免道路网络完全树状。
5. 使用地形代价寻路生成实际路线。
6. 跨河位置形成桥梁候选。

道路成本示例：

```text
平原：1
森林：2
沙漠：2
沼泽：5
山地：8
浅水：12
深水：不可通行
```

## 8.11 区块细节

区块生成器输入：

```text
世界种子
区块坐标
宏观地图
跨区块固定特征
```

输出：

```text
PackedByteArray tiles
出生点描述
兴趣点描述
静态阻挡信息
```

区块边界必须通过世界坐标采样，不能使用仅从局部坐标开始的噪声。

## 8.12 建筑与兴趣点

建议：

- 野外细节：噪声阈值或稀疏采样。
- 独立建筑：预制模板。
- 村庄布局：道路骨架 + 建筑模板。
- 地牢房间：BSP。
- 小范围复杂装饰：WFC。

不要使用 WFC 生成整个 8 km 世界。

---

# 9. 区块流式加载

## 9.1 区块生命周期

```text
UNLOADED
  ↓ 请求
QUEUED
  ↓ 后台生成/读取
GENERATING
  ↓ 数据完成
READY
  ↓ 主线程显示
ACTIVE
  ↓ 离开范围
INACTIVE
  ↓ 保存脏数据
UNLOADED
```

## 9.2 每帧更新

玩家跨入新区块时：

1. 计算新的需要区块集合。
2. 对缺失区块排队。
3. 优先加载距离玩家最近的区块。
4. 将超出范围的区块标记卸载。
5. 脏区块必须先写入差异缓存。
6. 限制每帧创建和销毁的 ChunkView 数量。

## 9.3 缓存

建议有两层：

```text
active_chunks：当前显示和模拟
cached_chunks：暂时保留的纯数据
```

使用最近最少使用策略淘汰缓存，避免玩家在边界来回移动时重复生成。

## 9.4 任务去重

必须防止：

- 同一区块重复提交后台任务。
- 已经离开范围的任务完成后仍被显示。
- 旧世界任务结果写入新世界。

每个任务应携带：

```text
world_session_id
chunk_coord
request_generation
```

应用结果前再次核对。

---

# 10. 碰撞、视野与寻路

## 10.1 碰撞

ASCII 网格游戏优先使用逻辑碰撞：

```gdscript
func can_enter(cell: Vector2i) -> bool:
    var tile_id := world_manager.get_tile(cell)
    return tile_catalog.get_def(tile_id).walkable
```

如果是逐格移动，不必给每块墙创建 `StaticBody2D`。

## 10.2 视野

建议使用递归阴影投射或对称阴影投射：

- 以玩家格为中心。
- 只处理视野半径内地块。
- 透明属性来自 `TileDef.transparent`。
- 保存“当前可见”和“曾经探索”两个状态。

```text
visible_now：当前视野
explored：永久探索记录
```

探索状态属于存档差异。

## 10.3 分层寻路

### 局部路径

玩家附近使用格子级 A*：

- 当前区块及相邻区块。
- 考虑地形移动代价。
- 动态阻挡由 NPC 管理器提供。

### 全局路径

远距离移动使用：

```text
聚落图
道路图
区块入口图
```

NPC 不需要计算数千格的完整路径。它只需知道：

```text
当前位置 → 最近道路节点 → 一系列全局节点 → 目标附近 → 局部路径
```

### 路径缓存

缓存键可以是：

```text
起始区域 ID
目标区域 ID
移动类型
道路版本
```

道路改变后提升道路版本或局部失效缓存。

---

# 11. NPC 模拟

## 11.1 三级模拟

### Level 0：活跃模拟

玩家附近：

- 实际 Node2D/NPCView。
- 逐格移动。
- 局部寻路。
- 视野和战斗。
- 高频需求变化。
- Utility AI 选择行动。
- 状态机执行行动。

### Level 1：简化模拟

附近但不可见：

- 只有 `NPCData`。
- 不更新动画。
- 不逐格移动。
- 使用预计抵达时间。
- 战斗使用快速结算。
- 每数秒或数十秒更新一次。

### Level 2：统计模拟

远距离区域：

- 普通 NPC 合并到聚落人口统计。
- 每游戏小时或每天更新。
- 只保留重要、命名或与任务相关 NPC 的个体数据。
- 生产、消耗、出生、死亡、迁移以统计公式处理。

## 11.2 Utility AI

推荐需求：

```text
饥饿
疲劳
安全
工作
社交
医疗
任务
探索
```

行为评分示例：

```gdscript
func score_eat(npc: NPCData, context: Dictionary) -> float:
    if not context.has_food_source:
        return 0.0
    return pow(npc.hunger, 2.0) * 1.4

func score_flee(npc: NPCData, context: Dictionary) -> float:
    return npc.danger * 2.5
```

选择行为时应加入：

- 行为最低阈值。
- 当前行为保持奖励，防止频繁抖动。
- 紧急行为中断。
- 行为冷却。
- 目标有效性检查。

## 11.3 状态机

Utility AI 决定做什么，状态机负责怎么做。

例如“吃饭”：

```text
SELECT_FOOD_SOURCE
  ↓
REQUEST_PATH
  ↓
MOVE_TO_TARGET
  ↓
INTERACT
  ↓
CONSUME
  ↓
COMPLETE
```

任何阶段都必须处理：

- 目标消失。
- 路径失败。
- 被攻击。
- 区块卸载。
- 游戏读取存档。

## 11.4 NPC 调度器

不要遍历全部 NPC。

每个 NPC 保存：

```text
next_update_time
```

调度器按时间取出到期 NPC，并限制每帧预算。

```gdscript
const MAX_ACTIVE_UPDATES_PER_FRAME := 80
const MAX_COARSE_UPDATES_PER_FRAME := 200
```

可以使用：

- 最小堆。
- 分桶时间轮。
- 按下一更新时间排序的数组。

第一版建议最小堆，逻辑清晰且便于测试。

## 11.5 NPC 实体化

进入活跃范围：

```text
NPCData → NPCView
```

离开活跃范围：

```text
NPCView 当前状态 → NPCData
销毁 NPCView
```

`NPCData` 是权威状态，`NPCView` 只是表现。

## 11.6 NPC 稳定 ID

所有需要持久化的实体必须有全局唯一整数 ID：

```text
NPC
建筑
箱子
任务对象
聚落
队伍
```

禁止用 NodePath 或运行时 Instance ID 作为存档 ID。

---

# 12. 时间与离线模拟

## 12.1 游戏时间

统一使用整数游戏时间：

```text
game_time_seconds
```

所有系统基于这一时间源，不直接使用系统日期。

## 12.2 区块重新激活

不要逐秒补算。

```gdscript
var elapsed := current_game_time - npc.last_update_time
npc.hunger = clampf(
    npc.hunger + HUNGER_RATE * elapsed,
    0.0,
    1.0
)
```

长时间行为直接比较结束时间：

```gdscript
if current_game_time >= npc.action_finish_time:
    complete_action(npc)
```

## 12.3 随机离线事件

随机事件必须是确定性的，建议使用：

```text
实体 ID
时间窗口编号
事件类型
世界种子
```

混合成事件种子，避免每次读取存档得到不同结果。

---

# 13. 存档系统

## 13.1 核心方案

```text
程序生成基础世界
+ 区块差异
+ 玩家状态
+ 全局状态
+ 重要实体状态
```

不要保存全部原始地图。

## 13.2 目录

```text
user://saves/slot_01/
├─ manifest.dat
├─ manifest.bak
├─ player.dat
├─ global.dat
├─ settlements.dat
├─ named_npcs.dat
└─ chunks/
   ├─ 0_0.dat
   ├─ 0_1.dat
   └─ 1_0.dat
```

## 13.3 Manifest

至少包含：

```gdscript
{
    "magic": "ASCII_WORLD_SAVE",
    "save_version": 1,
    "generator_version": 1,
    "world_seed": 123456789,
    "world_size": Vector2i(8000, 8000),
    "chunk_size": 64,
    "game_time": 0,
    "created_at_unix": 0,
    "last_saved_at_unix": 0
}
```

## 13.4 区块差异

只保存：

- 被修改的地块。
- 被发现的地块。
- 被移除的程序生成实体。
- 新增的持久实体。
- 容器状态。
- 门、机关、火灾等状态。
- 重要局部事件。

对于 `64 × 64` 区块，地块索引范围是 `0～4095`，可以紧凑保存。

```gdscript
class_name ChunkDelta
extends RefCounted

var coord: Vector2i
var modified_indices: PackedInt32Array
var modified_tile_ids: PackedByteArray
var explored_bits: PackedByteArray
var removed_spawn_ids: PackedInt64Array
var persistent_entities: Array[Dictionary]
```

## 13.5 保存算法

```text
1. 收集脏数据快照
2. 写入 .tmp
3. flush 并关闭文件
4. 验证文件头和基本字段
5. 旧文件移动为 .bak
6. .tmp 替换正式文件
7. 成功后清除 dirty 标记
```

只有写入成功后才能清除脏标记。

## 13.6 读取算法

```text
1. 尝试正式文件
2. 检查 magic 和版本
3. 读取失败则尝试 .bak
4. 执行版本迁移
5. 根据种子生成基础数据
6. 应用区块差异
7. 恢复实体引用
8. 恢复游戏时间和玩家状态
```

## 13.7 版本迁移

必须区分：

```text
save_version：序列化格式版本
generator_version：世界生成算法版本
```

生成算法改变时可选择：

1. 保留旧生成器代码。
2. 提供迁移程序。
3. 明确旧世界不兼容。

开发阶段至少要检测版本并给出清晰错误，不能静默生成错误世界。

## 13.8 自动保存

建议触发条件：

- 固定真实时间间隔。
- 玩家进入新区域。
- 睡觉或休息。
- 退出到主菜单。
- 重要任务完成。

每次只保存有限数量的脏区块，避免单帧集中写入。

---

# 14. 多线程边界

## 14.1 可以放入工作线程

- 噪声采样。
- 地形数组生成。
- 生物群系计算。
- 区块差异编码。
- 不访问场景树的 NPC 批量计算。
- 路径预计算。

## 14.2 应留在主线程

- 创建或销毁 Node。
- 修改活动场景树。
- 将生成结果写入 TileMapLayer。
- 更新 HUD。
- 发出依赖场景节点的信号。

## 14.3 线程结果格式

工作线程只返回纯数据：

```gdscript
{
    "session_id": session_id,
    "request_id": request_id,
    "chunk_coord": chunk_coord,
    "tiles": PackedByteArray(),
    "spawn_descriptors": []
}
```

主线程验证结果仍然有效后再应用。

---

# 15. 调试工具

第一版就应提供调试覆盖层，显示：

```text
FPS
玩家世界坐标
玩家区块坐标
已加载区块数
等待生成任务数
缓存区块数
脏区块数
活跃 NPC 数
简化模拟 NPC 数
本帧 NPC 更新数
世界种子
游戏时间
```

建议快捷键：

```text
F1：显示/隐藏调试信息
F2：显示区块边界
F3：显示生物群系
F4：显示寻路区域
F5：强制保存
F6：重新读取当前存档
F7：暂停 NPC 模拟
F8：时间加速
```

提供开发控制台命令：

```text
teleport x y
spawn_npc count
save
reload
set_time seconds
regenerate_chunk x y
profile_chunks
```

---

# 16. 测试与验收标准

## 16.1 确定性测试

同一世界种子和区块坐标：

```text
生成两次 → tiles 完全相同
```

不同加载顺序：

```text
A → B → C
C → A → B
```

最终区块校验值必须一致。

## 16.2 区块边界测试

检查：

- 河流不会在区块边界突然中断。
- 道路不会错位。
- 噪声地形连续。
- 建筑不会被重复生成。
- 跨区块实体只有一个稳定 ID。

## 16.3 坐标测试

至少测试：

```text
(0, 0)
(63, 63)
(64, 64)
(-1, -1)
(-64, -64)
(-65, -65)
```

## 16.4 存档往返测试

步骤：

1. 新建世界。
2. 修改地块。
3. 创建或移除实体。
4. 保存。
5. 清空运行时状态。
6. 读取。
7. 比较恢复后的权威数据。

## 16.5 崩溃恢复测试

模拟：

- 正式文件损坏。
- 临时文件残留。
- 备份文件存在。
- 版本号不兼容。

程序必须避免静默丢档。

## 16.6 NPC 测试

- 同一 NPC 不会被调度两次。
- NPC 卸载前状态会写回数据。
- 远距离移动在正确时间抵达。
- 目标消失后行为能够失败并重新选择。
- NPC 在存档读取后保持稳定 ID。
- 每帧更新量不会超过预算。

## 16.7 最小可玩验收

完成后必须能：

1. 输入或随机创建世界种子。
2. 出生在合法陆地区域。
3. 使用键盘移动。
4. 按需加载周围区块。
5. 看到连续地形和生物群系。
6. 遇到至少一种聚落或兴趣点。
7. 看到 NPC 执行吃饭、休息、工作或移动。
8. 修改一个地块并保存。
9. 退出重新进入后恢复玩家位置和地块变化。
10. 传送到远处后，原区域被卸载且游戏不中断。

---

# 17. 按顺序实施的开发阶段

任何阶段失败时不得跳到后面的阶段。

## 阶段 0：建立项目和技术骨架

### 工作

- 创建或检查 Godot 4.x 项目。
- 检测实际 Godot 版本。
- 建立目录结构。
- 建立主场景和世界场景。
- 创建常量、事件总线、游戏时钟。
- 建立无第三方插件的测试入口。
- 建立静态类型和错误输出约定。

### 验收

- 项目可从编辑器和命令行启动。
- 没有脚本解析错误。
- 测试入口能够报告成功和失败。

---

## 阶段 1：ASCII 渲染与玩家移动

### 工作

- 创建字符图集或临时内置字符资源。
- 创建 TileDef 和 GlyphCatalog。
- 渲染一个固定 `64 × 64` 测试区块。
- 创建玩家字符。
- 实现网格移动和逻辑碰撞。
- 创建相机和 HUD。

### 验收

- 不存在每格一个 Node。
- 玩家无法走入不可通行地块。
- 摄像机正确跟随。
- ASCII 字符清晰且无过滤模糊。

---

## 阶段 2：确定性区块生成

### 工作

- 实现 SeedUtil。
- 实现 FastNoiseLite 配置。
- 使用世界坐标采样。
- 实现基础海拔、湿度、温度和生物群系。
- 实现 ChunkData 和 ChunkGenerator。
- 增加确定性测试和边界测试。

### 验收

- 相同种子生成结果一致。
- 加载顺序不影响结果。
- 相邻区块边界连续。

---

## 阶段 3：宏观世界

### 工作

- 创建 `250 × 250` 宏观地图。
- 生成海陆、温度、湿度、生物群系。
- 生成河流。
- 选择聚落。
- 创建道路图。
- 将宏观特征投影到地块区块。

### 验收

- 河流能够抵达海洋或湖泊。
- 聚落不位于非法地块。
- 主要聚落道路连通。
- 同一世界种子生成相同宏观地图。

---

## 阶段 4：区块流式加载

### 工作

- 实现 WorldManager。
- 实现 ChunkRepository。
- 实现 ChunkStreamer。
- 实现请求队列和优先级。
- 实现数据缓存。
- 实现 ChunkView 池或受控创建销毁。
- 实现传送测试。

### 验收

- 玩家移动时区块正确加载和卸载。
- 不出现空洞、重复任务或旧结果覆盖。
- 快速传送不会崩溃。
- 活跃区块数量受限。

---

## 阶段 5：视野、探索与局部寻路

### 工作

- 实现透明度和视野。
- 保存区块探索位图。
- 实现本地 A*。
- 支持动态阻挡。
- 实现路径缓存的基本失效机制。

### 验收

- 墙壁正确遮挡视野。
- 已探索区域在离开视野后保留记忆显示。
- NPC 能绕过障碍抵达同一区域内目标。

---

## 阶段 6：差异存档

### 工作

- 创建 SaveManifest。
- 创建 ChunkDelta。
- 实现玩家和全局状态保存。
- 实现脏区块跟踪。
- 实现 `.tmp`、正式文件和 `.bak` 流程。
- 实现版本检查和迁移入口。
- 实现存档往返测试。

### 验收

- 基础世界不会被完整写入存档。
- 修改地块后重新读取能够恢复。
- 未修改区块不产生不必要的大文件。
- 损坏正式文件时可以尝试备份。
- 写入失败时 dirty 状态不会被错误清除。

---

## 阶段 7：基础 NPC 系统

### 工作

- 创建 NPCData 和稳定 ID 分配器。
- 创建 NPCManager 和 NPCScheduler。
- 创建 NPCView 实体化系统。
- 实现饥饿、疲劳和危险需求。
- 实现 Utility AI。
- 实现吃饭、睡觉、闲逛、逃跑行为。
- 实现行为状态机。

### 验收

- NPC 会根据需求改变行为。
- 当前行为不会每帧抖动。
- 目标失效后可以恢复。
- NPC 离开活跃区域后不再保留 Node。
- NPC 返回时状态一致。

---

## 阶段 8：分层 NPC 和远距离旅行

### 工作

- 实现三级模拟精度。
- 实现聚落和道路区域图。
- 实现远距离旅行时间。
- 实现离线时间推进。
- 实现聚落人口和资源统计。
- 实现重要 NPC 个体持久化。

### 验收

- 远距离 NPC 不执行逐格寻路。
- 聚落人口不会全部实体化。
- NPC 能在预计时间到达目标区域。
- 读取存档后长时间行为正确恢复。

---

## 阶段 9：后台任务与性能优化

### 工作

- 将纯数据区块生成移到 WorkerThreadPool。
- 增加会话 ID 和请求 ID 防止过期结果。
- 批量处理简化 NPC。
- 限制每帧显示应用数量。
- 增加性能计数器。
- 根据分析结果决定是否优化渲染。

### 验收

- 工作线程不修改场景树。
- 快速创建新世界不会混入旧任务结果。
- 不会因区块集中完成造成长帧。
- 性能统计能够定位生成、渲染和 NPC 成本。

---

## 阶段 10：整合、文档与发布检查

### 工作

- 运行全部自动测试。
- 执行 30 分钟移动、传送和存档压力测试。
- 检查错误日志。
- 编写 README。
- 记录操作键位。
- 记录存档格式和生成器版本。
- 清理临时代码和调试硬编码。

### 验收

- 满足最小可玩验收清单。
- 没有脚本解析错误。
- 没有持续增长的区块、线程任务或 NPCView。
- 新建、保存、读取和退出流程完整。

---

# 18. 交给编码 Agent 的总控 Prompt

以下内容可以整体复制给具备文件编辑和命令执行能力的编码 Agent。

```text
你是一名资深 Godot 4.x 游戏系统工程师。你的任务是在当前工作区内，从现有状态开始，构建一个“8 km × 8 km ASCII 风格程序生成开放世界”的完整可运行原型。

你必须严格按阶段工作，不得跳过测试，不得一次性随意堆砌未经验证的代码。你必须先检查仓库现状、Godot 版本、已有文件和用户改动，然后根据实际情况增量实现。不得覆盖或回退不属于你的现有改动。

====================
一、项目目标
====================

1. 使用 Godot 4.x 和静态类型 GDScript。
2. 世界逻辑尺寸为 8000 × 8000 个地块，默认每格 1 米。
3. 画面完全使用 ASCII/字符图块表现，但内部必须保存语义化整数 Tile ID，不能把字符本身当作地形数据。
4. 地图按 64 × 64 地块划分为区块。
5. 默认显示玩家附近 5 × 5 个活跃区块，并可预加载 7 × 7 范围。
6. 世界必须由 world_seed 确定性生成。
7. 区块加载顺序不得影响生成结果。
8. 相邻区块地形、河流和道路必须连续。
9. 不得为每个地块创建一个 Node。
10. 未激活 NPC 不得保留场景节点。
11. 存档采用“世界种子 + 区块差异 + 玩家状态 + 全局状态”。
12. NPC 使用活跃、简化、统计三级模拟。

====================
二、强制工程约束
====================

1. 开始编码前：
   - 列出项目目录。
   - 阅读 project.godot、README、现有脚本和测试。
   - 检测可用的 Godot 命令及版本。
   - 汇报现状和本阶段计划。

2. 修改规则：
   - 保留已有用户代码和行为。
   - 不执行破坏性 Git 操作。
   - 不删除不理解的文件。
   - 不引入第三方插件，除非用户明确允许。
   - 优先使用 Godot 内置 API。
   - 所有核心脚本尽量使用静态类型。
   - 复杂逻辑必须有简短注释，但不要为显而易见代码添加噪声注释。

3. 架构规则：
   - 数据、生成、显示、持久化必须分层。
   - ChunkGenerator 只返回纯数据，不能操作 TileMapLayer。
   - NPCData 是权威状态，NPCView 只是表现。
   - SaveManager 不能依赖具体 UI。
   - 后台线程不能修改场景树。
   - 所有持久实体使用稳定整数 ID，不能保存 NodePath 或运行时实例 ID。

4. 每完成一个阶段：
   - 运行项目解析检查。
   - 运行与该阶段相关的自动测试。
   - 修复所有由本阶段引入的错误。
   - 汇报修改文件、测试命令、测试结果和剩余风险。
   - 只有阶段验收通过后才能进入下一阶段。

5. 如果项目没有测试框架：
   - 创建最小测试场景和 GDScript 测试运行器。
   - 测试失败时返回非零退出状态，或打印明确的 TEST_FAILED 标记。
   - 确定性、坐标、区块边界、存档往返和 NPC 调度必须有自动测试。

====================
三、基础参数
====================

创建统一常量定义：

WORLD_SIZE_CELLS = Vector2i(8000, 8000)
CHUNK_SIZE = 64
ACTIVE_RADIUS = 2
PRELOAD_RADIUS = 3
MACRO_CELL_SIZE = 32

实现并测试：

world_to_chunk(world_cell)
world_to_local(world_cell)
local_to_index(local_cell)
index_to_local(index)

必须正确处理 0、边界值和负坐标。

====================
四、按顺序实施
====================

阶段 0：项目骨架
- 创建主场景、世界场景、核心目录。
- 创建 Constants、EventBus、GameClock、GameState。
- 创建测试入口和基础 README。
- 验收：项目能启动，脚本无解析错误，测试入口可运行。

阶段 1：ASCII 渲染
- 创建 TileId、TileDef、GlyphCatalog。
- 使用 TileMapLayer 或等效批量绘制方案显示字符。
- 创建固定测试区块。
- 创建玩家字符、相机、网格移动、逻辑碰撞和 HUD。
- 禁止每格一个 Node。
- 验收：玩家能移动，墙壁阻挡，字符清晰。

阶段 2：确定性区块生成
- 创建 SeedUtil、ChunkData、ChunkGenerator。
- 使用 FastNoiseLite 和世界坐标采样。
- 生成海拔、温度、湿度、生物群系和基础地块。
- 不得为每个区块重新建立不连续的局部噪声坐标。
- 添加同种子一致性、加载顺序独立和边界连续测试。

阶段 3：宏观世界
- 创建约 250 × 250 的 MacroWorldData。
- 依次生成海陆、温湿度、生物群系、河流、聚落、道路和兴趣点。
- 河流从高地沿低势流动到海洋、湖泊或已有河流。
- 聚落按淡水、平坦度、资源和间距评分。
- 道路图先保证主要聚落连通，再增加少量冗余连接。
- 将宏观特征稳定投影到区块。

阶段 4：区块流式加载
- 创建 WorldManager、ChunkRepository、ChunkStreamer、ChunkView。
- 实现 UNLOADED、QUEUED、GENERATING、READY、ACTIVE、INACTIVE 生命周期。
- 使用距离优先队列。
- 防止重复请求。
- 使用 session_id 和 request_id 拒绝过期结果。
- 限制每帧应用和销毁区块数量。
- 加入快速传送压力测试。

阶段 5：视野和寻路
- 创建当前可见和永久探索状态。
- 实现网格视野算法。
- 实现本地区域 A*。
- 加入地形移动代价和动态阻挡。
- 不允许在 8000 × 8000 全图逐格搜索。

阶段 6：存档
- 创建 SaveManifest、ChunkDelta、SaveManager、SaveMigrations。
- 保存 world_seed、save_version、generator_version、游戏时间和玩家状态。
- 每个区块只保存修改地块、探索位图、移除出生项和持久实体。
- 使用 user://saves/<slot>/。
- 使用 .tmp 写入、.bak 备份和正式文件替换流程。
- 写入成功前不得清除 dirty。
- 读取失败时尝试备份。
- 添加存档往返、损坏恢复和版本检查测试。

阶段 7：基础 NPC
- 创建 NPCData、NPCManager、NPCScheduler、NPCMaterializer。
- 所有 NPC 有稳定整数 ID。
- 实现饥饿、疲劳、危险和工作需求。
- 使用 Utility AI 选择行动。
- 使用状态机执行吃饭、睡觉、闲逛、逃跑。
- 加入行为保持奖励、紧急中断、冷却和目标失效恢复。
- 玩家附近才创建 NPCView。

阶段 8：分层 NPC
- 实现 ACTIVE、COARSE、AGGREGATE 三级模拟。
- ACTIVE 使用局部路径和完整行动。
- COARSE 使用低频更新、预计抵达时间和快速事件结算。
- AGGREGATE 按聚落人口、职业、资源和风险统计更新。
- 重要命名 NPC 保持个体数据。
- 离线推进不能逐秒补算，应根据 elapsed 时间直接计算。

阶段 9：线程和优化
- 仅将纯数据计算提交给 WorkerThreadPool。
- 工作线程返回不可依赖场景树的结果对象。
- 主线程验证 session_id、request_id 和当前需求后再显示。
- 批量处理简化 NPC。
- 增加 FPS、区块数、任务数、NPC 数和每帧预算统计。
- 只有性能分析证明确有瓶颈时才从 TileMapLayer 改为 MultiMesh 或自定义渲染。

阶段 10：整合
- 完成新建世界、移动、探索、NPC、保存、读取和退出流程。
- 执行长时间移动、快速传送、重复存读和多次新建世界测试。
- 编写 README，包含启动方式、按键、目录架构、存档格式和已知限制。

====================
五、关键数据结构
====================

ChunkData 至少包含：
- coord: Vector2i
- tiles: PackedByteArray，固定长度 4096
- flags: PackedByteArray，可选
- generated: bool
- dirty: bool

ChunkDelta 至少包含：
- coord: Vector2i
- modified_indices: PackedInt32Array
- modified_tile_ids: PackedByteArray
- explored_bits: PackedByteArray
- removed_spawn_ids: PackedInt64Array
- persistent_entities: Array[Dictionary]

NPCData 至少包含：
- id
- archetype_id
- world_cell
- settlement_id
- home_id
- workplace_id
- health
- hunger
- fatigue
- morale
- danger
- current_action
- action_target_cell
- action_target_entity_id
- action_finish_time
- next_update_time
- last_update_time
- simulation_level
- persistent

====================
六、必须通过的自动测试
====================

1. 坐标转换：
   (0,0)、(63,63)、(64,64)、(-1,-1)、(-64,-64)、(-65,-65)。

2. 确定性：
   相同种子和区块坐标生成结果完全一致。

3. 加载顺序：
   A-B-C 与 C-A-B 的区块结果校验值一致。

4. 边界：
   相邻区块的连续地形、河流和道路不存在错位。

5. 存档往返：
   修改地块和实体，保存、清空、读取后数据一致。

6. 损坏恢复：
   正式文件损坏时尝试有效备份，并输出明确日志。

7. NPC 调度：
   NPC 不重复调度，每帧不超过预算，离线时间推进正确。

8. 生命周期：
   玩家远离后 ChunkView 和 NPCView 数量能够下降，不持续泄漏。

====================
七、完成定义
====================

只有同时满足下列条件才能宣布完成：

- 项目可以实际启动。
- 玩家可以在连续程序生成世界中移动。
- 区块可以正确加载、缓存和卸载。
- 地图结果由种子稳定复现。
- 至少存在河流、道路、聚落或兴趣点中的三种。
- 至少一种 NPC 能吃饭、休息、移动和逃跑。
- NPC 在不同距离使用不同模拟精度。
- 玩家位置、游戏时间、地块变化和重要 NPC 可以保存并读取。
- 自动测试全部通过。
- README 清楚说明运行方法、设计和限制。

不要只输出代码片段或设计建议。你必须直接检查并修改工作区文件、运行测试、修复问题，直到当前阶段通过。如果由于缺少 Godot 可执行文件等外部条件无法运行，仍需完成可静态检查的实现，并明确列出未能执行的命令、原因和用户需要采取的操作。
```

---

# 19. 分阶段 Agent Prompt

如果不希望 Agent 一次承担整个项目，推荐按下面顺序逐条发送。每次只发送一个阶段，验收通过后再发送下一条。

## Prompt 0：项目审计与骨架

```text
检查当前 Godot 项目，确认 Godot 版本、目录结构、现有代码、现有测试和用户未提交改动。不要覆盖或回退已有修改。

然后完成“项目骨架”：
1. 建立 core、data、generation、world、rendering、actors、navigation、npc、save、debug、tests 目录。
2. 建立可启动的 main.tscn 和 world.tscn。
3. 创建统一 Constants，包含：
   WORLD_SIZE_CELLS = Vector2i(8000, 8000)
   CHUNK_SIZE = 64
   ACTIVE_RADIUS = 2
   PRELOAD_RADIUS = 3
   MACRO_CELL_SIZE = 32
4. 创建 EventBus、GameClock、GameState。
5. 创建最小自动测试运行器。
6. 实现并测试 world_to_chunk、world_to_local、local_to_index、index_to_local，必须覆盖负坐标。
7. 更新 README，说明启动和测试命令。

完成后运行项目解析检查和测试。汇报：
- 修改文件
- 关键设计
- 执行的命令
- 测试结果
- 尚未解决的问题

只有所有阶段 0 验收项通过后才结束。
```

## Prompt 1：ASCII 渲染和玩家

```text
在阶段 0 已通过的基础上实现 ASCII 渲染和玩家控制。先阅读现有代码并保持架构一致。

要求：
1. 创建语义化 TileId，不允许把 ASCII 字符作为地形权威数据。
2. 创建 TileDef 和 GlyphCatalog，定义字符、前景色、背景色、walkable、transparent、movement_cost。
3. 使用 TileMapLayer 或等效批量方案显示一个 64×64 测试区块。
4. 禁止每个地块创建一个 Node。
5. 创建玩家场景，以字符显示。
6. 实现四方向网格移动。
7. 碰撞通过 TileDef.walkable 判断，不给每格创建物理节点。
8. 创建相机和基础 HUD，显示 FPS、世界坐标和区块坐标。
9. 保证字符图像清晰，无线性过滤模糊。
10. 添加渲染数据映射和移动碰撞测试。

运行测试并实际启动场景检查。阶段验收：
- 测试地图正确显示。
- 玩家能移动。
- 玩家不能进入墙壁。
- 场景树中没有数千个地块节点。

汇报修改文件、测试命令和结果。
```

## Prompt 2：确定性地形生成

```text
实现确定性区块生成，不要开始流式加载。

要求：
1. 创建 SeedUtil，世界种子、区块坐标和生成层产生稳定种子。
2. 创建 ChunkData，tiles 使用固定长度 4096 的 PackedByteArray。
3. 创建 ChunkGenerator。
4. 使用 FastNoiseLite 生成海拔、温度和湿度。
5. 所有噪声使用世界地块坐标采样，禁止从每个区块的局部 0,0 单独开始。
6. 通过海拔、温度和湿度规则生成海洋、海滩、平原、森林、沙漠、沼泽、山地和雪地。
7. 渲染器改为显示生成区块。
8. 增加：
   - 同种子一致性测试
   - 不同种子差异测试
   - 区块加载顺序独立测试
   - 相邻区块边界连续性测试
9. 输出区块校验值便于调试。

只有测试全部通过才结束。不得依赖全局随机调用顺序。
```

## Prompt 3：宏观世界、河流、聚落和道路

```text
在现有确定性区块生成基础上，实现宏观世界层。

要求：
1. 创建约 250×250 的 MacroWorldData，每格对应 32×32 世界地块。
2. 宏观数据包含海拔、温度、湿度、生物群系、水流、聚落和道路信息。
3. 新建世界时确定性生成宏观地图并缓存。
4. 河流：
   - 从高海拔和高降水候选点产生。
   - 沿最低邻格流动。
   - 汇入海洋、湖泊或已有河流。
   - 防止无限循环。
5. 聚落：
   - 根据淡水、平坦度、耕地、资源和间距评分。
   - 每个聚落拥有稳定 ID。
   - 不得生成在深水等非法区域。
6. 道路：
   - 构建聚落图。
   - 保证主要聚落连通。
   - 按地形代价生成路线。
7. 将河流、道路和聚落稳定投影到地块级区块。
8. 保证跨区块连续，不能因加载顺序重复生成或错位。
9. 添加宏观确定性、河流终点、聚落合法性和道路连通测试。

运行测试并提供至少一个固定种子的调试输出摘要。
```

## Prompt 4：区块流式加载

```text
实现区块流式加载。保持生成器是纯数据层，不得让生成器直接操作 TileMapLayer。

要求：
1. 创建 WorldManager、ChunkRepository、ChunkStreamer、ChunkView。
2. 实现区块状态：
   UNLOADED、QUEUED、GENERATING、READY、ACTIVE、INACTIVE。
3. 玩家附近 ACTIVE_RADIUS=2，预加载 PRELOAD_RADIUS=3。
4. 请求按与玩家距离排序。
5. 防止重复请求。
6. 建立 active_chunks 和 cached_chunks。
7. 缓存使用有限容量和 LRU 或等效策略。
8. 限制每帧创建、显示和卸载区块数量。
9. 每个请求携带 session_id 和 request_id，拒绝过期结果。
10. 增加传送到远距离坐标的压力测试。
11. HUD 显示活跃区块、缓存区块和排队任务数量。

阶段验收：
- 玩家跨区块移动时无明显空洞。
- 活跃区块数量受限。
- 快速传送不会把旧区块结果显示在新位置。
- 来回跨边界不会无限重复生成。
```

## Prompt 5：视野、探索和局部寻路

```text
实现视野、探索记忆和局部寻路。

要求：
1. TileDef.transparent 控制视野阻挡。
2. 使用适合网格的阴影投射算法计算玩家视野。
3. 区分 visible_now 和 explored。
4. explored 按区块保存为紧凑位图或 PackedByteArray。
5. FogLayer 显示未探索、已探索不可见和当前可见三种状态。
6. 创建 LocalPathfinder，使用 AStarGrid2D 或自有格子 A*。
7. 支持地形 movement_cost。
8. 支持动态占用，但避免每帧重建整个 8000×8000 图。
9. 搜索范围只覆盖局部活跃区域。
10. 添加墙壁遮挡、探索保留、可达和不可达路径测试。

运行测试和实际场景检查后汇报。
```

## Prompt 6：安全的差异存档

```text
实现版本化差异存档。

目录使用：
user://saves/<slot>/
  manifest.dat
  manifest.bak
  player.dat
  global.dat
  settlements.dat
  named_npcs.dat
  chunks/<x>_<y>.dat

要求：
1. Manifest 保存 magic、save_version、generator_version、world_seed、world_size、chunk_size、game_time 和时间戳。
2. ChunkDelta 只保存：
   - modified_indices
   - modified_tile_ids
   - explored_bits
   - removed_spawn_ids
   - persistent_entities
3. 基础生成地形不得完整保存。
4. 实现脏区块集合。
5. 采用 .tmp → 验证 → .bak → 正式文件的安全保存流程。
6. 保存成功前不得清除 dirty。
7. 正式文件损坏时尝试备份并记录错误。
8. 实现 save_version 与 generator_version 检查。
9. 提供 SaveMigrations 入口，即使第一版只支持当前版本。
10. 不允许反序列化任意对象；存档只含受控基础类型和数组。
11. 实现玩家位置、游戏时间和世界种子恢复。
12. 添加：
    - 地块修改往返测试
    - 探索状态往返测试
    - 玩家状态往返测试
    - 损坏正式文件恢复测试
    - 不兼容版本错误测试

确保测试不会污染正式用户存档，使用独立测试槽。
```

## Prompt 7：基础 NPC、Utility AI 和状态机

```text
实现基础 NPC 系统。NPCData 必须是权威状态，NPCView 只能作为附近表现。

要求：
1. 创建稳定整数 ID 分配器。
2. 创建 NPCData、NPCManager、NPCScheduler、NPCMaterializer。
3. NPCData 至少包含：
   id、archetype_id、world_cell、settlement_id、home_id、workplace_id、
   health、hunger、fatigue、morale、danger、
   current_action、action_target_cell、action_target_entity_id、
   action_finish_time、next_update_time、last_update_time、
   simulation_level、persistent。
4. 创建 Utility AI，至少支持：
   EAT、SLEEP、WANDER、WORK、FLEE。
5. 评分必须支持：
   - 最低阈值
   - 当前行为保持奖励
   - 紧急行为中断
   - 冷却
   - 无效目标返回 0
6. 创建 ActionExecutor 状态机。
7. 行为必须处理目标消失、路径失败和区块卸载。
8. 玩家附近创建 NPCView，离开后写回数据并释放视图。
9. 不允许用 Node 实例 ID 作为存档 ID。
10. 添加 Utility 选择、行为稳定、目标失败恢复、实体化往返测试。

在测试地图中生成少量 NPC，让它们可观察地执行吃饭、休息、移动和逃跑。
```

## Prompt 8：三级 NPC 模拟与聚落统计

```text
在基础 NPC 系统上实现多精度模拟。

要求：
1. 定义 ACTIVE、COARSE、AGGREGATE。
2. ACTIVE：
   - 有 NPCView
   - 使用局部寻路
   - 高频更新
3. COARSE：
   - 无 NPCView
   - 使用预计到达时间
   - 低频更新
   - 战斗和工作快速结算
4. AGGREGATE：
   - 普通人口合并到 SettlementData
   - 按游戏小时或天更新粮食、职业、人口、治安和风险
   - 重要命名 NPC 仍保留个体数据
5. 创建 RegionGraph 和 RouteService。
6. 远距离路径只能使用区域、道路和聚落图，不得执行全图逐格 A*。
7. 离线更新使用 elapsed 时间直接计算，不得逐秒回放。
8. 随机离线事件使用 world_seed、entity_id、时间窗口和事件类型生成稳定随机结果。
9. NPCScheduler 使用 next_update_time 和每帧预算，不得每帧扫描全部 NPC。
10. 添加调度不重复、预算限制、远距离抵达、离线推进和存档恢复测试。
```

## Prompt 9：后台生成与性能

```text
在所有功能测试通过后进行后台任务和性能优化。不要在测量前重写渲染系统。

要求：
1. 记录优化前的区块生成、渲染和 NPC 更新时间。
2. 将区块纯数据生成提交给 WorkerThreadPool。
3. 工作线程不得访问活动场景树或修改 TileMapLayer。
4. 结果必须携带 session_id、request_id、chunk_coord。
5. 主线程应用前验证任务仍属于当前世界且区块仍需要。
6. 限制每帧最多应用的生成结果。
7. 将 COARSE NPC 以批次处理。
8. HUD 增加：
   FPS、活跃区块、缓存区块、排队任务、脏区块、
   ACTIVE NPC、COARSE NPC、本帧 NPC 更新数、
   区块生成耗时、区块应用耗时。
9. 测试连续移动、边界往返、快速传送、多次新建世界和存读档。
10. 只有分析证明 TileMapLayer 是主要瓶颈时，才设计并实现 MultiMesh 或自定义绘制替代；替换时必须保持渲染接口不变。

修复所有竞态、过期结果和资源泄漏问题，并汇报优化前后数据。
```

## Prompt 10：最终整合与发布检查

```text
对整个项目进行最终整合，不新增无关功能。

必须完成：
1. 从空存档创建新世界。
2. 选择或输入世界种子。
3. 玩家出生在合法可行走陆地。
4. 玩家可以移动、探索和跨区块。
5. 世界出现连续生物群系，以及河流、道路、聚落或兴趣点中的至少三种。
6. NPC 可以吃饭、休息、工作/闲逛和逃跑。
7. NPC 根据距离切换模拟精度。
8. 玩家位置、游戏时间、探索、地块修改和重要 NPC 能保存及读取。
9. 运行全部自动测试。
10. 执行长时间移动、快速传送、重复存读和多次新建世界压力测试。
11. 检查场景节点、区块缓存、任务队列和 NPCView 不会持续增长。
12. 更新 README：
    - Godot 版本
    - 启动方式
    - 测试方式
    - 控制键
    - 系统架构
    - 存档目录
    - save_version
    - generator_version
    - 已知限制

完成后提供最终报告：
- 功能清单
- 修改文件
- 测试命令和全部结果
- 性能观测
- 未完成项
- 已知风险
- 后续建议

除非所有完成定义都满足，否则不要声称项目已经完成。
```

---

# 20. 最终完成清单

## 世界

- [ ] 8000 × 8000 逻辑边界。
- [ ] 64 × 64 区块。
- [ ] 确定性世界种子。
- [ ] 生物群系。
- [ ] 河流。
- [ ] 聚落。
- [ ] 道路。
- [ ] 兴趣点。

## 渲染

- [ ] ASCII 字符图集。
- [ ] 地形与表现分离。
- [ ] 无每格 Node。
- [ ] 玩家、NPC、雾层正确叠加。

## 流式加载

- [ ] 活跃范围。
- [ ] 预加载范围。
- [ ] 任务去重。
- [ ] 过期任务拒绝。
- [ ] 数据缓存。
- [ ] 有限的每帧应用预算。

## 游戏逻辑

- [ ] 网格移动。
- [ ] 逻辑碰撞。
- [ ] 视野。
- [ ] 探索记忆。
- [ ] 局部寻路。
- [ ] 全局区域路径。

## NPC

- [ ] 稳定 ID。
- [ ] NPCData 权威状态。
- [ ] Utility AI。
- [ ] 行为状态机。
- [ ] 活跃模拟。
- [ ] 简化模拟。
- [ ] 聚落统计模拟。
- [ ] 离线时间推进。

## 存档

- [ ] Manifest。
- [ ] 玩家状态。
- [ ] 全局状态。
- [ ] 区块差异。
- [ ] 探索状态。
- [ ] 重要 NPC。
- [ ] 临时文件。
- [ ] 备份恢复。
- [ ] 版本检查。
- [ ] 自动保存。

## 质量

- [ ] 坐标测试。
- [ ] 确定性测试。
- [ ] 区块边界测试。
- [ ] 存档往返测试。
- [ ] NPC 调度测试。
- [ ] 快速传送测试。
- [ ] 长时间运行测试。
- [ ] README。

---

## 推荐实施结论

不要从“完整大世界”同时开工。正确顺序是：

```text
稳定的网格与数据模型
→ ASCII 渲染
→ 确定性单区块
→ 宏观世界
→ 区块流式加载
→ 视野和寻路
→ 差异存档
→ 基础 NPC
→ 分层 NPC
→ 多线程和性能优化
→ 最终整合
```

只要每一个阶段都通过明确的自动测试和可玩验收，项目就不会在后期因为世界规模、存档格式或 NPC 数量而被迫整体重写。
