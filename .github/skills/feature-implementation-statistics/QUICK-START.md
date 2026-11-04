# Feature Implementation Statistics - Quick Start

## Minimal Example

### Step 1: Invoke the Skill
At the beginning of feature work:
```
I'm implementing a new feature. I'll use feature-implementation-statistics 
to track metrics during this implementation.
```

### Step 2: Create a Metrics Checkpoint
```
## Metrics Checkpoint
START: 2026-11-03 11:20:00
Feature: Add user authentication
Session: abc123
Phase: Planning

Initial state:
- tool_calls: 0
- memory_reads: 0
- memory_writes: 0
- build_cycles: 0
- failed_attempts: 0
```

### Step 3: Log As You Work
After each major action:
```
PHASE: Investigation
- Tool: view CatalogApi.cs (3.2s)
- Tool: grep existing auth patterns (0.8s)
- Memory: read fact about auth conventions

PHASE: Implementation
- Tool: edit Program.cs (add middleware)
- Tool: edit AuthApi.cs (add endpoints)
- Tool: bash dotnet build (26.5s, failed: syntax error)
- RETRY: fix syntax error
- Tool: bash dotnet build (26.5s, passed)
```

### Step 4: Generate Report
```markdown
# Feature Implementation Statistics: User Authentication

## Overview
- Start: 2026-11-03 11:20:00
- End: 2026-11-03 11:52:15
- Duration: 32 minutes
- Sessions: 1

## Token Consumption (if available)
| Phase | Input | Output | % |
|-------|-------|--------|---|
| Planning | 400 | 200 | 15% |
| Implementation | 2500 | 1800 | 70% |
| Testing | 500 | 400 | 15% |
| TOTAL | 3400 | 2400 | 100% |

## Tool Execution
| Tool | Count | Time | Success |
|------|-------|------|---------|
| view | 5 | 2.1s | 100% |
| edit | 3 | 1.2s | 100% |
| bash | 3 | 79.5s | 66% (1 fail) |
| grep | 2 | 0.6s | 100% |

## Memory Operations
- Reads: 2 (auth conventions, middleware patterns)
- Writes: 1 (new fact about eShop auth strategy)

## Outcome
- Files modified: 3
- Lines added: 42
- Tests passed: 15/15 ✅
- Build: Success ✅

## Retry Analysis
- Failed attempts: 1 (syntax error in auth middleware)
- Retry rate: 1/3 (33%)
- Time to fix: 2 minutes

## Unavailable Metrics
- Exact token/tool breakdown (system provides totals only)
- Memory heap allocation (not instrumented)
- LSP latency (no performance hooks)

## Insights
- Implementation was most expensive (70% of tokens)
- One retry due to typo suggests test-first would help next time
- grep was efficient: 2 calls, 0.6s, found needed patterns immediately
```

---

## Quick Reference: What to Track

| Dimension | You should... |
|-----------|--------------|
| **Tokens** | Record input/output per phase if available; acknowledge if not |
| **Context** | Count memory loads, note when context was compacted |
| **Memory** | List which facts you read/wrote/voted on |
| **Tools** | Count each tool (view, edit, bash, grep, etc.) and note times |
| **Outcome** | Report files changed, tests passed, build success |

---

## Red Flags (Things NOT to Do)

❌ Don't report percentages without totals: "70% on implementation" (70% of what?)  
❌ Don't forget failed attempts: "bash 2 times, passed both" (but you tried 3 times with 1 failure)  
❌ Don't mix metrics with narrative: "Implementation was tricky" (not a number, not actionable)  
❌ Don't start measuring mid-feature: "Realized I should track after 20 minutes" (measure from start)  
❌ Don't make up data: "Estimated 5000 tokens" without seeing a count (report what you can measure)  

---

## Copy-Paste Template for Your Work

```markdown
# Feature Implementation Statistics: [FEATURE NAME]

## Overview
- **Start:** [timestamp]
- **End:** [timestamp]
- **Duration:** [X minutes]
- **Sessions:** [count]

## Token Consumption
| Phase | Input | Output | Total | % |
|-------|-------|--------|-------|---|
| Planning |  |  |  | % |
| Implementation |  |  |  | % |
| Testing |  |  |  | % |
| Review |  |  |  | % |
| **TOTAL** | | | | 100% |

## Tool Execution
| Tool | Count | Time | Success Rate |
|------|-------|------|--------------|
| view |  | s | % |
| edit |  | s | % |
| bash |  | s | % |
| grep |  | s | % |

## Memory Operations
- Reads: [count] ([facts listed])
- Writes: [count] ([facts listed])

## Outcome
- Files modified: [count]
- Lines added: [count]
- Tests passed: [count/total] ✅/❌
- Build: ✅/❌

## Retry Analysis
- Failed attempts: [count]
- Retry rate: [X/Y]

## Unavailable Metrics
- [What you couldn't measure and why]

## Insights
- [What was most expensive?]
- [What would be faster next time?]
```

---

## When You're Done
1. Fill in the template above with your actual numbers
2. Include the report in your task completion summary
3. Over time, you'll build a dataset to optimize your approach
