# skill-doctor

Your Claude Code skills probably have problems you don't know about. Missing tool restrictions, wasted tokens from inline file reads, vague descriptions that never auto-trigger. skill-doctor finds these issues and fixes them.

Three commands. Diagnose, treat, rollback.

## Install

```bash
claude plugin marketplace add amaljithkuttamath/skill-doctor
claude plugin install skill-doctor@skill-doctor-marketplace
```

Or test locally:

```bash
cc --plugin-dir /path/to/skill-doctor
```

## Commands

### `/skill-doctor` — Diagnose

Reads every skill, agent definition, and CLAUDE.md in your setup. Uses Claude Code's built-in `claude-code-guide` agent to evaluate each skill against current best practices, then saves findings to `~/.skill-doctor/diagnosis.json`.

Three modes:

| Mode | What it does |
|------|-------------|
| `/skill-doctor` | Interactive intake. Asks what's bothering you, then routes to the right analysis. |
| `/skill-doctor checkup` | Automated scan. Produces a report card with scores and top recommendations. |
| `/skill-doctor consult` | Guided Q&A. Connects findings to your actual pain points, one question at a time. |

### `/skill-doctor:treat` — Fix

Reads the diagnosis and builds upgraded versions of your skills in a staging directory. Nothing touches your originals until you say so.

```bash
# Test the upgrades in a separate session
cc --plugin-dir ~/.skill-doctor/staging-<date>-<source>/
```

If the upgrades work, say **migrate**. Originals get backed up to `~/.skill-doctor/backups/` before anything is replaced.

If they don't, say **discard** and nothing changes.

### `/skill-doctor:rollback` — Undo

Restores your original skills from the most recent backup. One command, no questions.

## What It Checks

skill-doctor doesn't use a hardcoded checklist. It sends each skill to Claude Code's `claude-code-guide` agent, which evaluates it against Claude Code's current documentation and capabilities. This means:

- Checks stay current as Claude Code adds new features (hooks, scripts, frontmatter fields)
- The agent understands mechanism tradeoffs (when to use a hook vs an instruction vs a script vs dynamic injection)
- Findings include specific fixes, not generic advice

It also checks cross-cutting concerns: overlap between skills, CLAUDE.md content that should be extracted to skills, and agent-skill wiring gaps.

## How It Works

```
/skill-doctor checkup
    │
    ▼
  Discover skills ──► claude-code-guide evaluates each ──► Score + save diagnosis.json
                                                        │
                                                        ▼
                                              /skill-doctor:treat
                                                        │
                                                        ▼
                                              Build staged upgrades
                                                        │
                                              ┌─────────┼─────────┐
                                              ▼         ▼         ▼
                                           migrate   tweak    discard
                                              │         │
                                              ▼         ▼
                                        backup + apply  adjust staged
```

1. **Diagnose** discovers all skills (`~/.claude/skills/`, `.claude/skills/`, plugins)
2. Sends each skill to the `claude-code-guide` agent for evaluation against current Claude Code docs
3. Scores each skill based on the agent's findings, saves to `~/.skill-doctor/diagnosis.json`
4. **Treat** reads the diagnosis and asks `claude-code-guide` how to implement each fix
5. Builds upgraded skills in `~/.skill-doctor/staging-<date>-<source>/`
6. You test with `cc --plugin-dir`, then migrate (with backup) or discard
7. **Rollback** restores from `~/.skill-doctor/backups/` if anything breaks

## Integrations

skill-doctor uses Claude Code's built-in `claude-code-guide` agent to fetch the latest skill authoring best practices at scan time. This means it stays current with Claude Code's own documentation without needing external plugins.

Optional integrations that add more capabilities:

| Integration | What it adds |
|-------------|-------------|
| [skill-creator](https://github.com/anthropics/skills) | Format validation and blind comparison evals for upgrades |

Without skill-creator, skill-doctor shows diffs instead of running evals.

## Requirements

Claude Code.

## License

MIT
