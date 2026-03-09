---
name: audit
description: Use when user wants to analyze their Claude Code usage patterns, find repeated workflows, or identify what should become skills, plugins, agents, or CLAUDE.md rules. Triggers on "audit", "analyze my sessions", "usage patterns", "what do I repeat", "what should I automate".
---

# Audit

## Overview

Scrape all Claude Code session history on this machine, categorize every prompt, and produce a structured breakdown of what should become skills, plugins, agents, or CLAUDE.md entries.

**Core principle:** Let the data speak. Count everything, then prioritize by frequency and impact.

## Data Sources

1. **`~/.claude/history.jsonl`** — main history file (primary source). Each line is JSON:
   - `display`: the user's prompt text
   - `project`: the project path
   - `timestamp`: unix timestamp in milliseconds
   - `sessionId`: groups prompts into sessions (use for multi-step workflow detection)
   - `pastedContents`: dict keyed by index, each with `id`, `type`, `content` fields

2. **Existing automations** (check for duplicates before recommending):
   - `~/.claude/commands/` — custom commands (`.md` files)
   - `~/.claude/skills/` — personal skills
   - `~/.claude/plugins/installed_plugins.json` — installed plugins

## Workflow

### Phase 1: Extract

1. Read `~/.claude/history.jsonl` — parse every line as JSON, skip malformed lines (count and report parse errors)
2. Skip entries with empty or blank `display`
3. Extract: `display`, `project`, `timestamp`, `sessionId`
4. Normalize project names (extract last path segment)
5. Convert timestamps to local system timezone
6. Report: total prompts, unique projects, date range, any parse errors

### Phase 2: Categorize

**All matching is case-insensitive, word-boundary aware** — use `\b` word boundaries to avoid false positives (e.g. `\badd\b` matches "add redis" but not "additionally").

Run every prompt through these category checks (a prompt can match multiple):

| Category | Keyword patterns (word-boundary) |
|----------|----------------------------------|
| **Slash commands** | starts with `/` |
| **Git operations** | `\b(commit|branch|push|merge|rebase|cherry.pick|revert|stash|checkout)\b` |
| **Bug fixing** | `\b(fix|bug|error|broken|not working|crash|fail|debug)\b` |
| **Feature implementation** | `\b(implement|create|build)\b`, or `\badd\b` followed by a noun |
| **Investigation/learning** | `\b(what|how|why|where|check|look|find|show|explain|learn|investigate)\b` at start or near start |
| **UI/Frontend** | `\b(component|style|css|design|button|modal|form|layout|figma)\b`, or `[image` |
| **API/Backend** | `\b(api|endpoint|route|controller|service|migration|schema|model)\b` |
| **Testing** | `\b(test|spec|mock|jest|phpunit|cypress|playwright)\b` |
| **DevOps/Infra** | `\b(deploy|docker|terraform|aws|ci|pipeline|helm|k8s|kubernetes|gcp|ecr)\b` |
| **Code review** | `\b(review|pr|pull request|code review)\b` |
| **Refactoring** | `\b(refactor|rename|restructure|clean up|organize)\b` |
| **Database** | `\b(sql|postgres|mongo|redis|mysql|migration|table|database)\b` |
| **Monitoring** | `\b(sentry|monitor|alert|logging)\b` |
| **Performance** | `\b(performance|optimize|slow|cache|memory|profil)\b` |
| **Config/Setup** | `\b(config|env|setup|install|dependency|package|docker.compose)\b` |
| **Pasted content** | `[pasted text` or `[image` or non-empty `pastedContents` |

Also detect these specific recurring patterns:

| Pattern | Detection |
|---------|-----------|
| **Recurring exact prompts** | Normalize (lowercase, trim, collapse whitespace), count duplicates with count >= 2 |
| **Think hard / ultrathink** | Contains "think hard" or "ultrathink" |
| **Figma references** | Contains "figma" or `figma.com` URL |
| **Jira/ticket workflows** | Contains "jira", "ticket", "pr description", "ticket description" |
| **Curl/API debugging** | Contains `curl ` (with trailing space) |
| **Branch+commit+push** | Contains "create branch" + ("commit" or "push") |
| **Learn branch** | Contains "learn what was implemented" or "what was implemented in current branch" |

