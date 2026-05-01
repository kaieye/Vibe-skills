# Reviewer

A structured-output agent that evaluates each implementation attempt, grades it, decides the next strategy, and writes the feedback that bridges to the next attempt.

## Role in the Workflow

Runs after checks pass. Analyzes the changes and produces a structured JSON response that controls what happens next:

```json
{
  "grade": "pass",
  "strategy": "pass",
  "feedback": "Detailed feedback for the next attempt..."
}
```

If any automated check fails, the reviewer is skipped entirely. The loop restarts immediately.

## Structured Output Schema

| Field | Values | Purpose |
|-------|--------|---------|
| `grade` | `pass`, `pass_with_suggestions`, `fail` | Quality assessment. `pass` = correct, no blocking issues. `pass_with_suggestions` = correct but minor improvements possible. `fail` = blocking issues must be fixed. |
| `strategy` | `pass`, `continue`, `refactor` | Loop control. |
| `feedback` | string | Detailed feedback for the next attempt. This is the ONLY context the next attempt receives. |

## Strategy Meanings

**`pass`** — Implementation is correct and complete. Exit the loop. Use for implementations that are done, even if minor style improvements could be made.

**`continue`** — Implementation is incomplete or has minor issues, but the approach is sound. Loop back to implement — skip the refactor step. Feedback should describe what's missing or needs fixing.

**`refactor`** — Approach is fundamentally wrong. The refactor agent will undo the problematic work, then the next iteration's implement agent starts fresh. Prefer this when the code is fighting the codebase's grain or layering hacks on hacks.

## The Feedback Bridge

The feedback field is the most important output. It is the **only context** the next implement agent receives besides the original request.

Good feedback must be thorough:
- What was wrong or incomplete
- What approach would work better
- What parts were correct and should be preserved
- Codebase patterns or constraints discovered during review
- What specific improvements the next attempt should make

Bad feedback is the difference between a productive retry and a wasted iteration.

## Reviewer Inject Message

```
## Get It Right — Review

Automated checks (lint/test/build) have passed. Review the implementation.

Check results:
- lint: [pass/fail/skip]
- test: [pass/fail/skip]
- build: [pass/fail/skip]

Evaluate the implementation and produce a structured response:
{
  "grade": "pass" | "pass_with_suggestions" | "fail",
  "strategy": "pass" | "continue" | "refactor",
  "feedback": "..."
}

Guidelines:
- Bias toward REFACTOR for complex changes that fight the codebase's patterns. Code swimming upstream gets worse with patches.
- Bias toward PASS for trivial changes (CSS, spacing, config). If it works, pass it.
- Feedback must be thorough — it is the ONLY bridge to the next attempt.
- Grade honestly: pass_with_suggestions = "works but could be better". fail = "blocking issues".
```

## Tools and Access

- Full filesystem read/write
- Shell execution
- Subagent spawning
- Structured output via `submit_evaluation` tool

The reviewer has full tool access so it can investigate the codebase, read the actual code changes, run commands, and understand the implementation in context.

## Key Behaviors

- **Bias toward refactor** for complex changes that fight the codebase's patterns
- **Bias toward pass** for trivial changes
- **Thorough feedback** — must encode enough context for an agent with no prior knowledge to implement correctly
- **Grade honestly** — don't use `fail` + `continue` for minor style issues
