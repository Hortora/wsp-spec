# Staleness Hybrid Enforcement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement a five-layer staleness enforcement system (A + selective B + C) so garden entries that age past their `staleness_threshold` are visibly flagged, reviewed opportunistically at session end, and systematically swept during dedicated harvest sessions.

**Architecture:** S2 annotation runs at SEARCH time (zero friction, always-on); S6 `verified_on` and S3 soft rationale enrichment happen at CAPTURE time; S1 domain-filtered review runs at the end of SWEEP; S5 `harvest REVIEW` is the systematic backstop triggered by `validate_garden.py --freshness`. All state is stored in YAML frontmatter (`last_reviewed`, `verified_on`) and GARDEN.md (`Last staleness review`).

**Tech Stack:** Python 3.11+, markdown (YAML frontmatter), soredium validate_garden.py, forage/harvest Claude skill files.

---

## File Map

**Files to modify in soredium repo:**
- `soredium/scripts/validate_garden.py` — add `--freshness` early-exit flag
- `soredium/tests/test_validate_garden.py` — add `TestFreshnessFlag` class
- `soredium/forage/submission-formats.md` — add `verified_on`, `last_reviewed` optional fields
- `soredium/forage/SKILL.md` — three changes: SEARCH (S2), CAPTURE (S6+S3), SWEEP (S1)
- `soredium/harvest/SKILL.md` — add REVIEW mode, update description and success criteria

**Files to modify in live garden:**
- `~/.hortora/garden/GARDEN.md` — add `Last staleness review: never` metadata field

---

## Task 1: Add Staleness Tracking Field to Live GARDEN.md

**Files:**
- Modify: `~/.hortora/garden/GARDEN.md` (live garden — not spec repo)

This defines the new metadata field before any code reads or writes it.

- [ ] **Step 1: Read current GARDEN.md header**

```bash
git -C ~/.hortora/garden show HEAD:GARDEN.md | head -6
```

Expected output:
```
**Last legacy ID:** GE-0180
**Last full DEDUPE sweep:** 2026-04-12
**Entries merged since last sweep:** 0
**Drift threshold:** 10
```

- [ ] **Step 2: Add the staleness review field**

Edit `~/.hortora/garden/GARDEN.md` — insert one line after `**Drift threshold:** 10`:

```
**Last staleness review:** never
```

New header block should be:
```
**Last legacy ID:** GE-0180
**Last full DEDUPE sweep:** 2026-04-12
**Entries merged since last sweep:** 0
**Drift threshold:** 10
**Last staleness review:** never
```

- [ ] **Step 3: Commit**

```bash
git -C ~/.hortora/garden add GARDEN.md
git -C ~/.hortora/garden commit -m "meta: add Last staleness review field to GARDEN.md header"
```

Expected: commit succeeds, `git show HEAD --stat` shows `GARDEN.md` changed.

---

## Task 2: Document Optional Fields in submission-formats.md

**Files:**
- Modify: `soredium/forage/submission-formats.md`

Adds `verified_on` and `last_reviewed` as documented optional YAML fields. Does not require changes to `validate_pr.py` — fields are optional and the validator ignores unknown keys.

- [ ] **Step 1: Add an Optional Frontmatter Fields section**

In `soredium/forage/submission-formats.md`, insert a new section immediately after the `## File Naming` section (before `## Gotcha Template`):

```markdown
## Optional Frontmatter Fields

These fields can be added to any entry type. The validator accepts but does not require them.

| Field | Type | Purpose |
|-------|------|---------|
| `verified_on` | string | Version(s) this was verified on. Used by forage SEARCH to produce a concrete staleness annotation rather than a generic age warning. Example: `"quarkus: 3.34.2"` or `"tmux: 3.4"`. Only meaningful for library/tool-specific entries — omit for technique/cross-cutting entries. |
| `last_reviewed` | YYYY-MM-DD | Date of last manual staleness review. Resets the staleness clock — forage SWEEP and harvest REVIEW use `max(submitted, last_reviewed)` as the reference date when computing entry age. Set by forage SWEEP (Confirm) and harvest REVIEW (Confirm). |

**Adding to an entry frontmatter:**

```yaml
staleness_threshold: 730
verified_on: "quarkus: 3.34.2"   # optional — omit if not applicable
last_reviewed: 2026-04-14         # optional — set by staleness review
submitted: 2026-04-14
```

**Why this fix section** (optional body section, scores ≥12 only):

For entries scoring 12 or above, forage CAPTURE will prompt for an optional rationale. If provided, it is added as a body section after **Why this is non-obvious**:

```markdown
### Why this fix
[Why this approach over the obvious alternative — written by submitter at capture time]
```
```

