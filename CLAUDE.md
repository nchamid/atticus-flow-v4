# CLAUDE.md — McDermott Will & Schulte

## Firm-Wide Configuration | AI Solutions

**Version:** 1.0 (draft) | **Last Updated:** May 2026 | **Owner:** AI Solutions Lead
**Deploy to:** `CLAUDE.md` at the project root (loads in every Claude session)

---

> **These files apply to every Claude session at McDermott Will & Schulte.** Role-specific overlays (`.claude/rules/product/_core-requirements.md`, `.claude/rules/dev/_core-requirements.md`, `.claude/rules/design/_core-requirements.md`) layer on top and may add but not contradict these rules. Where an overlay duplicates or conflicts, the rules in this file take precedence and the overlay should be corrected.

---

## FIRM CONTEXT

McDermott Will & Schulte is an international law firm. The AI Solutions team builds Claude-powered tools for attorneys, practice groups, finance, marketing, and operations. We operate inside a regulated environment where attorney-client privilege, client confidentiality, data sensitivity, and partner accountability are non-negotiable.

---

## DATA SENSITIVITY — THE UNIVERSAL FLOOR

Every session must respect the sensitivity classification of the content it touches:

- **Privileged** — attorney-client privileged matter content. Highest protection. Never store, log, or transmit outside approved firm systems.
- **Confidential — Client** — non-privileged client matter content. Treated as Privileged for handling purposes unless explicitly downgraded.
- **Regulated** — HIPAA-covered PHI, GDPR/CCPA personal data, financial-regulated content.
- **PII** — employee, client, or third-party personal data.
- **Internal Only** — firm-internal information not intended for external eyes.
- **Public** — already publicly available; no special handling required.

**When sensitivity is unclear or ambiguous, stop and escalate to the AI Solutions Lead.** Do not infer the classification. Do not downgrade it to keep work moving. Privileged-data leakage is unrecoverable; a stalled intake is not.

---

## TONE & QUALITY STANDARDS

Apply `.claude/rules/product/standards.md` to every output. The document defines **Audience Levels A / B / C** for tone (Client-Facing, Internal/Partner-Facing, Analyst Working Documents).

**Audience Level (tone) is distinct from Solution Tier (build model).** Same numbers do not mean the same thing — never conflate them.

---

## SOLUTION TIER — QUICK REFERENCE

Every solution carries a Solution Tier classification, which determines who builds it and what approvals are required:

- **Tier 1** — Analyst-built in Claude Code. Human always present at run time. No integrations.
- **Tier 2** — Runs automatically, integrates with firm systems, or has a UI beyond Claude Code. Analyst builds the AI layer; Development wraps the infrastructure.
- **Tier 3** — Multi-system, external-client-facing, custom authentication, or requires a new MCP. Development leads from intake.

Detailed criteria live in the `.claude/rules/product/_core-requirements.md`. **When uncertain between two tiers, default to the lower one.** Re-tiering up mid-build is cheaper than discovering over-scope.

---

## UNIVERSAL GUARDRAILS

These rules apply in every session, regardless of role or solution:

- **Never present inferred information as fact.** Source every specific claim.
- **Never produce legal conclusions or recommendations.** Claude is not the attorney; present information factually and let the attorney conclude.
- **Never invent source-specific facts** (dates, dollar amounts, party names, citations). If unknown, mark null and surface the gap.
- **Never log, store, or transmit privileged or confidential content outside approved firm systems.** This includes web-based AI tools that are not on the approved list.
- **Never sign off on or send anything client-facing without partner review.** All client-bound output is draft until a partner approves.
- **Never bypass the approved-integrations list** to connect to a new firm system. Integration scoping happens at build time, owned by Development.

---

## ESCALATION

| Situation                                             | Escalate to                           |
| ----------------------------------------------------- | ------------------------------------- |
| Data sensitivity unclear or contested                 | AI Solutions Lead → IT / Legal        |
| Privileged content in an unexpected place             | AI Solutions Lead immediately         |
| Cross-tool data movement (e.g., into Claude Design)   | AI Solutions Lead → IT / Legal        |
| Solution scope materially larger than initial framing | AI Solutions Lead                     |
| Compliance or regulatory question                     | AI Solutions Lead → Compliance        |
| Production incident or health alert                   | Development on-call                   |
| Disagreement between this file and an overlay         | AI Solutions Lead — this file governs |

---

## QUALITY GATES — slice-completion + ship/undo workflow

The slice-completion gate is `/dev-review-and-remediate` — one bounded loop covering unit tests, code review, security review, and remediation. Run it after each slice, before `/dev-ship`. Direct `git commit` is blocked by a PreToolUse hook; `/dev-ship` is the only path to a commit.

