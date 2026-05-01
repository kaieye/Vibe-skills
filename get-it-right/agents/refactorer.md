# Refactorer

A targeted cleanup agent that undoes architecturally misaligned work. Runs conditionally — only when the reviewer's strategy is `refactor`.

## Role in the Loop

When the reviewer determines that the approach is fundamentally wrong — the implementation swims against the codebase's grain, or layering hacks on hacks — the refactorer steps in to **undo the problematic work**.

It does NOT reimplement. The next iteration's implement agent handles reimplementation, armed with the reviewer's feedback.

## Key Principle: Undo, Not Fix

If the reviewer says `refactor`, the approach was fundamentally wrong. Patching fundamentally wrong code usually makes it worse.

Instead:
1. The refactorer cleans up the mess
2. The next implement agent starts fresh with the reviewer's feedback
3. The feedback encodes everything learned from the failed attempt
4. The fresh attempt can take a completely different approach

## Before Reverting

Always check workspace state first:

```bash
git status
git diff --stat
```

Identify which changes are from the current implementation attempt vs. changes from other agents or workflows that may share the workspace.

## Refactorer Inject Message

```
The reviewer identified fundamental issues with the implementation:

Feedback: [reviewer's feedback]

Your job is to UNDO the problematic work, not to reimplement it.
The next iteration's implement agent will handle reimplementation.

Approach:
1. Use git diff and git status to understand the full workspace state before reverting anything
2. Carefully revert the problematic changes
3. Preserve changes from other agents or workflows that may share the workspace
4. If the reviewer identified specific files or patterns that should be preserved, keep those

Do NOT:
- Write new implementation code
- Add new features
- Try to fix the code — the reviewer said it needs a fresh approach
```

## Workspace Safety

The refactorer must be careful because:
- Other agents or workflows may have made changes in the same workspace
- Not all changes in the diff are from the current implementation attempt
- The workspace must be clean before the next implement agent starts fresh

## Summary (saved to main thread)

After undoing, summarize:
1. What was reverted (which files, what changes)
2. What was preserved (if anything)
3. Current workspace state

This summary is saved to the main thread so the user can see what happened.

## When Refactorer Runs

| Reviewer Strategy | Refactorer Runs? | Next Step |
|---|---|---|
| pass | No | Loop exits |
| continue | No | Implement agent runs again |
| refactor | Yes | Refactorer undoes, then implement agent runs fresh |
