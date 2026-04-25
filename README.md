# skills

A collection of [OpenCode](https://opencode.ai) agent skills.

## What's Inside

This repository contains self-contained, prompt-based skills that extend an OpenCode agent's capabilities. Each skill lives in its own directory and includes a `SKILL.md` file with instructions, templates, and heuristics the agent follows when the skill is invoked.

### Included Skills

| Skill | Directory | Description |
|-------|-----------|-------------|
| **reflect** | [`reflect/`](reflect/) | Session retrospective analysis. When invoked (e.g., `/reflect`), the agent scans the current conversation, categorizes effort, detects friction points, and generates a polished HTML report styled after the *WIRED* design system. The report is written to `docs/retro/` with a session-derived filename prefix and opened in your default browser. |

## Repository Structure

```
.
├── reflect/
│   └── SKILL.md              # Reflect skill definition & embedded HTML template
├── docs/
│   ├── retro/                # Generated retrospective reports
│   └── superpowers/
│       ├── specs/            # Design specifications
│       └── plans/            # Implementation plans
└── test-conversation.md      # Synthetic conversation for skill testing
```

## Installation

Skills are loaded by OpenCode from `~/.config/opencode/skills/`. To install a skill from this repo, create a symlink:

```bash
ln -s /path/to/this/repo/reflect ~/.config/opencode/skills/reflect
```

After symlinking, the skill is available in any OpenCode session. For example:

```
/reflect
```

## Design & Planning

- [`docs/superpowers/specs/2026-04-24-reflect-skill-design.md`](docs/superpowers/specs/2026-04-24-reflect-skill-design.md) — Architecture, analysis methodology, and WIRED template design for the `reflect` skill.
- [`docs/superpowers/plans/2026-04-24-reflect-skill-implementation.md`](docs/superpowers/plans/2026-04-24-reflect-skill-implementation.md) — Step-by-step implementation plan, verification checklist, and testing strategy (RED-GREEN-REFACTOR).

## Testing

The `test-conversation.md` file provides a synthetic session log used to validate the `reflect` skill's categorization and friction-detection logic.

## Contributing

Add new skills by creating a top-level directory with a `SKILL.md` file following the OpenCode skill format (YAML frontmatter + Markdown instructions). Update this README when adding skills.
