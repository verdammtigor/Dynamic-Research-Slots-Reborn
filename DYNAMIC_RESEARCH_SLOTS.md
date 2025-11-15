# Dynamic Research Slots - Internal Logic

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

- `common/decisions/dynamic_research_slots_decisions.txt`  
  Provides the help decision and several debug decisions to (re)initialize or recalculate the system.

- `events/dynamic_research_slot_events.txt`  
  Implements the help popup and the notification events when the number of research slots changes.

- `common/game_rules/00_dr_dynamic_research_rules.txt`  
  Defines all **Custom Game Rules** that influence Easy Slots, factory weights, war/alliance modifiers and law-based modifiers.

- `common/scripted_localisation/00_dynamic_research_slots_loc.txt`  
  Simple scripted localisation that adds helper notes to some game rules depending on their settings.

---

## 2. Update cycle (on_actions)

File: `common/on_actions/00_dynamic_research_slots_on_actions.txt`

### Game start (`on_startup`)

At game start, every country runs:

- `initialize_dynamic_research_slots = yes`
- `calculate_modifiers_to_rp = yes`

This sets up all internal variables and the RP thresholds, but does **not** immediately change the number of vanilla research slots beyond the initial configuration.

### Daily update (`on_daily`)

Every in-game day:

- `dr_days_in_war` is updated (counts how long the country has been at war).
- New countries that appear later in the game get initialized once via `initialize_dynamic_research_slots`.
- The system branches between **player** and **AI**:

Player country:
- Recalculates modifiers every day: `calculate_modifiers_to_rp`.
- Recalculates target slots every day: `recalculate_dynamic_research_slots`.
- Uses a cooldown variable `dr_player_event_cooldown` to avoid spamming the player with events.

AI countries:
- Use a staggered update: `dr_days_until_update` counts down from a random offset (1–30 days), then triggers:
  - `calculate_modifiers_to_rp`
  - `recalculate_dynamic_research_slots`

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

Total RP:

- `total_research_power` – final RP used to determine how many slots you should have.
- `facility_research_power` – share of RP that comes from experimental facilities (nuclear, naval, air, land).

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

- `total_rp_modifier` – overall additive RP modifier (includes war/alliance/law effects).  
  This is applied on top of the summed factory/facility RP.

Arrays:

- `base_research_for_slot[index]` – baseline RP thresholds for each slot.
- `research_for_slot[index]` – effective thresholds actually used, after Easy Slot adjustments.
- `max_research_slots` – maximum slot index that currently has a defined threshold.
- `next_research_slot_at` – RP threshold for the next slot above `target_research_slots`.

---

## 4. Initialization & thresholds

File: `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`

### `initialize_dynamic_research_slots`

Key steps:

1. Marks the country as initialized:  
   - `set_country_flag = dynamic_research_slots_initialized`

2. Sets base slot counts:
   - `current_research_slots = amount_research_slots`
   - `target_research_slots = 2`
   - `easy_research_slots = 2`

3. Sets base RP weights per factory type (balanced by default):
   - `research_power_per_civ = 3`
   - `research_power_per_mil = 2`
   - `research_power_per_nav = 2`

   These can be overridden by the **Factory Weights** game rule:
   - Industry-focused, Military-focused or Naval-focused distributions.

4. Initializes the baseline RP thresholds for each slot in `base_research_for_slot` and `research_for_slot`:

   - Slots 1–2: `0` (free, i.e. vanilla starting slots).
   - Slot 3: `50`
   - Slot 4: `200`
   - Slot 5: `400`
   - Slot 6: `700`
   - Slot 7: `1000`
   - Slot 8: `1500`
   - Slot 9: `2000`
   - Slot 10: `3000`

5. Initializes Easy Slot data:
   - `easy_research_slots` starts at 2.
   - `easy_research_slot_coefficient` starts at `0.6` (60% of the normal threshold).

6. Applies **Easy Slots (Base)** game rule:
   - `DR_EASY_SLOTS_PLUS_*` options add up to +5 Easy Slots on top of the default.
   - Optional extra Easy Slot for **non-majors** via `DR_MINOR_EASY_SLOTS_RULE`.

7. Applies **Easy Slot RP Cost Factor** rule:
   - `DR_EASY_COST_*` options set `easy_research_slot_coefficient` from `0.1` (very cheap) up to `1.0` (same as normal slots).

