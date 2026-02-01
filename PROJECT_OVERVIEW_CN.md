# 项目结构与架构总览（详细版）

> 目标：**读完这份文档≈读完一遍项目代码**。它不仅告诉你“文件在哪”，更会告诉你“为什么这么写、数据怎么流、扩展怎么做、哪里最容易踩坑”。

## 0. 快速心智模型（先把整套东西装进脑子里）

### 0.1 这是什么项目？

这是一个“AI Native”的回合制叙事游戏引擎：

- **规则与状态**：完全由代码（ECS + Systems）决定，确定性更新世界状态。
- **AI 的职责**：只做三件事：
  1. 把玩家自然语言解析成引擎可执行的 **JSON 指令**（`Command`）
  2. 给 NPC（以“小队协同”的方式）生成本回合 **指令计划**（`Command`）
  3. 把本回合发生的 **事件** 翻译成有代入感的 **旁白文本**（叙事渲染）

> 关键边界：AI **不直接改世界**；它只能给出 “想做什么”（指令/文本），能不能做、做完发生什么，全由引擎规则裁决并产出事件。

### 0.2 一句话看懂主链路

```
assets(JSON) -> GameLoader -> World(Entity+Component+Scene)
玩家输入 -> InputAI(parse) -> CommandsQueue
CommandsQueue -> Action -> Systems(update) -> Events
Events -> Facts -> OutputAI(旁白) -> UI显示
Events -> MemoryAI -> NPC记忆写回
```

### 0.3 术语表（读代码必备）

- **Entity**：实体 ID（UUID），纯标识；真正的数据在组件里。
- **Component**：挂在实体上的数据块（Pydantic Model），例如生命/位置/背包/门锁等。
- **System**：规则执行器。读取 `World.get_actions()`，匹配 `Command`，改组件、产事件。
- **Command**：指令（不可变），例如 `MOVE/ATTACK/SEARCH/...`；由玩家或 AI 生成。
- **Action**：指令被“出队并准备执行”的记录，包含执行者与（可选）执行时场景 `scene_id`。
- **Event**：本回合真实发生的事（或失败原因），是“旁白唯一事实来源”。
- **Scene**：场景（地图节点），包含出口关系与持续伤害。
- **AP（Action Point）**：行动点预算；每个实体一回合能执行的指令受 AP 限制。
- **短 ID（E1/E2...）**：给 AI 用的临时 ID，避免把 UUID 暴露给模型。

## 1. 顶层目录（仓库结构）

- `assets/`：游戏内容数据（数据驱动的“原材料”）
  - `global.json`：标题、开场白、玩家开局配置
  - `archetypes/*.json`：实体原型（NPC/怪物/物品/门/可搜索物等）
  - `scenes/*.json`：场景与场景内实体实例
  - `instances.json`：跨场景/可引用的全局实例清单（稳定字符串 ID）
- `src/`：引擎代码（“工厂”）
  - `src/engine/main.py`：启动调试前端的入口（Tkinter）
- `docs/`：仓库内文档
  - `docs/PROJECT_OVERVIEW.md`：你正在看的这份
  - `docs/NEXT_STEPS.md`：下一步改进建议（优先级清单）
- `requirements.txt`：依赖
- `README.md` / `Proposal.md`：路线图与数据驱动策划说明

## 2. 引擎分层总览（src/engine）

从“架构风格”上看，这里近似是 **Ports & Adapters（六边形）**：

- `ports/`：抽象协议（AI / UI）
- `adapters/`：具体实现（各家模型、GUI）
- `model/`：领域模型（ECS）
- `runtime/`：编排（回合驱动、渲染）
- `content/`：把数据资产装配成 World
- `ai/`：提示词+解析+编排（AI 侧的胶水层）

下面开始按“读代码”的粒度逐文件梳理。

## 3. `model/`（ECS 核心：数据与规则）

### 3.1 `src/engine/model/entity.py`

- `Entity = UUID`
  - 这里非常纯粹：实体就是一个 UUID（值对象）。
  - 任何“名字/血量/位置”等都不在 Entity 上，而在组件里。

### 3.2 `src/engine/model/components.py`

组件定义（Pydantic `BaseModel`）。这里是“世界状态”的核心结构。

**常用组件一览（按用途分组）：**

- 标识/叙事：
  - `IntroComponent(name, description)`：实体对外展示的名字和描述（也用于 `EntitySnapshot`）
- 空间：
  - `LocationComponent(scene_id)`：实体所在场景
- 生存：
  - `HealthComponent(current, max)`：生命值
  - `StatusComponent(statuses=[StatusEntry])`：状态列表（带 duration）
- 经济/容器：
  - `InventoryComponent(items, is_open=False)`：背包/容器；`is_open` 用于“搜索后可见/可拿取”规则
- 行为：
  - `CommandsQueueComponent(commands: Deque[Command])`：行为队列
  - `ActionPointComponent(current_point, max_point)`：AP（回合末重置）
  - `AttackableComponent(damage)`：基础攻击力（同时“武器物品”也可以带它）
