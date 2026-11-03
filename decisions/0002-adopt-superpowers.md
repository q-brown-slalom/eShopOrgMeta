# 0002. Adopt Superpowers for spec-driven development

- Status: Accepted
- Date: 2026-11-02
- Deciders: Q Brown

## Context

We want a repeatable agent workflow: clarify intent → design spec → plan → TDD implementation →
review. Developers use VS Code Copilot Chat, GitHub Copilot CLI, and Claude Code.

## Decision

Adopt [Superpowers](https://github.com/obra/superpowers) v7.0.1. Vendor its skills into
`eShopOrgMeta/.github/skills/` (unmodified) so VS Code and Copilot CLI load the same pinned
version; Claude Code users install the official plugin at the matching version. Org-specific
behavior is applied through `AGENTS.md` preferences, never by editing vendored skills.

## Consequences

- Positive: consistent, pinned workflow; TDD and review enforced by default.
- Negative: we own upgrades (see `.github/skills/VENDORED.md`). VS Code has no session-start hook,
  so `AGENTS.md` must instruct agents to load `using-superpowers`.
