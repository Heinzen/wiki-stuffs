# Miscellaneous Damage Pipeline (`BF_MISC`)

`BF_MISC` is used by skills that do not fit the standard weapon-ATK or MATK formula. This covers traps, falcon attacks, alchemist explosions, zeny throws, self-destruct, and similar skills where the damage formula is entirely skill-specific.

**Source file:** `src/map/battle.cpp` — `battle_calc_misc_attack()` starts at line 8318.

---

## Pipeline at a Glance

```
battle_calc_attack()
  └─ battle_calc_misc_attack()
       ├─ 1. Struct initialization              — div_, amotion, flags, BF_MISC|BF_SKILL
       ├─ 2. battle_get_misc_element()           — determine skill element
       ├─ 3. battle_range_type()                 — set BF_SHORT or BF_LONG sub-flag
       ├─ 4. Skill-specific damage formula switch — custom formula per skill
       ├─ 5. NK_SPLASHSPLIT                      — divide damage per target count
       ├─ 6. Hit/flee check (if !NK_IGNOREFLEE)  — same HIT−FLEE roll as physical
       ├─ 7. battle_calc_cardfix(BF_MISC)        — element/race card multipliers
       ├─ 8. pc_skillatk_bonus / pc_sub_skillatk_bonus — skill-specific % bonus
       ├─ 9. battle_attr_fix()                   — elemental table (if !NK_IGNOREELEMENT)
       ├─ 10. battle_apply_div_fix()             — finalize multi-hit
       ├─ 11. Post-formula additions             — e.g. trap + weapon ATK for RA traps
       ├─ 12. battle_calc_damage()               — GVG/BG rates, SC blocks
       ├─ 13. battle_skill_damage()              — global skill damage override
       └─ 14. battle_absorb_damage() / battle_do_reflect()
```

---

## Step 1 — Dispatcher (`battle_calc_attack`, line 8782)

```cpp
case BF_MISC: d = battle_calc_misc_attack(bl, target, skill_id, skill_lv, flag); break;
```

---

## Step 2 — Struct Initialization (line 8337)

```cpp
md.amotion   = sstatus->amotion;                  // 0 for ground skills
md.dmotion   = tstatus->dmotion;
md.div_      = skill_get_num(skill_id, skill_lv);
md.blewcount = skill_get_blewcount(skill_id, skill_lv);
md.dmg_lv    = ATK_DEF;
md.flag      = BF_MISC | BF_SKILL;
md.miscflag  = mflag;
```

Key difference from physical/magic: **no base damage is set here**. Every BF_MISC skill must compute its own `md.damage` in the switch below.

---

## Step 3 — Element & Range (lines 8360–8363)

```cpp
s_ele = battle_get_misc_element(src, target, skill_id, skill_lv, mflag);
md.flag |= battle_range_type(src, target, skill_id, skill_lv);
```

The skill element comes from `skill_db.yml`. Range (`BF_SHORT` / `BF_LONG`) is assigned the same way as physical.

---

## Step 4 — Skill-Specific Damage Formulas (line 8365)

This is the core of BF_MISC: every skill has its own formula. There is no generic base stat calculation.

### Traps — `HT_LANDMINE`, `HT_BLASTMINE`, `HT_CLAYMORETRAP`

**RENEWAL:**
```cpp
md.damage = skill_lv * sstatus->dex * (3.0 + status_get_lv(src)/100.0) * (1.0 + sstatus->int_/35.0);
md.damage += md.damage * (rnd()%20 - 10) / 100;            // ±10% random variance
md.damage += pc_checkskill(sd, RA_RESEARCHTRAP) * 40;       // Research Trap flat bonus
```

**Pre-Renewal (skill-specific):**
```cpp
// Land Mine:
md.damage = skill_lv * (sstatus->dex + 75.0) * (100.0 + sstatus->int_) / 100.0;
// Blast Mine:
md.damage = skill_lv * (sstatus->dex/2.0 + 50.0) * (100.0 + sstatus->int_) / 100.0;
// Claymore Trap:
md.damage = skill_lv * (sstatus->dex/2.0 + 75.0) * (100.0 + sstatus->int_) / 100.0;
```

