# Physical Ranged Damage Pipeline (`BF_WEAPON | BF_LONG`)

Ranged physical attacks — Hunter's Arrow, Gunslinger skills, Sniper skills, thrown weapons — use the same `battle_calc_weapon_attack()` function as melee. The `BF_LONG` flag, set during `initialize_weapon_data()`, controls which branches are taken throughout the pipeline.

This document covers **only what differs from the melee pipeline**. Read `damage-physical-melee.md` first for the base pipeline.

**Source file:** `src/map/battle.cpp`

---

## What Changes with `BF_LONG`

```
battle_calc_weapon_attack()
  ├─ initialize_weapon_data()    — range flag set to BF_LONG
  ├─ is_attack_hitting()         — Vulture's Eye bonus added; Fog Wall penalty
  ├─ battle_calc_skill_base_damage() — arrow ATK contribution
  ├─ battle_attack_sc_bonus()    — long_attack_atk_rate bonus
  ├─ battle_calc_cardfix()       — long_attack_atk_rate (pre-re); BF_LONG card categories
  ├─ battle_calc_damage()        — target SC blocks (Pneuma, Neutral Barrier, Adjustment)
  └─ final modifiers             — range-specific damage bonuses
```

---

## 1. Range Flag Assignment (`battle_range_type`, line 2686)

`battle_range_type()` inspects the skill and weapon type to decide `BF_SHORT` or `BF_LONG`:

```cpp
// Bow/gun/thrown weapons always return BF_LONG
if (sd->state.arrow_atk)
    return BF_LONG;

// Specific long-range skills regardless of weapon:
switch (skill_id) {
    case AC_DOUBLE:
    case AC_SHOWER:
    case GS_DESPERADO:
    // ... many ranged skills
        return BF_LONG;
}

// Default for non-skill attacks with no arrow: BF_SHORT
return BF_SHORT;
```

The result is OR'd into `wd.flag`: `BF_WEAPON | BF_LONG | BF_SKILL`.

---

## 2. Arrow / Ammo ATK Contribution (line 2564)

Arrow ATK is added to `weaponAtk` inside `battle_calc_base_damage()` when `sd->state.arrow_atk` is set:

```cpp
// In RENEWAL, arrows contribute to the weapon ATK component
if (flag & BDMG_ARROW && sd->bonus.arrow_atk)
    damage += (flag & BDMG_CRIT) ? sd->bonus.arrow_atk        // crit: full arrow atk
                                  : rnd() % sd->bonus.arrow_atk; // normal: 0~(arrow_atk-1)
```

`sd->bonus.arrow_atk` comes from the equipped arrow/ammo item's ATK field. This random component means ranged damage has natural variance even without crits.

---

## 3. Hit Rate — Vulture's Eye & Fog Wall (line 3264)

**Bonus (RENEWAL only):**
```cpp
#ifdef RENEWAL
if (sd)
    hitrate += pc_checkskill(sd, AC_VULTURE);  // +1 hit per level of Vulture's Eye
#endif
```

**Penalty:**
```cpp
if ((wd.flag & (BF_LONG|BF_MAGIC)) == BF_LONG && !skill_id && tsc->getSCE(SC_FOGWALL))
    hitrate -= 50;  // Fog Wall: -50 hit on basic ranged attacks only
```

---

## 4. Long-Range Damage Bonuses

### Attacker bonus — `sd->bonus.long_attack_atk_rate`

Applied during damage assembly (RENEWAL, line 7047):
```cpp
if (wd.flag & BF_LONG && skill_id != RA_WUGBITE && skill_id != RA_WUGSTRIKE)
    ATK_ADDRATE(wd.damage, wd.damage2, sd->bonus.long_attack_atk_rate);
```

In pre-renewal, this is applied inside `battle_calc_cardfix()` instead:
```cpp
if (flag & BF_LONG)
    cardfix = cardfix * (100 + sd->bonus.long_attack_atk_rate) / 100;
```

`long_attack_atk_rate` is set by cards and equipment (e.g. Archer Skeleton Card: +10% ranged damage).

### Target resistance — `near_attack_def_rate` / `long_attack_def_rate`

Some cards and items set resistance specifically against short or long-range attacks. These are checked via the `BF_RANGEMASK` portion of card resist entries:

```cpp
// In battle_calc_cardfix target-side:
for (const auto &it : tsd->subele2) {
    if (!(((it.flag)&flag)&BF_RANGEMASK))  // Must match BF_LONG or BF_SHORT
        continue;
    ele_fix += it.rate;
}
```

---

## 5. Target Status Effects That Block Ranged Attacks