- 角色标记：
  - `PlayerComponent(game_history)`：玩家标记（目前主要用于区分玩家/非玩家）
  - `NPCComponent(memory, character, mission, limit)`：NPC 的“本能与记忆”
  - `MonsterComponent()`：怪物标记
  - `SquadMemberComponent()`：小队成员标记（用于小队协同）
- 交互/机关：
  - `DoorLockerComponent(is_locked, locked_direction, key_entity)`：锁门组件（key_entity 是一个实体引用）
  - `SearchableComponent(texts)`：可搜索的“未来文本”（注意：没 SearchEvent 不能当事实）
  - `ChallengeComponent(challenges)`：检定规则集合（如 `UNLOCK_AGI`）
  - `PreventFromExitComponent(escape_rate, reason)`：阻止离开（概率）
- 掉落/任务：
  - `DropOnDeathComponent(items)`：死亡后额外掉落物品（实体引用）
  - `QuestItemComponent(tag)`：任务关键物品标记（胜利条件用）

> 读系统时的关键点：很多规则依赖“组件是否存在”，所以组件既是数据，也是“能力标签”。

### 3.3 `src/engine/model/commands.py`

指令是不可变对象（继承 `ImmutableModel`），用于描述“意图”。

- `Command(cost=1)`：基类，有默认 cost
- 主要指令：
  - `MoveCommand(direction, cost=2)`
  - `AttackCommand(target, cost=2)`
  - `PerformCheckCommand(target, intent, attribute, cost=2)`
  - `SearchCommand(target, cost=1)`
  - `UnlockCommand(target, key, cost=1)`
  - `GetCommand(target, cost=1)`
  - `UseCommand(item, target, cost=1)`
  - `TalkCommand(target, text, cost=0)`（target 可以是实体、`"scene"`、`"fluff"`）

此外还有：

- `Action(executor, command, scene_id=None)`：不是“命令”，而是“执行记录”。`scene_id` 用于叙事一致性（例如先移动后说话时，仍能把对话归属到说话时所在场景）。

### 3.4 `src/engine/model/event.py`

事件是“真实发生的事”。旁白生成时，只能相信事件。

核心结构：

- `EntitySnapshot(id, intro)`：把实体与 `IntroComponent` 快照化
  - 目的：即使实体之后被删除（例如死亡删除），事件里仍能安全引用它的名字与描述。
- `Event(executor)`：基类，executor 是 `EntitySnapshot` 或 `Scene`

常见事件：

- 行动意图（用于日志/事实完整性）：`ActionIntentEvent(command, target/item/...可选)`
- 行动成功：`MoveEvent / UnlockEvent / GetEvent / ItemUseEvent / SearchEvent / DamageEvent / CheckEvent / DialogueEvent`
- 行动失败：`ActionFailEvent(texts)`（硬失败，带原因）
- 情景不成立：`ActionContextFailEvent(texts)`（软失败，常用于“fluff”）
- 死亡：`EntityDiedEvent(scene_id)`（由 HealthSystem 在检测到血量<=0 后发）
- 结局：`GameOverEvent(reason, winner)`

> 你看到的“回合事实”文本，就是把这些 Event 变成可读句子（见 `ai/facts.py`）。

### 3.5 `src/engine/model/scene.py`

- `Direction(Enum)`：`north/south/east/west`（注意 value 是小写）
- `Scene(id, name, description, exits, damage)`：
  - `exits: Dict[Direction, str]`：方向 -> 目标 scene_id
  - `damage`：每回合对场景内有生命实体造成持续伤害（由 `SceneSystem` 执行）

### 3.6 `src/engine/model/world.py`

`World` 是 ECS 的容器与执行上下文：

- 实体存储：`_entities: Dict[Entity, Dict[Type[Component], Component]]`
- 场景存储：`_scenes: Dict[str, Scene]`
- 队列：
  - `_events_queue: List[Event]`
  - `_actions_queue: List[Action]`
- 系统：
  - `_action_systems: List[System]`
  - `_environment_systems: List[System]`
- 叙事/AI 相关字段：
  - `history: List[str]`：回合转写（混合玩家可见与不可见事件的串）
  - `last_player_input_raw: str`：玩家原始输入（仅供旁白文笔参考，不等价事实）
  - `title/intro_text`
  - `_short_id_map/_short_id_counter`：短 ID 映射（给 AI 用）

核心方法：

- `create_entity()`：生成 UUID 并初始化组件字典
- `add_component()/get_component()/get_entities_with()`：ECS 查询接口
- `publish_action()/get_actions()/clear_actions()`
- `publish_event()/get_events()/clear_events()`
- `update_actions()`：遍历 `_action_systems` 执行
- `update_environment()`：遍历 `_environment_systems` 执行

> 注意：组件虽然是 Pydantic Model，但系统里会直接修改其字段（例如 `health.current -= damage`）。因此“不可变”主要用于 Command/Event，组件仍是可变状态。

### 3.7 `src/engine/model/systems.py`（规则全集）

这里是“引擎玩法”的真实来源：AI 只能生成指令，**能不能做、做完如何变化**全在系统里。

#### 3.7.1 `InteractionSystem`

处理：`SEARCH / UNLOCK / GET / USE`

