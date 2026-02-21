# Buff Skills — Implementation Guide

Covers three buff archetypes found in rAthena:
- **Toggle buff** — `SM_AUTOBERSERK` (Auto Berserk, Swordsman)
- **Single-target buff** — `AL_BLESSING` (Blessing, Acolyte)
- **Party AoE buff** — `AB_CLEMENTIA` (Clementia, Arch Bishop)

---

## Architecture Overview

Every buff follows the same pipeline:

```
skill_db.yml  →  castendNoDamageId()  →  sc_start()  →  status_change_start()  →  clif_status_change()
(definition)     (C++ skill impl)        (applies SC)    (stat recalc + timer)     (packet to client)
```

Status changes (SC) are the engine behind all buffs. Each SC has:
- An enum value in `src/map/status.hpp` (`SC_*`)
- A matching client icon constant (`EFST_*`)
- Initialization logic in `status_change_start()` (`src/map/status.cpp`)
- Optional timer logic in `status_change_timer()` (`src/map/status.cpp`)
- Optional removal logic in `status_change_end()` (`src/map/status.cpp`)

---

## 1. Skill Database (`db/re/skill_db.yml`)

All three skills share `NoDamage: true` and `TargetType: Self` or `Support`. The `Status:` field maps the skill to its SC enum.

### SM_AUTOBERSERK

```yaml
- Id: 146
  Name: SM_AUTOBERSERK
  Description: Auto Berserk
  MaxLevel: 1
  Type: Weapon
  TargetType: Self
  DamageFlags:
    NoDamage: true
  Flags:
    IsQuest: true          # Passive-style skill, treated as quest skill
  Hit: Single
  HitCount: 1
  Requires:
    SpCost: 1
  Status: AutoBerserk      # → SC_AUTOBERSERK
```

### AL_BLESSING

```yaml
- Id: 34
  Name: AL_BLESSING
  Description: Blessing
  MaxLevel: 10
  Type: Magic
  TargetType: Support      # Can be cast on allies
  DamageFlags:
    NoDamage: true
  Range: 9
  Hit: Single
  HitCount: 1
  Duration1:               # Duration scales with level (ms)
    - Level: 1
      Time: 60000          # 60s at Lv1
    - Level: 10
      Time: 240000         # 240s at Lv10
  Requires:
    SpCost:
      - Level: 1
        Amount: 28
      - Level: 10
        Amount: 48
  Status: Blessing         # → SC_BLESSING
```

### AB_CLEMENTIA

```yaml
- Id: 2041
  Name: AB_CLEMENTIA
  Description: Crementia
  MaxLevel: 3
  Type: Magic
  TargetType: Self         # Caster-centered, AoE splash handles spread
  DamageFlags:
    NoDamage: true
    Splash: true           # KEY: enables party splash application
  SplashArea:
    - Level: 1
      Area: 3              # 3-cell radius (7×7 area)
    - Level: 2
      Area: 7
    - Level: 3
      Area: 15
  CastTime: 3000
  FixedCastTime: 1000
  Duration1:
    - Level: 1
      Time: 120000
    - Level: 3
      Time: 240000
  Requires:
    SpCost:
      - Level: 1
        Amount: 280
      - Level: 3
        Amount: 360
  Status: Blessing         # → SC_BLESSING (same SC as AL_BLESSING)
```

---

## 2. C++ Skill Implementation

Skill implementations live in `src/map/skills/<job>/`. Each class overrides `castendNoDamageId()`.

### SM_AUTOBERSERK — `src/map/skills/swordman/autoberserk.cpp`

```cpp
void SkillAutoBerserk::castendNoDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    sc_type type = skill_get_sc(getSkillId());
    status_change *tsc = status_get_sc(bl);
    status_change_entry *tsce = (tsc) ? tsc->getSCE(type) : nullptr;

    int32 i;
    if (tsce)
        i = status_change_end(bl, type);       // Toggle OFF if already active
    else
        i = sc_start(src, bl, type, 100, skill_lv, 60000); // Toggle ON

    clif_skill_nodamage(src, *bl, getSkillId(), skill_lv, i);
}
```

