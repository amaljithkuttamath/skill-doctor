# Skill Doctor - Design Spec

v1.0.0

## What

A Claude Code plugin that audits, diagnoses, and upgrades users' skills and agents. Published to the Claude Code Marketplace.

## Who

- Power users with existing skills who want to optimize them
- New users who haven't written skills yet (light guidance toward their first skill)

## Scope

- Skills (`SKILL.md` files) - personal and project
- Agents (`AGENTS.md` files)
- `CLAUDE.md` (is it doing what should be a skill or agent?)
- Skill-scoped hooks
- Overlap and gaps between skills/agents
- Skill-agent wiring

Out of scope: MCP servers, general coding help, creating business logic skills.

## Plugin Structure

```
skill-doctor/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── skill-doctor/
│   │   ├── SKILL.md              # Entry point: intake, checkup, consult
│   │   └── references/
│   │       └── best-practices.md # Hardcoded baseline checklist
│   ├── treat/
│   │   ├── SKILL.md              # Build upgraded skills in staging
│   │   └── templates/
│   │       ├── skill-template.md
│   │       └── agent-template.md
│   └── rollback/
│       └── SKILL.md              # Restore from backup
├── README.md
├── LICENSE
└── CHANGELOG.md
```

## Three Skills

### 1. `/skill-doctor` - Diagnose

Entry point. Three modes via `$ARGUMENTS[0]`:

| Mode | Trigger | Behavior |
|------|---------|----------|
| (none) | `/skill-doctor` | Ask about pain points, then run checkup or consult based on answers |
| `checkup` | `/skill-doctor checkup` | Automated scan, report |
| `consult` | `/skill-doctor consult` | Interactive Q&A, tailored suggestions |

**What it reads:**
- `~/.claude/skills/*/SKILL.md` (personal skills)
- `.claude/skills/*/SKILL.md` (project skills)
- `AGENTS.md` and `CLAUDE.md` in cwd
- Latest Claude Code docs via Context7 (`/anthropics/claude-code`)
- Falls back to `references/best-practices.md` if Context7 is unavailable

**Checkup checklist (scored per skill):**
- Has `allowed-tools`?
- Has `disable-model-invocation` where appropriate?
- Uses dynamic injection (`!`backticks``) where it could?
- Uses `$ARGUMENTS` if skill has multiple modes?
- Has supporting files (templates, references) where useful?
- Appropriate `model` override?
- `context: fork` where beneficial?
- Skill-scoped hooks where applicable?
- Overlap with other skills?
- CLAUDE.md content that should be a skill?
- Agents wired to the right skills?

**Consult flow:**
1. Silent examination (same reads as checkup)
2. Ask about pain points one at a time
3. Connect findings to user's actual frustrations
4. Tailored suggestions, not a generic checklist

**Both modes persist findings** to `~/.skill-doctor/diagnosis.json` so treat can read them. Includes: skill path, findings per skill, severity, suggested fix.

Both modes end with a summary and route to `/skill-doctor:treat`.

### 2. `/skill-doctor:treat` - Build and Test

Takes diagnosis from `~/.skill-doctor/diagnosis.json` and builds upgraded skills.

**Flow:**
1. Read `~/.skill-doctor/diagnosis.json`. If missing, tell user to run `/skill-doctor` first.
2. If `~/.skill-doctor-staging/` exists from a previous run, warn and ask to overwrite or abort.
3. Create `~/.skill-doctor-staging/` as a valid plugin directory (with `.claude-plugin/plugin.json`).
4. Copy only the diagnosed skills into staging.
5. Apply upgrades based on diagnosis.
6. Show diff of changes per skill.
7. Tell user: `cc --plugin-dir ~/.skill-doctor-staging/` to test.
8. Wait for user confirmation to migrate.
9. On "migrate":
   - Backup only the skills being replaced to `~/.skill-doctor-backup-<timestamp>/` with a manifest listing what was backed up.
   - Copy staged skills to real location.
   - Clean up staging.

### 3. `/skill-doctor:rollback` - Undo

Restores only the specific skills that were modified during the last migration.

**Flow:**
1. Find latest `~/.skill-doctor-backup-*` directory.
2. Read manifest to show exactly which skills will be restored.
3. On confirmation: restore those skills only, leaving other skills untouched.

## State

All state lives in `~/.skill-doctor/`:

| File | Purpose | Created by |
|------|---------|-----------|
| `diagnosis.json` | Findings from last diagnosis | `/skill-doctor` |
| `~/.skill-doctor-staging/` | Upgraded skills for testing | `/skill-doctor:treat` |
| `~/.skill-doctor-backup-<ts>/` | Backup of replaced skills + manifest | `/skill-doctor:treat` (migrate step) |

## Data Sources

| Source | Purpose | When | Required? |
|--------|---------|------|-----------|
| Local skill/agent files | Current state of user's setup | Always | Yes |
| `references/best-practices.md` | Baseline checklist | Always | Yes (ships with plugin) |
| Claude Code version (`claude --version`) | Tailor advice to supported features | At startup | Yes |
| Context7 `/anthropics/claude-code` | Latest best practices, new features | During checkup/consult | No (enhances baseline) |
| skill-creator `quick_validate.py` | Format validation | During checkup | No (skipped if unavailable) |
| skill-creator eval pipeline | Blind comparison of original vs upgraded | During treat | No (shows diffs only if unavailable) |

## Competitive Landscape

No tool occupies this space today.

| Tool | Format Check | Quality Diagnosis | Upgrades | Overlap Detection |
|------|:-:|:-:|:-:|:-:|
| `quick_validate.py` (Anthropic) | Yes | No | No | No |
| `skill-reviewer` (community) | Yes | Partial | No | No |
| Repello SkillCheck (security only) | No | No | No | No |
| **Skill Doctor** | Yes | Yes | Yes | Yes |

Complements Anthropic's `skill-creator` (create new) with skill-doctor (maintain and improve existing).

## Distribution

- GitHub repo: `amaljithkuttamath/skill-doctor`
- Claude Code Marketplace: `/plugin install skill-doctor@official-marketplace`
- Submit at: claude.ai/settings/plugins/submit

## User Flow

```
/skill-doctor
    -> "What's not working well?" (or checkup/consult)
    -> Reads skills, agents, CLAUDE.md
    -> Fetches latest docs from Context7 (or falls back to hardcoded)
    -> Produces diagnosis, saves to ~/.skill-doctor/diagnosis.json
    -> "Ready to fix these? Run /skill-doctor:treat"

/skill-doctor:treat
    -> Reads diagnosis.json
    -> Builds upgraded skills in ~/.skill-doctor-staging/
    -> Shows diffs per skill
    -> "Test with: cc --plugin-dir ~/.skill-doctor-staging/"
    -> User tests, comes back
    -> "Ready to migrate?" -> backs up originals, moves staging in place

/skill-doctor:rollback  (if needed)
    -> Reads backup manifest
    -> Restores only modified skills
```
