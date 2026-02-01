# Project Structure & Architecture Overview (Detailed)

Goal: after reading this document, you should be able to navigate and reason about the codebase almost as if you had read the entire project once—**where things are, why they exist, how data flows, how to extend, and where bugs usually hide**.

> Note: `docs/PROJECT_OVERVIEW.md` is the Chinese detailed overview. This file is the English counterpart.

## 0. Mental Model (load the whole system into your head first)

### 0.1 What is this project?

An AI-native, turn-based narrative game engine:

- **Rules & state** are owned by code (ECS + Systems). The engine updates the world deterministically.
- **AI is constrained** to three jobs only:
  1. Parse player natural language into executable **JSON commands** (`Command`)
  2. Plan NPC actions (currently via a **squad planning** prompt) as **JSON commands**
  3. Turn the turn’s **events** into **narration text** (the GM/host renderer)

Key boundary: **AI does not mutate world state directly**. It can only propose intent (commands/text). Whether it happens and what it causes is decided by engine rules and recorded as events.

### 0.2 The main pipeline in one line

```
assets(JSON) -> GameLoader -> World(Entity+Component+Scene)
player input -> InputAI(parse) -> CommandsQueue
CommandsQueue -> Action -> Systems(update) -> Events
Events -> Facts -> OutputAI(narration) -> UI
Events -> MemoryAI -> write back NPC memory
```

### 0.3 Glossary (the minimum vocabulary to read the code)

- **Entity**: a UUID identifier; all data lives in components.
- **Component**: data blocks attached to entities (Pydantic models).
- **System**: rule executor; reads `World.get_actions()`, matches `Command`, mutates components, publishes `Event`.
- **Command**: immutable instruction like `MOVE/ATTACK/SEARCH/...`; produced by player/NPC AI.
- **Action**: a dequeued command “about to be executed”, including `scene_id` snapshot for narrative consistency.
- **Event**: what actually happened (or why it failed). Narration must be grounded in events.
- **Scene**: map node with exits and environmental damage.
- **AP (Action Points)**: per-entity budget; total command cost per phase/turn is limited by AP.
- **Short IDs (`E1`, `E2`, …)**: AI-facing IDs mapped to UUIDs to keep the model stable.

## 1. Top-level layout

- `assets/`: data-driven game content
  - `global.json`: title/intro + player start config
  - `archetypes/*.json`: entity prefabs (NPCs/monsters/items/doors/searchables/etc.)
  - `scenes/*.json`: scenes and per-scene instances
  - `instances.json`: globally referenceable instances (stable string IDs)
- `src/`: engine code
  - `src/engine/main.py`: Tk debug frontend entrypoint
- `docs/`: repo docs
  - `docs/PROJECT_OVERVIEW.md`: Chinese detailed overview
  - `docs/PROJECT_OVERVIEW_EN.md`: this file
  - `docs/NEXT_STEPS.md`: prioritized improvements
- `requirements.txt`: Python deps
- `README.md` / `Proposal.md`: roadmap + data-driven proposal

## 2. Engine layers (src/engine)

Architecturally this resembles **Ports & Adapters (Hexagonal)**:

- `ports/`: interfaces (AI / UI)
- `adapters/`: implementations (LLM vendors, GUI)
- `model/`: domain model (ECS)
- `runtime/`: orchestration (game loop + rendering)
- `content/`: assets -> world loader
- `ai/`: prompts + parsing + orchestration glue

Below is a file-level tour.

## 3. `model/` (ECS core: state + rules)

### 3.1 `src/engine/model/entity.py`

- `Entity = UUID`

Entities are pure identifiers. Names/HP/location/etc. live in components.

### 3.2 `src/engine/model/components.py`

Components are Pydantic models. This is where the “shape of the world state” is defined.

Common components (grouped by responsibility):

- Narrative identity:
  - `IntroComponent(name, description)`
- Space:
  - `LocationComponent(scene_id)`
- Survival:
  - `HealthComponent(current, max)`
  - `StatusComponent(statuses=[StatusEntry])` (durations tick down each turn)
- Containers:
  - `InventoryComponent(items, is_open=False)` (open state gates visibility + looting)
