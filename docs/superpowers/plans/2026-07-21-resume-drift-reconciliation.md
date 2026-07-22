# Resume-Initiative Drift Reconciliation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `/resume-initiative` surfaces scope drift (mirror vs. tracker linkage, and optionally vs. an operator-declared `## Scope probe` ground truth) and offers confirmed follow-up filings for unenumerated items.

**Architecture:** Prompt-ware feature — the deliverables are markdown edits across one command doc, two skills, two templates, one backend-contract note, and one walkthrough, pinned by a small pytest "executable spec" of the documented grammar over three fixtures. Spec: `docs/superpowers/specs/2026-07-21-resume-drift-reconciliation-design.md`.

**Tech Stack:** Markdown (agent-prompt DSL), Python 3.11 + pytest (fixture pin only).

## Global Constraints

- The backend contract stays **eight operations** — `backends/_interface.md` gains prose only, never a new `` ### `op` `` heading (the `backend-contract` CI job greps those headings).
- Resume stays read-only except the explicit, confirmed follow-up filing. Never auto-edit the `## Children` mirror.
- Part 1 (mirror-vs-native diff) adds at most **one** `list_child_issues` call per node; parts 2/3 are inert without a declared `## Scope probe`.
- Grammar (must match verbatim between docs and the Task 1 parser):
  - Mirror line: `^[-*+] \[([ x])\] (\S+) — ` — bullet glyph tolerant (`-`/`*`/`+`), group 2 is the ref, separator is the em-dash ` — `. Checked AND unchecked lines both count toward the mirror set.
  - Scope probe: the **first fenced code block** after the `## Scope probe` heading holds the command; language tag advisory; stdout is one ground-truth item per line, blank lines ignored.
  - Probe matching: an item is *enumerated* iff its text is a literal case-sensitive substring of any subtree mirror line or any live child title.
- A fully-consistent epic prints **nothing** (zero-noise acceptance).
- All existing traversal guards (depth cap, cycle guard, mixed-backend skip) unchanged; every new failure path soft-warns, never crashes.
- Markdown files here use agent-prompt DSL conventions; only `README.md`, `CONTRIBUTING.md`, `CHANGELOG.md`, `examples/**` are markdownlint-gated.

---

### Task 1: Drift fixtures + executable-spec tests

**Files:**
- Create: `tests/fixtures/drift/a_epic_body.md`
- Create: `tests/fixtures/drift/a_native_children.json`
- Create: `tests/fixtures/drift/b_epic_body.md`
- Create: `tests/fixtures/drift/b_native_children.json`
- Create: `tests/fixtures/drift/b_probe_output.txt`
- Create: `tests/fixtures/drift/c_epic_body.md`
- Create: `tests/fixtures/drift/c_native_children.json`
- Test: `tests/test_drift_fixtures.py`

**Interfaces:**
- Consumes: nothing (self-contained; parser lives in the test file as executable spec).
- Produces: the pinned grammar (regex + probe extraction + diff semantics) that Tasks 2–3 must document verbatim: `MIRROR_LINE = re.compile(r"^[-*+] \[([ x])\] (\S+) — ")`, `parse_mirror_refs(body) -> dict[ref, checked]`, `parse_scope_probe(body) -> str | None`, `unmirrored_children(mirror, native) -> list[dict]`, `probe_unenumerated(items, mirror_lines, child_titles) -> list[str]`.

- [ ] **Step 1: Write the three fixture sets**

`tests/fixtures/drift/a_epic_body.md` — mirror omits live native child `#145`; closed `#201` is mirrored `[x]` (must NOT flag):

````markdown
## Goal
Migrate the Tests/ directory to Swift Testing.

## Status block
- **Phase:** Phase 1 · 1/3 sub-issues closed
- **Next up:** #131 — migrate AccountTests
- **Current branch:** none
- **Last updated:** 2026-06-30

## Children
- [x] #201 — migrate LegacyTests (Phase 0) — closed 2026-06-20
- [ ] #131 — migrate AccountTests (Phase 1)
- [ ] #132 — migrate BillingTests (Phase 1)
````

`tests/fixtures/drift/a_native_children.json`:

```json
[
  {"ref": "#201", "title": "migrate LegacyTests", "status": "closed"},
  {"ref": "#131", "title": "migrate AccountTests", "status": "open"},
  {"ref": "#132", "title": "migrate BillingTests", "status": "open"},
  {"ref": "#145", "title": "migrate FooTests", "status": "open"}
]
```

