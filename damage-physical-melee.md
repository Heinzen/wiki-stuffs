# Physical Melee Damage Pipeline (`BF_WEAPON | BF_SHORT`)

All physical melee attacks — basic attacks and skills like Bash, Bowling Bash, Brandish Spear — flow through `battle_calc_weapon_attack()` with the `BF_SHORT` range flag. The full sequence from dispatcher to final damage is documented here.

**Source file:** `src/map/battle.cpp`

---

## Pipeline at a Glance

```
battle_calc_attack()
  └─ battle_calc_weapon_attack()
       ├─ 1. initialize_weapon_data()         — set div_, amotion, flags, zero all damage components
       ├─ 2. Lucky dodge check                — flee2 roll, early exit
       ├─ 3. battle_calc_multi_attack()       — double attack, multi-hit chance
       ├─ 4. is_attack_critical()             — crit rate roll
       ├─ 5. is_attack_hitting()              — hit/flee roll
       ├─ 6. battle_calc_skill_base_damage()  — statusAtk + weaponAtk + equipAtk + percentAtk
       ├─ 7. battle_attack_sc_bonus()         — EDP, Berserk, Overthrust, etc.
       ├─ 8. Damage assembly                  — combine components, apply P.ATK
       ├─ 9. Add masteryAtk
       ├─ 10. Crit multiplier                 — ×(1.4 + 0.01×crate) for players
       ├─ 11. battle_calc_attack_skill_ratio() — ×skillratio/100
       ├─ 12. battle_calc_skill_constant_addition() — flat damage bonus
       ├─ 13. battle_calc_cardfix()           — size / race / element / class card multipliers
       ├─ 14. battle_calc_defense_reduction() — DEF formula: atk × (4000+eDEF)/(4000+10×eDEF) − sDEF
       ├─ 15. RES reduction (RENEWAL)         — mres / (mres+400) × 80%
       ├─ 16. battle_calc_attack_post_defense() — Aura Blade, min-damage floor
       ├─ 17. battle_calc_element_damage()    — call battle_attr_fix()
       ├─ 18. Skill-specific late modifiers   — Shield Press, Tiger Cannon, etc.
       ├─ 19. battle_calc_attack_gvg_bg()     — GVG/BG damage rates
       └─ 20. battle_absorb_damage() / battle_do_reflect()
```

---

## Step 1 — Dispatcher (`battle_calc_attack`, line 8782)

```cpp
switch(attack_type) {
    case BF_WEAPON: d = battle_calc_weapon_attack(bl, target, skill_id, skill_lv, flag); break;
    case BF_MAGIC:  d = battle_calc_magic_attack(...); break;
    case BF_MISC:   d = battle_calc_misc_attack(...);  break;
}
// Post-calc: if damage < 1 → ATK_MISS, else ATK_DEF
```

---

## Step 2 — Initialization (`initialize_weapon_data`, line 6644)

Zeroes and pre-fills the `Damage` struct. Key fields set here:

| Field | Value |
|-------|-------|
| `div_` | `skill_get_num(skill_id, skill_lv)` — number of hits |
| `amotion` | `sstatus->amotion` (0 for ground skills) |
| `dmotion` | `tstatus->dmotion` |
| `blewcount` | Knockback cells from skill_db.yml |
| `flag` | `BF_WEAPON | BF_SHORT` (+ `BF_SKILL` if skill-based) |
| RENEWAL components | `statusAtk`, `weaponAtk`, `equipAtk`, `masteryAtk`, `percentAtk` = 0 |
| Pre-renewal | `basedamage` = 0 |

The `BF_SHORT` / `BF_LONG` range flag is set inside `initialize_weapon_data` via `battle_range_type()`.

---

## Step 3 — Lucky Dodge (line 6933)

```cpp
if ((!skill_id || skill_id == PA_SACRIFICE) && tstatus->flee2 && rnd()%1000 < tstatus->flee2) {
    wd.type  = DMG_LUCY_DODGE;
    wd.dmg_lv = ATK_LUCKY;
    return wd;  // early exit — no damage
}
```

`flee2` is the "perfect dodge" stat (from Thara Frog card etc.). Only triggers against basic attacks and Sacrifice.

---

## Step 4 — Critical Check (`is_attack_critical`, line 3005)

```cpp
int16 cri = sstatus->cri;
cri += sd->indexed_bonus.critaddrace[tstatus->race];  // racial crit bonus
cri += sd->bonus.arrow_cri;                           // arrow crit (ranged only)
cri -= tstatus->luk * (vs_player ? 3 : 2);           // target LUK reduces crit
if (tsc->getSCE(SC_SLEEP)) cri *= 2;                 // sleeping target: 2× crit chance

return (rnd()%1000 < cri);
```

Sets `wd.type = DMG_CRITICAL`. Critical attacks **always hit** (skip the flee check).

---

## Step 5 — Hit/Flee Check (`is_attack_hitting`, line 3211)

**Auto-hit conditions:** critical, perfect hit, frozen/stunned target, `NK_IGNOREFLEE` flag.

