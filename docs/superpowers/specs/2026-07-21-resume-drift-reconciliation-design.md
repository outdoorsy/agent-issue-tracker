# Design: drift reconciliation for `/resume-initiative`

**Date:** 2026-07-21
**Issue:** [#85](https://github.com/maxdimitrov/agent-issue-tracker/issues/85)
**Status:** approved

## Problem

An epic's enumerated scope drifts between the moment the `## Children`
mirror is frozen and the moment a leaf is executed: children get filed
or natively linked without a mirror update, mirror entries go dead, and
(for enumerate-the-work epics) new in-scope artifacts land in the
codebase after the batch list was written. `/resume-initiative` today
trusts the mirror blindly, so a stale next-up is reported without
warning and the operator catches the gap only by hand-recounting.

## Approaches considered

- **A (chosen) — reconcile on resume, report-only, probe opt-in.**
  Always-on mirror-vs-native diff (one `list_child_issues` call per
  node, inside the existing recursion), an opt-in operator-authored
  `## Scope probe` command for ground-truth diffing, and confirmed
  follow-up filing as the only write. Matches the issue sketch.
- **B — a separate reconcile command (extend `/tracker-doctor`).**
  Rejected: doctor is config-health, not per-epic state; the operator
  is already *at* the epic when resuming — that is where stale scope
  bites.
- **C — offer to auto-repair the mirror on confirmation.** Rejected:
  the issue's constraint is explicit — resume stays read-only except
  the confirmed follow-up filing; mirror repair remains
  `initiative-tracking`'s job (adoption/maintenance procedures).

## Design

### 1. Mirror-vs-native diff (always on, backend-generic)

Runs per epic node, inside the existing Mode 2 recursion (and Mode 1's
subtree walk), immediately after that node's `## Children` mirror is
parsed. For node `N`:

- `M` = refs parsed from `N`'s `## Children` mirror — **checked and
  unchecked lines both** (closed native children must not
  false-positive).
- `T` = refs returned by the backend's `list_child_issues({parent_ref:
  N})` — direct children, open and closed. This is the **one added
  call per node**.

Two drift categories:

1. **Unmirrored native child** (`T \ M`): the tracker links a child the
   mirror omits. Always real drift — flag it, naming ref + title +
   open/closed state.
2. **Dead mirror entry** (subset of `M \ T`): an *unchecked* mirror
   entry whose `view_issue` fetch (already performed by the existing
   child-enumeration step — no added call) returns not-found. Flag it.

**Explicitly NOT drift:** a mirror entry whose issue is live but has no
native link (`M \ T` with a successful fetch). Native linkage is
best-effort augmentation per cross-backend invariant 6 — cross-repo
children and children past a backend's native ceiling (Jira's
three-level cap) legitimately live in the mirror alone. Flagging them
would make every deep Jira tree noisy. Checked (closed) mirror entries
are not fetched (that would add N calls); dead-entry detection covers
unchecked entries only.

Remediation pointer, not action: the drift report tells the operator to
reconcile via `initiative-tracking`'s adoption procedure ("Reconcile,
tracker wins"). Resume never edits the mirror.

**Guards:** the diff runs per node inside the same recursion, so the
depth cap, cycle guard, and mixed-backend skip apply unchanged — a
skipped node gets no reconciliation. A `list_child_issues` failure
soft-warns and skips reconciliation for that node. Never crash.

### 2. `## Scope probe` (opt-in, operator-authored)

An epic node MAY declare a `## Scope probe` section in its body:

````markdown
## Scope probe
One optional prose line describing what the probe enumerates.
```sh
git ls-files 'Tests/**/*Tests.swift'
```
````

Spec:

- The **first fenced code block** under the heading holds the command;
  the language tag is advisory. Prose around it is ignored by the
  runner.
- The command runs from the **consumer repo root** (the session CWD),
  with the session's normal tool permissions — the harness's Bash
  permission layer applies; the command is shown to the operator before
  it runs. Trust model: the probe is operator-authored shell embedded
  in an issue body; only run probes on epics whose authorship you
  trust.
- stdout = the ground-truth work set, **one item per line**; blank
  lines ignored. Non-zero exit → soft-warn (`scope probe failed
  (exit N) — skipping ground-truth diff`) and skip; never crash.
- Scope: the declaring node's subtree. A sub-epic may carry its own
  probe; each node's probe reconciles its own subtree's leaves.
- Absent section → skip silently (fully backward compatible).

**Probe diff.** Baseline matching rule is deterministic: a probe item
counts as *enumerated* iff its text appears as a literal
(case-sensitive) substring in any `## Children` mirror line of the
declaring node's subtree, or in the title of any live child fetched
during enumeration. Two directions:

- **Present-but-unenumerated** (deterministic): probe items matching no
  mirror line / child title. Listed up to 20, then
  `…and N more`.
- **Enumerated-but-missing** (judgment-assisted): leaves whose
  title/mirror line clearly names a probe-domain item absent from the
  probe output (e.g. a file since deleted). The executor is an agent;
  baseline heuristic is the reverse substring check, agent judgment may
  refine. Reported under its own label, never offered a follow-up.

