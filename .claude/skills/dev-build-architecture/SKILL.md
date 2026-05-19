---
name: dev-build-architecture
description: Step 1 — produce architecture artifacts under /artifacts/docs/dev/architecture and /shared/types from a requirements doc. No code. Stops at the architecture review gate.
version: '0.1'
---

# /dev-build-architecture — Step 1: Architecture First

Lock the contracts every later step builds against. **No implementation code.**

`$ARGUMENTS` must be a requirements doc (path, attachment, or pasted). It is the authoritative source — every artifact must trace back to it.

## Flags

- `--verbose` — opt-in plumbing view. When present in `$ARGUMENTS`,
  strip the flag (it is not part of the requirements doc) before
  parsing the rest, then print `[verbose] git <command and args>` on a
  line of its own *before* each git invocation this skill runs.
  Verbose is additive — keep the normal narration. See
  [rules/dev/git-workflow.md](../../rules/dev/git-workflow.md) §
  `--verbose` for the full contract.

## Session-start protocol

At the start of every session, batch-read every file you expect to need in a single message with parallel Read tool calls. Tool results land in conversation history and become part of the cached prefix from the second turn onward.

Read in this order:

1. `@/CLAUDE.md`, `@/api/CLAUDE.md`, `@/web/CLAUDE.md`, `@/database/CLAUDE.md`
2. `@/.claude/profile.json`, `@/.claude/rules/slicing.md`

If steps 1–2 are skipped, restart the session.

After the first read, treat these files as already in context — do NOT re-read mid-session unless a `git diff` or hook message shows the file changed. Mid-session re-reads cost 10× the original cache-hit price.

If you find yourself reaching to re-Read any of the above, stop and reuse what's already in context.

## Steps

### 1. Confirm inputs

- If `$ARGUMENTS` is empty, ask for the doc and **STOP**.
- Read it in full before writing any artifact.
- If `/artifacts/docs/dev/architecture/` already has files: list them, ask whether to extend, replace, or abort. Never silently overwrite.

### 2. Produce the seven artifacts

Markdown under `/artifacts/docs/dev/architecture/`, types under `/shared/types/`. Nothing else gets written this step.

| #   | Artifact                                              | File                                           |
| --- | ----------------------------------------------------- | ---------------------------------------------- |
| 1   | Data model — entities, relationships, constraints     | `data-model.md`                                |
| 2   | API contracts — endpoints, request/response, errors   | `api-contracts.md`                             |
| 3   | Module boundaries — what each module owns and exposes | `module-boundaries.md`                         |
| 4   | Shared types — vocabulary all layers speak            | `/shared/types/*.ts` + `shared-types.md` index |
| 5   | Dependency graph — module → module, must be acyclic   | `dependency-graph.md`                          |
| 6   | Shared components inventory (see below)               | `shared-inventory.md`                          |
| 7   | Slice plan (see below)                                | `slice-plan.md`                                |

### 3. Shared inventory (artifact 6)

List every cross-cutting utility (errors, validation, logging, formatters), shared UI primitive (forms, modals, tables, layouts), and infra helper (HTTP client, retry, auth guards, middleware). Per entry:

```
### <name>
- Interface: <signature or short prose>
- Location: /shared/<subdir>/<name>
- Consumers: <slices that will use it>
```

### 4. Slice plan (artifact 7)

Read `slicing.md` first. The plan must declare:

- **Target slice count** — typically 1-3× the spec's user-capability count. Justify if outside.
- **Reviewable LoC ceiling** — typically 5,000-8,000. Lower for security/regulated, higher for greenfield CRUD.
- **Per-slice entry**:

  ```
  ### Slice <n>: <name>
  - Spec section: <…>
  - User capability: "user can do X"
  - Scope: endpoints <…>, UI <…>, audit events <…>
  - Estimated LoC: <≤ ceiling>
  ```

- **Drift cap** — written verbatim: "slice count cannot grow by more than 25% during Step 3 without an architecture-doc update and a re-review."

This plan is the locked contract for Step 3 review cadence.

### 5. Cross-check before stopping

- Every spec requirement maps to at least one slice or decision.
- Dependency graph has no cycles (walk it).
- Every spec requirement maps to ≥1 slice or decision.
- Slice count, ceiling, and drift cap are all on the page.

Fix any failure — do not paper over.

### 6. Report and stop

Report: files written (with paths), slice count + ceiling, and any assumptions you made (call them out explicitly). Then **STOP** for architecture review by a human. Do not start scaffolding.

## Do not

- Write implementation code (controllers, components, migrations).
- Modify locked signatures from a prior run without updating the doc and flagging the change.
- Invent contracts that don't trace to the requirements doc.
- Skip the slice plan or shared inventory — both are required.
- Pick a side when the requirements doc contradicts itself — surface the conflict and ask.
- Produce contracts for subsystems excluded by `.claude/profile.json`.
