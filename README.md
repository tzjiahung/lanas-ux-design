# lanas-ux-design

Claude Code skills for product design work in Figma.

| Skill | What it does |
| --- | --- |
| [`mobbin-pattern-scout`](mobbin-pattern-scout/SKILL.md) | Competitive UI/UX analysis for a screen redesign. Pulls Mobbin references, places them in Figma beside your design, compares competitors' information architecture, and turns the findings into wireframes and design-system hi-fi variants. |
| [`new-component-checker`](new-component-checker/SKILL.md) | Audits a proposed new Figma component or variant against an existing design system (e.g. Microsoft 365 / Fluent 2 UI Kit), reports what breaks the system, ranked by severity, and fixes it in the file when asked. |

## Requirements

- [Claude Code](https://claude.com/claude-code)
- Figma MCP (both skills)
- Mobbin MCP (`mobbin-pattern-scout` only)

## Install

Copy a skill folder into your Claude Code skills directory:

```sh
git clone https://github.com/tzjiahung/lanas-ux-design.git
cp -r lanas-ux-design/mobbin-pattern-scout ~/.claude/skills/
cp -r lanas-ux-design/new-component-checker ~/.claude/skills/
```

Or place them in a project's `.claude/skills/` to scope them to that project.

## Usage

- **mobbin-pattern-scout**: share a Figma link to the screen you're redesigning and ask Claude to find competitor patterns, e.g. *"Find Mobbin references for this My Account screen and suggest layouts."*
- **new-component-checker**: share a Figma link to the new component and the design-system page it should follow, e.g. *"Does this new card follow our design system? Fix it."*

## Contributing

`main` is protected: open a pull request instead of pushing directly.
