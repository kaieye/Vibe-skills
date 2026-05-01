---
name: get-it-right
description: >
  A structured retry loop for AI-assisted coding in complex codebases.
  Implement → [lint, test, build] → review → [refactor?] → loop.
  Use when the user wants a feature implemented correctly in a brownfield codebase,
  LLMs keep making the same category of mistakes, or understanding the existing
  architecture IS the bottleneck. NOT for greenfield, simple isolated changes, or
  when speed beats correctness.
version: 1.0.0
author: Hermes Agent (adapted from reliant-labs/get-it-right)
license: MIT
tags: [retry-loop, brownfield, implementation, verification, refactor]
related_skills: [systematic-debugging, test-driven-development, requesting-code-review, subagent-driven-development]
---

# Get It Right

A structured retry loop for AI-assisted coding in complex codebases.

LLMs struggle with brownfield codebases — they paper-mâché on top instead of understanding existing patterns. Get It Right forces understanding through structured iteration: **implement → verify → review → refactor if needed → loop**. Each iteration's review feedback is the only bridge to the next attempt, keeping context lean and focused.

## The Loop

```
implement → [lint, test, build] → review → [refactor?] → loop back
```

- **Only review feedback** crosses iteration boundaries — no stale code, no growing context pile
- **Check gate**: if any check fails, skip review and loop back immediately
- **Refactor** only runs when the approach is fundamentally wrong — undo, don't patch

## When to Use

**Good fit:**
- Complex brownfield codebases with implicit conventions
- Tasks where LLMs keep making the same category of mistakes
- Changes that touch multiple interconnected systems
- When understanding the architecture IS the bottleneck

**Not the right fit:**
- Greenfield projects → use an agent directly
- Simple, well-isolated changes → overhead not worth it
- Well-documented codebases with clear patterns → overhead not worth it
- Speed is the priority over correctness → overhead not worth it

## The Four Steps

### Step 1: Implement

A fresh agent investigates the codebase and implements the requested changes.

Each attempt runs in a **forked thread** (no memo). It sees:
- The original user request
- All accumulated review feedback from previous iterations

It sees **NOTHING** else from previous attempts — no implementation narratives, no check logs, no code diffs.

At end of attempt, summarize (NOT saved to main thread):
1. What changed and why
2. What tests ran and their results
3. What surprised about the codebase
4. What is uncertain about

### Step 2: Checks (parallel)

After implementation, run lint, test, and build **in parallel** (each skipped if command is empty).

If **ANY check fails** (non-zero exit):
- Skip review entirely
- Loop back to implement immediately
- Failed check output is available to the next implement attempt

No point reviewing code that doesn't compile or pass tests.

### Step 3: Review

Structured evaluation after checks pass. Produces:

```json
{
  "grade": "pass" | "pass_with_suggestions" | "fail",
  "strategy": "pass" | "continue" | "refactor",
  "feedback": "detailed feedback for the next attempt"
}
```

**Strategy meanings:**
- **pass** — Implementation is correct and complete. Exit loop.
- **continue** — Incomplete but approach is sound. Loop back, skip refactor.
- **refactor** — Approach is fundamentally wrong. Refactor agent will undo the work, then next implement starts fresh.

**Bias toward refactor** for complex changes that fight the codebase's patterns.
**Bias toward pass** for trivial changes.

The **feedback is the ONLY context** the next implement agent receives. Be thorough:
- What was wrong or incomplete
- What approach would work better
- What parts were correct and should be preserved
- Codebase patterns discovered during review
- What specific improvements the next attempt should make

### Step 4: Refactor (conditional)

Only runs when strategy is `refactor`. The refactorer **undoes the problematic work** — it does NOT reimplement.

Using `git diff` and `git status` to understand full workspace state before reverting anything. Preserve changes from other agents or workflows that may share the workspace.

After undoing, summarize what was reverted and current workspace state.

The next iteration's implement agent handles reimplementation, armed with the reviewer's feedback.

## Context Management

This is the core design principle:

- **Only review feedback** is saved to the parent thread
- Implementation narratives, check details, and review analysis stay in their forks
- Each implement agent starts fresh: original request + accumulated review feedback

This prevents **context rot** — no growing pile of stale code snippets and debug logs.

After two iterations, the main thread looks like:

```
[user]      Original request
[assistant] Review (Attempt 1): continue — feedback about what to fix
[assistant] Review (Attempt 2): pass — implementation is correct
```

Clean and focused.

## Configuration

Set these before starting the loop:

| Input | Default | Description |
|-------|---------|-------------|
| `model` | flagship | LLM model for all phases |
| `max_retries` | 3 | Maximum implement→review→refactor iterations (1–10) |
| `lint_command` | (empty) | Lint command (skipped if empty) |
| `test_command` | (empty) | Test command (skipped if empty) |
| `build_command` | (empty) | Build command (skipped if empty) |
| `yield` | true | Pause after each iteration for user review |

**`max_retries`**: Default 3 gives room for meaningful iteration. For trivial changes, 1–2 is enough. For complex multi-system changes, consider 5+.

**`yield`**: When true, pauses after each iteration so you can review feedback and decide whether to continue, adjust, or abort.

**Check commands**: All optional. If empty, that check is skipped. When configured, they run in parallel after each implementation.

## Agents

| Agent | Role | Access |
|-------|------|--------|
| **Implementer** | Writes code, runs tests, makes changes | Full filesystem, shell, subagents |
| **Reviewer** | Grades implementation, decides strategy, writes feedback | Full tool access, structured output |
| **Refactorer** | Undoes problematic work before next attempt | Full filesystem, shell, subagents |

See `agents/implementer.md`, `agents/reviewer.md`, `agents/refactorer.md` for full prompts.

## When to Stop

- **pass** → exit, done
- **max_retries reached** → report what was learned across all iterations, present remaining options
- **user aborts** → exit

## Quick Reference

```
Loop: implement → checks → review → [refactor?] → loop
Only feedback crosses boundaries — no stale context
Check gate: any failure → skip review → loop immediately
Refactor = undo, not fix
```

| Strategy | Checks Pass? | Review Result | Next Step |
|----------|-------------|---------------|-----------|
| pass | yes | correct | **exit** |
| continue | yes | incomplete | implement again |
| refactor | yes | wrong approach | refactorer undoes, then implement |
| (failure) | no | skipped | implement again |
