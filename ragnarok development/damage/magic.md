# Magic Damage Pipeline (`BF_MAGIC`)

All magic attacks — bolt skills, AoE spells, and support-converted damage — flow through `battle_calc_magic_attack()`. Unlike the physical pipeline, magic damage uses MATK instead of ATK, MDEF instead of DEF, and has no hit/flee check by default.

**Source file:** `src/map/battle.cpp` — `battle_calc_magic_attack()` starts at line 7311.

---

## Pipeline at a Glance

```
battle_calc_attack()
  └─ battle_calc_magic_attack()
       ├─ 1. Struct initialization           — div_, amotion, element, flags
       ├─ 2. battle_get_magic_element()      — determine spell element
       ├─ 3. Infinite defense check          — plant/boss mode → damage = 1
       ├─ 4. Base MATK calculation           — matk_min ~ matk_max random
       ├─ 5. Skill-specific formula switch   — overrides base for special skills
       ├─ 6. NK_SPLASHSPLIT                  — divide MATK among multiple targets
       ├─ 7. calculateSkillRatio() / ratio switch — ×skillratio/100
       ├─ 8. S.MATK bonus (RENEWAL)          — ×(1 + smatk/100)
       ├─ 9. ignore_mdef checks              — flag.imdef if applicable
       ├─ 10. pc_skillatk_bonus (pre-re)     — skill-specific % bonus
       ├─ 11. MRes reduction (RENEWAL)       — mres/(mres+400) × 80%
       ├─ 12. MDEF reduction                 — RE: atk×(1000+eMDEF)/(1000+10×eMDEF)−sMDEF
       ├─ 13. pc_skillatk_bonus (RENEWAL)    — applied after MDEF
       ├─ 14. battle_calc_cardfix()          — element/race/class card multipliers
       ├─ 15. battle_attr_fix()              — elemental table multiplier
       ├─ 16. battle_calc_damage()           — target SC blocks / GVG rates
       └─ 17. battle_absorb_damage() / battle_do_reflect()
```

---

## Step 1 — Dispatcher (`battle_calc_attack`, line 8782)

```cpp
case BF_MAGIC: d = battle_calc_magic_attack(bl, target, skill_id, skill_lv, flag); break;
```

---

## Step 2 — Struct Initialization (line 7336)

```cpp
ad.damage   = 1;          // default for plant targets (infinite defense)
ad.div_     = skill_get_num(skill_id, skill_lv);   // hit count (1-10 for bolt skills)
ad.amotion  = sstatus->amotion;                    // 0 for ground skills
ad.dmotion  = tstatus->dmotion;
ad.blewcount = skill_get_blewcount(skill_id, skill_lv);
ad.flag     = BF_MAGIC | BF_SKILL;
ad.miscflag = mflag;
ad.dmg_lv   = ATK_DEF;
```

`div_` drives multi-hit display (bolt count). A negative `div_` means each hit does the full damage independently rather than splitting the total.

---

## Step 3 — Element Determination (`battle_get_magic_element`, line 7362)

```cpp
s_ele = battle_get_magic_element(src, target, skill_id, skill_lv, mflag);
```

Determines the casting element for the spell. Sources in priority order:
1. Skill's own element (from `skill_db.yml` `Element:` field)
2. Endow/converter status (e.g. `SC_ENCPOISON`, `SC_FIREWEAPON`)
3. Neutral if nothing applies

The element is later used in `battle_attr_fix()` to look up the elemental table.

---

## Step 4 — Infinite Defense Check (line 7418)

```cpp
flag.infdef = is_infinite_defense(target, ad.flag) ? 1 : 0;
```

If true (plant, emperium in some modes, boss with `MD_NOKNOCKBACK`), the final damage is capped to 1. All intermediate calculations are skipped.

---

## Step 5 — Base MATK (line 7529, inside `default:` of skill switch)

For most skills, base damage is a random roll between `matk_min` and `matk_max`:

```cpp
if (sstatus->matk_max > sstatus->matk_min)
    MATK_ADD(sstatus->matk_min + rnd() % (sstatus->matk_max - sstatus->matk_min));
else
    MATK_ADD(sstatus->matk_min);
```

`matk_min` / `matk_max` come from `status_calc_bl_main()` and are derived from:

```
MATK_base = INT² / 5  (pre-re)
MATK = MATK_base + equipment_MATK + card_MATK_bonuses
MATK range = [MATK × (1 − refinement_variance), MATK]
```

In RENEWAL, `S.MATK` (a separate stat from `sstatus->smatk`) is applied as a multiplier after `skillratio`.

---

## Step 6 — Skill-Specific Formula Overrides (line 7450)