### 3. Follow-up offers (confirmed writes only)

When the probe surfaces present-but-unenumerated items, offer — never
auto-file — a `followup-tracking` filing per item:

- Parent block: spun out of the declaring epic node's ref.
- **Why deferred: `drift`** — a new fifth entry in the deferral-reason
  vocabulary ("surfaced by drift reconciliation — the item existed in
  ground truth but was never enumerated in the initiative's scope").
- Dedup before offering: per `followup-tracking`'s "already tracked"
  rule, search open issues for each item first; already-tracked items
  render as `already tracked by <ref>` instead of an offer. (This also
  stops re-offers on every subsequent resume.)
- The operator picks all/some/none. Declining files nothing.
- Follow-ups are NOT children (`link_sub_issue` stays reserved for the
  epic → sub-issue relationship); if the operator wants the item in the
  batch proper, they adopt it as a child via `initiative-tracking`.

Mirror-vs-native findings get **no** follow-up offer — the issue
already exists; the remediation is mirror reconciliation (pointer
only).

### 4. Surfacing

- **Mode 2:** a `Drift report` block prints **above the child tree**,
  one line per finding, prefixed with the node ref for sub-epic
  findings. A fully-consistent epic (and no probe declared) prints
  **nothing** — zero-noise acceptance.
- **Mode 1:** the per-root line gains a trailing `· ⚠ drift: N`
  annotation, where N counts mirror-vs-native findings across the
  root's subtree (the same walk that computes rolled-up leaf counts).
  Probes do NOT run in Mode 1 — arbitrary shell per root on a list
  view is wrong; the probe runs in Mode 2/3 where the operator has
  named the epic. N = 0 → no annotation.
- **Mode 3 (`--start`):** the report prints (Mode 3 runs Mode 2), and
  any follow-up offer is asked **once** before the worktree handoff;
  declining proceeds straight into the leaf.

Report shape (worked example):

```text
Drift report — #123 engine/operator split
  ⚠ #145 — "migrate FooTests" (open) — in tracker, missing from ## Children mirror
  ⚠ #131 (mirror entry) — no live issue in the tracker
  ⚠ probe: 4 items present but unenumerated:
      Tests/AccountTests.swift
      Tests/BillingTests.swift
      …and 2 more
  → mirror findings: reconcile via initiative-tracking's adoption procedure (tracker wins)
  → unenumerated items: file follow-ups? [all / pick / none]
```

## Files changed

| File | Change |
|---|---|
| `commands/resume-initiative.md` | New "Drift reconciliation" section; wire into Mode 1 (roll-up annotation), Mode 2 (report above tree), Mode 3 (offer-once before handoff); failure modes. |
| `skills/initiative-tracking/SKILL.md` | Document the `## Scope probe` block (exact spec + worked example) and the mirror-vs-linkage invariant resume reconciles against; frontmatter description mention. |
| `templates/epic-body.md` | Optional `## Scope probe` section in the canonical skeleton (like `## Parent epic`). |
| `skills/followup-tracking/SKILL.md` | `drift` origination in the deferral-reason prose. |
| `templates/followup-body.md` | `drift` as the fifth Why-deferred vocabulary entry. |
| `backends/_interface.md` | Note under `list_child_issues` that `/resume-initiative`'s drift reconciliation is a second consumer. No new operation — the contract stays eight ops. |
| `examples/workflows/resume-an-initiative.md` | Show the drift report in the Mode 2 walkthrough; variations entry. |
| `tests/fixtures/drift/` + `tests/test_drift_fixtures.py` | Three fixtures — (a) mirror missing a native child, (b) probe surfacing unenumerated items, (c) consistent no-probe epic — pinned by a minimal executable-spec parser of the documented grammar (mirror-line refs, probe block extraction, category-1 diff, forward probe diff). |
| `CHANGELOG.md` | `## [Unreleased]` entry. |

## Error handling

- `list_child_issues` fails → soft-warn, skip that node's diff.
- Probe non-zero exit / empty output → soft-warn, skip probe diff
  (empty output with exit 0 is a valid "ground truth is empty" and
  diffs normally).
- Probe section present but no fenced block → soft-warn (`## Scope
  probe declared but no fenced command block found`), skip.
- All existing traversal guards unchanged.

## Testing

`pytest -q` (existing suite + the three new fixture tests). CI's
markdown-lint / yaml-validate / backend-contract jobs unchanged and
must stay green — the contract heading set is untouched. There are no
tracked resume-initiative behavioral evals in the repo; the fixture
tests are the executable regression net, pinning the documented grammar
the same way `tests/test_doc_currency.py` pins the audit script.

## Out of scope

- Auto-repairing the mirror (stays with `initiative-tracking`).
- Running probes in Mode 1.
- A reverse-direction deterministic probe diff (judgment-assisted
  only).
- Any new backend contract operation.
