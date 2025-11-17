# Submodding: Dynamic Research Slots Reborn

> **Document Note**: This documentation was created with AI assistance to provide more comprehensive and detailed coverage, but has been manually reviewed and controlled by a human. The examples and best practices have been verified, but if you notice any errors or have suggestions for improvement, please report them.

This document is aimed at modders who want to create **their own submods** for  
_Dynamic Research Slots Reborn_ – for example balance tweaks or integration into larger overhauls, without rewriting the core logic.

The focus is on:
- **simple, clear getting‑started steps**,
- **concrete examples** for common changes,
- and **best practices** so your submod plays nicely with the main mod.

For a deeper technical description of the internal logic, see `technical_reference.md`.

---

## 1. Prerequisites

You should already:

- have basic HOI4 modding experience (folder structure, `descriptor.mod`, paths),
- know how to create a mod in the Paradox launcher and set the **load order**,
- be roughly familiar with `common/scripted_effects` and `common/game_rules`.

If these concepts are new to you, read a general HOI4 modding tutorial first.

---

## 2. Architecture (short overview)

The most important files of the system are:

- `common/on_actions/00_dynamic_research_slots_on_actions.txt`  
  Hooks the logic into the game flow (`on_startup`, `on_daily`).

- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`  
  Core logic: compute RP, determine target slots, fire events.

- `common/scripted_effects/00_dr_dynamic_research_config.txt`  
  **Configuration**: RP per factory type, slot thresholds, Easy Slots, cost factor, RP per experimental facility.  
  ⇒ This is the main **entry point for submods**.

- `common/scripted_effects/00_dr_dynamic_research_modifiers.txt`  
  **Modifier logic**: war, peace, laws, alliances (`dr_apply_rp_modifier_logic` + sub‑effects).

As a submodder you usually only touch `00_dr_dynamic_research_config.txt` – and optionally parts of the modifier logic.

---

### 2.1. Extension hooks

To avoid editing the core logic, the mod exposes empty scripted effects that you can override in your submod. All hooks are empty in the base mod, so overriding them in your submod is conflict-free.

**Quick start**: For a quick-start template with all 8 hooks, see `submod_quick_start_template.md` (copy code blocks, uncomment, adjust). For complete, working example submods, see the `example_submods/` directory on GitHub (developer resources, not published on Steam Workshop).

**Hook execution flow**:

```
┌─────────────────────────────────────────────────────────────┐
│ INITIALIZATION FLOW (runs once per country at game start)   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  initialize_dynamic_research_slots                          │
│    │                                                          │
│    ├─► dr_check_compatibility_submods  ← Hook 0 (rare)     │
│    │                                                          │
│    ├─► dr_apply_research_config                              │
│    │     │                                                    │
│    │     └─► dr_apply_research_config_submods  ← Hook 2     │
│    │                                                          │
│    └─► dr_initialize_submods  ← Hook 1                       │
│                                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ RUNTIME FLOW (runs daily for players, staggered for AI)     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  recalculate_dynamic_research_slots                         │
│    │                                                          │
│    ├─► Easy Slot logic & threshold rebuilding               │
│    │                                                          │
│    ├─► dr_adjust_research_thresholds_submods  ← Hook 4      │
│    │                                                          │
│    ├─► dr_apply_factory_modifiers_submods  ← Hook 3         │
│    │                                                          │
│    ├─► Factory RP calculation (civ/mil/nav)                │
│    │                                                          │
│    ├─► Count vanilla facilities (nuclear/naval/air/land)    │
│    │                                                          │
│    ├─► dr_collect_facility_counts_submods  ← Hook 5         │
│    │                                                          │
│    ├─► Apply vanilla facility RP                              │
│    │                                                          │
│    ├─► dr_apply_facility_rp_submods  ← Hook 6               │
│    │                                                          │
│    ├─► Apply global modifiers (war/alliance/law)             │
│    │                                                          │
│    ├─► dr_total_rp_modifier_submods  ← Hook 7               │
│    │                                                          │
│    └─► Calculate target slots & apply changes                │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Hook overview**:

| Hook | Called from | When it runs | What to do there |
|------|-------------|--------------|------------------|
| `dr_check_compatibility_submods` | `initialize_dynamic_research_slots` | Very start of initialization (before config) | Perform compatibility checks, signal mod presence, or set flags before config is applied. |
| `dr_initialize_submods` | `initialize_dynamic_research_slots` | Once per init (after config is applied) | Initialize custom variables, set flags, or perform setup tasks. |
| `dr_apply_research_config_submods` | `dr_apply_research_config` | Once per init (and whenever you manually re-run the config) | Final adjustments to RP weights, thresholds, Easy Slots or facility RP. For extensive changes, override the entire config file instead. |
| `dr_apply_factory_modifiers_submods` | `recalculate_dynamic_research_slots` | Before factory RP calculation | Set factory modifiers (`civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`) based on ideas, national spirits, focuses, etc. |
| `dr_adjust_research_thresholds_submods` | `recalculate_dynamic_research_slots` | After Easy Slot logic is applied | Adjust research slot thresholds dynamically (e.g., difficulty-based scaling, country-specific tweaks). |
| `dr_collect_facility_counts_submods` | `recalculate_dynamic_research_slots` | Every recalculation tick (daily for players, staggered for AI) | Count custom buildings, scripted states or other sources that should translate into facility-style RP. |
| `dr_apply_facility_rp_submods` | `recalculate_dynamic_research_slots` | Immediately after vanilla facilities converted to RP | Turn your custom counters into flat RP, or add extra RP buckets. |
| `dr_total_rp_modifier_submods` | `recalculate_dynamic_research_slots` | After the vanilla `total_rp_modifier` multiplier is applied | Final tweaks to `total_research_power`, e.g., faction-wide buffs, ideology bonuses, or penalties. |

**Detailed hook descriptions**:

- **`dr_check_compatibility_submods`**  
  Executes at the very start of initialization, before any configuration is applied. Use this to perform compatibility checks, signal to the main mod that your mod is active, or set flags before initialization. Other mods can check the `dynamic_research_slots_active` global flag and `dr_mod_version` variable to verify the system is active. Rare use case, but useful when you need to run code before config.

- **`dr_initialize_submods`**  
  Executes once during initialization, after config is applied. Perfect for setting up custom variables, flags, or performing one-time setup tasks. Example: Initialize a custom RP tracker variable.

- **`dr_apply_research_config_submods`**  
  Runs after all vanilla defaults and game rules have been applied. Perfect for small tweaks like "add +1 RP per civ factory" without copying the entire config file. For extensive changes, override the entire config file instead (see section 3.1).

- **`dr_apply_factory_modifiers_submods`**  
  Executes before the factory RP calculation. Use this to set multiplicative factory modifiers (`civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`) based on ideas, national spirits, focuses, or other conditions. These modifiers are applied multiplicatively to the base factory RP.

- **`dr_adjust_research_thresholds_submods`**  
  Executes after the Easy Slot logic has been applied but before threshold checks. Use this to dynamically adjust research slot thresholds (e.g., difficulty scaling, country-specific adjustments) without overriding the entire threshold array.

- **`dr_collect_facility_counts_submods`**  
  Executes right after the game counted nuclear/naval/air/land facilities per state. Use this to track custom buildings or scripted states (e.g., add a new `quantum_lab_count` variable).

- **`dr_apply_facility_rp_submods`**  
  Called after vanilla RP has been added for the four facilities. Convert the counters from the previous hook (or anything else) into RP and add it to `total_research_power`/`facility_research_power`. Note: This hook is skipped if `dr_disable_facility_rp` flag is set.

- **`dr_total_rp_modifier_submods`**  
  Fires after `total_rp_modifier` was applied globally. Use it for extra multiplicative/flat tweaks (e.g., ideology-based bonuses) without touching `dr_apply_rp_modifier_logic`.

**Multiple submods**: If multiple submods override the same hook, standard HOI4 load-order rules apply—the later one wins. Coordinate via dependencies when distributing public submods, or design your hooks to be additive rather than overwriting.

**Quick snippets**:

```txt
# Compatibility check (before config)
dr_check_compatibility_submods = {
  # Example: Disable system if incompatible mod is active
  # if = { limit = { has_global_flag = my_incompatible_mod }
  #   set_country_flag = dr_disable_dynamic_research_slots
  # }
}
```

```txt
# Initialize custom variables (after config)
dr_initialize_submods = {
  set_variable = { my_submod_custom_rp = 0 }
}
```

```txt
# Adjust configuration values
dr_apply_research_config_submods = {
  add_to_variable = { rp_per_nuclear_facility = 10 }
}
```