**Stats used:** DEX (primary), INT (modifier), skill level, base level (RENEWAL only).

---

### Falcon — `HT_BLITZBEAT`, `SN_FALCONASSAULT`

```cpp
uint16 steelcrow = pc_checkskill(sd, HT_STEELCROW);

// RENEWAL:
md.damage = skill_lv*20 + steelcrow*6 + (sstatus->agi/2)*2 + (sstatus->dex/10)*2;

// Pre-Renewal:
md.damage = (sstatus->dex/10 + sstatus->int_/2 + steelcrow*3 + 40) * 2;
if (mflag > 1) nk.set(NK_SPLASHSPLIT);   // Auto-blitz splits among targets

// Falcon Assault modifier:
DAMAGE_DIV_FIX2(md.damage, skill_get_num(HT_BLITZBEAT, 5));  // div fix of Blitz Lv5
md.damage = md.damage * (150 + 70*skill_lv) / 100;
```

---

### Ranger Traps — `RA_CLUSTERBOMB`, `RA_FIRINGTRAP`, `RA_ICEBOUNDTRAP`

```cpp
md.damage = skill_lv * status_get_dex(src) + status_get_int(src)*5;
RE_LVL_TMDMOD();   // base level modifier

int32 research_lv = pc_checkskill(sd, RA_RESEARCHTRAP);
md.damage = md.damage * 20 * research_lv / (skill_id == RA_CLUSTERBOMB ? 50 : 100);

nk.set(NK_IGNOREELEMENT);  // Trap damage ignores element table
nk.set(NK_IGNOREFLEE);
nk.set(NK_IGNOREDEFCARD);
```

After `battle_apply_div_fix`, weapon ATK is also added:
```cpp
struct Damage wd = battle_calc_weapon_attack(src, target, skill_id, skill_lv, mflag);
md.damage += wd.damage;   // trap formula + physical weapon damage
```

---

### Acid Demonstration / Fire Expansion Acid

**RENEWAL (`GN_FIRE_EXPANSION_ACID`):**
```cpp
// Hybrid: combines weapon ATK and MATK
struct Damage atk  = battle_calc_weapon_attack(src, target, skill_id, skill_lv, 0);
struct Damage matk = battle_calc_magic_attack(src, target, skill_id, skill_lv, 0);
md.damage = 7 * ((atk.damage/skill_lv + matk.damage/skill_lv) * tstatus->vit / 100);
// Force neutral element on final result:
md.damage = battle_attr_fix(src, target, md.damage, ELE_NEUTRAL, tstatus->def_ele, tstatus->ele_lv);
```

**Pre-Renewal (`CR_ACIDDEMONSTRATION`):**
```cpp
// Ignores ATK/MATK; uses target VIT and caster INT only:
md.damage = 7 * tstatus->vit * sstatus->int_ * sstatus->int_ / (10 * (tstatus->vit + sstatus->int_));
if (tsd) md.damage /= 2;   // Half damage vs players
```

---

### Zeny Throw — `NJ_ZENYNAGE`, `KO_MUCHANAGE`

```cpp
md.damage = skill_get_zeny(skill_id, skill_lv);   // base zeny cost from skill_db
md.damage += rnd_value(0, md.damage);              // +0~100% random bonus

if (status_get_class_(target) == CLASS_BOSS) md.damage /= 3;  // 1/3 vs boss
if (tsd) md.damage /= 2;                                       // 1/2 vs player

// Zeny is actually deducted from caster after hit:
if (sd && md.damage > sd->status.zeny)
    md.damage = sd->status.zeny;
pc_payzeny(sd, md.damage, LOG_TYPE_CONSUME);
```

---

### Other Notable Formulas

