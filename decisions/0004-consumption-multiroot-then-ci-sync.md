# 0004. Consume Meta via multi-root workspace, then CI sync

- Status: Accepted
- Date: 2026-11-02
- Deciders: Q Brown

## Context

Options considered: (1) side-by-side clone + multi-root workspace, (2) git submodule,
(3) CI job syncing files into each repo's `.github/`, (4) user-level skill install.

## Decision

Start with **(1)** using `eshop-org.code-workspace`. Add **(3)** once multiple product repos exist
or the Copilot coding agent / CI needs org instructions. Synced files will carry a
"generated from eShopOrgMeta — do not edit" header and a drift check.

## Consequences

- Positive: zero duplication now; edits are live.
- Negative: CI and cloud agents don't see org memory until (3) exists; developers must open the
  workspace file rather than a single folder.
