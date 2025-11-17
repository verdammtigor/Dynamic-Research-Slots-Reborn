# Example Submods

This directory contains **complete, working example submods** that demonstrate how to extend the Dynamic Research Slots Reborn mod.

> **Note**: These examples are **developer resources** available only on GitHub. They are not published on Steam Workshop. If you're viewing this on GitHub, you can browse the code directly. If you want to use these examples, clone or download the repository.

## Purpose

These examples are **reference implementations** showing:
- How to use the hook system
- Common patterns and best practices
- Complete, working code (not just snippets)

## Available Examples

### 1. `example_balance_tweak/`
**Difficulty**: ⭐ Easy  
**Demonstrates**: Simple config tweaks and conditional bonuses

A simple submod that:
- Increases civilian factory RP by +1
- Increases nuclear facility RP by +10
- Adds a stability bonus (+5% RP for countries with >80% stability)

**Best for**: Learning basic hook usage and simple variable operations.

---

### 2. `example_custom_facility/`
**Difficulty**: ⭐⭐ Medium  
**Demonstrates**: Adding custom buildings as RP sources

A submod that adds support for a fictional "Quantum Research Lab" building:
- Counts `quantum_lab` buildings across all states
- Each lab provides +40 flat RP

**Best for**: Learning how to count custom buildings and convert them to RP.

---

### 3. `example_factory_modifiers/`
**Difficulty**: ⭐ Easy  
**Demonstrates**: Factory RP modifiers based on ideas/spirits

A submod that adds factory RP bonuses:
- Free Trade idea: +10% civilian factory RP
- War Economy idea: +15% military factory RP

**Best for**: Learning how to add conditional factory modifiers.

---

## How to Use These Examples

1. **Read the README** in each example directory to understand what it does
2. **Examine the code** in `common/scripted_effects/` to see how it works
3. **Copy and adapt** the code to your own submod
4. **Test thoroughly** with the main mod

## Important Notes

- ⚠️ These examples are **for reference only** - they're not meant to be used as actual submods
- ⚠️ Some examples use fictional building types or ideas - adjust to your needs
- ⚠️ Always test your submods with the current version of Dynamic Research Slots Reborn

## Creating Your Own Submod

1. Start with the **quick-start template**: `docs/submod_quick_start_template.md` (all hooks with code blocks)
2. Look at these **example submods** for complete working code
3. Read the **documentation**: `docs/submodding_guide.md`
4. Test with **debug tools**: Use `event dynamic_research_slots.4` in console

## Need Help?

- Check `docs/submodding_guide.md` for detailed explanations
- Check `docs/compatibility.md` for hook reference
- Check `docs/technical_reference.md` for internal logic

