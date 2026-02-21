# Ground-Targeted Skills — Implementation Guide

Covers the ground-targeted spell archetype using `WZ_METEOR` (Meteor Storm, Wizard) as the reference implementation.

Ground-targeted skills require the player to select a map cell as the target (not an entity). The server then places one or more **skill units** on the map that deal damage or apply effects over time to anything that enters or stands within their range.

---

## Architecture Overview

```
Player clicks ground position
        ↓
skill_castend_pos2()        ← entry point for all ground-targeted skills
        ↓
skill_unitsetting()         ← creates s_skill_unit_group + skill_unit[] on the map
        ↓
skill_unit_timer()          ← fires every 100ms, manages all active skill units
        ↓
skill_unit_timer_sub()      ← per-unit: checks lifetime, triggers state transitions
        ↓
skill_unit_timer_sub_onplace()  ← iterates all targets standing in unit's range
        ↓
skill_unit_onplace_timer()  ← checks tick interval, calls skill_attack() per target
        ↓
skill_attack()              ← applies damage via battle_calc_attack()
```

---

## 1. Skill Database (`db/re/skill_db.yml`)

`TargetType: Ground` is the defining property. The `Unit:` block configures the on-map skill unit.

### WZ_METEOR Full Entry (abbreviated)

```yaml
- Id: 83
  Name: WZ_METEOR
  Description: Meteor Storm
  MaxLevel: 10
  Type: Magic
  TargetType: Ground          # Player selects a ground position
  Flags:
    IsAutoShadowSpell: true
  Range: 9                    # Cast range from caster to target cell
  Hit: Multi_Hit
  HitCount:                   # Number of individual meteors per cast
    - Level: 1
      Count: 2
    - Level: 5
      Count: 4
    - Level: 10
      Count: 7
  Element: Fire
  SplashArea:                 # Impact radius around each meteor's landing cell
    - Level: 1-10
      Area: 3
  CastTime: 6300              # Variable cast time (ms)
  FixedCastTime: 1500         # Fixed cast time added on top (ms)
  AfterCastActDelay: 1000
  Duration1:                  # Total unit lifetime (ms)
    - Level: 1
      Time: 2000
    - Level: 5
      Time: 4000
    - Level: 10
      Time: 7000
  Cooldown:
    - Level: 5
      Time: 4500
  Requires:
    SpCost:
      - Level: 5
        Amount: 40
      - Level: 10
        Amount: 64
  Unit:
    Id: Dummyskill            # No visible ground decal; UNT_DUMMYSKILL (0x86)
    Range: 3                  # Damage radius around each unit cell
    Interval: 1000            # How often (ms) units can deal damage per target
    Target: Enemy
    Flag:
      PathCheck: true         # Only place units on cells with a line-of-sight path to caster
```

### Key `Unit:` Sub-fields

| Field | Effect |
|-------|--------|
| `Id` | Selects the `UNT_*` enum value. `Dummyskill` = no ground sprite. Use named IDs (e.g. `Quagmire`, `FireWall`) for visible ground effects. |
| `Range` | Radius in cells around each unit tile where damage is checked. |
| `Interval` | Minimum time (ms) between hits on the same target within this group. |
| `Target` | Who the unit can hit: `Enemy`, `Friend`, `All`, `Sameguild`, etc. |
| `Flag/PathCheck` | Skips cells that have no walkable line of sight to the caster. |
| `Flag/NoOverlap` | Prevents two groups of this skill from stacking on the same cell. |

---

## 2. Entry Point — `skill_castend_pos2()` (`src/map/skill.cpp`)

Called when a ground-targeted skill finishes casting. For most ground skills, the work is delegated immediately to `skill_unitsetting()`.