- `SEARCH`：
  - 目标需要 `SearchableComponent`
  - 若目标有 `InventoryComponent`：把 `is_open=True`（容器被打开）
  - 事件：发布 `SearchEvent(texts=...)`（文本会列出容器里物品名）
- `UNLOCK`：
  - 目标需要 `DoorLockerComponent`
  - 执行者需要 `InventoryComponent` 且持有正确钥匙（door.key_entity）
  - 成功：发布 `UnlockEvent` 并把 `is_locked=False`
  - 失败：发布 `ActionFailEvent("You don't have the key.")` 等
- `GET`（三种路径）：
  1. 目标本身是容器（有 `InventoryComponent`）：必须 `is_open=True` 才能把里面物品转移到执行者背包
  2. 目标是“打开的容器内的某个物品”：从容器移除该 item，放入执行者背包
  3. 目标是场景内可拾取物：需要与执行者同场景；拾取后移除其 `LocationComponent`
  - 成功事件：`GetEvent(entities=[...])`
- `USE`：
  - 目前仅支持 `ConsumableComponent(effect, amount)`
  - 成功：发布 `ItemUseEvent`，并删除被使用的物品实体（`world.delete_entity(use.item)`）

#### 3.7.2 `CheckSystem`

处理：`PERFORM_CHECK`

- 从目标的 `ChallengeComponent.challenges` 中按 `"{intent}_{attribute}"` 查规则
- 掷骰：d20 + 属性值（来自执行者的 `AttributeComponent`，默认 10）
- 规则效果：`destroy/unlock/damage/add_status/modify_relation`
  - `damage` 会发 `DamageEvent` 并扣血
  - `add_status` 往 `StatusComponent` 增加或刷新 `StatusEntry(name, duration)`
- 事件：始终发布 `CheckEvent(...)`，包含 roll、dc、success、effect

#### 3.7.3 `MovementSystem`

处理：`MOVE`

- 依据当前场景 `Scene.exits` 找目标场景
- 若出口方向存在：
  - 检查同场景内所有 `DoorLockerComponent`：
    - 如果某门 `locked_direction == move.direction` 且 `is_locked=True`：动作失败（`ActionFailEvent`）
  - 检查 `PreventFromExitComponent`：按 escape_rate 与随机数决定是否阻止
  - 若无阻碍：发布 `MoveEvent(scene_from, scene_to)` 并更新 `LocationComponent.scene_id`
- 若出口不存在：发布 `ActionFailEvent("你无法移动，因为这个方向没有出口。")`

> 你提到的“locked”失败文案，就是在这里产出。

#### 3.7.4 `CombatSystem`

处理：`ATTACK`

- 目标必须有 `HealthComponent` 且在场景内
- 伤害来源：执行者 `AttackableComponent.damage` + 执行者背包内所有带 `AttackableComponent` 的物品之和
- 成功：扣血并发布 `DamageEvent`

#### 3.7.5 `SceneSystem`

环境系统：场景持续伤害

- 对每个 `scene.damage > 0` 的场景：对场景内的 NPC 与玩家发布 `DamageEvent` 并扣血

#### 3.7.6 `StatusSystem`

环境系统：状态回合衰减

- 每回合把所有 `StatusEntry.duration -= 1`，移除 duration<=0 的状态

#### 3.7.7 `HealthSystem`

环境系统：死亡与尸体生成

- 监听 `DamageEvent` / `ItemUseEvent` 后的血量：
  - `current <= 0`：触发死亡流程 `_handle_death()`
  - `current > max`：截断回 `max`
- 死亡流程：
  - 非玩家、且是 NPC/怪物：生成“尸体实体”（带 `SearchableComponent` 与 `InventoryComponent(items=掉落)`），并删除原实体
  - 发布 `EntityDiedEvent(scene_id=...)`

#### 3.7.8 `DialogSystem`

处理：`TALK`

- `target="fluff"`：发布 `ActionContextFailEvent`（软失败，用于“原地转圈”等表演动作）
- `target="scene"`：发布 `DialogueEvent(receiver=Scene, at_scene_id=...)`
- `target=Entity`：发布 `DialogueEvent(receiver=EntitySnapshot, at_scene_id=...)`

## 4. `content/`（把 assets 装配成 World）

### 4.1 `src/engine/content/registry.py`

组件注册表：字符串 -> 组件类（例如 `"HealthComponent" -> HealthComponent`）。

- 资产里 `components` 的 key 必须在这里注册，否则加载时会 `print` 警告并跳过该组件。

### 4.2 `src/engine/content/loader.py`

`GameLoader` 是“数据驱动架构”的落地点：把 JSON 组装成 ECS 世界。

#### 4.2.1 GameLoader 的内部状态

- `assets_path: Path`
- `archetypes: dict[str, dict]`：全部原型缓存（聚合自 `assets/archetypes/*.json`）
- `instance_map: dict[str, Entity]`：稳定实例 ID -> 真实实体 UUID（用于引用解析）
- `pending_refs: list[(Entity, comp_name, comp_args)]`：第一阶段先记下含实体引用的组件，第二阶段再解析
- `global_config: dict`

#### 4.2.2 加载主流程（load_all）

按顺序：

