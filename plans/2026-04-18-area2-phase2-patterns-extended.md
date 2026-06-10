# Area 2 Phase 2 — Patterns-Garden Extended Entry Format

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional extended fields to the patterns-garden entry schema — provenance, suitability, variants, developer attribution, and stability — with validator warnings for malformed values and updated submission templates.

**Architecture:** A new `validate_patterns_extended(fm)` function returns a list of warning strings for any malformed optional fields. It is called from `validate()` when `garden == 'patterns'`. All violations are WARNINGs (not CRITICALs) because the fields are optional enrichments. Submission template and forage skill updated to show the full patterns-garden format.

**Tech Stack:** Python 3.11+, PyYAML, pytest. All changes in `Hortora/soredium`.

**Issues:** Closes #33 (part of epic #31)

**Tests:** TDD throughout — write failing test, verify fail, implement, verify pass, commit.

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `scripts/validate_pr.py` | Modify | Add `validate_patterns_extended(fm)`; wire into `validate()` when `garden == 'patterns'` |
| `tests/test_garden_types.py` | Modify | Append patterns extended field tests |
| `forage/submission-formats.md` | Modify | Update patterns-garden template with all extended fields |
| `forage/SKILL.md` | Modify | Update CAPTURE Step 3 for patterns-garden entry extraction; update success criteria |

---

## Extended Fields — spec reference

From `spec/docs/superpowers/specs/2026-04-18-area2-taxonomy-design.md`:

```yaml
observed_in:
  - project: serverless-workflow          # required per item
    url: https://github.com/...           # optional
    path: impl/core/src/main/java/...    # optional
    first_seen: "2022-03-14"             # optional, YYYY-MM-DD
    last_seen: "2026-04-01"             # optional, YYYY-MM-DD

suitability: |                           # optional, freetext
  Works when evaluation logic must be swappable at runtime.

variants:                                # optional list
  - name: in-memory                     # required per item
    description: Evaluator holds state  # optional
    tradeoffs: Fast; not distributable  # optional

variant_frequency:                       # optional dict[str, int]
  in-memory: 12
  event-sourced: 31

authors:                                 # optional list
  - github_handle: fabian-martinez       # required per item
    role: originator                     # required, one of: originator|adopter|innovator

stability: high                          # optional, one of: low|medium|high
```

All extended fields are optional. Malformed values produce WARNINGs, not CRITICALs.

---

## Task 1: `validate_patterns_extended` — observed_in and authors (TDD)

**Files:**
- Modify: `scripts/validate_pr.py` (add new function after `check_injection`)
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write failing tests**

Append to `tests/test_garden_types.py`:

```python
from validate_pr import validate_patterns_extended


def test_patterns_extended_valid_observed_in_no_warnings():
    fm = {
        'observed_in': [
            {'project': 'serverless-workflow', 'url': 'https://github.com/x', 'first_seen': '2022-03-14'}
        ]
    }
    warnings = validate_patterns_extended(fm)
    assert warnings == []


def test_patterns_extended_observed_in_not_list():
    fm = {'observed_in': 'serverless-workflow'}
    warnings = validate_patterns_extended(fm)
    assert any('observed_in' in w and 'list' in w.lower() for w in warnings)


def test_patterns_extended_observed_in_item_missing_project():
    fm = {'observed_in': [{'url': 'https://github.com/x'}]}
    warnings = validate_patterns_extended(fm)
    assert any('observed_in' in w and 'project' in w for w in warnings)


def test_patterns_extended_valid_authors_no_warnings():
    fm = {
        'authors': [
            {'github_handle': 'fabian-martinez', 'role': 'originator'},
            {'github_handle': 'sanne-grinovero', 'role': 'innovator'},
        ]
    }
    warnings = validate_patterns_extended(fm)
    assert warnings == []


def test_patterns_extended_authors_not_list():
    fm = {'authors': {'github_handle': 'fabian-martinez', 'role': 'originator'}}
    warnings = validate_patterns_extended(fm)
    assert any('authors' in w and 'list' in w.lower() for w in warnings)


def test_patterns_extended_authors_item_missing_github_handle():
    fm = {'authors': [{'role': 'originator'}]}
    warnings = validate_patterns_extended(fm)
    assert any('authors' in w and 'github_handle' in w for w in warnings)


def test_patterns_extended_authors_invalid_role():
    fm = {'authors': [{'github_handle': 'fabian-martinez', 'role': 'creator'}]}
    warnings = validate_patterns_extended(fm)
    assert any('authors' in w and 'role' in w for w in warnings)


def test_patterns_extended_authors_item_missing_role():
    fm = {'authors': [{'github_handle': 'fabian-martinez'}]}
    warnings = validate_patterns_extended(fm)
    assert any('authors' in w and 'role' in w for w in warnings)


def test_patterns_extended_empty_fm_no_warnings():
    warnings = validate_patterns_extended({})
    assert warnings == []
```