```cpp
int32 skill_castend_pos2(block_list* src, int32 x, int32 y,
                         uint16 skill_id, uint16 skill_lv, t_tick tick, int32 flag)
{
    // WZ_METEOR does NOT send the position effect packet here.
    // The packet is deferred to the moment each meteor drops (in the timer).
    switch (skill_id) {
        case WZ_METEOR:
        // ... other deferred-effect skills ...
            break;
        default:
            clif_skill_poseffect(*src, skill_id, skill_lv, x, y, tick);
    }

    // ... (other skill-specific setup) ...

    // WZ_METEOR passes the duration via `flag` to skill_unitsetting.
    // For most ground skills, skill_unitsetting is called here directly.
}
```

For WZ_METEOR specifically, each meteor is placed as a separate call to `skill_unitsetting()` (one per meteor count from `HitCount`), with a random offset from the target cell and a staggered start time to create the "shower" effect.

---

## 3. Skill Unit Creation — `skill_unitsetting()` (`src/map/skill.cpp`)

Creates the `s_skill_unit_group` (the container) and each individual `skill_unit` tile on the map.

```cpp
std::shared_ptr<s_skill_unit_group> skill_unitsetting(
    block_list *src,      // Caster
    uint16 skill_id,      // WZ_METEOR
    uint16 skill_lv,
    int16 x, int16 y,     // Target cell (center of the unit group)
    int32 flag)           // For WZ_METEOR: duration override in ms
{
    // Duration comes from skill_db.yml Duration1, but WZ_METEOR overrides it:
    case WZ_METEOR:
        limit = flag;   // `flag` carries the per-meteor lifetime
        flag = 0;
        break;

    // Creates s_skill_unit_group with:
    //   group->limit    = lifetime of the group
    //   group->interval = 1000ms (from Unit/Interval in skill_db.yml)
    //   group->unit_id  = UNT_DUMMYSKILL
    //   group->val2     = 0  (falling phase — see §5 below)

    // Then places individual skill_unit tiles according to the layout,
    // checking PathCheck and wall passability per cell.
}
```

### Data Structures (`src/map/skill.hpp`)

```cpp
// The group: one per skill cast
struct s_skill_unit_group {
    int32  src_id;        // Caster ID
    t_tick limit;         // Total lifetime
    int32  interval;      // Damage tick interval
    uint16 skill_id;
    uint16 skill_lv;
    int32  val1, val2, val3;  // val2 = meteor drop state (0=falling, 1=impact)
    int32  unit_id;       // UNT_DUMMYSKILL for WZ_METEOR
    skill_unit *unit;     // Array of individual cells
    int32  unit_count;
    int32  alive_count;
};

// One tile on the map
struct skill_unit : public block_list {
    std::shared_ptr<s_skill_unit_group> group;
    t_tick limit;         // Individual tile lifetime (can differ from group)
    int32  val1, val2;
    int16  range;         // Damage radius of this tile
    bool   alive;
    bool   hidden;
};
```

---

## 4. Unit ID Enum (`src/map/skill.hpp`)

```cpp
enum e_skill_unit_id : uint16 {
    UNT_SAFETYWALL     = 0x7e,
    UNT_FIREWALL,
    UNT_WARP_WAITING,
    UNT_WARP_ACTIVE,
    UNT_BENEDICTIO,
    UNT_SANCTUARY,
    UNT_DUMMYSKILL     = 0x86,  // ← WZ_METEOR, WZ_STORMGUST, etc. (no sprite)
    UNT_FIREPILLAR_WAITING,
    UNT_FIREPILLAR_ACTIVE,
    // ...
    UNT_QUAGMIRE       = 0xa0,  // Has visible ground sprite
    // ...
};
```

If a skill should have a visible ground decal, use a named `UNT_*` that the client recognises. The client maps each `UNT_*` value to a sprite/animation.

---

## 5. Damage Timer — State Machine for WZ_METEOR

WZ_METEOR uses a **two-phase state machine** stored in `group->val2`:

```
val2 = 0  →  Falling phase: unit is alive but dealing no damage.
val2 = 1  →  Impact phase:  next timer tick applies damage, then unit is deleted.
```

### `skill_unit_timer_sub()` — runs every 100ms per unit