- Behavior:
  - `CommandsQueueComponent(commands: Deque[Command])`
  - `ActionPointComponent(current_point, max_point)`
  - `AttackableComponent(damage)` (also used by “weapon items”)
- Role markers:
  - `PlayerComponent(game_history)`
  - `NPCComponent(memory, character, mission, limit)`
  - `MonsterComponent()`
  - `SquadMemberComponent()` (used for squad planning)
- Interactions:
  - `DoorLockerComponent(is_locked, locked_direction, key_entity)`
  - `SearchableComponent(texts)` (potential text; only becomes “fact” after a `SearchEvent`)
  - `ChallengeComponent(challenges)` (check rules like `UNLOCK_AGI`)
  - `PreventFromExitComponent(escape_rate, reason)`
- Drops / quests:
  - `DropOnDeathComponent(items)`
  - `QuestItemComponent(tag)` (used in victory checks)

Important reading hint: the engine often treats “component presence” as an ability/tag (not just data).

### 3.3 `src/engine/model/commands.py`

Commands are immutable (`ImmutableModel`). They represent intent.

- `Command(cost=1)` base
- Main commands:
  - `MoveCommand(direction, cost=2)`
  - `AttackCommand(target, cost=2)`
  - `PerformCheckCommand(target, intent, attribute, cost=2)`
  - `SearchCommand(target, cost=1)`
  - `UnlockCommand(target, key, cost=1)`
  - `GetCommand(target, cost=1)`
  - `UseCommand(item, target, cost=1)`
  - `TalkCommand(target, text, cost=0)` (`target` can be an entity, `"scene"`, or `"fluff"`)

Also:

- `Action(executor, command, scene_id=None)` records the execution context for more consistent narration.

### 3.4 `src/engine/model/event.py`

Events are the only reliable record of “what happened”.

- `EntitySnapshot(id, intro)` captures entity + `IntroComponent` so events remain safe even if entities are deleted later.
- `Event(executor)` base (executor is `EntitySnapshot` or `Scene`)

Common events:

- Intent/logging: `ActionIntentEvent(...)` (shows up as `attempt:` lines in facts)
- Success: `MoveEvent`, `UnlockEvent`, `GetEvent`, `ItemUseEvent`, `SearchEvent`, `DamageEvent`, `CheckEvent`, `DialogueEvent`
- Failure:
  - `ActionFailEvent(texts)` (hard failure with a reason)
  - `ActionContextFailEvent(texts)` (soft failure, typically for `"fluff"`)
- Death: `EntityDiedEvent(scene_id)`
- Endgame: `GameOverEvent(reason, winner)`

### 3.5 `src/engine/model/scene.py`

- `Direction(Enum)`: `north/south/east/west` (values are lowercase)
- `Scene(id, name, description, exits, damage)`
  - `exits: Dict[Direction, str]` maps direction -> destination `scene_id`
  - `damage` is per-turn environmental damage applied by `SceneSystem`

### 3.6 `src/engine/model/world.py`

`World` is the ECS container and runtime context:

- Storage:
  - `_entities: Dict[Entity, Dict[Type[Component], Component]]`
  - `_scenes: Dict[str, Scene]`
- Queues:
  - `_events_queue: List[Event]`
  - `_actions_queue: List[Action]`
- Systems:
  - `_action_systems: List[System]`
  - `_environment_systems: List[System]`
- AI/narrative:
  - `history: List[str]` (turn transcripts used as AI context)
  - `last_player_input_raw: str` (for narration style only; not facts)
  - `title`, `intro_text`
  - `_short_id_map/_short_id_counter` (short ID mapping)

Core methods:

- ECS: `create_entity()`, `add_component()`, `get_component()`, `get_entities_with()`
- Action/Event: `publish_action()`, `publish_event()`, `update_actions()`, `update_environment()`

### 3.7 `src/engine/model/systems.py` (the ruleset)

This file defines gameplay. AI proposes commands; systems decide outcomes and publish events.

#### 3.7.1 `InteractionSystem` (SEARCH / UNLOCK / GET / USE)

