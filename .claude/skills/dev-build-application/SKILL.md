---
name: dev-build-application
description: Step 3 — implement the locked slice plan one slice at a time, in order, pausing for human confirmation between each slice. Auto-picks the first unstarted slice. Enforces grain rules in slicing.md and hard-stops at the rare review gates.
version: '0.1'
---

# /dev-build-application — Step 3: Vertical Slice Implementation

Implement the locked slice plan **one slice per iteration, in order**. After each slice is implemented (code + tests as the layer-cut plan dictates), hand off to `/dev-review-and-remediate` — the slice-completion gate that runs the tests, runs code review and security review, and remediates findings — and then `/dev-ship`, before pausing for human confirmation to continue. STOP on any non-yes answer; **hard-STOP** at the rare review gates (architecture every 5 slices, product review at spec-section completion).

This skill does **not run** tests, lint, code review, or security review — those run at slice-completion via `/dev-review-and-remediate`. Tests are still **authored** here as part of the slice deliverable.

A slice = **one user-visible capability** (something a user can describe in one sentence) — not a layer, not a single endpoint, not a single audit event.

`$ARGUMENTS` is optional:

- **Empty (common case)** → auto-pick the first slice in `slice-plan.md` without `Status: completed`.
- **Slice name** → start from that slice (must match an entry in `slice-plan.md`). Warn if there are earlier unstarted slices.

## Flags

- `--verbose` — opt-in plumbing view. When present in `$ARGUMENTS`,
  strip the flag before resolving the slice name, then print
  `[verbose] git <command and args>` on a line of its own *before* each
  git invocation this skill runs. Verbose is additive — keep the
  normal narration. See
  [rules/dev/git-workflow.md](../../rules/dev/git-workflow.md) §
  `--verbose` for the full contract.

## Session-start protocol

At the start of every session, batch-read every file you expect to need in a single message with parallel Read tool calls. Tool results land in conversation history and become part of the cached prefix from the second turn onward.

Read in this order:

1. `@/CLAUDE.md`, `@/api/CLAUDE.md`, `@/web/CLAUDE.md`, `@/database/CLAUDE.md`
2. `@/.claude/profile.json`, `@/.claude/rules/slicing.md`
3. `/docs/architecture/` (all files)
4. `/shared/` (full tree — know what exists before writing anything)
5. `slice-plan.md` — identify the starting slice
6. Paste the contract sections (not just filenames) the first slice will build against

Only then generate code. If steps 1–6 are skipped, restart the session.

After the first read, treat these files as already in context — do NOT re-read mid-session unless a `git diff` or hook message shows the file changed. Mid-session re-reads cost 10× the original cache-hit price.

Per-slice reads are limited to: the slice's row in `slice-plan.md`, the specific source files the slice modifies, and any contract section explicitly referenced by the slice.

If you find yourself reaching to re-Read any of the above, stop and reuse what's already in context.

## Steps

### 1. Guards (once per session)

- Missing `slice-plan.md` → **STOP**, run `/dev-build-architecture`.
- Missing or empty `/shared/` → **STOP**, run `/dev-build-scaffold`.
- Every slice in `slice-plan.md` already marked `Status: completed` → final report and **STOP**.
- `$ARGUMENTS` provided but doesn't match a slice → **STOP**, list available slice names.
- `$ARGUMENTS` provided that skips earlier unstarted slices → warn explicitly, ask the user to confirm intent before proceeding.

### 2. Pick the starting slice

- `$ARGUMENTS` empty → first slice in `slice-plan.md` without `Status: completed`.
- `$ARGUMENTS` set → that slice.

State the slice name + capability sentence aloud before doing anything else.

---

The remaining steps **3–10 run per slice in a loop**. Step 11 decides whether to continue.

### 3. Enter the slice workspace and confirm scope

**Enter the slice workspace.** Each slice runs in its own workspace
branched off the latest `dev` (the integration branch — what gets
demoed). Derive a kebab-case slug from the slice's name in
`slice-plan.md` (e.g. `conversations-crud`, `export-pdf`), then run:

```bash
SLICE_SLUG=<derived-slug>
SLICE_WT="$(bash .claude/hooks/begin-change.sh --type slice "$SLICE_SLUG")"
```

The script is idempotent: a fresh slug creates a new workspace branched
off the latest `dev`; an existing slug (resuming) reattaches to its
existing workspace.

