# Terminal (C1 Games) – Game Engine Documentation

A comprehensive reference for every function, class, attribute, and data structure available  
to you when writing `algo_strategy.py`.

---

## Table of Contents

1. [Quick-Start Cheat Sheet](#1-quick-start-cheat-sheet)
2. [Unit Types & Constants](#2-unit-types--constants)
3. [GameState](#3-gamestate)
   - 3.1 [Attributes](#31-attributes)
   - 3.2 [Resource Functions](#32-resource-functions)
   - 3.3 [Spawning & Building Functions](#33-spawning--building-functions)
   - 3.4 [Pathfinding & Targeting Functions](#34-pathfinding--targeting-functions)
   - 3.5 [Utility Functions](#35-utility-functions)
4. [GameMap](#4-gamemap)
   - 4.1 [Attributes & Indexing](#41-attributes--indexing)
   - 4.2 [GameMap Functions](#42-gamemap-functions)
5. [GameUnit](#5-gameunit)
   - 5.1 [Attributes](#51-attributes)
   - 5.2 [Methods](#52-methods)
6. [AlgoCore (Base Class)](#6-algocore-base-class)
7. [Utility Functions (gamelib.debug_write)](#7-utility-functions)
8. [Accessing Enemy State](#8-accessing-enemy-state)
9. [Action Frame Events](#9-action-frame-events)
10. [Game Configuration Values](#10-game-configuration-values)
11. [Full Usage Example](#11-full-usage-example)

---

## 1. Quick-Start Cheat Sheet

```python
import gamelib

class AlgoStrategy(gamelib.AlgoCore):

    def on_game_start(self, config):
        self.config = config
        global WALL, SUPPORT, TURRET, SCOUT, DEMOLISHER, INTERCEPTOR, MP, SP
        WALL        = config["unitInformation"][0]["shorthand"]   # "FF"
        SUPPORT     = config["unitInformation"][1]["shorthand"]   # "EF"
        TURRET      = config["unitInformation"][2]["shorthand"]   # "DF"
        SCOUT       = config["unitInformation"][3]["shorthand"]   # "PI"
        DEMOLISHER  = config["unitInformation"][4]["shorthand"]   # "EI"
        INTERCEPTOR = config["unitInformation"][5]["shorthand"]   # "SI"
        MP = 1   # Mobile Points resource index
        SP = 0   # Structure Points resource index

    def on_turn(self, turn_state):
        game_state = gamelib.GameState(self.config, turn_state)
        game_state.suppress_warnings(True)

        # ----- read state -----
        gamelib.debug_write("Turn:", game_state.turn_number)
        gamelib.debug_write("My HP:", game_state.my_health)
        gamelib.debug_write("Enemy HP:", game_state.enemy_health)
        gamelib.debug_write("My SP:", game_state.get_resource(SP))
        gamelib.debug_write("My MP:", game_state.get_resource(MP))

        # ----- build & deploy -----
        game_state.attempt_spawn(TURRET, [[3, 11]])
        game_state.attempt_spawn(WALL,   [[2, 12], [3, 12]])
        game_state.attempt_upgrade([[3, 11]])
        game_state.attempt_spawn(SCOUT,  [13, 0], 5)

        # ----- MUST call at end of every on_turn -----
        game_state.submit_turn()
```

---

## 2. Unit Types & Constants

### 2.1 Structure Units (cost SP – Structure Points)

| Constant    | Shorthand | Display Name | SP Cost | HP (base) | HP (upgraded) | Notes |
|-------------|-----------|--------------|---------|-----------|---------------|-------|
| `WALL`      | `FF`      | Filter       | 1       | 75        | 150           | Pure HP sponge; no attack |
| `SUPPORT`   | `EF`      | Encryptor    | 7       | 30        | —             | Generates 1 SP/turn (base); upgrade additionally generates 2 MP/turn |
| `TURRET`    | `DF`      | Destructor   | 2       | 90        | —             | Attacks mobile units; 5 dmg / range 2.5; upgraded: 15 dmg / range 3.5 |

Structure units occupy a single cell, can only be placed on your half of the map (y < 14),  
and block path-finding for all mobile units.

### 2.2 Mobile Units (cost MP – Mobile Points)

| Constant      | Shorthand | Display Name | MP Cost | HP  | Speed | Attack Range | Dmg vs Structures | Dmg vs Mobile |
|---------------|-----------|--------------|---------|-----|-------|--------------|-------------------|---------------|
| `SCOUT`       | `PI`      | Ping         | 1       | 15  | 1.0   | 3.5          | 2                 | 2             |
| `DEMOLISHER`  | `EI`      | EMP          | 3       | 5   | 0.5   | 4.5          | 6                 | 6             |
| `INTERCEPTOR` | `SI`      | Scrambler    | 1       | 40  | 0.25  | 3.5          | 0                 | 20            |

Mobile units must be spawned on edge locations (bottom-left or bottom-right edges).  
Multiple mobile units may stack on the same cell.

### 2.3 Action Constants

| Constant    | Shorthand | Purpose |
|-------------|-----------|---------|
| `REMOVE`    | `RM`      | Flag a structure for removal (returned in build stack via `attempt_remove`) |
| `UPGRADE`   | `UP`      | Flag a structure for upgrade (returned in build stack via `attempt_upgrade`) |

### 2.4 Resource Constants

| Python name | Integer | Description |
|-------------|---------|-------------|
| `SP`        | 0       | Structure Points – used to build/upgrade structures |
| `MP`        | 1       | Mobile Points – used to deploy mobile units |

### 2.5 List Constants (on GameState after `__init__`)

| Name              | Contents |
|-------------------|----------|
| `STRUCTURE_TYPES` | `[WALL, SUPPORT, TURRET]` |
| `ALL_UNITS`       | `[SCOUT, DEMOLISHER, INTERCEPTOR, WALL, SUPPORT, TURRET]` |

---

## 3. GameState

`GameState` is the primary object you interact with each turn.  
It is created inside `on_turn` with `game_state = gamelib.GameState(self.config, turn_state)`.

### 3.1 Attributes

| Attribute          | Type      | Description |
|--------------------|-----------|-------------|
| `turn_number`      | `int`     | Current turn number (starts at 0) |
| `my_health`        | `float`   | Your current HP (starts at 40) |
| `enemy_health`     | `float`   | Enemy's current HP |
| `my_time`          | `float`   | Time (ms) you spent on your previous turn |
| `enemy_time`       | `float`   | Time (ms) the enemy spent on their previous turn |
| `game_map`         | `GameMap` | The current game map (see Section 4) |
| `ARENA_SIZE`       | `int`     | `28` – total board size |
| `HALF_ARENA`       | `int`     | `14` – your half ends at y < 14 |
| `MP`               | `int`     | `1` – constant to pass to `get_resource` |
| `SP`               | `int`     | `0` – constant to pass to `get_resource` |
| `config`           | `dict`    | Full game configuration JSON |
| `enable_warnings`  | `bool`    | Whether debug warnings are printed |
| `WALL`             | `str`     | `"FF"` |
| `SUPPORT`          | `str`     | `"EF"` |
| `TURRET`           | `str`     | `"DF"` |
| `SCOUT`            | `str`     | `"PI"` |
| `DEMOLISHER`       | `str`     | `"EI"` |
| `INTERCEPTOR`      | `str`     | `"SI"` |
| `REMOVE`           | `str`     | `"RM"` |
| `UPGRADE`          | `str`     | `"UP"` |
| `STRUCTURE_TYPES`  | `list`    | `[WALL, SUPPORT, TURRET]` |
| `ALL_UNITS`        | `list`    | `[SCOUT, DEMOLISHER, INTERCEPTOR, WALL, SUPPORT, TURRET]` |

---

### 3.2 Resource Functions

#### `get_resource(resource_type, player_index=0)`
Returns the current amount of the specified resource for the given player.

- **`resource_type`** – `game_state.SP` (0) or `game_state.MP` (1)  
- **`player_index`** – `0` = you, `1` = enemy  
- **Returns** – `float`

```python
my_sp  = game_state.get_resource(SP)          # your SP
my_mp  = game_state.get_resource(MP)          # your MP
enemy_sp = game_state.get_resource(SP, 1)     # enemy SP
enemy_mp = game_state.get_resource(MP, 1)     # enemy MP
```

---

#### `get_resources(player_index=0)`
Returns both resources at once.

- **`player_index`** – `0` = you, `1` = enemy  
- **Returns** – `[SP_amount, MP_amount]` (list of two floats)

```python
sp, mp = game_state.get_resources()           # your resources
esp, emp = game_state.get_resources(1)        # enemy resources
```

---

#### `number_affordable(unit_type)`
Returns how many of a given unit type you can currently afford.

- **`unit_type`** – any unit constant, e.g. `SCOUT`, `TURRET`  
- **Returns** – `int`

```python
scouts_affordable = game_state.number_affordable(SCOUT)
turrets_affordable = game_state.number_affordable(TURRET)
```

---

#### `type_cost(unit_type, upgrade=False)`
Returns the SP and MP cost of a unit as a list.

- **`unit_type`** – unit constant  
- **`upgrade`** – if `True`, returns the upgrade cost instead of the base cost  
- **Returns** – `[SP_cost, MP_cost]`

```python
wall_cost   = game_state.type_cost(WALL)         # [1, 0]
scout_cost  = game_state.type_cost(SCOUT)        # [0, 1]
turret_upgrade_cost = game_state.type_cost(TURRET, upgrade=True)
```

---

#### `project_future_MP(turns_in_future=1, player_index=0, current_MP=None)`
Predicts how much MP a player will have after a given number of turns.

- **`turns_in_future`** – integer 1–99  
- **`player_index`** – `0` = you, `1` = enemy  
- **`current_MP`** – override the starting MP value (optional)  
- **Returns** – `float`

```python
mp_next_turn = game_state.project_future_MP(1)
mp_in_3      = game_state.project_future_MP(3)
enemy_mp_3   = game_state.project_future_MP(3, player_index=1)
```

---

### 3.3 Spawning & Building Functions

#### `can_spawn(unit_type, location, num=1)`
Checks whether you can spawn the given unit at the given location.

- **`unit_type`** – unit constant  
- **`location`** – `[x, y]`  
- **`num`** – number of units to check for (default 1)  
- **Returns** – `True` / `False`

Checks performed:
1. Unit type is valid
2. Location is in arena bounds
3. You can afford `num` units
4. Location is not blocked (for structures)
5. Location is on your half (y < 14)
6. Location is on the edge (for mobile units)

```python
if game_state.can_spawn(SCOUT, [13, 0]):
    game_state.attempt_spawn(SCOUT, [13, 0])
```

---

#### `attempt_spawn(unit_type, locations, num=1)`
Attempts to spawn units; automatically deducts resources.

- **`unit_type`** – unit constant  
- **`locations`** – a single `[x, y]` **or** a list of locations  
- **`num`** – number of units to spawn at **each** location (useful for mobile units)  
- **Returns** – `int` number of units actually spawned

```python
# Spawn a single turret
game_state.attempt_spawn(TURRET, [5, 11])

# Spawn turrets at multiple locations
game_state.attempt_spawn(TURRET, [[5, 11], [8, 11], [13, 11]])

# Spawn as many scouts as possible from one spot
spawned = game_state.attempt_spawn(SCOUT, [13, 0], 1000)
```

---

#### `attempt_remove(locations)`
Flags friendly structures for removal at the end of the turn.

- **`locations`** – a single `[x, y]` **or** a list of locations  
- **Returns** – `int` number of structures flagged for removal

```python
game_state.attempt_remove([5, 11])
game_state.attempt_remove([[5, 11], [8, 11]])
```

---

#### `attempt_upgrade(locations)`
Attempts to upgrade friendly structures at the given locations.

- **`locations`** – a single `[x, y]` **or** a list of locations  
- **Returns** – `int` number of units successfully upgraded

```python
game_state.attempt_upgrade([5, 11])
game_state.attempt_upgrade([[5, 11], [8, 11]])
```

---

#### `submit_turn()`
**Must be called at the end of every `on_turn`.** Sends your build and deploy queues to the game engine.

```python
game_state.submit_turn()
```

---

### 3.4 Pathfinding & Targeting Functions

#### `find_path_to_edge(start_location, target_edge=None)`
Returns the path a mobile unit would travel from `start_location`.

- **`start_location`** – `[x, y]` (must be unblocked)  
- **`target_edge`** – one of `game_map.TOP_LEFT`, `game_map.TOP_RIGHT`, `game_map.BOTTOM_LEFT`, `game_map.BOTTOM_RIGHT`; auto-detected from `start_location` if `None`  
- **Returns** – list of `[x, y]` locations forming the path (empty / `None` if blocked start)

```python
path = game_state.find_path_to_edge([13, 0])
# path is a list: [[13,0], [13,1], ..., [x, 27]]
```

---

#### `get_target_edge(start_location)`
Returns the target edge a unit spawned at `start_location` would travel toward.

- **`start_location`** – `[x, y]`  
- **Returns** – `int` edge constant (`game_map.TOP_LEFT`, etc.)

```python
edge = game_state.get_target_edge([13, 0])  # game_map.TOP_RIGHT
```

---

#### `get_target(attacking_unit)`
Simulates which unit `attacking_unit` would choose to attack given current game state.

Priority order (highest to lowest):
1. **Mobile over Stationary** – mobile units are always preferred as targets over structures
2. **Nearest** – among equal priority, the unit with the smallest Euclidean distance wins
3. **Lowest HP** – if distance is equal, the unit with less remaining health wins
4. **Lowest Y (player 0 attacker) / Highest Y (player 1 attacker)** – if HP is also equal, the unit closest to the attacker's own edge wins (lower y for player 0, higher y for player 1)
5. **Closest to board edge** – final tie-break: highest `|x − 13.5|` (farthest from vertical centre)

- **`attacking_unit`** – a `GameUnit`  
- **Returns** – `GameUnit` or `None`

```python
for location in game_state.game_map:
    for unit in game_state.game_map[location]:
        target = game_state.get_target(unit)
        if target:
            gamelib.debug_write(f"{unit} targets {target}")
```

---

#### `get_attackers(location, player_index)`
Returns a list of enemy stationary units that would attack a unit at `location`.

- **`location`** – `[x, y]`  
- **`player_index`** – player whose unit is *being attacked* (`0` = yours, `1` = enemy's)  
- **Returns** – `list` of `GameUnit` objects

```python
# How many enemy turrets threaten [13, 0]?
attackers = game_state.get_attackers([13, 0], 0)
total_damage = sum(a.damage_i for a in attackers)
```

---

#### `contains_stationary_unit(location)`
Checks whether a location contains a structure.

- **`location`** – `[x, y]`  
- **Returns** – the `GameUnit` structure if one exists, `False` otherwise

```python
unit = game_state.contains_stationary_unit([5, 11])
if unit:
    gamelib.debug_write(f"Structure at [5,11]: {unit.unit_type}")
```

---

### 3.5 Utility Functions

#### `suppress_warnings(suppress)`
Enables or disables internal warning messages printed via `debug_write`.

- **`suppress`** – `True` to silence warnings, `False` to re-enable  

```python
game_state.suppress_warnings(True)   # silence warnings
game_state.suppress_warnings(False)  # re-enable warnings
```

---

#### `warn(message)`
Prints a debug warning (if warnings are not suppressed). Used internally but callable.

```python
game_state.warn("Custom warning message")
```

---

## 4. GameMap

Accessed via `game_state.game_map`.  
Represents the 28×28 diamond-shaped arena.

### 4.1 Attributes & Indexing

| Attribute      | Type   | Value | Description |
|----------------|--------|-------|-------------|
| `ARENA_SIZE`   | `int`  | 28    | Side length of the bounding square |
| `HALF_ARENA`   | `int`  | 14    | Half of ARENA_SIZE |
| `TOP_RIGHT`    | `int`  | 0     | Edge constant for top-right |
| `TOP_LEFT`     | `int`  | 1     | Edge constant for top-left |
| `BOTTOM_LEFT`  | `int`  | 2     | Edge constant for bottom-left (your left spawn) |
| `BOTTOM_RIGHT` | `int`  | 3     | Edge constant for bottom-right (your right spawn) |
| `enable_warnings` | `bool` | `True` | If `False`, map-related warnings are silenced |

#### Indexing: `game_map[x, y]`
Returns a `list` of `GameUnit` objects at that location (empty list if none).

```python
units_at = game_state.game_map[13, 13]   # list of GameUnit
for unit in units_at:
    gamelib.debug_write(unit)
```

#### Iterating the whole map
```python
for location in game_state.game_map:
    units = game_state.game_map[location]
    if units:
        gamelib.debug_write(f"Units at {location}: {units}")
```

---

### 4.2 GameMap Functions

#### `in_arena_bounds(location)`
Returns `True` if `[x, y]` is inside the diamond-shaped arena.

```python
game_state.game_map.in_arena_bounds([13, 13])   # True
game_state.game_map.in_arena_bounds([0, 0])     # False (corner of bounding box)
```

---

#### `get_edge_locations(quadrant_description)`
Returns the list of `[x, y]` locations along the specified edge.

- **`quadrant_description`** – `game_map.TOP_LEFT`, `game_map.TOP_RIGHT`, `game_map.BOTTOM_LEFT`, or `game_map.BOTTOM_RIGHT`

```python
bottom_left_edge  = game_state.game_map.get_edge_locations(game_state.game_map.BOTTOM_LEFT)
bottom_right_edge = game_state.game_map.get_edge_locations(game_state.game_map.BOTTOM_RIGHT)
friendly_edges    = bottom_left_edge + bottom_right_edge  # all your spawn locations
```

---

#### `get_edges()`
Returns all four edges as a list of lists.

- **Returns** – `[[top_right], [top_left], [bottom_left], [bottom_right]]`

```python
all_edges = game_state.game_map.get_edges()
top_right_edge   = all_edges[0]
top_left_edge    = all_edges[1]
bottom_left_edge = all_edges[2]
bottom_right_edge = all_edges[3]
```

---

#### `get_locations_in_range(location, radius)`
Returns all arena locations within `radius` of `location`.

- **`location`** – `[x, y]` centre  
- **`radius`** – float/int (max `ARENA_SIZE`)  
- **Returns** – list of `[x, y]`

```python
nearby = game_state.game_map.get_locations_in_range([13, 13], 3.5)
```

---

#### `distance_between_locations(location_1, location_2)`
Euclidean distance between two `[x, y]` points.

- **Returns** – `float`

```python
dist = game_state.game_map.distance_between_locations([0, 0], [3, 4])  # 5.0
```

---

#### `add_unit(unit_type, location, player_index=0)`
Manually adds a `GameUnit` to the map (for hypothetical analysis only).  
**Does not affect your actual turn.** Using this on the live `game_map` will desync it.

```python
# Safe usage: work on a copy
import copy
hypothetical_map = copy.deepcopy(game_state.game_map)
hypothetical_map.add_unit(TURRET, [5, 11], player_index=0)
```

---

#### `remove_unit(location)`
Removes all units from a location (for hypothetical analysis only).

```python
game_state.game_map.remove_unit([5, 11])
```

---

## 5. GameUnit

Represents a single unit (structure or mobile) on the board.  
Instances are returned by `game_map[x, y]`, `get_attackers()`, `get_target()`, etc.

### 5.1 Attributes

| Attribute        | Type    | Description |
|------------------|---------|-------------|
| `unit_type`      | `str`   | Shorthand string, e.g. `"FF"`, `"PI"` |
| `player_index`   | `int`   | `0` = your unit, `1` = enemy unit |
| `x`              | `int`   | X coordinate on the map |
| `y`              | `int`   | Y coordinate on the map |
| `health`         | `float` | Current HP |
| `max_health`     | `float` | Starting HP (can be exceeded via shielding) |
| `stationary`     | `bool`  | `True` if this is a structure (WALL/SUPPORT/TURRET) |
| `speed`          | `float` | Frames moved per turn (0 for structures) |
| `damage_f`       | `int`   | Damage dealt to structures per attack frame |
| `damage_i`       | `int`   | Damage dealt to mobile units per attack frame |
| `attackRange`    | `float` | Attack radius |
| `shieldRange`    | `float` | Shield radius (SUPPORT units) |
| `shieldPerUnit`  | `float` | Shield HP granted per unit in range |
| `shieldBonusPerY`| `float` | Additional shield per Y-unit distance |
| `cost`           | `list`  | `[SP_cost, MP_cost]` |
| `pending_removal`| `bool`  | `True` if owner flagged this unit for removal |
| `upgraded`       | `bool`  | `True` if this unit has been upgraded |

### 5.2 Methods

#### `upgrade()`
Applies upgrade stats to this `GameUnit` instance (called internally by `attempt_upgrade`).

---

## 6. AlgoCore (Base Class)

`AlgoStrategy` inherits from `gamelib.AlgoCore`.  
You override the following methods:

### `on_game_start(config)`
Called **once** at the beginning of the game. Use it to read the config and set global constants.

```python
def on_game_start(self, config):
    self.config = config
    global WALL, SUPPORT, TURRET, SCOUT, DEMOLISHER, INTERCEPTOR, MP, SP
    WALL        = config["unitInformation"][0]["shorthand"]
    SUPPORT     = config["unitInformation"][1]["shorthand"]
    TURRET      = config["unitInformation"][2]["shorthand"]
    SCOUT       = config["unitInformation"][3]["shorthand"]
    DEMOLISHER  = config["unitInformation"][4]["shorthand"]
    INTERCEPTOR = config["unitInformation"][5]["shorthand"]
    MP = 1
    SP = 0
```

---

### `on_turn(turn_state)`
Called **once per turn** with the raw game-state JSON string.  
Create a `GameState` object inside this method, build your strategy, then call `submit_turn()`.

```python
def on_turn(self, turn_state):
    game_state = gamelib.GameState(self.config, turn_state)
    # ... your logic ...
    game_state.submit_turn()
```

---

### `on_action_frame(action_frame_game_state)`
Called for **every frame** of the action phase (potentially hundreds of times per turn).  
Use it to track events such as breaches, deaths, or attacks.  
Keep this function fast – avoid slow computation here.

```python
def on_action_frame(self, turn_string):
    import json
    state = json.loads(turn_string)
    events = state["events"]
    for breach in events["breach"]:
        location = breach[0]
        owner    = breach[4]   # 1 = you, 2 = opponent
        if owner == 2:
            gamelib.debug_write(f"Enemy scored at {location}")
```

---

### `start()`
**Do not override.** Runs the main game loop, reading from stdin and dispatching to  
`on_game_start`, `on_turn`, or `on_action_frame` as appropriate.

---

## 7. Utility Functions

Imported directly from `gamelib`:

### `gamelib.debug_write(*msg)`
Prints a message to the game's debug/error output (stderr). Does **not** affect your turn.

```python
gamelib.debug_write("Hello world")
gamelib.debug_write("Turn:", game_state.turn_number, "HP:", game_state.my_health)
```

---

## 8. Accessing Enemy State

All enemy information is available through `game_state` and `game_state.game_map`.

### 8.1 Enemy Health & Resources

```python
enemy_hp = game_state.enemy_health
enemy_sp = game_state.get_resource(SP, 1)
enemy_mp = game_state.get_resource(MP, 1)
enemy_sp, enemy_mp = game_state.get_resources(1)

# Predict enemy MP 3 turns from now
future_enemy_mp = game_state.project_future_MP(3, player_index=1)
```

### 8.2 Enemy Timing

```python
enemy_turn_time = game_state.enemy_time   # ms the enemy spent on their last turn
```

### 8.3 Enemy Structures on the Map

Iterate over the map and filter for `player_index == 1`:

```python
enemy_turrets = []
for location in game_state.game_map:
    for unit in game_state.game_map[location]:
        if unit.player_index == 1 and unit.unit_type == TURRET:
            enemy_turrets.append(unit)
            gamelib.debug_write(f"Enemy turret at [{unit.x},{unit.y}] HP:{unit.health} upgraded:{unit.upgraded}")
```

### 8.4 Count Enemy Units in a Region

```python
def count_enemy_units(game_state, unit_type=None, valid_x=None, valid_y=None):
    total = 0
    for location in game_state.game_map:
        for unit in game_state.game_map[location]:
            if unit.player_index == 1:
                if unit_type and unit.unit_type != unit_type:
                    continue
                if valid_x and location[0] not in valid_x:
                    continue
                if valid_y and location[1] not in valid_y:
                    continue
                total += 1
    return total

# Count all enemy structures in the front two rows (y=14,15)
front_defences = count_enemy_units(game_state, valid_y=[14, 15])
```

### 8.5 Enemy Units Threatening a Location

```python
# Which enemy turrets can hit a unit at [13, 0]?
attackers = game_state.get_attackers([13, 0], 0)   # player_index=0 = defending as player 0
for a in attackers:
    gamelib.debug_write(f"Attacked by {a.unit_type} at [{a.x},{a.y}]")
```

### 8.6 Enemy Unit Attributes You Can Read

For any `GameUnit` `unit` where `unit.player_index == 1`:

| Attribute        | What it tells you |
|------------------|-------------------|
| `unit.unit_type` | Unit type shorthand (`"FF"`, `"DF"`, etc.) |
| `unit.x`, `unit.y` | Position on the board |
| `unit.health`    | Current HP |
| `unit.max_health` | Base HP (useful to check if it's been damaged) |
| `unit.stationary`| `True` if it's a structure |
| `unit.attackRange` | How far it can attack |
| `unit.damage_f`  | Damage vs structures |
| `unit.damage_i`  | Damage vs mobile units |
| `unit.upgraded`  | Whether it has been upgraded |
| `unit.pending_removal` | Whether the enemy has flagged it for removal |
| `unit.shieldRange` | Effective shield radius (SUPPORT) |

---

## 9. Action Frame Events

Parsed inside `on_action_frame` via `state = json.loads(turn_string)`.  
`state["events"]` is a dict with the following keys:

### 9.1 Event Types

| Key           | Fired when… |
|---------------|-------------|
| `"breach"`    | A mobile unit reaches an enemy edge and scores |
| `"death"`     | A unit is destroyed |
| `"attack"`    | A unit fires an attack |
| `"damage"`    | A unit takes damage |
| `"spawn"`     | A unit is spawned |
| `"move"`      | A mobile unit moves one step |
| `"selfDestruct"` | A mobile unit self-destructs |
| `"shield"`    | A SUPPORT unit applies a shield |
| `"melee"`     | A mobile unit performs a melee (edge) attack |

### 9.2 Event Data Formats

#### `breach` – `[location, unit_type_index, ?, player_damage, owner_index]`

| Index | Value |
|-------|-------|
| 0     | `[x, y]` location of the breach |
| 1     | Unit type index (0–5) |
| 2     | Unused / extra info |
| 3     | HP damage dealt to enemy |
| 4     | `1` = you scored, `2` = opponent scored |

```python
for breach in events["breach"]:
    location        = breach[0]
    damage_dealt    = breach[3]
    scored_by_enemy = (breach[4] == 2)
    if scored_by_enemy:
        self.scored_on_locations.append(location)
```

#### `death` – `[location, unit_type_index, player_index, ?, was_self_destruct]`

| Index | Value |
|-------|-------|
| 0     | `[x, y]` |
| 1     | Unit type index |
| 2     | `1` = your unit, `2` = enemy unit |
| 3     | Unit instance ID |
| 4     | `True` if died via self-destruct |

```python
for death in events["death"]:
    location    = death[0]
    is_enemy    = (death[2] == 2)
    was_selfdes = death[4]
```

#### `attack` – `[attacker_location, target_location, damage, unit_type_index, target_unit_type_index, attacker_player_index]`

| Index | Value |
|-------|-------|
| 0     | `[x, y]` attacker location |
| 1     | `[x, y]` target location |
| 2     | Damage dealt |
| 3     | Attacker unit type index |
| 4     | Target unit type index |
| 5     | Attacker player index (`1` = you, `2` = enemy) |

#### `spawn` – `[location, unit_type_index, player_index]`

| Index | Value |
|-------|-------|
| 0     | `[x, y]` |
| 1     | Unit type index |
| 2     | `1` = your unit, `2` = enemy unit |

#### `move` – `[location, unit_type_index, player_index]`

Same format as `spawn` – emitted each frame a mobile unit steps to a new tile.

#### `damage` – `[location, damage_amount, unit_type_index, player_index]`

| Index | Value |
|-------|-------|
| 0     | `[x, y]` |
| 1     | Damage amount |
| 2     | Unit type index |
| 3     | `1` = your unit was damaged, `2` = enemy unit was damaged |

#### `selfDestruct` – `[location, damage, unit_type_index, player_index, affected_player_index]`

#### `shield` – `[location, shield_amount, unit_type_index, player_index]`

#### `melee` – Same format as `attack`

---

## 10. Game Configuration Values

`self.config` / `game_state.config` is a Python dict matching `game-configs.json`.

### Key Paths

| Path | Value | Description |
|------|-------|-------------|
| `config["unitInformation"][i]` | dict | Stats for unit type `i` (0=WALL,1=SUPPORT,2=TURRET,3=SCOUT,4=DEMOLISHER,5=INTERCEPTOR,6=REMOVE,7=UPGRADE) |
| `config["unitInformation"][i]["shorthand"]` | str | The constant string for that unit |
| `config["unitInformation"][i]["startHealth"]` | float | Base HP |
| `config["unitInformation"][i]["cost1"]` | float | SP cost |
| `config["unitInformation"][i]["cost2"]` | float | MP cost |
| `config["unitInformation"][i]["attackRange"]` | float | Attack radius |
| `config["unitInformation"][i]["attackDamageWalker"]` | float | Damage to mobile units |
| `config["unitInformation"][i]["attackDamageTower"]` | float | Damage to structures |
| `config["unitInformation"][i]["speed"]` | float | Movement speed |
| `config["unitInformation"][i]["upgrade"]` | dict | Upgraded stats (overrides) |
| `config["resources"]["startingHP"]` | float | Starting health (`40.0`) |
| `config["resources"]["startingBits"]` | float | Starting MP (`5.0`) |
| `config["resources"]["startingCores"]` | float | Starting SP (`40.0`) |
| `config["resources"]["bitsPerRound"]` | float | MP gained per turn (`5.0`) |
| `config["resources"]["coresPerRound"]` | float | SP gained per turn (`5.0`) |
| `config["resources"]["bitDecayPerRound"]` | float | Fraction of MP lost each turn (`0.25`) |
| `config["resources"]["bitGrowthRate"]` | float | Additional MP gain ramp-up per interval (`1.0`) |
| `config["resources"]["turnIntervalForBitSchedule"]` | int | Turns between MP ramp-ups (`10`) |

---

## 11. Full Usage Example

Below is an annotated strategy that demonstrates all major API surfaces:

```python
import gamelib
import json
import math
import random
from sys import maxsize

class AlgoStrategy(gamelib.AlgoCore):

    def __init__(self):
        super().__init__()
        self.scored_on_locations = []

    # ------------------------------------------------------------------ #
    #  Called once at game start                                           #
    # ------------------------------------------------------------------ #
    def on_game_start(self, config):
        self.config = config
        global WALL, SUPPORT, TURRET, SCOUT, DEMOLISHER, INTERCEPTOR, MP, SP
        WALL        = config["unitInformation"][0]["shorthand"]
        SUPPORT     = config["unitInformation"][1]["shorthand"]
        TURRET      = config["unitInformation"][2]["shorthand"]
        SCOUT       = config["unitInformation"][3]["shorthand"]
        DEMOLISHER  = config["unitInformation"][4]["shorthand"]
        INTERCEPTOR = config["unitInformation"][5]["shorthand"]
        MP = 1
        SP = 0

    # ------------------------------------------------------------------ #
    #  Called every turn                                                   #
    # ------------------------------------------------------------------ #
    def on_turn(self, turn_state):
        game_state = gamelib.GameState(self.config, turn_state)
        game_state.suppress_warnings(True)

        # ---- READING YOUR STATE ----
        turn  = game_state.turn_number
        my_hp = game_state.my_health
        my_sp = game_state.get_resource(SP)
        my_mp = game_state.get_resource(MP)
        gamelib.debug_write(f"Turn {turn}: HP={my_hp} SP={my_sp} MP={my_mp}")

        # ---- READING ENEMY STATE ----
        enemy_hp = game_state.enemy_health
        enemy_sp = game_state.get_resource(SP, 1)
        enemy_mp = game_state.get_resource(MP, 1)
        future_enemy_mp = game_state.project_future_MP(3, player_index=1)
        gamelib.debug_write(f"Enemy HP={enemy_hp} SP={enemy_sp} MP={enemy_mp} MP+3turns={future_enemy_mp}")

        # ---- ANALYSE ENEMY DEFENCES ----
        enemy_turrets = []
        enemy_walls   = []
        for location in game_state.game_map:
            for unit in game_state.game_map[location]:
                if unit.player_index == 1:
                    if unit.unit_type == TURRET:
                        enemy_turrets.append(unit)
                    elif unit.unit_type == WALL:
                        enemy_walls.append(unit)
        gamelib.debug_write(f"Enemy has {len(enemy_turrets)} turrets and {len(enemy_walls)} walls")

        # ---- BUILD DEFENCES ----
        turret_locs = [[0, 13], [27, 13], [8, 11], [19, 11]]
        game_state.attempt_spawn(TURRET, turret_locs)

        wall_locs = [[8, 12], [19, 12]]
        game_state.attempt_spawn(WALL, wall_locs)
        game_state.attempt_upgrade(wall_locs)

        # React to where we were scored on last
        for loc in self.scored_on_locations:
            game_state.attempt_spawn(TURRET, [loc[0], loc[1] + 1])

        # ---- CHOOSE ATTACK STRATEGY ----
        # Find the safest edge to spawn from
        spawn_options = [[13, 0], [14, 0]]
        damages = []
        for loc in spawn_options:
            path   = game_state.find_path_to_edge(loc)
            damage = sum(
                len(game_state.get_attackers(step, 0)) *
                gamelib.GameUnit(TURRET, game_state.config).damage_i
                for step in path
            )
            damages.append(damage)
        best_spawn = spawn_options[damages.index(min(damages))]

        # Count enemy front-line defences
        front_count = sum(
            1
            for loc in game_state.game_map
            for unit in game_state.game_map[loc]
            if unit.player_index == 1 and loc[1] in [14, 15]
        )

        if front_count > 10:
            # Use demolishers behind a wall line for long-range attack
            # range(27, 5, -1) places walls at x=27 down to x=6 (5 is excluded)
            for x in range(27, 5, -1):
                game_state.attempt_spawn(WALL, [x, 11])
            game_state.attempt_spawn(DEMOLISHER, [24, 10], 1000)
        else:
            # Spam scouts from the safest edge
            game_state.attempt_spawn(SCOUT, best_spawn, 1000)

        # ---- END TURN ----
        game_state.submit_turn()

    # ------------------------------------------------------------------ #
    #  Called for every action frame (fast path – avoid heavy logic here) #
    # ------------------------------------------------------------------ #
    def on_action_frame(self, turn_string):
        state  = json.loads(turn_string)
        events = state["events"]

        # Track breach locations (where enemy scored on us)
        for breach in events["breach"]:
            location    = breach[0]
            owner_index = breach[4]  # 1=us, 2=enemy
            if owner_index == 2:
                gamelib.debug_write(f"Scored on at {location}")
                self.scored_on_locations.append(location)

        # Track our unit deaths
        for death in events["death"]:
            location  = death[0]
            is_us     = (death[2] == 1)
            self_dest = death[4]
            if is_us:
                gamelib.debug_write(f"We lost a unit at {location} (self-destruct={self_dest})")


if __name__ == "__main__":
    algo = AlgoStrategy()
    algo.start()
```

---

*This documentation was generated from the C1 Games Terminal StarterKit source files:*  
*`gamelib/game_state.py`, `gamelib/game_map.py`, `gamelib/unit.py`,*  
*`gamelib/algocore.py`, `gamelib/navigation.py`, `gamelib/util.py`, and `game-configs.json`.*