`tests/fixtures/drift/b_epic_body.md` — consistent mirror, plus a `## Scope probe`; ground truth holds four files of which two are unenumerated:

````markdown
## Goal
Migrate the Tests/ directory to Swift Testing.

## Status block
- **Phase:** Phase 1 · 0/2 sub-issues closed
- **Next up:** #131 — migrate Tests/AccountTests.swift
- **Current branch:** none
- **Last updated:** 2026-06-30

## Children
- [ ] #131 — migrate Tests/AccountTests.swift (Phase 1)
- [ ] #132 — migrate Tests/BillingTests.swift (Phase 1)

## Scope probe
Lists the in-scope test files.
```sh
git ls-files 'Tests/*Tests.swift'
```
````

`tests/fixtures/drift/b_native_children.json`:

```json
[
  {"ref": "#131", "title": "migrate Tests/AccountTests.swift", "status": "open"},
  {"ref": "#132", "title": "migrate Tests/BillingTests.swift", "status": "open"}
]
```

`tests/fixtures/drift/b_probe_output.txt`:

```text
Tests/AccountTests.swift
Tests/BillingTests.swift
Tests/CheckoutTests.swift
Tests/InventoryTests.swift
```

`tests/fixtures/drift/c_epic_body.md` — fully consistent, no probe (backward-compat / zero-noise):

````markdown
## Goal
Extract shared logging into obs/logging.

## Status block
- **Phase:** Phase 1 · 1/2 sub-issues closed
- **Next up:** #203 — api + worker cutover
- **Current branch:** none
- **Last updated:** 2026-06-30

## Children
- [x] #202 — logging format spec (Phase 0) — closed 2026-06-25
- [ ] #203 — api + worker cutover (Phase 1)
````

`tests/fixtures/drift/c_native_children.json`:

```json
[
  {"ref": "#202", "title": "logging format spec", "status": "closed"},
  {"ref": "#203", "title": "api + worker cutover", "status": "open"}
]
```

- [ ] **Step 2: Write the failing test file**

`tests/test_drift_fixtures.py` (complete file):