For the rest of this slice (steps 3–10), **all file operations use
paths inside `$SLICE_WT`** and **all git operations use
`git -C "$SLICE_WT"`**. Do not modify files outside the workspace
until the slice ships via `/dev-ship`. The integration-branch guard
hook will block any drift back to `dev`.

In plain English to the user (per `.claude/rules/dev/git-workflow.md`):
something like *"Starting slice **`<derived display name>`** — I'm
setting up an isolated workspace for it (this is a git worktree
branched off the integration branch)."* Subsequent slice mechanics stay
silent unless something needs attention.

**Record the slice start time** as ISO-8601 with offset (e.g.
`2026-05-08T14:32:10-04:00`) — Step 10 needs it to compute duration.
Get it via `date -Iseconds` (macOS/Linux) and hold it in working memory
for this slice.

**Confirm scope.** Paste back from `slice-plan.md` for this slice:
user-capability sentence, scope (endpoints, UI, audit events), LoC
ceiling. If your understanding diverges → **STOP**, update the
architecture doc, re-run architecture review, then resume.

### 4. Plan the cut across layers

Sketch in chat (not code):

| Layer                | This slice changes                                   |
| -------------------- | ---------------------------------------------------- |
| Data / migrations    | tables, columns, indexes — or "none"                 |
| Shared types         | new types in `/shared/types/` — or "none, reusing X" |
| `/shared/` utilities | what exists vs. what to extract here                 |
| API                  | endpoints + handlers + validation + audit events     |
| Frontend             | pages, components, hooks, state, styles              |
| Tests                | unit, integration, E2E that prove the capability     |

Get user confirmation before coding.

> **Test authoring vs test execution.** Tests are authored here as part of the slice deliverable, per the per-tier requirements in `CLAUDE.md` and the per-layer rules in `.claude/rules/dev/*-testing.md`. They are **executed** at slice-completion by `/dev-review-and-remediate` (Phase 0), not mid-slice. The only continuous check during implementation is `npx tsc --noEmit`.

### 5. Implement

- Walk the Pre-Implementation Checklist tiers that apply (per CLAUDE.md). State the tier list ("Tiers: Always, Code, API endpoint, Database table") before coding.
- Build against locked contracts. Don't invent new ones mid-slice — if a contract is wrong, stop, propose the change, update the doc, then resume.
- Complete the slice fully before continuing.
- Slice must be testable in isolation (not necessarily small).
- **Edit-time check, runs continuously:** `npx tsc --noEmit` (cheap, catches type errors immediately). Do NOT run `lint`, `jest`, `playwright`, `dotnet test`, or `tSQLt` mid-slice — those are slice-completion gates owned by `/dev-review-and-remediate`. Tests are still **written** here; their execution moves to slice-completion.

### 6. /shared/ discipline

- Check `/shared/` before writing any helper. If it exists, import it.
- Future slice will clearly need it? Extract to `/shared/` now.
- Never duplicate logic already in `/shared/`.
- Update `shared-inventory.md` for every addition.

### 7. Living architecture docs

`/artifacts/docs/dev/architecture/` is the source of truth. If a request would change a contract, update the doc first, flag it, wait for approval. Keep module inventory, data flows, contracts, decisions, and `/shared/` inventory current.

### 8. Drift cap

Slice plan declared a 25% drift cap. Tempted to atomize "to make review easier" → **stop**; the grain in `slicing.md` is the contract. Slice genuinely needs more scope → stop, update `slice-plan.md`, re-run architecture review.

Never silently expand or split.

### 9. Hand off to `/dev-ship`

When the slice is implemented, **stop here**. Tell the developer in
plain English:

```
Slice <name> is implemented (code + tests). Run /ship to send it to the integration
branch — it handles the test gate, code/security review, commit, merge, and push in
one go.
```

`/dev-ship` will invoke `/dev-review-and-remediate` itself if the
test/review cache is missing or stale, then commit on the slice or fix
branch, then merge the slice into `dev` and push. Do not run reviews,
tests, lint, or any quality gates inside this skill — they're owned by
the ship pipeline so the slice loop stays focused on implementation.

### 10. Mark slice complete and document

Capture slice **start time** at the moment Step 3 begins and **end time** at the moment this step runs. Use ISO-8601 with offset (`2026-05-08T14:32:10-04:00`) and compute total duration as `HH:MM:SS`. These three timing fields are required regardless of whether a slice doc is written — record them on the `slice-plan.md` row and (when applicable) in the slice doc front matter.

Slice is not done until:

