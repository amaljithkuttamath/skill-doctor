---
name: skill-doctor
description: Audit and improve your Claude Code skills and agents. Use when skills feel slow, bloated, overlapping, or you want to optimize your setup. Also use when you have no skills and want guidance on what to create.
argument-hint: [checkup|consult]
allowed-tools: Read, Glob, Grep, Bash, Write
---

## Environment

- Claude Code: !`claude --version 2>/dev/null || echo "unknown"`
- skill-creator: !`ls ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/SKILL.md 2>/dev/null && echo "available" || echo "not installed"`

## Mode

Based on `$ARGUMENTS[0]`:
- Empty: run **Intake** flow
- `checkup`: run **Checkup** flow
- `consult`: run **Consult** flow

---

## Intake Flow (no arguments)

Ask the user:

"What's going on with your skills setup? Pick what resonates:

1. My skills feel slow or use too many tokens
2. I'm not sure if I'm using skills effectively
3. My skills and agents overlap or conflict
4. I just want a health check
5. I don't have skills yet, not sure where to start
6. Something else"

**Routing:**
- Options 1, 2, 3: route to **Consult** (needs conversation to understand the pain)
- Option 4: route to **Checkup** (automated scan is fine)
- Option 5: run the examination, then suggest 2-3 starter skills based on their workflow. Don't build a full system, just point them in the right direction.
- Option 6: ask a follow-up, then route

---

## Examination (shared by Checkup and Consult)

This step is always run first, silently for Consult, with output for Checkup.

### 1. Discover skills and agents

```
# Personal skills
~/.claude/skills/*/SKILL.md

# Project skills (cwd)
.claude/skills/*/SKILL.md

# CLAUDE.md and AGENTS.md in cwd
./CLAUDE.md
./AGENTS.md

# Also check for plugin skills
~/.claude/plugins/*/skills/*/SKILL.md
```

Use Glob to find all matching files. Read each one.

### 2. Fetch latest best practices

Try Context7 first:
1. Use `resolve-library-id` tool with `libraryName: "claude code"`, `query: "skills SKILL.md frontmatter best practices"`
2. If it resolves, use `query-docs` with the library ID to fetch latest skill docs
3. Note any new features or patterns not in the hardcoded checklist

