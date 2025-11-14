# Dynamic Research Slots Reborn – Technical Overview

This document describes how the *Dynamic Research Slots Reborn* mod works internally and how to extend or integrate it.

The mod is based on the original Workshop mod (ID `2782571928`) but has been updated and expanded for current Hearts of Iron IV versions.

---

## Credits

- Original concept and implementation by **Lemur**  
  Steam profile: <https://steamcommunity.com/id/crouchinglemur>
- This version (“Reborn”) updates the mod for newer HOI4 versions, adds performance improvements, Experimental Facility integration and Custom Game Rules while keeping the original design philosophy.

---

## Core Concept

Instead of a fixed number of research slots, each country receives a dynamic number of slots based on its **Research Power (RP)**:

- Civilian factories, military factories and naval dockyards generate RP.
- Experimental Facilities (Physics / Naval / Air / Land) provide additional flat RP.
- When RP crosses defined thresholds, additional research slots are unlocked.
- Some slots are treated as **Easy Research Slots** that require less RP to unlock.

The system runs automatically for all countries and is fully configurable via Custom Game Rules.

---

## Files and Structure

### Scripted Effects

- `common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`

Main scripted effects:

- `initialize_dynamic_research_slots`
- `calculate_modifiers_to_rp`
- `recalculate_dynamic_research_slots`

### On Actions

- `common/on_actions/00_dynamic_research_slots_on_actions.txt`

Hooks the scripted effects into the game loop:

- `on_startup`
- `on_daily`

### Decisions & Events

- `common/decisions/dynamic_research_slots_decisions.txt`
- `common/decisions/categories/dynamic_research_slots_categories.txt`
- `events/dynamic_research_slot_events.txt`

Provides:

- Decision category `dynamic_research_slots_decisions`
- HELP decision and debug helper decisions
- Events for slot changes and initialization

### Localisation

- `localisation/english/dynamic_research_slots_l_english.yml`
- `localisation/english/dynamic_research_l_english.yml`

Contains:

- Texts for decisions and events
- Custom Game Rule names and descriptions
- Coloured breakdown of RP sources

### Game Rules

- `common/game_rules/00_dr_dynamic_research_rules.txt`

Adds two Custom Game Rules:

- `DR_EASY_SLOTS_RULE` (base Easy Slots)
- `DR_EASY_COST_RULE` (Easy Slot RP cost factor)

---

## Research Power (RP) Calculation

All RP logic lives in `recalculate_dynamic_research_slots` in  
`common/scripted_effects/00_dynamic_research_slots_scripted_effects.txt`.

### Base RP per Factory Type

During initialization:

- `research_power_per_civ = 3`
- `research_power_per_mil = 2`
- `research_power_per_nav = 2`

Per recalculation:

- `civilian_research_power`  
  = `research_power_per_civ * num_of_civilian_factories` (with modifier)
- `military_research_power`  
  = `research_power_per_mil * num_of_military_factories` (with modifier)
- `naval_research_power`  
  = `research_power_per_nav * num_of_naval_factories` (with modifier)

Modifiers:

- `civilian_rp_modifier`, `military_rp_modifier`, `naval_rp_modifier`  
  are reserved for future extensions; currently all default to `0` and are multiplied into the base values via:

```txt
set_variable = { temp = 1 }
add_to_variable = { temp = civilian_rp_modifier }
set_variable = { civilian_research_power = base_civilian_research_power }
multiply_variable = { civilian_research_power = temp }
```

### Experimental Facilities

After the three base RP contributions are summed into `total_research_power`, Experimental Facilities add flat RP:

- Physics / Nuclear Facility (`nuclear_facility > 0`): **+50 RP**
- Naval Facility (`naval_facility > 0`): **+35 RP**
- Air Facility (`air_facility > 0`): **+35 RP**
- Land Facility (`land_facility > 0`): **+35 RP**

Implementation:

```txt
set_variable = { total_research_power = 0 }
set_variable = { facility_research_power = 0 }

add_to_variable = { total_research_power = civilian_research_power }
add_to_variable = { total_research_power = military_research_power }
add_to_variable = { total_research_power = naval_research_power }

if = {
    limit = { nuclear_facility > 0 }
    add_to_variable = { total_research_power = 50 }
    add_to_variable = { facility_research_power = 50 }
}
if = {
    limit = { naval_facility > 0 }
    add_to_variable = { total_research_power = 35 }
    add_to_variable = { facility_research_power = 35 }
}
if = {
    limit = { air_facility > 0 }
    add_to_variable = { total_research_power = 35 }
    add_to_variable = { facility_research_power = 35 }
}
if = {
    limit = { land_facility > 0 }
    add_to_variable = { total_research_power = 35 }
    add_to_variable = { facility_research_power = 35 }
}
```

`facility_research_power` is used only for UI display – it is not part of the threshold table.

### Thresholds and Slot Targets

Threshold arrays are set in `initialize_dynamic_research_slots` and then mirrored into variables in `recalculate_dynamic_research_slots`. The important array is `research_for_slot` and derived `base_research_for_slot^i`.