```cpp
// Phase transition: when lifetime is almost up, switch to impact
if (group->val2 == 0 &&
    DIFF_TICK(tick, group->tick) >= group->limit - group->interval)
{
    // Send meteor drop visual to clients
    clif_skill_poseffect(*src, group->skill_id, group->skill_lv,
                         bl->x, bl->y, tick);

    group->val2 = 1;  // Transition to impact phase

    // Adjust remaining lifetime: impact happens 700ms after the drop animation
    group->limit = group->limit - group->interval + 700;
}

// During falling phase, no damage is dealt
if (group->val2 == 0)
    return 0;

// Impact phase: allow damage application (falls through to normal unit timer)
```

### `skill_unit_timer_sub_onplace()` — iterates targets in range

```cpp
// Called for every entity standing on or in range of a unit tile.
// Uses map_foreachinallarea() with the unit's range (3 for WZ_METEOR).
int32 skill_unit_timer_sub_onplace(block_list* bl, va_list ap)
{
    skill_unit* unit = va_arg(ap, skill_unit *);
    t_tick tick = va_arg(ap, t_tick);

    if (!unit->alive || bl->prev == nullptr)
        return 0;

    if (battle_check_target(unit, bl, group->target_flag) <= 0)
        return 0;

    skill_unit_onplace_timer(unit, bl, tick);
    return 1;
}
```

### `skill_unit_onplace_timer()` — per-target damage

```cpp
int32 skill_unit_onplace_timer(skill_unit *unit, block_list *bl, t_tick tick)
{
    // Tickset check: prevents hitting the same target more than once per `interval`
    if ((ts = skill_unitgrouptickset_search(bl, sg, tick))) {
        diff = DIFF_TICK(tick, ts->tick);
        if (diff < 0)
            return 0;               // Too soon since last hit on this target
        ts->tick = tick + sg->interval;  // Schedule next allowed hit (+1000ms)
    }

    switch (sg->unit_id) {
        case UNT_DUMMYSKILL:        // WZ_METEOR falls here
            // Default: caster is the damage source
            skill_attack(skill_get_type(sg->skill_id), ss, unit, bl,
                         sg->skill_id, sg->skill_lv, tick, 0);
            break;
    }
}
```

Note: `dsrc` (second source param) is the `skill_unit`, not the caster. This signals to `skill_attack()` that damage comes from a placed unit, not direct player action (sets `dmg.amotion = 0`).

---

## 6. Client Communication (`src/map/clif.cpp`)

### Ground Skill Packet — `clif_skill_poseffect()`

Notifies the client that a ground-targeted skill fired at a position. Used both for initial cast effects and for per-meteor drop animations.

```cpp
void clif_skill_poseffect(block_list& bl,   // Caster
                          uint16 skill_id,
                          uint16 skill_lv,
                          uint16 x, uint16 y,   // Target cell
                          t_tick tick)
{
    PACKET_ZC_NOTIFY_GROUNDSKILL packet{};
    packet.PacketType = HEADER_ZC_NOTIFY_GROUNDSKILL;
    packet.SKID      = skill_id;
    packet.AID       = bl.id;
    packet.level     = skill_lv;
    packet.xPos      = x;
    packet.yPos      = y;
    packet.startTime = client_tick(tick);

    clif_send(&packet, sizeof(packet), &bl, AREA);  // Broadcast to all nearby clients
}
```

This is the packet that triggers the client-side meteor drop animation. For WZ_METEOR it is called once per meteor at the moment of impact (val2 transition), not at the start of the cast.

---

## 7. Client-Side Files

### Visual Effects

Ground skill animations are driven by the client's internal effect table, keyed by `skill_id`. No Lua changes are typically needed for the ground effect itself — the client looks up the effect by skill ID from its own data files (inside `data.grf` or `data/`).

### Skill Info Tooltip — `data/luafiles514/lua files/skillinfoz/`

Add or update the skill's tooltip entry here so the description shows in-game. The file format uses the skill's numeric ID.

---

## 8. Adding a New Ground-Targeted Skill

