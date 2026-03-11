---
name: treat
description: Apply skill-doctor diagnosis by building upgraded skills in a staging directory for testing. Run /skill-doctor or /skill-doctor checkup first to generate a diagnosis.
disable-model-invocation: true
allowed-tools: Read, Write, Bash, Glob, Agent
model: sonnet
---

## Load Diagnosis

1. Read `~/.skill-doctor/diagnosis.json`
2. If the file doesn't exist: "No diagnosis found. Run `/skill-doctor` or `/skill-doctor checkup` first."
3. Show a summary:
   ```
   Diagnosis from [timestamp] ([mode] mode)
   Skills examined: N
   Findings: X critical, Y warnings, Z suggestions
   ```
4. Ask: "Proceed with treatment?"

---

## Setup Staging

1. Determine staging name from the skills being upgraded:
   - All from `~/.claude/skills/` -> `personal-skills`
   - All from `.claude/skills/` in a repo -> `<repo-dir-name>-skills`
   - Mixed locations -> `mixed-skills`
   - Build directory name: `staging-<YYYYMMDD>-<source>`
   - Full path: `~/.skill-doctor/staging-<YYYYMMDD>-<source>/`

2. Check if that directory already exists:
   - If yes: "Staging directory already exists. Overwrite, rename, or abort?"
   - On overwrite: `rm -rf` the existing one
   - On rename: append `-2`, `-3`, etc.
   - On abort: stop

3. Create staging as a valid plugin directory:
   ```bash
   STAGING=~/.skill-doctor/staging-$(date +%Y%m%d)-<source>
   mkdir -p $STAGING/.claude-plugin
   mkdir -p $STAGING/skills
   ```

4. Write staging plugin.json:
   ```bash
   cat > $STAGING/.claude-plugin/plugin.json << STEOF
   {
     "name": "staging-$(date +%Y%m%d)-<source>",
     "description": "Staged skill upgrades for <source>",
     "version": "0.0.0"
   }
   STEOF
   ```

---

## Apply Upgrades

For each skill in `diagnosis.json` that has findings:

### 1. Copy original to staging

```bash
cp -r <original-skill-dir> $STAGING/skills/<skill-name>/
```

### 2. Apply fixes based on findings

For each finding in the diagnosis, apply the suggestion from the `claude-code-guide` agent's evaluation. The diagnosis contains specific fixes for each issue, use those directly.

If a finding requires understanding the correct Claude Code mechanism (e.g. converting instructions to hooks, writing scripts, setting up dynamic injection), spawn an Agent with `subagent_type: "claude-code-guide"` and ask it for the exact implementation:

> "I need to apply this fix to a Claude Code skill: [finding description and suggestion from diagnosis]. Here is the current SKILL.md: [content]. Show me the exact changes needed, including any new files (hooks, scripts, supporting files) that should be created."

Use the agent's response to make the changes. This ensures upgrades use the correct, current syntax for hooks, scripts, frontmatter fields, and other Claude Code features.

### 3. Show diff per skill

After applying fixes to a skill, show the diff:

```bash
diff -u <original-SKILL.md> $STAGING/skills/<skill-name>/SKILL.md
```

Ask: "Does this look right for [skill-name]?" before moving to the next skill.

---

## Optional: Eval with skill-creator

If skill-creator was detected (check `diagnosis.json` > `integrations.skill_creator`):

1. Run format validation on each staged skill:
   ```bash
   python3 ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/scripts/quick_validate.py $STAGING/skills/<skill-name>/SKILL.md
   ```

2. For skills with critical findings, offer blind comparison:
   "skill-creator is available. Want to run a blind comparison between the original and upgraded version of [skill-name]? This will test both versions against the same prompts and grade which performs better."

3. If user accepts, guide them through skill-creator's eval workflow for that skill.

If skill-creator is not available, skip this section entirely. Don't mention it.

---

## Test

After all upgrades are applied:

"Your upgraded skills are staged and ready for testing.

To test, open a new terminal and run:

```bash
cc --plugin-dir <staging-path>
```

Replace `<staging-path>` with the actual `$STAGING` path you created (e.g., `~/.skill-doctor/staging-20260310-personal-skills/`).

In that session, try your skills and see if they work as expected. When you're done, come back here and tell me:

- **migrate** to apply the changes permanently
- **discard** to throw them away
- Or describe what needs tweaking and I'll adjust"

---

## Migrate

When the user says "migrate":

1. Create backup directory under `~/.skill-doctor/backups/`:
   ```bash
   BACKUP_DIR=~/.skill-doctor/backups/$(date +%Y%m%d-%H%M%S)-<source>
   mkdir -p $BACKUP_DIR
   ```
   Use the same `<source>` name from the staging directory (e.g., `personal-skills`).

2. Create manifest listing what's being replaced:
   ```json
   {
     "timestamp": "<ISO 8601>",
     "skills_replaced": [
       {
         "name": "<skill-name>",
         "original_path": "<absolute path>",
         "backup_path": "<path in backup dir>"
       }
     ]
   }
   ```
   Write to `$BACKUP_DIR/manifest.json`

3. For each skill being replaced:
   ```bash
   cp -r <original-skill-dir> $BACKUP_DIR/<skill-name>/
   cp -r $STAGING/skills/<skill-name>/* <original-skill-dir>/
   ```

4. Clean up the staging directory:
   ```bash
   rm -rf $STAGING
   ```

5. Confirm:
   "Migration complete. Your original skills are backed up at `$BACKUP_DIR`.
   If anything breaks, run `/skill-doctor:rollback` to restore them."

---

## Discard

When the user says "discard":

```bash
rm -rf $STAGING
```

"Staging discarded. Your original skills are untouched."

---

## Rules

- Always show diffs before applying. Never silently modify.
- Ask confirmation per skill, not all at once.
- Don't modify skills that had no findings.
- If a skill is in a git-tracked directory, mention that the user should commit after migration.
- Backup before migrate, always.
