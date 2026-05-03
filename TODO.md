# Human Bubble Replies TODO

Plan source: `docs/superpowers/plans/2026-05-03-human-bubble-replies-implementation.md`

## Execution mode

- Mode: inline execution
- Commit policy: commit after each self-contained step/task
- Safety: no Telegram send-path bypass; no direct Bot API implementation

## Tasks

- [ ] Task 0: Prepare implementation workspace
  - [x] Write project TODO
  - [ ] Locate or clone editable OpenClaw source repository
  - [ ] Create isolated branch/worktree for implementation
  - [ ] Verify clean baseline or record blocker
- [ ] Task 1: Add pure bubble marker parser tests
- [ ] Task 2: Implement parser, stripping, and dedupe normalization
- [ ] Task 3: Add block-reply emission integration tests
- [ ] Task 4: Integrate parser into block-reply emission boundary
- [ ] Task 5: Apply per-bubble delay in block reply pipeline
- [ ] Task 6: Prevent final duplicate sends with marker-normalized comparison
- [ ] Task 7: Add skill guidance after core safety is proven
- [ ] Task 8: Manual Telegram validation
- [ ] Task 9: Documentation and release guardrails

## Current note

The installed OpenClaw package under `/usr/lib/node_modules/openclaw` is not a git checkout and appears to contain compiled `dist/` files but no editable `src/` tree. Implementation should target an editable source checkout before touching runtime files.