Skills that do not use the standard MATK base have explicit cases. Examples:

| Skill | Formula |
|-------|---------|
| `AL_HEAL` / `PR_SANCTUARY` | `skill_calc_heal()` — INT + heal bonus |
| `PR_TURNUNDEAD` | RNG instant-kill check; on fail: `MATK × skill_lv` |
| `PF_SOULBURN` | `target_SP × 2` |
| `NPC_DARKBREATH` | `target_HP × (skill_lv scaling)%` |
| `AB_RENOVATIO` | `src_base_level × 10 + INT` |
| `NPC_EARTHQUAKE` | `STR×2 + weapon_attack` (RENEWAL) |

---

## Step 7 — Splash Split (`NK_SPLASHSPLIT`, line 7535)

```cpp
if (nk[NK_SPLASHSPLIT]) {
    if (mflag > 0)
        ad.damage /= mflag;  // divide MATK equally among all targets hit
}
```

`mflag` holds the number of targets when `NK_SPLASHSPLIT` is active (e.g. Storm Gust, Lord of Vermilion). Each target receives `MATK / target_count`.

---

## Step 8 — Skill Ratio (line 7543)

```cpp
// From the skill's implementation class:
skill->impl->calculateSkillRatio(&ad, src, target, skill_lv, skillratio, mflag);

// Then inline adjustments in the default switch, e.g.:
case LG_RAYOFGENESIS:
    skillratio += -100 + 350 * skill_lv;
    skillratio += sstatus->int_ * 3;
    RE_LVL_DMOD(100);   // += (base_level - 100) / 100
    break;

MATK_RATE(skillratio);  // ad.damage = ad.damage * skillratio / 100
```

`skillratio` starts at 100 (100% = no change). Most offensive spells add to it. `RE_LVL_DMOD(x)` adds `(caster_base_level − 100) / x` to the ratio — a RENEWAL level scaling bonus.

---

## Step 9 — S.MATK Multiplier (RENEWAL, line 8044)

```cpp
#ifdef RENEWAL
if (sd && sstatus->smatk > 0)
    ad.damage += ad.damage * sstatus->smatk / 100;
MATK_RATE(skillratio);  // skill ratio applied after S.MATK
#endif
```

`smatk` (Spell Attack stat) is a RENEWAL-only stat sourced from equipment and base level. It acts as an additive % multiplier on top of the MATK base before the skill ratio is applied.

---

## Step 10 — MDEF Ignore Check (line 8059)

```cpp
if (!flag.imdef && (
    sd->bonus.ignore_mdef_ele   & (1 << tstatus->def_ele) ||
    sd->bonus.ignore_mdef_race  & (1 << tstatus->race)    ||
    sd->bonus.ignore_mdef_class & (1 << tstatus->class_)
))
    flag.imdef = 1;
```

`flag.imdef = 1` skips the MDEF reduction step entirely. Set by `NK_IGNOREDEFENSE` in the skill database, or by player bonuses from cards/equipment.

---

## Step 11 — Magic Resistance (RENEWAL, line 8072)

```
damage -= (MRes / (MRes + 400)) × 80% × damage
```

`tstatus->mres` is the magic resistance stat (analogous to RES for physical). Applied **before** MDEF, unlike physical where RES is applied after DEF. Capped via `max_res_mres_ignored` (default 50%).

---

## Step 12 — MDEF Reduction (line 8099)

### RENEWAL Formula

```
damage = damage × (1000 + eMDEF) / (1000 + 10×eMDEF) − sMDEF
```

| eMDEF value | % of damage that passes |
|-------------|------------------------|
| 0 | 100% |
| 100 | ~73% |
| 500 | 33% |
| 1000 | 17% |
| ∞ | 10% (floor) |

- **eMDEF** = `tstatus->mdef` (equipment MDEF) — reduced by attacker's `ignore_mdef_by_race`
- **sMDEF** = `tstatus->mdef2` (soft/INT-based MDEF) — flat subtraction

### Pre-Renewal Formula

```cpp
if (battle_config.magic_defense_type)
    damage = damage - mdef * magic_defense_type - mdef2;  // flat subtraction mode
else
    damage = damage * (100 - mdef) / 100 - mdef2;         // default: % then flat
```

`battle_config.magic_defense_type` controls which formula is used (0 = % mode, 1+ = flat).

---

## Step 13 — `pc_skillatk_bonus` (line 8151)

```cpp
// RENEWAL: applied after MDEF reduction
if (sd != nullptr)
    i += pc_skillatk_bonus(sd, skill_id);
ad.damage += ad.damage * i / 100;
```

In pre-renewal this is applied before MDEF instead. Contains bonuses from equipment that grant % damage boosts to specific skills.

