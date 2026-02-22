# Targeted Skills — Implementation Guide

Covers two targeted skill archetypes:
- **AoE melee with knockback** — `KN_BOWLINGBASH` (Bowling Bash, Knight)
- **Single-target magic multi-hit** — `MG_FIREBOLT` (Fire Bolt, Mage/Wizard)

Targeted skills use `TargetType: Attack` and require the player to select a **specific entity** as the target. The skill then deals damage to that entity and optionally to nearby entities via `SplashArea`.

---

## Architecture Overview

```
Player selects target entity
        ↓
skill_castend_damage_id()         ← entry point for all damaging targeted skills
        ↓
skill->impl->castendDamageId()    ← C++ skill implementation (or default case)
        ↓
skill_attack(BF_WEAPON or BF_MAGIC, ...)   ← damage pipeline
        ↓
battle_calc_attack()              ← dispatches to weapon or magic calculator
   ├─ battle_calc_weapon_attack() ← for physical skills (KN_BOWLINGBASH)
   └─ battle_calc_magic_attack()  ← for magic skills (MG_FIREBOLT)
        ↓
clif_skill_damage()               ← packet to all nearby clients
        ↓
(AoE only) map_foreachinallarea() + skill_area_sub()   ← splash damage
        ↓
(knockback) skill_blown()         ← moves target cells away
```

---

## 1. Skill Database (`db/re/skill_db.yml`)

### KN_BOWLINGBASH

```yaml
- Id: 62
  Name: KN_BOWLINGBASH
  Description: Bowling Bash
  MaxLevel: 10
  Type: Weapon               # Physical melee damage
  TargetType: Attack         # Entity target
  Flags:
    TargetTrap: true
  Range: -2                  # Melee range (-1 = 1 cell, -2 = 2 cells)
  Hit: Single
  HitCount: 2                # Two hits per cast (div_ = 2)
  Element: Weapon            # Inherits the weapon's element
  SplashArea: 2              # 5×5 cell splash (2 cells in each direction)
  Knockback:
    - Level: 1
      Amount: 1              # Cells knocked back
    - Level: 5
      Amount: 3
    - Level: 10
      Amount: 5
  CopyFlags:
    Skill:
      Plagiarism: true
      Reproduce: true
  AfterCastActDelay: 300
  Duration1: 1000
  Cooldown: 1000
  FixedCastTime: 350
  Requires:
    SpCost:
      - Level: 1
        Amount: 13
      - Level: 10
        Amount: 22
```

### MG_FIREBOLT

```yaml
- Id: 19
  Name: MG_FIREBOLT
  Description: Fire Bolt
  MaxLevel: 10
  Type: Magic                # Magical damage
  TargetType: Attack         # Entity target
  Flags:
    IsAutoShadowSpell: true  # Can be copied by Shadow Spell
  Range: 9                   # Long cast range
  Hit: Multi_Hit             # Multiple separate hits
  HitCount:
    - Level: 1
      Count: 1               # 1 bolt at Lv1
    - Level: 5
      Count: 5
    - Level: 10
      Count: 10              # 10 bolts at Lv10
  Element: Fire
  CopyFlags:
    Skill:
      Plagiarism: true
      Reproduce: true
  CastTime:
    - Level: 1
      Time: 500
    - Level: 10
      Time: 3200
  AfterCastActDelay: 1400
  FixedCastTime:
    - Level: 1
      Time: 300
    - Level: 10
      Time: 1200
  Requires:
    SpCost:
      - Level: 1
        Amount: 12
      - Level: 10
        Amount: 30
```

### Critical Database Field Differences

| Field | KN_BOWLINGBASH | MG_FIREBOLT |
|-------|---------------|-------------|
| `Type` | `Weapon` | `Magic` |
| `Hit` | `Single` | `Multi_Hit` |
| `HitCount` | 2 (fixed) | 1–10 (scales with level) |
| `SplashArea` | 2 (5×5 AoE) | — (single target) |
| `Knockback` | 1–5 cells | — |
| `Element` | `Weapon` | `Fire` |
| `Range` | -2 (melee) | 9 (ranged) |

---

## 2. Skill Entry Point — `skill_castend_damage_id()` (`src/map/skill.cpp`)

