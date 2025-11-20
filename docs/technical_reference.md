# Dynamic Research Slots - Internal Logic

> **Document Note**: This documentation was created with AI assistance to provide more comprehensive and detailed coverage, but has been manually reviewed and controlled by a human. The technical information has been verified for accuracy, but if you notice any errors or have suggestions for improvement, please report them.

This document gives a technical overview of how **Dynamic Research Slots Reborn** works under the hood and how you can extend or rebalance it.

The focus is on:
- where the logic is implemented,
- which variables and arrays are used,
- how **Research Power (RP)** is calculated,
- how **Easy Research Slots** are derived,
- how the **Custom Game Rules** feed into the calculation,
- and which **debug tools** exist.

---

## 1. Entry points & script files

The core of the system lives in the following files:

- `common/on_actions/00_dynamic_research_slots_on_actions.txt`  
  Hooks the system into the game flow (startup and daily tick) and calls the scripted effects.

- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`  
  Contains the full RP calculation, research-slot thresholds, dynamic modifiers and events for the player/AI.

- `common/scripted_effects/00_dr_dynamic_research_config.txt`  
  Central **configuration script**: defines default RP weights, slot thresholds and Easy Slot behaviour.  
  This file is the main extension point for **submods** that want to rebalance the system.

- `common/scripted_effects/00_dr_mod_metadata.txt`  
  Mod metadata (version, compatibility signals). Called at the very start of initialization.

- `common/decisions/dynamic_research_slots_decisions.txt`  
  Provides the help decision and several debug decisions to (re)initialize or recalculate the system.

- `events/dynamic_research_slot_events.txt`  
  Implements the help popup and the notification events when the number of research slots changes.

- `common/game_rules/00_dr_dynamic_research_rules.txt`  
  Defines all **Custom Game Rules** that influence Easy Slots, factory weights, war/alliance modifiers and law-based modifiers.

- `scripted_localisation/00_dynamic_research_slots_loc.txt`  
  Simple scripted localisation that adds helper notes to some game rules depending on their settings.

---

## 2. Update cycle (on_actions)

File: `common/on_actions/00_dynamic_research_slots_on_actions.txt`

### Game start (`on_startup`)

At game start, every country runs:

- `initialize_dynamic_research_slots = yes`
- `calculate_modifiers_to_rp = yes`

This sets up all internal variables and the RP thresholds.

**AI countries** additionally:
- Initialize the staggered update timer (`dr_days_until_update`) with a random offset (1–14 days by default, configurable via `dr_ai_update_frequency`)
- Run `recalculate_dynamic_research_slots = yes` to set correct research slots from day 1

This ensures AI countries have accurate research slots immediately at game start, before their first staggered update cycle begins.

A one-time startup explanation event (`dynamic_research_slots.6`) is shown to human players.

### Daily update (`on_daily`)

Every in-game day:

- `dr_days_in_war` is updated (counts how long the country has been at war). Resets to 0 when at peace.
- New countries that appear later in the game get initialized once via `initialize_dynamic_research_slots`.
- Decrease event cooldown (`dr_player_event_cooldown`) if present.
- Calculate modifiers (`calculate_modifiers_to_rp`).
- Increment `dr_days_since_last_full_check`.
- **Smart check:** `dr_recalculate_if_needed` (only recalculates if factories changed or 7 days passed).

**Note:** The Smart Detection System *(since version 1.5)* replaces the previous daily recalculation for both players and AI, significantly reducing performance overhead while maintaining responsiveness.

AI countries:
- **Note**: AI countries get an initial calculation at startup (see `on_startup` above) to ensure correct research slots from day 1.
- Use a staggered update for subsequent calculations: `dr_days_until_update` counts down from a random offset (1–14 days by default, configurable via `dr_ai_update_frequency`, initialized at startup), then triggers:
  - `calculate_modifiers_to_rp`
  - `recalculate_dynamic_research_slots`
- The timer resets to `dr_ai_update_frequency` (14 days by default) after each update cycle.
- This staggered approach reduces performance overhead while maintaining accurate research slots.
- The update frequency can be customized via the `dr_ai_update_frequency` config variable (set in `dr_reset_research_config_defaults`, overrideable via `dr_apply_research_config_submods`).

### 2.2. Smart Detection System *(since version 1.5)*

The mod uses an intelligent detection system that only triggers recalculation when necessary:

**`dr_check_for_factory_changes`** - Checks if factory counts (Civ/Mil/Nav) have changed since last check:
- Compares `dr_last_civ_count`, `dr_last_mil_count`, `dr_last_nav_count` with current factory counts
- Uses early exit strategy (stops checking once a change is found)
- Sets `dr_factories_changed` flag if any factory type changed

**`dr_recalculate_if_needed`** - Conditional recalculation wrapper:
- Runs `dr_check_for_factory_changes` first
- Checks if 7 days have passed since last full check (`dr_days_since_last_full_check >= 7`)
- Only calls `recalculate_dynamic_research_slots` if:
  - Factories changed OR
  - 7 days have passed (weekly fallback for custom buildings)
- Resets `dr_days_since_last_full_check` after recalculation

**Benefits:**
- Reduces unnecessary recalculations by ~75-85% in typical gameplay
- Maintains responsiveness (immediate update when factories change)
- Weekly fallback ensures custom buildings from submods are still detected
- Works seamlessly for both players and AI

**Variables:**
- `dr_last_civ_count` - Last known civilian factory count
- `dr_last_mil_count` - Last known military factory count
- `dr_last_nav_count` - Last known naval factory count
- `dr_days_since_last_full_check` - Days since last full recalculation
- `dr_factories_changed` - Flag set when factory counts change
- `dr_force_weekly_update` - Flag set when 7-day fallback triggers

---

### 2.1 Extension hooks (quick reference)

To keep submods compatible, the system exposes several empty scripted effects that run at fixed points. You can override any of them in your own mod; they incur zero cost if left untouched.

**Hook execution flow**:

```
INITIALIZATION (once per country at game start):
┌─────────────────────────────────────────────────────────────┐
│ initialize_dynamic_research_slots                           │
│   │                                                           │
│   ├─► dr_check_compatibility_submods  ← Hook 0              │
│   │    (before config - rare use case)                       │
│   │                                                           │
│   ├─► dr_apply_research_config                               │
│   │    └─► dr_apply_research_config_submods  ← Hook 2       │
│   │         (config tweaks)                                  │
│   │                                                           │
│   └─► dr_initialize_submods  ← Hook 1                      │
│        (after config - common use case)                     │
└─────────────────────────────────────────────────────────────┘

