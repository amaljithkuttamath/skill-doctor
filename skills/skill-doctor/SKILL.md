---
name: skill-doctor
description: Audit and improve your Claude Code skills and agents. Use when skills feel slow, bloated, overlapping, or you want to optimize your setup. Also use when you have no skills and want guidance on what to create.
argument-hint: [checkup|consult]
allowed-tools: Read, Glob, Grep, Bash, Write, Agent
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

### 2. Evaluate each skill with claude-code-guide

For each discovered skill, spawn an Agent with `subagent_type: "claude-code-guide"` and send it the skill's full SKILL.md content (including any supporting files in the skill directory). Ask it:

> "Here is a Claude Code skill. Review it against Claude Code's current documentation and capabilities. For each issue you find, state the severity (critical/warning/suggestion) and a specific fix.
>
> Evaluate:
> - Is the skill using the right mechanism for each workflow step? (hooks vs instructions vs scripts vs dynamic injection)
> - Are frontmatter fields correct? (allowed-tools, disable-model-invocation, context, model, description, argument-hint)
> - Will the description trigger auto-invocation reliably?
> - Is the skill structured well? (size, progressive disclosure, supporting files)
> - Anything else that doesn't match current best practices.
>
> [Full SKILL.md content here]"

The `claude-code-guide` agent knows Claude Code's current hooks system, script support, skill features, and agent capabilities. It decides what mechanism each workflow should use. Do not second-guess its evaluation with hardcoded rules.

If the agent call fails for a skill, record `"evaluation": "failed"` and move on.

### 3. Run format validation (if skill-creator available)

If skill-creator was detected in Environment:
```bash
python3 ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/scripts/quick_validate.py <path-to-SKILL.md>
```
Run on each discovered skill. Record pass/fail.

If skill-creator not available, skip this step.

### 4. Score each skill

Use the `claude-code-guide` agent's findings to score:

- **Healthy**: 0-1 findings (suggestion severity only)
- **Warning**: 2-3 findings, or any warning-severity item
- **Critical**: 4+ findings, or any critical-severity item

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
2. Ask up to 5 questions, one at a time. Each question should connect a specific finding from the `claude-code-guide` evaluation to the user's experience. Examples:
   - If the agent found instructions that should be hooks: "I see your [skill-name] has rules like 'before writing, validate X'. Does this actually get followed reliably?" (leads to hooks/scripts suggestion)
   - If the agent found missing dynamic injection: "Your [skill-name] reads [file] every time it runs. Does it feel slow to start?"
   - If the agent found overlap: "Your [skill-a] and [skill-b] both mention [keyword]. Do they ever conflict?"
   - Derive questions from the actual findings. Don't ask about things the agent didn't flag.
3. For mechanism changes (hooks, scripts, dynamic injection), ask follow-up questions to understand the user's workflow before prescribing. These add complexity and should only be suggested when the user confirms the pain.
4. After questions, present a tailored prescription connecting each finding to the user's stated pain
5. Only surface findings relevant to the user's experience. Don't dump everything.

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
    "claude_code_guide": true|false,
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
