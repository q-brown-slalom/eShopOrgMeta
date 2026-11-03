# eShopOrgMeta

Organization-level memory for the eShop org: ways of working, standards, architecture,
decisions, templates, and every spec/plan produced across repos. Written for humans and
AI agents alike — agents start at [AGENTS.md](AGENTS.md).

## Getting started

1. Clone this repo **next to** the product repos:

   ```text
   AE-Immersion/
     eShop/
     eShopOrgMeta/
   ```

2. Open [eshop-org.code-workspace](eshop-org.code-workspace) in VS Code. Copilot then sees this
   repo's `AGENTS.md`, instructions, prompts, and skills alongside the product code.
3. Superpowers setup per agent host:

   | Host | Setup |
   |------|-------|
   | VS Code Copilot Chat | Nothing — skills load from [.github/skills/](.github/skills/) via the workspace. |
   | GitHub Copilot CLI | Run from this repo or the workspace root; or install the plugin: `copilot plugin marketplace add obra/superpowers-marketplace` then `copilot plugin install superpowers@superpowers-marketplace`. |
   | Claude Code | `/plugin install superpowers@claude-plugins-official`. [CLAUDE.md](CLAUDE.md) imports `AGENTS.md` so the org path overrides apply. |

   Keep plugin installs on the same version as the vendored copy (see
   [.github/skills/VENDORED.md](.github/skills/VENDORED.md)).

## How product repos consume this repo

- **Now:** side-by-side clone + multi-root workspace ([ADR-0004](decisions/0004-consumption-multiroot-then-ci-sync.md)).
- **Later:** a GitHub Action in this repo opens PRs that sync selected files into each product
  repo's `.github/` so CI and the Copilot coding agent see them.

## Contributing

- Changes go through PRs; standards changes need an ADR ([templates/adr.md](templates/adr.md)).
- Owners are listed in [.github/CODEOWNERS](.github/CODEOWNERS).