- `SEARCH`
  - requires `SearchableComponent`
  - if target has `InventoryComponent`: set `is_open=True`
  - publishes `SearchEvent(texts=...)` listing visible items
- `UNLOCK`
  - requires `DoorLockerComponent`
  - requires the correct key in executor inventory (matches `door.key_entity`)
  - success: sets `is_locked=False` and publishes `UnlockEvent`
  - failure: publishes `ActionFailEvent(...)`
- `GET` has three main paths:
  1. target is a container (`InventoryComponent`): must be open to transfer all items
  2. target is an item inside an open container: remove it from that container and add to executor inventory
  3. target is a world item in the same scene: remove its `LocationComponent` and add to executor inventory
  - publishes `GetEvent(entities=[...])` on success
- `USE`
  - supports `ConsumableComponent(effect, amount)` for now
  - publishes `ItemUseEvent(...)` and deletes the consumed item entity

#### 3.7.2 `CheckSystem` (PERFORM_CHECK)

- looks up `ChallengeComponent.challenges[f\"{intent}_{attribute}\"]`
- roll: `d20 + attribute_value` (from `AttributeComponent`, defaults to 10)
- applies effect: `destroy/unlock/damage/add_status/modify_relation`
- publishes `CheckEvent(...)` (includes roll, dc, success, effect)

#### 3.7.3 `MovementSystem` (MOVE)

- uses current scene exits (`Scene.exits`) to get destination
- if exit exists:
  - scans same-scene `DoorLockerComponent` entities:
    - if a door locks the attempted direction and is locked -> publishes `ActionFailEvent`
  - scans `PreventFromExitComponent` blockers (probabilistic)
  - otherwise publishes `MoveEvent` and updates `LocationComponent.scene_id`
- if exit does not exist: publishes `ActionFailEvent` (“no exit”)

#### 3.7.4 `CombatSystem` (ATTACK)

- target must be alive (`HealthComponent`) and in-scene (`LocationComponent`)
- damage = executor base `AttackableComponent.damage` + sum of `AttackableComponent.damage` on items in executor inventory
- applies HP change and publishes `DamageEvent`

#### 3.7.5 `SceneSystem` (environmental damage)

For each scene with `damage > 0`, publishes `DamageEvent` against living entities in that scene and reduces HP.

#### 3.7.6 `StatusSystem` (status duration decay)

Ticks down `StatusEntry.duration` each turn; removes expired entries.

#### 3.7.7 `HealthSystem` (death + corpse spawning)

- watches `DamageEvent`/`ItemUseEvent` outcomes via current HP
- on `current <= 0`: calls `_handle_death()`
  - for non-player NPC/monster: spawns a corpse entity with `SearchableComponent` + `InventoryComponent(items=drop)` and deletes the original entity
  - publishes `EntityDiedEvent(scene_id=...)`
- clamps `current` back to `max` if healing exceeds max

#### 3.7.8 `DialogSystem` (TALK)

- `target="fluff"` -> `ActionContextFailEvent` (soft failure)
- `target="scene"` -> `DialogueEvent(receiver=Scene, at_scene_id=...)`
- `target=Entity` -> `DialogueEvent(receiver=EntitySnapshot, at_scene_id=...)`

## 4. `content/` (assets -> World)

### 4.1 `src/engine/content/registry.py`

Component registry: string name -> component class. If an archetype uses a component name not in this registry, the loader warns and skips it.

### 4.2 `src/engine/content/loader.py`

`GameLoader` is the data-driven assembly line.

Internal state:

- `archetypes`: aggregated from `assets/archetypes/*.json`
- `instance_map`: stable instance_id (string) -> real `Entity(UUID)`
- `pending_refs`: components that reference other entities are delayed for a second pass
- `global_config`

Load order (`load_all()`):

1. `_load_global()` reads `assets/global.json` and sets `world.title/world.intro_text`
2. `_load_all_archetypes()` reads all archetype files
3. `_build_scenes_first_pass()` creates scenes, then creates per-scene entities
4. `_build_instances_first_pass()` creates globally referenced entities (if `instances.json` exists)
5. `_build_player()` creates the player from `global.json.player.initial_archetype`
6. `_resolve_pending_refs()` resolves entity references (string IDs -> UUID entities)
7. `_ensure_player_components()` validates required player components exist
8. `_register_systems()` wires action and environment systems