- [ ] **Step 2: Verify the document renders cleanly**

Read the file and confirm the new section appears between `## File Naming` and `## Gotcha Template`, with no broken markdown.

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/soredium
git add forage/submission-formats.md
git commit -m "docs(forage): document verified_on, last_reviewed optional fields and rationale section"
```

---

## Task 3: Tests for validate_garden.py --freshness (TDD)

**Files:**
- Modify: `soredium/tests/test_validate_garden.py`

Write the failing tests before implementing `--freshness`. The tests define the contract.

- [ ] **Step 1: Add the helper function and test class**

In `soredium/tests/test_validate_garden.py`, add after the existing `run_validator` function (after line 20):

```python
from datetime import date, timedelta


def write_yaml_entry(root: Path, domain: str, ge_id: str, title: str,
                     submitted: str, threshold: int,
                     last_reviewed: str = None,
                     verified_on: str = None) -> Path:
    """Write a garden entry with YAML frontmatter for freshness testing."""
    category = root / domain
    category.mkdir(exist_ok=True)
    path = category / f"{ge_id}.md"
    lines = [
        "---",
        f"id: {ge_id}",
        f'title: "{title}"',
        "type: gotcha",
        f"domain: {domain}",
        'stack: "Test Stack"',
        "tags: [test]",
        "score: 10",
        "verified: true",
        f"staleness_threshold: {threshold}",
        f"submitted: {submitted}",
    ]
    if last_reviewed:
        lines.append(f"last_reviewed: {last_reviewed}")
    if verified_on:
        lines.append(f'verified_on: "{verified_on}"')
    lines.extend(["---", "", f"## {title}", "", f"**ID:** {ge_id}", ""])
    path.write_text("\n".join(lines))
    return path


def run_freshness(garden_root: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(VALIDATOR), '--freshness', str(garden_root)],
        capture_output=True, text=True
    )


class TestFreshnessFlag(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        # Minimal GARDEN.md so the flag doesn't choke on a missing file
        (self.root / "GARDEN.md").write_text("**Last legacy ID:** GE-0001\n")

    def tearDown(self):
        self.tmp.cleanup()

    def test_no_entries_exits_0(self):
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn("0 entries", result.stdout)

    def test_fresh_entry_exits_0(self):
        yesterday = (date.today() - timedelta(days=1)).isoformat()
        write_yaml_entry(self.root, "tools", "GE-20260414-aabbcc",
                         "Fresh entry", yesterday, 730)
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn("0 entries", result.stdout)

    def test_overdue_entry_exits_2(self):
        old = (date.today() - timedelta(days=800)).isoformat()
        write_yaml_entry(self.root, "tools", "GE-20260414-aabbcc",
                         "Old entry", old, 730)
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 2, result.stdout + result.stderr)
        self.assertIn("1 entries", result.stdout)

    def test_last_reviewed_resets_clock(self):
        old = (date.today() - timedelta(days=800)).isoformat()
        recent = (date.today() - timedelta(days=10)).isoformat()
        write_yaml_entry(self.root, "tools", "GE-20260414-aabbcc",
                         "Reviewed entry", old, 730, last_reviewed=recent)
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn("0 entries", result.stdout)

    def test_entry_without_staleness_threshold_is_skipped(self):
        # Entry with no staleness_threshold field — skipped by freshness check
        category = self.root / "tools"
        category.mkdir()
        (category / "GE-20260414-aabbcc.md").write_text(
            '---\nid: GE-20260414-aabbcc\ntitle: "No threshold"\nsubmitted: 2020-01-01\n---\n'
        )
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn("0 entries", result.stdout)

    def test_multiple_overdue_entries_reported(self):
        old = (date.today() - timedelta(days=800)).isoformat()
        write_yaml_entry(self.root, "tools", "GE-20260414-aabbcc",
                         "Old entry A", old, 730)
        write_yaml_entry(self.root, "java", "GE-20260414-ddeeff",
                         "Old entry B", old, 730)
        result = run_freshness(self.root)
        self.assertEqual(result.returncode, 2, result.stdout + result.stderr)
        self.assertIn("2 entries", result.stdout)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py::TestFreshnessFlag -v
