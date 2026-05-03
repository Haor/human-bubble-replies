# Human Bubble Replies TODO

Plan source: `docs/superpowers/plans/2026-05-03-human-bubble-replies-implementation.md`

## Execution mode

- Mode: inline execution
- Commit policy: commit after each self-contained step/task
- Safety: no Telegram send-path bypass; no direct Bot API implementation

## Tasks

- [ ] Task 0: Prepare implementation workspace
  - [x] Write project TODO
  - [x] Locate or clone editable OpenClaw source repository
  - [x] Create isolated branch/worktree for implementation
  - [x] Verify clean baseline or record blocker
- [x] Task 1: Add pure bubble marker parser tests
- [x] Task 2: Implement parser, stripping, and dedupe normalization
- [x] Task 3: Add block-reply emission integration tests
- [x] Task 4: Integrate parser into block-reply emission boundary
- [x] Task 5: Apply per-bubble delay in block reply pipeline
- [x] Task 6: Prevent final duplicate sends with marker-normalized comparison
- [x] Task 7: Add skill guidance after core safety is proven
- [x] Task 8: Manual Telegram validation
- [ ] Task 9: Documentation and release guardrails

## Current note

Editable source checkout: `/root/.openclaw/repos/openclaw`; implementation worktree: `/root/.openclaw/repos/openclaw/.worktrees/human-bubble-replies` on branch `feature/human-bubble-replies`. Baseline targeted test passed: `corepack pnpm test -- src/auto-reply/reply/block-reply-pipeline.test.ts` (14 tests).