| Skill | Formula |
|-------|---------|
| `NPC_SELFDESTRUCTION` | `md.damage = sstatus->hp` (caster's current HP) |
| `PA_GOSPEL` (random attack) | `rnd(1500, 5500)` − DEF |
| `SP_SOULEXPLOSION` | `target_HP × (20 + 10×lv) / 100` |
| `HVAN_EXPLOSION` | `sstatus->max_hp × (50 + 50×lv) / 100` |
| `GS_FLING` | `sd->status.job_level` |
| `TF_THROWSTONE` | Fixed 50 (player) / 30 (NPC) |
| `SJ_NOVAEXPLOSING` | `(batk + rhw.atk) × ratio + max_hp/factor + max_sp×(2×lv)` |
| `RL_B_TRAP` | `DEX×10 + (lv×3×target_HP)/100` |
| `NJ_ISSEN` (RENEWAL) | `hp + (atk.damage×hp×lv)/max_hp` + mirror bonus − (eDEF+sDEF) |

---

## Step 5 — Splash Split (`NK_SPLASHSPLIT`, line 8614)

```cpp
if (nk[NK_SPLASHSPLIT]) {
    if (mflag > 0)
        md.damage /= mflag;  // divide total damage among all targets
}
```

Auto-blitz (`HT_BLITZBEAT` on multi-target) and some NPC skills use this.

---

## Step 6 — Hit/Flee Check (line 8621)

Unlike magic, BF_MISC **does** roll against flee unless `NK_IGNOREFLEE` is set:

```cpp
if (!nk[NK_IGNOREFLEE]) {
    int16 hitrate = 0;   // RENEWAL default
    hitrate += sstatus->hit - tstatus->flee;
    hitrate += pc_checkskill(sd, AC_VULTURE);   // RENEWAL bonus
    hitrate = cap_value(hitrate, min_hitrate, max_hitrate);
    if (rnd()%100 >= hitrate)
        md.dmg_lv = ATK_FLEE;  // miss
}
```

Traps (`RA_CLUSTERBOMB`, etc.) set `NK_IGNOREFLEE` — they always hit.

---

## Step 7 — Card Fix (`battle_calc_cardfix(BF_MISC)`, line 8665)

```cpp
md.damage += battle_calc_cardfix(BF_MISC, src, target, nk, s_ele, 0, md.damage, 0, md.flag);
```

For `BF_MISC`, the card fix applies:
- **Target's element resist cards** (`subele`, `subele2` filtered by `BF_MISC`)
- **Target's race resist cards** (`subrace`)
- **No attacker size/class cards** (BF_MISC attacker side is minimal)

If `NK_IGNOREDEFCARD` is set (Ranger traps), the entire target card step is skipped.

---

## Step 8 — Skill Attack Bonus (line 8667)

```cpp
if (sd && (i = pc_skillatk_bonus(sd, skill_id)))
    md.damage += md.damage * i / 100;   // attacker % bonus for this skill

if (tsd && (i = pc_sub_skillatk_bonus(tsd, skill_id)))
    md.damage -= md.damage * i / 100;  // target % resistance for this skill
```

Equipment that grants `+X% damage with skill Y` is applied here for both attacker and defender.

---

## Step 9 — Element Fix (`battle_attr_fix`, line 8673)

```cpp
if (!nk[NK_IGNOREELEMENT])
    md.damage = battle_attr_fix(src, target, md.damage, s_ele, tstatus->def_ele, tstatus->ele_lv);
```

Same elemental table lookup as physical and magic. Skills that have `NK_IGNOREELEMENT` (e.g. Ranger traps, `SN_FALCONASSAULT`) bypass this entirely — their damage is treated as property-less.

---

## Step 10 — div Fix (line 8684)

```cpp
battle_apply_div_fix(&md, skill_id);
```

Adjusts for negative `div_` (fixed-damage-per-hit skills) and ensures minimum damage.

---

## Step 11 — Post-Formula Additions (line 8686)

After the element fix and div_fix, some skills add extra damage:

```cpp
case RA_FIRINGTRAP:
case RA_ICEBOUNDTRAP:
case RA_CLUSTERBOMB: {
    // Add weapon ATK on top of trap formula
    struct Damage wd = battle_calc_weapon_attack(src, target, skill_id, skill_lv, mflag);
    md.damage += wd.damage;
}
```

---

## Step 12 — Final Steps (line 8708)

```cpp
md.damage = battle_calc_damage(src, target, &md, md.damage, skill_id, skill_lv);

if (mapdata_flag_gvg2(mapdata))
    md.damage = battle_calc_gvg_damage(src, target, md.damage, skill_id, md.flag);
else if (mapdata->getMapFlag(MF_BATTLEGROUND))
    md.damage = battle_calc_bg_damage(src, target, md.damage, skill_id, md.flag);

// Global skill damage adjust table (conf/battle/):
if ((skill_damage = battle_skill_damage(src, target, skill_id)) != 0)
    md.damage += md.damage * skill_damage / 100;

battle_absorb_damage(target, &md);
battle_do_reflect(BF_MISC, &md, src, target, skill_id, skill_lv);
```

---

## NK_ Flags (No-Kill modifiers)

These bitset flags from `skill_db.yml` skip or modify specific pipeline steps:

| Flag | Effect |
|------|--------|
| `NK_IGNOREFLEE` | Skip hit/flee roll — always connects |
| `NK_IGNOREELEMENT` | Skip `battle_attr_fix()` — no elemental modifier |
| `NK_IGNOREDEFENSE` | Skip MDEF/DEF reduction |
| `NK_IGNOREDEFCARD` | Skip target defense card fix |
| `NK_SPLASHSPLIT` | Divide damage by number of targets |
| `NK_SIMPLEDEFENSE` | Flat DEF subtraction instead of formula |

---

## Comparison: BF_MISC vs BF_WEAPON vs BF_MAGIC

| Aspect | BF_WEAPON | BF_MAGIC | BF_MISC |
|--------|-----------|----------|---------|
| Base stat | ATK (STR) | MATK (INT) | **Skill-specific formula** |
| Hit check | Yes (hit−flee) | No (auto-hit) | Optional (NK_IGNOREFLEE) |
| Defense | DEF formula | MDEF formula | **No default DEF — varies by skill** |
| Element | Weapon element | Spell element | Skill element (often NK_IGNOREELEMENT) |
| Card fix | Full (size/race/ele/class) | Element/race | Element/race only |
| Crit | Yes | No | No |
| Reflect | Shield reflect | Magic reflect (Kaite) | BF_MISC reflect (limited) |
| Dual-wield | Yes | No | No |

---

## Adding a New BF_MISC Skill

1. In `skill_db.yml`, set `Type: Misc` and configure `NK` flags as needed.
2. Add a `case NEW_SKILL:` inside `battle_calc_misc_attack()` that sets `md.damage` to your formula.
3. If the skill should ignore element: `nk.set(NK_IGNOREELEMENT)`.
4. If the skill should always hit: `nk.set(NK_IGNOREFLEE)`.
5. If the skill should ignore DEF: `nk.set(NK_IGNOREDEFENSE)`.

```cpp
case MY_NEW_SKILL:
    // Example: damage scales with caster DEX and target VIT
    md.damage = sstatus->dex * skill_lv + tstatus->vit * 2;
    nk.set(NK_IGNOREFLEE);      // always hits
    nk.set(NK_IGNOREELEMENT);   // no elemental modifier
    break;
```

---

## Key Functions Reference

| Function | Line | Purpose |
|----------|------|---------|
| `battle_calc_misc_attack()` | 8318 | Misc damage pipeline |
| `battle_get_misc_element()` | ~8360 | Determine skill element |
| `battle_calc_cardfix(BF_MISC)` | 769 | Card multipliers (element/race only) |
| `battle_attr_fix()` | 511 | Elemental table lookup |
| `battle_apply_div_fix()` | — | Finalize multi-hit (negative div_) |
| `pc_skillatk_bonus()` | — | Equipment % bonus per skill |
| `pc_sub_skillatk_bonus()` | — | Target % resistance per skill |
| `battle_calc_damage()` | — | SC blocks, GVG/BG rates |
| `battle_skill_damage()` | — | Global skill damage override |