Thresholds (RP required for slot `i`):

1. Slot 1: 0 RP
2. Slot 2: 0 RP
3. Slot 3: 50 RP
4. Slot 4: 200 RP
5. Slot 5: 400 RP
6. Slot 6: 700 RP
7. Slot 7: 1000 RP
8. Slot 8: 1500 RP
9. Slot 9: 2000 RP
10. Slot 10: 3000 RP

Selection logic:

```txt
for_each_loop = {
    array = research_for_slot

    if = {
        limit = {
            check_variable = {
                var = total_research_power
                value = research_for_slot^i
                compare = greater_than_or_equals
            }
        }
        set_variable = { target_research_slots = i }
    }
}
```

The loop walks all entries and sets `target_research_slots` to the highest `i` where `total_research_power >= research_for_slot^i`.

At the end:

- `set_research_slots = var:target_research_slots`
- `current_research_slots = amount_research_slots` (sync with engine)

---

## Easy Research Slots

Easy slots represent research slots that are easier to maintain (lower RP requirements). They start with:

- `easy_research_slots = 2`
- `easy_research_slot_coefficient = 0.6` (60% of normal RP cost)

### Conversion of Vanilla Slots to Easy Slots

If a country starts with more slots than `target_research_slots` (determined by RP), the difference is converted into Easy Slots:

```txt
if = {
    limit = {
        check_variable = { current_research_slots > target_research_slots }
    }
    set_variable = { removed_research_slots = var:current_research_slots }
    subtract_from_variable = { removed_research_slots = var:target_research_slots }
    set_research_slots = var:target_research_slots
    add_to_variable = { easy_research_slots = var:removed_research_slots }
}
```

### Game Rule: Base Easy Slots

The Custom Game Rule `DR_EASY_SLOTS_RULE` (group `RULE_GROUP_DYNAMIC_RESEARCH`) controls extra Easy Slots on top of the default behaviour.

Defined in `common/game_rules/00_dr_dynamic_research_rules.txt`:

- Options:
  - `DR_EASY_SLOTS_DEFAULT` (default – original behaviour)
  - `DR_EASY_SLOTS_PLUS_1` … `DR_EASY_SLOTS_PLUS_5` (+1 … +5 extra Easy Slots)

Hook in the script:

```txt
if = {
    limit = { has_game_rule = { rule = DR_EASY_SLOTS_RULE option = DR_EASY_SLOTS_PLUS_1 } }
    add_to_variable = { easy_research_slots = 1 }
}
...
```

### Game Rule: Easy Slot Cost Factor

The Custom Game Rule `DR_EASY_COST_RULE` controls `easy_research_slot_coefficient`:

- Options: 10%, 20%, 30%, 40%, 50%, 60% (default), 70%, 80%, 90%, 100%.
- Implemented by overriding `easy_research_slot_coefficient` in `initialize_dynamic_research_slots`.

Example:

```txt
if = {
    limit = { has_game_rule = { rule = DR_EASY_COST_RULE option = DR_EASY_COST_30 } }
    set_variable = { easy_research_slot_coefficient = 0.3 }
}
```

Easy slot thresholds are recalculated by scaling `research_for_slot^i` with this coefficient.

---

## Update Timing (On Actions)

### Startup

`common/on_actions/00_dynamic_research_slots_on_actions.txt`:

```txt
on_startup = {
  effect = {
    every_country = {
      initialize_dynamic_research_slots = yes
      calculate_modifiers_to_rp = yes
    }
  }
}
```

At game start:

- All countries get their variables and arrays initialized.
- No slots are changed yet – avoids weird jumps on day 0.

### Daily

```txt
on_daily = {
  effect = {
    # Initialize late-spawned countries
    if = {
      limit = { NOT = { has_country_flag = dynamic_research_slots_initialized } }
      initialize_dynamic_research_slots = yes
      calculate_modifiers_to_rp = yes
      recalculate_dynamic_research_slots = yes
    }

    # Player: recalculate every day
    if = {
      limit = { is_ai = no }

      if = {
        limit = { has_variable = dr_player_event_cooldown }
        subtract_from_variable = { dr_player_event_cooldown = 1 }
      }

      calculate_modifiers_to_rp = yes
      recalculate_dynamic_research_slots = yes
    }

    # AI: roughly once every 30 days, staggered
    if = {
      limit = { is_ai = yes }

      if = {
        limit = { NOT = { has_variable = dr_days_until_update } }
        set_temp_variable_to_random = {
          var = dr_rand_offset
          min = 1
          max = 30
          integer = yes
        }
        set_variable = { dr_days_until_update = var:dr_rand_offset }
      }

      subtract_from_variable = { dr_days_until_update = 1 }

      if = {
        limit = {
          check_variable = {
            var = dr_days_until_update
            value = 0
            compare = less_than_or_equals
          }
        }
        set_variable = { dr_days_until_update = 30 }
        calculate_modifiers_to_rp = yes
        recalculate_dynamic_research_slots = yes
      }
    }
  }
}
```

