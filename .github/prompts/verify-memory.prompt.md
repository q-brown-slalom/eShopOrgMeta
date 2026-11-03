---
description: "Re-verify an eShopOrgMeta memory file against the current product code"
agent: agent
argument-hint: "Path of the memory file, e.g. architecture/integration-events.md"
---

Verify the given eShopOrgMeta memory file is still accurate.

1. Read the file and its `source` front matter.
2. Check each factual claim against the cited source in the product repo(s).
3. Report: confirmed, outdated (with evidence), and unverifiable claims.
4. After I confirm, update the file and set `last_reviewed` to today.
