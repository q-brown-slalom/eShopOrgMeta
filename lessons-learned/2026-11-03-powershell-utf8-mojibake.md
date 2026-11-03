---
title: "Windows PowerShell read/write corrupts UTF-8 Markdown"
date: 2026-11-03
area: "eShopOrgMeta / agent tooling (Windows PowerShell 5.1)"
tags: [encoding, powershell, agents, markdown]
---

# Windows PowerShell read/write corrupts UTF-8 Markdown

**Situation:** An agent wrote the implementation plan
[2026-11-03-runtime-business-rules-and-experiments.md](../plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments.md)
as UTF-8. It then fixed two lines with a PowerShell `Get-Content -Raw` → `.Replace()` →
`Set-Content -Encoding utf8` round trip.

**What went wrong / surprised us:** Windows PowerShell 5.1's `Get-Content` without `-Encoding` reads a
BOM-less file as the ANSI code page (Windows-1252), not UTF-8. Writing the result back as UTF-8
double-encoded every non-ASCII character, e.g. `—` → `â€”`, `→` → `â†’`, `≤` → `â‰¤`, `Σ` → `Î£`.
`Set-Content -Encoding utf8` also adds a BOM. The plan was full of mojibake until it was repaired by
re-encoding the text as Windows-1252 and decoding the bytes as UTF-8.

**Lesson:** Don't round-trip Markdown through Windows PowerShell `Get-Content`/`Set-Content`. Edit files
with the editor/agent edit tool. If a script is unavoidable, read and write with explicit UTF-8 (no BOM),
e.g. Python `open(p, encoding="utf-8")`. Afterwards, check that no `â€`/`Ã`/`Î` sequences appear.

**Applied to:** [.github/instructions/markdown.instructions.md](../.github/instructions/markdown.instructions.md)
(applies to all `**/*.md` edits). No standard or ADR change needed.