Archetype overrides:

- `remove_components`: remove component names from base
- `override_components`: deep-merge dict parameters onto base component args

Reference resolution (two-pass):

- `InventoryComponent.items`, `DoorLockerComponent.key_entity`, `DropOnDeathComponent.items` are treated as entity references:
  - pass 1: record them in `pending_refs`
  - pass 2: replace string IDs via `instance_map` and instantiate the component

Normalization:

- `CommandsQueueComponent.commands`: list -> `deque`
- `DoorLockerComponent.locked_direction`: `"NORTH"` -> `Direction.NORTH`

## 5. `ai/` (prompts + context + parsing + orchestration)

### 5.1 `src/engine/ai/context.py` (what the model can “see”)

Purpose: convert UUID world into a stable “short ID + entity list” context.

- `_get_short_id()` generates `E1/E2/...` using base62 and stores mapping on `World`
- `build_input_context()`:
  - entity set = same-scene entities + player inventory + items in open containers
  - returns `context_string` + `short_id_map` (short -> uuid)
- `build_squad_context()`:
  - returns entity list + squad member cards (AP/inventory/character/mission/limit/memory)

### 5.2 `src/engine/ai/prompts.py` (prompt templates)

Key templates:

- `COMMAND_MANUAL`: allowed JSON command schemas
- `build_player_parse_prompt(...)`: player NL -> JSON command array (must be pure JSON)
- `build_squad_prompt(...)`: NPC squad planning per scene (pure JSON array of plans)
- `build_batch_memory_prompt(...)`: batch NPC memory updates (pure JSON object)
- `build_host_prompt(...)`: GM narration (facts-only; no hallucination)
- `build_game_over_prompt(...)`: ending narration
- `build_repair_prompt(...)`: JSON repair loop prompt when parsing fails

### 5.3 `src/engine/ai/parsers.py` (JSON -> Command)

Responsibilities:

- strip code fences if present
- map short IDs back to UUID strings
- parse dicts into strongly-typed `Command` objects
- return `ParseResult` to drive retry/repair logic

### 5.4 `src/engine/ai/orchestrator.py` (AI call orchestration)

- `parse_player_input(...)`:
  - build context -> build prompt -> `ai_client.send()` -> parse -> repair retry (max 2)
- `squad_think(...)`:
  - build squad context -> build prompt -> parse plans -> repair retry (max 2)
- `batch_npc_memory_update(...)`:
  - build per-NPC cards -> batch prompt -> parse JSON object -> write memory back to NPC components

## 6. `runtime/` (turn orchestration + narration)

### 6.1 `src/engine/runtime/game_loop.py`

`run_game_loop(world, player, io, ai_client)` stitches everything into turns:

- intro: `world.intro_text` -> first narration -> initial status
- per turn (high-level):
  1. wait for player input if player queue empty
  2. player action phase (AP-budgeted)
  3. build transcript for AI context
  4. NPC planning per scene (squad)
  5. monster planning (simple heuristic)
  6. NPC+monster action phase
  7. environment phase
  8. build UI turn-facts display
  9. game over check (dead / quest item obtained)
  10. NPC memory update
  11. narration output + status refresh
  12. AP reset + clear events + token flush

#### 6.1.1 Action phase details (AP + attempt ordering)

Inside the loop, action execution is roughly:

1. For each executor with `CommandsQueueComponent` + `ActionPointComponent`:
   - if `ap.current_point >= next_command.cost`, dequeue and publish an `Action` (including `scene_id`)
   - optionally publish an `ActionIntentEvent` at dequeue time (this becomes `attempt:` in facts)
2. If any actions were published:
   - `world.update_actions()` runs systems and publishes events
   - on `ActionFailEvent`, executor queue may be cleared to prevent repeated failures
   - `world.clear_actions()` for the next loop

This is why logs often look like: `attempt -> success/fail events -> turn facts`.

### 6.2 `src/engine/runtime/renderer.py`