**Key points:**
- Toggle pattern: `getSCE(type)` check → end or start.
- Passes `60000` ms as duration, but `status_change_start()` overrides this to `INFINITE_TICK`.
- `sc_start(src, bl, type, rate, val1, duration)` — `rate` is 100 (guaranteed).

---

### AL_BLESSING — `src/map/skills/acolyte/blessing.cpp`

```cpp
void SkillBlessing::castendNoDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    map_session_data *dstsd = BL_CAST(BL_PC, bl);
    status_change *tsc = status_get_sc(bl);
    sc_type type = skill_get_sc(getSkillId());

    clif_skill_nodamage(src, *bl, getSkillId(), skill_lv); // Animation first

    // Special: undead targets take damage instead of receiving buff
    if (dstsd != nullptr && tsc && tsc->getSCE(SC_CHANGEUNDEAD)) {
        status_data* tstatus = status_get_status_data(*bl);
        if (tstatus->hp > 1)
            skill_attack(BF_MISC, src, src, bl, getSkillId(), skill_lv, tick, flag);
        return;
    }

    sc_start(src, bl, type, 100, skill_lv,
             skill_get_time(getSkillId(), skill_lv)); // Duration from skill_db.yml Duration1
}
```

**Key points:**
- `clif_skill_nodamage()` is called before the SC so the animation plays regardless.
- `skill_get_time(skill_id, skill_lv)` reads `Duration1` from `skill_db.yml`.
- Undead targets redirect to `skill_attack(BF_MISC, ...)` — converts the buff into damage.

---

### AB_CLEMENTIA — `src/map/skills/acolyte/crementia.cpp`

```cpp
void SkillCrementia::castendNoDamageId(block_list *src, block_list *target,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    sc_type type = skill_get_sc(getSkillId());
    map_session_data* sd = BL_CAST(BL_PC, src);

    // Blessing level = caster's AL_BLESSING level + (job level / 10)
    int32 bless_lv = ((sd) ? pc_checkskill(sd, AL_BLESSING) : skill_get_max(AL_BLESSING))
                   + (((sd) ? sd->status.job_level : 50) / 10);

    if (sd == nullptr || sd->status.party_id == 0 || flag & 1) {
        // Phase 2: Apply to individual target (called recursively per party member)
        clif_skill_nodamage(target, *target, getSkillId(), skill_lv,
            sc_start(src, target, type, 100, bless_lv,
                     skill_get_time(getSkillId(), skill_lv)));
    } else if (sd) {
        // Phase 1: Iterate all party members in splash area
        party_foreachsamemap(skill_area_sub, sd,
            skill_get_splash(getSkillId(), skill_lv),  // Splash radius from YAML
            src, getSkillId(), skill_lv, tick,
            flag | BCT_PARTY | 1,                      // BCT_PARTY = include party members
            skill_castend_nodamage_id);                // Callback re-enters this function per member
    }
}
```

**Key points:**
- Two-phase execution: Phase 1 dispatches via `party_foreachsamemap`, Phase 2 applies per member.
- `flag & 1` distinguishes the recursive call from the initial call.
- `bless_lv` is dynamic: it combines the caster's actual `AL_BLESSING` rank with job level — not the `AB_CLEMENTIA` level.
- `BCT_PARTY` limits targets to party members. Use `BCT_GUILD` for guild AoE, or `BCT_ALL` for everything.
- `skill_get_splash(skill_id, skill_lv)` reads `SplashArea` from `skill_db.yml`.

---

## 3. Status Change Handling (`src/map/status.cpp`)

### Status Enum (`src/map/status.hpp`)

```cpp
SC_BLESSING,      // 30
SC_AUTOBERSERK,   // 85
```

Each SC has a matching `EFST_*` icon constant used by the client.

---

### Initialization in `status_change_start()`

Called by `sc_start()`. This is where val1–val4 semantics are defined and tick overrides happen.

**SC_AUTOBERSERK:**
```cpp
case SC_AUTOBERSERK:
    tick_time = 100;       // Timer fires every 100ms to check HP
    tick = INFINITE_TICK;  // Status itself never expires
    break;
```