```txt
# Set factory modifiers based on ideas
dr_apply_factory_modifiers_submods = {
  if = {
    limit = { has_idea = my_custom_research_idea }
    add_to_variable = { civilian_rp_modifier = 0.10 }  # +10% Civ Factory RP
  }
}
```

```txt
# Adjust thresholds dynamically
dr_adjust_research_thresholds_submods = {
  # Hard mode: +50% to all thresholds
  for_each_loop = {
    array = research_for_slot
    multiply_variable = { research_for_slot^i = 1.5 }
  }
}
```

```txt
# Count custom buildings
dr_collect_facility_counts_submods = {
  set_variable = { quantum_lab_count = 0 }
  every_owned_state = {
    if = { limit = { quantum_lab > 0 } ROOT = { add_to_variable = { quantum_lab_count = 1 } } }
  }
}
```

```txt
# Convert custom buildings to RP
dr_apply_facility_rp_submods = {
  if = {
    limit = { check_variable = { var = quantum_lab_count value = 0 compare = greater_than } }
    set_variable = { temp_facility_rp = quantum_lab_count }
    multiply_variable = { temp_facility_rp = 40 }
    add_to_variable = { total_research_power = temp_facility_rp }
    add_to_variable = { facility_research_power = temp_facility_rp }
  }
}
```

```txt
# Final RP adjustments
dr_total_rp_modifier_submods = {
  if = { limit = { has_idea = idea_quantum_focus } 
    add_to_variable = { total_rp_modifier = 0.05 }  # +5% total RP
  }
}
```

**Template file**: For a quick-start template with all hooks, see `submod_quick_start_template.md`. Copy the code blocks to your submod as a starting point. For complete working examples, see `example_submods/`.

See "Example 4 – Adding RP from custom buildings (hooks)" below for a full walkthrough that ties multiple hooks together.

---

## 3. Core idea: how to override the mod cleanly

### 3.1. Own submod – minimal setup (recommended: use template)

**Option A: Use the template (easiest)**

1. Copy the code blocks from `submod_quick_start_template.md` to your submod:
   - `common/scripted_effects/ZZ_your_submod_name.txt`
2. Uncomment/modify the hooks you need.
3. In the launcher, set a **dependency** or **load order** so your submod loads **after** "Dynamic Research Slots Reborn".
4. Test your submod with the main mod.

**Option B: Override the config file (for extensive changes)**

If you need to make extensive changes to configuration values, override the entire config file:

1. Create a new mod in the launcher ("Local mod") or manually with its own `descriptor.mod`.  
2. In the launcher, set a **dependency** or at least a **load order** so that your submod loads **after** "Dynamic Research Slots Reborn".  
3. In your submod, create the folder structure:
   - `common/scripted_effects/`
4. Copy `00_dr_dynamic_research_config.txt` from the main mod to your submod:
   - `common/scripted_effects/00_dr_dynamic_research_config.txt`
5. Adjust the values you want to change (RP weights, thresholds, facility RP, etc.)
6. Keep the hook calls intact (e.g., `dr_apply_research_config_submods = yes`) if you want submods of your submod to work

**Important**: In HOI4 the **path** matters, not the filename. If your submod contains a file under the same path, it overrides the version from the main mod.

**Option C: Use hooks only (for small tweaks)**

For small adjustments, you don't need to copy the entire file. Just override the hook in a new file:

1. Create `common/scripted_effects/ZZ_your_submod_name.txt` in your submod
2. Override only the hooks you need:
   ```txt
   dr_apply_research_config_submods = {
     # Small tweaks only - e.g., add +1 RP per civ factory
     add_to_variable = { research_power_per_civ = 1 }
   }
   ```
3. For hooks, you can use any filename as long as it's in `common/scripted_effects/` and loaded after the main mod

### 3.2. What you should avoid touching

- `common/on_actions/00_dynamic_research_slots_on_actions.txt`  
  ⇒ avoid overriding this to reduce conflicts with other mods.

- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`  
  ⇒ this contains the core mechanics; changes here can easily break everything. Prefer using the hooks and config/modifier effects instead.

### 3.3. Opt-out mechanisms

If you need to disable parts of the system for specific countries or conditions, use these flags:

- **`dr_disable_dynamic_research_slots`** (country flag)  
  Completely disables the system for this country. Variables are kept in sync, but no slot changes occur.

- **`dr_disable_research_slot_changes`** (country flag)  
  Disables slot changes but still calculates RP (useful for display/debug purposes). The system will calculate and track RP but won't change the number of research slots.

- **`dr_disable_rp_modifiers`** (country flag)  
  Disables RP modifier logic (war/peace/alliance/law modifiers). Factory and facility RP are still calculated normally, but no global modifiers are applied.

- **`dr_disable_facility_rp`** (country flag)  
  Disables facility RP contributions (nuclear/naval/air/land facilities). Factory RP is still calculated normally.

**Granular control example**:
```txt
dr_initialize_submods = {
  # Disable completely for specific countries
  if = {
    limit = { OR = { tag = VANILLA_TAG_1 tag = VANILLA_TAG_2 } }
    set_country_flag = dr_disable_dynamic_research_slots
  }
  
  # Disable only facility RP for minors (they use custom research buildings)
  if = {
    limit = { is_major = no }
    set_country_flag = dr_disable_facility_rp
  }
  
  # Disable modifier logic for countries with custom modifier systems
  if = {
    limit = { has_country_flag = my_custom_modifier_system }
    set_country_flag = dr_disable_rp_modifiers
  }
}
```

---

## 4. Example 1 – Adjusting slot thresholds (RP thresholds)

### Goal

You want for example:

- slot 3 to unlock at 100 RP (instead of 50),
- slot 4 at 250 RP,
- slot 5 at 500 RP,
- etc.

### Steps

1. In your submod, copy the file  
   `common/scripted_effects/00_dr_dynamic_research_config.txt`  
   from the main mod.
2. Only edit the block **Base thresholds for research slots**.

Important:

- Index `0` is a dummy entry.  
- Indices 1–10 represent slots 1–10.  
- The helper effect `dr_rebuild_research_thresholds` ensures Easy Slots are automatically adjusted with the cost factor.

### Example snippet

```txt
# Base thresholds for research slots
# Index 0 is a dummy entry for easier indexing.
add_to_array = { base_research_for_slot = 0 }   # 0 (dummy)
add_to_array = { base_research_for_slot = 0 }   # 1
add_to_array = { base_research_for_slot = 0 }   # 2
add_to_array = { base_research_for_slot = 100 } # 3 (instead of 50)
add_to_array = { base_research_for_slot = 250 } # 4 (instead of 200)
add_to_array = { base_research_for_slot = 500 } # 5
add_to_array = { base_research_for_slot = 900 } # 6
add_to_array = { base_research_for_slot = 1400 }# 7
add_to_array = { base_research_for_slot = 2000 }# 8
add_to_array = { base_research_for_slot = 2600 }# 9
add_to_array = { base_research_for_slot = 3400 }# 10
```

You do **not** need to change `dr_rebuild_research_thresholds` – it automatically uses your new base values and Easy Slot settings.

If you want more than 10 slots, you can simply append additional entries; just keep in mind that UI texts (e.g. `dynamic_research_slots.help.desc`) might need extending.

---

## 5. Example 2 – Changing factory weights (RP per factory)

### Goal

You want for example:

- civilian factories to contribute more RP,
- military factories less,
- and maybe custom game rule variants.

### Steps

In `dr_apply_research_config` at the top, adjust the base values:

```txt
dr_reset_research_config_defaults = {
  # Base RP weights per factory type
  set_variable = { research_power_per_civ = 4 } # instead of 3
  set_variable = { research_power_per_mil = 1 } # instead of 2
  set_variable = { research_power_per_nav = 2 } # unchanged

  # ... rest unchanged
}
```

Alternatively, keep the vanilla defaults and only add custom tweaks in your own `dr_apply_research_config_submods`:

```txt
dr_apply_research_config_submods = {
  # Simple +10 RP for each experimental facility
  add_to_variable = { rp_per_nuclear_facility = 10 }
  add_to_variable = { rp_per_naval_facility = 10 }
  add_to_variable = { rp_per_air_facility = 10 }
  add_to_variable = { rp_per_land_facility = 10 }
}
```

You can also define new game rule options (in your submod's `common/game_rules`) and react to them here with `has_game_rule` – the pattern is already used by the original (`DR_FACTORY_WEIGHTS_RULE`).

---

## 6. Example 3 – Rebalancing experimental facilities

### Goal

You want for example:

- nuclear facilities to grant only +20 RP each,
- air/naval/land facilities to grant only +10 RP each.

### Steps

In `dr_apply_research_config` the RP per facility variables are defined:

```txt
  # RP per experimental facility (per building)
  set_variable = { rp_per_nuclear_facility = 20 }
  set_variable = { rp_per_naval_facility = 10 }
  set_variable = { rp_per_air_facility = 10 }
  set_variable = { rp_per_land_facility = 10 }
