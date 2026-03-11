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

Reads every skill, agent definition, and CLAUDE.md in your setup. Scores each skill against a best-practices checklist and saves findings to `~/.skill-doctor/diagnosis.json`.

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

| Check | What it catches |
|-------|----------------|
| Frontmatter completeness | Missing `allowed-tools`, `disable-model-invocation`, `context` |
| Dynamic context injection | File reads and shell commands that should use `!command` syntax |
| Argument support | Multi-mode skills without `$ARGUMENTS` routing |
| Supporting files & progressive disclosure | SKILL.md too large, missing the three-level system (frontmatter, body, linked files) |
| Skill-scoped hooks | Pre/post actions hardcoded as instructions instead of hooks |
| Overlap detection | Multiple skills triggering on the same keywords |
| CLAUDE.md audit | Reusable workflows that waste context every session |
| Agent wiring | Agents referencing missing skills, skills that should be agent-backed |
| Model override | Opus used for tasks sonnet could handle |
| Context isolation | Verbose skills that should run in `context: fork` |
| Description triggers | Vague descriptions that won't auto-invoke reliably |
| Security | XML injection in frontmatter, reserved names, hardcoded secrets |

Each check includes severity scoring (critical, warning, suggestion) and specific fix recommendations. The full checklist with examples lives in `skills/skill-doctor/references/best-practices.md`.

## How It Works

```
/skill-doctor checkup
    │
    ▼
  Discover skills ──► Find installed guides ──► Score against checklist ──► Save diagnosis.json
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

1. **Diagnose** discovers all skills (`~/.claude/skills/`, `.claude/skills/`, plugins) and any installed skill-authoring guides
2. Scores against the built-in checklist, augmented by any guides it finds (e.g. superpowers writing-skills, Anthropic best practices)
3. Findings are saved to `~/.skill-doctor/diagnosis.json`
4. **Treat** reads the diagnosis and builds upgraded skills in `~/.skill-doctor/staging-<date>-<source>/`
5. You test with `cc --plugin-dir`, then migrate (with backup) or discard
6. **Rollback** restores from `~/.skill-doctor/backups/` if anything breaks

## Integrations

skill-doctor dynamically discovers installed guides and reference materials at scan time. If you have plugins like superpowers (with its writing-skills guide and Anthropic best practices doc), skill-doctor reads them and adds any checks not already in the baseline. No configuration needed.

These optional integrations add more capabilities:

| Integration | What it adds |
|-------------|-------------|
| [Context7](https://context7.com) | Fetches latest Claude Code docs for checks that go beyond installed guides |
| [skill-creator](https://github.com/anthropics/skills) | Format validation and blind comparison evals for upgrades |

Without these, skill-doctor uses its built-in checklist plus whatever guides it finds locally.

## Requirements

Claude Code.

## License

MIT