- [ ] **Step 2: Run to verify they fail**

```bash
cd /Users/mdproctor/claude/hortora/soredium
python -m pytest tests/test_garden_types.py::test_patterns_extended_valid_observed_in_no_warnings \
                 tests/test_garden_types.py::test_patterns_extended_observed_in_not_list -v 2>&1 | tail -10
```

Expected: `ImportError: cannot import name 'validate_patterns_extended'`

- [ ] **Step 3: Implement `validate_patterns_extended` in validate_pr.py**

Insert after `check_injection` (after line ~138):

```python
VALID_AUTHOR_ROLES = {'originator', 'adopter', 'innovator'}
VALID_STABILITY = {'low', 'medium', 'high'}


def validate_patterns_extended(fm: dict) -> list:
    """Return warning strings for malformed optional patterns-garden fields."""
    warnings = []

    # observed_in: must be a list; each item must have 'project'
    observed_in = fm.get('observed_in')
    if observed_in is not None:
        if not isinstance(observed_in, list):
            warnings.append("observed_in must be a list of project dicts")
        else:
            for i, item in enumerate(observed_in):
                if not isinstance(item, dict) or 'project' not in item:
                    warnings.append(
                        f"observed_in[{i}] missing required 'project' key"
                    )

    # authors: must be a list; each item must have github_handle and valid role
    authors = fm.get('authors')
    if authors is not None:
        if not isinstance(authors, list):
            warnings.append("authors must be a list of dicts")
        else:
            for i, item in enumerate(authors):
                if not isinstance(item, dict):
                    warnings.append(f"authors[{i}] must be a dict")
                    continue
                if 'github_handle' not in item:
                    warnings.append(f"authors[{i}] missing required 'github_handle'")
                role = item.get('role')
                if role is None:
                    warnings.append(
                        f"authors[{i}] missing required 'role' "
                        f"(valid: {sorted(VALID_AUTHOR_ROLES)})"
                    )
                elif role not in VALID_AUTHOR_ROLES:
                    warnings.append(
                        f"authors[{i}] invalid role '{role}' "
                        f"(valid: {sorted(VALID_AUTHOR_ROLES)})"
                    )

    return warnings
```

- [ ] **Step 4: Run tests**

```bash
python -m pytest tests/test_garden_types.py -v -k "patterns_extended" 2>&1 | tail -20
```

Expected: all 9 tests PASS

- [ ] **Step 5: Run full suite**

```bash
python -m pytest tests/ --tb=short -q 2>&1 | tail -5
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add scripts/validate_pr.py tests/test_garden_types.py
git commit -m "feat(validate_pr): validate_patterns_extended — observed_in and authors

Refs #33"
```

---

## Task 2: `validate_patterns_extended` — stability, variants, variant_frequency (TDD)

**Files:**
- Modify: `scripts/validate_pr.py` (extend `validate_patterns_extended`)
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write failing tests**

Append to `tests/test_garden_types.py`:

```python
def test_patterns_extended_valid_stability_no_warnings():
    for val in ('low', 'medium', 'high'):
        warnings = validate_patterns_extended({'stability': val})
        assert warnings == [], f"stability '{val}' produced unexpected warning"


def test_patterns_extended_invalid_stability():
    warnings = validate_patterns_extended({'stability': 'very-high'})
    assert any('stability' in w for w in warnings)


def test_patterns_extended_valid_variants_no_warnings():
    fm = {
        'variants': [
            {'name': 'in-memory', 'description': 'Fast', 'tradeoffs': 'Not distributable'},
            {'name': 'event-sourced'},
        ]
    }
    warnings = validate_patterns_extended(fm)
    assert warnings == []


def test_patterns_extended_variants_not_list():
    warnings = validate_patterns_extended({'variants': 'in-memory'})
    assert any('variants' in w and 'list' in w.lower() for w in warnings)


def test_patterns_extended_variants_item_missing_name():
    warnings = validate_patterns_extended({'variants': [{'description': 'Fast'}]})
    assert any('variants' in w and 'name' in w for w in warnings)


def test_patterns_extended_valid_variant_frequency_no_warnings():
    fm = {'variant_frequency': {'in-memory': 12, 'event-sourced': 31}}
    warnings = validate_patterns_extended(fm)
    assert warnings == []


def test_patterns_extended_variant_frequency_not_dict():
    warnings = validate_patterns_extended({'variant_frequency': [12, 31]})
    assert any('variant_frequency' in w for w in warnings)


def test_patterns_extended_variant_frequency_non_int_value():
    warnings = validate_patterns_extended({'variant_frequency': {'in-memory': 'many'}})
    assert any('variant_frequency' in w and 'int' in w.lower() for w in warnings)
```

- [ ] **Step 2: Run to verify they fail**

```bash
python -m pytest tests/test_garden_types.py -v -k "stability or variants or variant_frequency" 2>&1 | tail -15
```

Expected: FAIL — stability/variants/variant_frequency not yet validated

- [ ] **Step 3: Extend `validate_patterns_extended` in validate_pr.py**

Append to the `validate_patterns_extended` function body (before the final `return warnings`):

```python
    # stability: one of low|medium|high
    stability = fm.get('stability')
    if stability is not None and stability not in VALID_STABILITY:
        warnings.append(
            f"stability '{stability}' invalid (valid: {sorted(VALID_STABILITY)})"
        )

    # variants: must be a list; each item must have 'name'
    variants = fm.get('variants')
    if variants is not None:
        if not isinstance(variants, list):
            warnings.append("variants must be a list of dicts")
        else:
            for i, item in enumerate(variants):
                if not isinstance(item, dict) or 'name' not in item:
                    warnings.append(f"variants[{i}] missing required 'name' key")

    # variant_frequency: must be a dict with int values
    variant_frequency = fm.get('variant_frequency')
    if variant_frequency is not None:
        if not isinstance(variant_frequency, dict):
            warnings.append("variant_frequency must be a dict mapping variant name to int count")
        else:
            for k, v in variant_frequency.items():
                if not isinstance(v, int):
                    warnings.append(
                        f"variant_frequency['{k}'] must be an int, got {type(v).__name__}"
                    )
```

- [ ] **Step 4: Run extended tests**

```bash
python -m pytest tests/test_garden_types.py -v -k "patterns_extended" 2>&1 | tail -25
```

Expected: all 17 tests PASS

- [ ] **Step 5: Run full suite**

```bash
python -m pytest tests/ --tb=short -q 2>&1 | tail -5
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add scripts/validate_pr.py tests/test_garden_types.py
git commit -m "feat(validate_pr): patterns extended validation — stability, variants, variant_frequency

Refs #33"
```

---

## Task 3: Wire `validate_patterns_extended` into `validate()` (TDD)

**Files:**
- Modify: `scripts/validate_pr.py` (`validate()` function)
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write failing tests**

Append to `tests/test_garden_types.py`:

```python
FULL_PATTERNS_ENTRY = """\
---
id: GE-20260418-aabbcc
garden: patterns
title: "Pluggable Evaluator with Swappable Persistence"
type: architectural
domain: jvm
stack: "Java, Quarkus"
tags: [java, architecture, quarkus]
score: 11
verified: true
staleness_threshold: 3650
observed_in:
  - project: serverless-workflow
    url: https://github.com/serverlessworkflow/specification
    first_seen: "2022-03-14"
suitability: |
  Works when evaluation logic must be swappable at runtime.
variants:
  - name: in-memory
    description: Evaluator holds state in-process
    tradeoffs: Fast, not distributable
  - name: event-sourced
    description: State from event log
    tradeoffs: Distributable, higher latency
variant_frequency:
  in-memory: 12
  event-sourced: 31
authors:
  - github_handle: fabian-martinez
    role: originator
stability: high
submitted: 2026-04-18
---

## Pattern

Description here.
"""

MALFORMED_PATTERNS_ENTRY_OBSERVED_IN = """\
---
id: GE-20260418-aabbcd
garden: patterns
title: "Some Pattern"
type: architectural
domain: jvm
stack: "Java"
tags: [java]
score: 10
verified: true
staleness_threshold: 3650
observed_in: not-a-list
submitted: 2026-04-18
---

## Pattern

Description.
"""

MALFORMED_PATTERNS_ENTRY_BAD_ROLE = """\
---
id: GE-20260418-aabbce
garden: patterns
title: "Some Pattern"
type: architectural
domain: jvm
stack: "Java"
tags: [java]
score: 10
verified: true
staleness_threshold: 3650
authors:
  - github_handle: fabian-martinez
    role: inventor
submitted: 2026-04-18
---

## Pattern

Description.
"""


def test_full_patterns_entry_with_extended_fields_passes():
    path = _write_entry(FULL_PATTERNS_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
        # Extended field warnings should NOT appear — entry is valid
        extended_warnings = [w for w in result['warnings'] if any(
            field in w for field in ['observed_in', 'authors', 'stability', 'variants', 'variant_frequency']
        )]
        assert extended_warnings == [], extended_warnings
    finally:
        os.unlink(path)


def test_patterns_entry_malformed_observed_in_produces_warning():
    path = _write_entry(MALFORMED_PATTERNS_ENTRY_OBSERVED_IN)
    try:
        result = validate(path)
        assert result['criticals'] == []
        assert any('observed_in' in w for w in result['warnings'])
    finally:
        os.unlink(path)


def test_patterns_entry_bad_author_role_produces_warning():
    path = _write_entry(MALFORMED_PATTERNS_ENTRY_BAD_ROLE)
    try:
        result = validate(path)
        assert result['criticals'] == []
        assert any('authors' in w and 'role' in w for w in result['warnings'])
    finally:
        os.unlink(path)


def test_discovery_entry_extended_fields_not_checked():
    """validate_patterns_extended must NOT run for non-patterns gardens."""
    # Add a malformed observed_in to a discovery entry — should produce no warning
    entry = DISCOVERY_ENTRY + "\nobserved_in: not-a-list\n"
    # Rebuild with the field in frontmatter
    malformed = DISCOVERY_ENTRY.replace(
        'submitted: 2026-04-18\n---',
        'submitted: 2026-04-18\nobserved_in: not-a-list\n---'
    )
    path = _write_entry(malformed)
    try:
        result = validate(path)
        extended_warnings = [w for w in result['warnings'] if 'observed_in' in w]
        assert extended_warnings == [], \
            "validate_patterns_extended must not run for discovery-garden"
    finally:
        os.unlink(path)
```

- [ ] **Step 2: Run to verify they fail**

```bash
python -m pytest tests/test_garden_types.py::test_full_patterns_entry_with_extended_fields_passes \
                 tests/test_garden_types.py::test_patterns_entry_malformed_observed_in_produces_warning -v 2>&1 | tail -15
```

Expected: FAIL — `validate()` doesn't call `validate_patterns_extended` yet

- [ ] **Step 3: Wire into `validate()` in validate_pr.py**

In the `validate()` function, find the garden-specific required fields block (after the `for field in garden_cfg['required_extra']` loop, around line 236). Insert immediately after:

```python
    # patterns-garden: validate optional extended fields
    if garden == 'patterns':
        result['warnings'].extend(validate_patterns_extended(fm))
```

- [ ] **Step 4: Run integration tests**

```bash
python -m pytest tests/test_garden_types.py -v -k "full_patterns or malformed_patterns or bad_author or discovery_entry_extended" 2>&1 | tail -20
```

Expected: all 4 new tests PASS

- [ ] **Step 5: Run full suite**

```bash
python -m pytest tests/ --tb=short -q 2>&1 | tail -5
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add scripts/validate_pr.py tests/test_garden_types.py
git commit -m "feat(validate_pr): wire validate_patterns_extended into validate()

Patterns-garden entries with malformed extended fields produce warnings.
Non-patterns gardens are unaffected.

Refs #33"
```

