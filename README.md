# Claude Toolkit

Personal Claude Code plugin with custom skills, commands, and hooks.

## Installation

### Option 1: Install from local path

```bash
/plugin install /Users/bilyuk/project/claude-toolkit
```

### Option 2: Push to GitHub and install as marketplace

```bash
# After pushing to GitHub:
/plugin marketplace add <github-username>/claude-toolkit
/plugin install claude-toolkit@claude-toolkit
```

## Skills

| Skill | Description |
|-------|-------------|
| **figma-match** | Implement Figma designs with automated Playwright screenshot verification loop (up to 3 rounds) |
| **audit** | Analyze all Claude Code session history, categorize prompts, and recommend what to build as skills/plugins/agents/CLAUDE.md |

## Commands

| Command | Description |
|---------|-------------|
| `/figma-match` | Implement a Figma design with automated Playwright screenshot verification |
| `/audit` | Analyze all Claude Code session history and recommend automations |
| `/catchup` | Review current branch changes efficiently using diffs (context-optimized) |

## Structure

```
claude-toolkit/
├── .claude-plugin/
│   ├── plugin.json          # Plugin metadata
│   └── marketplace.json     # Marketplace catalog
├── skills/
│   ├── figma-match/SKILL.md
│   └── audit/SKILL.md
├── commands/
│   ├── figma-match.md
│   ├── audit.md
│   └── catchup.md
├── hooks/
└── README.md
```
