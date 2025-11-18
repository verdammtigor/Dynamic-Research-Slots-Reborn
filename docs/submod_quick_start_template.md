# Submod Quick-Start Template

This is a **QUICK-START template** containing all 9 hooks with inline examples (8 hooks since version 1.0, 1 additional hook since version 1.4). Use this for rapid prototyping. For complete working examples, see [`example_submods/`](../example_submods/).

## Instructions

1. **Copy the code blocks** below to your submod at:
   - `common/scripted_effects/ZZ_your_submod_name.txt`
   - (Replace "ZZ_your_submod_name" with your actual submod name)
2. **Uncomment** the hooks you want to use by removing the `#` symbols
3. **Adjust** the examples to fit your needs
4. **Make sure** your submod loads **AFTER** "Dynamic Research Slots Reborn" in the launcher

## Important Notes

- All 9 hooks are available and optional - only use the ones you need
- All hooks are empty in the base mod, so overriding them is conflict-free
- Comment extensively on what your submod does
- Test thoroughly with the main mod
- If you share this template publicly, mention it in the description
- Some hooks require specific flags to be set (see individual hook documentation)

## For Detailed Documentation

- [`docs/submodding_guide.md`](submodding_guide.md) - Complete submodding guide
- [`docs/compatibility.md`](compatibility.md) - Compatibility features documentation
- [`docs/performance.md`](performance.md) - Comprehensive performance analysis
- [`docs/technical_reference.md`](technical_reference.md) - Technical documentation of internal logic

---

## Hook 0: Compatibility Check (rare use case)

**Executes at the very start of initialization (BEFORE config is applied)**  
Use this only if you need code that runs before config. For most cases, use `dr_initialize_submods` instead.

```txt
dr_check_compatibility_submods = {
	# Example 1: Disable system if incompatible mod is active
	# if = {
	#   limit = { has_global_flag = my_incompatible_research_mod }
	#   set_country_flag = dr_disable_dynamic_research_slots
	# }
	
	# Example 2: Set flag before config runs (config logic might check this)
	# if = { limit = { tag = GER } set_country_flag = my_submod_special_mode }
	
	# Example 3: Version check before config
	# if = {
	#   limit = { check_variable = { var = dr_mod_version value = 1.0 compare = less_than } }
	#   set_country_flag = dr_disable_dynamic_research_slots
	# }
	
	# NOTE: This hook is rarely needed. Most compatibility checks can run in dr_initialize_submods instead.
}
```

---

## Hook 1: Initialization

**Executes once at game start (per country), AFTER config is applied**  
Use this hook to initialize custom variables or perform setup tasks.

```txt
dr_initialize_submods = {
	# Example 1: Initialize custom variable
	# set_variable = { my_submod_custom_rp = 0 }
	
	# Example 2: Set a flag for specific countries
	# if = { limit = { tag = GER } set_country_flag = my_submod_special_country }
	
	# Example 3: Compatibility check (most checks can run here instead of dr_check_compatibility_submods)
	# if = {
	#   limit = { has_global_flag = my_other_mod_active }
	#   set_country_flag = my_coordination_flag
	# }
	
	# Example 4: Signal that this submod is active
	# set_global_flag = my_submod_active
}
```

---

## Hook 2: Adjust Configuration (for small tweaks)

**Executes during config application, after vanilla defaults and game rules**  
Use this hook for small adjustments. For extensive changes, override the entire config file instead.

```txt
dr_apply_research_config_submods = {
	# Example 1: Small tweak - Increase RP per Civ Factory by 1
	# add_to_variable = { research_power_per_civ = 1 }
	
	# Example 2: Adjust facility RP values
	# set_variable = { rp_per_nuclear_facility = 60 }
	# set_variable = { rp_per_naval_facility = 40 }
	
	# NOTE: For extensive changes (e.g., changing all thresholds), it's better
	# to override the entire config file:
	# 1. Copy 00_dr_dynamic_research_config.txt to your submod
	# 2. Adjust values directly in dr_reset_research_config_defaults
	# 3. This is the standard HOI4 approach and easier to maintain
}
```

---

## Hook 3: Factory Modifiers (before RP calculation)

**Executes BEFORE the factory RP calculation**  
Use this hook to set multiplicative modifiers for factory types.

```txt
dr_apply_factory_modifiers_submods = {
	# Example 1: Custom idea gives +10% Civ Factory RP
	# if = {
	#   limit = { has_idea = my_custom_research_idea }
	#   add_to_variable = { civilian_rp_modifier = 0.10 }
	# }
	
	# Example 2: National spirit gives +5% Military Factory RP
	# if = {
	#   limit = { has_idea = my_military_research_spirit }
	#   add_to_variable = { military_rp_modifier = 0.05 }
	# }
	
	# Example 3: Focus gives +15% Naval Factory RP
	# if = {
	#   limit = { has_completed_focus = my_naval_research_focus }
	#   add_to_variable = { naval_rp_modifier = 0.15 }
	# }
}
```

---

## Hook 4: Adjust Thresholds (after Easy Slot logic)

**Executes AFTER the Easy Slot calculation**  
Use this hook to dynamically adjust thresholds.

```txt
dr_adjust_research_thresholds_submods = {
	# Example 1: Hard Mode - All thresholds +50%
	# for_each_loop = {
	#   array = research_for_slot
	#   multiply_variable = { research_for_slot^i = 1.5 }
	# }
	
	# Example 2: Adjust specific thresholds (e.g., only slots 3 and 4)
	# multiply_variable = { research_for_slot^3 = 1.2 }
	# multiply_variable = { research_for_slot^4 = 1.2 }
	
	# Example 3: Difficulty-based adjustment
	# if = {
	#   limit = { is_ai = no }  # Only for players
	#   for_each_loop = {
	#     array = research_for_slot
	#     if = {
	#       limit = { check_variable = { var = research_for_slot^i value = 0 compare = greater_than } }
	#       multiply_variable = { research_for_slot^i = 1.1 }  # +10% harder
	#     }
	#   }
	# }
}
```

