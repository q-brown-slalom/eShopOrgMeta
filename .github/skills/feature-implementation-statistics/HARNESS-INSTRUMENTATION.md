# Harness-Specific Instrumentation Guide

This guide explains how to extract and track statistics based on your specific runtime/harness.

## Copilot Code (Claude Code)

### Token Metrics
- Available via: Session context + checkpoint notifications
- Track at: End of each major phase
- Command: Check current session token usage in footer

### Context Metrics
- `context_loads`: Count when you explicitly read from `.agents/memory/`
- `context_compactions`: Check notifications in console for "context compacted"
- `session_count`: Use `list_sessions` to count distinct sessions

### Memory Operations
- `memory_reads`: Each `store_memory` call to read existing facts
- `memory_writes`: Each new `store_memory` call
- `memory_votes`: Each `vote_memory` call

### Tool Execution
- Tools are logged automatically in session history
- Extract from: `read_bash`, `list_bash`, session_store_sql query
- Build times: Parse bash output for timing (e.g., `dotnet build` output)

### Outcome Metrics
- Files: Track with `git diff --stat` at session end
- Tests: Parse test runner output (xUnit, MSTest, NUnit output format)
- Build: Check exit codes and warning counts

## Codex

### Token Metrics
- Not directly exposed in Codex
- Workaround: Estimate from tool call log (view = ~200 tokens, bash output = ~X tokens/line)
- Track: Manual checkpoint at phase boundaries

### Context Metrics
- Codex doesn't expose context compaction
- Track: Session ID changes indicate new context
- Use `/sessions` to list active sessions

### Memory Operations
All skipped in Codex (no persistent memory across sessions)

### Tool Execution
- Tools logged in execution history
- Times extracted from bash output only
- Use: `measure-command` (PowerShell) or `time` (bash) around commands

### Outcome Metrics
Same as Claude Code

## Gemini CLI / Pi

### Token Metrics
- Depends on Gemini Flash vs Pro model
- Check: Runtime logs for token usage per API call
- Track: Input/output token counts from response headers

### Context Metrics
- Gemini models have fixed context windows (50K tokens typical)
- Count: Estimate remaining context = window - current usage
- Track: When you move to new file/branch

### Memory Operations
Not exposed (depends on custom integration)

### Tool Execution
Same as Codex

### Outcome Metrics
Same as Claude Code

## Generic Approach (Works Anywhere)

If your harness doesn't expose exact metrics:

### Tokens
- Estimate from tool call count: simple commands ~100-500 tokens, file reads ~1 token/line
- Track: number of calls only (exact count is valid metric)

### Context
- Manual checkpoint: "session ID: X, files in memory: [list]"
- Track: whenever you move to new branch/session

### Memory
- Only if using persistent memory system
- Otherwise, skip this dimension

### Tool Execution
- Count: number of each tool type used
- Time: Extract from bash/shell output (`.Elapsed` in PowerShell, `time` in bash)
- Success: Count failures vs passes

### Outcome
- Always available: files changed, tests passed, build exit code
- Use: `git diff --stat`, test runner exit codes, `dotnet build` output

---

## Example: Extracting Metrics from Session Data

```bash
# Count tool uses from session history
git log --oneline | grep "view\|edit\|bash\|grep" | wc -l

# Files changed
git diff --stat HEAD~0

# Build time from bash history
grep "real" .session/bash-times.log | awk '{sum += $2} END {print sum}'

# Test results from latest run
dotnet test --list-tests | tail -1
```

## Template for Custom Tracking

Create a metrics.json file at session start:

```json
{
  "feature_name": "Add health endpoint",
  "session_id": "abc123",
  "start_time": "2026-11-03T11:09:28Z",
  "phases": {
    "planning": {
      "tool_calls": 5,
      "memory_reads": 0,
      "memory_writes": 0,
      "duration_minutes": 3
    },
    "implementation": {
      "tool_calls": 12,
      "file_edits": 2,
      "build_cycles": 1,
      "failed_attempts": 0,
      "duration_minutes": 8
    },
    "testing": {
      "test_runs": 1,
      "tests_passed": 38,
      "tests_failed": 0,
      "duration_minutes": 2
    }
  },
  "unavailable_metrics": [
    "exact_token_usage (harness does not expose per-call tokens)",
    "memory_heap_allocation (not instrumented)",
    "lsp_latency (no performance hooks)"
  ]
}
```

Then populate this as you work, and use it to generate your final report.
