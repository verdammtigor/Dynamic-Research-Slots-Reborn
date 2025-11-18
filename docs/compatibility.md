# Compatibility Documentation: Dynamic Research Slots Reborn

> **Document Note**: This documentation was created with AI assistance to provide more comprehensive and detailed coverage, but has been manually reviewed and controlled by a human. The compatibility information has been verified, but if you notice any errors or have suggestions for improvement, please report them.

## Overview

The mod provides a comprehensive compatibility architecture with submod hooks, opt-out mechanisms, and compatibility signals for other mods. This documentation describes the available features and how they can be used.

---

## 1. Compatibility Features

### 1.1. Submod Hook System

The mod provides **9 hooks** that submods can override to customize behavior (8 hooks since version 1.0, 1 additional hook since version 1.4):

#### Initialization Hooks (run once at startup per country):

- **`dr_check_compatibility_submods`** - Runs at the very start of initialization (before config). Enables compatibility checks and setting flags before the config phase.

- **`dr_initialize_submods`** - Runs after the config phase. Ideal for initializing custom variables or performing setup tasks.

- **`dr_apply_research_config_submods`** - Runs during config application, after default values. Perfect for small adjustments like "+1 RP per Civ Factory". For more extensive changes, the entire config file should be overridden.

#### Runtime Hooks (run during every recalculation):

- **`dr_apply_factory_modifiers_submods`** - Runs before factory RP calculation. Enables setting multiplicative modifiers for factory types based on ideas, national spirits, focuses, etc.

- **`dr_adjust_research_thresholds_submods`** - Runs after Easy Slot logic. Enables dynamic adjustment of research slot thresholds (e.g., difficulty-based scaling, country-specific tweaks).

- **`dr_collect_facility_counts_submods`** - Runs after counting standard facilities. Enables counting custom buildings or scripted states that should be converted into facility-style RP.

- **`dr_apply_facility_rp_submods`** - Runs immediately after standard facilities are converted to RP. Enables adding custom RP sources or additional RP buckets.

- **`dr_total_rp_modifier_submods`** - Runs after the global `total_rp_modifier` is applied. Enables final adjustments to `total_research_power` (e.g., faction-wide buffs, ideology bonuses).

- **`dr_get_opinion_factor_for_ally_submods`** *(since version 1.4)* - Runs during alliance RP calculation for each alliance member. Allows submods to override the opinion factor calculation (0.0 to 1.0) used for alliance bonuses. Submods must set the `dr_opinion_factor_custom_set` flag when providing a custom value. See technical documentation for details.

**Usage**: All hooks are empty in the base mod. Submods can override them by creating a file with the same name (see `docs/submod_quick_start_template.md` or `example_submods/` for examples).

### 1.2. Opt-Out Mechanisms

The mod provides **4 Country Flags** for granular control:

- **`dr_disable_dynamic_research_slots`** - Disables the entire system for this country. Variables are kept in sync, but no slot changes occur.

- **`dr_disable_research_slot_changes`** - Disables slot changes but still calculates RP (useful for display/debug purposes).

- **`dr_disable_rp_modifiers`** - Disables RP modifier logic (war/peace/alliance/law modifiers). Factory and facility RP are still calculated normally.

- **`dr_disable_facility_rp`** - Disables facility RP contributions (nuclear/naval/air/land facilities). Factory RP is still calculated normally.

**Usage**: Set these flags in your submod via `dr_check_compatibility_submods` or `dr_initialize_submods`:

```txt
dr_check_compatibility_submods = {
  # Disable system for specific countries
  if = { limit = { OR = { tag = TAG1 tag = TAG2 } }
    set_country_flag = dr_disable_dynamic_research_slots
  }
}
```

### 1.3. Compatibility Signals for Other Mods

The mod sets standard flags and variables that other mods can use for detection and interaction:

- **`dynamic_research_slots_active`** (Global Flag) - Set at startup to indicate the system is active.

- **`dr_mod_version`** (Variable) - Contains the mod version (currently 1.4), enables version checks. The version is defined in `common/scripted_effects/00_dr_mod_metadata.txt` and should match the version in `descriptor.mod`.

**Usage in other mods**:

```txt
# Check if Dynamic Research Slots is active
if = { limit = { has_global_flag = dynamic_research_slots_active }
  # Adjust own logic
}

# Version check
if = {
  limit = { check_variable = { var = dr_mod_version value = 1.1 compare = less_than } }
  # Compatibility handling for older versions
}
```

