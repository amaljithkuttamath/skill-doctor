---
name: <skill-name>
description: <When to use this skill. Be specific about triggers. Include keywords users would search for.>
# allowed-tools: Read, Glob, Grep, Bash, Write    # Restrict to only tools this skill needs
# disable-model-invocation: true                   # Set for skills with side effects (deploy, send, commit)
# model: sonnet                                    # Use for simple tasks that don't need opus
# argument-hint: [mode]                            # Add if skill has multiple modes
# context: fork                                    # Run in isolated subagent (good for scans, reports)
# agent: general-purpose                           # Required when using context: fork
# user-invocable: false                            # Set for background knowledge skills
---

# Dynamic context (inject live data at load time)
# !`relevant-command-here`

## What This Skill Does

<One paragraph explaining purpose and when it activates.>

## Process

<Numbered steps. Be specific. Include exact commands, file paths, expected outputs.>

1. Step one
2. Step two
3. Step three

## Rules

<Bullet points. Hard constraints the skill must follow.>

- Rule one
- Rule two