- `player_info(...)`: debug status panel text (player + entity listing)
- `formatted_output(...)`: builds “current situation” text + uses `world.history[-1]` facts to build host prompt and call AI for narration
- `render_game_over(...)`: builds ending prompt; falls back to a deterministic ending text if AI fails

## 7. Ports & Adapters

### 7.1 `src/engine/ports/ai.py`

`AiClient` protocol: `send(prompt) -> str`

### 7.2 `src/engine/ports/ui.py`

`GameIO` protocol: narrate/update status/read input/token stats/append turn facts/show game over.

### 7.3 `src/engine/adapters/ai/router.py`

`RoutingAiClient` routes a model name to specific implementations:

- `gemini`, `gemini-flash`, `kimi`, `kimi-turbo`, `qwen-7b`, `qwen-plus`, `deepseek`

### 7.4 `src/engine/adapters/ai/*.py` and `clients.py`

- `clients.py` loads vendor clients and reads API keys from environment variables
- adapter functions send prompts, return raw text, and (currently) call back into the GUI for token accounting

## 8. `ui/` (Tk debug frontend)

### 8.1 `src/engine/ui/tk_debug.py`

`DebugGUI` implements `GameIO`:

- runs `run_game_loop()` in a background thread
- uses a thread-safe queue to marshal messages back to the Tk main thread
- collects and displays logs, narration, per-turn facts, and token usage

## 9. Assets data protocol (grounded in current files)

### 9.1 `assets/global.json`

- `meta.title/version/intro_text`
- `player.default_name/start_scene_id/initial_archetype`

### 9.2 `assets/archetypes/*.json`

- `players.json`: `player_base` (HP/inventory/AP/attack/attributes/status)
- `npcs.json`: squad members (have `NPCComponent` + `SquadMemberComponent`)
- `monsters.json`: `wolf_base`, `boss_bear` (includes challenges/drops)
- `items.json`: keys/pelts/core/herb + interactables like `prop_bush` and `door_wood`

### 9.3 `assets/scenes/chapter1.json`

Defines scenes and instances. Instance `id` fields are stable string IDs.

Instance overrides can reference global instance IDs, e.g.:

- wolf inventory contains `"rusty_key_1"`
- door key reference is `"rusty_key_1"`

### 9.4 `assets/instances.json`

Defines globally referenceable instance IDs (currently grouped under `items`).

### 9.5 End-to-end example: door + key + reference resolution

1. Declare the key as a stable instance in `instances.json`:

```json
{
  "items": [
    { "id": "rusty_key_1", "archetype_id": "item_rusty_key" }
  ]
}
```

2. Reference it from scene instances via overrides:

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

3. The loader’s second pass resolves strings to real UUID entities and parses direction strings to `Direction`.

Runtime effect:

- `MOVE NORTH` fails while the door is locked (`MovementSystem` scans `DoorLockerComponent`).
- After obtaining the key, `UNLOCK` succeeds (`InteractionSystem` validates key possession and matches `key_entity`).
- After `is_locked=False`, `MOVE NORTH` succeeds and publishes `MoveEvent`.

### 9.6 `world.history` transcript vs UI “turn facts” display

There are two “facts texts”:

1. **Transcript appended to `world.history`**
   - used as AI context (narration and parsing)
   - usually includes player-visible events and `attempt:` lines
2. **UI turn facts display (`--- Turn N ---`)**
   - grouped by scene for humans

When debugging narration, always inspect the `host input:` debug log (what the model actually received), not only the UI panel.

## 10. Troubleshooting (symptoms -> where to look)

### 10.0 Symptom → likely cause → file

| Symptom | Likely cause | Primary files | Inspect |
|---|---|---|---|
| Player input does nothing | LLM JSON parse/ID mapping failed | `src/engine/ai/parsers.py`, `src/engine/ai/orchestrator.py` | `ParseResult.error` + repair logs |
| NPCs ignore player speech | squad prompt doesn’t enforce replies | `src/engine/ai/prompts.py`, `src/engine/ai/orchestrator.py` | `build_squad_prompt()` rules + squad output |
| Narration “hallucinates” | weak facts boundary or inconsistent facts translation | `src/engine/ai/prompts.py`, `src/engine/ai/facts.py`, `src/engine/runtime/renderer.py` | `host input:` + `convert_events_to_facts()` |
| Can’t loot from containers | container not open (missing `SEARCH`) or wrong target | `src/engine/model/systems.py` | `InteractionSystem` SEARCH/GET logic |
| “Door is locked” but unclear which door | failure text doesn’t reference door entity | `src/engine/model/systems.py` | `MovementSystem` door scan |
| Empty model response | vendor safety/network instability | `src/engine/adapters/ai/*` | `AiClient.send()` return content |