1. `_load_global(world)`：读 `assets/global.json`，写入 `world.title/world.intro_text`
2. `_load_all_archetypes()`：读 `assets/archetypes/*.json` 合并到 `self.archetypes`
3. `_build_scenes_first_pass(world, "scenes/chapter1.json")`
   - 第一遍：创建 `Scene` 并 `world.add_scene()`
   - 第二遍：按 `entities` 列表创建实体并放入 `scene_id`
4. `_build_instances_first_pass(world, "instances.json")`：创建全局实例（如果文件存在）
5. `_build_player(world)`：根据 `global.json.player` 创建玩家实体并设置出生点
6. `_resolve_pending_refs(world)`：解析实体引用字段（如背包 items / key_entity）
7. `_ensure_player_components(world, player)`：校验玩家必要组件齐全
8. `_register_systems(world)`：注册动作系统与环境系统

返回：`(world, player)`

#### 4.2.3 Archetype 合成与覆写规则

创建实体时会：

- 从 archetype 的 `components` 作为 base
- 应用 `remove_components`：从 base 移除指定组件名
- 应用 `override_components`：对同名组件参数做深合并（dict 递归合并）

这使得“同一原型复用 + 少量差异”成为常态。

#### 4.2.4 实体引用解析（二阶段）

目前被视为“含实体引用”的组件（`_has_entity_refs`）：

- `InventoryComponent`：`items: [<instance_id>, ...]`
- `DoorLockerComponent`：`key_entity: "<instance_id>"`
- `DropOnDeathComponent`：`items: [<instance_id>, ...]`

解析策略：

- 第一阶段：先把这些组件加入 `pending_refs`，不立刻实例化（因为引用对象可能还没创建）
- 第二阶段：把字符串实例 ID 用 `instance_map` 解析为真实 UUID，再实例化组件并挂回实体

> 扩展提示：如果你新增一个“引用实体”的组件，一定要把它加进 `_has_entity_refs()` 和 `_resolve_entity_refs()`。

#### 4.2.5 参数规范化（_normalize_args）

加载器会对部分组件做特殊处理：

- `CommandsQueueComponent.commands`：list -> `deque`
- `DoorLockerComponent.locked_direction`：字符串 `"NORTH"` -> `Direction.NORTH`

## 5. `ai/`（AI 侧：提示词、上下文、解析、编排）

### 5.1 `src/engine/ai/context.py`（给 AI 的“可见世界”）

核心目标：把 UUID 世界转换成 “短 ID + 可用实体列表”，让模型稳定输出。

关键点：

- `_get_short_id(world, entity)`：
  - 在 world 上维护 `_short_id_map/_short_id_counter`
  - 用 base62 编码生成 `E1/E2/...`
- `_collect_open_inventory_items()`：
  - 递归遍历“已打开的容器”内物品，保证 “容器打开后里面物品也能出现在上下文里”
- `build_input_context(world, player_id)`：用于玩家输入解析（InputAI）
  - 实体集合 = 同场景实体 + 玩家背包 + 打开的容器里物品
  - 产出：`context_string`（给模型看）+ `short_id_map`（短 ID -> UUID 字符串）
- `build_squad_context(world, squad, scene_id)`：用于小队协同（NpcAI）
  - 产出：场景内实体列表 + 小队成员卡片（含 ap、inventory、character/mission/limit/memory）

### 5.2 `src/engine/ai/prompts.py`（所有提示词模板）

这里是“模型被如何约束”的源头。重要模板：

- `COMMAND_MANUAL`：允许的 JSON 指令格式与语义
- `build_player_parse_prompt(...)`：玩家自然语言 -> JSON 指令数组（必须严格 JSON，无 markdown）
- `build_squad_prompt(...)`：按场景对 NPC 小队做协同计划（JSON 数组：[{executor_id, commands}...]）
  - 这里的规则会直接决定 NPC 是否“理玩家”、是否会在同回合插入 `TALK` 等。
- `build_batch_memory_prompt(cards_json)`：批量 NPC 记忆更新（JSON 对象：{executor_id: "memory"}）
- `build_host_prompt(...)`：旁白（GM）生成（必须忠实于事实，不可脑补）
- `build_game_over_prompt(...)`：结局旁白
- `build_repair_prompt(...)`：解析失败时的纠错重试提示词

### 5.3 `src/engine/ai/parsers.py`（AI 输出 -> Command）

职责：把模型输出的 JSON 变成可执行的 `Command` 对象，并尽可能“容错但不乱猜”。