Summary:

- **Player countries**: full RP + slot recalculation **every day**.
- **AI countries**: only once every ~30 days, with a random offset per country to avoid lag spikes.

---

## Events and Anti-Spam Logic

Events are defined in `events/dynamic_research_slot_events.txt`:

- Namespace: `add_namespace = dynamic_research_slots`
- `country_event = dynamic_research_slots.help` – HELP popup (manual decision).
- `country_event = dynamic_research_slots.1` – slot count changed.
- `country_event = dynamic_research_slots.2` – Easy Slot count changed.

### Slot-Change Event Cooldown

In `recalculate_dynamic_research_slots`:

```txt
if = {
    limit = {
        check_variable = {
            var = target_research_slots
            value = current_research_slots
            compare = not_equals
        }
    }

    # Player event with cooldown
    if = {
        limit = { is_ai = no }

        if = {
            limit = {
                OR = {
                    NOT = { has_variable = dr_player_event_cooldown }
                    check_variable = {
                        var = dr_player_event_cooldown
                        value = 0
                        compare = less_than_or_equals
                    }
                }
            }
            country_event = dynamic_research_slots.1
            set_variable = { dr_player_event_cooldown = 14 }
        }
    }

    # AI can always fire the event (if shown at all)
    if = {
        limit = { is_ai = yes }
        country_event = dynamic_research_slots.1
    }
}
```

And in `on_daily`:

- `dr_player_event_cooldown` is decremented by 1 each day for the player.

Effect:

- Slots may change daily, but the **notification event** appears at most once every 14 days per player country.

---

## Decisions and UI

### Decision Category

- File: `common/decisions/categories/dynamic_research_slots_categories.txt`
- Category: `dynamic_research_slots_decisions`
  - Icon: `generic_research`
  - Always visible

### Decisions

`common/decisions/dynamic_research_slots_decisions.txt`:

- `dynamic_research_slots_help`
  - Available only for `is_ai = no`.
  - `complete_effect = { country_event = dynamic_research_slots.help }`.
  - Opens HELP popup with detailed explanation and thresholds.

- Debug helper decisions (visible in debug mode only):
  - `initialize_dynamic_research_slots`
  - `recalculate_dynamic_research_slots`

### Localisation and Colour Coding

`localisation/english/dynamic_research_slots_l_english.yml` shows the RP breakdown with different colours per source:

- Decision description:

```yml
Total RP: [?total_research_power|Y]
RP Required For Next Research Slot: [?next_research_slot_at|Y]

RP from civilian factories: [?civilian_research_power|G]
RP from military factories: [?military_research_power|R]
RP from naval dockyards: [?naval_research_power|C]
RP from experimental facilities: [?facility_research_power|O]
```

- HELP event uses the same colour scheme on the numbers, with descriptive text.

---

## Custom Game Rules (Dynamic Research Tab)

Defined in `common/game_rules/00_dr_dynamic_research_rules.txt` and localized in `dynamic_research_l_english.yml`.

Group:

- `RULE_GROUP_DYNAMIC_RESEARCH` – appears as its own tab in the Custom Game Rules menu.

Rules:

1. `RULE_DR_EASY_SLOTS` – Easy Research Slots (Base)
   - Options:
     - `Default`: original behaviour.
     - `+1 Easy Slot` … `+5 Easy Slots`: global additional Easy Slots.

2. `RULE_DR_EASY_COST` – Easy Slot RP Cost Factor
   - Options: 10%, 20%, 30%, 40%, 50%, **60% (default)**, 70%, 80%, 90%, 100%.

Both rules affect **all countries** in the campaign.

---

## Extensibility Notes

- **Additional RP Sources**:
  - You can add further contributions to `total_research_power` based on:
    - Ideas
    - National Spirits
    - Technologies
    - Country flags
  - Always update UI variables accordingly (`facility_research_power` or new variables).

- **Per-Country Tweaks**:
  - `research_power_per_civ/mil/nav` can be modified per country (e.g. via country-specific scripted effects or focus rewards).
  - You can add modifiers for specific tags using `calculate_modifiers_to_rp`.

- **Performance**:
  - Keep heavy logic inside `recalculate_dynamic_research_slots`.
  - Use the AI 30-day timer to avoid daily updates for all AI countries.

- **Compatibility**:
  - The mod does not overwrite vanilla technology or focus files.
  - It only adds:
    - New scripted effects
    - New on_actions file
    - New decisions/events
    - New localisation
    - New game_rules file

---

## Player-Facing Summary

For README-style usage instructions, you can extract the following points:

- Your number of research slots dynamically follows your industrial strength.
- Civilian, military and naval factories as well as Experimental Facilities all contribute to RP.
- The Dynamic Research Slots decision shows the current RP breakdown and next threshold.
- The HELP decision explains all thresholds and how Easy Research Slots work.
- Using Custom Game Rules, you can globally increase the number of Easy Slots and control how cheap they are to maintain.
