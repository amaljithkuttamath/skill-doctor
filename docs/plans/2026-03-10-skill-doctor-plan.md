# Skill Doctor Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that audits, diagnoses, and upgrades users' skills and agents. Zero hard dependencies, enhanced by optional integrations.

**Architecture:** Three skills in a plugin: diagnose (entry point with checkup/consult modes), treat (build upgrades in staging, test, migrate), rollback (restore from backup). State persisted to `~/.skill-doctor/`. Optional integrations enhance the experience but nothing is required.

**Tech Stack:** Claude Code plugin system (SKILL.md, plugin.json), Bash for file operations.

**Optional Integrations (graceful degradation):**

| Plugin | Enhancement | Without it |
|--------|------------|------------|
| Context7 | Fetches latest Claude Code docs for up-to-date best practices | Falls back to hardcoded `best-practices.md` |
| skill-creator | `quick_validate.py` for format validation, blind comparison eval for verifying upgrades | Skips validation, shows diffs only |
| claude-code-setup | Complementary (setup bootstraps, doctor maintains). Not integrated directly. | No impact |

---

## File Structure

```
skill-doctor/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── skill-doctor/
│   │   ├── SKILL.md                        # Entry point: intake, checkup, consult
│   │   └── references/
│   │       └── best-practices.md           # Hardcoded checklist fallback
│   ├── treat/
│   │   ├── SKILL.md                        # Build upgrades, stage, eval, migrate
│   │   └── templates/
│   │       ├── skill-template.md
│   │       └── agent-template.md
│   └── rollback/
│       └── SKILL.md                        # Restore from backup
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

## Chunk 1: Plugin Scaffold and References

### Task 1: Init repo and plugin manifest

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `LICENSE`

- [ ] **Step 1: Init git repo**

```bash
cd ~/Developer/skill-doctor && git init
```

- [ ] **Step 2: Create plugin.json**

```json
{
  "name": "skill-doctor",
  "description": "Audits, diagnoses, and upgrades your Claude Code skills and agents",
  "version": "1.0.0",
  "author": {
    "name": "Amaljith Kuttamath",
    "url": "https://amaljithkuttamath.github.io"
  },
  "repository": "https://github.com/amaljithkuttamath/skill-doctor",
  "license": "MIT",
  "keywords": ["skills", "audit", "diagnosis", "upgrade", "agents", "best-practices"]
}
```

- [ ] **Step 3: Create MIT LICENSE**

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json LICENSE
git commit -m "init: plugin scaffold with manifest"
```

### Task 2: Write best-practices reference

**Files:**
- Create: `skills/skill-doctor/references/best-practices.md`

- [ ] **Step 1: Write best-practices.md**

Checklist items, each with: what to check, why it matters, bad vs good example.

1. **Frontmatter completeness** - `allowed-tools`, `disable-model-invocation`, `model`, `argument-hint`, `context`
2. **Dynamic injection** - file reads that should use `!`backtick`` syntax
3. **Arguments** - multi-mode skills that should use `$ARGUMENTS`
4. **Supporting files** - templates, references, examples that would improve output
5. **Skill-scoped hooks** - pre/post actions that should be hooks
6. **Overlap detection** - similar descriptions, same trigger keywords
7. **CLAUDE.md audit** - reusable instructions that should be skills
8. **Agent wiring** - agents missing skills, skills missing agents
9. **Model override** - opus for tasks sonnet handles fine
10. **Context isolation** - `context: fork` for noisy skills

- [ ] **Step 2: Commit**

```bash
git add skills/skill-doctor/references/
git commit -m "feat: add best-practices reference checklist"
```

### Task 3: Write templates

**Files:**
- Create: `skills/treat/templates/skill-template.md`
- Create: `skills/treat/templates/agent-template.md`

- [ ] **Step 1: Write skill-template.md**

Well-structured SKILL.md with all recommended frontmatter fields and commented-out optional fields with explanations.

- [ ] **Step 2: Write agent-template.md**

Agent definition template with frontmatter (name, description, skills list).

- [ ] **Step 3: Commit**

```bash
git add skills/treat/templates/
git commit -m "feat: add skill and agent templates"
```

---

## Chunk 2: The Diagnose Skill

### Task 4: Write `/skill-doctor` SKILL.md

**Files:**
- Create: `skills/skill-doctor/SKILL.md`

- [ ] **Step 1: Write frontmatter**

```yaml
---
name: skill-doctor
description: Audit and improve your Claude Code skills and agents. Use when skills feel slow, bloated, or you want to optimize your setup.
argument-hint: [checkup|consult]
allowed-tools: Read, Glob, Grep, Bash, Write
---
```

- [ ] **Step 2: Write optional integration detection**

At skill start, detect what's available (don't require anything):
```markdown
## Environment Detection
!`claude --version 2>/dev/null || echo "unknown"`
!`ls ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/ 2>/dev/null && echo "skill-creator: yes" || echo "skill-creator: no"`
```
Also attempt Context7 `resolve-library-id` during examination. If it fails, use hardcoded checklist only.

Use Claude Code version to tailor advice. Don't suggest features the user's version doesn't support.

Log which integrations are active. If skill-creator available, mention it. If not, proceed silently.

- [ ] **Step 3: Write mode router**

Based on `$ARGUMENTS[0]`: empty -> intake, `checkup` -> automated scan, `consult` -> interactive Q&A.

- [ ] **Step 4: Write intake flow**

No arguments: ask what's bothering them with concrete options, then route to checkup or consult.

