# Dynamic Research Slots Reborn

This repository contains the updated and extended version of the mod **“Dynamic Research Slots Reborn”** for *Hearts of Iron IV*.

## What the mod does (for players)

- The number of your research slots is no longer static but depends on your **industry**.
- Civilian factories, military factories, dockyards, **experimental facilities** and **nuclear reactors** generate **Research Power (RP)**.
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

- `docs/technical_reference.md` – detailed description of the internal logic (Research Power, thresholds, Easy Slots, game rules, debug tools, etc.).
- `docs/submodding_guide.md` – step‑by‑step guide for creating your own balancing or integration submods, including the hook-based workflow.
- `docs/submod_quick_start_template.md` – quick-start template with all 8 hooks (copy code blocks, uncomment, adjust). For complete working examples, see `example_submods/`.
- `docs/performance.md` – comprehensive performance analysis of the mod, including base mod performance, compatibility features, and best practices for submod developers.
- `docs/compatibility.md` – detailed compatibility documentation and available features.

**Quick start for submods**: Copy the code blocks from `docs/submod_quick_start_template.md` to your submod at `common/scripted_effects/ZZ_your_submod_name.txt`, uncomment the hooks you need, and adjust values. For complete working examples, see `example_submods/`.

**Example submods**: Complete, working example submods are available on **GitHub** in the `example_submods/` directory. These demonstrate common use cases (balance tweaks, custom facilities, factory modifiers) and serve as reference implementations for developers. The examples are not published on Steam Workshop - they are developer resources available only on GitHub.

**Note on the hook system**: You may notice that the mod provides many empty hooks (8 total) that do nothing by default. This is **intentionally untypical** for HOI4 mods, which often prefer to overwrite entire files. However, this mod is built with a very modular architecture and places high value on compatibility. The empty hooks allow submods to extend functionality without conflicts, enabling multiple submods to work together seamlessly. This approach prioritizes compatibility and modularity over the more common "overwrite entire files" pattern in the HOI4 modding community.

Key script files:

- `common/scripted_effects/00_dr_mod_metadata.txt` – mod metadata (version, compatibility signals).
- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt` – central RP calculation, slot logic and modifiers.
- `common/scripted_effects/00_dr_dynamic_research_config.txt` – main configuration and balancing entry point. Can be overridden by submods for extensive changes.
- `common/scripted_effects/00_dr_dynamic_research_modifiers.txt` – war/peace/law/alliance modifier logic.
- `common/on_actions/00_dynamic_research_slots_on_actions.txt` – hooks into the game flow (startup / daily updates).
- `common/decisions/*.txt` and `events/dynamic_research_slot_events.txt` – UI, help popup, debug overlay.
- `common/game_rules/00_dr_dynamic_research_rules.txt` – configuration via Custom Game Rules.

## Further development

- Whenever possible, changes should be made within the existing effects and variables so that the debug tools and documentation remain usable.
- Use the dedicated **submod hooks** instead of copying core scripts:
  - **Initialization hooks**: `dr_check_compatibility_submods` (before config), `dr_initialize_submods` (after config), `dr_apply_research_config_submods` (config tweaks)
  - **Runtime hooks**: `dr_apply_factory_modifiers_submods`, `dr_adjust_research_thresholds_submods`, `dr_collect_facility_counts_submods`, `dr_apply_facility_rp_submods`, `dr_total_rp_modifier_submods`
- For small config tweaks, use `dr_apply_research_config_submods`. For extensive changes, override the entire `00_dr_dynamic_research_config.txt` file.
- For new RP sources (ideas, national spirits, technologies, etc.), please extend both the actual calculation and the display variables (for example `facility_research_power` or new variables).
- When integrating large overhaul mods that change laws or research mechanics, prefer adapting `00_dr_dynamic_research_config.txt` and `00_dr_dynamic_research_modifiers.txt` in a separate submod.

More technical background can be found in `docs/technical_reference.md` and `docs/submodding_guide.md`.