```

Expected: All 6 tests FAIL with `SystemExit` or similar — `--freshness` not yet implemented.

---

## Task 4: Implement validate_garden.py --freshness

**Files:**
- Modify: `soredium/scripts/validate_garden.py`

- [ ] **Step 1: Add the --freshness early-exit block**

In `soredium/scripts/validate_garden.py`, insert the following block immediately after the `--structural` block (after line 42, before `import os`):

```python
if '--freshness' in sys.argv:
    import os as _os
    from datetime import date as _date
    import re as _re

    _idx = sys.argv.index('--freshness')
    # Garden root: next positional arg if present, else $HORTORA_GARDEN or default
    if _idx + 1 < len(sys.argv) and not sys.argv[_idx + 1].startswith('--'):
        _garden = Path(sys.argv[_idx + 1]).expanduser().resolve()
    elif 'HORTORA_GARDEN' in _os.environ:
        _garden = Path(_os.environ['HORTORA_GARDEN']).expanduser().resolve()
    else:
        _garden = (Path.home() / '.hortora' / 'garden').resolve()

    _today = _date.today()
    _overdue = []
    _exclude = {'.git', 'submissions', 'scripts'}
    _skip_names = {'GARDEN.md', 'CHECKED.md', 'DISCARDED.md'}

    for _path in _garden.rglob('*.md'):
        if any(part in _exclude for part in _path.parts):
            continue
        if _path.name in _skip_names:
            continue
        _content = _path.read_text()
        _fm_match = _re.match(r'^---\n(.*?)\n---', _content, _re.DOTALL)
        if not _fm_match:
            continue
        _fm = _fm_match.group(1)

        _t = _re.search(r'staleness_threshold:\s*(\d+)', _fm)
        if not _t:
            continue
        _threshold = int(_t.group(1))

        _s = _re.search(r'submitted:\s*(\d{4}-\d{2}-\d{2})', _fm)
        if not _s:
            continue
        _submitted = _date.fromisoformat(_s.group(1))

        _r = _re.search(r'last_reviewed:\s*(\d{4}-\d{2}-\d{2})', _fm)
        _last_reviewed = _date.fromisoformat(_r.group(1)) if _r else None

        _ref = max(_submitted, _last_reviewed) if _last_reviewed else _submitted
        _age = (_today - _ref).days

        if _age > _threshold:
            _id_m = _re.search(r'^id:\s*(.+)$', _fm, _re.MULTILINE)
            _ti_m = _re.search(r'^title:\s*"?(.+?)"?\s*$', _fm, _re.MULTILINE)
            _overdue.append((
                _id_m.group(1).strip() if _id_m else 'unknown',
                _ti_m.group(1).strip() if _ti_m else 'unknown',
                _age,
                _threshold,
            ))

    _overdue.sort(key=lambda x: x[2], reverse=True)
    print(f"Freshness check: {len(_overdue)} entries past staleness threshold")
    for _ge_id, _entry_title, _age, _thresh in _overdue[:10]:
        print(f"  {_ge_id}: {_entry_title[:60]} ({_age}d / {_thresh}d threshold)")
    if len(_overdue) > 10:
        print(f"  ... and {len(_overdue) - 10} more")
    sys.exit(2 if _overdue else 0)
