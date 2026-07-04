# tool-configs — turn Groundwork on in your AI tool

Each AI coding tool reads a small "rules" file. Copy the matching one into your project.
Same content, different filename per tool. Full walkthrough: [../TUTORIAL.md](../TUTORIAL.md) §4.

| Tool | Copy this | → to this path in your project |
|---|---|---|
| **Claude Code** | (use root `CLAUDE.md` + optionally `../AGENTS.md`) | project root |
| **Cursor** | `cursor.mdc` | `.cursor/rules/groundwork.mdc` |
| **GitHub Copilot** | `copilot-instructions.md` | `.github/copilot-instructions.md` |
| **Windsurf** | `windsurfrules.md` (contents) | `.windsurfrules` |
| **Cline / Roo Code** | `clinerules.md` (contents) | `.clinerules` / `.roorules` |
| **Any other tool** | `../AGENTS.md` | project root (`AGENTS.md`) |

## How to verify it's active
Ask your assistant: *"What does groundwork/standards-kb say about multi-tenancy?"* — if it answers
from the files (RLS, tenant_id, deny cross-tenant), the rules file is working.

## How to turn it OFF
Delete the tool's rules file (or the `groundwork/` folder). Nothing else depends on it.
