# Area 2 Phase 1 — Core Taxonomy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `garden` field to the entry schema, teach `validate_pr.py` to validate garden types and per-garden entry types, and update submission templates and forage skill for all 6 garden types.

**Architecture:** Add a `GARDEN_TYPES` registry to `validate_pr.py` that maps each garden to its allowed entry types and garden-specific required fields. The `garden` field defaults to `discovery` when absent (backward compatible). New entries must declare their garden. Submission templates and forage skill updated to reflect all 6 gardens.

**Tech Stack:** Python 3.11+, PyYAML, pytest. All changes in `Hortora/soredium`.

**Issues:** Closes #32 (part of epic #31)

**Tests:** TDD throughout — write failing test, verify it fails, implement, verify it passes, commit.

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `scripts/validate_pr.py` | Modify | Add GARDEN_TYPES; garden field validation; per-garden type validation; per-garden required fields |
| `tests/test_garden_types.py` | Create | All garden type validation tests (new file, clean separation) |
| `tests/test_validate_pr.py` | Modify | Update VALID_ENTRY fixture to include garden field |
| `forage/submission-formats.md` | Modify | Add templates for all 6 garden types |
| `forage/SKILL.md` | Modify | Garden selection step; editorial bars per garden |

---

## Task 1: GARDEN_TYPES registry in validate_pr.py (TDD)

**Files:**
- Modify: `scripts/validate_pr.py` (after line 51, after BONUS_RULES)
- Create: `tests/test_garden_types.py`

- [ ] **Step 1: Write the failing test**

Create `tests/test_garden_types.py`:

```python
import pytest
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from validate_pr import GARDEN_TYPES, GARDEN_DEFAULT


def test_all_six_gardens_present():
    expected = {'discovery', 'patterns', 'examples', 'evolution', 'risk', 'decisions'}
    assert set(GARDEN_TYPES.keys()) == expected


def test_each_garden_has_valid_types():
    for garden, cfg in GARDEN_TYPES.items():
        assert 'valid_types' in cfg, f"{garden} missing valid_types"
        assert len(cfg['valid_types']) > 0, f"{garden} has empty valid_types"


def test_each_garden_has_required_extra():
    for garden, cfg in GARDEN_TYPES.items():
        assert 'required_extra' in cfg, f"{garden} missing required_extra"
        assert isinstance(cfg['required_extra'], list)


def test_each_garden_has_staleness_default():
    for garden, cfg in GARDEN_TYPES.items():
        assert 'staleness_default' in cfg, f"{garden} missing staleness_default"
        assert isinstance(cfg['staleness_default'], int)


def test_discovery_valid_types():
    assert set(GARDEN_TYPES['discovery']['valid_types']) == {
        'gotcha', 'technique', 'undocumented'
    }


def test_patterns_valid_types():
    assert set(GARDEN_TYPES['patterns']['valid_types']) == {
        'architectural', 'migration', 'integration', 'testing'
    }


def test_examples_valid_types():
    assert GARDEN_TYPES['examples']['valid_types'] == ['code']


def test_evolution_valid_types():
    assert set(GARDEN_TYPES['evolution']['valid_types']) == {
        'breaking', 'deprecation', 'capability'
    }


def test_evolution_requires_changed_in():
    assert 'changed_in' in GARDEN_TYPES['evolution']['required_extra']


def test_risk_valid_types():
    assert set(GARDEN_TYPES['risk']['valid_types']) == {
        'failure-mode', 'antipattern', 'incident'
    }


def test_risk_requires_severity():
    assert 'severity' in GARDEN_TYPES['risk']['required_extra']


def test_decisions_valid_types():
    assert set(GARDEN_TYPES['decisions']['valid_types']) == {
        'architecture', 'technology', 'process'
    }


def test_default_garden_is_discovery():
    assert GARDEN_DEFAULT == 'discovery'


def test_patterns_staleness_longer_than_discovery():
    # Patterns are durable; discovery entries go stale faster
    assert GARDEN_TYPES['patterns']['staleness_default'] > \
           GARDEN_TYPES['discovery']['staleness_default']


def test_risk_staleness_longer_than_evolution():
    # Failure modes are durable; evolution entries track changing versions
    assert GARDEN_TYPES['risk']['staleness_default'] > \
           GARDEN_TYPES['evolution']['staleness_default']
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/mdproctor/claude/hortora/soredium
python -m pytest tests/test_garden_types.py -v 2>&1 | head -30
```