---

## Hook 5: Count Custom Buildings

**Executes after counting standard facilities**  
Use this hook to count your own buildings.

```txt
dr_collect_facility_counts_submods = {
	# Example: Count Quantum Labs
	# set_variable = { quantum_lab_count = 0 }
	# every_owned_state = {
	#   if = {
	#     limit = { quantum_lab > 0 }
	#     ROOT = { add_to_variable = { quantum_lab_count = 1 } }
	#   }
	# }
	
	# Example: Count multiple building types
	# set_variable = { research_lab_count = 0 }
	# every_owned_state = {
	#   if = { limit = { research_lab > 0 } ROOT = { add_to_variable = { research_lab_count = 1 } } }
	# }
}
```

---

## Hook 6: Add Facility RP

**Executes after standard facility RP calculation**  
Use this hook to add your own RP sources.

```txt
dr_apply_facility_rp_submods = {
	# Example 1: Quantum Labs give +40 RP per lab
	# if = {
	#   limit = {
	#     check_variable = {
	#       var = quantum_lab_count
	#       value = 0
	#       compare = greater_than
	#     }
	#   }
	#   set_variable = { temp_facility_rp = quantum_lab_count }
	#   multiply_variable = { temp_facility_rp = 40 }
	#   add_to_variable = { total_research_power = temp_facility_rp }
	#   add_to_variable = { facility_research_power = temp_facility_rp }
	# }
	
	# Example 2: Flat RP bonus based on ideas
	# if = {
	#   limit = { has_idea = my_research_boost_idea }
	#   add_to_variable = { total_research_power = 50 }
	# }
	
	# Example 3: RP based on technologies
	# if = {
	#   limit = { has_tech = my_research_tech }
	#   set_variable = { tech_rp_bonus = 30 }
	#   add_to_variable = { total_research_power = tech_rp_bonus }
	# }
}
```

---

## Hook 7: Final RP Modification

**Executes AFTER all standard modifiers**  
Use this hook for final, global RP adjustments.

```txt
dr_total_rp_modifier_submods = {
	# Example 1: Ideology-based bonus
	# if = {
	#   limit = { has_government = democratic }
	#   if = { limit = { democratic > 0.8 }
	#     add_to_variable = { total_research_power = 25 }  # Flat +25 RP
	#   }
	# }
	
	# Example 2: Multiplicative modifier based on factors
	# if = {
	#   limit = {
	#     has_idea = my_research_focus_idea
	#     has_stability > 0.9
	#   }
	#   multiply_variable = { total_research_power = 1.05 }  # +5% RP
	# }
	
	# Example 3: Difficulty penalty for players
	# if = {
	#   limit = {
	#     is_ai = no
	#     difficulty = hard
	#   }
	#   multiply_variable = { total_research_power = 0.9 }  # -10% RP
	# }
}
```

---

## Hook 9: Override Alliance Opinion Factor (since version 1.4)

**Executes during alliance RP calculation for each alliance member**  
Use this to override the opinion factor calculation (0.0 to 1.0) used for alliance bonuses.

**IMPORTANT**: You must set the `dr_opinion_factor_custom_set` flag when providing a custom value, otherwise the default calculation will run.

```txt
dr_get_opinion_factor_for_ally_submods = {
	# Example 1: Simplified opinion calculation (2 tiers)
	# if = {
	#   limit = { has_opinion = { target = ROOT value > 50 } }
	#   set_variable = { alliance_rel_factor = 1.0 }
	# }
	# else = {
	#   set_variable = { alliance_rel_factor = 0.5 }
	# }
	# set_country_flag = dr_opinion_factor_custom_set  # REQUIRED: Signal custom value
	
	# Example 2: Explicitly set 0.0 for low opinion
	# if = {
	#   limit = { has_opinion = { target = ROOT value < 20 } }
	#   set_variable = { alliance_rel_factor = 0.0 }
	#   set_country_flag = dr_opinion_factor_custom_set  # REQUIRED even for 0.0
	# }
}
```

**Note**: This hook runs within the `every_other_country` loop during alliance RP calculation. The `ROOT` scope refers to the country calculating the alliance bonus, while the current scope is the alliance member being evaluated.

---

## Compatibility with Other Mods

If your submod should be compatible with a specific overhaul mod:
- Test thoroughly together with the main mod
- Document known compatibility issues
- Consider whether you need your own opt-out mechanisms

**Example: Opt-out for specific countries**

```txt
dr_apply_research_config_submods = {
	# Disable Dynamic Research Slots for specific countries
	# if = {
	#   limit = { OR = { tag = VANILLA_TAG_1 tag = VANILLA_TAG_2 } }
	#   set_country_flag = dr_disable_dynamic_research_slots
	# }
}
```

---

## Best Practices

### 1. Testing
- Test your submod thoroughly with the main mod
- Use the debug decisions (available in debug mode)
- Check RP values with the debug event (`event dynamic_research_slots.4`)

### 2. Variable Names
- Use unique variable names (e.g., `my_submod_*`)
- Avoid conflicts with the main mod or other submods
- Document all new variables

### 3. Performance
- Avoid unnecessary loops over all countries
- Use flags/variables to cache expensive calculations
- Test performance in large games

### 4. Documentation
- Comment your code extensively
- Describe in the mod description what your submod does
- Mention known compatibility issues

### 5. Versioning
- Test with the current version of the main mod
- Mention in the description which mod version you tested with
- React to updates of the main mod

