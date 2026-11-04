# Five Dimensions of Feature Statistics - Quick Reference

## Dimension 1: TOKENS (Cost)

**What it measures:** LLM model usage and cost  
**Why it matters:** Identifies expensive phases; enables cost optimization

### Required Metrics
- `total_input_tokens` - Sum of input across all tool calls
- `total_output_tokens` - Sum of output across all tool calls
- `tokens_per_phase` - Breakdown by planning/implementation/testing/review

### Optional
- `cache_read_tokens` - Cached prompt data (if supported)
- `cache_write_tokens` - Cached data written (if supported)
- `expensive_operations` - Top 3 token-consuming calls with reasons

### Example
```
| Phase | Input | Output | Total | % |
|-------|-------|--------|-------|---|
| Planning | 400 | 200 | 600 | 10% |
| Implementation | 2500 | 1800 | 4300 | 70% |
| Testing | 800 | 600 | 1400 | 20% |
| TOTAL | 3700 | 2600 | 6300 | 100% |
```

### Unavailable?
- Report what you can measure (tool call counts, phase durations)
- Explicitly document why exact token count isn't available

---

## Dimension 2: CONTEXT (Resource Pressure)

**What it measures:** Memory and context system strain  
**Why it matters:** Shows if work exceeded available context; informs when to split work

### Required Metrics
- `context_loads` - How many times shared context was loaded
- `context_compactions` - Times context was trimmed/reset
- `context_resets` - Full conversation resets (new session)
- `session_count` - If work spanned multiple sessions

### Example
```
- Context loads: 3 (Program.cs, CatalogApi.cs, Extensions.cs)
- Context compactions: 0
- Full resets: 0
- Sessions: 1
```

### Red Flags
- **3+ loads** = memory-heavy work
- **Any compaction** = context was tight
- **Multiple resets** = consider splitting next time

### Unavailable?
- Manual checkpoint: "session ID: X, context pressure: low/medium/high"
- Track file list in memory if system doesn't expose it

---

## Dimension 3: MEMORY (State Management)

**What it measures:** Use of persistent facts and stored knowledge  
**Why it matters:** Shows how much you leveraged recorded knowledge; tracks knowledge building

### Required Metrics
- `memory_reads` - Number of stored facts consulted
- `memory_writes` - New facts stored or updated
- `memory_votes` - Upvotes/downvotes to existing facts
- `memory_invalidations` - Facts marked obsolete

### Track Specific Facts
✅ DO THIS:
```
- Reads: 2
  - fact: "AppHost uses forwarded headers via BeforeStartEvent"
  - fact: "eShop uses Aspire orchestration pattern"
```

❌ NOT THIS:
```
- Reads: 2
- Writes: 1
```

### Example
```
Memory Operations:
- Reads: 2 (forwarded headers fact, AppHost convention)
- Writes: 1 (new fact: health endpoint pattern)
- Votes: 1 upvote (confirmed HTTPS convention)
- Invalidations: 0
```

### Unavailable?
- Skip this dimension (it requires persistent memory system)
- Or: List facts you looked up manually instead

---

## Dimension 4: TOOLS (Operational)

**What it measures:** Which tools you used, how often, how long, success rate  
**Why it matters:** Identifies efficient vs inefficient approaches; shows bottlenecks

### Required Metrics
- `tool_calls_by_type` - Count of each tool (view, edit, bash, grep, etc.)
- `tool_execution_times` - Wall-clock seconds per tool or per phase
- `build_cycles` - Count of builds/tests and results
- `failed_attempts` - Commands that failed, with reason
- `retries_needed` - How many retry/fallback strategies used

### Example Table
```
| Tool | Count | Total Time | Avg Time | Success Rate | Notes |
|------|-------|-----------|----------|--------------|-------|
| view | 12 | 3.2s | 0.27s | 100% | File exploration |
| edit | 3 | 0.8s | 0.27s | 100% | Code changes |
| bash | 4 | 31.2s | 7.8s | 75% | 1 failed (syntax) |
| grep | 5 | 1.1s | 0.22s | 100% | Pattern search |

Build/Test Cycles:
- Builds: 2 attempted (1 failed, 1 passed)
- Tests: 1 run (38/38 passed)
- Avg build time: 15.6s
```

### Retry Tracking
```
Failed Attempts:
1. bash: dotnet build → syntax error in line 42
   - Fix: Corrected `ReadAsJsonAsync` to `ReadFromJsonAsync<JsonElement>`
   - Time to fix: 1 minute
   - Outcome: Build succeeded

Retry Rate: 1 failed / 5 actions = 20%
```

### Available?
Build and test results are almost always visible. Extract:
- `time` command for execution time
- Exit codes for success/failure
- Test runner output for counts

---

## Dimension 5: OUTCOME (Results)

**What it measures:** What was actually delivered  
**Why it matters:** Proves work was completed; enables regression detection

### Required Metrics
- `files_modified` - Count of distinct files changed
- `lines_changed` - Total lines added/deleted
- `tests_passed` - Final test count and pass rate
- `build_status` - Final build outcome (pass/fail, warning count)
- `verification_status` - Was outcome verified before reporting?

### Example
```
Outcome Metrics:
- Files modified: 3
- Lines added: 42
- Lines deleted: 0
- Tests passed: 38/38 ✅
- Build: ✅ Success (0 warnings, 0 errors)
- Verification: ✅ Verified via dotnet test + build

Files Changed:
- src/Catalog.API/Apis/CatalogApi.cs (7 lines added)
- src/Catalog.API/Catalog.API.http (4 lines added)
- tests/Catalog.FunctionalTests/HealthApiTests.cs (2 lines added)
```

### Always Verifiable
```bash
# Files
git diff --stat HEAD~1

# Build
dotnet build --no-restore; echo $?  # Exit code 0 = success

# Tests
dotnet test --no-build
```

---

## Quick Recall Table

| Dimension | Tracks | Key Question | Always? | Example |
|-----------|--------|--------------|---------|---------|
| **Tokens** | Cost | How expensive? | Optional | 6300 total tokens |
| **Context** | Pressure | Did I hit walls? | If available | 3 context loads |
| **Memory** | State | What did I learn? | If system has it | 2 facts read |
| **Tools** | Efficiency | What worked? | Always | 12 views, 1 failure |
| **Outcome** | Results | Did it work? | Always | 38/38 tests ✅ |

---

## Putting It Together

```markdown
# Feature Implementation Statistics: Health Endpoint

## Tokens (Optional)
[If available: token breakdown by phase]

## Context
- Loads: 3
- Compactions: 0
- Sessions: 1

## Memory
- Reads: 2 (auth convention, AppHost pattern)
- Writes: 1 (health endpoint pattern)

## Tools
| Tool | Count | Time | Success |
| view | 5 | 2.1s | 100% |
| bash | 3 | 79.5s | 66% |

Build Cycles: 2 attempted (1 failed, 1 passed)

## Outcome
- Files: 2 | Lines: +11 | Tests: 38/38 ✅ | Build: ✅

## Retry Analysis
- Failed: 1 (syntax error)
- Rate: 33%
- Fixed in: 1 minute
```

This pattern works across all projects and harnesses.
