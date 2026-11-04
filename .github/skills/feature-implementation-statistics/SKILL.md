---
name: feature-implementation-statistics
description: Use when implementing features or completing complex tasks to track, measure, and report quantitative metrics such as tokens used, context state changes, memory operations, tool execution times, and resource consumption for each feature.
---

# Feature Implementation Statistics

## Overview

**Principle:** Every feature work produces measurable data. Capture it systematically.

When you implement a feature, task, or solve a problem, you generate telemetry:
- Token consumption (input/output per tool call)
- Context state (loads, compactions, resets)
- Memory operations (reads, writes, updates)
- Tool execution times (CLI, LSP, file I/O)
- Build/test cycles and their outcomes
- External service calls (GitHub, APIs)

A **feature statistics report** is a structured record of these metrics, enabling:
- Performance analysis across features
- Identifying bottlenecks (which tools consume most tokens?)
- Cost tracking and optimization
- Retrospective analysis of approach efficiency

This skill defines *when* to measure, *what* to track, and *how* to report.

## When to Track

**Always track when:**
- Implementing a named feature or user story
- Solving a non-trivial bug (> 30 minutes estimated)
- Running a complex task with 5+ subtasks
- Integrating across 3+ systems or codebases
- Anything the human partner will review or decide about

**Don't track:**
- Quick lookups or clarifications (< 5 min)
- Routine status checks
- Exploration with no decision point

## What to Track

Organize measurements into **five dimensions:**

### 1. **Token Metrics** (Cost)
Required:
- `total_input_tokens` — sum of all input tokens across all tool calls
- `total_output_tokens` — sum of all output tokens across all tool calls
- `tokens_per_phase` — breakdown by phase (planning, implementation, testing, review)

Optional:
- `cache_read_tokens` — if your harness supports prompt caching
- `cache_write_tokens` — if your harness supports prompt caching
- `expensive_operations` — list top 3 token-consuming tool calls with reason

### 2. **Context Metrics** (Resource Pressure)
Record:
- `context_loads` — how many times memory/shared context was loaded
- `context_compactions` — how many times context was trimmed/reset
- `context_resets` — complete conversation resets (moving to new session/branch)
- `session_count` — if work spanned multiple sessions

### 3. **Memory Operations** (State)
Track:
- `memory_reads` — how many stored facts were consulted
- `memory_writes` — new facts stored or updated
- `memory_votes` — upvotes/downvotes to existing facts
- `memory_invalidations` — facts marked as obsolete

### 4. **Tool Execution** (Operational)
Measure:
- `tool_calls_by_type` — count of each tool used (view, edit, bash, grep, etc.)
- `tool_execution_times` — wall-clock seconds per phase or per tool category
- `build_cycles` — count of builds/tests and their pass/fail results
- `failed_attempts` — commands that failed (with reason)
- `retries_needed` — how many retry/fallback strategies were used

### 5. **Outcome Metrics** (Results)
Document:
- `files_modified` — count of distinct files changed
- `lines_changed` — gross add/delete count
- `tests_passed` — final test pass count
- `build_status` — final build outcome (pass/fail/warning count)
- `verification_status` — was outcome verified before reporting?

## Measurement Strategy

### Setup Phase (Start of Work)
Create a **metrics checkpoint** as a simple table or JSON:
```
START: 2026-11-03 11:09:28
  session_id: <current-session-id>
  phase: planning
  
CHECKPOINT_START = {
  tokens_used_at_start: <if visible>,
  context_loads: 0,
  tool_calls: []
}
```

### During Work
After each major phase or tool call:
- Note the phase transition (e.g., "planning → implementation")
- If tokens are visible, record them
- Track memory operations (read/write a fact)
- Log failed attempts with reason

Example inline:
```
TOOL: bash → build project
  duration: 12s
  result: failed (syntax error in line 42)
  
TOOL: view → inspect error
  lines_read: 5
  context_load: yes (to reread memory about project structure)
  
RETRY: fix syntax error
  result: success
```

### Finalization Phase (End of Work)
Compile all measurements into a **Feature Statistics Report** (see template below).

## Report Template

Use this structure for feature statistics output:

```markdown
# Feature Implementation Statistics: [Feature Name]

## Overview
- **Start Time:** YYYY-MM-DD HH:MM:SS
- **End Time:** YYYY-MM-DD HH:MM:SS
- **Total Duration:** X minutes
- **Session(s):** [session IDs or count]

## Token Consumption
| Phase | Input | Output | Total | % of Total |
|-------|-------|--------|-------|-----------|
| Planning | 500 | 200 | 700 | 5% |
| Implementation | 2000 | 1500 | 3500 | 60% |
| Testing | 1000 | 800 | 1800 | 30% |
| Review | 200 | 100 | 300 | 5% |
| **TOTAL** | **3700** | **2600** | **6300** | **100%** |

Cache metrics (if available):
- Cache read tokens: X
- Cache write tokens: Y

Most expensive operations:
1. [Tool name] - 800 tokens (reason: large file read)
2. [Tool name] - 650 tokens (reason: LSP analysis)

## Context Metrics
- Context loads: 3
- Context compactions: 0
- Full resets: 0
- Sessions involved: 1

## Memory Operations
- Reads: 2 (fact: X, fact: Y)
- Writes: 1 (new fact: Z)
- Votes: 0
- Invalidations: 0

## Tool Execution Summary
| Tool | Count | Total Time | Avg Time | Success Rate |
|------|-------|-----------|----------|--------------|
| view | 12 | 3.2s | 0.27s | 100% |
| edit | 3 | 0.8s | 0.27s | 100% |
| bash | 4 | 31.2s | 7.8s | 75% (1 fail) |
| grep | 5 | 1.1s | 0.22s | 100% |

Build/Test cycles:
- Builds attempted: 2 (1 failed, 1 passed)
- Tests run: 1 (passed)
- Avg build time: 15.6s

## Outcome Metrics
- Files modified: 1
- Lines added: 7
- Lines deleted: 0
- Tests passed: ✅ All passing
- Build status: ✅ Success (0 warnings, 0 errors)
- Verification: ✅ Verified before completion

## Insights
- Most expensive phase: Implementation (60% of tokens)
- Biggest blocker: [Describe if applicable]
- Retry rate: 1 out of 5 tool invocations required retry
- Most-used tool: view (12 calls, 3.2s total)

## Lessons Learned
- [What would make this faster next time?]
- [Which tool was most impactful?]
- [Did this approach work well?]
```

## Common Mistakes

❌ **Tracking only tokens, ignoring context stress**  
Tokens alone don't show if you hit a wall. Context loads and compactions matter equally.

❌ **Forgetting failed attempts**  
Retries and failures are data. They show approach efficiency. Include them.

❌ **Reporting percentage without denominators**  
"60% of tokens on implementation" means nothing without the total. Always show the total.

❌ **Starting measurement mid-work**  
Measure from the start, even if you only get partial data. A checkpoint at any point beats no measurement.

❌ **Mixing metrics with narrative**  
"Implementation was tricky" is not a metric. Stick to numbers. Insights come *after* the data.

❌ **Abandoning measurement if tools don't expose data**  
If exact token counts aren't visible, use: build times, file counts, tool call counts, test results. Report what you *can* measure.

## Implementation Workflow

1. **Start work** → Create metrics checkpoint with timestamp
2. **During phases** → Log tool calls, memory reads/writes, phase transitions, failed attempts
3. **After each major step** → Quick snapshot of progress (files changed, tests status)
4. **Before finalizing** → Compile raw measurements into Feature Statistics Report
5. **Deliver report** → Include in task completion summary

## Structuring Across Phases

For multi-phase work, break measurements by phase:

```
PLANNING PHASE
├── Token usage: input/output
├── Memory loads: which facts consulted
├── Tools: grep, view (exploration)
├── Duration: XX minutes

IMPLEMENTATION PHASE
├── Token usage: input/output
├── Tools: edit, bash, create
├── Build cycles: attempts and results
├── Duration: XX minutes

TESTING PHASE
├── Token usage: input/output
├── Test runs: count and results
├── Retries: any failed tests first, then passed
├── Duration: XX minutes

REVIEW PHASE
├── Memory votes: any facts upvoted/downvoted
├── Verification: how was outcome confirmed
├── Duration: XX minutes
```

## When Measurement is Impossible

Some metrics genuinely aren't available. **Explicitly call this out:**

```
### Unavailable Metrics
- **Exact token breakdown per tool:** System provides cumulative tokens only
- **Compiler internal timing:** Only wall-clock build time visible
- **LSP response latency:** Not directly instrumented
- **Heap memory allocation:** Not accessible via available tools
```

Report what you *can* measure. Acknowledge gaps. Don't fabricate numbers.

## Cross-Feature Comparison

Over multiple features, you'll build a dataset:

```
Feature | Tokens | Context Loads | Build Cycles | Files Changed | Duration |
---------|--------|--------------|--------------|---------------|----------|
Auth    | 8500   | 2            | 3            | 4             | 45 min   |
API     | 6300   | 1            | 2            | 1             | 28 min   |
UI      | 12000  | 3            | 5            | 8             | 72 min   |
```

This enables:
- Identifying token-expensive patterns (UI > Auth > API)
- Spotting context pressure (3 loads signals memory-heavy work)
- Build-cycle efficiency (fewer attempts = cleaner approach)
```

---

**Quick Reference: The Five Dimensions**

| Dimension | Tracks | Key Metric |
|-----------|--------|-----------|
| **Tokens** | Cost, model usage | Total input + output tokens |
| **Context** | Resource pressure | Loads, compactions, resets |
| **Memory** | State management | Reads, writes, votes |
| **Tools** | Operational | Calls per type, execution time, retries |
| **Outcome** | Results | Files, tests, build status |