Expected: `ImportError: cannot import name 'GARDEN_TYPES'`

- [ ] **Step 3: Add GARDEN_TYPES to validate_pr.py**

Insert after the `BONUS_RULES` dict (after line 51):

```python
GARDEN_DEFAULT = 'discovery'

GARDEN_TYPES = {
    'discovery': {
        'valid_types': ['gotcha', 'technique', 'undocumented'],
        'required_extra': [],
        'staleness_default': 730,
    },
    'patterns': {
        'valid_types': ['architectural', 'migration', 'integration', 'testing'],
        'required_extra': [],
        'staleness_default': 3650,
    },
    'examples': {
        'valid_types': ['code'],
        'required_extra': [],
        'staleness_default': 1095,
    },
    'evolution': {
        'valid_types': ['breaking', 'deprecation', 'capability'],
        'required_extra': ['changed_in'],
        'staleness_default': 1095,
    },
    'risk': {
        'valid_types': ['failure-mode', 'antipattern', 'incident'],
        'required_extra': ['severity'],
        'staleness_default': 1825,
    },
    'decisions': {
        'valid_types': ['architecture', 'technology', 'process'],
        'required_extra': [],
        'staleness_default': 3650,
    },
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python -m pytest tests/test_garden_types.py -v
```

Expected: all 14 tests PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/validate_pr.py tests/test_garden_types.py
git commit -m "feat(validate_pr): add GARDEN_TYPES registry

Defines 6 garden types with valid entry types, required fields,
and staleness defaults. Foundation for per-garden validation.

Refs #32"
```

---

## Task 2: Validate garden field and entry type against GARDEN_TYPES (TDD)

**Files:**
- Modify: `scripts/validate_pr.py` (`validate()` function)
- Modify: `tests/test_garden_types.py` (append new tests)

- [ ] **Step 1: Write failing tests**

Append to `tests/test_garden_types.py`:

```python
from validate_pr import validate
import tempfile, os


def _write_entry(content: str) -> str:
    """Write entry to a temp file and return its path."""
    f = tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False)
    f.write(content)
    f.close()
    return f.name


DISCOVERY_ENTRY = """\
---
title: "Test gotcha"
garden: discovery
type: gotcha
domain: jvm
score: 10
tags: [java]
verified: true
staleness_threshold: 730
---

## Problem
Something breaks.

### Root cause
The cause.

### Fix
The fix.

### Why this is non-obvious
The insight.
"""

PATTERNS_ENTRY = """\
---
title: "Pluggable Evaluator Pattern"
garden: patterns
type: architectural
domain: jvm
score: 10
tags: [java, architecture]
verified: true
staleness_threshold: 3650
---

## Pattern
Description here.
"""

EVOLUTION_ENTRY = """\
---
title: "Quarkus 3.x breaks @ApplicationScoped on startup"
garden: evolution
type: breaking
domain: jvm
score: 10
tags: [quarkus]
verified: true
staleness_threshold: 1095
changed_in: "3.0.0"
---

## Change
What changed.
"""

RISK_ENTRY = """\
---
title: "Connection pool exhaustion under exception storm"
garden: risk
type: failure-mode
domain: jvm
score: 10
tags: [jdbc, connection-pool]
verified: true
staleness_threshold: 1825
severity: high
---

## Failure mode
Description.
"""


