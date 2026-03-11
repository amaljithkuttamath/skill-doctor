# Skill Best Practices Checklist

Use this checklist to audit each skill. Each item includes what to check, why it matters, and a bad vs good example.

---

## 1. Frontmatter Completeness

**Check:** Does the skill define `allowed-tools`, `disable-model-invocation`, `model`, `argument-hint`, and `context` where appropriate?

**Why:** Missing frontmatter means the skill runs with full tool access, may auto-trigger when it shouldn't, and uses the default model even for simple tasks.

**Bad:**
```yaml
---
name: deploy
description: Deploy to production
---
```

**Good:**
```yaml
---
name: deploy
description: Deploy to production. Use when code is tested and ready for release.
disable-model-invocation: true
allowed-tools: Bash, Read
model: sonnet
---
```

---

## 2. Dynamic Context Injection

**Check:** Does the skill instruct Claude to read files or run commands that could instead use `!`backtick`` syntax to inject data at load time?

**Why:** Dynamic injection loads data before Claude sees the skill, saving tool calls and tokens. File reads and shell commands in the skill body cost extra round trips.

**Bad:**
```markdown
## Process
1. Read the file at ~/status.md
2. Check git status
```

**Good:**
```markdown
## Current State
!`cat ~/status.md`

## Git Status
!`git status --short`
```

---

## 3. Argument Support

**Check:** Does the skill have multiple modes or behaviors that should be selectable via `$ARGUMENTS`?

**Why:** Without arguments, users must describe what they want in natural language. Arguments make invocation precise and predictable.

**Bad:**
```markdown
## Process
Ask the user if they want a quick check or a deep review.
```

**Good:**
```yaml
---
argument-hint: [quick|deep]
---
```
```markdown
## Mode
- If `$ARGUMENTS[0]` is `quick`: run fast scan
- If `$ARGUMENTS[0]` is `deep`: run thorough review
- If empty: ask user which mode
```

---

## 4. Supporting Files

**Check:** Would templates, references, or examples in the skill directory improve output quality or keep SKILL.md under 500 lines?

**Why:** Large SKILL.md files waste context. Supporting files load on demand. Templates give Claude a concrete target format. Examples show what "good" looks like.

**Bad:** A 800-line SKILL.md with inline templates, API docs, and examples all in one file.

**Good:**
```
my-skill/
├── SKILL.md (under 500 lines)
├── templates/
│   └── output-format.md
├── references/
│   └── api-docs.md
└── examples/
    └── good-output.md
```
SKILL.md references them: "Use the template at `${CLAUDE_SKILL_DIR}/templates/output-format.md`"

---

## 5. Skill-Scoped Hooks

**Check:** Are there pre/post actions (validation, logging, notifications) that should be hooks instead of inline instructions?

**Why:** Hooks run automatically and reliably. Inline instructions depend on Claude following them. Hooks scoped to a skill only run while that skill is active.

**Bad:**
```markdown
## Rules
- Before writing any file, validate the path exists
- After every edit, log the change to audit.txt
```

**Good:**
```yaml
---
hooks:
  PreToolUse:
    - matcher: "Write"
      hooks:
        - type: command
          command: "test -d $(dirname $TOOL_INPUT_PATH)"
  PostToolUse:
    - matcher: "Edit"
      hooks:
        - type: command
          command: "echo '$TOOL_INPUT_PATH modified' >> ~/audit.txt"
---
```

---

## 6. Overlap Detection

**Check:** Do multiple skills have similar descriptions, trigger on the same keywords, or do overlapping work?

**Why:** Overlapping skills confuse Claude's auto-invocation. It may pick the wrong one, invoke both, or invoke neither. Each skill should have a unique responsibility.

**Signs of overlap:**
- Two skills with "review" in the description
- A skill and CLAUDE.md both defining the same workflow
- An agent and a skill that do the same thing in different contexts

**Fix:** Merge overlapping skills, or sharpen descriptions so they trigger on distinct keywords.

---

## 7. CLAUDE.md Audit

**Check:** Does CLAUDE.md contain reusable instructions that should be extracted into skills?

**Why:** CLAUDE.md loads every session. Reusable workflows waste context when not needed. Skills load on demand.

**Should stay in CLAUDE.md:** Project conventions, file paths, build commands, team preferences.

**Should be a skill:** Multi-step workflows, templates, procedures that apply only sometimes.

**Example:** A deploy checklist in CLAUDE.md should be a `/deploy` skill with `disable-model-invocation: true`.

---

## 8. Agent Wiring

**Check:** Do AGENTS.md agents reference the right skills? Are there agents without skills or skills that should be agent-backed?

**Why:** Agents without skills lack domain knowledge. Skills without agents may clutter the main conversation when they should run in isolation.

**Check for:**
- Agents with `skills:` field pointing to nonexistent skills
- Skills that dump lots of output (should use `context: fork` or be agent-backed)
- Multiple agents that could share a common skill

---

## 9. Model Override

**Check:** Is the skill using the default model (opus) for tasks that sonnet could handle?

**Why:** Opus is slower and more expensive. Simple tasks (formatting, file operations, lookups) don't need it. Reserve opus for complex reasoning.

**Good candidates for `model: sonnet`:**
- File scaffolding or boilerplate generation
- Status checks and reporting
- Simple transformations
- Formatting and linting

**Keep on opus:**
- Architecture decisions
- Complex debugging
- Code review requiring deep understanding

---

## 10. Context Isolation

**Check:** Would `context: fork` keep the main conversation cleaner?

**Why:** Skills that read many files, make many tool calls, or produce verbose output pollute the main context. Forking runs the skill in an isolated subagent, returning only the summary.

**Good candidates for `context: fork`:**
- Audit/scan skills that read dozens of files
- Report generation
- Bulk operations

**Bad candidates:**
- Interactive skills that need back-and-forth with the user
- Skills that modify the current working state

```yaml
---
context: fork
agent: general-purpose
---
```