```

- [ ] **Step 2: Run the freshness tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py::TestFreshnessFlag -v
```

Expected: All 6 tests PASS.

- [ ] **Step 3: Run the full test suite to check for regressions**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ -v
```

Expected: All existing tests still pass.

- [ ] **Step 4: Smoke-test against the live garden**

```bash
python3 ~/claude/hortora/soredium/scripts/validate_garden.py \
  --freshness ~/.hortora/garden
```

Expected: Output shows count of overdue entries (likely many, since `last_reviewed` is not yet set on any entries). Exit code 0 or 2 depending on count.

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_garden.py tests/test_validate_garden.py
git commit -m "feat(validate): add --freshness flag to report entries past staleness_threshold"
```

---

## Task 5: forage SEARCH — S2 Staleness Annotation

**Files:**
- Modify: `soredium/forage/SKILL.md` (SEARCH section)

- [ ] **Step 1: Locate the SEARCH section**

In `soredium/forage/SKILL.md`, find the SEARCH section. It currently ends:

```markdown
4. Return the full entry.

5. If the user just fixed something related, offer to submit via CAPTURE.
```

- [ ] **Step 2: Insert the staleness annotation step**

Replace those two lines with:

```markdown
4. Return the full entry.

5. Append a staleness annotation immediately after the entry content:

   - From the returned entry's YAML frontmatter, read:
     - `submitted` (required)
     - `staleness_threshold` (required)
     - `last_reviewed` (optional — use if present)
     - `verified_on` (optional — include in annotation if present)
   - Compute `reference_date = last_reviewed if present, else submitted`
   - Compute `age_days = (today - reference_date).days`

   **If `age_days > staleness_threshold`:**
   > ⚠️ **Stale entry** — last verified {reference_date} ({age_days} days ago, threshold {staleness_threshold} days). The fix may not apply to your current stack version. Verify before acting.
   > Verified on: {verified_on}  ← only if `verified_on` is present

   **If `age_days > staleness_threshold * 0.75` (approaching threshold):**
   > ℹ️ Entry is {age_days} days old (threshold {staleness_threshold} days) — worth re-verifying if you encounter issues.

   **If `age_days <= staleness_threshold * 0.75`:** no annotation. Do not annotate fresh entries.

6. If the user just fixed something related, offer to submit via CAPTURE.
```

- [ ] **Step 3: Update SEARCH success criteria**

In the Success Criteria section, update the SEARCH entry:

```markdown
SEARCH is complete when:
- ✅ Index read from `GARDEN.md` or `_index/global.md`
- ✅ Entry read from `git cat-file blob HEAD:<path>` (not filesystem)
- ✅ Staleness annotation appended if entry is past or approaching threshold
```

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add forage/SKILL.md
git commit -m "feat(forage): add S2 staleness annotation to SEARCH results"
```

---

## Task 6: forage CAPTURE — S6 verified_on Prompt + S3 Soft Rationale

**Files:**
- Modify: `soredium/forage/SKILL.md` (CAPTURE section)

Two additions: S6 prompts for `verified_on` at CAPTURE time for library/tool entries; S3 softly prompts for a rationale section for high-scoring entries.

- [ ] **Step 1: Extend the Step 3 field extraction table**

In the CAPTURE section, find Step 3 "Extract the 8 fields from conversation context". The table currently ends with `Why non-obvious`. Add two rows:

```markdown
| verified_on | For library/tool-specific entries only (type gotcha or undocumented with a specific stack version): version(s) this was verified on — e.g. "quarkus: 3.34.2". Extract from context if mentioned. Skip for technique/cross-cutting entries. |
| rationale | For entries scoring ≥12: why this fix over the obvious alternative. Extract if explained in context; otherwise prompt in Step 5. |
```

- [ ] **Step 2: Add prompts in Step 5 (Draft and confirm)**

In Step 5, find the line `Wait for confirmation before writing.` and insert before it:

```markdown
**For library/tool-specific entries** (type gotcha or undocumented with a named stack version), if `verified_on` is not already known from context, ask:
> "What version was this verified on? (e.g. 'quarkus: 3.34.2' — or press Enter to skip)"