### Phase 3: Aggregate

Produce these aggregations (truncate tables to top 20 entries, note if more exist):

1. **Top projects by prompt count** — table: project, count
2. **Category distribution** — table: category, count, percentage of total
3. **Slash command frequency** — table: command, count
4. **Recurring exact prompts** — all prompts appearing 3+ times, sorted by frequency
5. **Tech stack indicators** — count mentions of specific frameworks/tools/languages
6. **Temporal patterns** — peak hours (local time), day-of-week distribution
7. **Per-project breakdown** — top 5 categories within each top-5 project
8. **Multi-step workflows** — group consecutive prompts by `sessionId`, identify common sequences (e.g. "implement → fix → commit → push" patterns)

### Phase 4: Classify into Buckets

For each recurring pattern or workflow, classify into one of four buckets.

**Time-saved estimation rubric:**
- Simple lookup/generation (e.g. git command, ticket description): **2-5 min saved**
- Multi-step workflow (e.g. branch+commit+push+PR): **5-15 min saved**
- Investigation/debugging workflow (e.g. Sentry triage): **15-30 min saved**
- Full implementation cycle (e.g. Figma to code with verification): **30-60 min saved**

#### 1. Skills (reusable prompts/workflows)
Criteria: repeatable thinking or writing task, can be captured as a prompt template with steps, runs within a single session.

For each item include:
- What the user keeps doing (with example prompts from data)
- Why it belongs here (not plugin/agent/CLAUDE.md)
- Frequency (exact count from data)
- Potential time saved (use rubric above)
- Suggested implementation (skill name, key steps)
- Priority: high/medium/low based on frequency × time saved

#### 2. Plugins/Tools (external integrations)
Criteria: needs external APIs, data sources, or system access that Claude doesn't have natively.

Same fields as above.

#### 3. Agents (autonomous multi-step workflows)
Criteria: multi-step process requiring decision-making, iteration, or orchestration across tools. Benefits from running autonomously with minimal user input.

Same fields as above.

#### 4. CLAUDE.md (rules and preferences)
Criteria: repeated instructions, preferences, project context, or constraints the user states multiple times. Things the user shouldn't have to say twice.

Same fields as above.

### Phase 5: Prioritized Recommendations

Produce these ranked lists:

1. **Top 10 skills to build** — sorted by (frequency × estimated time saved)
2. **Top 5 plugins/tools needed** — sorted by impact
3. **Top 5 agents to build** — sorted by autonomy value
4. **Most important CLAUDE.md sections missing** — sorted by how often the user repeats the instruction

Final section: **"Build first"** — pick the top 3 highest-impact items across all buckets, considering:
- Repeat frequency (from data)
- Time saved per use (from rubric)
- Ease of implementation (CLAUDE.md > skill > plugin > agent)

**Before recommending anything**, check existing automations (commands, skills, plugins) and note if something already partially exists.

## Output Format

Use markdown tables for data, bullet lists for recommendations. Keep each item concise (2-3 sentences max). Lead with numbers, not narrative. Truncate long tables to top 20 with a note.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Only analyzing recent history | Read ALL of `history.jsonl`, not just tail |
| Counting `/clear` as meaningful usage | Separate slash commands from real work prompts |
| Substring matching without word boundaries | Use `\b` regex boundaries to avoid "add" matching "additionally" |
| Ignoring pasted content signals | Non-empty `pastedContents` and `[pasted text` / `[image` are usage patterns |
| Guessing frequencies | Always show exact counts from data |
| Recommending things that already exist | Check `~/.claude/commands/`, installed plugins, and `~/.claude/skills/` first |
| Ignoring session grouping | Use `sessionId` to detect multi-step workflow patterns |
| Tables too long to read | Truncate to top 20, note total count |
