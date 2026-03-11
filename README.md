# skill-doctor

Audit, diagnose, and upgrade your Claude Code skills and agents.

## Install

**From GitHub (recommended):**
```bash
claude plugin marketplace add amaljithkuttamath/skill-doctor
claude plugin install skill-doctor@skill-doctor-marketplace
```

**From official marketplace** (if available):
```
/plugin install skill-doctor@official-marketplace
```

**Local testing:**
```bash
cc --plugin-dir /path/to/skill-doctor
```

## Usage

### `/skill-doctor` - Diagnose

Examines your skills, agents, and CLAUDE.md. Three modes:

- `/skill-doctor` - interactive intake, asks what's bothering you
- `/skill-doctor checkup` - automated scan with report card
- `/skill-doctor consult` - guided Q&A connecting findings to your pain points

### `/skill-doctor:treat` - Fix

Builds upgraded skills in a staging directory. You test them before committing.

```bash
# Test staged upgrades in a separate session
cc --plugin-dir ~/.skill-doctor/staging-<date>-<source>/
```

When happy, say "migrate" to apply permanently (originals are backed up).

### `/skill-doctor:rollback` - Undo

Restores skills from the last backup if upgrades cause problems.

## What It Checks

- Frontmatter completeness (`allowed-tools`, `disable-model-invocation`, `context`)
- Dynamic context injection opportunities
- Argument support for multi-mode skills
- Supporting files & progressive disclosure
- Skill-scoped hooks
- Overlap between skills
- CLAUDE.md content that should be extracted to skills
- Agent-skill wiring
- Model override opportunities
- Context isolation (`context: fork`)
- Description trigger quality (specific phrases, paraphrased variants)
- Security (XML in frontmatter, reserved names, hardcoded secrets)

## How It Works

1. **Diagnose** reads all your skills/agents and scores them against a best-practices checklist
2. Findings are saved to `~/.skill-doctor/diagnosis.json`
3. **Treat** reads the diagnosis and builds upgraded skills in a staging directory under `~/.skill-doctor/`
4. You test the staged skills with `cc --plugin-dir`, then migrate or discard
5. **Rollback** restores from backup if anything breaks

## Optional Integrations

These enhance skill-doctor but aren't required:

- **Context7** - fetches latest Claude Code docs for up-to-date best practices
- **skill-creator** - adds format validation and blind comparison eval for upgrades

Without these, skill-doctor uses its built-in checklist and shows diffs instead of running evals.

## Requirements

- Claude Code

## License

MIT