1. `- Status: completed` is appended to the slice's entry in `slice-plan.md`, along with `- Started: <ISO>`, `- Ended: <ISO>`, `- Duration: <HH:MM:SS>`.
2. The README index row exists in `artifacts/docs/dev/architecture/README.md` (always — links the slice plan row, not necessarily a slice doc).

A standalone `artifacts/docs/dev/architecture/<NN>-slice-<slug>.md` is **only** required when the slice produced something the diff and commit message can't carry:

- A non-obvious decision or tradeoff worth recording for future slices (e.g. "rejected approach X because Y", "deferred Z to slice N+3").
- A runbook step (manual migration order, feature-flag flip sequence, one-time data backfill).
- A contract change that other teams consume.

Routine slices (single-layer change, no architectural decision, no runbook, no cross-team contract) **skip the slice doc entirely** — the commit message + diff + updated `artifacts/docs/dev/architecture/` files + the timing row in `slice-plan.md` are the record.

When a slice doc is warranted, keep it to ~50 lines. Required front matter (timing is mandatory):

```markdown
---
slice: <NN>-<slug>
capability: <one-sentence user-visible capability>
spec-section: <ref>
started: <ISO-8601 with offset>
ended: <ISO-8601 with offset>
duration: <HH:MM:SS>
---
```

Body: the specific decision/runbook/contract point, links to the relevant commits. Do **not** restate layers changed or list every modified file — `git log --stat` does that.

The slice doc is a snapshot of _what happened during this slice_, not a living spec. Do not edit it when later slices change the same surface.

Then give a per-slice chat report: capability sentence, layers touched (one line), duration, whether a slice doc was written and why. (Review outcomes are not reported here — they come from `/dev-review-and-remediate` separately and live in `reviews/iteration-log/<label>.md`.)

### 11. Decide whether to continue

Check rare-gate triggers **before** asking the user:

| Gate                    | When                                                                               | Reviewer                 |
| ----------------------- | ---------------------------------------------------------------------------------- | ------------------------ |
| **Product review**      | Just completed the last slice in a spec section                                    | Product + design (human) |
| **Architecture review** | Slice swapped a mock for real impl, **or** completed slice count is divisible by 5 | Senior engineer (human)  |

- If a rare gate fires → **hard-STOP**. Print: which gate, why it fired, who needs to review, and which slice will resume the loop afterwards. Do **not** ask to continue. The user re-invokes `/dev-build-application` after the human review.
- If no rare gate **and** slices remain → ask exactly: `Slice <n> implemented. Run /ship, then continue to slice <n+1>: <name>? (yes/no)`
- If all slices are completed → final report, **STOP**.

### 12. Handle the answer

- Unambiguous yes (`yes`, `y`, `continue`, `next`) → loop back to **step 3** with the next slice.
- Anything else (`no`, `stop`, `wait`, `hmm`, free-form, ambiguous) → **STOP** cleanly. Final report:
  - Slice just completed (status, review outcome).
  - Slices remaining in `slice-plan.md` (names, in order).
  - How to resume: re-invoke `/dev-build-application` (picks up from first unstarted), or pass a specific slice name to jump.

Do not try to interpret the "no" further (don't branch into remediation, don't switch slices, don't edit `slice-plan.md`). Those are separate skills/actions.

## Do not

- Start without `/artifacts/docs/dev/architecture/` and `/shared/` in place.
- Silently invent a missing contract — surface and ask.
- Inline or duplicate anything already in `/shared/`.
- Expand a slice beyond the locked plan — update the doc first.
- Build horizontally (all of one layer, then the next).
- Run `/dev-code-review`, `/dev-security-review`, `/dev-unit-test-and-remediate`, jest, playwright, lint, dotnet test, or tSQLt inside this skill — they belong to `/dev-review-and-remediate`. The only edit-time check this skill runs is `npx tsc --noEmit`.
- Atomize a slice into sub-slices — `slicing.md` grain is the contract.
- Bend a slice around an architecture defect — fix the doc instead.
- Auto-continue past a rare review gate — always hard-STOP.
- Continue on any answer that isn't an unambiguous yes.
- Mark a slice complete without its README index row, `Status: completed`, and the start/end/duration timing fields in `slice-plan.md`. (Slice docs themselves are conditional — see Step 10.)
- Write/Edit using absolute paths rooted at the project directory in worktree sessions — those resolve to the main repo, not the cwd worktree. Use cwd-relative paths, or absolute paths prefixed with the worktree directory (visible in the session-start environment block as `Worktree path: …`).