RUNTIME (daily for players, staggered for AI):
┌─────────────────────────────────────────────────────────────┐
│ recalculate_dynamic_research_slots                          │
│   │                                                           │
│   ├─► Easy Slot logic & threshold rebuilding                │
│   │    └─► dr_adjust_research_thresholds_submods  ← Hook 4  │
│   │                                                           │
│   ├─► dr_apply_factory_modifiers_submods  ← Hook 3          │
│   │    (set civ/mil/nav modifiers)                           │
│   │                                                           │
│   ├─► Factory RP calculation                                │
│   │    (civ × count × modifier + mil × count × modifier...) │
│   │                                                           │
│   ├─► Count vanilla facilities                               │
│   │    (nuclear/naval/air/land via every_owned_state)       │
│   │    └─► dr_collect_facility_counts_submods  ← Hook 5     │
│   │         (count custom buildings)                          │
│   │                                                           │
│   ├─► Apply vanilla facility RP                              │
│   │    └─► dr_apply_facility_rp_submods  ← Hook 6           │
│   │         (convert custom counts to RP)                     │
│   │                                                           │
│   ├─► Apply global modifiers                                 │
│   │    (war/alliance/law modifiers)                          │
│   │    └─► dr_total_rp_modifier_submods  ← Hook 7           │
│   │         (final RP adjustments)                          │
│   │                                                           │
│   └─► Calculate target slots & apply changes                 │
│        (compare total_research_power to thresholds)         │
└─────────────────────────────────────────────────────────────┘
```

| Hook | Called from | Timing | Purpose |
|------|-------------|--------|---------|
| `dr_check_compatibility_submods` | `initialize_dynamic_research_slots` | Very start of initialization (before config) | Perform compatibility checks, signal mod presence, or set flags before config is applied. |
| `dr_initialize_submods` | `initialize_dynamic_research_slots` | Once per init (after config is applied) | Initialize custom variables, set flags, or perform setup tasks. |
| `dr_apply_research_config_submods` | `dr_apply_research_config` | Once per init/re-init | Final chance to tweak config variables after vanilla defaults + game rules. |
| `dr_apply_factory_modifiers_submods` | `recalculate_dynamic_research_slots` | Before factory RP calculation | Set factory modifiers (`civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`) based on ideas, national spirits, focuses, etc. |
| `dr_adjust_research_thresholds_submods` | `recalculate_dynamic_research_slots` | After Easy Slot logic is applied | Adjust research slot thresholds dynamically (e.g., difficulty-based scaling, country-specific tweaks). |
| `dr_collect_facility_counts_submods` | `recalculate_dynamic_research_slots` | Every recalculation | Adjust the facility counters after the vanilla `every_owned_state` loop (e.g., count custom buildings). |
| `dr_apply_facility_rp_submods` | `recalculate_dynamic_research_slots` | Every recalculation | Convert the (vanilla or custom) counters into extra RP, or add new flat RP sources. |
| `dr_total_rp_modifier_submods` | `recalculate_dynamic_research_slots` | Every recalculation | Tweak the final `total_research_power` after all global modifiers have been applied. |
| `dr_get_opinion_factor_for_ally_submods` *(since v1.4)* | `dr_apply_alliance_rp_logic` | During alliance RP calculation (per ally) | Override opinion factor calculation (0.0 to 1.0) for alliance members. Must set `dr_opinion_factor_custom_set` flag when providing custom value. |

Hook call order during recalculation:

1. Easy Slot adjustments and threshold rebuilding.
2. `dr_adjust_research_thresholds_submods`.
3. Built-in facility counters are zeroed and filled via `every_owned_state`.
4. `dr_apply_factory_modifiers_submods`.
5. Factory RP calculation (base × count × modifier).
6. `dr_collect_facility_counts_submods`.
7. Vanilla RP from facilities is added (if enabled).
8. `dr_apply_facility_rp_submods` (only if facility RP is enabled).
9. Built-in global modifier is applied (`total_rp_modifier`).
10. `dr_total_rp_modifier_submods`.
11. Slot threshold checks and slot application.

**Note**: During alliance RP calculation (step 9, within `dr_apply_alliance_rp_logic`), the hook `dr_get_opinion_factor_for_ally_submods` is called for each alliance member to allow custom opinion factor calculation.

This makes it clear where to plug custom logic without touching the core script.

---

## 3. Core variables & arrays

Most of the internal state is stored in country variables. The most important ones:

- `current_research_slots` – how many research slots the country currently has in vanilla terms.
- `target_research_slots` – how many slots the system *wants* the country to have based on its RP.
- `easy_research_slots` – number of slots that use reduced RP thresholds (Easy Slots).
- `removed_research_slots` – number of slots that were removed (for bookkeeping and potential future extensions).

Factory weights and RP components:

- `research_power_per_civ` – RP per civilian factory.
- `research_power_per_mil` – RP per military factory.
- `research_power_per_nav` – RP per dockyard.
- `base_civilian_research_power`, `base_military_research_power`, `base_naval_research_power` – unmodified RP from each factory type.
- `civilian_research_power`, `military_research_power`, `naval_research_power` – RP from factories after applying modifiers.
- `civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier` – multiplicative modifiers for factory types (default 0, can be set by submods).

Total RP:

- `total_research_power` – final RP used to determine how many slots you should have.
- `facility_research_power` – share of RP that comes from experimental facilities (nuclear, naval, air, land).

Facility counts:

- `nuclear_facility_count`, `naval_facility_count`, `air_facility_count`, `land_facility_count` – number of each facility type (counted per state).
- `rp_per_nuclear_facility`, `rp_per_naval_facility`, `rp_per_air_facility`, `rp_per_land_facility` – RP per facility type (default: 30, 15, 15, 15).

Easy slots:

- `easy_research_slot_coefficient` – factor by which thresholds for Easy Slots are reduced (default 0.6 = 60% of normal cost).

War / alliance / law modifiers:

- `war_rp_base_penalty` – base maximum war-time RP penalty from the game rule.
- `war_penalty_factor_ws` – contribution of war support to the penalty.
- `war_penalty_factor_stab` – contribution of stability.
- `war_penalty_factor_party` – contribution of ruling party support.
- `war_phase_factor` – phase-based factor depending on war duration (`dr_days_in_war`).
- `war_defense_factor` – defensive war protection factor.
- `war_penalty_factor_total` – combined war penalty factor.
- `war_rp_penalty_effective` – effective RP penalty fraction applied in the end.
- `dr_days_in_war` – counter for how long the country has been at war.

- `total_rp_modifier` – overall additive RP modifier (includes war/alliance/law effects).  
  This is applied on top of the summed factory/facility RP.

Alliance variables *(updated in version 1.4)*:

- `alliance_rel_factor` *(since version 1.4)* – opinion factor for a single alliance member (0.0 to 1.0, calculated per member via `dr_get_opinion_factor_for_ally` helper effect or custom hook `dr_get_opinion_factor_for_ally_submods`).
- `alliance_member_count` – number of faction members (excluding self).
- `alliance_rel_factor_sum` – sum of relation factors across all allies.
- `alliance_avg_rel_factor` – average relation factor (0-1).
- `alliance_bonus_cap` – maximum bonus allowed by game rule.
- `alliance_bonus_pct` – effective alliance bonus percentage.
- `dr_days_until_alliance_update` – timer to stagger alliance bonus recalculations (updates roughly once per week).
- `dr_opinion_factor_custom_set` *(since version 1.4)* – flag set by submods when overriding opinion factor calculation via `dr_get_opinion_factor_for_ally_submods` hook.

Arrays:

- `base_research_for_slot[index]` – baseline RP thresholds for each slot.
- `research_for_slot[index]` – effective thresholds actually used, after Easy Slot adjustments.
- `max_research_slots` – RP threshold of the highest defined research slot (used as helper for UI logic).
- `next_research_slot_at` – RP threshold for the next slot above `target_research_slots`.

Timers and cooldowns:

- `dr_player_event_cooldown` – cooldown timer for player notification events (14 days).
- `dr_days_until_update` – AI update timer (1-14 days by default, staggered, configurable via `dr_ai_update_frequency`).
- `dr_ai_update_frequency` – Configuration variable for AI update frequency (default: 14 days, set in `dr_reset_research_config_defaults`).
- `dr_days_until_alliance_update` – alliance bonus update timer (7 days).

---

## 4. Initialization & thresholds

File: `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`

### `initialize_dynamic_research_slots`

Key steps:

1. Marks the country as initialized:  
   - `set_country_flag = dynamic_research_slots_initialized`

2. Sets mod metadata (must be first):
   - Calls `dr_set_mod_metadata` which sets:
     - `set_global_flag = dynamic_research_slots_active` (once globally)
     - `set_variable = { dr_mod_version = 1.4 }` (per country)

3. Sets base slot counts:
   - `current_research_slots = amount_research_slots`
   - `target_research_slots = 2`
   - `removed_research_slots = 0`

4. Calls compatibility check hook:
   - `dr_check_compatibility_submods = yes` (before config, allows early opt-out)

5. Applies configuration:
   - `dr_apply_research_config = yes` (sets RP weights, thresholds, Easy Slots, facility RP)

6. Handles initial slot adjustment:
   - If the country starts with more than 2 slots, the extra slots are converted to Easy Slots
   - `removed_research_slots` tracks how many slots were converted
   - `easy_research_slots` is increased by the number of converted slots

7. Rebuilds effective thresholds:
   - `dr_rebuild_research_thresholds = yes` (adjusts `research_for_slot` based on Easy Slot settings)

8. Calls initialization hook:
   - `dr_initialize_submods = yes` (after config, allows submods to set up custom variables)

### `dr_rebuild_research_thresholds`

This helper effect adjusts `research_for_slot[index]` based on `base_research_for_slot[index]`, `easy_research_slots`, and `easy_research_slot_coefficient`:

- For indices `i` that are `<= easy_research_slots`, the threshold is reduced:
  - `research_for_slot[i] = base_research_for_slot[i] * easy_research_slot_coefficient`.
- For higher slots, the baseline cost is unchanged:
  - `research_for_slot[i] = base_research_for_slot[i]`.

This effect is called:
- After initialization
- After Easy Slot changes (when slots are added/removed)
- After any threshold adjustments from submods

The same helper is also called during `recalculate_dynamic_research_slots`, so any changes to `base_research_for_slot` (e.g. from submods) are consistently reflected whenever thresholds are rebuilt. This way the first few slots are easier to acquire but later slots still require full RP.

### Configuration defaults

File: `common/scripted_effects/00_dr_dynamic_research_config.txt`

Default RP weights per factory type:
- `research_power_per_civ = 3`
- `research_power_per_mil = 2`
- `research_power_per_nav = 2`

These can be overridden by the **Factory Weights** game rule:
- Industry-focused: 4/2/1 (civ/mil/nav)
- Military-focused: 2/4/2
- Naval-focused: 2/2/4

Default RP per facility:
- `rp_per_nuclear_facility = 30`
- `rp_per_naval_facility = 15`
- `rp_per_air_facility = 15`
- `rp_per_land_facility = 15`

Default RP per nuclear reactor:
- `rp_per_nuclear_reactor = 20`
- `rp_per_heavy_water_reactor = 35`
- `rp_per_commercial_reactor = 12`

Government support thresholds for war/peace RP modifiers:
- `government_support_threshold_very_low = 0.25`
- `government_support_threshold_low = 0.4`
- `government_support_threshold_medium = 0.55`
- `government_support_threshold_high = 0.7`
- `government_support_threshold_very_high = 0.85`

These thresholds determine the government support factor used in war penalty and peace bonus calculations. The factor ranges from 0.0 (best, very high support) to 0.4 (worst, very low support). They can be adjusted in the config file without code changes.

Baseline RP thresholds for each slot in `base_research_for_slot` and `research_for_slot`:

- Slots 1–2: `0` (free, i.e. vanilla starting slots).
- Slot 3: `50`
- Slot 4: `200`
- Slot 5: `400`
- Slot 6: `700`
- Slot 7: `1000`
- Slot 8: `1500`
- Slot 9: `2000`
- Slot 10: `3000`

Default Easy Slot setup:
- `easy_research_slots = 2` (slots 1-2 are Easy Slots)
- `easy_research_slot_coefficient = 0.6` (60% of the normal threshold)

Easy Slot rules:
- **Easy Slots (Base)** game rule (`DR_EASY_SLOTS_RULE`):
  - `DR_EASY_SLOTS_PLUS_*` options add up to +5 Easy Slots on top of the default.
  - Optional extra Easy Slot for **non-majors** via `DR_MINOR_EASY_SLOTS_RULE`.
- **Easy Slot RP Cost Factor** rule (`DR_EASY_COST_RULE`):
  - `DR_EASY_COST_*` options set `easy_research_slot_coefficient` from `0.1` (very cheap) up to `1.0` (same as normal slots).
  - Default: `0.6` (60%)

---

## 5. Research Power calculation

The RP calculation happens mainly in:

- `calculate_modifiers_to_rp` – calculates war/peace/alliance/law modifiers
- the subsequent block in `recalculate_dynamic_research_slots` that uses the factory counts.
- the configuration effect `dr_apply_rp_modifier_logic` (file `common/scripted_effects/00_dr_dynamic_research_modifiers.txt`), which encapsulates the war/peace/law/alliance logic and can be overridden by submods. The implementation is split into three helper effects:
  - `dr_apply_war_rp_logic`
  - `dr_apply_law_rp_logic`
  - `dr_apply_alliance_rp_logic`

### 5.1 Factory contributions

For each factory type:

1. Start from the per-factory weight:
   - `research_power_per_civ`, `research_power_per_mil`, `research_power_per_nav`.

2. Multiply by the number of factories:
   - `num_of_civilian_factories`, `num_of_military_factories`, `num_of_naval_factories`.

3. Apply the corresponding RP modifier (via `dr_apply_factory_modifiers_submods` hook):
   - `civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`.
   - These are multiplicative modifiers: `final_rp = base_rp × (1 + modifier)`

4. Sum up:
   - `total_research_power = civilian_research_power + military_research_power + naval_research_power`.

The hook `dr_apply_factory_modifiers_submods` is called **before** the factory RP calculation, allowing submods to set these modifiers based on ideas, national spirits, focuses, etc.

### 5.2 Experimental facilities

If the country has any **Experimental Facilities**, additional flat RP is added **per facility**:

- Each nuclear facility: +30 RP (default).
- Each naval facility: +15 RP (default).
- Each air facility: +15 RP (default).
- Each land facility: +15 RP (default).

Additionally, the mod supports **Nuclear Reactors** with the following RP values:

- Each nuclear reactor (`nuclear_reactor`): +20 RP (with diminishing returns).
- Each heavy water nuclear reactor (`nuclear_reactor_heavy_water`): +35 RP (with diminishing returns).
- Each commercial nuclear reactor (`commercial_nuclear_reactor`): +12 RP (with diminishing returns).

All reactors use a reduction factor of 0.15 (15% reduction per additional reactor) to prevent excessive RP farming.

Because the building variables (`nuclear_facility`, `naval_facility`, `air_facility`, `land_facility`, `nuclear_reactor`, `nuclear_reactor_heavy_water`, `commercial_nuclear_reactor`) only exist at **state** scope, the script does not read them directly on the country. Instead `recalculate_dynamic_research_slots` runs an `every_owned_state` loop to count how many owned states have each facility type and stores these counts in the internal variables `nuclear_facility_count`, `naval_facility_count`, `air_facility_count`, `land_facility_count`, `nuclear_reactor_count`, `heavy_water_reactor_count` and `commercial_reactor_count`.

After the vanilla counting pass, the hook `dr_collect_facility_counts_submods` is called, allowing submods to count custom buildings or adjust the counts.

**Facility count validation** *(since version 1.4)*: After `dr_collect_facility_counts_submods`, the system validates that all facility counts (including reactor counts) are non-negative. If any count is negative, it is automatically set to 0 to prevent calculation errors.

These counters are then multiplied by the configured `rp_per_*_facility` and `rp_per_*_reactor` values from `00_dr_dynamic_research_config.txt` to obtain the flat RP bonus. The facility RP calculation uses two helper effects:

- `dr_apply_facility_rp_if_present` - Checks if a facility count is greater than 0 and applies RP calculation if present. This helper eliminates code duplication by providing a unified interface for all facility types.
- `dr_calculate_single_facility_rp` *(since version 1.4)* - Handles the reduction logic for all facility types, applying diminishing returns based on the number of facilities.

The hook `dr_apply_facility_rp_submods` is called after vanilla facility RP is added, allowing submods to convert custom facility counts into RP or add additional RP sources.

New configuration values (`experimental_facility_rp_reduction_nuclear`, `experimental_facility_rp_reduction_naval`, `experimental_facility_rp_reduction_air`, `experimental_facility_rp_reduction_land`) apply a per-facility reduction to each type. These defaults are `0.15` (15% reduction) and the core script applies a linear diminishing return directly: for `n` facilities the total RP is calculated as `n * base_rp * (1 - reduction * (n - 1) / 2)` (clamped at 0). This means:
- The first facility provides full RP (base value)
- Each additional facility reduces the multiplier, making later facilities less efficient
- At 15 facilities, the multiplier reaches 0 and no RP is generated
- The optimal number of facilities is typically around 7, where total RP peaks before diminishing returns make additional facilities counterproductive

After this calculation runs, the hook `dr_apply_experimental_facility_rp_scaling` is called so submods can further tweak `temp_facility_rp` without affecting saves.

This RP goes both into:
- `total_research_power`
- `facility_research_power` (for display/debug purposes).

Facility RP can be disabled per-country via the `dr_disable_facility_rp` flag.

#### 5.2.1 Diminishing Returns and Balance Analysis

The diminishing returns system for experimental facilities is designed to prevent excessive stacking while maintaining their value for strategic research investment.

**Mathematical Formula:**
- Multiplier = `1 - reduction * (n - 1) / 2`
- Total RP = `n * base_rp * multiplier`
- When multiplier < 0, it is clamped to 0

**With 15% reduction (0.15) and current base values:**

**Naval/Air/Land Facilities (15 RP base):**
- 1 facility: 15.0 RP (multiplier: 1.0)
- 2 facilities: 27.75 RP (multiplier: 0.925)
- 3 facilities: 38.25 RP (multiplier: 0.85)
- 4 facilities: 46.5 RP (multiplier: 0.775)
- 5 facilities: 52.5 RP (multiplier: 0.7)
- 6 facilities: 56.25 RP (multiplier: 0.625)
- 7 facilities: **57.75 RP** (multiplier: 0.55) - **Optimal point**
- 8 facilities: 57.0 RP (multiplier: 0.475) - Total RP starts decreasing
- 15 facilities: 0 RP (multiplier: 0.0) - No RP generated

**Nuclear Facilities (30 RP base):**
- 1 facility: 30.0 RP (multiplier: 1.0)
- 2 facilities: 55.5 RP (multiplier: 0.925)
- 3 facilities: 76.5 RP (multiplier: 0.85)
- 4 facilities: 93.0 RP (multiplier: 0.775)
- 5 facilities: 105.0 RP (multiplier: 0.7)
- 6 facilities: 112.5 RP (multiplier: 0.625)
- 7 facilities: **115.5 RP** (multiplier: 0.55) - **Optimal point**
- 8 facilities: 114.0 RP (multiplier: 0.475) - Total RP starts decreasing
- 15 facilities: 0 RP (multiplier: 0.0) - No RP generated

**Efficiency Comparison:**
- **Early facilities (1-3)**: Very efficient compared to factories. A single Naval/Air/Land facility provides 5x the RP of a civilian factory (15 vs 3 RP), and a Nuclear facility provides 10x (30 vs 3 RP).
- **Mid-range (4-7)**: Still efficient, but diminishing returns are noticeable. At 7 facilities, Naval/Air/Land facilities provide ~8.25 RP each on average, still better than factories.
- **Late facilities (8+)**: Less efficient than factories. Beyond 7-8 facilities, building additional civilian factories becomes more cost-effective for RP generation.

**Strategic Implications:**
- The optimal strategy is to build 6-7 facilities of each type for maximum RP efficiency.
- Building more than 7 facilities of the same type becomes counterproductive as total RP decreases.
- The system encourages diversification across facility types rather than stacking a single type.

### 5.3 War-time RP modifier

If the country is at war and the **War RP** rule (`DR_WAR_RP_RULE`) is not set to OFF:

The war penalty is calculated in `dr_apply_war_rp_logic`:

- `war_rp_base_penalty` is set based on the rule:
  - OFF, −5%, −10%, −15%, −20%, −25%, −30% (default: −10%).
- The penalty is shaped by:
  - **War support** (`war_support`), mapped into `war_penalty_factor_ws` (0.0-0.4, 5 steps).
  - **Stability** (`has_stability`), mapped into `war_penalty_factor_stab` (0.0-0.3, 5 steps).
  - **Ruling party support** (ideology popularity), mapped into `war_penalty_factor_party` (0.0-0.3, ideology-specific thresholds).
  - **War duration** (via `dr_days_in_war`) and phase factors:
    - Offensive war: 0% at day 0, 33% at 30 days, 66% at 60 days, 100% at 90+ days.
    - Defensive war: 0% at day 0, 33% at 60 days, 66% at 120 days, 100% at 180+ days.
  - **Defensive war protection**: `war_defense_factor = 0.5` for pure defensive wars (penalty ramps up more slowly).
- All of these are combined into `war_penalty_factor_total`:
  - `war_penalty_factor_total = (war_penalty_factor_ws + war_penalty_factor_stab + war_penalty_factor_party) × war_phase_factor × war_defense_factor`
- The effective penalty is computed:
  - `war_rp_penalty_effective = war_rp_base_penalty × war_penalty_factor_total`
  - This is then negated and added to `total_rp_modifier`

**Peacetime bonus**:
- If the country is at peace, a small bonus is applied based on stability and ruling party support:
  - Stability >70%: +2% RP
  - Stability >85%: +1% RP (additional)
  - Ruling party support >70%: +2% RP
  - Ruling party support >85%: +1% RP (additional)
  - Cap: +5% total
- This is added to `total_rp_modifier` as a positive value.

War RP modifiers can be disabled per-country via the `dr_disable_rp_modifiers` flag.

### 5.4 Alliance & law modifiers

The following rules influence `total_rp_modifier` as positive or negative terms:

- `DR_ALLIANCE_RP_RULE` – determines the maximum positive RP bonus you can get from being in an alliance (OFF, +5%, +10%, +15%, +20%, +25%, +30%, default: +10%).
  - Alliance bonus calculation:
    - Base: 5% per ally, scaled by relations (0-100 opinion mapped to 0-1.0 relation factor in 10 steps)
    - Opinion factor calculation is **bidirectional** *(since version 1.5)*:
      - Calculates "They like Us" (Ally's opinion of You)
      - Calculates "We like Them" (Your opinion of Ally)
      - Uses the **average** of these two factors
    - Uses helper effects `dr_get_opinion_factor_for_ally` and `dr_get_opinion_factor_from_root` to check opinions
    - Submods can override opinion calculation via `dr_get_opinion_factor_for_ally_submods` hook *(since version 1.4)* - must set `dr_opinion_factor_custom_set` flag when providing custom value
    - Average relation factor across all allies
    - Cap: `min(5% × ally_count, game_rule_cap)`
    - Effective bonus: `cap × average_relation_factor`
    - Recalculated roughly once per week (7-day timer) to avoid daily overhead
- `DR_LAW_RP_RULE` – toggles whether trade/economy/conscription laws modify RP; when turned off, law-based RP modifiers are removed (default: ON).
  - Trade laws:
    - Free Trade: +2% RP
    - Export Focus: +1% RP
    - Closed Economy: -1% RP
  - Economy laws:
    - Civilian Economy: +2% RP
    - Early Mobilization: +1% RP
    - War Economy: -1% RP
    - Total Mobilization: -2% RP
  - Conscription laws:
    - Extensive Conscription: -2% RP
    - Service by Requirement: -4% RP
    - All Adults Serve: -7% RP
    - Scraping the Barrel: -10% RP

Compatibility note:

- `dr_apply_law_rp_logic` assumes the vanilla law idea IDs (`free_trade`, `export_focus`, `closed_economy`, `civilian_economy`, `low_economic_mobilisation`, `war_economy`, `tot_economic_mobilisation`, `extensive_conscription`, `service_by_requirement`, `all_adults_serve`, `scraping_the_barrel`).
- If a large overhaul mod renames or replaces these ideas, create a submod that overrides `00_dr_dynamic_research_modifiers.txt` and adjust only the `dr_apply_law_rp_logic` block to the new IDs.

Together with the war penalty and peacetime bonus, these form a **single additive modifier** `total_rp_modifier`.

**Modifier validation** *(since version 1.4)*: After `dr_apply_rp_modifier_logic`, the system validates and caps `total_rp_modifier`:
- Maximum bonus: capped at 1.0 (100% bonus)
- Maximum penalty: capped at -0.5 (50% penalty)

Finally:

- A temporary variable `temp` is set to `1 + total_rp_modifier`.
- `total_research_power` is multiplied by `temp`.

**Total RP validation** *(since version 1.4)*: After all modifiers are applied and `dr_total_rp_modifier_submods` has run, the system validates that `total_research_power` is non-negative. If negative, it is set to 0.

**Array validation** *(since version 1.4)*: Before slot threshold checks, the system validates that the `research_for_slot` array exists. If missing, it re-initializes the configuration and rebuilds thresholds.

This is the value used for slot thresholds.

RP modifiers can be disabled per-country via the `dr_disable_rp_modifiers` flag.

**War RP logic structure** *(since version 1.4)*: The war RP calculation is split into modular helper effects for better maintainability:
- `dr_calculate_war_penalty_factors` - calculates war support, stability, and ruling party factors
- `dr_calculate_war_phase_factor` - calculates war duration and type factors
- `dr_calculate_peace_bonus` - calculates peacetime bonus based on stability and ruling party support

**Helper effect for government support** *(since version 1.4)*:
- `dr_get_government_support_factor` - calculates government support factor (0.0, 0.1, 0.2, 0.3, or 0.4) based on popularity and config thresholds
  - Expects: `temp_government_popularity` (variable containing popularity value)
  - Sets: `government_support_factor` based on `government_support_threshold_very_low/low/medium/high/very_high` config values
  - Factor mapping:
    - > very_high (0.85): 0.0 (best)
    - high - very_high (0.7-0.85): 0.1
    - medium - high (0.55-0.7): 0.1
    - low - medium (0.4-0.55): 0.2
    - very_low - low (0.25-0.4): 0.3
    - < very_low (< 0.25): 0.4 (worst)
  - Used by both `dr_calculate_war_penalty_factors` and `dr_calculate_peace_bonus` for consistent logic
  - Performance: ~0.001ms per call, no loops

---

## 6. From RP to research slots

Once `total_research_power` is known, the system determines the desired number of slots:

1. Iterate over all entries in `research_for_slot`:
   - For each index `i`, if `total_research_power >= research_for_slot[i]`, set `target_research_slots = i`.
   - The highest `i` that passes this check becomes the desired slot count.

2. Compute UI helper values:
   - `max_research_slots = research_for_slot^num - 1` (RP threshold of the highest defined research slot minus 1).
   - `next_research_slot_at = 0` (default)
   - If `target_research_slots < max_research_slots`, set `next_research_slot_at = research_for_slot^(target_research_slots + 1)`.

3. If `target_research_slots` differs from `current_research_slots`, slots have changed:
   - For the player, a notification event (`dynamic_research_slots.1`) is fired if the cooldown `dr_player_event_cooldown` allows it (cooldown is 14 days).
   - For AI countries, the same event can be fired without cooldown (mainly for debugging or logging).

4. Apply the result to the game (if slot changes are enabled):
   - `set_research_slots = var:target_research_slots` (only if `dr_disable_research_slot_changes` flag is not set)
   - `current_research_slots = amount_research_slots`

This keeps the script's internal state in sync with the engine.

---

## 7. Custom Game Rules (overview)

File: `common/game_rules/00_dr_dynamic_research_rules.txt`

Summary of the key rules:

- `DR_EASY_SLOTS_RULE` – baseline number of Easy Slots for all countries (Default, +1, +2, +3, +4, +5).
- `DR_EASY_COST_RULE` – cost factor for Easy Slots (10%–100%, default: 60%).
- `DR_MINOR_EASY_SLOTS_RULE` – optional extra Easy Slot for non-major countries (OFF/ON, default: OFF).
- `DR_FACTORY_WEIGHTS_RULE` – distribution of RP between civilian, military and naval factories (Balanced/Industry/Military/Naval, default: Balanced).
- `DR_WAR_RP_RULE` – war-time RP penalty (OFF, −5%, −10%, −15%, −20%, −25%, −30%, default: −10%).
- `DR_ALLIANCE_RP_RULE` – maximum positive RP bonus from alliances (OFF, +5%, +10%, +15%, +20%, +25%, +30%, default: +10%).
- `DR_LAW_RP_RULE` – whether trade/economy/conscription laws affect RP (ON/OFF, default: ON).

Many of these rules have `allow_achievements = no` on extreme settings; check the localisation for the exact behaviour.

---

## 8. Debug tools & UI

### Decisions

File: `common/decisions/dynamic_research_slots_decisions.txt`

- `dynamic_research_slots_help`  
  - Visible only for human players.  
  - Triggers the help event `dynamic_research_slots.help`, which explains the system in-game.

- `initialize_dynamic_research_slots` (debug)  
  - Visible only in debug mode (`is_debug = yes`).  
  - Available only if the country is not yet initialized.
  - Calls `initialize_dynamic_research_slots` and `recalculate_dynamic_research_slots` for the current country.

- `recalculate_dynamic_research_slots` (debug)  
  - Visible only in debug mode.  
  - Available only if the country is already initialized.
  - Forces a recalculation of all RP and slot thresholds.

- `dynamic_research_slots_debug` (debug)  
  - Visible in debug mode, cost 0, infinite duration.  
  - Triggers `dynamic_research_slots.4` – used as a debug overlay / information event.

### Events

File: `events/dynamic_research_slot_events.txt`

- `dynamic_research_slots.help` – main help popup, opened via the help decision.
  - Has an option to open `dynamic_research_slots.5` for more details about war/law/alliance modifiers.
- `dynamic_research_slots.1` – notification when slots change.
- `dynamic_research_slots.2` – notification when Easy Slots are added (e.g., when slots above 2 are converted to Easy Slots at game start).
- `dynamic_research_slots.4` – debug information event showing all internal variables and RP breakdown.
- `dynamic_research_slots.5` – detailed explanation of war/law/alliance modifiers.
- `dynamic_research_slots.6` – one-time startup explanation shown to players at game start.

---

## 9. Extending the system

When you extend or rebalance the mod, try to follow these guidelines so the existing debug tools and documentation stay useful.

The recommended entry point for configuration changes is the scripted effect:

- `dr_apply_research_config` in `common/scripted_effects/00_dr_dynamic_research_config.txt`

This effect is called from `initialize_dynamic_research_slots` and is responsible for:

- Setting base RP weights (`research_power_per_civ`, `research_power_per_mil`, `research_power_per_nav`).
- Filling the arrays `base_research_for_slot` and `research_for_slot` with the default thresholds.
- Setting and adjusting `easy_research_slots` and `easy_research_slot_coefficient` (including game-rule overrides).
- Setting AI update frequency (`dr_ai_update_frequency`, default: 14 days).

Internally the effect is split into smaller helpers:
- `dr_reset_research_config_defaults` – sets all base values
- `dr_apply_research_factory_weight_rules` – applies factory weight game rule overrides
- `dr_apply_easy_slot_rule_overrides` – applies Easy Slot game rule overrides
- `dr_apply_easy_slot_cost_rules` – applies Easy Slot cost factor game rule overrides

After the vanilla logic has finished, the empty hook `dr_apply_research_config_submods` is called. Submods can override this scripted effect to tweak or replace any variables without copying the full config block.

For extensive changes, submods can override the entire `00_dr_dynamic_research_config.txt` file instead.

By overriding this single effect in a submod, you can change all core configuration while leaving the main logic untouched.

### 9.1 Adding new RP sources

- Prefer to hook additional RP sources into the existing calculation rather than creating parallel systems.
- Typical options:
  - Add a new **flat RP component** and include it in `total_research_power`.
  - Adjust `civilian_rp_modifier`, `military_rp_modifier` or `naval_rp_modifier` from ideas, spirits or laws.
  - Add another component to `total_rp_modifier` (e.g. a global modifier from difficulty or a special focus).
- Use the dedicated extension hooks:
  - `dr_collect_facility_counts_submods` (after vanilla facility counting) – count custom buildings
  - `dr_apply_facility_rp_submods` (after vanilla facility RP application) – convert counts to RP
  - `dr_apply_factory_modifiers_submods` (before factory RP calculation) – set factory modifiers
  - `dr_total_rp_modifier_submods` (after the global RP modifier was applied) – final RP adjustments

Always make sure that:
- `total_research_power` remains the *single source of truth* for slot thresholds.
- New variables are either clearly named or documented in this file.

### 9.2 Changing thresholds

- To change when slots unlock, edit the values in:
  - `base_research_for_slot` in `dr_apply_research_config` (file `00_dr_dynamic_research_config.txt`).
- Keep in mind:
  - Index 0 is a dummy entry for easier indexing.
  - Indices 1 and 2 correspond to the first two slots and are usually kept at 0 (free).
  - Higher indices represent later slots; make them non-decreasing to avoid strange behaviour.

If you add more slots:
- Extend both `base_research_for_slot` and `research_for_slot` arrays with matching indices.
- Make sure any loops that iterate over these arrays pick up the new maximum index.

You can also use `dr_adjust_research_thresholds_submods` to dynamically adjust thresholds after Easy Slot logic is applied, without overriding the entire threshold array.

### 9.3 Modifying Easy Slots

- To adjust how strong Easy Slots are:
  - Change `easy_research_slot_coefficient` defaults or the `DR_EASY_COST_RULE` options in `dr_apply_research_config`.
- To change how many Easy Slots exist:
  - Adjust the base value of `easy_research_slots` in `dr_apply_research_config`.
  - Or change the game rule `DR_EASY_SLOTS_RULE` / `DR_MINOR_EASY_SLOTS_RULE`.

### 9.4 Integrating into larger mods

When integrating this system into a large overhaul:

- Keep the **scripted effect names** (`initialize_dynamic_research_slots`, `calculate_modifiers_to_rp`, `recalculate_dynamic_research_slots`) intact where possible.
- If you rename files or move content, update:
  - on_actions hooks,
  - decisions,
  - events,
  - and any localisation that references these.
- Avoid duplicating the RP logic in multiple places; extend the existing effects instead.

When creating a **submod** that rebalances the system, typical patterns are:

- Override `dr_apply_research_config` in your own mod:
  - Copy `common/scripted_effects/00_dr_dynamic_research_config.txt` into your mod.
  - Adjust factory weights, thresholds and Easy Slot defaults as desired.
  - Make sure your mod is loaded **after** Dynamic Research Slots Reborn so your version of `dr_apply_research_config` is used.
- Optionally override `dr_apply_rp_modifier_logic` in your own mod (file `00_dr_dynamic_research_modifiers.txt`) to change:
  - War-time RP penalties and curves (see `dr_apply_war_rp_logic`).
  - Peacetime bonuses.
  - Law-based RP effects (see `dr_apply_law_rp_logic`).
  - Alliance-based bonuses and caps (see `dr_apply_alliance_rp_logic`).
- Optionally add your own helper effects that:
  - Modify `total_rp_modifier` based on new global conditions (difficulty, special ideas, etc.).
  - Adjust `civilian_rp_modifier`, `military_rp_modifier` or `naval_rp_modifier` from your own ideas/spirits.

This way, the **core logic and debug tools remain unchanged**, while submods only replace or extend the configuration layer. If you need to completely opt out a country from dynamic slot changes (for example in a large overhaul), you can set the country flag `dr_disable_dynamic_research_slots`; the system will still keep its internal variables in sync but will not change the number of research slots for that country.

### 9.5 Helper Effects

The mod provides several helper effects that can be used by submods or for internal calculations:

**Facility RP Helpers:**

- `dr_apply_facility_rp_if_present` - Checks if a facility count is greater than 0 and applies RP calculation if present. Expects:
  - `facility_count_var` - Variable containing the facility count
  - `rp_per_facility_value` - Variable or literal value for RP per facility
  - `reduction_factor_value` - Variable or literal value for the reduction factor (typically 0.15)
  - Calls `dr_calculate_single_facility_rp` internally if count > 0

- `dr_calculate_single_facility_rp` *(since version 1.4)* - Calculates RP for a facility type with diminishing returns. Expects:
  - `facility_rp_count` - Number of facilities
  - `rp_per_facility_var` - RP per facility
  - `facility_rp_reduction_factor` - Reduction factor for diminishing returns
  - Sets `temp_facility_rp` and adds it to `total_research_power` and `facility_research_power`

**Government Support Helpers:**

- `dr_set_government_popularity` *(since version 1.5)* - Sets `temp_government_popularity` based on current government type (democratic, fascism, communism, neutrality). Used internally by war penalty and peace bonus calculations.

- `dr_get_government_support_factor` *(since version 1.5)* - Calculates government support factor (0.0, 0.1, 0.2, 0.3, or 0.4) based on popularity thresholds. Expects:
  - `temp_government_popularity` - Variable containing popularity value
  - Uses config variables: `government_support_threshold_very_low`, `government_support_threshold_low`, `government_support_threshold_medium`, `government_support_threshold_high`, `government_support_threshold_very_high`
  - Sets `government_support_factor`

- `dr_map_government_support_to_peace_bonus` *(since version 1.5)* - Maps government support factor to peace bonus. Expects:
  - `government_support_factor` (0.0, 0.1, 0.2, 0.3, or 0.4)
  - Modifies `peace_rp_bonus` (adds 0.03 for 0.0, 0.02 for 0.1, 0.0 for 0.2+)

**Other Helpers:**

- `dr_rebuild_research_thresholds` - Rebuilds effective RP thresholds based on base thresholds, Easy Slots, and Easy Slot coefficient
- `dr_get_opinion_factor_for_ally` *(since version 1.4)* - Calculates opinion factor for alliance members
- `dr_check_for_factory_changes` *(since version 1.5)* - Checks if factory counts (Civ/Mil/Nav) have changed since last check
- `dr_recalculate_if_needed` *(since version 1.5)* - Conditional recalculation wrapper that only triggers when factories changed or 7 days passed
- Various war/law/alliance helper effects (see `00_dr_dynamic_research_modifiers.txt`)

These helpers follow the DRY (Don't Repeat Yourself) principle and make the codebase more maintainable.

### 9.6 Code Quality Improvements *(since version 1.5)*

The mod has been refactored to reduce code redundancy:

- **Government Type Detection:** Consolidated into `dr_set_government_popularity` helper effect
- **Peace Bonus Mapping:** Consolidated into `dr_map_government_support_to_peace_bonus` helper effect
- **Smart Detection System:** Factory change detection reduces unnecessary calculations

These improvements maintain backward compatibility while improving maintainability and performance.

---

## 10. Worked example (simplified)

Example country:

- 20 civilian factories, 15 military factories, 5 dockyards
- 1 nuclear facility, no other facilities
- No war/alliance penalties or bonuses (all related rules at default)
- Easy Slots coefficient at 60% (default), 2 Easy Slots

Step 1 - Base factory RP:

- Civ: `20 × 3 = 60 RP`
- Mil: `15 × 2 = 30 RP`
- Nav: `5 × 2 = 10 RP`
- Total from factories: `60 + 30 + 10 = 100 RP`

Step 2 - Facilities:

- Nuclear facility: `1 × 50 = 50 RP`
- Total RP so far: `100 + 50 = 150 RP`

Step 3 - Modifiers:

- No war penalty, alliance bonus or law-based modifiers: `total_rp_modifier = 0`
- Effective RP stays `150`.

Step 4 - Thresholds:

- With 2 Easy Slots at 60%, thresholds (roughly) look like:
  - Slot 1–2: 0 RP (Easy Slots, but base is already 0)
  - Slot 3: `50 × 0.6 = 30 RP` (Easy Slot)
  - Slot 4: `200 RP` (normal)
  - Slot 5: `400 RP` (normal)
  - ...
- 150 RP is:
  - above the threshold for slot 3 (30 RP)
  - below the threshold for slot 4 (200 RP)
- So `target_research_slots` becomes **3**.

Step 5 - Application:

- If the country previously had 2 slots (`current_research_slots = 2`), the script:
  - Sets `set_research_slots = 3`
  - Updates `current_research_slots` to the new engine value
  - Fires the player notification event (respecting the cooldown).

This example ignores war/alliance/law modifiers, but illustrates the flow from factories/facilities -> RP -> thresholds -> final slot count.

