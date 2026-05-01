# Implementer

The workhorse of Get It Right. A general-purpose coding agent that investigates the codebase and implements the requested changes. Runs once per iteration in the retry loop.

## Role in the Workflow

Receives the original task + accumulated review feedback. Investigates the relevant code, implements the changes, runs tests, and reports honestly on what was found — including surprises and uncertainties.

Each attempt runs in a **forked thread** with `memo: false`. This means:
- Starts fresh every time — no stale context from previous attempts
- Sees: original user request + accumulated review feedback
- Does NOT see: previous implementation narratives, check logs, code diffs, reviewer analysis

## Inject Message

```
## Get It Right — Attempt N of M

Implement the requested changes. Start by investigating the relevant code,
then make your changes and run tests to verify them.

The review feedback from previous attempts (if any) is in the conversation above.
Use it to guide your implementation.

After finishing, summarize:
1. What you changed and why
2. What tests you ran and their results
3. What surprised you about the codebase
4. What you're uncertain about
```

## Context Available

- The original user request
- All previous review feedback messages (the only artifacts saved to the main thread)
- The inject message with attempt instructions

## Context NOT Available

- Previous implementation narratives
- Previous check logs
- The reviewer's internal analysis (only the saved feedback survives)
- Previous refactoring details

This is deliberate. The reviewer's feedback is a distilled, high-signal summary. The raw implementation context from previous attempts would be noise.

## Tools and Access

- Full filesystem read/write
- Shell execution (build, test, run commands)
- Spawn subagents for parallel work
- Git operations

## Key Behaviors

- **Investigate before action** — Read code, understand patterns, form a plan before making changes
- **Test-driven verification** — Run existing tests after making changes. Write new tests when appropriate
- **Subagent delegation** — Spawn specialized subagents for independent pieces of work
- **Honest reporting** — Report what actually happened, including surprises and uncertainties. The reviewer relies on honest reporting to write accurate feedback

## Summary (NOT saved to main thread)

End each attempt with a summary that helps the reviewer understand what was attempted without needing to reconstruct it from code diffs:

1. What changed and why
2. What tests ran and their results
3. What surprised about the codebase
4. What is uncertain about

The reviewer will synthesize this into feedback for the next attempt.
