# Feature Implementation Statistics Skill

A TDD-verified skill for tracking and reporting quantitative metrics during feature implementation work.

## Quick Navigation

### For First-Time Users
- Start with [QUICK-START.md](QUICK-START.md) - 4-step minimal example and copy-paste template

### For Reference
- [SKILL.md](SKILL.md) - Full reference with all five dimensions, common mistakes, and best practices
- [HARNESS-INSTRUMENTATION.md](HARNESS-INSTRUMENTATION.md) - Tool-specific guidance for your harness

## What This Skill Does

Provides a structured framework for capturing five dimensions of metrics:

1. **Tokens** - Input/output cost by phase
2. **Context** - Memory loads, compactions, resets
3. **Memory** - Reads, writes, votes to stored facts
4. **Tools** - Execution counts, times, success rates, retries
5. **Outcome** - Files, tests, build results, verification

## When to Use

**Always track when:**
- Implementing a named feature
- Solving a non-trivial bug (> 30 min)
- Complex task with 5+ subtasks
- Cross-system integration

**Don't track:** Quick lookups, routine checks, pure exploration

## Example Output

```markdown
# Feature Statistics: User Authentication

## Token Consumption
| Phase | Input | Output | % |
|-------|-------|--------|---|
| Planning | 400 | 200 | 15% |
| Implementation | 2500 | 1800 | 70% |
| Testing | 500 | 400 | 15% |

## Tool Execution
| Tool | Count | Time | Success |
|------|-------|------|---------|
| view | 5 | 2.1s | 100% |
| bash | 3 | 79.5s | 66% |

## Outcome
- Files: 3 | Lines: +42 | Tests: 15/15 ✅ | Build: ✅

## Retry Analysis
- Failed: 1 | Rate: 33% | Time to fix: 2min

## Unavailable Metrics
- Token/tool breakdown (system provides totals only)
```

## Usage

1. **Start:** Invoke skill at beginning of feature work
2. **During:** Log tool calls, memory ops, phase transitions, failures
3. **End:** Fill in Feature Statistics Report template
4. **Deliver:** Include report in task completion summary

## How It Was Tested

Following Test-Driven Development:

- **RED** (baseline): Showed agents struggling with structure, missing dimensions
- **GREEN** (with skill): Verified agents produce structured reports correctly
- **REFACTOR** (edge cases): Tested multi-session work, retries, unavailable metrics

All three scenarios passed. Skill is production-ready.

## Files in This Directory

| File | Size | Purpose |
|------|------|---------|
| SKILL.md | 9.8KB | Complete reference guide |
| QUICK-START.md | 4.4KB | Minimal example and template |
| HARNESS-INSTRUMENTATION.md | 4.3KB | Tool-specific guidance |
| README.md | This file | Navigation and overview |

## Key Features

✅ **Structured approach** - Five dimensions prevent incomplete tracking  
✅ **Template-based** - Ready-to-fill report for consistency  
✅ **Pragmatic** - Works with partial data, acknowledges gaps  
✅ **TDD-verified** - Tested with 3 pressure scenarios  
✅ **Tool-agnostic** - Works across Copilot Code, Codex, Gemini CLI  
✅ **Actionable** - Includes quick start, templates, and examples  

## Next Steps

1. Read [QUICK-START.md](QUICK-START.md) for a 5-minute overview
2. Use the skill on your next feature
3. Build a dataset across features to optimize your approach

---

**Skill Name:** `feature-implementation-statistics`  
**Status:** Production-ready ✓  
**Created:** 2026-11-03
