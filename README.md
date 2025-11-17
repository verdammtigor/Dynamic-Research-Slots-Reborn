# Dynamic Research Slots Reborn

This repository contains the updated and extended version of the mod **“Dynamic Research Slots Reborn”** for *Hearts of Iron IV*.

## What the mod does (for players)

- The number of your research slots is no longer static but depends on your **industry**.
- Civilian factories, military factories, dockyards and **experimental facilities** generate **Research Power (RP)**.
- When your RP crosses certain thresholds, you gain additional research slots.
- Some of the slots are **Easy Research Slots**: they are easier to maintain (require less RP).

In game you will find:

- A **decision** in the category `dynamic_research_slots_decisions` that shows an overview of your current RP and the next threshold.
- A **HELP decision** that opens a popup with a detailed explanation of all mechanics.
- A dedicated **Custom Game Rules** tab *Dynamic Research*, where you can adjust global settings for Easy Slots and RP factors.

## Installation & usage

- The mod is loaded as usual via the HOI4 launcher (Steam Workshop / local mod file).
- Make sure the mod is loaded **after** other large overhaul mods that might have their own research-slot mechanics.
- The mod does **not** overwrite vanilla tech or focus trees but works via **scripted_effects**, **on_actions**, **decisions** and **game_rules**.

## Supported version / compatibility

- Target version: the current HOI4 version at the time of the last update of this repository.
- The mechanics are intentionally generic and should be compatible with most content mods, as long as they do not also heavily modify research slots and the underlying engine mechanics.

## In-game configuration (Custom Game Rules)

In the Custom Game Rules menu you will find, among others:

- **Easy Research Slots (Base)** – additional Easy Slots for all countries.
- **Easy Slot RP Cost Factor** – how much RP an Easy Slot costs compared to a normal slot.
- **Factory Weights** – how strongly civilian/military factories and dockyards are weighted.
- **War RP / Alliance RP / Law RP** – global modifiers for research during war, in alliances and from laws.

Non-standard options can in some cases **disable achievements** (see the game-rule descriptions in-game).

## For modders / technical documentation

If you want to extend, rebalance or integrate the mod into a larger project, please read the technical overview:

- `DYNAMIC_RESEARCH_SLOTS.md` – detailed description of the internal logic (Research Power, thresholds, Easy Slots, game rules, debug tools, etc.).
- `SUBMODDING_DYNAMIC_RESEARCH_SLOTS.md` – step‑by‑step guide for creating your own balancing or integration submods, including the new hook-based workflow.

Key script files:

- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt` – central RP calculation, slot logic and modifiers.
- `common/scripted_effects/00_dr_dynamic_research_config.txt` – main configuration and balancing entry point.
- `common/scripted_effects/00_dr_dynamic_research_modifiers.txt` – war/peace/law/alliance modifier logic.
- `common/on_actions/00_dynamic_research_slots_on_actions.txt` – hooks into the game flow (startup / daily updates).
- `common/decisions/*.txt` and `events/dynamic_research_slot_events.txt` – UI, help popup, debug overlay.
- `common/game_rules/00_dr_dynamic_research_rules.txt` – configuration via Custom Game Rules.

## Further development

- Whenever possible, changes should be made within the existing effects and variables so that the debug tools and documentation remain usable.
- Use the dedicated **submod hooks** instead of copying core scripts:
  - `dr_apply_research_config_submods` (configuration tweaks)
  - `dr_collect_facility_counts_submods`, `dr_apply_facility_rp_submods`, `dr_total_rp_modifier_submods` (runtime RP extensions)
- For new RP sources (ideas, national spirits, technologies, etc.), please extend both the actual calculation and the display variables (for example `facility_research_power` or new variables).
- When integrating large overhaul mods that change laws or research mechanics, prefer adapting `00_dr_dynamic_research_config.txt` and `00_dr_dynamic_research_modifiers.txt` in a separate submod.

More technical background can be found in `DYNAMIC_RESEARCH_SLOTS.md` and `SUBMODDING_DYNAMIC_RESEARCH_SLOTS.md`.