```python
"""Executable spec for /resume-initiative's drift-reconciliation grammar.

Pins the documented rules (commands/resume-initiative.md "Drift
reconciliation"; skills/initiative-tracking "Scope probe") the same way
test_doc_currency.py pins the audit script:

- mirror-line grammar: bullet-tolerant, checked AND unchecked lines
- scope-probe extraction: first fenced block under `## Scope probe`
- category-1 diff: native children absent from the mirror
- forward probe diff: literal case-sensitive substring matching
"""

import json
import re
from pathlib import Path

FIXTURES = Path(__file__).parent / "fixtures" / "drift"

# The documented mirror-line grammar: `-`/`*`/`+` bullet (Jira's ADF
# round-trip flips `-` to `*`), checkbox, ref (no spaces), em-dash.
MIRROR_LINE = re.compile(r"^[-*+] \[([ x])\] (\S+) — ")


def parse_mirror_refs(body: str) -> dict:
    """All `## Children` mirror refs -> checked flag (checked AND unchecked)."""
    refs = {}
    in_children = False
    for line in body.splitlines():
        if line.startswith("## "):
            in_children = line.strip() == "## Children"
            continue
        if in_children:
            m = MIRROR_LINE.match(line)
            if m:
                refs[m.group(2)] = m.group(1) == "x"
    return refs


def parse_mirror_lines(body: str) -> list:
    """Raw mirror lines, for probe substring matching."""
    lines = []
    in_children = False
    for line in body.splitlines():
        if line.startswith("## "):
            in_children = line.strip() == "## Children"
            continue
        if in_children and MIRROR_LINE.match(line):
            lines.append(line)
    return lines


def parse_scope_probe(body: str):
    """Command inside the FIRST fenced code block under `## Scope probe`.

    Returns None when the section is absent; raises ValueError when the
    section exists but holds no fenced block (documented soft-warn case).
    """
    in_probe = False
    in_fence = False
    command_lines = []
    for line in body.splitlines():
        if line.startswith("## "):
            if in_probe:
                break
            in_probe = line.strip() == "## Scope probe"
            continue
        if not in_probe:
            continue
        if line.startswith("```"):
            if in_fence:
                return "\n".join(command_lines)
            in_fence = True
            continue
        if in_fence:
            command_lines.append(line)
    if in_probe:
        raise ValueError("## Scope probe declared but no fenced command block found")
    return None


def unmirrored_children(mirror: dict, native: list) -> list:
    """Category 1: native children (open AND closed) missing from the mirror."""
    return [child for child in native if child["ref"] not in mirror]


def probe_unenumerated(items: list, mirror_lines: list, child_titles: list) -> list:
    """Forward probe diff: items matching no mirror line and no live child title."""
    haystacks = mirror_lines + child_titles
    return [
        item
        for item in items
        if item and not any(item in hay for hay in haystacks)
    ]


def load(stem: str):
    body = (FIXTURES / f"{stem}_epic_body.md").read_text(encoding="utf-8")
    native = json.loads(
        (FIXTURES / f"{stem}_native_children.json").read_text(encoding="utf-8")
    )
    return body, native


def test_fixture_a_flags_only_the_unmirrored_live_child():
    body, native = load("a")
    mirror = parse_mirror_refs(body)
    # Checked (closed) mirror lines count toward the mirror set: #201 is
    # closed AND mirrored, so it must not flag.
    assert mirror == {"#201": True, "#131": False, "#132": False}
    findings = unmirrored_children(mirror, native)
    assert [f["ref"] for f in findings] == ["#145"]
    assert findings[0]["title"] == "migrate FooTests"


def test_fixture_a_has_no_probe():
    body, _ = load("a")
    assert parse_scope_probe(body) is None


def test_fixture_b_probe_extracts_first_fenced_block():
    body, _ = load("b")
    assert parse_scope_probe(body) == "git ls-files 'Tests/*Tests.swift'"


def test_fixture_b_probe_surfaces_exactly_the_unenumerated_items():
    body, native = load("b")
    # Mirror and native agree — category 1 is clean.
    assert unmirrored_children(parse_mirror_refs(body), native) == []
    items = [
        line.strip()
        for line in (FIXTURES / "b_probe_output.txt")
        .read_text(encoding="utf-8")
        .splitlines()
        if line.strip()
    ]
    unenumerated = probe_unenumerated(
        items,
        parse_mirror_lines(body),
        [child["title"] for child in native if child["status"] == "open"],
    )
    assert unenumerated == [
        "Tests/CheckoutTests.swift",
        "Tests/InventoryTests.swift",
    ]


def test_fixture_c_consistent_epic_yields_zero_findings():
    body, native = load("c")
    assert unmirrored_children(parse_mirror_refs(body), native) == []
    assert parse_scope_probe(body) is None


def test_bullet_glyph_is_tolerated():
    # Jira's ADF round-trip flips `-` to `*`; the grammar must not care.
    for glyph in "-*+":
        line = f"{glyph} [ ] PROJ-9 — migrate widget (Phase 1)"
        m = MIRROR_LINE.match(line)
        assert m and m.group(2) == "PROJ-9"


def test_probe_section_without_fence_is_the_documented_error():
    body = "## Scope probe\nJust prose, no fence.\n"
    try:
        parse_scope_probe(body)
    except ValueError as err:
        assert "no fenced command block" in str(err)
    else:
        raise AssertionError("expected ValueError for fence-less probe section")
```

- [ ] **Step 3: Run tests — expect FAIL (fixtures missing) if written before Step 1, else PASS**

Run: `python -m pytest tests/test_drift_fixtures.py -v`
If Step 1 fixtures are not yet on disk this fails with `FileNotFoundError`; after Step 1, expected: **7 passed**.

- [ ] **Step 4: Run the full suite**

Run: `python -m pytest -q`
Expected: all existing tests + 7 new pass, 0 failures.

- [ ] **Step 5: Commit**

```bash
git add tests/fixtures/drift tests/test_drift_fixtures.py
git commit -m "test: pin drift-reconciliation grammar with three fixture sets (#85)"
```

---

### Task 2: `commands/resume-initiative.md` — the reconciliation pass

**Files:**
- Modify: `commands/resume-initiative.md`

**Interfaces:**
- Consumes: the Task 1 grammar (mirror-line regex semantics, first-fenced-block probe extraction, substring matching) — the prose here must describe exactly those rules.
- Produces: the section anchor `## Drift reconciliation (per node)` that Tasks 3 and 5 cross-reference.

- [ ] **Step 1: Insert the new top-level section** after the end of `## Tree traversal (shared rules)` (immediately before `## What you should do`):

````markdown
## Drift reconciliation (per node)

An epic's enumerated scope drifts between freeze and execution: children
get filed or natively linked without a mirror update, mirror entries go
dead, and (for enumerate-the-work epics) new in-scope artifacts land
after the batch list was written. Every Mode 2 node parse runs a
reconciliation pass and prints a **drift report** above the child tree;
a fully-consistent node prints **nothing**. The pass runs per node,
inside the same recursion, so the depth cap, cycle guard, and
mixed-backend skip apply unchanged — a skipped node gets no
reconciliation.

### Part 1 — mirror vs. native linkage (always on, backend-generic)

For each node, invoke the backend's `list_child_issues({parent_ref})`
operation — the **one added call per node** — and diff it against the
node's `## Children` mirror, where the mirror set counts **checked and
unchecked lines both** (a closed native child mirrored `[x]` is
consistent, not drift). Two finding categories:

1. **Unmirrored native child** — the tracker links a direct child
   (open or closed) that the mirror omits. Always real drift; report
   ref + title + state.
2. **Dead mirror entry** — an *unchecked* mirror entry whose
   `view_issue` fetch (already performed by child enumeration — no
   added call) returns not-found. Checked entries are not fetched, so
   dead-entry detection covers unchecked entries only.

**Explicitly NOT drift:** a mirror entry whose issue is live but has no
native link. Native linkage is best-effort augmentation per
`backends/_interface.md` invariant 6 — cross-repo children and children
past a backend's native ceiling (Jira's three-level cap) legitimately
live in the mirror alone. Do not flag them.

Mirror findings get a **remediation pointer, never an action**: point
the operator at `initiative-tracking`'s adoption procedure ("Reconcile,
tracker wins"). Resume never edits the mirror. A `list_child_issues`
failure soft-warns (`drift check skipped for <ref> — list_child_issues
failed`) and skips reconciliation for that node; never crash.

### Part 2 — `## Scope probe` ground truth (opt-in, Mode 2/3 only)

If the node's body declares a `## Scope probe` section (spec: the
`initiative-tracking` skill's "Scope probe" section), run it: the
**first fenced code block** under the heading holds an operator-authored
shell command; execute it from the consumer repo root (the session CWD)
under the session's normal tool permissions, showing the command to the
operator first. Its stdout is the ground-truth work set, one item per
line, blank lines ignored. Diff it against the declaring node's subtree:

- An item is **enumerated** iff its text appears as a literal
  case-sensitive substring in any subtree `## Children` mirror line or
  any live child title fetched during enumeration.
- **Present-but-unenumerated** (deterministic): items matching nothing.
  List up to 20, then `…and N more`.
- **Enumerated-but-missing** (judgment-assisted): leaves whose
  title/mirror line clearly names a probe-domain item absent from the
  probe output (e.g. a file since deleted). Baseline heuristic is the
  reverse substring check; report under its own label, never offer a
  follow-up for it.

Absent section → skip silently (fully backward compatible). Probe
non-zero exit → soft-warn (`scope probe failed (exit N) — skipping
ground-truth diff`) and skip. Section present but no fenced block →
soft-warn (`## Scope probe declared but no fenced command block found`)
and skip. Empty output with exit 0 is a valid empty ground truth and
diffs normally. Probes do **NOT** run in Mode 1 — arbitrary shell per
root on a list view is wrong; the probe runs only when the operator has
named the node (Mode 2/3).

### Part 3 — offer follow-ups, never auto-file

For **present-but-unenumerated** probe items only, offer a
`followup-tracking` filing per item: Parent = the declaring node's ref,
`Why deferred: drift` (surfaced by drift reconciliation — see
`templates/followup-body.md`). Before offering, apply that skill's
already-tracked rule: search open issues for each item first and render
already-tracked items as `already tracked by <ref>` instead of an offer
(this also stops re-offers on every subsequent resume). The operator
picks all/some/none; declining files nothing. Follow-ups are NOT
children — `link_sub_issue` stays reserved for the epic → sub-issue
relationship; an operator who wants the item in the batch proper adopts
it as a child via `initiative-tracking` instead.

### Report shape

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

Sub-epic findings carry the sub-epic's ref prefix. No findings and no
probe → print nothing.
````

- [ ] **Step 2: Wire Mode 1** — in Mode 1 step 4, after "walk the subtree to compute the rolled-up `<closed-leaves>/<total-leaves>` for display", add:

```markdown
During the same walk, run the drift reconciliation **Part 1** diff per
node (see "Drift reconciliation (per node)"; probes never run in
Mode 1) and count findings across the subtree.
```

And in step 5, extend the render spec: append `· ⚠ drift: <N>` to a root's line when its subtree has N > 0 mirror-vs-native findings; N = 0 renders nothing. Update the example block's first line to show ` · ⚠ drift: 2` on `#123`.

- [ ] **Step 3: Wire Mode 2** — insert a new step 4 between current steps 3 and 4:

```markdown
4. **Run drift reconciliation and print the report.** Per "Drift
   reconciliation (per node)": Part 1 (mirror vs. `list_child_issues`)
   for every node visited in step 3's recursion, Part 2 (the
   `## Scope probe`, when declared) for the named node and any sub-epic
   declaring its own, then Part 3's follow-up offer for
   present-but-unenumerated items. Print the drift report **above the
   child tree**; a fully-consistent subtree with no probe prints
   nothing.
```

Renumber current steps 4→5 and 5→6, and update the two stale cross-references: Mode 2 step 2's "(see step 4)" → "(see step 5)", and Mode 3 step 1's "(step 4 of Mode 2)" → "(step 5 of Mode 2)".

- [ ] **Step 4: Wire Mode 3** — in Mode 3 step 1, after the drill-path sentence, add:

```markdown
Mode 2's drift report prints as part of this run; when it offers
follow-up filings, ask **once** before entering the worktree —
declining proceeds straight into the leaf.
```

- [ ] **Step 5: Extend `## Failure modes`** with three bullets:

```markdown
- `list_child_issues` fails for a node during drift reconciliation → soft-warn (`drift check skipped for <ref>`) and skip that node's diff; the resume itself continues.
- `## Scope probe` command exits non-zero, or the section has no fenced code block → soft-warn and skip the ground-truth diff; never crash, never block the resume.
- Drift findings but the operator declines all follow-up offers → file nothing; the report was the deliverable. Resume never edits the `## Children` mirror (remediation is `initiative-tracking`'s adoption procedure).
```

- [ ] **Step 6: Verify and commit**

Run: `python -m pytest -q` (expected: all pass, drift tests untouched)
Run: `grep -n "step 4" commands/resume-initiative.md` — expected: no stale references to the old step numbering remain (only the new step 4 heading itself).

```bash
git add commands/resume-initiative.md
git commit -m "feat(resume-initiative): drift reconciliation pass — mirror vs. native + optional scope probe (#85)"
```

---

### Task 3: `initiative-tracking` skill + epic body template — the `## Scope probe` spec

**Files:**
- Modify: `skills/initiative-tracking/SKILL.md`
- Modify: `templates/epic-body.md`

**Interfaces:**
- Consumes: Task 2's section anchor `## Drift reconciliation (per node)` and the Task 1 grammar.
- Produces: the skill-side spec section `## Scope probe — optional ground-truth hook` that Task 2's Part 2 cites.

- [ ] **Step 1: Add the skill section** in `skills/initiative-tracking/SKILL.md`, after the `### Worked example` block that closes "Status block — exact field spec" (before `## Creating sub-issues`):

````markdown
## Scope probe — optional ground-truth hook

For enumerate-the-work initiatives (a test migration, a lint sweep —
epics whose children mirror a countable artifact set), the enumerated
batch drifts as the codebase moves: files land after the list was
frozen. An epic node MAY declare a `## Scope probe` section so
`/resume-initiative` can diff enumerated scope against ground truth on
every resume (see that command's "Drift reconciliation (per node)").

Exact block spec:

- Heading: `## Scope probe`, anywhere in the epic body (by convention
  after `## Children`).
- The **first fenced code block** under the heading holds a shell
  command; the language tag is advisory. Prose around the fence is
  ignored by the runner (use it to say what the probe enumerates).
- The command runs from the **consumer repo root** (the resuming
  session's CWD), under the session's normal tool permissions, and is
  shown to the operator before it runs.
- stdout is the ground-truth work set, **one item per line**; blank
  lines ignored. Non-zero exit → `/resume-initiative` soft-warns and
  skips the diff.
- Scope: the declaring node's own subtree. A sub-epic may declare its
  own probe.
- Omit the section entirely when the initiative has no countable ground
  truth — behaviour is then exactly as before (the probe is opt-in;
  mirror-vs-linkage reconciliation runs regardless).

**Trust model:** the probe is operator-authored shell embedded in an
issue body — anyone who can edit the body can put a command there. Only
declare/run probes on epics whose authorship you trust; the harness's
own permission layer still gates execution.

### Worked example

The iOS Swift-Testing migration epic that motivated the feature — the
batch was frozen at 9 files while the directory grew to 13:

````markdown
## Scope probe
Lists the in-scope XCTest files still to migrate or already migrated.
```sh
git ls-files 'MyAppTests/**/*Tests.swift'
```
````

On resume, the four files that landed after the freeze surface as
`present but unenumerated`, and `/resume-initiative` offers a
`followup-tracking` filing for each (`Why deferred: drift`, parent =
this epic). Items matched by a `## Children` mirror line or a live
child title (literal case-sensitive substring) count as enumerated.

### What resume reconciles even without a probe

`/resume-initiative` always diffs each node's `## Children` mirror
against the backend's `list_child_issues` — the same invariant the
adoption procedure enforces ("Reconcile, tracker wins"): the mirror is
the canonical traversal index, but the tracker's native linkage is
authoritative for *membership*. Drift (an unmirrored native child, a
dead mirror entry) is **reported, never auto-repaired** — resume stays
read-only; the repair path is this skill's adoption procedure. A live
mirror entry without a native link is NOT drift (invariant 6: native
linkage is best-effort — cross-repo children and children past a
backend's ceiling live in the mirror alone).
````

- [ ] **Step 2: Extend the frontmatter description** — in the same file's YAML `description`, after "and `/resume-initiative` walks the whole tree; see "Nested initiatives"." append:

```text
An epic may declare an optional `## Scope probe` (an operator-authored
ground-truth command) that `/resume-initiative` diffs against
enumerated scope on resume, surfacing drift — see "Scope probe".
```

- [ ] **Step 3: Add the optional section to `templates/epic-body.md`** — after the `## Children` section's closing paragraph (before `## Decision log`):

````markdown
## Scope probe
OMIT this whole section unless the initiative enumerates a countable
ground-truth work set (a test migration, a lint sweep). When present,
`/resume-initiative` runs the command below on resume and diffs its
output (one item per line) against this node's enumerated scope,
surfacing unenumerated items — see the `initiative-tracking` skill's
"Scope probe — optional ground-truth hook" for the exact spec and
trust model. The first fenced code block under this heading holds the
command; it runs from the consumer repo root.
<one line saying what the probe enumerates>
```sh
<command printing one ground-truth item per line>
```
````

- [ ] **Step 4: Verify and commit**

Run: `python -m pytest -q` (expected: all pass)
Run: `grep -c "Scope probe" skills/initiative-tracking/SKILL.md templates/epic-body.md commands/resume-initiative.md` — expected: non-zero in all three.

```bash
git add skills/initiative-tracking/SKILL.md templates/epic-body.md
git commit -m "docs(initiative-tracking): Scope probe block spec + mirror-vs-linkage reconciliation note (#85)"
```

---

### Task 4: `drift` in the follow-up deferral vocabulary

**Files:**
- Modify: `templates/followup-body.md`
- Modify: `skills/followup-tracking/SKILL.md`

**Interfaces:**
- Consumes: Task 2's Part 3 (`Why deferred: drift` offers).
- Produces: the vocabulary entry `drift` that Task 2's prose cites.

- [ ] **Step 1: Add the fifth vocabulary entry** in `templates/followup-body.md`'s `## Why deferred` list, after the `- **time**` bullet:

```markdown
- **drift** — surfaced by drift reconciliation: `/resume-initiative`'s
  scope probe found the item in ground truth but it was never
  enumerated in the initiative's scope. Parent = the epic node whose
  reconciliation surfaced it.
```

- [ ] **Step 2: Reference the origination** in `skills/followup-tracking/SKILL.md` — in "What each block unlocks", extend the **Why deferred** bullet's text with:

```markdown
A `drift` deferral marks work surfaced by `/resume-initiative`'s
drift reconciliation (an in-scope item the initiative never
enumerated) — normally workable immediately, and a candidate for
adoption as a proper child of the parent epic via
`initiative-tracking`.
```

And add one trigger to the "Strong triggers" list in `## When to file`:

```markdown
- `/resume-initiative`'s drift report surfaced an unenumerated in-scope
  item and the operator confirmed filing it.
```

- [ ] **Step 3: Verify and commit**

Run: `python -m pytest -q` (expected: all pass)
Run: `grep -n "drift" templates/followup-body.md skills/followup-tracking/SKILL.md` — expected: the new entries.

```bash
git add templates/followup-body.md skills/followup-tracking/SKILL.md
git commit -m "docs(followup-tracking): drift as a deferral-reason vocabulary entry (#85)"
```

---

### Task 5: Contract note, walkthrough, CHANGELOG (consistency sweep)

**Files:**
- Modify: `backends/_interface.md`
- Modify: `examples/workflows/resume-an-initiative.md`
- Modify: `CHANGELOG.md`

**Interfaces:**
- Consumes: Task 2's section anchor and report shape.
- Produces: nothing downstream (leaf task).

- [ ] **Step 1: Note the second consumer** in `backends/_interface.md` under `` ### `list_child_issues` `` — extend the **Purpose** paragraph's final sentence:

```markdown
It is also the operation `/resume-initiative`'s drift reconciliation
dispatches once per epic node to diff the `## Children` mirror against
the tracker's native linkage — no new capability was needed; the
existing return shape (direct children, open AND closed) is exactly
the diff's input.
```

Do NOT add or rename any `` ### `op` `` heading (backend-contract CI).

- [ ] **Step 2: Show the drift report in the walkthrough** — in `examples/workflows/resume-an-initiative.md` Mode 2 section, after the existing output block, add:

````markdown
If the mirror had drifted — say `#205` was filed and natively linked as
a child but never added to the `## Children` mirror — a drift report
prints above the children (a fully-consistent epic prints nothing):

```text
Drift report — #200 extract shared logging into obs/logging
  ⚠ #205 — "obs/logging metrics shim" (open) — in tracker, missing from ## Children mirror
  → reconcile via initiative-tracking's adoption procedure (tracker wins)
```

Epics that enumerate a countable work set can additionally declare a
`## Scope probe` (see [`initiative-tracking`](../../skills/initiative-tracking/SKILL.md)
"Scope probe — optional ground-truth hook"): a body-declared command
whose output `/resume-initiative` diffs against the enumerated scope,
offering a [`followup-tracking`](../../skills/followup-tracking/SKILL.md)
filing (`Why deferred: drift`) for each unenumerated item. Declining
files nothing — resume never writes except the confirmed filing.
````

And add to `## Variations`:

```markdown
- **Drifted mirror** — a native child missing from the `## Children` mirror (or a dead mirror entry) prints a drift report above the child tree; remediation is `initiative-tracking`'s adoption procedure, never an auto-edit. A live mirror entry without a native link is NOT flagged (cross-repo / past the backend's native ceiling is legitimate — invariant 6).
```

- [ ] **Step 3: CHANGELOG** — under `## [Unreleased]` add:

```markdown
### Added

- **`/resume-initiative` drift reconciliation (#85).** Every resumed
  node now diffs its `## Children` mirror against the backend's
  `list_child_issues` (one added call per node) and prints a drift
  report above the child tree — unmirrored native children and dead
  mirror entries surface immediately; a consistent epic prints nothing.
  Mode 1 roots gain a `· ⚠ drift: N` annotation. Epics may opt in to a
  body-declared `## Scope probe` command whose output is diffed against
  enumerated scope; unenumerated items get an offered (never automatic)
  `followup-tracking` filing with the new `drift` deferral reason.
  Resume stays read-only except that confirmed filing; mirror repair
  remains `initiative-tracking`'s adoption procedure. No new backend
  contract operation.
```

- [ ] **Step 4: Verify and commit**

Run: `python -m pytest -q` (expected: all pass)
Run the CI checks locally:
- `grep -oP '^### \x60\K[a-z_]+(?=\x60)' backends/_interface.md | sort -u` — expected: the same eight ops as before (`add_label close_issue create_issue edit_body link_sub_issue list_child_issues list_open_issues view_issue`).
- `npx markdownlint-cli2 "CHANGELOG.md" "examples/**/*.md"` (or visually match house style if npx unavailable) — expected: no new violations.

```bash
git add backends/_interface.md examples/workflows/resume-an-initiative.md CHANGELOG.md
git commit -m "docs: drift-reconciliation consistency sweep — contract note, walkthrough, changelog (#85)"
```