| Command                  | Purpose                                                                              |
| ------------------------ | ------------------------------------------------------------------------------------ |
| `/dev-review-and-remediate`  | **Slice-completion gate.** Generates/runs unit tests, runs code review + security review, auto-fixes mechanical findings, batches architectural findings into one prompt per iteration. Writes a cache file `reviews/.last-clean-run.json`. Does NOT commit. |
| `/dev-ship`                  | **End of a piece of work.** Test gate (re-runs `/dev-review-and-remediate` if cache invalid) → commits on the slice or fix branch → merges into `dev` (the integration branch — what gets demoed) → pushes → silent worktree cleanup. No pull requests. |
| `/dev-undo`                  | **Recovery.** Reverts the most recent commit on `dev` (creates a revert commit) and pushes it. Optionally keeps the reverted slice's workspace alive for a retry. |
| `/dev-unit-test-and-remediate` | Unit-only ad-hoc (rare — usually composed by `/dev-review-and-remediate`).             |
| `/dev-code-review`           | Run a code review standalone, no remediation loop.                                   |
| `/dev-security-review`       | Run a security review standalone, no remediation loop.                               |
| `/dev-remediation`           | Fix issues found by reviews or external scanners (GitLeaks, SonarQube, Dependabot).  |

---

## GIT WORKFLOW — the user does not see git

The user knows two verbs: **ship** and **undo**. They never see branches,
worktrees, merges, pushes, or commits. All git plumbing is handled by
skills and hooks underneath. The integration branch is **`dev`** for
every project generated by this CLI; `main`/`master` are not part of the
workflow.

Read `.claude/rules/dev/git-workflow.md` once at session start for the
full policy, including the language rule (*"use the git term but pair it
with a plain-English clause on first use in a response"*) and the
failure-surface translations.

### Integration-branch guard (PreToolUse hook)

A `PreToolUse` hook (`.claude/hooks/guard-integration-branch.sh`) blocks
any Edit, Write, or NotebookEdit whose target file lives on `dev`,
`main`, or `master`. When you see a block like:

> *Refusing to mutate a file on the integration branch (dev/main/master).
> Spin up a workspace first: ...*

**Do not surface this block to the user.** They don't see worktrees.
Respond as follows:

1. Derive a short kebab-case name (2–4 words) from the user's current
   request — e.g. *"login button is broken"* → `login-button-fix`,
   *"add export to PDF"* → `export-pdf`.
2. Run `bash .claude/hooks/begin-change.sh <name>` (or
   `bash .claude/hooks/begin-change.sh --type slice <name>` when called
   from `/dev-build-application`). The script prints the new workspace's
   absolute path on stdout.
3. Re-issue the blocked Edit/Write using a path inside that workspace.
   For any git operation, use `git -C <workspace>` so you don't shift
   your working directory.
4. Continue the conversation in plain English — say something like
   *"starting work on the login button"* and proceed. Never say
   "worktree."

Files under `.claude/worktrees/**` are always allowed by the guard — the
hook's fast path lets mid-slice edits through without overhead.

---

## SESSION START — resume pending work in plain English

A `SessionStart` hook (`.claude/hooks/session-resume.sh`) runs at the
beginning of every session. It silently cleans up any slice or fix
workspaces whose branches were already merged into `dev`, then — if any
in-progress work remains — emits a `<session-resume>` block into your
context describing each remaining workspace:

```
<session-resume>
PENDING_WORK
- name: "conversations crud"
  branch: slice/conversations-crud
  worktree: .claude/worktrees/conversations-crud
  last_activity: "2 hours ago"
  dirty: true
  merged_into_dev: false
  is_current_cwd: false
</session-resume>
```

**If a `<session-resume>` block is present in your context on the first
turn of a session, surface it to the user in plain English.** Do not
quote the block, do not use the word "worktree." Use the derived `name`
field. Examples:

- One pending workspace, dirty: *"Looks like you have unfinished work on
  **conversations crud** from 2 hours ago — there are uncommitted edits
  there. Keep going, or set it aside and start something new?"*
- One pending workspace, clean and not merged: *"You have **login button**
  from 3 days ago waiting to be shipped. Pick it back up, or move on?"*
- Multiple: list them with the same plain-English framing.

If no `<session-resume>` block is present, say nothing about it — the
common case is a quiet start.

When the user picks one up: route their next mutation into that workspace
(use `git -C <worktree-path>` for git operations; for `Edit`/`Write`
calls, operate on paths inside that worktree). When they choose to set
work aside or start fresh: do not delete anything — the workspace stays
until they decide what to do with it.

---

## ROLE OVERLAYS

The following role-scoped files layer on top of this one. A given session loads this file plus the relevant overlay:

- **`.claude/rules/product/_core-requirements.md`** — Analyst intake work. _Drafted._
- **`.claude/rules/dev/_core-requirements.md`** — Development work. _Established by Dev Manager + Architect (rules set, skills library, completion gates)._
- **`.claude/rules/design/_core-requirements.md`** — Design system and UI/UX rules. _Drafted._

---

_Maintained by the AI Solutions Lead. Changes are reviewed and logged with date and rationale. Every overlay must reference, not duplicate, the rules in this file. If a rule needs to evolve, evolve it here once — not in every overlay._