```cpp
int32 hitrate = 0;             // RENEWAL default (80 in pre-re)
hitrate += sstatus->hit - flee;
// AGI penalty if many attackers on same target:
flee -= penalty_per_extra_attacker;

hitrate = cap_value(hitrate, min_hitrate, max_hitrate);
return (rnd()%100 < hitrate);
```

Failure sets `wd.dmg_lv = ATK_FLEE` (damage = 0).

---

## Step 6 — Base Damage (`battle_calc_skill_base_damage`, line 4141)

### RENEWAL — 5 separate components

| Component | Source |
|-----------|--------|
| `statusAtk` | STR-based base ATK |
| `weaponAtk` | Right-hand weapon ATK |
| `equipAtk` | Non-weapon gear bonuses |
| `masteryAtk` | Mastery skills (Sword/2H-Sword Mastery, etc.) |
| `percentAtk` | % ATK bonuses from cards/equip |

Computed via `battle_calc_damage_parts()`. Left-hand (`*2` variants) filled if dual-wielding.

### Pre-Renewal — single value

```cpp
wd.damage = battle_calc_base_damage(src, sstatus, &sstatus->rhw, sc, tstatus->size, bflag);
```

---

## Step 7 — Status Change Bonuses (`battle_attack_sc_bonus`, line 5902)

Applied to individual components before assembly. Key examples:

| Status | Effect |
|--------|--------|
| `SC_EDP` (Enchant Deadly Poison) | `weaponAtk × (2.5 + 0.3×lv)` |
| `SC_BERSERK` | `+200% skillratio` (RENEWAL) |
| `SC_OVERTHRUST` | `+val3%` to skillratio |
| `SC_MAXOVERTHRUST` | `+val2%` to skillratio |
| `SC_DRUMBATTLE` | `+val2` flat to equipAtk |
| `SC_CRUSHSTRIKE` | Replaces skillratio: `weapon_lv × (refine+6) × 100 + weapon_atk + weight/10` |

---

## Step 8 — Damage Assembly (RENEWAL, line 7034)

```cpp
wd.damage = statusAtk + weaponAtk + equipAtk + percentAtk;
wd.damage = wd.damage * (100 + sstatus->patk) / 100;   // P.ATK multiplier
wd.damage += masteryAtk;
```

P.ATK (`sstatus->patk`) is the RENEWAL physical attack stat derived from base level and equipment.

---

## Step 9 — Critical Damage Multiplier (RENEWAL, line 7155)

```cpp
// Players
wd.damage = floor(wd.damage * (1.4f + 0.01f * sstatus->crate));
// Monsters: fixed ×1.4
```

`crate` is the critical damage rate stat (base 0; can be increased by cards/skills). Non-crit attacks receive `sd->bonus.non_crit_atk_rate` additive bonus instead.

Additionally:

```cpp
if (wd.flag & BF_SHORT)
    ATK_ADDRATE(wd.damage, wd.damage2, sd->bonus.short_attack_atk_rate);
```

---

## Step 10 — Skill Ratio (`battle_calc_attack_skill_ratio`, line 4631)

```cpp
int32 skillratio = 100;  // 100 = no change, 200 = double damage
// skill-specific switch adds/subtracts from skillratio
ATK_RATE(wd.damage, wd.damage2, skillratio);
// Macro: damage = damage * skillratio / 100
```

Every damaging skill has an entry here. Basic attacks use 100 (no change). Custom skill ratios are added in `calculateSkillRatio()` in the skill's implementation `.cpp` file.

---

## Step 11 — Constant Addition (`battle_calc_skill_constant_addition`, line 5850)

Flat post-ratio bonus. Most skills return 0. Examples that use it:

| Skill | Formula |
|-------|---------|
| `MO_EXTREMITYFIST` | `250 + 150×lv` |
| `PA_SHIELDCHAIN` | `rnd(100, 0.7×shield_weight + (lv+refine)²)` |

```cpp
ATK_ADD(wd.damage, wd.damage2, constant_addition);
// Macro: damage += bonus
```

---

## Step 12 — Card Fix (`battle_calc_cardfix`, line 769)

Applied in two passes: **attacker cards** (modify `weaponAtk`/`equipAtk`) then **target cards** (modify all components).

Each pass multiplies damage by:

```cpp
cardfix = cardfix * (100 + size_bonus)  / 100;  // attacker size card vs target size
cardfix = cardfix * (100 + race_bonus)  / 100;  // attacker race card vs target race
cardfix = cardfix * (100 + ele_bonus)   / 100;  // attacker element card vs target element
cardfix = cardfix * (100 + class_bonus) / 100;  // specific monster card bonuses
// Target cards: subtract resistance bonuses instead:
cardfix = cardfix * (100 - subele[atk_ele])  / 100;
cardfix = cardfix * (100 - subrace[atk_race]) / 100;
cardfix = cardfix * (100 - subsize[atk_size]) / 100;
```

`statusAtk` and `masteryAtk` use `NK_IGNOREELEMENT` so element-resist cards do not reduce them.

---

## Step 13 — Defense Reduction (`battle_calc_defense_reduction`, line 6087)

### RENEWAL Formula

