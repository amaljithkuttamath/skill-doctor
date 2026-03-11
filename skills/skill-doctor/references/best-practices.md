# Skill Best Practices Checklist

Use this checklist to audit each skill. Each item includes what to check, why it matters, and a bad vs good example.

---

## 1. Frontmatter Completeness

**Check:** Does the skill define `allowed-tools`, `disable-model-invocation`, and `context` where appropriate?

**Why:** Missing `allowed-tools` means the skill runs with full tool access. Missing `disable-model-invocation` means Claude can auto-trigger side-effecting skills. Missing `context` means verbose skills pollute the main conversation.

Note: `model` is covered by check 9 (Model Override). `argument-hint` is covered by check 3 (Argument Support). Description quality is covered by check 10 (Description Trigger Quality).

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

## 4. Supporting Files & Progressive Disclosure

**Check:** Is SKILL.md under 500 lines? Are templates, references, and examples in separate files? Is the skill using the three-level system effectively?

**Why:** Skills use progressive disclosure: frontmatter description (always loaded in system prompt) -> SKILL.md body (loaded when relevant) -> linked files (loaded on demand). Large SKILL.md files waste context. Supporting files load on demand. Templates give Claude a concrete target format.

**Three-level system:**
- Frontmatter description: what it does + trigger phrases (under 200 chars)
- SKILL.md body: core workflow steps (under 500 lines)
- `references/`, `templates/`, `examples/`: detailed docs, long examples, domain knowledge

**Bad:** An 800-line SKILL.md with inline templates, API docs, and examples all in one file. Or a 500-character description that includes step-by-step instructions in the frontmatter.

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

---

## 11. Description Trigger Quality

**Check:** Does the description include specific phrases users would actually say? Does it cover paraphrased variants?

**Why:** The description is how Claude decides whether to load the skill. Vague descriptions undertrigger. Missing trigger phrases mean users have to invoke manually. The description must contain BOTH what the skill does AND when to use it.

**Bad:**
```yaml
---
description: Helps with projects.
---
```

**Good:**
```yaml
---
description: Manages Linear project workflows including sprint planning, task creation, and status tracking. Use when user mentions "sprint", "Linear tasks", "project planning", or asks to "create tickets".
---
```

**Debugging:** Ask Claude "When would you use the [skill-name] skill?" It will quote the description back. Adjust based on what's missing.

---

## 12. Security

**Check:** Does the frontmatter contain XML angle brackets (`<` or `>`)? Does the skill name contain "claude" or "anthropic"?

**Why:** Frontmatter appears in Claude's system prompt. XML brackets could inject instructions. Skill names with "claude" or "anthropic" are reserved by Anthropic and will be rejected.

**Also check:**
- No executable code in YAML values (safe YAML parsing prevents this, but malformed YAML can cause upload failures)
- No hardcoded secrets, API keys, or credentials in SKILL.md or supporting files
