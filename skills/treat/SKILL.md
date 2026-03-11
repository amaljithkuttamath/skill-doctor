---
name: treat
description: Apply skill-doctor diagnosis by building upgraded skills in a staging directory for testing. Run /skill-doctor or /skill-doctor checkup first to generate a diagnosis.
disable-model-invocation: true
allowed-tools: Read, Write, Bash, Glob
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

Work through each finding and apply the suggestion. Reference the templates at `${CLAUDE_SKILL_DIR}/templates/` for structure guidance.

**Common upgrade operations:**

**Adding frontmatter fields:**
- Add `allowed-tools` listing only the tools the skill actually uses
- Add `disable-model-invocation: true` if the skill has side effects (writes files, runs commands, sends messages)
- Add `model: sonnet` if the skill does simple tasks (formatting, status checks, lookups)
- Add `argument-hint` if the skill has multiple modes

**Converting to dynamic injection:**
- Find instructions like "Read the file at X" or "Run command Y"
- Replace with `!`command`` syntax in the SKILL.md
- Only convert reads/commands that should run at load time, not conditionally

**Adding $ARGUMENTS support:**
- If the skill has if/else branches for different modes, add `argument-hint` to frontmatter
- Add a mode router section that checks `$ARGUMENTS[0]`

**Creating supporting files:**
- If SKILL.md > 500 lines, extract reference material to `references/` directory
- If the skill generates structured output, create a template in `templates/`
- Reference supporting files with `${CLAUDE_SKILL_DIR}/path`

**Adding context: fork:**
- If the skill reads many files or generates verbose output, add `context: fork` and `agent: general-purpose`
- Don't fork interactive skills that need user back-and-forth

**Fixing overlap:**
- If two skills overlap, sharpen their descriptions to trigger on distinct keywords
- Or merge them into one skill with `$ARGUMENTS` for different modes

**Extracting from CLAUDE.md:**
- If `claude_md_findings` exist, create new skill files from the extracted content
- Put them in staging alongside the upgraded skills

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