This is the main dispatcher for all damaging targeted skills. Most skills, including both of these, use the **database-driven default path**:

```cpp
int32 skill_castend_damage_id(block_list *src, block_list *bl,
    uint16 skill_id, uint16 skill_lv, t_tick tick, int32 flag)
{
    // ...
    switch (skill_id) {
        // Skills with custom logic have explicit cases here.
        // KN_BOWLINGBASH and MG_FIREBOLT fall through to:

        default:
            if (auto skill = skill_db.find(skill_id);
                skill != nullptr && skill->impl != nullptr)
            {
                skill->impl->castendDamageId(src, bl, skill_lv, tick, flag);
                break;
            }
    }
}
```

The implementation class calls `skill_attack()` with the appropriate battle flag.

---

## 3. Damage Pipeline — `skill_attack()` (`src/map/skill.cpp`)

```cpp
int64 skill_attack(int32 attack_type,   // BF_WEAPON or BF_MAGIC
                   block_list *src,     // Caster
                   block_list *dsrc,    // Actual hit source (same as src for direct attacks)
                   block_list *bl,      // Target
                   uint16 skill_id,
                   uint16 skill_lv,
                   t_tick tick,
                   int32 flag)
{
    Damage dmg = battle_calc_attack(attack_type, src, bl, skill_id, skill_lv, flag & 0xFFF);

    // If dsrc != src (ground unit), amotion is cleared (instant hit)
    if (src != dsrc)
        dmg.amotion = 0;

    // Apply damage, status effects, cards, etc.
    // Send clif_skill_damage() to clients
}
```

---

## 4. Damage Calculation (`src/map/battle.cpp`)

### Dispatcher — `battle_calc_attack()`

```cpp
struct Damage battle_calc_attack(int32 attack_type, block_list *src,
    block_list *target, uint16 skill_id, uint16 skill_lv, int32 flag)
{
    switch (attack_type) {
        case BF_WEAPON: return battle_calc_weapon_attack(src, target, skill_id, skill_lv, flag);
        case BF_MAGIC:  return battle_calc_magic_attack(src, target, skill_id, skill_lv, flag);
        case BF_MISC:   return battle_calc_misc_attack(src, target, skill_id, skill_lv, flag);
    }
}
```

---

### Physical Path — `battle_calc_weapon_attack()` (KN_BOWLINGBASH)

Key steps inside the function:

```cpp
// 1. Initialize Damage struct
wd = initialize_weapon_data(src, target, skill_id, skill_lv, wflag);
// Sets wd.div_ = HitCount (2 for Bowling Bash)

// 2. Determine weapon element (Weapon element = inherits equipped weapon)
right_element = battle_get_weapon_element(&wd, src, target, skill_id, skill_lv, EQI_HAND_R, false);

// 3. Hit check
if (!is_attack_hitting(&wd, src, target, skill_id, skill_lv, true))
    wd.dmg_lv = ATK_FLEE;
else {
    // 4. Base damage from ATK stat
    battle_calc_skill_base_damage(&wd, src, target, skill_id, skill_lv);

    // 5. Apply skill damage ratio (from battle_config or skill formula)
    ATK_RATE(wd.damage, wd.damage2,
             battle_calc_attack_skill_ratio(&wd, src, target, skill_id, skill_lv));

    // 6. Additive bonuses (cards, equip, etc.)
    ATK_ADD(wd.damage, wd.damage2,
            battle_calc_skill_constant_addition(&wd, src, target, skill_id, skill_lv));

    // 7. Card modifiers
    wd.weaponAtk += battle_calc_cardfix(BF_WEAPON, src, target, nk,
                                        right_element, left_element, wd.weaponAtk, 2, wd.flag);
}
```

`div_ = 2` means two separate damage numbers are shown on the target. Each hit is independent — it can miss, critical, or be blocked separately.

---

### Magic Path — `battle_calc_magic_attack()` (MG_FIREBOLT)