If Context7 is unavailable (tool doesn't exist or fails), proceed with hardcoded checklist only.

Always load the baseline: read `${CLAUDE_SKILL_DIR}/references/best-practices.md`

### 3. Run format validation (if skill-creator available)

If skill-creator was detected in Environment:
```bash
python3 ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/scripts/quick_validate.py <path-to-SKILL.md>
```
Run on each discovered skill. Record pass/fail.

If skill-creator not available, skip this step.

### 4. Score each skill

For each skill, check against the best-practices checklist (13 items). Score:

- **Healthy**: 0-1 findings (suggestion severity only)
- **Warning**: 2-3 findings, or any warning-severity item
- **Critical**: 4+ findings, or any critical-severity item

**Severity guide:**
- **Critical**: `allowed-tools` missing on a skill that writes files. `disable-model-invocation` missing on a skill with side effects. Major overlap with another skill.
- **Warning**: No dynamic injection where it would help. No `$ARGUMENTS` on a multi-mode skill. Missing supporting files. Wrong model override. Description missing trigger phrases. Poor progressive disclosure (too much in frontmatter or everything in SKILL.md body).
- **Suggestion**: Could benefit from `context: fork`. Could add hooks. Could use a template. Security check notes (XML brackets, reserved names, hardcoded secrets).

### 5. Check cross-cutting concerns

- **Overlap**: Compare skill descriptions pairwise. Flag if two skills share 3+ keywords or cover similar workflows.
- **CLAUDE.md bloat**: If CLAUDE.md > 200 lines, check for multi-step workflows that should be skills.
- **Agent gaps**: If AGENTS.md exists, check that referenced skills exist and vice versa.

---

## Checkup Flow

After examination, output a report:

```
## Skill Health Report

| Skill | Location | Score | Top Issue |
|-------|----------|-------|-----------|
| ... | ... | Healthy/Warning/Critical | ... |

**Summary:** N skills examined. X healthy, Y warnings, Z critical.

### Top Recommendations

1. [Most impactful fix]
2. [Second most impactful]
3. [Third most impactful]
```

If format validation was run, include results:
```
### Format Validation
- skill-a: PASS
- skill-b: FAIL (missing description)
```

End with: "Run `/skill-doctor:treat` to apply these fixes."

Save diagnosis (see Persistence section below).

---

## Consult Flow

After silent examination:

1. Start with the user's stated pain point (from intake, or ask if they came here directly)
2. Ask up to 5 questions, one at a time. Connect each question to a finding from the examination:
   - "I see your [skill-name] reads [file] every time it runs. Does it feel slow to start?" (leads to dynamic injection suggestion)
   - "Your [skill-a] and [skill-b] both mention [keyword] in their descriptions. Do they ever conflict?" (leads to overlap fix)
   - "Nothing in your setup uses argument modes. Do you ever want different behavior from the same skill?" (leads to $ARGUMENTS suggestion)
   - "Are there things you always want to happen before/after a skill runs? Like validation, logging, or notifications?" (leads to hooks suggestion)
   - "Do any of your skills produce a lot of output that clutters the conversation?" (leads to context: fork suggestion)
   - "Do your skills load automatically when you expect them to, or do you find yourself typing the slash command every time?" (leads to description trigger quality)
   - "Is your SKILL.md getting long? Over 300 lines?" (leads to progressive disclosure, move detail to references/)
3. For hooks specifically, ask follow-up questions to understand the workflow:
   - What should trigger the hook? (before a tool runs? after? on a specific tool?)
   - What should the hook do? (validate, log, notify, block?)
   - Should it only run during a specific skill, or always?
   - Don't prescribe hooks unless the user describes a concrete pain. Hooks add complexity.
4. After questions, present a tailored prescription connecting each finding to the user's stated pain
5. Don't dump all 10 checklist items. Only surface what's relevant to their experience.

End with: "Run `/skill-doctor:treat` to apply these fixes."

Save diagnosis (see Persistence section below).

---

## Persistence

After both Checkup and Consult, save findings:

```bash
mkdir -p ~/.skill-doctor
```

Write to `~/.skill-doctor/diagnosis.json`:

```json
{
  "timestamp": "<ISO 8601>",
  "claude_code_version": "<from environment detection>",
  "mode": "checkup|consult",
  "integrations": {
    "context7": true|false,
    "skill_creator": true|false
  },
  "skills_examined": [
    {
      "path": "<absolute path to SKILL.md>",
      "name": "<skill name from frontmatter>",
      "location": "personal|project|plugin",
      "score": "healthy|warning|critical",
      "format_validation": "pass|fail|skipped",
      "findings": [
        {
          "check": "<checklist item name>",
          "severity": "critical|warning|suggestion",
          "description": "<what's wrong>",
          "suggestion": "<what to do about it>"
        }
      ]
    }
  ],
  "claude_md_findings": [
    {
      "description": "<what should be extracted>",
      "suggested_skill_name": "<name for the new skill>"
    }
  ],
  "agent_findings": [
    {
      "description": "<what's wrong with agent setup>"
    }
  ],
  "overlap_findings": [
    {
      "skills": ["<skill-a>", "<skill-b>"],
      "description": "<how they overlap>",
      "suggestion": "<merge or sharpen>"
    }
  ]
}
```

---

## Rules

- Never modify skills directly. Diagnosis only. Modifications happen in `/skill-doctor:treat`.
- One question at a time in consult mode. Don't overwhelm.
- If no skills found, don't just say "nothing to audit." Help the user understand what skills could do for them based on their CLAUDE.md and workflow.
- Tailor advice to the detected Claude Code version. Don't suggest features that may not be supported.
- Be direct about findings. Don't soften critical issues.