### Step 1 — `db/re/skill_db.yml`

```yaml
- Id: <NEW_ID>
  Name: XX_NEWGROUND
  Description: New Ground Skill
  MaxLevel: 5
  Type: Magic
  TargetType: Ground          # Required for ground targeting
  Range: 9
  Element: Fire
  CastTime: 2000
  FixedCastTime: 500
  Duration1:
    - Level: 1
      Time: 3000
    - Level: 5
      Time: 7000
  Requires:
    SpCost:
      - Level: 1
        Amount: 30
  Unit:
    Id: Dummyskill            # No sprite, or use a named UNT_* for a visible one
    Range: 2                  # Damage radius per tile
    Interval: 1000            # Ms between damage ticks per target
    Target: Enemy
    Flag:
      PathCheck: true
```

### Step 2 — Handle in `skill_castend_pos2()` (`src/map/skill.cpp`)

For most cases, call `skill_unitsetting()` directly. Add any pre-placement logic here.

```cpp
case XX_NEWGROUND:
    skill_unitsetting(src, skill_id, skill_lv, x, y, 0);
    break;
```

If the skill needs a deferred position effect (like WZ_METEOR), add it to the switch that suppresses `clif_skill_poseffect()` at the top of `skill_castend_pos2()` and send it manually in the timer.

### Step 3 — Handle damage in `skill_unit_onplace_timer()` (`src/map/skill.cpp`)

The default `UNT_DUMMYSKILL` path calls `skill_attack()` automatically. Only add a specific `case` if special behaviour is needed (e.g. stacking counters, conditional damage).

### Step 4 — Add to `skill_unitsetting()` (if needed)

If val1/val2/val3 carry special meaning for the new skill, add a case in the `switch(skill_id)` block inside `skill_unitsetting()` to initialize them.

---

## Comparison: WZ_METEOR vs Other Ground Skills

| Property | WZ_METEOR | WZ_STORMGUST | WZ_QUAGMIRE |
|----------|-----------|--------------|-------------|
| `Unit/Id` | Dummyskill | Dummyskill | Quagmire (visible) |
| `Unit/Interval` | 1000ms | 450ms | -1 (one-time on enter) |
| `Unit/Flag/NoOverlap` | No | Yes | No |
| Delayed impact | Yes (val2 state) | No | No |
| Damage per tick | Fire magic | Water magic | None (status only) |
| Status effect | Stun (chance) | Freeze (chance) | SC_QUAGMIRE (speed debuff) |
| Multiple units | Yes (one per meteor) | No (one group) | No |

---

## Key Functions Reference

| Function | File | Purpose |
|----------|------|---------|
| `skill_castend_pos2(src, x, y, id, lv, tick, flag)` | `skill.cpp` | Entry point for ground skills |
| `skill_unitsetting(src, id, lv, x, y, flag)` | `skill.cpp` | Place skill unit group on map |
| `skill_unit_timer()` | `skill.cpp` | Global timer, runs every 100ms |
| `skill_unit_timer_sub(bl, ap)` | `skill.cpp` | Per-unit lifetime management |
| `skill_unit_timer_sub_onplace(bl, ap)` | `skill.cpp` | Iterate targets in unit range |
| `skill_unit_onplace_timer(unit, bl, tick)` | `skill.cpp` | Apply damage per target |
| `skill_delunit(unit)` | `skill.cpp` | Remove an individual skill unit |
| `skill_delunitgroup(group)` | `skill.cpp` | Remove entire skill unit group |
| `skill_attack(type, src, dsrc, bl, id, lv, tick, flag)` | `skill.cpp` | Deal skill damage |
| `clif_skill_poseffect(bl, id, lv, x, y, tick)` | `clif.cpp` | Send ground skill packet to client |
| `map_foreachinallarea(func, m, x0, y0, x1, y1, type, ...)` | `map.cpp` | Iterate targets in rectangular area |
| `skill_unitgrouptickset_search(bl, group, tick)` | `skill.cpp` | Per-target hit interval enforcement |