Several SC block or penalise `BF_LONG` attacks specifically (checked in `battle_calc_damage`, line 1475):

| Status on target | Effect |
|-----------------|--------|
| `SC_PNEUMA` | Blocks all `BF_LONG` non-magic attacks completely |
| `SC_NEUTRALBARRIER` | Blocks `BF_LONG` non-magic (hit check returns false) |
| `SC_ADJUSTMENT` | Reduces ranged damage: `−(BF_LONG|BF_WEAPON)` penalty |
| `SC_TATAMIGAESHI` | Blocks `BF_LONG` non-magic attacks |
| `SC_LIGHTNINGWALK` | On `BF_LONG` non-magic hit: teleports target next to attacker |
| `SC_DODGE` | +20% dodge chance against `BF_LONG` |

```cpp
// Example: Pneuma check in battle_calc_damage
if (sc->getSCE(SC_PNEUMA) && (flag & (BF_MAGIC|BF_LONG)) == BF_LONG)
    return 0;  // No damage

// Neutral Barrier in is_attack_hitting
if (tsc->getSCE(SC_NEUTRALBARRIER) && (wd->flag & (BF_LONG|BF_MAGIC)) == BF_LONG)
    return false;
```

---

## 6. Attacker Status Effects for Ranged

| Status on attacker | Effect |
|-------------------|--------|
| `SC_TRUESIGHT` | Adds HIT bonus (also increases ranged crit chance) |
| `SC_DANCEWITHWUG` | Increases `BF_LONG|BF_WEAPON` damage |
| `SC_HOLY_OIL` | Extra damage on `BF_LONG|BF_WEAPON` hits |

```cpp
// Dance with Wug ranged bonus (line 2006):
if (sc->getSCE(SC_DANCEWITHWUG) && (flag&(BF_LONG|BF_WEAPON)) == (BF_LONG|BF_WEAPON))
    ATK_ADDRATE(wd.damage, wd.damage2, bonus);
```

---

## 7. Card Fix — BF_LONG Category

`battle_calc_cardfix()` applies attacker range bonuses in pre-renewal inside the card fix loop:

```cpp
// Pre-re only (RENEWAL uses the assembly step instead):
if (flag & BF_SHORT) cardfix = cardfix * (100 + sd->bonus.short_attack_atk_rate) / 100;
if (flag & BF_LONG)  cardfix = cardfix * (100 + sd->bonus.long_attack_atk_rate) / 100;
```

Target-side long-range resist cards (`subele2`, `subrace2` etc.) all carry the `BF_RANGEMASK` flag and are filtered per-hit.

---

## 8. Monster Mode — Ignore Ranged (`MD_IGNORERANGED`, line 2893)

```cpp
if (status_has_mode(tstatus, MD_IGNORERANGED) && (flag & (BF_WEAPON|BF_LONG)) == (BF_WEAPON|BF_LONG))
    return 0;  // Monster is immune to ranged weapon attacks
```

Bosses and specific monsters can have this mode set in `mob_db.yml` to ignore ranged physical entirely.

---

## 9. Skill Ratio Differences for Ranged Skills

Some skills have ranged-specific ratio modifiers inside `battle_calc_attack_skill_ratio()`. Examples:

```cpp
case AC_SHOWER:      skillratio += -25 + 5*skill_lv;     // pre-re
                     skillratio += 50 + 10*skill_lv;     // renewal
case AC_DOUBLE:      skillratio += 10*(skill_lv-1);
case GS_DESPERADO:   skillratio += -30 + 10*skill_lv;
case SN_SHARPSHOOTING:
    skillratio += 100 + 50*skill_lv;
    // +crit rate bonus also applied here
```

---

## Summary of Ranged-Only Differences

| Aspect | Melee (`BF_SHORT`) | Ranged (`BF_LONG`) |
|--------|-------------------|--------------------|
| Range flag | `BF_SHORT` | `BF_LONG` |
| Arrow ATK | Not used | `rnd(0, arrow_atk)` added to weaponAtk |
| Hit bonus | None | +Vulture's Eye level (RENEWAL) |
| Hit penalty | None | −50 vs Fog Wall (basic attack only) |
| Damage bonus | `short_attack_atk_rate` | `long_attack_atk_rate` |
| Blocked by | None (range-specific) | Pneuma, Neutral Barrier, Tatami |
| Monster immune | None | `MD_IGNORERANGED` |
| Card resist filters | `BF_SHORT` entries | `BF_LONG` entries |

Everything else — DEF formula, element table, card fix, skill ratio — is identical to the melee pipeline.