If provided, include as a YAML frontmatter field after `staleness_threshold`:
```yaml
staleness_threshold: 730
verified_on: "quarkus: 3.34.2"
submitted: YYYY-MM-DD
```

**For entries scoring 12 or above**, ask:
> "Optionally: why this fix over the obvious alternative? (Enter to skip)"

If provided, add a `### Why this fix` body section immediately after `### Why this is non-obvious`:
```markdown
### Why this fix
[Submitter's rationale — why this approach over the obvious alternative]
```
```

- [ ] **Step 3: Update CAPTURE success criteria**

In the Success Criteria section, add to the CAPTURE checklist:

```markdown
- ✅ `verified_on` field included in YAML frontmatter for library/tool-specific entries (if version known)
- ✅ `### Why this fix` section added for entries scoring ≥12 (if rationale provided)
```

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add forage/SKILL.md
git commit -m "feat(forage): add S6 verified_on prompt and S3 rationale soft-prompt to CAPTURE"
```

---

## Task 7: forage SWEEP — S1 Domain-Filtered Staleness Review

**Files:**
- Modify: `soredium/forage/SKILL.md` (SWEEP section)

Adds a staleness spot-check at the end of SWEEP, limited to domains touched in the session.

- [ ] **Step 1: Locate the SWEEP section end**

In the SWEEP section, find:

```markdown
**Step 4 — Submit confirmed entries**

For each finding confirmed by the user: run the CAPTURE workflow with the specific content already known from context. Do NOT ask the user to re-describe things you already know.

**Step 5 — Report**
```

- [ ] **Step 2: Insert Step 5 and renumber Report to Step 6**

Replace the `**Step 5 — Report**` heading and everything after it in SWEEP with:

```markdown
**Step 5 — Staleness spot-check (domain-filtered)**

Derive session domains from:
- The `domain` field of entries submitted in Steps 1–4 of this SWEEP
- Technology stacks mentioned in the session conversation (Quarkus → `quarkus`, tmux → `tools`, etc.)

If no domains can be identified from either source, skip this step entirely.

Otherwise:

1. Read the committed index:
   ```bash
   git -C ${HORTORA_GARDEN:-~/.hortora/garden} show HEAD:GARDEN.md
   ```

2. For each entry in the By Technology section whose domain is in the session domain set:
   - Read the entry's YAML frontmatter (head only):
     ```bash
     git -C ${HORTORA_GARDEN:-~/.hortora/garden} show HEAD:<domain>/GE-XXXX.md | head -20
     ```
   - Compute `reference_date = max(submitted, last_reviewed)` if `last_reviewed` present, else `submitted`
   - If `(today - reference_date).days > staleness_threshold`: add to overdue list

3. For each overdue entry, present to the user:
   > ⚠️ **GE-XXXX** "[title]" — {N} days old (threshold {T} days).
   > **Confirm** still valid / **Revise** / **Retire** / **Skip**?

   - **Confirm**: add `last_reviewed: YYYY-MM-DD` to YAML frontmatter in working tree. Commit after all responses collected.
   - **Revise**: run the REVISE workflow for this entry.
   - **Retire**: run REVISE with `deprecated` kind — add `**Deprecated:** [reason] — {date}` near the top of the entry body.
   - **Skip**: no action.

4. If any entries were Confirmed: commit all `last_reviewed` additions atomically:
   ```bash
   git -C ${HORTORA_GARDEN:-~/.hortora/garden} add .
   git -C ${HORTORA_GARDEN:-~/.hortora/garden} commit -m "review: staleness spot-check — N confirmed, M revised, K retired"
   ```

If no overdue entries are found in session domains, omit this step from the report entirely (no noise for fresh gardens).

**Step 6 — Report**

Tell the user:
- How many candidates were found in each category (gotchas, techniques, undocumented)
- How many were confirmed and submitted
- If staleness spot-check ran: N overdue entries found in session domains, M resolved
- If nothing was found: "Nothing garden-worthy surfaced in this session across gotchas, techniques, or undocumented items."
```

