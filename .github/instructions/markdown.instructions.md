---
description: "Use when creating or editing any Markdown file in eShopOrgMeta: preserve UTF-8 encoding."
applyTo: "**/*.md"
---

- Files are UTF-8 without BOM. Edit with the editor/agent edit tool, not a shell read/replace/write round trip.
- Never use Windows PowerShell `Get-Content`/`Set-Content` to rewrite Markdown: it reads BOM-less UTF-8 as
  ANSI and turns `—`, `→`, `≤` into `â€”`, `â†’`, `â‰¤`. If a script is unavoidable, use explicit UTF-8
  (e.g. Python `encoding="utf-8"`) and check afterwards that no `â€`/`Ã`/`Î` sequences appear.
- Background: [lessons-learned/2026-11-03-powershell-utf8-mojibake.md](../../lessons-learned/2026-11-03-powershell-utf8-mojibake.md).