**SC_BLESSING:**
```cpp
case SC_BLESSING:
    // Pre-application: remove conflicting statuses
    if (bl->type == BL_PC) {
        if (sc->getSCE(SC_CURSE)) {
            status_change_end(bl, SC_CURSE);
            return true; // Abort — only removes curse, no stat bonus
        } else if (sc->getSCE(SC_STONE)) {
            status_change_end(bl, SC_STONE);
            return true; // Abort — only removes stone
        }
    }

    // val1 = skill level (passed in)
    // val2 = effective stat modifier
    if (bl->type == BL_PC || (!undead_flag && status->race != RC_DEMON))
        val2 = val1;   // Normal: +val1 to STR/INT/DEX
    else
        val2 = 0;      // Undead/demon: 0 triggers -50% penalty instead
    break;
```

---

### Stat Calculation in `status_calc_bl_main()`

Stat modifiers from active SCs are applied here. This is called whenever the status set changes.

**SC_BLESSING example (in STR/INT/DEX calc blocks):**
```cpp
if (sc->getSCE(SC_BLESSING)) {
    if (sc->getSCE(SC_BLESSING)->val2)
        str += sc->getSCE(SC_BLESSING)->val2;  // +val2 STR
    else
        str -= str / 2;                         // Undead/demon: -50% STR
}
// Same pattern repeated for INT and DEX
```

---

### Timer Callback in `status_change_timer()`

Handles recurring logic for statuses with `tick_time > 0`.

**SC_AUTOBERSERK:**
```cpp
case SC_AUTOBERSERK:
    if (sc->hasSCE(SC_PROVOKE) && sc->getSCE(SC_PROVOKE)->val4 == 1) {
        // Auto-granted Provoke is active: remove it if HP recovered
        if (status->hp > status->max_hp / 4)
            status_change_end(bl, SC_PROVOKE);
    } else {
        // No Provoke: apply it if HP ≤ 25%
        if (status->hp <= status->max_hp / 4)
            sc_start4(bl, bl, SC_PROVOKE, 100, 10, 0, 0, 1, INFINITE_TICK);
        //                                  lv  v1 v2 v3 v4=1 marks as auto-granted
    }
    sc_timer_next(100 + tick); // Re-schedule in 100ms
    return 0;
```

**val4 = 1** is the sentinel used by `status_change_end(SC_AUTOBERSERK)` to know which `SC_PROVOKE` instance to clean up.

---

### Removal in `status_change_end()`

```cpp
case SC_AUTOBERSERK:
    // Clean up the auto-granted Provoke when Auto Berserk is toggled off
    if (sc->getSCE(SC_PROVOKE) && sc->getSCE(SC_PROVOKE)->val4 == 1)
        status_change_end(bl, SC_PROVOKE);
    break;
```

---

## 4. Client Communication (`src/map/clif.cpp`)

### Skill Animation Packet — `clif_skill_nodamage()`

Sent at the moment of casting. Triggers the visual and sound effect of the skill.

```cpp
// Signature
bool clif_skill_nodamage(const block_list* src, const block_list& dst,
                         uint16 skill_id, int32 heal, bool success);

// Usage in buff skills
clif_skill_nodamage(src, *bl, AL_BLESSING, skill_lv, true);
```

Sends `PACKET_ZC_USE_SKILL` to all clients in the area.

---

### Status Icon Packet — `clif_status_change()`

Called automatically by `status_change_start()` and `status_change_end()`. Tells the client which icon to show/hide and for how long.

```cpp
void clif_status_change(const block_list* bl, int32 type, int32 flag,
                        t_tick tick, int32 val1, int32 val2, int32 val3);
// type  = EFST_BLESSING, EFST_AUTOBERSERK, etc.
// flag  = 1 (start) or 0 (end)
// tick  = remaining duration in ms
// val1–3 = status-specific values shown to client
```

Packet header depends on `PACKETVER`: `0x983` (with timer display) or `0x196` (legacy).

---

## 5. Client-Side Files

### Skill Info Display — `data/luafiles514/lua files/skillinfoz/`

This folder contains `.lub` files that define the tooltip text and icon for each skill. If adding a new skill, register it here.

### Status Icons