- `_strip_code_fence()`：去掉 ``` 包裹（但 prompts 强制要求不要输出 markdown）
- `_resolve_entity_id(raw_id, id_map)`：把短 ID 映射回 UUID 字符串
  - 支持大小写容错（若有唯一匹配）
- `parse_command_dicts(list[dict], id_map)`：逐条解析指令
  - `TALK` 支持 `target_id="scene"|"fluff"|实体ID`
  - `GET_ALL` 兼容老指令
- `parse_json_to_command_result()`：返回 `ParseResult(commands, error, raw_count, parsed_count)`
  - 用于 orchestrator 的“重试/修复/降级”策略

### 5.4 `src/engine/ai/orchestrator.py`（AI 调用编排）

- `parse_player_input(player_text, world, player_id, ai_client)`：
  1. `build_input_context()` 得到短 ID 可用实体列表
  2. 拼 `build_player_parse_prompt()`
  3. 调用 `ai_client.send()`
  4. `parse_json_to_command_result()`
  5. 失败则用 `build_repair_prompt()` 重试（默认最多 2 次）
- `squad_think(world, squad, current_turn_transcript, scene_id, ai_client)`：
  1. `build_squad_context()` 得到成员卡片与短 ID 映射
  2. 拼 `build_squad_prompt()`
  3. 调用模型得到 plans（JSON 数组）
  4. 解析失败则 repair 重试；成功返回 `(plans, id_map)`
- `batch_npc_memory_update(world, npc_entities, events, ai_client)`：
  - 把多个 NPC 的卡片打包成 JSON，要求模型返回 `{short_id: "new memory"}`
  - 然后写回对应 NPCComponent.memory

## 6. `runtime/`（回合编排 + 旁白渲染）

### 6.1 `src/engine/runtime/game_loop.py`（主回合循环）

`run_game_loop(world, player, io, ai_client)` 的核心任务：把“输入、规则执行、AI 协同、旁白输出、记忆更新”串成回合。

**开局：**

- 输出 `world.intro_text`（来自 `assets/global.json`）
- 调用 `formatted_output()` 给出第一段“当前情景旁白”
- UI 更新状态栏（`player_info()`）

**每回合关键步骤（更贴近代码的顺序）：**

1. 玩家输入阶段：如果玩家队列为空
   - UI 开启输入
   - 读取输入，写 `world.last_player_input_raw`
   - `parse_player_input()` -> commands，append 到玩家 `CommandsQueueComponent`
2. 玩家行动阶段：`run_action_phase([player], allowed_executors={player})`
   - 出队指令，按 AP 扣点
   - 对选定 executor 发布 `Action(executor, command, scene_id)`
   - `world.update_actions()` 执行系统，产事件
   - 若出现 `ActionFailEvent`：清空该 executor 的队列（防止持续失败卡死）
3. 组装“本回合转写 turn_transcript”（用于 history 与后续 AI）：
   - 玩家部分：把玩家相关事件转事实（包含 ActionIntentEvent 与同场景可见事件）
4. NPC 协同阶段：按场景分组 `npc_groups[scene_id]`
   - 对每个场景：`squad_think(scene_facts)` 得 plans
   - `apply_squad_plans()`：解析 commands 写入各 NPC 的 CommandsQueueComponent
5. 怪物阶段（非 AI）：
   - 按 AP 选择攻击目标（同场景玩家或 NPC）
   - 给怪物队列写入 `AttackCommand`
6. NPC + 怪物行动阶段：`run_action_phase(npc_entities + monster_entities)`
   - 产出更多事件，追加到 turn_events
7. 环境阶段：`world.update_environment()`
   - 场景持续伤害/状态衰减/死亡处理等
8. 回合事实展示：
   - `build_turn_facts_by_scene(turn_events)`：按场景拼成 UI 的“回合 6/7/…”展示文本
   - `world.history.append(turn_transcript)`
9. 胜负判定：
   - 玩家死亡
   - 任意一方持有 `QuestItemComponent(tag="demon_core")`
   - 若结束：发布 `GameOverEvent` 并 `render_game_over()`
10. NPC 记忆更新：`batch_npc_memory_update()` 写回 `NPCComponent.memory`
11. 旁白输出：`formatted_output()` -> UI narrate；刷新状态栏
12. 回合收尾：
   - 所有有 `ActionPointComponent` 的实体：`current_point = max_point`
   - `world.clear_events()`；UI 刷新 token 显示

> 读主循环时你只要记住：**所有“事实”来自 event_queue**；AI 只参与生成 commands 与输出文本。

**补充：run_action_phase 的执行细节（理解 AP 与事件顺序的关键）**

主循环内部定义了一个 `run_action_phase(...)`，它做的事情可以概括为：

1. 找到所有具有 `CommandsQueueComponent` 与 `ActionPointComponent` 的实体。
2. 在“本 phase”内反复循环：
   - 只要某实体 `ap.current_point >= queue.commands[0].cost` 就允许出队执行；不足则跳过。
   - 出队时会构造 `Action(executor, command, scene_id=executor当前scene)` 并 `world.publish_action(...)`。
   - 当配置了“意图日志”时（只对玩家等特定实体），会在出队瞬间额外发布 `ActionIntentEvent`（用于事实转写里的 `attempt:` 行）。
3. 若某轮有任何 action 入队：
   - `world.update_actions()` 让系统消化 action 并产出 event。
   - 若出现 `ActionFailEvent`：会把该 executor 的 `CommandsQueueComponent.commands` 清空（避免重复失败导致卡死）。
   - `world.clear_actions()` 清空本轮 action 队列，准备下一轮。

因此你在日志里看到的典型顺序是：`attempt`（意图）→（成功/失败事件）→（回合事实展示）。

### 6.2 `src/engine/runtime/renderer.py`（旁白与状态栏）

- `player_info(world, player)`：
  - 汇总玩家位置/血量/背包
  - 列出全部实体及其位置与血量（调试用）
- `formatted_output(world, entity, ai_client)`：
  - 构造“当前情景”文本（场景描述 + 场景内实体组件摘要）
  - 取 `world.history[-1]` 的事实，并格式化为 host prompt 的 `[玩家行动]/[非玩家行动]` 分段
  - 调 `build_host_prompt()`，交给模型生成旁白
  - 失败则返回 `【情景生成失败】...`
- `render_game_over(...)`：
  - 构造结局条件文本 -> `build_game_over_prompt()`
  - 若 AI 失败则回退到固定 `【结局】...`

## 7. `ports/` 与 `adapters/`（抽象与实现）

### 7.1 `src/engine/ports/ai.py`

- `AiClient`：只定义 `send(prompt)->str`
  - 这让 runtime/ai 层不关心具体模型实现

### 7.2 `src/engine/ports/ui.py`

- `GameIO`：主循环需要的 UI 能力协议（叙事输出、读输入、更新状态、token、回合事实、结局）

### 7.3 `src/engine/adapters/ai/router.py`

- `RoutingAiClient(model, app)`：根据命令行参数选择具体模型实现：
  - `gemini / gemini-flash / kimi / kimi-turbo / qwen-7b / qwen-plus / deepseek`

### 7.4 `src/engine/adapters/ai/*.py` 与 `clients.py`

- `clients.py`：初始化各家客户端并从环境变量读取 key（缺失则抛异常）
- 各适配器会：
  - 调用具体 API 得到文本
  - 读取 token 统计并回调 `app.update_token_stats()`（当前直接耦合 GUI）

## 8. `ui/`（Tk 调试前端）

### 8.1 `src/engine/ui/tk_debug.py`

- `DebugGUI`：实现 `GameIO` 协议
  - 创建 Tk 窗口与多个 tab（游戏/日志/回合事实/token）
  - 用线程跑 `run_game_loop()`，用线程安全队列把消息推回 UI 线程
  - `QueueSink`：把 loguru 输出写入 GUI 日志窗口

消息类型（队列里用字符串区分）：

- `NARRATE`：旁白输出
- `LOG`：日志输出（按 level 着色）
- `ENABLE_INPUT`：启用/禁用输入框
- `UPDATE_STATUS`：刷新右侧状态栏
- `UPDATE_TOKEN_DISPLAY`：刷新 token 面板
- `APPEND_TURN_FACTS`：追加回合事实面板
- `GAME_OVER`：展示结局

## 9. assets 数据协议（结合当前内容）

### 9.1 `assets/global.json`

- `meta.title/version/intro_text`：标题与开场白
- `player.default_name/start_scene_id/initial_archetype`：玩家开局设置

### 9.2 `assets/archetypes/*.json`

- `players.json`：`player_base`（玩家基础组件：血量/背包/AP/攻击/属性/状态等）
- `npcs.json`：三名队友 archetype（带 `NPCComponent`、`SquadMemberComponent`）
- `monsters.json`：`wolf_base`、`boss_bear`（怪物基础组件 + 挑战规则 + 掉落）
- `items.json`：钥匙/妖狼皮/妖丹/止血草、以及 `prop_bush`、`door_wood` 等“可交互道具”

### 9.3 `assets/scenes/chapter1.json`

场景数组（目前 3 个场景）：

- `meeting_point`：集合点（NPC 初始位置）
- `forest`：密林（狼/灌木/木门）
- `fortress`：黑风寨（boss + 妖丹掉落）

关键点：

- 场景 `entities` 中的 `id` 是“实例 ID”（字符串）。
- `override_components` 可把实例 ID 引用写进背包/掉落/门钥匙字段：
  - 例如狼的 `InventoryComponent.items=["rusty_key_1"]`
  - 门的 `DoorLockerComponent.key_entity="rusty_key_1"`

### 9.4 `assets/instances.json`

提供全局实例（目前 items 组）：

- `rusty_key_1 / wolf_pelt_1 / herb_1 / demon_core_1`

这些实体会被创建成真实 UUID，并写入 `GameLoader.instance_map`，供引用解析。

### 9.5 一个完整的数据→运行示例（门 + 钥匙 + 掉落）

你可以把“密林的门”这一套机制理解为三段拼接：

1. **先在 `instances.json` 声明稳定实例 ID**（被引用的东西必须有实例 ID）：

```json
{
  "items": [
    { "id": "rusty_key_1", "archetype_id": "item_rusty_key" }
  ]
}
```

2. **在 `scenes/chapter1.json` 放置实体实例，并在 override 中引用实例 ID**：

```json
{
  "id": "forest",
  "entities": [
    {
      "id": "wolf_1",
      "archetype_id": "wolf_base",
      "override_components": {
        "InventoryComponent": { "items": ["rusty_key_1"] }
      }
    },
    {
      "id": "forest_gate",
      "archetype_id": "door_wood",
      "override_components": {
        "DoorLockerComponent": {
          "is_locked": true,
          "locked_direction": "NORTH",
          "key_entity": "rusty_key_1"
        }
      }
    }
  ]
}
```

3. **加载器二阶段把字符串引用解析成真实 UUID**：

- `wolf_1` 的 `InventoryComponent.items=["rusty_key_1"]` 会被替换为 `items=[<UUID_of_rusty_key_1>]`。
- `forest_gate` 的 `DoorLockerComponent.key_entity="rusty_key_1"` 会被替换为 `key_entity=<UUID_of_rusty_key_1>`。
- `locked_direction="NORTH"` 会被解析成 `Direction.NORTH`。

运行时规则如何生效：

- 未解锁时，`MovementSystem` 在尝试 `MOVE NORTH` 时会扫描同场景内的 `DoorLockerComponent`，发现 `locked_direction` 命中且 `is_locked=True`，从而发布 `ActionFailEvent`。
- 拿到钥匙后执行 `UNLOCK`：`InteractionSystem` 检查钥匙是否在执行者背包中、是否匹配门的 `key_entity`；成功则把 `is_locked=False` 并发布 `UnlockEvent`。
- 解锁后再次 `MOVE NORTH` 才会发布 `MoveEvent` 并真正改变 `LocationComponent.scene_id`。

### 9.6 一回合“转写 history”与“展示 facts”的区别（非常重要）

你会看到两种“文本事实”：

1. **`turn_transcript`（写入 `world.history`）**
   - 目标：给 AI 用的上下文（尤其是旁白与后续解析）。
   - 内容：通常是“玩家可见事件”+意图（`attempt:`）拼出来的一段文本，可能跨场景过滤。
2. **`turn_facts_display`（UI 的 `--- 回合 N ---` 展示）**
   - 目标：给人看的回合复盘。
   - 内容：按场景分组的事件汇总（`【场景：...】` 分段）。

旁白生成用的是 `world.history[-1]`（而不是 UI 展示那份），并在 `facts._format_turn_facts_for_host()` 里被拆成 `[玩家行动]` 与 `[非玩家行动]` 两段，避免空间叙事错乱。

## 10. 常见排障（读完就能自己定位）

### 10.0 症状→定位速查（先看这个）

| 你看到的症状 | 最可能的原因 | 优先定位文件 | 直接看哪里 |
|---|---|---|---|
| 玩家输入没反应 / 指令不执行 | AI 输出 JSON 无法解析或 ID 不匹配 | `src/engine/ai/parsers.py`、`src/engine/ai/orchestrator.py` | `ParseResult.error` 与 repair 重试日志 |
| NPC “不理玩家”只会 MOVE/ATTACK | 小队 prompt 未强制回应对话 | `src/engine/ai/prompts.py`、`src/engine/ai/orchestrator.py` | `build_squad_prompt()` 规则 + `squad_think()` 输出 |
| 旁白“脑补”了没发生的事 | host prompt 事实边界不够严或 facts 翻译不一致 | `src/engine/ai/prompts.py`、`src/engine/ai/facts.py`、`src/engine/runtime/renderer.py` | `host input:` 与 `convert_events_to_facts()` |
| “拿箱子里的东西”拿不到 | 容器未打开（没 SEARCH）或 target 指错 | `src/engine/model/systems.py` | `InteractionSystem` 的 SEARCH/GET 分支 |
| “门锁住了”但不知道是哪扇门 | MOVE 失败文案没引用门实体名 | `src/engine/model/systems.py` | `MovementSystem` 的 DoorLocker 命中与失败文本 |
| 模型偶尔返回空字符串导致无输出 | 上游安全策略/网络波动 | `src/engine/adapters/ai/*` | `AiClient.send()` 返回值是否为空 |


### 10.1 “NPC 不理玩家”

当前 NPC 行为主要由 `squad_think()` 的 `build_squad_prompt()` 决定。

- 如果 prompt 没有硬性要求“必须回应玩家对话”，模型完全可能选择全体 MOVE/ATTACK。
- 解决策略（按耦合度从低到高）：
  1. 调整 `build_squad_prompt()` 的规则，让“看到玩家说话就至少一人 TALK”成为硬约束。
  2. 引擎侧加兜底：若本回合玩家产生了 `DialogueEvent`，且 NPC 计划里没有任何 TALK，则强插一句（更确定，但更脚本化）。

### 10.2 “为什么旁白不能引用 SearchableComponent.texts？”

因为 `SearchableComponent.texts` 是“未来可能发生”的结果，只有当系统真的执行了 `SEARCH` 并发布了 `SearchEvent`，它才变成事实。

旁白的唯一事实来源是事件（`Event`），这条约束写在 `build_host_prompt()` 里。

### 10.3 “AI 输出烂 JSON 导致指令不执行”

`orchestrator.py` 对玩家解析与小队协同都做了：

- 解析失败 -> `build_repair_prompt()` -> 最多重试 2 次
- 仍失败 -> 降级为空指令（不崩）

你要提升稳定性，优先从：提示词更严格、`ParseResult` 的错误反馈更精确、以及 `AiClient.send()` 的统一兜底入手。

### 10.4 “GET/SEARCH 逻辑跟直觉不一致”

当前规则是：

- `SEARCH` 的作用是“打开容器并看到内容”，它会把 `InventoryComponent.is_open=True`，并发布 `SearchEvent`（事实里会出现“你搜索了X，看到了：...”）。
- `GET` 的作用是“把物品转移到执行者背包”。
  - 如果 target 是容器：必须先 `is_open=True` 才能把里面所有物品转移。
  - 如果 target 是容器中的某个物品：容器必须是 open 状态，且目标确实在该容器 items 里。
  - 如果 target 是场景地面的物品：它必须带 `LocationComponent` 且与执行者同场景。

因此玩家说“搜刮尸体”在自然语言里可能等价于“SEARCH+GET”，这部分是靠 `build_player_parse_prompt()` 的 “Implicit Pre-actions” 规则让模型补全的。

### 10.5 “对话发生的场景不对 / 先 MOVE 再 TALK 导致叙事错乱”

`Action(scene_id=...)` 与 `DialogueEvent.at_scene_id` 是为了解决这个问题：

- `run_action_phase` 出队时给 Action 记录“执行瞬间的 scene_id”。
- `DialogSystem` 在生成 `DialogueEvent` 时优先使用 `action.scene_id`，避免“移动后对话被算到新场景”。

如果你仍看到对话错位，优先检查：

- `src/engine/runtime/game_loop.py`：Action 构造时是否传了 scene_id
- `src/engine/model/systems.py`：DialogSystem 对 at_scene_id 的取值逻辑

### 10.6 “为什么 UI 上的回合事实和旁白用的事实不一样？”

这是设计使然（见 9.6）：

- UI 展示：按场景分组的 `turn_facts_display`
- 旁白输入：`world.history[-1]` 的 `turn_transcript`（可能只包含玩家可见片段）

当你要排“旁白为什么这么写”，请优先看 debug 日志里 `host input:`（renderer 会打印）而不是 UI 面板。

### 10.7 调试建议（像读日志一样读引擎）

**追踪一条指令的最短路径**（从“一句中文”到“事件/旁白”）：

1. UI 输入：`src/engine/ui/tk_debug.py`（`read_player_input`）
2. AI 解析：`src/engine/ai/orchestrator.py:parse_player_input()`
3. JSON→Command：`src/engine/ai/parsers.py:parse_json_to_command_result()`
4. 出队执行：`src/engine/runtime/game_loop.py`（`run_action_phase` 内部逻辑）
5. 规则裁决：`src/engine/model/systems.py`（匹配对应 Command 的 System）
6. 事实生成：`src/engine/ai/facts.py:convert_events_to_facts()`
7. 旁白输入：`src/engine/runtime/renderer.py:formatted_output()`（看 `host input:`）
8. 旁白输出：AI 返回文本 → UI `narrate`

**最有用的观察点**：

- 状态栏：`player_info()` 会列出全部实体、位置与血量（强力调试工具）。
- `host input:`：旁白真正看到的上下文（排叙事问题必看）。
- `Squad AI input/output:`：NPC 行为真正看到的上下文与返回计划（排 NPC 行为问题必看）。


## 11. 扩展指南（按“改哪里”的顺序）

### 11.1 新增一个组件（最常见）

1. 在 `src/engine/model/components.py` 新增组件类（中文字段注释可写在 docstring）
2. 在 `src/engine/content/registry.py` 注册字符串名
3. 若组件包含 Entity 引用：
   - `src/engine/content/loader.py`：`_has_entity_refs()` + `_resolve_entity_refs()` + `_normalize_args()`（必要时）
4. 若会影响旁白或 AI 可见性：
   - `src/engine/ai/context.py`：决定是否展示到实体列表
   - `src/engine/ai/facts.py`：决定是否翻译成事实文本

### 11.2 新增一个指令（Command）

1. `src/engine/model/commands.py`：新增 `XxxCommand`
2. `src/engine/model/systems.py`：在对应 System 中实现对该 Command 的匹配与效果
3. `src/engine/ai/prompts.py`：把新指令写进 `COMMAND_MANUAL`（否则 AI 永远不会产出）
4. `src/engine/ai/parsers.py`：实现从 JSON -> `XxxCommand` 的解析
5. `src/engine/ai/facts.py`：新增对应 `Event` 的事实翻译（如果你会发布新 Event）

### 11.3 新增一个系统（规则模块化）

1. `src/engine/model/systems.py`：新增 `System.update()` 实现
2. `src/engine/content/loader.py:_register_systems()`：注册到 action 或 environment 系统列表

### 11.4 新增一个模型适配器

1. `src/engine/adapters/ai/` 新增实现函数（返回字符串）
2. `src/engine/adapters/ai/router.py` 增加 `case`
3. `src/engine/adapters/ai/clients.py` 增加 client 初始化（如需）

## 12. 推荐阅读路径（按“等价读代码”的顺序）

1. `src/engine/main.py`：入口（怎么启动、依赖什么）
2. `src/engine/content/loader.py`：世界如何由数据构建（核心装配）
3. `src/engine/model/components.py`：世界有哪些状态（数据面）
4. `src/engine/model/systems.py`：规则如何执行（逻辑面）
5. `src/engine/runtime/game_loop.py`：回合如何编排（流程面）
6. `src/engine/ai/*`：AI 如何被约束与接入（可靠性面）
7. `src/engine/ui/tk_debug.py`：调试 UI 如何与主循环交互（可观测性面）