- [ ] **Step 3: Update SWEEP success criteria**

In the Success Criteria section, update the SWEEP entry:

```markdown
SWEEP is complete when:
- ✅ All three categories checked from session memory
- ✅ Each finding proposed explicitly with type and description
- ✅ Confirmed entries submitted via CAPTURE
- ✅ Staleness spot-check run for session domains (if identifiable)
- ✅ Overdue entries in session domains presented and resolved/skipped
- ✅ Report given: N found, M submitted per category; overdue entries resolved
```

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add forage/SKILL.md
git commit -m "feat(forage): add S1 domain-filtered staleness spot-check to SWEEP"
```

---

## Task 8: harvest SKILL.md — Add REVIEW Mode

**Files:**
- Modify: `soredium/harvest/SKILL.md`

Adds REVIEW as a second harvest operation alongside DEDUPE. Updates the skill description, adds the workflow, updates success criteria and skill chaining.

- [ ] **Step 1: Update the YAML frontmatter description**

In `soredium/harvest/SKILL.md`, replace the `description:` field:

```yaml
description: >
  Use when deduplicating existing garden entries (DEDUPE) or reviewing all stale entries
  across the garden (REVIEW). DEDUPE: user says "dedupe the garden", "check for duplicates".
  REVIEW: user says "review stale entries", "staleness review", "harvest review", or when
  validate_garden.py --freshness shows many overdue entries. Run as a dedicated session
  with full context budget. Do NOT use during normal session work — use forage for
  session-time operations. Never invoked automatically; always a deliberate maintenance
  operation.
```

- [ ] **Step 2: Update the opening paragraph**

Replace:

```markdown
Harvest handles one maintenance operation for the knowledge garden:

- **DEDUPE** — find and resolve duplicate entries within the existing garden
```

With:

```markdown
Harvest handles two maintenance operations for the knowledge garden:

- **DEDUPE** — find and resolve duplicate entries within the existing garden
- **REVIEW** — find and resolve entries past their `staleness_threshold` across all domains
```

- [ ] **Step 3: Add the REVIEW workflow**

Insert the following section immediately after the `### DEDUPE` section ends (after Step 8 — Report and its content), before `## Common Pitfalls`:

```markdown
---

### REVIEW (find and resolve stale entries)

Use when: "review stale entries", "staleness review", "harvest review", or when
`validate_garden.py --freshness` shows many overdue entries across the garden.

Unlike the SWEEP staleness spot-check (which is domain-filtered and session-scoped),
REVIEW covers ALL domains systematically — it is the backstop that guarantees
full garden coverage.

**Step 1 — Report overdue count**

```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/validate_garden.py \
  --freshness ${HORTORA_GARDEN:-~/.hortora/garden}
```

Note the count and the oldest entries listed. If count is 0, report to user and stop — no
REVIEW needed.

**Step 2 — Load all entries and build the overdue list**

Read the committed index:
```bash
git -C ${HORTORA_GARDEN:-~/.hortora/garden} show HEAD:GARDEN.md
```

For each entry in the By Technology section, read its YAML frontmatter:
```bash
git -C ${HORTORA_GARDEN:-~/.hortora/garden} show HEAD:<domain>/GE-XXXX.md | head -25
```

Build the overdue list: entries where `(today - max(submitted, last_reviewed)).days > staleness_threshold`.
Sort oldest-first.

**Step 3 — Process each overdue entry**

For each entry in the overdue list, present:

> ⚠️ **GE-XXXX** "[title]"
> Domain: {domain} | Submitted: {submitted} | Last reviewed: {last_reviewed or "never"} | Age: {N} days | Threshold: {T} days
> **CONFIRM** still valid / **REVISE** / **RETIRE** / **Skip**?

- **CONFIRM**: add `last_reviewed: YYYY-MM-DD` to YAML frontmatter in working tree (or update if already present).
- **REVISE**: run the REVISE workflow for this entry in-place.
- **RETIRE**: add `**Deprecated:** [reason] — {date}` near the top of the entry body. Keep content intact — users on older versions still need it.
- **Skip**: no action. Entry will appear on the next REVIEW.

**Step 4 — Update GARDEN.md staleness tracking**

In GARDEN.md working tree, update the `Last staleness review` line:

```
**Last staleness review:** YYYY-MM-DD
```

**Step 5 — Commit atomically**

```bash
git -C ${HORTORA_GARDEN:-~/.hortora/garden} add .
git -C ${HORTORA_GARDEN:-~/.hortora/garden} commit -m "review: staleness sweep — N confirmed, M revised, K retired, P skipped"
```

If the commit fails (concurrent forage session committed first):
```bash
git -C ${HORTORA_GARDEN:-~/.hortora/garden} rebase HEAD
# Re-commit
```

**Step 6 — Report**

Tell the user:
- How many entries were overdue in total
- How many confirmed / revised / retired / skipped
- That `Last staleness review` in GARDEN.md is now updated to today
```