Status icons (EFST constants) are handled by the client's own sprite sheets and the `stateicon/` Lua folder. The server sends the numeric EFST value; the client maps it to a sprite.

---

## 6. Adding a New Buff Skill

### Step 1 — `db/re/skill_db.yml`

```yaml
- Id: <NEW_ID>
  Name: XX_NEWBUFF
  Description: New Buff
  MaxLevel: 5
  Type: Magic
  TargetType: Support      # Use Self for self-only, Support for ally-targetable
  DamageFlags:
    NoDamage: true
  Range: 9
  Duration1:
    - Level: 1
      Time: 30000
    - Level: 5
      Time: 150000
  Requires:
    SpCost:
      - Level: 1
        Amount: 20
  Status: NewBuff          # Must match SC enum name
```

For party AoE, add:
```yaml
  DamageFlags:
    Splash: true
  SplashArea:
    - Level: 1
      Area: 5
```

### Step 2 — Add SC enum (`src/map/status.hpp`)

```cpp
SC_NEWBUFF,
```

Also add the matching `EFST_NEWBUFF` to the client effect enum if adding a status icon.

### Step 3 — Handle SC in `status.cpp`

**In `status_change_start()`** — define val semantics and duration:
```cpp
case SC_NEWBUFF:
    val2 = val1 * 5;  // val2 = some derived value, e.g. +5 per level
    break;
```

**In `status_calc_bl_main()`** — apply stat changes:
```cpp
if (sc->getSCE(SC_NEWBUFF))
    agi += sc->getSCE(SC_NEWBUFF)->val2;
```

**In `status_change_end()`** — cleanup (if needed):
```cpp
case SC_NEWBUFF:
    // remove linked statuses here
    break;
```

### Step 4 — Create skill implementation file

`src/map/skills/<job>/newbuff.cpp`:

```cpp
// Single-target pattern
void SkillNewBuff::castendNoDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    clif_skill_nodamage(src, *bl, getSkillId(), skill_lv);
    sc_start(src, bl, skill_get_sc(getSkillId()), 100, skill_lv,
             skill_get_time(getSkillId(), skill_lv));
}

// Party AoE pattern
void SkillNewBuffAoE::castendNoDamageId(block_list *src, block_list *bl,
    uint16 skill_lv, t_tick tick, int32& flag) const
{
    map_session_data* sd = BL_CAST(BL_PC, src);

    if (sd == nullptr || sd->status.party_id == 0 || flag & 1) {
        clif_skill_nodamage(bl, *bl, getSkillId(), skill_lv,
            sc_start(src, bl, skill_get_sc(getSkillId()), 100, skill_lv,
                     skill_get_time(getSkillId(), skill_lv)));
    } else {
        party_foreachsamemap(skill_area_sub, sd,
            skill_get_splash(getSkillId(), skill_lv),
            src, getSkillId(), skill_lv, tick,
            flag | BCT_PARTY | 1, skill_castend_nodamage_id);
    }
}
```

Register the implementation in `skill_db` during server init (see existing skill constructors for the registration pattern).

---

## Key Functions Reference

| Function | File | Purpose |
|----------|------|---------|
| `sc_start(src, bl, type, rate, val1, tick)` | `status.cpp` | Apply a status change |
| `sc_start2(...)` | `status.cpp` | Apply SC with explicit val2 |
| `sc_start4(...)` | `status.cpp` | Apply SC with val1–val4 |
| `status_change_end(bl, type)` | `status.cpp` | Remove a status change |
| `skill_get_sc(skill_id)` | `skill.cpp` | Get SC type for a skill |
| `skill_get_time(skill_id, lv)` | `skill.cpp` | Get Duration1 from skill_db |
| `skill_get_splash(skill_id, lv)` | `skill.cpp` | Get SplashArea from skill_db |
| `party_foreachsamemap(func, sd, range, ...)` | `party.cpp` | Iterate party members in range |
| `clif_skill_nodamage(src, dst, id, lv, success)` | `clif.cpp` | Send skill use animation packet |
| `clif_status_change(bl, type, flag, tick, ...)` | `clif.cpp` | Send status icon packet |
