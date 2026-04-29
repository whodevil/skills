# skills

A collection of [agent skills](https://agentskills.io/home).

## What's Inside

This repository contains self-contained, prompt-based skills that extend an agent's capabilities. Each skill lives in its own directory and includes a `SKILL.md` file with instructions, templates, and heuristics the agent follows when the skill is invoked.

### Included Skills

| Skill | Directory | Description |
|-------|-----------|-------------|
| **readme** | [`readme/`](readme/) | README review and maintenance. When invoked (e.g., `/readme`), the agent analyzes the codebase, compares it against the existing README (or best practices if missing), and presents categorized improvement suggestions for user approval before applying them. |
| **retro** | [`retro/`](retro/) | Session retrospective analysis. When invoked (e.g., `/retro`), the agent scans the current conversation, categorizes effort, detects friction points, and generates a polished HTML report styled after the *WIRED* design system. The report is written to `docs/retro/` with a session-derived filename prefix and opened in your default browser. |

## Repository Structure

```
.
├── readme/
│   └── SKILL.md              # README skill definition
├── retro/
│   └── SKILL.md              # Reflect skill definition & embedded HTML template
├── docs/
│   ├── retro/                # Generated retrospective reports
│   └── superpowers/
│       ├── specs/            # Design specifications
│       └── plans/            # Implementation plans
└── test-conversation.md      # Synthetic conversation for skill testing
```

## Installation

Skills are loaded by the agent from `~/.agents/skills/`. To install a skill from this repo, create a symlink:

```bash
ln -s /path/to/this/repo/readme ~/.agents/skills/readme
ln -s /path/to/this/repo/retro ~/.agents/skills/retro
```

After symlinking, the skill is available in any agent session. For example:

```
/readme
/retro
```

## Design & Planning

- [`docs/superpowers/specs/2026-04-25-readme-skill-design.md`](docs/superpowers/specs/2026-04-25-readme-skill-design.md) — Architecture, six-phase pipeline, approval loop, and error handling for the `readme` skill.
- [`docs/superpowers/plans/2026-04-25-readme-skill.md`](docs/superpowers/plans/2026-04-25-readme-skill.md) — Step-by-step implementation plan and testing strategy for the `readme` skill.
- [`docs/superpowers/specs/2026-04-24-reflect-skill-design.md`](docs/superpowers/specs/2026-04-24-reflect-skill-design.md) — Architecture, analysis methodology, and WIRED template design for the `retro` skill.
- [`docs/superpowers/plans/2026-04-24-reflect-skill-implementation.md`](docs/superpowers/plans/2026-04-24-reflect-skill-implementation.md) — Step-by-step implementation plan, verification checklist, and testing strategy (RED-GREEN-REFACTOR) for the `retro` skill.

## Testing

The `test-conversation.md` file provides a synthetic session log used to validate the `retro` skill's categorization and friction-detection logic.

## Contributing

Add new skills by creating a top-level directory with a `SKILL.md` file following the [agent skill](https://agentskills.io/specification) format (YAML frontmatter + Markdown instructions). Update this README when adding skills.