After initialization, a helper loop adjusts `research_for_slot[index]`:
- For indices `i` that are `<= easy_research_slots`, the threshold is reduced:
  - `research_for_slot[i] = base_research_for_slot[i] * easy_research_slot_coefficient`.
- For higher slots, the baseline cost is unchanged.

This way the first few slots are easier to acquire but later slots still require full RP.

---

## 5. Research Power calculation

The RP calculation happens mainly in:

- `calculate_modifiers_to_rp`
- the subsequent block in `recalculate_dynamic_research_slots` that uses the factory counts.
- the configuration effect `dr_apply_rp_modifier_logic` (file `common/scripted_effects/00_dr_dynamic_research_modifiers.txt`), which encapsulates the war/peace/law/alliance logic and can be overridden by submods.

### 5.1 Factory contributions

For each factory type:

1. Start from the per-factory weight:
   - `research_power_per_civ`, `research_power_per_mil`, `research_power_per_nav`.

2. Multiply by the number of factories:
   - `num_of_civilian_factories`, `num_of_military_factories`, `num_of_naval_factories`.

3. Apply the corresponding RP modifier:
   - `civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`.

4. Sum up:
   - `total_research_power = civilian_research_power + military_research_power + naval_research_power`.

### 5.2 Experimental facilities

If the country has any **Experimental Facilities**, additional flat RP is added:

- Nuclear facility: +50 RP.
- Naval facility: +35 RP.
- Air facility: +35 RP.
- Land facility: +35 RP.

This RP goes both into:
- `total_research_power`
- `facility_research_power` (for display/debug purposes).

### 5.3 War-time RP modifier

If the country is at war and the **War RP** rule (`DR_WAR_RP_RULE`) is not set to OFF:

- `war_rp_base_penalty` is set based on the rule:
  - e.g. −5%, −10%, … up to −30%.
- The penalty is shaped by:
  - **War support** (`war_support`), mapped into `war_penalty_factor_ws`.
  - **War duration** (via `dr_days_in_war`) and phase factors.
  - Other factors (stability, ruling party), captured in `war_penalty_factor_*`.
- All of these are combined into `war_penalty_factor_total`, which is then used to compute:
  - `war_rp_penalty_effective`.

This penalty is folded into `total_rp_modifier` as a negative contribution.

### 5.4 Alliance & law modifiers

The following rules influence `total_rp_modifier` as positive or negative terms:

- `DR_ALLIANCE_RP_RULE` – determines the maximum positive RP bonus you can get from being in an alliance (up to +30% at the high end).
- `DR_LAW_RP_RULE` – toggles whether trade/economy/conscription laws modify RP; when turned off, law-based RP modifiers are removed.

Together with the war penalty, these form a **single additive modifier** `total_rp_modifier`. Finally:

- A temporary variable `temp` is set to `1 + total_rp_modifier`.
- `total_research_power` is multiplied by `temp`.

This is the value used for slot thresholds.

---

## 6. From RP to research slots

Once `total_research_power` is known, the system determines the desired number of slots:

1. Iterate over all entries in `research_for_slot`:
   - For each index `i`, if `total_research_power >= research_for_slot[i]`, set `target_research_slots = i`.
   - The highest `i` that passes this check becomes the desired slot count.

2. If `target_research_slots` differs from `current_research_slots`, slots have changed:
   - For the player, a notification event (`dynamic_research_slots.1`) is fired if the cooldown `dr_player_event_cooldown` allows it.
   - For AI countries, the same event can be fired without cooldown (mainly for debugging or logging).

3. Compute UI helper values:
   - `max_research_slots` – last valid index in `research_for_slot` minus 1 (to ignore the dummy index 0).
   - `next_research_slot_at` – RP required for the next slot above `target_research_slots` (if any).

4. Apply the result to the game:
   - `set_research_slots = var:target_research_slots`
   - `current_research_slots = amount_research_slots`

This keeps the script’s internal state in sync with the engine.

---

## 7. Custom Game Rules (overview)

File: `common/game_rules/00_dr_dynamic_research_rules.txt`

Summary of the key rules:

- `DR_EASY_SLOTS_RULE` – baseline number of Easy Slots for all countries.
- `DR_EASY_COST_RULE` – cost factor for Easy Slots (10%–100%).
- `DR_MINOR_EASY_SLOTS_RULE` – optional extra Easy Slot for non-major countries.
- `DR_FACTORY_WEIGHTS_RULE` – distribution of RP between civilian, military and naval factories.
- `DR_WAR_RP_RULE` – war-time RP penalty (OFF, −5%, −10%, …, −30%).
- `DR_ALLIANCE_RP_RULE` – maximum positive RP bonus from alliances (OFF, +5%, …, +30%).
- `DR_LAW_RP_RULE` – whether trade/economy/conscription laws affect RP.

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
  - Calls `initialize_dynamic_research_slots` and `recalculate_dynamic_research_slots` for the current country.

- `recalculate_dynamic_research_slots` (debug)  
  - Visible only in debug mode.  
  - Forces a recalculation of all RP and slot thresholds.

- `dynamic_research_slots_debug` (debug)  
  - Visible in debug mode, cost 0, infinite duration.  
  - Triggers `dynamic_research_slots.4` – used as a debug overlay / information event.

### Events

File: `events/dynamic_research_slot_events.txt`

- `dynamic_research_slots.help` – main help popup, opened via the help decision.
- `dynamic_research_slots.1` – notification when slots change.
- `dynamic_research_slots.2` / `.3` / `.4` / `.5` – additional detail or debug events (see localisation for text).

---

## 9. Extending the system

When you extend or rebalance the mod, try to follow these guidelines so the existing debug tools and documentation stay useful.

The recommended entry point for configuration changes is the scripted effect:

- `dr_apply_research_config` in `common/scripted_effects/00_dr_dynamic_research_config.txt`

This effect is called from `initialize_dynamic_research_slots` and is responsible for:

- Setting base RP weights (`research_power_per_civ`, `research_power_per_mil`, `research_power_per_nav`).
- Filling the arrays `base_research_for_slot` and `research_for_slot` with the default thresholds.
- Setting and adjusting `easy_research_slots` and `easy_research_slot_coefficient` (including game-rule overrides).

By overriding this single effect in a submod, you can change all core configuration while leaving the main logic untouched.

### 9.1 Adding new RP sources

- Prefer to hook additional RP sources into the existing calculation rather than creating parallel systems.
- Typical options:
  - Add a new **flat RP component** and include it in `total_research_power`.
  - Adjust `civilian_rp_modifier`, `military_rp_modifier` or `naval_rp_modifier` from ideas, spirits or laws.
  - Add another component to `total_rp_modifier` (e.g. a global modifier from difficulty or a special focus).

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
  - War-time RP penalties and curves.
  - Peacetime bonuses.
  - Law-based RP effects.
  - Alliance-based bonuses and caps.
- Optionally add your own helper effects that:
  - Modify `total_rp_modifier` based on new global conditions (difficulty, special ideas, etc.).
  - Adjust `civilian_rp_modifier`, `military_rp_modifier` or `naval_rp_modifier` from your own ideas/spirits.

This way, the **core logic and debug tools remain unchanged**, while submods only replace or extend the configuration layer.


---

## 10. Worked example (simplified)

Example country:

- 20 civilian factories, 15 military factories, 5 dockyards
- 1 nuclear facility, no other facilities
- No war/alliance penalties or bonuses (all related rules at default)
- Easy Slots coefficient at 60% (default), 2 Easy Slots

Step 1 - Base factory RP:

- Civ: `20 * 3 = 60 RP`
- Mil: `15 * 2 = 30 RP`
- Nav: `5 * 2 = 10 RP`
- Total from factories: `60 + 30 + 10 = 100 RP`

Step 2 - Facilities:

- Nuclear facility: `+50 RP`
- Total RP so far: `100 + 50 = 150 RP`

Step 3 - Modifiers:

- No war penalty, alliance bonus or law-based modifiers: `total_rp_modifier = 0`
- Effective RP stays `150`.

Step 4 - Thresholds:

- With 2 Easy Slots at 60%, thresholds (roughly) look like:
  - Slot 1–2: 0 RP
  - Slot 3: `50 * 0.6 = 30 RP`
  - Slot 4: `200 RP`
  - Slot 5: `400 RP`
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