---

## Step 14 — Card Fix (`battle_calc_cardfix` for BF_MAGIC, line 8665)

Magic attacks call a simplified version of cardfix. The target-side resist cards check against `BF_MAGIC` in their flags:

```cpp
md.damage += battle_calc_cardfix(BF_MISC, src, target, nk, s_ele, 0, md.damage, 0, md.flag);
```

For magic, relevant card categories:
- **Attacker**: `MATK%` cards, element-boost cards (`addele[ELE_FIRE]` etc.)
- **Target**: `subele[ELE_FIRE]` (elemental resist), `subrace[RC_DEMON]` (race resist), `submagic%` (generic magic resist)

The `BF_MAGIC` flag in resist entries means that card only applies against magic attacks specifically.

---

## Step 15 — Element Fix (`battle_attr_fix`, line 511)

Identical lookup to physical damage but using the **spell element** (`s_ele`) rather than weapon element:

```cpp
if (!nk[NK_IGNOREELEMENT])
    ad.damage = battle_attr_fix(src, target, ad.damage, s_ele, tstatus->def_ele, tstatus->ele_lv);
```

Status effects that modify elemental ratios for magic:

| Attacker status | Effect |
|----------------|--------|
| `SC_VOLCANO` | +ratio for Fire spells |
| `SC_VIOLENTGALE` | +ratio for Wind spells |
| `SC_DELUGE` | +ratio for Water spells |

| Target status | Effect |
|--------------|--------|
| `SC_SPIDERWEB` | Fire damage +100% (×2) |
| `SC_EARTH_INSIGNIA` | Fire damage +50% |
| `SC_ORATIO` | Holy damage +2% per Oratio level |
| `SC_FREEZING` | Specific skills deal greatly increased damage |

---

## Step 16 — Final Steps

| Step | Notes |
|------|-------|
| `battle_calc_damage()` | Applies SC blocks, GVG/BG rates, damage multipliers |
| `battle_skill_damage()` | Global skill damage override table |
| `battle_absorb_damage()` | HP/SP absorb items/cards |
| `battle_do_reflect()` | `SC_KAITE`, `SC_MAGICMIRROR` — reflects magic back at caster |

### Magic Reflection (`SC_KAITE`, `SC_MAGICMIRROR`)

Magic-specific reflection is checked in `battle_do_reflect(BF_MAGIC, ...)`. Physical damage is never reflected this way. Reflected damage hits the original caster using the same element.

---

## Notable Interactions

### No Default Hit/Flee Check

Magic attacks do not roll against the target's flee. There is no `is_attack_hitting()` call. Accuracy is guaranteed unless a skill sets `NK_IGNOREFLEE` override or a target SC causes special handling.

### Multi-Hit (`div_`)

For bolt skills (`MG_FIREBOLT` Lv10: `div_ = 10`):
- `ad.damage` is the per-bolt damage
- The client receives `count = div_` and renders each bolt hit separately
- Each bolt is independent — no splitting of total damage
- A negative `div_` (e.g. `WZ_FIREPILLAR`) means each hit is the full `ad.damage`, not divided

### Plant Damage

Plants and similar have `is_infinite_defense() = true`. All magic attacks deal exactly 1 damage to them regardless of MATK, element, or skill ratio.

---

## Formula Summary (RENEWAL)

```
base_matk  = rnd(matk_min, matk_max)
with_smatk = base_matk × (1 + smatk/100)
with_ratio = with_smatk × skillratio / 100
with_mres  = with_ratio − (MRes/(MRes+400)) × 0.8 × with_ratio
with_mdef  = with_mres × (1000+eMDEF)/(1000+10×eMDEF) − sMDEF
with_cards = with_mdef × attacker_cards × (1 − target_cards)
final      = battle_attr_fix(with_cards, spell_element, def_element, def_level)
```

---

## Key Functions Reference

| Function | Line | Purpose |
|----------|------|---------|
| `battle_calc_magic_attack()` | 7311 | Magic damage pipeline |
| `battle_get_magic_element()` | ~7362 | Determine spell element |
| `is_infinite_defense()` | — | Plant/boss check |
| `MATK_RATE(x)` | macro | `damage = damage * x / 100` |
| `MATK_ADDRATE(x)` | macro | `damage += damage * x / 100` |
| `MATK_ADD(x)` | macro | `damage += x` |
| `RE_LVL_DMOD(x)` | macro | `skillratio += (base_level−100)/x` |
| `battle_calc_cardfix()` | 769 | Card multipliers |
| `battle_attr_fix()` | 511 | Elemental table lookup |
| `battle_do_reflect()` | — | Magic reflection (Kaite, etc.) |