```cpp
// 1. div_ = number of bolts (HitCount from skill_db.yml, 1-10)
ad.div_ = skill_get_num(skill_id, skill_lv);

// 2. Element for the skill
s_ele = battle_get_magic_element(src, target, skill_id, skill_lv, mflag);
// For MG_FIREBOLT: Fire element

// 3. Base MATK-based damage
ad.damage = base_matk * skill_ratio / 100;

// 4. Magic defense reduction
ad.damage = battle_calc_magic_defense(ad.damage, tstatus->mdef, src, target, skill_id, skill_lv);

// 5. Element table lookup (Fire vs target's element/race)
ad.damage = battle_attr_fix(src, target, ad.damage, s_ele, def_ele, ele_lv);
```

`div_` for MG_FIREBOLT equals the skill level (10 bolts at Lv10). All bolts hit the same target, each doing the same elemental magic damage.

---

## 5. AoE Splash (KN_BOWLINGBASH)

### `map_foreachinallarea()` — target iteration (`src/map/map.cpp`)

```cpp
// Iterates all block_lists in a rectangular area, calls `func` on each.
int32 map_foreachinallarea(int32 (*func)(block_list*, va_list),
    int16 m,
    int16 x0, int16 y0,   // Top-left corner
    int16 x1, int16 y1,   // Bottom-right corner
    int32 type,           // BL_CHAR, BL_MOB, BL_PC, etc.
    ...);                 // Extra args passed to func
```

For `SplashArea: 2`, the call area is `(target_x - 2, target_y - 2)` to `(target_x + 2, target_y + 2)` — a 5×5 cell square.

### `skill_area_sub()` — per-target validity check (`src/map/skill.cpp`)

```cpp
int32 skill_area_sub(block_list *bl, va_list ap)
{
    block_list *src   = va_arg(ap, block_list *);
    uint16 skill_id   = va_arg(ap, int32);
    uint16 skill_lv   = va_arg(ap, int32);
    t_tick tick       = va_arg(ap, t_tick);
    int32  flag       = va_arg(ap, int32);
    SkillFunc func    = va_arg(ap, SkillFunc);  // Callback

    if (battle_check_target(src, bl, flag) > 0) {
        return func(src, bl, skill_id, skill_lv, tick, flag);
        // For splash damage: func = skill_castend_damage_id
    }
    return 0;
}
```

The `flag` contains `BCT_ENEMY` to limit hits to hostile targets. Splash damage from `skill_castend_damage_id` goes through the full damage pipeline for each entity in range.

---

## 6. Knockback — `skill_blown()` (`src/map/skill.cpp`)

Called after damage for skills with `Knockback` defined in `skill_db.yml`.

```cpp
int16 skill_blown(block_list *src, block_list *target,
                  char count,   // Cells to knock back (1-5 for KN_BOWLINGBASH)
                  int8 dir,     // Direction (0-7, or -1 to auto-calculate)
                  enum e_skill_blown flag)
{
    // Auto-direction: away from caster
    if (dir == -1)
        dir = map_calc_dir(target, src->x, src->y);

    // dirx[dir] / diry[dir] = unit vector for each of the 8 directions
    int32 dx = -dirx[dir];
    int32 dy = -diry[dir];

    // Cancel certain statuses on knockback
    if (tsc) {
        if (tsc->getSCE(SC_SU_STOOP))      status_change_end(target, SC_SU_STOOP);
        if (tsc->getSCE(SC_ROLLINGCUTTER)) status_change_end(target, SC_ROLLINGCUTTER);
        // ... others
        if (tsc->getSCE(SC_SV_ROOTTWIST))  return 0; // Root: immune to knockback
    }

    return unit_blown(target, dx, dy, count, flag);  // Applies the actual movement
}
```

Knockback immunity is checked via `unit_blown_immune()`. Maps with `mapflag: noknockback`, monsters with the `MD_KNOCKBACKIMMUNE` mode, and skills under certain statuses are all immune.

### Chain Knockback (MS_BOWLINGBASH variant)

The Summoner's version (`MS_BOWLINGBASH`) demonstrates the recursive chain hit logic used as a reference model for custom chain-bounce skills:

```cpp
case MS_BOWLINGBASH: {
    int32 c = (skill_lv - (flag & 0xFFF) + 1) / 2;  // Bounce range shrinks with recursion depth

    // Knock target back one cell at a time
    for (int32 i = 0; i < c; i++) {
        skill_blown(src, bl, 1, dir, BLOWN_NONE);

        // After each cell, check for enemies in 3×3 area
        int32 count = map_foreachinallarea(skill_area_sub, bl->m,
            tx-1, ty-1, tx+1, ty+1,
            splash_target(src),
            src, skill_id, skill_lv, tick, flag | BCT_ENEMY, skill_area_sub_count);

        if (count) {
            // Recursively trigger Bowling Bash on each hit enemy (flag incremented)
            map_foreachinallarea(skill_area_sub, bl->m,
                tx-1, ty-1, tx+1, ty+1,
                splash_target(src),
                src, skill_id, skill_lv, tick, (flag | BCT_ENEMY) + 1,
                skill_castend_damage_id);
            break;
        }
    }

    idb_put(bowling_db, bl->id, bl);     // Track hit targets (prevents double-hit)
    skill_attack(BF_WEAPON, src, src, bl, skill_id, skill_lv, tick, flag > 0 ? SD_ANIMATION : 0);
}
```

`bowling_db` is a temporary hash table cleared at the start of each top-level cast. `flag & 0xFFF` encodes recursion depth; `SD_ANIMATION` flag plays the hit animation on recursive targets.

---

## 7. Post-Damage Effects — Doublecast (MG_FIREBOLT)

After `skill_attack()` returns, certain skills trigger secondary effects:

```cpp
// In skill_attack(), after damage is applied:
if (!(flag & 2)) {
    switch (skill_id) {
        case MG_COLDBOLT:
        case MG_FIREBOLT:
        case MG_LIGHTNINGBOLT:
            // If Doublecast is active and RNG succeeds, queue a second cast
            if (sc && sc->getSCE(SC_DOUBLECAST) &&
                rnd() % 100 < sc->getSCE(SC_DOUBLECAST)->val2)
            {
                skill_addtimerskill(src, tick + dmg.amotion, bl->id,
                                    0, 0, skill_id, skill_lv, BF_MAGIC, flag | 2);
            }
            break;
    }
}
```

`flag | 2` prevents the secondary cast from triggering another Doublecast chain. `skill_addtimerskill()` queues the second cast to fire `amotion` ms after the first, so it appears as a delayed re-cast.

---

## 8. Client Communication (`src/map/clif.cpp`)

### `clif_skill_damage()` — primary damage packet

Sent for every damaging targeted skill hit.

```cpp
void clif_skill_damage(const block_list& src, const block_list& dst,
    t_tick tick, int32 sdelay, int32 ddelay,
    int64 sdamage, int16 div,        // div = HitCount
    uint16 skill_id, uint16 skill_lv, e_damage_type type)
{
    PACKET_ZC_NOTIFY_SKILL packet{};
    packet.SKID      = skill_id;
    packet.AID       = src.id;       // Attacker
    packet.targetID  = dst.id;       // Victim
    packet.damage    = sdamage;      // Total damage value
    packet.count     = div;          // Number of hits shown (2 for BB, 1-10 for Firebolt)
    packet.action    = type;         // DMG_SINGLE, DMG_MULTI_HIT, DMG_CRITICAL, etc.
    packet.startTime = client_tick(tick);

    clif_send(&packet, sizeof(packet), &dst, AREA);
}
```

The `count` (div) field tells the client how many hit numbers to render. For KN_BOWLINGBASH it is 2; for MG_FIREBOLT it equals the bolt count (1–10).

The client automatically picks the correct skill animation based on `skill_id`.

---

## 9. Client-Side Files

### Skill Info Tooltip — `data/luafiles514/lua files/skillinfoz/`

Each skill's tooltip text and icon path are defined here in `.lub` files keyed by skill ID. Add or update the entry when adding a new skill.

### Skill Icon / Effect

Skill projectile and hit animations are resolved client-side via sprite files packed in the GRF archives. No source changes needed for existing animation types; for entirely new visual effects, new sprite/effect files must be added to `data.grf`.

---

## 10. Adding a New Targeted Skill

### Step 1 — `db/re/skill_db.yml`