---

## Task 4: Happy path + correctness tests for all extended field combinations

**Files:**
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write correctness tests**

Append to `tests/test_garden_types.py`:

```python
def test_patterns_entry_with_no_extended_fields_still_passes():
    """A minimal patterns entry (no extended fields) must pass cleanly."""
    path = _write_entry(PATTERNS_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
        extended_warnings = [w for w in result['warnings'] if any(
            f in w for f in ['observed_in', 'authors', 'stability', 'variants', 'variant_frequency']
        )]
        assert extended_warnings == []
    finally:
        os.unlink(path)


def test_patterns_entry_invalid_stability_produces_warning_not_critical():
    entry = FULL_PATTERNS_ENTRY.replace('stability: high', 'stability: extreme')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert result['criticals'] == []
        assert any('stability' in w for w in result['warnings'])
    finally:
        os.unlink(path)


def test_patterns_entry_variants_missing_name_produces_warning():
    entry = FULL_PATTERNS_ENTRY.replace(
        '  - name: in-memory\n    description: Evaluator holds state in-process\n    tradeoffs: Fast, not distributable\n',
        '  - description: Evaluator holds state in-process\n    tradeoffs: Fast, not distributable\n'
    )
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert result['criticals'] == []
        assert any('variants' in w and 'name' in w for w in result['warnings'])
    finally:
        os.unlink(path)


def test_patterns_entry_variant_frequency_non_int_produces_warning():
    entry = FULL_PATTERNS_ENTRY.replace('  in-memory: 12', '  in-memory: "twelve"')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert result['criticals'] == []
        assert any('variant_frequency' in w for w in result['warnings'])
    finally:
        os.unlink(path)
```

- [ ] **Step 2: Run new tests**

```bash
python -m pytest tests/test_garden_types.py -v \
  -k "no_extended_fields or invalid_stability_produces or variants_missing or variant_frequency_non_int" \
  2>&1 | tail -15
```

Expected: all 4 PASS

- [ ] **Step 3: Run full suite**

```bash
python -m pytest tests/ --tb=short -q 2>&1 | tail -5
```

Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git add tests/test_garden_types.py
git commit -m "test(garden_types): correctness tests for patterns-garden extended fields

Refs #33"
```

---

## Task 5: Update submission-formats.md patterns-garden template

**Files:**
- Modify: `forage/submission-formats.md`

- [ ] **Step 1: Read the file to find the patterns-garden template**

Read `forage/submission-formats.md`. Find the `## patterns-garden Entry Template` section.

- [ ] **Step 2: Replace the patterns-garden YAML frontmatter block**

Replace the existing patterns frontmatter block (the simple one without extended fields) with:

````markdown
## patterns-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: patterns
title: "Pattern name — what it does, not how"
type: architectural | migration | integration | testing
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Technology, Version"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 3650
submitted: YYYY-MM-DD

# Extended fields — all optional, each enriches retrieval and attribution
observed_in:
  - project: project-name
    url: https://github.com/org/repo
    path: path/to/relevant/directory/
    first_seen: "YYYY-MM-DD"         # when this pattern first appeared here
    last_seen: "YYYY-MM-DD"          # when last confirmed still present

suitability: |
  Works when [condition]. Less suitable when [constraint].

variants:
  - name: variant-name              # required
    description: One sentence.      # optional
    tradeoffs: Pros and cons.       # optional

variant_frequency:
  variant-name: N                   # how many projects use this variant

authors:
  - github_handle: github-username  # required
    role: originator | adopter | innovator  # required

stability: low | medium | high      # high = consistent across 3+ major versions of 5+ projects
---
```
````

- [ ] **Step 3: Verify the file has all 6 gardens and the extended fields**

```bash
python3 -c "
with open('forage/submission-formats.md') as f:
    c = f.read()
for g in ['discovery', 'patterns', 'examples', 'evolution', 'risk', 'decisions']:
    assert g in c, f'Missing {g}'
for field in ['observed_in', 'suitability', 'variants', 'variant_frequency', 'authors', 'stability']:
    assert field in c, f'Missing extended field {field}'