- [ ] **Step 5: Write examination step (shared)**

Read all skills (personal + project), AGENTS.md, CLAUDE.md. If Context7 available: fetch latest from `/anthropics/claude-code` (query for skills best practices). Always load `${CLAUDE_SKILL_DIR}/references/best-practices.md` as baseline. If skill-creator available: run `quick_validate.py` on each skill for format validation. Score each skill against the quality checklist.

- [ ] **Step 6: Write checkup output**

Per-skill report card (name, score, top issues). Summary counts. Top 3 recommendations. Route to treat.

- [ ] **Step 7: Write consult flow**

Silent examination, then ask pain-point questions one at a time (max 5). Connect findings to stated frustrations. Build tailored prescription. Route to treat.

- [ ] **Step 8: Write diagnosis persistence**

Save to `~/.skill-doctor/diagnosis.json`: timestamp, mode, skills examined (path, name, score, findings with severity and suggestions), claude_md_findings, agent_findings, overlap_findings.

- [ ] **Step 9: Commit**

```bash
git add skills/skill-doctor/SKILL.md
git commit -m "feat: add diagnose skill with checkup and consult modes"
```

---

## Chunk 3: The Treat Skill

### Task 5: Write `/skill-doctor:treat` SKILL.md

**Files:**
- Create: `skills/treat/SKILL.md`

- [ ] **Step 1: Write frontmatter**

```yaml
---
name: treat
description: Apply skill-doctor diagnosis by building upgraded skills in staging for testing. Run /skill-doctor first.
disable-model-invocation: true
allowed-tools: Read, Write, Bash, Glob
---
```

- [ ] **Step 2: Write diagnosis loading**

Read `~/.skill-doctor/diagnosis.json`. If missing, tell user to run `/skill-doctor` first. Show summary, ask to confirm.

- [ ] **Step 3: Write staging setup**

Check for existing staging (warn if exists). Create `~/.skill-doctor-staging/` with valid plugin structure (`.claude-plugin/plugin.json`, `skills/`).

- [ ] **Step 4: Write upgrade application**

For each diagnosed skill: copy to staging, apply fixes, use templates as reference. Show diff per skill. Ask user to confirm each.

Key operations: add frontmatter fields, convert reads to dynamic injection, add `$ARGUMENTS`, create supporting files, add hooks, split overlapping skills, extract CLAUDE.md content.

- [ ] **Step 5: Write optional eval integration**

After upgrades are built, if skill-creator is available:
1. Run `quick_validate.py` on each staged skill
2. Offer to run blind comparison (original vs upgraded) for critical skills
3. Show eval results before test instructions

If skill-creator is not available: show diffs only, skip to test instructions.

- [ ] **Step 6: Write test and migrate flow**

Tell user to test with `cc --plugin-dir ~/.skill-doctor-staging/`. Wait for feedback. On "migrate": backup originals (only modified skills) to `~/.skill-doctor-backup-<timestamp>/` with manifest.json, copy staged to real location, clean up staging.

- [ ] **Step 7: Commit**

```bash
git add skills/treat/SKILL.md
git commit -m "feat: add treat skill with eval integration and migration"
```

---

## Chunk 4: Rollback, README, Publish

### Task 6: Write `/skill-doctor:rollback` SKILL.md

**Files:**
- Create: `skills/rollback/SKILL.md`

- [ ] **Step 1: Write rollback skill**

```yaml
---
name: rollback
description: Restore skills from the last skill-doctor backup. Use if migrated skills cause problems.
disable-model-invocation: true
allowed-tools: Read, Bash, Glob
---
```

Find latest backup, read manifest, show what will be restored, confirm, restore only modified skills.

- [ ] **Step 2: Commit**

```bash
git add skills/rollback/SKILL.md
git commit -m "feat: add rollback skill"
```

### Task 7: README and CHANGELOG

**Files:**
- Create: `README.md`
- Create: `CHANGELOG.md`

- [ ] **Step 1: Write README**

Sections: What, Install, Usage (three commands), How it works (5 bullets), What it checks (brief checklist), Optional integrations (skill-creator for eval, Context7 for latest docs), Requirements (Claude Code only), License.

Under 100 lines.

- [ ] **Step 2: Write CHANGELOG**

v1.0.0 entry with initial features list.

- [ ] **Step 3: Commit**

```bash
git add README.md CHANGELOG.md
git commit -m "docs: add README and CHANGELOG"
```

### Task 8: Local testing

- [ ] **Step 1: Validate plugin**

```bash
claude plugin validate ~/Developer/skill-doctor
```

- [ ] **Step 2: Test skill discovery**

```bash
cc --plugin-dir ~/Developer/skill-doctor
```

Verify all three skills are available.

- [ ] **Step 3: Test checkup against own skills**
- [ ] **Step 4: Test consult flow**
- [ ] **Step 5: Test treat with staging and diffs**
- [ ] **Step 6: Test migrate and rollback**
- [ ] **Step 7: Fix issues, commit**

### Task 9: Publish

- [ ] **Step 1: Create GitHub repo**

```bash
gh repo create amaljithkuttamath/skill-doctor --public --description "Audit, diagnose, and upgrade your Claude Code skills and agents" --source ~/Developer/skill-doctor
```

- [ ] **Step 2: Push and tag**

```bash
git push -u origin main
git tag v1.0.0
git push origin v1.0.0
```

- [ ] **Step 3: Submit to Claude Code Marketplace**

Submit at claude.ai/settings/plugins/submit

- [ ] **Step 4: Verify marketplace install works**