### 1.4. Config File Override

Submods can override the entire config file `00_dr_dynamic_research_config.txt` to make extensive changes. This is the standard HOI4 approach and works for both small and large changes.

**For small tweaks**: Use `dr_apply_research_config_submods`.  
**For extensive changes**: Override the entire config file.

---

## 2. Submod Creation

### 2.1. Quick Start

1. Copy the code blocks from `docs/submod_quick_start_template.md` to your submod at `common/scripted_effects/ZZ_your_submod_name.txt`
2. Uncomment the hooks you want to use by removing the `#` symbols
3. Adjust the values to fit your needs
4. Make sure your submod loads AFTER "Dynamic Research Slots Reborn" in the launcher

### 2.2. Available Resources

- **`docs/submod_quick_start_template.md`** - Quick-start template with all 9 hooks (copy code blocks, uncomment, adjust)
- **`docs/submodding_guide.md`** - Complete submodding guide with detailed examples
- **`docs/performance.md`** - Comprehensive performance analysis including base mod performance, compatibility features, and best practices for submod developers

### 2.3. Multi-Submod Coordination

**Important**: If multiple submods override the same hook, the last loaded submod "wins". This is standard HOI4 behavior.

**Recommendation**: 
- Design hooks additively rather than overwriting where possible
- Document conflicts in the mod description
- Use unique variable names (e.g., `my_submod_*`)

---

## 3. Backward Compatibility

The mod is fully backward compatible:

- ✅ All hooks are empty and don't override anything in the base mod
- ✅ Existing hooks remain unchanged
- ✅ Opt-out mechanisms extend rather than replace
- ✅ No breaking changes for existing submods

---

## 4. Technical Details

### 4.1. Hook Structure

**9 hooks total** (since version 1.4):
- 3 Initialization hooks
- 6 Runtime hooks (5 since version 1.0, 1 additional since version 1.4)

**Performance Impact**: Empty hooks have practically no overhead (~0.001ms per hook). Performance impact depends entirely on submod implementation. See `docs/performance.md` for best practices.

### 4.2. Opt-Out Flags

**4 flags total**:
- Complete deactivation
- Disable slot changes
- Disable RP modifiers
- Disable facility RP

### 4.3. Compatibility Signals

- 1 Global Flag (`dynamic_research_slots_active`)
- 1 Variable (`dr_mod_version = 1.4`)

---

## 5. Examples

### Example 1: Add Custom Facility

```txt
dr_collect_facility_counts_submods = {
  set_variable = { quantum_lab_count = 0 }
  every_owned_state = {
    if = { limit = { quantum_lab > 0 }
      ROOT = { add_to_variable = { quantum_lab_count = 1 } }
    }
  }
}

dr_apply_facility_rp_submods = {
  if = { limit = { check_variable = { var = quantum_lab_count value = 0 compare = greater_than } }
    set_variable = { temp_facility_rp = quantum_lab_count }
    multiply_variable = { temp_facility_rp = 40 }
    add_to_variable = { total_research_power = temp_facility_rp }
    add_to_variable = { facility_research_power = temp_facility_rp }
  }
}
```

### Example 2: Factory Modifier Based on Ideas

```txt
dr_apply_factory_modifiers_submods = {
  if = { limit = { has_idea = my_custom_research_idea }
    add_to_variable = { civilian_rp_modifier = 0.10 }
  }
}
```

### Example 3: Compatibility Check

```txt
dr_check_compatibility_submods = {
  # Disable system if incompatible mod is active
  if = { limit = { has_global_flag = my_incompatible_research_mod }
    set_country_flag = dr_disable_dynamic_research_slots
  }
}
```

---

## Conclusion

The mod provides a solid foundation for compatibility and submodding:

- ✅ **9 Hooks** for maximum flexibility (8 since version 1.0, 1 additional since version 1.4)
- ✅ **4 Opt-Out Flags** for granular control
- ✅ **Compatibility Signals** for other mods
- ✅ **Full backward compatibility**
- ✅ **Negligible performance impact** (<0.3ms per day in base mod)
- ✅ **Comprehensive documentation** and ready-to-use template

For more information, see:
- `docs/submodding_guide.md` - Complete submodding guide
- `docs/submod_quick_start_template.md` - Quick-start template for new submods
- `example_submods/` - Complete working example submods
- `docs/performance.md` - Comprehensive performance analysis
- `docs/technical_reference.md` - Technical documentation of internal logic