```

The core logic (`recalculate_dynamic_research_slots`) multiplies these values with the number of respective facilities. You don't need to change anything there.

---

## 7. Example 4 – Adding RP from custom buildings (hooks)

### Goal

You added a new state building `quantum_lab` and want each lab to grant +40 flat RP.

### Steps

1. Track the building count via `dr_collect_facility_counts_submods`.
2. Convert it into RP via `dr_apply_facility_rp_submods`.

```txt
# Count custom buildings
dr_collect_facility_counts_submods = {
  set_variable = { quantum_lab_count = 0 }

  every_owned_state = {
    if = {
      limit = { quantum_lab > 0 }
      ROOT = { add_to_variable = { quantum_lab_count = 1 } }
    }
  }
}

# Add research power
dr_apply_facility_rp_submods = {
  if = {
    limit = {
      check_variable = {
        var = quantum_lab_count
        value = 0
        compare = greater_than
      }
    }
    set_variable = { temp_facility_rp = quantum_lab_count }
    multiply_variable = { temp_facility_rp = 40 }
    add_to_variable = { facility_research_power = temp_facility_rp }
    add_to_variable = { total_research_power = temp_facility_rp }
  }
}
```

Need a global modifier instead? Hook into `dr_total_rp_modifier_submods`:

```txt
dr_total_rp_modifier_submods = {
  # +5% RP if the country has your custom idea
  if = {
    limit = { has_idea = idea_quantum_focus }
    add_to_variable = { total_rp_modifier = 0.05 }
    add_to_variable = { temp = 0.05 }
    add_to_variable = { total_research_power = temp }
  }
}
```

---

## 8. Example 5 – Adjusting war / law / alliance effects

The RP modifier logic is intentionally split into three effects:

- `dr_apply_war_rp_logic`  
- `dr_apply_law_rp_logic`  
- `dr_apply_alliance_rp_logic`

and is invoked via `dr_apply_rp_modifier_logic`.

### 8.1. Changing war logic only

1. In your submod, copy  
   `common/scripted_effects/00_dr_dynamic_research_modifiers.txt`  
   or – preferably – define your own `dr_apply_war_rp_logic` in a new file in the same folder (HOI4 merges all script files).
2. Define **your own version** of `dr_apply_war_rp_logic` with the behaviour you want.

Important:

- In the end, you must set `war_rp_penalty_effective` and add it as a **negative** contribution to `total_rp_modifier`, so the rest of the logic still works.

Minimal example (highly simplified, just to illustrate):

```txt
dr_apply_war_rp_logic = {
  set_variable = { war_rp_base_penalty = 0 }
  set_variable = { war_rp_penalty_effective = 0 }

  if = {
    limit = { has_war = yes }

    # Fixed -10%, independent of war support etc.
    set_variable = { war_rp_base_penalty = 0.10 }
    set_variable = { war_rp_penalty_effective = war_rp_base_penalty }
    multiply_variable = { war_rp_penalty_effective = -1 }
  }

  add_to_variable = { total_rp_modifier = war_rp_penalty_effective }
}
```

### 8.2. Changing law effects only

Similarly, you can override `dr_apply_law_rp_logic` and define new bonuses/penalties for your own ideas or laws:

```txt
dr_apply_law_rp_logic = {
  if = {
    limit = { has_game_rule = { rule = DR_LAW_RP_RULE option = DR_LAW_RP_ON } }

    # Example: Free Trade gives +3% instead of +2%
    if = {
      limit = { has_idea = free_trade }
      add_to_variable = { total_rp_modifier = 0.03 }
    }

    # ... more custom effects
  }
}
```

### 8.3. Changing the alliance bonus

`dr_apply_alliance_rp_logic`:

- computes the bonus from the number of allies,
- scales it with average relations,
- respects the game rule cap (`DR_ALLIANCE_RP_RULE`),
- uses a timer `dr_days_until_alliance_update` so the full loop is not run every day.

You can, for example, change only the caps by modifying the `alliance_bonus_cap` logic, or replace the calculation entirely as long as you set `alliance_bonus_pct` and add it to `total_rp_modifier` at the end.

---

## 9. Debugging and testing submods

For balance submods it's important to see what happens internally. The mod ships with its own debug tools:

- **Debug decision** `dynamic_research_slots_debug`  
  (see `common/decisions/dynamic_research_slots_decisions.txt`), visible when `is_debug = yes`.

- **Debug event** `dynamic_research_slots.4`  
  shows `total_research_power`, breakdown by factory type, facility share, current/target slots, Easy Slots, etc.  
  You can fire the event, for example, via console:
  - `event dynamic_research_slots.4` (for the current country)

Typical test flow for a submod:

1. Start a game with main mod + your submod active.  
2. For a test country, use console commands (`tag`, `pp`, `add_building_construction`, etc.) to set up factories/facilities.  
3. Use the debug event/decision to check whether:
   - your new thresholds are effective,
   - RP values per factory/facility look correct,
   - war/law/alliance modifiers behave as intended.

---

## 10. Best practices and common pitfalls

- **Override as few files as possible.**  
  In most cases `00_dr_dynamic_research_config.txt` is enough. Only touch `00_dr_dynamic_research_modifiers.txt` if you really need different war/alliance logic.

- **Don't duplicate core logic.**  
  Avoid copying `initialize_dynamic_research_slots` or `recalculate_dynamic_research_slots`. These effects are intentionally generic; rely on the config and modifier effects instead.

- **Respect game rules.**  
  If you introduce new rules, document them (localisation) and follow the existing pattern.  
  If you reinterpret existing rules, update the texts so players understand the new behaviour.

- **Compatibility with overhauls.**  
  Large overhauls often implement their own research mechanics. Make sure your submod is only used together with mods that actually use the Dynamic Research Slots logic – or clearly state in your workshop description which combinations are tested.

### 10.1. Performance considerations

**Important**: The base mod's hooks are empty and have **zero overhead**. However, what you put in them can significantly impact performance.

#### ✅ Performance-friendly patterns

1. **Use initialization hooks for expensive setup**:
   ```txt
   # Calculate once at startup, not every day
   dr_initialize_submods = {
     set_variable = { my_expensive_value = 100 }  # Expensive calculation here
   }
   
   # Use cached value daily
   dr_total_rp_modifier_submods = {
     add_to_variable = { total_research_power = my_expensive_value }
   }
   ```

2. **Use flags to avoid repeated checks**:
   ```txt
   dr_initialize_submods = {
     if = { limit = { has_idea = expensive_check }
       set_country_flag = my_submod_active
     }
   }
   
   dr_apply_factory_modifiers_submods = {
     # Fast flag check instead of expensive idea check every day
     if = { limit = { has_country_flag = my_submod_active }
       add_to_variable = { civilian_rp_modifier = 0.10 }
     }
   }
   ```

3. **Keep runtime hooks simple**:
   ```txt
   # Simple variable operations are fast
   dr_apply_factory_modifiers_submods = {
     if = { limit = { has_idea = my_idea }
       add_to_variable = { civilian_rp_modifier = 0.10 }
     }
   }
   ```

#### ❌ Performance anti-patterns (avoid these)

1. **Nested loops in runtime hooks**:
   ```txt
   # BAD: Runs every day for players, can be very slow
   dr_collect_facility_counts_submods = {
     every_owned_state = {
       every_other_country = {  # NESTED LOOP - VERY SLOW!
         # ...
       }
     }
   }
   ```

2. **Expensive calculations every recalculation**:
   ```txt
   # BAD: Loops over all countries every day
   dr_total_rp_modifier_submods = {
     every_other_country = {
       if = { limit = { is_in_faction_with = ROOT }
         # Complex logic runs every day...
       }
     }
   }
   ```

3. **Uncached state iterations**:
   ```txt
   # BAD: Recalculates every day instead of caching
   dr_apply_facility_rp_submods = {
     set_variable = { temp_count = 0 }
     every_owned_state = {  # Can be 50+ states
       if = { limit = { has_building = custom_building }
         ROOT = { add_to_variable = { temp_count = 1 } }
       }
     }
   }
   # Better: Count in dr_collect_facility_counts_submods and cache
   ```

**Performance testing**:
- Test in large games (70+ countries, 50+ states)
- Use debug events (`event dynamic_research_slots.4`) to profile
- Monitor daily tick times
- If your submod adds >1ms per day, optimize it

For detailed performance analysis, see `performance.md`.

If needed, this document can later be expanded with more "recipes".