**Single-target magic:**
```yaml
- Id: <NEW_ID>
  Name: XX_NEWBOLT
  Description: New Bolt
  MaxLevel: 10
  Type: Magic
  TargetType: Attack
  Range: 9
  Hit: Multi_Hit
  HitCount:
    - Level: 1
      Count: 1
    - Level: 10
      Count: 10
  Element: Wind
  CastTime:
    - Level: 1
      Time: 500
    - Level: 10
      Time: 3000
  Requires:
    SpCost:
      - Level: 1
        Amount: 10
```

**AoE melee with knockback:**
```yaml
- Id: <NEW_ID>
  Name: XX_NEWBASH
  Description: New Bash
  MaxLevel: 5
  Type: Weapon
  TargetType: Attack
  Range: -1
  Hit: Single
  HitCount: 2
  Element: Weapon
  SplashArea: 1              # 3×3 splash
  Knockback:
    - Level: 1
      Amount: 2
    - Level: 5
      Amount: 4
  Requires:
    SpCost:
      - Level: 1
        Amount: 10
```

### Step 2 — Create `src/map/skills/<job>/newskill.cpp`

```cpp
// Magic bolt (single target, multi-hit)
void SkillNewBolt::castendDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    skill_attack(BF_MAGIC, src, src, bl, getSkillId(), skill_lv, tick, flag);
}

// Physical AoE with knockback
void SkillNewBash::castendDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    // Primary hit
    skill_attack(BF_WEAPON, src, src, bl, getSkillId(), skill_lv, tick, flag);

    // Knockback (direction auto-calculated from caster → target)
    skill_blown(src, bl, skill_get_blewcount(getSkillId(), skill_lv), -1, BLOWN_NONE);

    // Splash to nearby enemies
    int32 splash = skill_get_splash(getSkillId(), skill_lv);
    if (splash > 0) {
        map_foreachinallarea(skill_area_sub,
            bl->m,
            bl->x - splash, bl->y - splash,
            bl->x + splash, bl->y + splash,
            splash_target(src),
            src, getSkillId(), skill_lv, tick,
            flag | BCT_ENEMY | SD_SPLASH, skill_castend_damage_id);
    }
}
```

### Step 3 — Add post-damage effects (optional)

If the skill applies a status effect on hit, add it in `skill_additional_effect()` in `skill.cpp`:

```cpp
case XX_NEWBASH:
    sc_start(src, bl, SC_STUN, 30 * skill_lv, skill_lv,
             skill_get_time2(skill_id, skill_lv));
    break;
```

---

## Key Functions Reference

| Function | File | Purpose |
|----------|------|---------|
| `skill_castend_damage_id(src, bl, id, lv, tick, flag)` | `skill.cpp` | Entry point for damaging targeted skills |
| `skill_attack(type, src, dsrc, bl, id, lv, tick, flag)` | `skill.cpp` | Execute damage, call `battle_calc_attack` |
| `battle_calc_attack(type, src, target, id, lv, flag)` | `battle.cpp` | Dispatch to weapon/magic/misc calculator |
| `battle_calc_weapon_attack(...)` | `battle.cpp` | Physical damage: ATK, hit/flee, element |
| `battle_calc_magic_attack(...)` | `battle.cpp` | Magic damage: MATK, MDEF, element table |
| `skill_area_sub(bl, ap)` | `skill.cpp` | Per-target validity check in AoE |
| `map_foreachinallarea(func, m, x0, y0, x1, y1, type, ...)` | `map.cpp` | Iterate all entities in a rectangle |
| `skill_blown(src, target, count, dir, flag)` | `skill.cpp` | Apply knockback to a target |
| `unit_blown(bl, dx, dy, count, flag)` | `unit.cpp` | Move entity `count` cells in (dx, dy) direction |
| `skill_additional_effect(src, bl, id, lv, type, dmg_lv, tick)` | `skill.cpp` | On-hit status application |
| `skill_get_blewcount(id, lv)` | `skill.cpp` | Read `Knockback` amount from skill_db |
| `skill_get_splash(id, lv)` | `skill.cpp` | Read `SplashArea` from skill_db |
| `clif_skill_damage(src, dst, tick, sd, dd, dmg, div, id, lv, type)` | `clif.cpp` | Send damage packet to clients |