### 10.1 “NPCs don’t respond to the player”

NPC behavior is primarily driven by `squad_think()` which builds `build_squad_prompt()`.

If the prompt doesn’t hard-require a reply when the player speaks, the model may choose only movement/combat. Fix options:

1. Strengthen `build_squad_prompt()` rules (low coupling, best default).
2. Add an engine-level fallback: if the player produced a `DialogueEvent` but the NPC plan has no `TALK`, inject a leader reply (more deterministic, more scripted).

### 10.2 Why narration must not “use SearchableComponent.texts”

`SearchableComponent.texts` is a *potential outcome*. It becomes a fact only if the engine executes `SEARCH` and publishes `SearchEvent`. Narration is supposed to be grounded in events only (enforced in `build_host_prompt()`).

### 10.3 Malformed JSON from the model

Both player parsing and squad planning implement a repair loop:

- parse -> if fail, build `build_repair_prompt()` -> retry (max 2) -> fallback to empty commands

To improve stability: tighten prompts, improve parse error messages, and consider adding a unified “empty output retry/fallback” in `AiClient.send()`.

### 10.4 GET/SEARCH logic vs intuition

- `SEARCH` opens containers and publishes what you “saw”.
- `GET` transfers items into executor inventory.

So “loot the corpse” usually needs `SEARCH + GET` (the player parse prompt has an “Implicit Pre-actions” rule to encourage that).

### 10.5 Dialogue scene attribution issues

`Action.scene_id` and `DialogueEvent.at_scene_id` exist to avoid “move then talk” narrative drift. If you see dialogue assigned to the wrong scene, inspect:

- `src/engine/runtime/game_loop.py` action publishing (`scene_id` capture)
- `src/engine/model/systems.py` `DialogSystem` at_scene selection logic

## 11. Extension guide (what to change, in order)

### 11.1 Add a new component

1. Add the component class in `src/engine/model/components.py`
2. Register its string name in `src/engine/content/registry.py`
3. If it contains entity references:
   - extend `GameLoader._has_entity_refs()` / `_resolve_entity_refs()` / `_normalize_args()` as needed
4. If it affects AI visibility or narration:
   - update `src/engine/ai/context.py` (visibility)
   - update `src/engine/ai/facts.py` (facts translation)

### 11.2 Add a new command

1. Define `XxxCommand` in `src/engine/model/commands.py`
2. Implement rule handling in `src/engine/model/systems.py`
3. Add schema+semantics to `COMMAND_MANUAL` in `src/engine/ai/prompts.py`
4. Parse JSON into the new command in `src/engine/ai/parsers.py`
5. If you introduce new events, translate them in `src/engine/ai/facts.py`

### 11.3 Add a new system

1. Implement a `System.update()` in `src/engine/model/systems.py`
2. Register it in `src/engine/content/loader.py:_register_systems()` (action or environment list)

### 11.4 Add a new model adapter

1. Add an adapter function in `src/engine/adapters/ai/`
2. Add a `case` in `src/engine/adapters/ai/router.py`
3. Add client initialization in `src/engine/adapters/ai/clients.py` (if needed)

## 12. Recommended reading order (≈ “read the code once”)

1. `src/engine/main.py` (entrypoint)
2. `src/engine/content/loader.py` (assets -> world)
3. `src/engine/model/components.py` (state model)
4. `src/engine/model/systems.py` (rules)
5. `src/engine/runtime/game_loop.py` (turn orchestration)
6. `src/engine/ai/*` (constraints + reliability)
7. `src/engine/ui/tk_debug.py` (observability + IO)