- [ ] **Step 4: Add REVIEW to Common Pitfalls table**

In the `## Common Pitfalls` table, add a row:

```markdown
| Running REVIEW during normal session work | Needs full context budget for full garden sweep | Always run harvest as a dedicated session |
```

- [ ] **Step 5: Add REVIEW success criteria**

In `## Success Criteria`, add after the DEDUPE block:

```markdown
REVIEW is complete when:
- ✅ `validate_garden.py --freshness` run and count noted
- ✅ All overdue entries read via `git show HEAD:<path>` (not filesystem)
- ✅ All overdue entries processed (CONFIRM / REVISE / RETIRE / Skip)
- ✅ `last_reviewed` field added/updated for all CONFIRMed entries
- ✅ RETIRE entries have deprecation note added, original content preserved
- ✅ `Last staleness review` in GARDEN.md updated to today
- ✅ All changes committed atomically with `review:` message format
```

- [ ] **Step 6: Update Skill Chaining section**

Replace the `**Invoked by:**` and `**Does NOT handle:**` lines with:

```markdown
**Invoked by:** User directly for maintenance sessions
- DEDUPE: "dedupe the garden", "check for duplicates", drift counter at threshold
- REVIEW: "review stale entries", "staleness review", "harvest review", or when `--freshness` shows many overdue entries

**Does NOT handle:** CAPTURE, SWEEP, SEARCH, REVISE — those are forage operations.
```

- [ ] **Step 7: Commit**

```bash
cd ~/claude/hortora/soredium
git add harvest/SKILL.md
git commit -m "feat(harvest): add REVIEW mode for systematic staleness enforcement (S5)"
```

---

## Self-Review

**Spec coverage check:**

| Requirement | Covered by |
|-------------|-----------|
| S2 — annotation at SEARCH time, always-on | Task 5 |
| S6 — `verified_on` captured at CAPTURE time | Task 6 |
| S6 — `verified_on` used in SEARCH annotation | Task 5 (annotation step references `verified_on`) |
| S3 — soft rationale prompt at CAPTURE for ≥12 | Task 6 |
| S1 — domain-filtered staleness in SWEEP | Task 7 |
| S5 — `harvest REVIEW` systematic sweep | Task 8 |
| S5 — `validate_garden.py --freshness` as trigger | Task 4 |
| `last_reviewed` schema defined | Task 2 |
| `last_reviewed` set by SWEEP Confirm | Task 7 |
| `last_reviewed` set by harvest REVIEW Confirm | Task 8 |
| `Last staleness review` in GARDEN.md | Tasks 1 + 8 |
| `GARDEN.md` updated by harvest REVIEW | Task 8 |

**No gaps found.**

**Placeholder scan:** No "TBD", "TODO", or vague steps. All skill file changes include exact replacement text. All Python code is complete and runnable.

**Type consistency:** `last_reviewed` used consistently as YAML field name throughout. `verified_on` used consistently. `staleness_threshold` unchanged from existing schema. `reference_date` and `age_days` are local computation variables only (no inter-task dependency). `--freshness` flag name used consistently in tests, implementation, and skill files.