print('All 6 gardens + all extended fields present')
"
```

Expected: `All 6 gardens + all extended fields present`

- [ ] **Step 4: Commit**

```bash
git add forage/submission-formats.md
git commit -m "docs(submission-formats): patterns-garden template with extended fields

Refs #33"
```

---

## Task 6: Update forage SKILL.md — patterns-garden CAPTURE guidance

**Files:**
- Modify: `forage/SKILL.md`

- [ ] **Step 1: Read Step 3 (Extract the 8 fields) in forage/SKILL.md**

Read `forage/SKILL.md`. Find `**Step 3 — Extract the 8 fields from conversation context**`.

- [ ] **Step 2: Add patterns-garden extended field extraction guidance**

After the existing extraction table in Step 3, add:

```markdown
**For patterns-garden entries**, also extract these optional fields from context:

| Field | Extract from |
|-------|-------------|
| `observed_in` | Projects where this pattern was observed in the session — URL, path if mentioned |
| `suitability` | Any discussion of when the pattern works / when it doesn't |
| `variants` | Named adaptations discussed (in-memory vs event-sourced, etc.) |
| `variant_frequency` | Any frequency or adoption counts mentioned |
| `authors` | Developer names or GitHub handles attributed to the pattern |
| `stability` | Any discussion of how widely or consistently the pattern is used |

These fields are optional — include only what's known from context. Do not ask the user to supply them if they weren't discussed.
```

- [ ] **Step 3: Update CAPTURE success criteria**

Find the CAPTURE success criteria and add after the existing garden-field items:

```markdown
- ✅ For patterns-garden: optional extended fields (`observed_in`, `suitability`, `variants`, `authors`, `stability`) included where known from context; validator warnings checked
```

- [ ] **Step 4: Verify**

```bash
python3 -c "
with open('forage/SKILL.md') as f:
    c = f.read()
for field in ['observed_in', 'suitability', 'variants', 'variant_frequency', 'authors', 'stability']:
    assert field in c, f'Missing {field} in SKILL.md'
print('All extended fields referenced in SKILL.md')
"
```

Expected: `All extended fields referenced in SKILL.md`

- [ ] **Step 5: Commit**

```bash
git add forage/SKILL.md
git commit -m "feat(forage): patterns-garden extended field extraction guidance

Refs #33"
```

---

## Task 7: Full suite green, sync, close issue, push

- [ ] **Step 1: Run full test suite — must be green**

```bash
cd /Users/mdproctor/claude/hortora/soredium
python -m pytest tests/ -v --tb=short 2>&1 | tail -20
```

Expected: all tests PASS. Do not proceed if any fail.

- [ ] **Step 2: Sync forage skill**

```bash
echo "Y" | python3 scripts/claude-skill sync-local
```

Expected: `✅ forage` synced

- [ ] **Step 3: Final closing commit**

```bash
git add -A
git status  # verify nothing unexpected
git commit -m "feat(taxonomy): Area 2 Phase 2 complete — patterns-garden extended format

validate_pr.py validates optional extended fields (observed_in, authors,
stability, variants, variant_frequency) for patterns-garden entries.
Submission template and forage skill updated.

Closes #33
Refs #31"
```

- [ ] **Step 4: Push**

```bash
git push origin main
```

- [ ] **Step 5: Report git log of all commits**

```bash
git log origin/main..HEAD --oneline 2>/dev/null || git log -10 --oneline
```

---

## Test Coverage Summary

| Test file | Tests added | What they cover |
|-----------|-------------|-----------------|
| `tests/test_garden_types.py` | ~21 | Unit: `validate_patterns_extended` structure; Correctness: each field validated; Integration: wired into `validate()`; Happy path: full valid entry, minimal entry; Correctness: malformed produces warning not critical; Isolation: non-patterns gardens unaffected |

**Test types per requirement:**
- **Unit:** `validate_patterns_extended` for each field in isolation
- **Correctness:** Each malformed field produces a WARNING (not CRITICAL)
- **Happy path:** Full valid patterns entry with all extended fields passes cleanly
- **Isolation:** Non-patterns gardens not affected by extended validation
- **Backward compat:** Minimal patterns entry (no extended fields) still passes