```
final = damage × (4000 + eDEF) / (4000 + 10×eDEF) − sDEF
```

| eDEF value | % of damage that passes |
|------------|------------------------|
| 0 | 100% |
| 100 | ~73% |
| 400 | 55% |
| 1000 | 30% |
| ∞ | 10% (hard floor) |

- **eDEF** = `tstatus->def` (equipment DEF) — reduced by attacker's `ignore_def_by_race`
- **sDEF** = `tstatus->def2` (soft/VIT-based DEF) — flat subtraction after the ratio

Skills with `NK_SIMPLEDEFENSE` use flat subtraction only: `damage − (eDEF + sDEF)`.
Skills with `NK_IGNOREDEFENSE` skip this step entirely.

### Pre-Renewal Formula

```
damage = damage * (100 − DEF) / 100 − DEF2
```

---

## Step 14 — Physical Resistance (RENEWAL, line 7090)

```
damage -= (RES / (RES + 400)) × 80% × damage
```

`tstatus->res` is the physical resistance stat (RENEWAL only). Analogous to MDEF for magic. Capped at 50% ignore via `max_res_mres_ignored`.

---

## Step 15 — Element Fix (`battle_attr_fix`, line 511)

```cpp
int32 ratio = elemental_attribute_db.getAttribute(def_lv, atk_elem, def_type);
// Modifiers from attacker SC (Volcano +ratio for Fire, etc.)
// Modifiers from target SC (Spider Web +100 for Fire, etc.)
damage = damage - (damage * (100 - ratio) / 100);  // RENEWAL
damage = damage * ratio / 100;                      // Pre-Renewal
```

The element table is indexed by `(elemental_level, attack_element, defense_element)`. Typical ratios: 0 (immune), 50 (resistant), 100 (neutral), 125/150 (weak), 200 (very weak).

---

## Step 16 — Final Steps

| Step | Function | Notes |
|------|----------|-------|
| GVG/BG rates | `battle_calc_attack_gvg_bg()` | Applies map-flag damage multipliers |
| Left/right hand | `battle_calc_attack_left_right_hands()` | Finalizes dual-wield split |
| Damage absorb | `battle_absorb_damage()` | HP absorb cards/bonuses |
| Reflect | `battle_do_reflect()` | Shield Reflect, Maya card, etc. |

---

## Damage Struct Fields (`battle.hpp`)

```cpp
struct Damage {
    // RENEWAL only — 5 components per hand:
    int64 statusAtk, statusAtk2;
    int64 weaponAtk, weaponAtk2;
    int64 equipAtk, equipAtk2;
    int64 masteryAtk, masteryAtk2;
    int64 percentAtk, percentAtk2;

    int64  damage, damage2;   // Final right/left hand damage
    int16  div_;              // Hit count (negative = fixed per-hit damage)
    int32  amotion, dmotion;  // Attack/damage motion delay
    int32  blewcount;         // Knockback cells
    int32  flag;              // BF_WEAPON | BF_SHORT | BF_SKILL
    int32  miscflag;          // Skill-specific flags
    enum   damage_lv dmg_lv; // ATK_DEF / ATK_FLEE / ATK_LUCKY / ATK_MISS
    enum   e_damage_type type;// DMG_NORMAL / DMG_CRITICAL / DMG_MULTI_HIT
    bool   isspdamage;        // True if damaging SP instead of HP
};
```

---

## Formula Summary (RENEWAL)

```
base       = statusAtk + weaponAtk + equipAtk + percentAtk
with_patk  = base × (100 + P.ATK) / 100
with_mast  = with_patk + masteryAtk
with_crit  = with_mast × (1.4 + 0.01×crate)  [if critical]
with_ratio = with_crit × skillratio / 100
with_const = with_ratio + flat_addition
with_cards = with_const × attacker_cards × (1 − target_cards)
with_def   = with_cards × (4000 + eDEF) / (4000 + 10×eDEF) − sDEF
with_res   = with_def − (RES / (RES+400)) × 0.8 × with_def
final      = battle_attr_fix(with_res, atk_element, def_element, def_level)
```

---

## Key Functions Reference

| Function | Line | Purpose |
|----------|------|---------|
| `battle_calc_attack()` | 8776 | Main dispatcher |
| `battle_calc_weapon_attack()` | 6902 | Physical damage pipeline |
| `initialize_weapon_data()` | 6644 | Zero and pre-fill Damage struct |
| `is_attack_hitting()` | 3211 | Hit/flee roll |
| `is_attack_critical()` | 3005 | Critical rate roll |
| `battle_calc_skill_base_damage()` | 4141 | Stat-based damage components |
| `battle_calc_attack_skill_ratio()` | 4631 | Skill multiplier (×skillratio/100) |
| `battle_calc_skill_constant_addition()` | 5850 | Flat bonus after ratio |
| `battle_attack_sc_bonus()` | 5902 | Status effect modifiers |
| `battle_calc_cardfix()` | 769 | Size/race/element/class card multipliers |
| `battle_calc_defense_reduction()` | 6087 | DEF formula |
| `battle_attr_fix()` | 511 | Elemental multiplier table lookup |