def test_valid_discovery_entry_passes():
    path = _write_entry(DISCOVERY_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_valid_patterns_entry_passes():
    path = _write_entry(PATTERNS_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_valid_evolution_entry_passes():
    path = _write_entry(EVOLUTION_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_valid_risk_entry_passes():
    path = _write_entry(RISK_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_unknown_garden_fails():
    entry = DISCOVERY_ENTRY.replace('garden: discovery', 'garden: unknown-garden')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert any('garden' in c.lower() for c in result['criticals'])
    finally:
        os.unlink(path)


def test_type_invalid_for_garden_fails():
    # 'gotcha' is valid for discovery but not for patterns
    entry = PATTERNS_ENTRY.replace('type: architectural', 'type: gotcha')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert any('type' in c.lower() for c in result['criticals'])
    finally:
        os.unlink(path)


def test_evolution_missing_changed_in_fails():
    entry = EVOLUTION_ENTRY.replace('changed_in: "3.0.0"\n', '')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert any('changed_in' in c for c in result['criticals'])
    finally:
        os.unlink(path)


def test_risk_missing_severity_fails():
    entry = RISK_ENTRY.replace('severity: high\n', '')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert any('severity' in c for c in result['criticals'])
    finally:
        os.unlink(path)


def test_no_garden_field_defaults_to_discovery():
    # Existing entries without garden field — backward compatible
    entry = DISCOVERY_ENTRY.replace('garden: discovery\n', '')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_gotcha_type_valid_without_garden_field():
    # Backward compat: gotcha is valid when garden defaults to discovery
    entry = DISCOVERY_ENTRY.replace('garden: discovery\n', '')
    path = _write_entry(entry)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python -m pytest tests/test_garden_types.py::test_valid_discovery_entry_passes \
                 tests/test_garden_types.py::test_unknown_garden_fails \
                 tests/test_garden_types.py::test_type_invalid_for_garden_fails -v
```

Expected: FAIL — validate() doesn't check garden field yet

- [ ] **Step 3: Add garden validation to validate() in validate_pr.py**

In the `validate()` function, after the `REQUIRED_FIELDS` check block (after the score check, before the injection check), add:

```python
    # Garden type validation
    garden = fm.get('garden', GARDEN_DEFAULT)
    if garden not in GARDEN_TYPES:
        result['criticals'].append(
            f"Unknown garden '{garden}'. Valid: {sorted(GARDEN_TYPES)}"
        )
        return result

    garden_cfg = GARDEN_TYPES[garden]

    # Entry type must be valid for this garden
    entry_type = fm.get('type', '')
    if entry_type not in garden_cfg['valid_types']:
        result['criticals'].append(
            f"Type '{entry_type}' invalid for {garden}-garden. "
            f"Valid: {garden_cfg['valid_types']}"
        )

    # Garden-specific required fields
    for field in garden_cfg['required_extra']:
        if field not in fm:
            result['criticals'].append(
                f"Missing required field for {garden}-garden: '{field}'"
            )
```

- [ ] **Step 4: Run all garden type tests**

```bash
python -m pytest tests/test_garden_types.py -v
```

Expected: all tests PASS

- [ ] **Step 5: Run full test suite to check for regressions**

```bash
python -m pytest tests/ -v --tb=short 2>&1 | tail -20
```

Expected: all previously passing tests still PASS

- [ ] **Step 6: Commit**

```bash
git add scripts/validate_pr.py tests/test_garden_types.py
git commit -m "feat(validate_pr): garden field + per-garden type validation

- validate garden field against GARDEN_TYPES registry
- validate entry type against per-garden allowed types
- validate garden-specific required fields (changed_in, severity)
- backward compatible: absent garden defaults to 'discovery'

Refs #32"
```

---

## Task 3: Happy path correctness tests for all 6 garden types

**Files:**
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write correctness tests for remaining garden types**

Append to `tests/test_garden_types.py`:

```python
EXAMPLES_ENTRY = """\
---
title: "Multi-datasource Quarkus configuration"
garden: examples
type: code
domain: jvm
score: 9
tags: [quarkus, datasource]
verified: true
staleness_threshold: 1095
---

## Example

```java
@DataSource("users")
DataSource usersDs;
```
"""

DECISIONS_ENTRY = """\
---
title: "Vert.x over Netty as Quarkus reactive engine"
garden: decisions
type: architecture
domain: jvm
score: 10
tags: [quarkus, vertx, netty]
verified: true
staleness_threshold: 3650
---

## Decision
Chose Vert.x.

### Alternatives considered
- Netty: lower-level, more boilerplate
"""


def test_valid_examples_entry_passes():
    path = _write_entry(EXAMPLES_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_valid_decisions_entry_passes():
    path = _write_entry(DECISIONS_ENTRY)
    try:
        result = validate(path)
        assert result['criticals'] == [], result['criticals']
    finally:
        os.unlink(path)


def test_all_six_gardens_happy_path():
    """Each garden type must produce a passing validation with a valid entry."""
    entries = {
        'discovery': DISCOVERY_ENTRY,
        'patterns': PATTERNS_ENTRY,
        'examples': EXAMPLES_ENTRY,
        'evolution': EVOLUTION_ENTRY,
        'risk': RISK_ENTRY,
        'decisions': DECISIONS_ENTRY,
    }
    for garden, content in entries.items():
        path = _write_entry(content)
        try:
            result = validate(path)
            assert result['criticals'] == [], \
                f"{garden}-garden entry failed: {result['criticals']}"
        finally:
            os.unlink(path)


def test_each_invalid_type_rejected_per_garden():
    """Types from other gardens are rejected."""
    cross_type_cases = [
        ('patterns', 'gotcha'),       # discovery type in patterns
        ('examples', 'architectural'), # patterns type in examples
        ('evolution', 'code'),         # examples type in evolution
        ('risk', 'breaking'),          # evolution type in risk
        ('decisions', 'failure-mode'), # risk type in decisions
        ('discovery', 'architecture'), # decisions type in discovery
    ]
    base_entries = {
        'patterns': PATTERNS_ENTRY,
        'examples': EXAMPLES_ENTRY,
        'evolution': EVOLUTION_ENTRY,
        'risk': RISK_ENTRY,
        'decisions': DECISIONS_ENTRY,
        'discovery': DISCOVERY_ENTRY,
    }
    for garden, wrong_type in cross_type_cases:
        original_type = GARDEN_TYPES[garden]['valid_types'][0]
        entry = base_entries[garden].replace(
            f'type: {original_type}', f'type: {wrong_type}'
        )
        path = _write_entry(entry)
        try:
            result = validate(path)
            assert any('type' in c.lower() for c in result['criticals']), \
                f"{garden}-garden accepted invalid type '{wrong_type}'"
        finally:
            os.unlink(path)
```

- [ ] **Step 2: Run new tests**

```bash
python -m pytest tests/test_garden_types.py -v -k "happy_path or decisions or examples or invalid_type"
```

Expected: all PASS

- [ ] **Step 3: Run full suite**

```bash
python -m pytest tests/ -v --tb=short 2>&1 | tail -20
```

Expected: all PASS, no regressions

- [ ] **Step 4: Commit**

```bash
git add tests/test_garden_types.py
git commit -m "test(garden_types): happy path + cross-type rejection for all 6 gardens

Refs #32"
```

---

## Task 4: Update VALID_ENTRY fixture in test_validate_pr.py

**Files:**
- Modify: `tests/test_validate_pr.py`

- [ ] **Step 1: Update the VALID_ENTRY fixture to include garden field**

In `tests/test_validate_pr.py`, find `VALID_ENTRY` and add `garden: discovery` after `type: gotcha`:

```python
VALID_ENTRY = """\
---
title: "Quarkus CDI: @UnlessBuildProfile fails in consumers"
garden: discovery
type: gotcha
domain: quarkus/cdi
score: 13
tags: [quarkus, cdi, build-profile]
verified: 2026-04-09
staleness_threshold: 180
summary: "@UnlessBuildProfile causes Unsatisfied dependency when consumed externally"
---

## Problem
Detail here.

## Fix
Fix here.
"""
```

- [ ] **Step 2: Run existing validate_pr tests**

```bash
python -m pytest tests/test_validate_pr.py -v --tb=short 2>&1 | tail -30
```

Expected: all PASS (garden field accepted, backward-compatible entries still pass)

- [ ] **Step 3: Commit**

```bash
git add tests/test_validate_pr.py
git commit -m "test(validate_pr): add garden field to VALID_ENTRY fixture

Refs #32"
```

---

## Task 5: Integration test — CLI validation for each garden type

**Files:**
- Modify: `tests/test_garden_types.py` (append)

- [ ] **Step 1: Write CLI integration tests**

Append to `tests/test_garden_types.py`:

```python
import subprocess
import json
import tempfile
import os

SCRIPT = str(Path(__file__).parent.parent / 'scripts' / 'validate_pr.py')


def _validate_cli(content: str) -> dict:
    f = tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False)
    f.write(content)
    f.close()
    try:
        result = subprocess.run(
            ['python3', SCRIPT, f.name],
            capture_output=True, text=True
        )
        return json.loads(result.stdout)
    finally:
        os.unlink(f.name)


def test_cli_discovery_entry_exits_zero():
    result = subprocess.run(
        ['python3', SCRIPT],
        input=None, capture_output=True
    )
    # Just testing the script is importable via CLI; full test uses file path
    f = tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False)
    f.write(DISCOVERY_ENTRY)
    f.close()
    try:
        r = subprocess.run(['python3', SCRIPT, f.name], capture_output=True, text=True)
        data = json.loads(r.stdout)
        assert r.returncode == 0, f"Expected exit 0, got {r.returncode}: {data}"
        assert data['criticals'] == []
    finally:
        os.unlink(f.name)


def test_cli_invalid_garden_exits_one():
    entry = DISCOVERY_ENTRY.replace('garden: discovery', 'garden: bogus')
    f = tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False)
    f.write(entry)
    f.close()
    try:
        r = subprocess.run(['python3', SCRIPT, f.name], capture_output=True, text=True)
        assert r.returncode == 1
        data = json.loads(r.stdout)
        assert any('garden' in c.lower() for c in data['criticals'])
    finally:
        os.unlink(f.name)


def test_cli_all_six_gardens_exit_zero():
    entries = [
        DISCOVERY_ENTRY, PATTERNS_ENTRY, EXAMPLES_ENTRY,
        EVOLUTION_ENTRY, RISK_ENTRY, DECISIONS_ENTRY,
    ]
    for content in entries:
        f = tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False)
        f.write(content)
        f.close()
        try:
            r = subprocess.run(['python3', SCRIPT, f.name], capture_output=True, text=True)
            data = json.loads(r.stdout)
            assert r.returncode == 0, f"CLI failed for entry: {data['criticals']}"
        finally:
            os.unlink(f.name)
```

- [ ] **Step 2: Run integration tests**

```bash
python -m pytest tests/test_garden_types.py -v -k "cli"
```

Expected: all 3 PASS

- [ ] **Step 3: Run full suite**

```bash
python -m pytest tests/ -v --tb=short 2>&1 | tail -20
```

Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git add tests/test_garden_types.py
git commit -m "test(garden_types): CLI integration tests for all 6 garden types

Refs #32"
```

---

## Task 6: Update submission-formats.md with all 6 garden templates

**Files:**
- Modify: `forage/submission-formats.md`

- [ ] **Step 1: Add garden field to existing discovery template**

Find the existing gotcha/technique/undocumented YAML template in `forage/submission-formats.md` and add `garden: discovery` as the first field after `id`:

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: discovery
title: "Short descriptive title"
type: gotcha | technique | undocumented
domain: jvm | python | tools | web | data | infrastructure | security | cloud
...
```

- [ ] **Step 2: Append templates for patterns, examples, evolution, risk, decisions**

Append to `forage/submission-formats.md`:

````markdown
---

## patterns-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: patterns
title: "Pattern name"
type: architectural | migration | integration | testing
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Technology, Version"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 3650
submitted: YYYY-MM-DD
---

## Pattern

[One paragraph: what problem this solves and why it works]

## Structure

[Description of the structural elements]

## Suitability

[When to use. When not to use.]

## Variants

### Variant: [name]
[Description and tradeoffs]
```

**Editorial bar:** Would a practitioner facing this class of problem reach for something more complex or less elegant without this pattern?

---

## examples-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: examples
title: "Intent — e.g. 'Multi-datasource Quarkus with named injection'"
type: code
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Technology, Version"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 1095
submitted: YYYY-MM-DD
---

## Example

[One sentence of intent]

```java
// Minimal working example — copy-paste ready
```

## Notes

[Optional: what to watch out for, what this doesn't cover]
```

**Editorial bar:** Minimal, working, demonstrating a real use case — not a toy?

---

## evolution-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: evolution
title: "Library X.Y — what changed and what breaks"
type: breaking | deprecation | capability
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Library, Version range"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 1095
changed_in: "X.Y.Z"        # required — version where this change landed
breaking: true              # required for type: breaking
migration_effort: low | medium | high
submitted: YYYY-MM-DD
---

## What Changed

[What the library did before and what it does now]

## Impact

[What breaks. What needs to change in consumer code.]

## Migration

[Step-by-step if migration_effort is medium or high]
```

**Editorial bar:** Does this describe a breaking change, deprecation, or capability shift that would change code correctness for someone on that version?

---

## risk-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: risk
title: "Failure mode name"
type: failure-mode | antipattern | incident
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Technology, Version"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 1825
severity: low | medium | high | critical   # required
failure_pattern: "brief pattern name"
observed_at_scale: true | false
submitted: YYYY-MM-DD
---

## Failure Mode

[What goes wrong and how it manifests]

## Root Cause

[Mechanism — not just what breaks, but why]

## Mitigation

[How to prevent or recover]

## Detection

[How to know this is happening]
```

**Editorial bar:** Has this failure mode caused production harm at meaningful scale, and is the mechanism universal enough to recur in other systems?

---

## decisions-garden Entry Template

```yaml
---
id: GE-YYYYMMDD-xxxxxx
garden: decisions
title: "Technology/approach X chosen over Y for Z"
type: architecture | technology | process
domain: jvm | python | tools | web | data | infrastructure | security | cloud
stack: "Technology, Version"
tags: [tag1, tag2]
score: N
verified: true
staleness_threshold: 3650
submitted: YYYY-MM-DD
---

## Decision

[What was chosen and a one-sentence summary of why]

## Context

[What problem was being solved. What constraints applied.]

## Alternatives Considered

- **[Alternative A]:** [Why rejected]
- **[Alternative B]:** [Why rejected]

## Reasoning

[The argument for the chosen approach over the alternatives]

## Consequences

[What this decision makes easier. What it makes harder.]
```

**Editorial bar:** Does this capture the reasoning clearly enough that someone facing the same choice could apply it — including what was rejected and why?
````

- [ ] **Step 2: Verify the file renders correctly**

```bash
python3 -c "
with open('forage/submission-formats.md') as f:
    content = f.read()
for garden in ['discovery', 'patterns', 'examples', 'evolution', 'risk', 'decisions']:
    assert garden in content, f'Missing {garden} template'
print('All 6 garden templates present')
"
```

Expected: `All 6 garden templates present`

- [ ] **Step 3: Commit**

```bash
git add forage/submission-formats.md
git commit -m "docs(submission-formats): add templates for all 6 garden types

Each template includes YAML frontmatter, body structure, and
the one-sentence editorial bar for that garden type.

Refs #32"
```

---

## Task 7: Update forage SKILL.md — garden selection and editorial bars

**Files:**
- Modify: `forage/SKILL.md`

- [ ] **Step 1: Update Step 4 (Determine the domain) to add garden selection**

Find `**Step 4 — Determine the domain**` in `forage/SKILL.md`. Replace the existing step with:

```markdown
**Step 4 — Determine the garden and domain**

First, select the garden based on knowledge type:

| Knowledge type | Garden |
|---------------|--------|
| Non-obvious behaviour, silent failure, undocumented feature | `discovery` |
| Reusable architectural or structural solution | `patterns` |
| Minimal working code example, copy-paste ready | `examples` |
| Breaking change, deprecation, or new capability in a version | `evolution` |
| Production failure mode, anti-pattern, or incident pattern | `risk` |
| Architectural or technology choice with alternatives considered | `decisions` |

**Editorial bar per garden:**

| Garden | Bar |
|--------|-----|
| `discovery` | Would a skilled developer familiar with the technology still have spent significant time on this? |
| `patterns` | Would a practitioner reach for something more complex or less elegant without this pattern? |
| `examples` | Is this minimal, working, and demonstrating a real use case — not a toy? |
| `evolution` | Does this describe a breaking change, deprecation, or capability shift that would change code correctness for someone on that version? |
| `risk` | Has this failure mode caused production harm at meaningful scale, and is the mechanism universal enough to recur? |
| `decisions` | Does this capture the reasoning clearly enough that someone facing the same choice could apply it — including what was rejected and why? |

Then, select the coarse domain (Qdrant partition key):

| Technology | Domain |
|-----------|--------|
| AppKit, WKWebView, NSTextField | `macos-native-appkit` |
| Panama FFM, jextract, GraalVM native | `java-panama-ffm` |
| Quarkus, Java, Drools, JVM | `jvm` |
| Python | `python` |
| Git, tmux, Docker, CLI tools, cross-cutting patterns | `tools` |
| Web, frontend, Node.js | `web` |
| Databases, data pipelines | `data` |
| Cloud platforms, Kubernetes | `cloud` |
| Security tooling | `security` |
| Doesn't fit existing | `<new-descriptive-dir>` |

Include `garden: <garden>` as the first field in the entry frontmatter.
```

- [ ] **Step 2: Update the Common Pitfalls table**

Find the `| SWEEP: only checking gotchas |` row and add after it:

```markdown
| CAPTURE: omitting garden field | New entries need explicit garden declaration | Always include `garden: <type>` in frontmatter |
| CAPTURE: using gotcha/technique/undocumented in non-discovery garden | Type vocabulary is per-garden | Check GARDEN_TYPES: patterns uses architectural/migration/integration/testing |
```

- [ ] **Step 3: Update Success Criteria for CAPTURE**

Find the CAPTURE success criteria and add:

```markdown
- ✅ `garden` field declared in YAML frontmatter
- ✅ `type` value is valid for the declared garden
- ✅ Garden-specific required fields present (evolution: `changed_in`; risk: `severity`)
```

- [ ] **Step 4: Verify skill file consistency**

```bash
python3 -c "
with open('forage/SKILL.md') as f:
    content = f.read()
for garden in ['discovery', 'patterns', 'examples', 'evolution', 'risk', 'decisions']:
    assert garden in content, f'Missing {garden} in SKILL.md'
print('All 6 garden types referenced in SKILL.md')
"
```

Expected: `All 6 garden types referenced in SKILL.md`

- [ ] **Step 5: Commit**

```bash
git add forage/SKILL.md
git commit -m "feat(forage): garden selection + editorial bars for all 6 garden types

Step 4 now guides garden selection before domain selection.
Per-garden editorial bars and type vocabulary documented.
Success criteria updated.

Refs #32"
```

---

## Task 8: Sync installed skill and close issue

- [ ] **Step 1: Run full test suite — must be green**

```bash
python -m pytest tests/ -v --tb=short 2>&1 | tail -30
```

Expected: all tests PASS. Do not proceed if any fail.

- [ ] **Step 2: Sync forage skill to installed location**

```bash
echo "Y" | python3 scripts/claude-skill sync-local
```

Expected: `✅ forage` and `✅ harvest` synced

- [ ] **Step 3: Close issue with final commit**

```bash
git add -A
git status  # confirm nothing untracked
git commit -m "feat(taxonomy): Area 2 Phase 1 complete — 6-garden schema live

All 6 garden types validated in validate_pr.py. Submission templates
and forage skill updated. 139+ tests passing.

Closes #32
Refs #31"
```

- [ ] **Step 4: Push**

```bash
git push origin main
```

---

## Test Coverage Summary

| Test file | Tests added | What they cover |
|-----------|-------------|-----------------|
| `tests/test_garden_types.py` | ~30 | GARDEN_TYPES structure; per-garden type validation; happy path for all 6; cross-type rejection; CLI integration; backward compat |
| `tests/test_validate_pr.py` | 0 new, fixture updated | Existing tests pass with garden field |

**Test types per requirement:**
- **Unit:** GARDEN_TYPES structure, validate() garden logic
- **Correctness:** Each garden type accepts valid entries, rejects invalid types
- **Integration:** CLI validates each garden type, exits correctly
- **Happy path:** All 6 gardens produce passing validation with canonical entries
- **Backward compat:** Entries without `garden` field still pass (defaults to discovery)
