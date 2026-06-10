# Phase 4 — Full Deduplication System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Complete the deduplication system with auto-incrementing drift counter, `--dedupe-check` threshold enforcement, and a Jaccard pre-scoring scanner that prioritises duplicate candidates and owns CHECKED.md state.

**Architecture:** Approach A+B+C hybrid. `integrate_entry.py` increments GARDEN.md drift counter on every merge. `validate_garden.py --dedupe-check` reads the counter and cross-verifies against git log (git-log wins when higher). New `dedupe_scanner.py` scans domain directories for YAML-frontmatter entries, computes Jaccard similarity for unchecked within-domain pairs, and manages CHECKED.md via `--record`. harvest DEDUPE calls the scanner instead of manually enumerating pairs.

**Tech Stack:** Python 3.11+, re, subprocess, pathlib, pytest. All tests use temporary directories or git repos — the real garden is never touched.

---

## Task 1: Drift counter in integrate_entry.py (TDD)

**Files:**
- Modify: `soredium/scripts/integrate_entry.py`
- Modify: `soredium/tests/test_integrate_entry.py`

- [ ] **Step 1: Write failing tests**

Add to `soredium/tests/test_integrate_entry.py` after the existing imports and constants:

```python
import re


def _garden_with_drift(tmp_path, drift: int = 0) -> Path:
    """Fixture: minimal garden with GARDEN.md drift counter."""
    domain = tmp_path / 'quarkus' / 'cdi'
    domain.mkdir(parents=True)
    (domain / 'INDEX.md').write_text(
        '| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n'
    )
    (tmp_path / '_index').mkdir()
    (tmp_path / '_index' / 'global.md').write_text(
        '| Domain | Index |\n|--------|-------|\n'
    )
    (tmp_path / 'GARDEN.md').write_text(
        f'**Last legacy ID:** GE-0001\n'
        f'**Last full DEDUPE sweep:** 2026-04-14\n'
        f'**Entries merged since last sweep:** {drift}\n'
        f'**Drift threshold:** 10\n'
    )
    return tmp_path


class TestDriftCounter:

    def _run(self, entry, garden):
        with patch('integrate_entry.run_validate'), \
             patch('integrate_entry.git_commit'):
            return integrate(str(entry), str(garden))

    def test_drift_counter_increments_from_zero(self, tmp_path):
        garden = _garden_with_drift(tmp_path, drift=0)
        entry = garden / 'quarkus' / 'cdi' / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        self._run(entry, garden)
        content = (garden / 'GARDEN.md').read_text()
        m = re.search(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', content)
        assert m and int(m.group(1)) == 1

    def test_drift_counter_increments_from_nonzero(self, tmp_path):
        garden = _garden_with_drift(tmp_path, drift=3)
        entry = garden / 'quarkus' / 'cdi' / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        self._run(entry, garden)
        content = (garden / 'GARDEN.md').read_text()
        m = re.search(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', content)
        assert m and int(m.group(1)) == 4

    def test_drift_counter_missing_field_no_crash(self, tmp_path):
        domain = tmp_path / 'quarkus' / 'cdi'
        domain.mkdir(parents=True)
        (domain / 'INDEX.md').write_text('| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n')
        (tmp_path / '_index').mkdir()
        (tmp_path / '_index' / 'global.md').write_text('| Domain | Index |\n|--------|-------|\n')
        (tmp_path / 'GARDEN.md').write_text('**Last legacy ID:** GE-0001\n')
        entry = domain / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        result = self._run(entry, tmp_path)
        assert result['status'] == 'ok'

    def test_no_garden_md_no_crash(self, tmp_path):
        domain = tmp_path / 'quarkus' / 'cdi'
        domain.mkdir(parents=True)
        (domain / 'INDEX.md').write_text('| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n')
        (tmp_path / '_index').mkdir()
        (tmp_path / '_index' / 'global.md').write_text('| Domain | Index |\n|--------|-------|\n')
        entry = domain / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        result = self._run(entry, tmp_path)
        assert result['status'] == 'ok'

    def test_other_garden_md_fields_preserved(self, tmp_path):
        garden = _garden_with_drift(tmp_path, drift=0)
        entry = garden / 'quarkus' / 'cdi' / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        self._run(entry, garden)
        content = (garden / 'GARDEN.md').read_text()
        assert '**Last legacy ID:** GE-0001' in content
        assert '**Drift threshold:** 10' in content
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integrate_entry.py::TestDriftCounter -v
```

Expected: 5 tests FAIL with `AttributeError` or similar — `increment_drift_counter` not defined yet.

- [ ] **Step 3: Implement increment_drift_counter**

In `soredium/scripts/integrate_entry.py`, add after the `import subprocess` line:

```python
import re
```

Then add this function after `run_validate()`:

```python
def increment_drift_counter(garden: Path):
    """Increment 'Entries merged since last sweep' in GARDEN.md by 1."""
    garden_md = garden / 'GARDEN.md'
    if not garden_md.exists():
        return
    content = garden_md.read_text(encoding='utf-8')
    def _inc(m):
        return f'**Entries merged since last sweep:** {int(m.group(1)) + 1}'
    new_content = re.sub(
        r'\*\*Entries merged since last sweep:\*\*\s*(\d+)',
        _inc,
        content
    )
    if new_content != content:
        garden_md.write_text(new_content, encoding='utf-8')
```

Then in the `integrate()` function, add the call after `update_global_index(...)` and before `run_validate(...)`:

```python
    update_global_index(domain, garden)
    increment_drift_counter(garden)   # ← add this line
    run_validate(garden)
```

Also add `'_summaries/', '_index/', 'labels/', 'GARDEN.md'` to the git_commit staging:

```python
def git_commit(garden: Path, ge_id: str):
    subprocess.run(['git', '-C', str(garden), 'add', '_summaries/', '_index/', 'labels/', 'GARDEN.md'], check=True)
    subprocess.run(['git', '-C', str(garden), 'add', '--update'], check=True)
    subprocess.run(
        ['git', '-C', str(garden), 'commit', '-m', f'index: integrate {ge_id}'],
        check=True
    )
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integrate_entry.py -v
```

Expected: All tests PASS including the 5 new TestDriftCounter tests.

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/integrate_entry.py tests/test_integrate_entry.py
git commit -m "feat(integrate): auto-increment drift counter on every entry integration"
```

---

## Task 2: --dedupe-check in validate_garden.py (TDD)

**Files:**
- Modify: `soredium/scripts/validate_garden.py`
- Modify: `soredium/tests/test_validate_garden.py`

- [ ] **Step 1: Write failing tests**

Add to `soredium/tests/test_validate_garden.py`:

```python
import subprocess as _subprocess


def run_dedupe_check(garden_root: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(VALIDATOR), '--dedupe-check', str(garden_root)],
        capture_output=True, text=True
    )


def _garden_with_drift(root: Path, drift: int, threshold: int = 10) -> Path:
    (root / 'GARDEN.md').write_text(
        f'**Last legacy ID:** GE-0001\n'
        f'**Last full DEDUPE sweep:** 2026-04-14\n'
        f'**Entries merged since last sweep:** {drift}\n'
        f'**Drift threshold:** {threshold}\n'
    )
    return root


class TestDedupeCheck(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_drift_below_threshold_exits_0(self):
        _garden_with_drift(self.root, drift=3, threshold=10)
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn('OK', result.stdout)

    def test_drift_at_threshold_exits_2(self):
        _garden_with_drift(self.root, drift=10, threshold=10)
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 2, result.stdout + result.stderr)
        self.assertIn('DEDUPE recommended', result.stdout)

    def test_drift_above_threshold_exits_2(self):
        _garden_with_drift(self.root, drift=15, threshold=10)
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 2, result.stdout + result.stderr)

    def test_drift_just_below_threshold_exits_0(self):
        _garden_with_drift(self.root, drift=9, threshold=10)
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 0)

    def test_output_shows_drift_and_threshold(self):
        _garden_with_drift(self.root, drift=5, threshold=10)
        result = run_dedupe_check(self.root)
        self.assertIn('drift=5', result.stdout)
        self.assertIn('threshold=10', result.stdout)

    def test_custom_threshold_respected(self):
        _garden_with_drift(self.root, drift=3, threshold=3)
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 2)

    def test_no_garden_md_exits_0(self):
        # No GARDEN.md → drift=0, threshold=10 → OK
        result = run_dedupe_check(self.root)
        self.assertEqual(result.returncode, 0)


class TestDedupeCheckGitLog(unittest.TestCase):
    """Git-log cross-verification: git-log count wins when higher than counter."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        subprocess.run(['git', 'init', str(self.root)], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.root), 'config', 'user.email', 'test@test.com'], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.root), 'config', 'user.name', 'Test'], check=True, capture_output=True)

    def tearDown(self):
        self.tmp.cleanup()

    def _commit(self, message: str):
        f = self.root / f'{len(list(self.root.iterdir()))}.txt'
        f.write_text(message)
        subprocess.run(['git', '-C', str(self.root), 'add', '.'], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.root), 'commit', '-m', message], check=True, capture_output=True)

    def test_git_log_count_wins_when_higher(self):
        # Counter says 2, git-log shows 4 integrate commits since last dedupe
        (self.root / 'GARDEN.md').write_text(
            '**Entries merged since last sweep:** 2\n**Drift threshold:** 3\n'
        )
        self._commit('dedupe: baseline sweep')
        for i in range(4):
            self._commit(f'index: integrate GE-000{i}')
        # Counter=2, git-log=4, threshold=3 → git-log (4) >= threshold → exit 2
        result = subprocess.run(
            [sys.executable, str(VALIDATOR), '--dedupe-check', str(self.root)],
            capture_output=True, text=True
        )
        self.assertEqual(result.returncode, 2, result.stdout)

    def test_git_log_does_not_override_higher_counter(self):
        # Counter says 8, git-log shows only 2 → counter (8) used, below threshold=10
        (self.root / 'GARDEN.md').write_text(
            '**Entries merged since last sweep:** 8\n**Drift threshold:** 10\n'
        )
        self._commit('dedupe: baseline sweep')
        for i in range(2):
            self._commit(f'index: integrate GE-000{i}')
        result = subprocess.run(
            [sys.executable, str(VALIDATOR), '--dedupe-check', str(self.root)],
            capture_output=True, text=True
        )
        self.assertEqual(result.returncode, 0, result.stdout)
        self.assertIn('drift=8', result.stdout)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py::TestDedupeCheck tests/test_validate_garden.py::TestDedupeCheckGitLog -v
```

Expected: All 9 tests FAIL — `--dedupe-check` not implemented.

- [ ] **Step 3: Implement --dedupe-check**

In `soredium/scripts/validate_garden.py`, add the following block after the `--freshness` block and before `import os` (around line 107):

```python
if '--dedupe-check' in sys.argv:
    import os as _os
    import re as _re
    import subprocess as _sp

    _idx = sys.argv.index('--dedupe-check')
    if _idx + 1 < len(sys.argv) and not sys.argv[_idx + 1].startswith('--'):
        _garden = Path(sys.argv[_idx + 1]).expanduser().resolve()
    elif 'HORTORA_GARDEN' in _os.environ:
        _garden = Path(_os.environ['HORTORA_GARDEN']).expanduser().resolve()
    else:
        _garden = (Path.home() / '.hortora' / 'garden').resolve()

    _drift = 0
    _threshold = 10
    _garden_md = _garden / 'GARDEN.md'
    if _garden_md.exists():
        _content = _garden_md.read_text()
        _m = _re.search(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', _content)
        if _m:
            _drift = int(_m.group(1))
        _t = _re.search(r'\*\*Drift threshold:\*\*\s*(\d+)', _content)
        if _t:
            _threshold = int(_t.group(1))

    # Cross-check with git log — count 'index: integrate' commits since last 'dedupe:' commit
    try:
        _log = _sp.run(
            ['git', '-C', str(_garden), 'log', '--format=%s'],
            capture_output=True, text=True, check=True
        ).stdout.strip().splitlines()
        _git_count = 0
        for _line in _log:
            if _line.startswith('dedupe:'):
                break
            if _line.startswith('index: integrate'):
                _git_count += 1
        if _git_count > _drift:
            _drift = _git_count
    except Exception:
        pass  # not a git repo or git unavailable — use counter only

    if _drift >= _threshold:
        print(f"Dedupe check: drift={_drift}, threshold={_threshold} — DEDUPE recommended")
        sys.exit(2)
    else:
        print(f"Dedupe check: drift={_drift}, threshold={_threshold} — OK")
        sys.exit(0)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py -v
```

Expected: All tests PASS including the 9 new tests.

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_garden.py tests/test_validate_garden.py
git commit -m "feat(validate): add --dedupe-check flag with git-log cross-verification"
```

---

## Task 3: dedupe_scanner.py core functions (TDD — unit tests)

**Files:**
- Create: `soredium/scripts/dedupe_scanner.py`
- Create: `soredium/tests/test_dedupe_scanner.py`

- [ ] **Step 1: Create the test file with failing unit tests**

Create `soredium/tests/test_dedupe_scanner.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for dedupe_scanner.py."""

import sys
import subprocess
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from dedupe_scanner import (
    tokenize, jaccard, canonical_pair, parse_tags,
    load_checked_pairs, record_pair,
    load_entries, compute_pairs, format_text,
)

SCANNER = Path(__file__).parent.parent / 'scripts' / 'dedupe_scanner.py'


def run_scanner(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(SCANNER)] + list(args),
        capture_output=True, text=True
    )


def make_entry(path: Path, ge_id: str, title: str, tags: list, domain: str):
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(
        f'---\nid: {ge_id}\ntitle: "{title}"\ntype: gotcha\n'
        f'domain: {domain}\ntags: [{", ".join(tags)}]\nscore: 10\n'
        f'verified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        f'## {title}\n\n**ID:** {ge_id}\n'
    )


# ── tokenize ──────────────────────────────────────────────────────────────────

class TestTokenize(unittest.TestCase):

    def test_empty_string(self):
        self.assertEqual(tokenize(''), set())

    def test_short_words_excluded(self):
        self.assertEqual(tokenize('is an of to'), set())

    def test_three_char_included(self):
        self.assertIn('the', tokenize('the'))

    def test_lowercase_normalisation(self):
        self.assertEqual(tokenize('Python REGEX'), tokenize('python regex'))

    def test_deduplication(self):
        self.assertEqual(tokenize('regex regex regex'), {'regex'})

    def test_punctuation_stripped(self):
        result = tokenize('parsing, yaml; frontmatter.')
        self.assertIn('parsing', result)
        self.assertIn('yaml', result)
        self.assertIn('frontmatter', result)

    def test_hyphenated_words_split(self):
        result = tokenize('crlf line-ending')
        self.assertIn('crlf', result)
        self.assertIn('line', result)
        self.assertIn('ending', result)

    def test_numbers_excluded(self):
        # Numbers don't match [a-z]{3,}
        result = tokenize('123 456')
        self.assertEqual(result, set())


# ── jaccard ───────────────────────────────────────────────────────────────────

class TestJaccard(unittest.TestCase):

    def test_empty_sets(self):
        self.assertEqual(jaccard(set(), set()), 0.0)

    def test_identical_sets(self):
        s = {'regex', 'yaml', 'parsing'}
        self.assertEqual(jaccard(s, s), 1.0)

    def test_disjoint_sets(self):
        self.assertEqual(jaccard({'regex'}, {'yaml'}), 0.0)

    def test_partial_overlap(self):
        # {aaa, bbb} ∩ {bbb, ccc} = {bbb}, union = {aaa, bbb, ccc} → 1/3
        result = jaccard({'aaa', 'bbb'}, {'bbb', 'ccc'})
        self.assertAlmostEqual(result, 1 / 3, places=4)

    def test_one_empty(self):
        self.assertEqual(jaccard({'regex'}, set()), 0.0)
        self.assertEqual(jaccard(set(), {'regex'}), 0.0)

    def test_formula_correctness(self):
        a = {'one', 'two', 'six'}
        b = {'one', 'two', 'thr', 'fou', 'fiv'}
        expected = len(a & b) / len(a | b)
        self.assertAlmostEqual(jaccard(a, b), expected, places=4)


# ── canonical_pair ────────────────────────────────────────────────────────────

class TestCanonicalPair(unittest.TestCase):

    def test_lower_first(self):
        self.assertEqual(canonical_pair('GE-0002', 'GE-0001'), 'GE-0001 × GE-0002')

    def test_already_ordered(self):
        self.assertEqual(canonical_pair('GE-0001', 'GE-0002'), 'GE-0001 × GE-0002')

    def test_legacy_before_new_format(self):
        # GE-0001 < GE-20260414-aabbcc lexicographically ('0' < '2')
        result = canonical_pair('GE-20260414-aabbcc', 'GE-0001')
        self.assertEqual(result, 'GE-0001 × GE-20260414-aabbcc')

    def test_new_format_ordering(self):
        # Lexicographic within new format
        a = 'GE-20260414-aaaaaa'
        b = 'GE-20260414-bbbbbb'
        self.assertEqual(canonical_pair(b, a), f'{a} × {b}')


# ── parse_tags ────────────────────────────────────────────────────────────────

class TestParseTags(unittest.TestCase):

    def test_basic(self):
        self.assertEqual(parse_tags('regex, yaml, parsing'), ['regex', 'yaml', 'parsing'])

    def test_with_quotes(self):
        self.assertEqual(parse_tags('"regex", "yaml"'), ['regex', 'yaml'])

    def test_empty(self):
        self.assertEqual(parse_tags(''), [])

    def test_single_tag(self):
        self.assertEqual(parse_tags('regex'), ['regex'])


# ── load_checked_pairs ────────────────────────────────────────────────────────

class TestLoadCheckedPairs(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_file(self):
        (self.root / 'CHECKED.md').write_text(
            '| Pair | Result | Date | Notes |\n|------|--------|------|-------|\n'
        )
        self.assertEqual(load_checked_pairs(self.root), set())

    def test_missing_file(self):
        self.assertEqual(load_checked_pairs(self.root), set())

    def test_single_pair(self):
        (self.root / 'CHECKED.md').write_text(
            '| GE-0001 × GE-0002 | distinct | 2026-04-14 | |\n'
        )
        result = load_checked_pairs(self.root)
        self.assertIn('GE-0001 × GE-0002', result)

    def test_both_orderings_recognised(self):
        (self.root / 'CHECKED.md').write_text(
            '| GE-0002 × GE-0001 | distinct | 2026-04-14 | |\n'
        )
        result = load_checked_pairs(self.root)
        self.assertIn('GE-0001 × GE-0002', result)

    def test_malformed_rows_skipped(self):
        (self.root / 'CHECKED.md').write_text(
            'This is not a pair row\n'
            '| GE-0001 × GE-0002 | distinct | 2026-04-14 | |\n'
            'Another bad row\n'
        )
        result = load_checked_pairs(self.root)
        self.assertEqual(len(result), 1)

    def test_multiple_pairs(self):
        (self.root / 'CHECKED.md').write_text(
            '| GE-0001 × GE-0002 | distinct | 2026-04-14 | |\n'
            '| GE-0003 × GE-0004 | related | 2026-04-14 | cross-referenced |\n'
            '| GE-0005 × GE-0006 | duplicate-discarded | 2026-04-14 | GE-0005 kept |\n'
        )
        self.assertEqual(len(load_checked_pairs(self.root)), 3)


# ── record_pair ───────────────────────────────────────────────────────────────

class TestRecordPair(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        (self.root / 'CHECKED.md').write_text(
            '| Pair | Result | Date | Notes |\n|------|--------|------|-------|\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_appends_row(self):
        record_pair(self.root, 'GE-0001 × GE-0002', 'distinct', 'no overlap')
        content = (self.root / 'CHECKED.md').read_text()
        self.assertIn('GE-0001 × GE-0002', content)
        self.assertIn('distinct', content)
        self.assertIn('no overlap', content)

    def test_canonical_ordering_enforced(self):
        record_pair(self.root, 'GE-0002 × GE-0001', 'distinct', '')
        content = (self.root / 'CHECKED.md').read_text()
        self.assertIn('GE-0001 × GE-0002', content)
        self.assertNotIn('GE-0002 × GE-0001', content)

    def test_idempotent(self):
        record_pair(self.root, 'GE-0001 × GE-0002', 'distinct', '')
        record_pair(self.root, 'GE-0001 × GE-0002', 'distinct', '')
        content = (self.root / 'CHECKED.md').read_text()
        self.assertEqual(content.count('GE-0001 × GE-0002'), 1)

    def test_all_valid_results_accepted(self):
        for i, result in enumerate(['distinct', 'related', 'duplicate-discarded']):
            record_pair(self.root, f'GE-000{i+1} × GE-999{i+1}', result, '')
        content = (self.root / 'CHECKED.md').read_text()
        self.assertIn('distinct', content)
        self.assertIn('related', content)
        self.assertIn('duplicate-discarded', content)

    def test_invalid_result_exits_1(self):
        with self.assertRaises(SystemExit) as ctx:
            record_pair(self.root, 'GE-0001 × GE-0002', 'bogus', '')
        self.assertEqual(ctx.exception.code, 1)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py -v 2>&1 | head -30
```

Expected: All tests FAIL with `ModuleNotFoundError: No module named 'dedupe_scanner'`.

- [ ] **Step 3: Create dedupe_scanner.py with core functions**

Create `soredium/scripts/dedupe_scanner.py`:

```python
#!/usr/bin/env python3
"""
dedupe_scanner.py — Jaccard similarity pre-scorer for harvest DEDUPE.

Scans garden domain directories for YAML-frontmatter entries, computes
Jaccard similarity for all unchecked within-domain pairs, and returns
pairs sorted highest-first. Owns CHECKED.md state via --record.

Usage:
  dedupe_scanner.py [garden_root]
  dedupe_scanner.py [garden_root] --domain quarkus
  dedupe_scanner.py [garden_root] --top 20
  dedupe_scanner.py [garden_root] --json
  dedupe_scanner.py [garden_root] --record "GE-X × GE-Y" distinct "note"
"""

import re
import sys
import json
import os
from datetime import date
from pathlib import Path

SKIP_FILES = {'GARDEN.md', 'CHECKED.md', 'DISCARDED.md', 'INDEX.md', 'README.md'}
SKIP_DIRS  = {'.git', '_summaries', '_index', 'labels', 'submissions', 'scripts'}
TOKEN_RE   = re.compile(r'\b[a-z]{3,}\b')


def tokenize(text: str) -> set:
    """Extract 3+ char lowercase word tokens from text."""
    return set(TOKEN_RE.findall(text.lower()))


def jaccard(a: set, b: set) -> float:
    """Jaccard similarity: |A∩B| / |A∪B|. Returns 0.0 if both empty."""
    if not a and not b:
        return 0.0
    return len(a & b) / len(a | b)


def parse_tags(tags_str: str) -> list:
    """Parse comma-separated tag list from frontmatter value string."""
    return [t.strip().strip('"\'') for t in tags_str.split(',') if t.strip()]


def canonical_pair(id_a: str, id_b: str) -> str:
    """Return canonical 'lower × higher' pair string (lexicographic)."""
    lo, hi = (id_a, id_b) if id_a <= id_b else (id_b, id_a)
    return f"{lo} × {hi}"


def load_checked_pairs(garden: Path) -> set:
    """Return set of canonical pair strings already in CHECKED.md."""
    checked_md = garden / 'CHECKED.md'
    if not checked_md.exists():
        return set()
    pairs = set()
    for m in re.finditer(r'\|\s*(GE-[\w-]+)\s*[×x]\s*(GE-[\w-]+)\s*\|',
                         checked_md.read_text(encoding='utf-8')):
        pairs.add(canonical_pair(m.group(1), m.group(2)))
    return pairs


def record_pair(garden: Path, pair: str, result: str, note: str = ''):
    """Append a pair comparison result to CHECKED.md. Idempotent."""
    valid = {'distinct', 'related', 'duplicate-discarded'}
    if result not in valid:
        print(f"ERROR: result must be one of {valid}", file=sys.stderr)
        sys.exit(1)
    m = re.search(r'(GE-[\w-]+)\s*[×x]\s*(GE-[\w-]+)', pair)
    if not m:
        print(f"ERROR: invalid pair format: {pair!r}", file=sys.stderr)
        sys.exit(1)
    canon = canonical_pair(m.group(1), m.group(2))
    if canon in load_checked_pairs(garden):
        return  # already recorded — idempotent
    today = date.today().isoformat()
    row = f"| {canon} | {result} | {today} | {note} |\n"
    with open(garden / 'CHECKED.md', 'a', encoding='utf-8') as f:
        f.write(row)


def load_entries(garden: Path, domain_filter: str = None) -> dict:
    """
    Scan domain directories for entries with YAML frontmatter.
    Returns {domain: [{'id': str, 'title': str, 'tags': list, 'path': Path}]}
    """
    entries_by_domain: dict = {}
    for path in garden.rglob('*.md'):
        if path.name in SKIP_FILES:
            continue
        if any(part in SKIP_DIRS for part in path.relative_to(garden).parts):
            continue
        content = path.read_text(encoding='utf-8').replace('\r\n', '\n')
        fm_match = re.match(r'^---\n(.*?)\n---', content, re.DOTALL)
        if not fm_match:
            continue
        fm = fm_match.group(1)
        id_m = re.search(r'^id:\s*(.+)$', fm, re.MULTILINE)
        if not id_m:
            continue
        ge_id = id_m.group(1).strip()
        title_m = re.search(r'^title:\s*"?(.+?)"?\s*$', fm, re.MULTILINE)
        title = title_m.group(1).strip().strip('"') if title_m else ''
        tags_m = re.search(r'^tags:\s*\[(.+)\]', fm, re.MULTILINE)
        tags = parse_tags(tags_m.group(1)) if tags_m else []
        domain_m = re.search(r'^domain:\s*(.+)$', fm, re.MULTILINE)
        domain = domain_m.group(1).strip() if domain_m else path.parent.name
        if domain_filter and domain != domain_filter:
            continue
        entries_by_domain.setdefault(domain, []).append({
            'id': ge_id, 'title': title, 'tags': tags, 'path': path,
        })
    return entries_by_domain


def compute_pairs(entries_by_domain: dict, checked_pairs: set,
                  top: int = None) -> list:
    """Generate all unchecked within-domain pairs with Jaccard scores, sorted desc."""
    results = []
    for domain, entries in entries_by_domain.items():
        for i in range(len(entries)):
            for j in range(i + 1, len(entries)):
                a, b = entries[i], entries[j]
                pair_key = canonical_pair(a['id'], b['id'])
                if pair_key in checked_pairs:
                    continue
                tokens_a = tokenize(a['title'] + ' ' + ' '.join(a['tags']))
                tokens_b = tokenize(b['title'] + ' ' + ' '.join(b['tags']))
                score = jaccard(tokens_a, tokens_b)
                results.append({
                    'pair': pair_key, 'id_a': a['id'], 'id_b': b['id'],
                    'title_a': a['title'], 'title_b': b['title'],
                    'domain': domain, 'score': round(score, 4),
                })
    results.sort(key=lambda x: x['score'], reverse=True)
    return results[:top] if top is not None else results


def format_text(pairs: list) -> str:
    """Format pairs as human-readable text grouped by domain."""
    if not pairs:
        return "No unchecked pairs found."
    by_domain: dict = {}
    for p in pairs:
        by_domain.setdefault(p['domain'], []).append(p)
    lines = []
    for domain in sorted(by_domain):
        domain_pairs = by_domain[domain]
        lines.append(f"\nDOMAIN: {domain}  ({len(domain_pairs)} unchecked pairs)")
        for p in domain_pairs:
            ta = p['title_a'][:40]
            tb = p['title_b'][:40]
            lines.append(f"  {p['score']:.2f}  {p['pair']}  [{ta} / {tb}]")
    return '\n'.join(lines)


def _default_garden() -> Path:
    env = os.environ.get('HORTORA_GARDEN')
    return Path(env).expanduser().resolve() if env else \
           (Path.home() / '.hortora' / 'garden').resolve()


def main():
    args = sys.argv[1:]
    non_flags = [a for a in args if not a.startswith('--')]
    garden = Path(non_flags[0]).expanduser().resolve() if non_flags else _default_garden()

    if '--record' in args:
        idx = args.index('--record')
        if idx + 2 >= len(args):
            print('ERROR: --record requires <pair> <result> [note]', file=sys.stderr)
            sys.exit(1)
        record_pair(garden, args[idx + 1], args[idx + 2],
                    args[idx + 3] if idx + 3 < len(args) else '')
        sys.exit(0)

    domain_filter = None
    if '--domain' in args:
        i = args.index('--domain')
        domain_filter = args[i + 1] if i + 1 < len(args) else None

    top = None
    if '--top' in args:
        i = args.index('--top')
        try:
            top = int(args[i + 1])
        except (IndexError, ValueError):
            print('ERROR: --top requires an integer', file=sys.stderr)
            sys.exit(1)

    use_json = '--json' in args

    entries_by_domain = load_entries(garden, domain_filter)
    checked_pairs = load_checked_pairs(garden)
    pairs = compute_pairs(entries_by_domain, checked_pairs, top)

    if use_json:
        print(json.dumps(pairs, indent=2))
    else:
        print(format_text(pairs))
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Run unit tests to verify they pass**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py::TestTokenize tests/test_dedupe_scanner.py::TestJaccard tests/test_dedupe_scanner.py::TestCanonicalPair tests/test_dedupe_scanner.py::TestParseTags tests/test_dedupe_scanner.py::TestLoadCheckedPairs tests/test_dedupe_scanner.py::TestRecordPair -v
```

Expected: All 32 unit tests PASS.

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/dedupe_scanner.py tests/test_dedupe_scanner.py
git commit -m "feat(dedupe): add dedupe_scanner.py core functions with full unit test coverage"
```

---

## Task 4: load_entries and compute_pairs integration tests (TDD)

**Files:**
- Modify: `soredium/tests/test_dedupe_scanner.py`

- [ ] **Step 1: Add integration tests for load_entries and compute_pairs**

Add the following classes to `soredium/tests/test_dedupe_scanner.py`:

```python
class TestLoadEntries(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_finds_yaml_entries(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'CRLF regex skip', ['regex', 'yaml'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002', 'fromisoformat crash', ['datetime'], 'python')
        result = load_entries(self.root)
        self.assertIn('python', result)
        self.assertEqual(len(result['python']), 2)

    def test_skips_non_frontmatter_entries(self):
        (self.root / 'tools').mkdir()
        (self.root / 'tools' / 'GE-0042.md').write_text(
            '## Legacy entry\n\n**ID:** GE-0042\n**Stack:** git\n'
        )
        result = load_entries(self.root)
        self.assertEqual(len(result.get('tools', [])), 0)

    def test_skips_index_and_control_files(self):
        (self.root / 'GARDEN.md').write_text('**Last legacy ID:** GE-0001\n')
        (self.root / 'CHECKED.md').write_text('| Pair |\n')
        (self.root / 'DISCARDED.md').write_text('| Pair |\n')
        (self.root / 'tools').mkdir()
        (self.root / 'tools' / 'INDEX.md').write_text('| GE-ID | Title |\n')
        make_entry(self.root / 'tools' / 'GE-0001.md', 'GE-0001', 'Test', ['git'], 'tools')
        result = load_entries(self.root)
        self.assertEqual(len(result.get('tools', [])), 1)

    def test_skips_hidden_dirs(self):
        (self.root / '_summaries').mkdir()
        (self.root / '_summaries' / 'GE-0001.md').write_text(
            '---\nid: GE-0001\ntitle: "Test"\ntype: gotcha\ndomain: tools\n'
            'tags: [git]\nscore: 10\nverified: true\nstaleness_threshold: 730\n'
            'submitted: 2026-04-14\n---\n\n## Test\n'
        )
        result = load_entries(self.root)
        self.assertEqual(result, {})

    def test_domain_filter(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'Python entry', ['regex'], 'python')
        make_entry(self.root / 'tools' / 'GE-0002.md', 'GE-0002', 'Tools entry', ['git'], 'tools')
        result = load_entries(self.root, domain_filter='python')
        self.assertIn('python', result)
        self.assertNotIn('tools', result)

    def test_crlf_files_not_skipped(self):
        path = self.root / 'python' / 'GE-0001.md'
        path.parent.mkdir(parents=True)
        # Write with CRLF line endings
        path.write_bytes(
            b'---\r\nid: GE-0001\r\ntitle: "CRLF test"\r\ntype: gotcha\r\n'
            b'domain: python\r\ntags: [regex]\r\nscore: 10\r\nverified: true\r\n'
            b'staleness_threshold: 730\r\nsubmitted: 2026-04-14\r\n---\r\n\r\n'
            b'## CRLF test\r\n'
        )
        result = load_entries(self.root)
        self.assertIn('python', result)
        self.assertEqual(len(result['python']), 1)


class TestComputePairs(unittest.TestCase):

    def test_no_entries_empty_result(self):
        self.assertEqual(compute_pairs({}, set()), [])

    def test_single_entry_no_pairs(self):
        entries = {'python': [{'id': 'GE-0001', 'title': 'Test', 'tags': ['regex']}]}
        self.assertEqual(compute_pairs(entries, set()), [])

    def test_two_entries_one_pair(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'regex yaml crlf parsing', 'tags': ['regex', 'yaml']},
            {'id': 'GE-0002', 'title': 'regex datetime parsing', 'tags': ['regex']},
        ]}
        pairs = compute_pairs(entries, set())
        self.assertEqual(len(pairs), 1)
        self.assertEqual(pairs[0]['pair'], 'GE-0001 × GE-0002')

    def test_three_entries_three_pairs(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'entry one', 'tags': ['tag']},
            {'id': 'GE-0002', 'title': 'entry two', 'tags': ['tag']},
            {'id': 'GE-0003', 'title': 'entry three', 'tags': ['tag']},
        ]}
        self.assertEqual(len(compute_pairs(entries, set())), 3)

    def test_checked_pairs_excluded(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'one', 'tags': []},
            {'id': 'GE-0002', 'title': 'two', 'tags': []},
        ]}
        checked = {'GE-0001 × GE-0002'}
        self.assertEqual(compute_pairs(entries, checked), [])

    def test_cross_domain_never_paired(self):
        entries = {
            'python': [{'id': 'GE-0001', 'title': 'regex yaml', 'tags': ['regex']}],
            'tools': [{'id': 'GE-0002', 'title': 'regex yaml', 'tags': ['regex']}],
        }
        pairs = compute_pairs(entries, set())
        pair_keys = [p['pair'] for p in pairs]
        self.assertNotIn('GE-0001 × GE-0002', pair_keys)

    def test_sorted_highest_score_first(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'regex yaml crlf parsing frontmatter', 'tags': ['regex', 'yaml', 'crlf']},
            {'id': 'GE-0002', 'title': 'regex yaml crlf parsing frontmatter', 'tags': ['regex', 'yaml', 'crlf']},
            {'id': 'GE-0003', 'title': 'completely different git branch rebase', 'tags': ['git']},
        ]}
        pairs = compute_pairs(entries, set())
        self.assertGreaterEqual(pairs[0]['score'], pairs[-1]['score'])
        # Identical entries score 1.0 and should be first
        self.assertEqual(pairs[0]['score'], 1.0)

    def test_top_limit(self):
        entries = {'python': [
            {'id': f'GE-000{i}', 'title': f'entry {i}', 'tags': ['tag']}
            for i in range(1, 5)
        ]}
        all_pairs = compute_pairs(entries, set())
        self.assertEqual(len(all_pairs), 6)  # 4×3/2 = 6
        top_pairs = compute_pairs(entries, set(), top=3)
        self.assertEqual(len(top_pairs), 3)

    def test_high_score_for_identical_titles_and_tags(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'yaml regex crlf skip silent', 'tags': ['regex', 'yaml', 'crlf']},
            {'id': 'GE-0002', 'title': 'yaml regex crlf skip silent', 'tags': ['regex', 'yaml', 'crlf']},
        ]}
        pairs = compute_pairs(entries, set())
        self.assertEqual(pairs[0]['score'], 1.0)

    def test_low_score_for_unrelated_entries(self):
        entries = {'python': [
            {'id': 'GE-0001', 'title': 'yaml frontmatter regex crlf parsing', 'tags': ['regex', 'yaml']},
            {'id': 'GE-0002', 'title': 'git commit rebase squash branch history', 'tags': ['git', 'workflow']},
        ]}
        pairs = compute_pairs(entries, set())
        self.assertLess(pairs[0]['score'], 0.1)
```

- [ ] **Step 2: Run all dedupe_scanner tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py -v
```

Expected: All tests PASS (core functions were implemented in Task 3).

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/soredium
git add tests/test_dedupe_scanner.py
git commit -m "test(dedupe): add integration tests for load_entries and compute_pairs"
```

---

## Task 5: CLI integration tests for dedupe_scanner (TDD)

**Files:**
- Modify: `soredium/tests/test_dedupe_scanner.py`

- [ ] **Step 1: Add CLI integration tests**

Add to `soredium/tests/test_dedupe_scanner.py`:

```python
class TestDedupesScannerCLI(unittest.TestCase):
    """Integration tests calling dedupe_scanner.py as a subprocess."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        (self.root / 'CHECKED.md').write_text(
            '| Pair | Result | Date | Notes |\n|------|--------|------|-------|\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_no_entries_exits_0(self):
        result = run_scanner(str(self.root))
        self.assertEqual(result.returncode, 0, result.stderr)
        self.assertIn('No unchecked pairs', result.stdout)

    def test_two_similar_entries_appear_in_output(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001',
                   'regex yaml crlf skip silent', ['regex', 'yaml', 'crlf'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002',
                   'regex yaml crlf skip parsing', ['regex', 'yaml'], 'python')
        result = run_scanner(str(self.root))
        self.assertEqual(result.returncode, 0)
        self.assertIn('GE-0001', result.stdout)
        self.assertIn('GE-0002', result.stdout)
        self.assertIn('python', result.stdout.lower())

    def test_domain_flag_filters(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'python regex', ['regex'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002', 'python yaml', ['yaml'], 'python')
        make_entry(self.root / 'tools' / 'GE-0003.md', 'GE-0003', 'git tools', ['git'], 'tools')
        make_entry(self.root / 'tools' / 'GE-0004.md', 'GE-0004', 'git branch', ['git'], 'tools')
        result = run_scanner(str(self.root), '--domain', 'python')
        self.assertIn('GE-0001', result.stdout)
        self.assertNotIn('GE-0003', result.stdout)

    def test_top_flag_limits_output(self):
        for i in range(1, 5):
            make_entry(self.root / 'python' / f'GE-000{i}.md', f'GE-000{i}',
                       f'regex yaml crlf entry {i}', ['regex', 'yaml'], 'python')
        result_all = run_scanner(str(self.root))
        result_top2 = run_scanner(str(self.root), '--top', '2')
        # Full scan: 4 entries = 6 pairs; top 2 should show fewer
        self.assertGreater(
            result_all.stdout.count('GE-'),
            result_top2.stdout.count('GE-')
        )

    def test_json_flag_outputs_valid_json(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'regex yaml', ['regex'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002', 'regex crlf', ['regex'], 'python')
        result = run_scanner(str(self.root), '--json')
        self.assertEqual(result.returncode, 0)
        data = json.loads(result.stdout)
        self.assertIsInstance(data, list)
        self.assertEqual(len(data), 1)
        self.assertIn('pair', data[0])
        self.assertIn('score', data[0])

    def test_record_flag_writes_to_checked_md(self):
        result = run_scanner(str(self.root), '--record', 'GE-0001 × GE-0002', 'distinct', 'no overlap')
        self.assertEqual(result.returncode, 0)
        content = (self.root / 'CHECKED.md').read_text()
        self.assertIn('GE-0001 × GE-0002', content)
        self.assertIn('distinct', content)

    def test_record_then_rescan_excludes_pair(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'regex yaml parsing', ['regex'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002', 'regex yaml crlf', ['regex'], 'python')
        # First scan shows the pair
        result1 = run_scanner(str(self.root))
        self.assertIn('GE-0001', result1.stdout)
        # Record it
        run_scanner(str(self.root), '--record', 'GE-0001 × GE-0002', 'distinct', '')
        # Second scan excludes it
        result2 = run_scanner(str(self.root))
        self.assertIn('No unchecked pairs', result2.stdout)

    def test_already_checked_pair_not_shown(self):
        make_entry(self.root / 'python' / 'GE-0001.md', 'GE-0001', 'regex yaml', ['regex'], 'python')
        make_entry(self.root / 'python' / 'GE-0002.md', 'GE-0002', 'regex yaml', ['regex'], 'python')
        (self.root / 'CHECKED.md').write_text(
            '| GE-0001 × GE-0002 | distinct | 2026-04-14 | |\n'
        )
        result = run_scanner(str(self.root))
        self.assertIn('No unchecked pairs', result.stdout)
```

- [ ] **Step 2: Run all dedupe_scanner tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py -v
```

Expected: All tests PASS.

- [ ] **Step 3: Run full test suite for regressions**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ -v 2>&1 | tail -20
```

Expected: No new failures (pre-existing failures unchanged).

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add tests/test_dedupe_scanner.py
git commit -m "test(dedupe): add CLI integration tests for dedupe_scanner"
```

---

## Task 6: E2E pipeline tests (TDD)

**Files:**
- Modify: `soredium/tests/test_integration.py`

- [ ] **Step 1: Add E2E tests**

Add to `soredium/tests/test_integration.py`:

```python
import sys as _sys
_sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from integrate_entry import integrate
from unittest.mock import patch as _patch

INTEGRATE = Path(__file__).parent.parent / 'scripts' / 'integrate_entry.py'
SCANNER   = Path(__file__).parent.parent / 'scripts' / 'dedupe_scanner.py'
VALIDATOR = Path(__file__).parent.parent / 'scripts' / 'validate_garden.py'


def run_integrate(entry: Path, garden: Path) -> dict:
    import json
    result = subprocess.run(
        [sys.executable, str(INTEGRATE), str(entry), str(garden)],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)


def run_scanner(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(SCANNER)] + list(args),
        capture_output=True, text=True
    )


def run_dedupe_check(garden: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(VALIDATOR), '--dedupe-check', str(garden)],
        capture_output=True, text=True
    )


YAML_ENTRY_A = textwrap.dedent("""\
    ---
    id: GE-20260414-aaaa01
    title: "YAML frontmatter regex silently skips CRLF line endings"
    type: gotcha
    domain: python
    stack: "Python (all versions), re module"
    tags: [regex, yaml, crlf, parsing]
    score: 12
    verified: true
    staleness_threshold: 730
    submitted: 2026-04-14
    ---

    ## YAML frontmatter regex silently skips CRLF line endings

    **ID:** GE-20260414-aaaa01
    **Stack:** Python (all versions), re module
    **Symptom:** Files silently skipped.
    **Context:** YAML parsing on Windows.

    ### Root cause
    CRLF vs LF mismatch in regex.

    ### Fix
    Normalise with .replace('\\r\\n', '\\n').

    ### Why this is non-obvious
    No error, just silence.

    *Score: 12/15 · Included because: silent failure · Reservation: none*
""")

YAML_ENTRY_B = textwrap.dedent("""\
    ---
    id: GE-20260414-bbbb02
    title: "Regex date validation insufficient for calendar values in fromisoformat"
    type: gotcha
    domain: python
    stack: "Python 3.7+, datetime.date"
    tags: [regex, datetime, validation, parsing]
    score: 11
    verified: true
    staleness_threshold: 730
    submitted: 2026-04-14
    ---

    ## Regex date validation insufficient for calendar values in fromisoformat

    **ID:** GE-20260414-bbbb02
    **Stack:** Python 3.7+, datetime.date
    **Symptom:** ValueError on valid-format dates.
    **Context:** Date parsing with regex pre-validation.

    ### Root cause
    Regex checks shape not calendar validity.

    ### Fix
    Wrap fromisoformat in try/except ValueError.

    ### Why this is non-obvious
    Regex feels complete but isn't.

    *Score: 11/15 · Included because: natural assumption · Reservation: none*
""")

YAML_ENTRY_C = textwrap.dedent("""\
    ---
    id: GE-20260414-cccc03
    title: "Structure architecture docs as property claims with guarantees"
    type: technique
    domain: tools
    stack: "Technical documentation (cross-cutting)"
    tags: [documentation, architecture, strategy, pattern]
    score: 14
    verified: true
    staleness_threshold: 1460
    submitted: 2026-04-14
    ---

    ## Structure architecture docs as property claims with guarantees

    **ID:** GE-20260414-cccc03
    **Stack:** Technical documentation
    **Labels:** #strategy #documentation

    ### The technique
    Use property claims as headings with Guarantees and Graceful Degradation.

    ### Why this is non-obvious
    Most developers write component tours, not property evaluations.

    *Score: 14/15 · Included because: high impact · Reservation: none*
""")


def _setup_yaml_garden(tmp_path: Path) -> tuple:
    """Set up a minimal garden with GARDEN.md, CHECKED.md, domain dirs."""
    (tmp_path / 'python').mkdir()
    (tmp_path / 'python' / 'INDEX.md').write_text(
        '| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n'
    )
    (tmp_path / 'tools').mkdir()
    (tmp_path / 'tools' / 'INDEX.md').write_text(
        '| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n'
    )
    (tmp_path / '_index').mkdir()
    (tmp_path / '_index' / 'global.md').write_text('| Domain | Index |\n|--------|-------|\n')
    (tmp_path / 'GARDEN.md').write_text(
        '**Last legacy ID:** GE-0001\n'
        '**Last full DEDUPE sweep:** 2026-04-14\n'
        '**Entries merged since last sweep:** 0\n'
        '**Drift threshold:** 3\n'
        '**Last staleness review:** never\n'
    )
    (tmp_path / 'CHECKED.md').write_text(
        '| Pair | Result | Date | Notes |\n|------|--------|------|-------|\n'
    )
    entry_a = tmp_path / 'python' / 'GE-20260414-aaaa01.md'
    entry_b = tmp_path / 'python' / 'GE-20260414-bbbb02.md'
    entry_c = tmp_path / 'tools' / 'GE-20260414-cccc03.md'
    entry_a.write_text(YAML_ENTRY_A)
    entry_b.write_text(YAML_ENTRY_B)
    entry_c.write_text(YAML_ENTRY_C)
    return entry_a, entry_b, entry_c


class TestE2EDedupeScanner(unittest.TestCase):
    """End-to-end: integrate entries → drift increments → scan → record → re-scan."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        self.entry_a, self.entry_b, self.entry_c = _setup_yaml_garden(self.garden)

    def tearDown(self):
        self.tmp.cleanup()

    def test_happy_path_related_pair(self):
        """Submit 2 similar entries → integrate → drift=2 → scan shows pair → record → re-scan empty."""
        # Integrate both entries (mocking git and validate since no real git repo)
        with _patch('integrate_entry.run_validate'), _patch('integrate_entry.git_commit'):
            integrate(str(self.entry_a), str(self.garden))
            integrate(str(self.entry_b), str(self.garden))

        # Drift should be 2
        import re as _re
        content = (self.garden / 'GARDEN.md').read_text()
        m = _re.search(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', content)
        self.assertEqual(int(m.group(1)), 2)

        # Scanner shows the python pair
        result = run_scanner(str(self.garden), '--domain', 'python')
        self.assertIn('GE-20260414-aaaa01', result.stdout)
        self.assertIn('GE-20260414-bbbb02', result.stdout)

        # Pair has non-zero score (both have 'regex' and 'parsing' in common)
        result_json = run_scanner(str(self.garden), '--domain', 'python', '--json')
        data = json.loads(result_json.stdout)
        self.assertEqual(len(data), 1)
        self.assertGreater(data[0]['score'], 0.0)

        # Record as related
        run_scanner(str(self.garden), '--record',
                    'GE-20260414-aaaa01 × GE-20260414-bbbb02', 'related', 'both regex validation gaps')

        # Re-scan — pair absent
        result2 = run_scanner(str(self.garden), '--domain', 'python')
        self.assertIn('No unchecked pairs', result2.stdout)

    def test_happy_path_distinct_pair(self):
        """Entry C (tools) is distinct from any python entries — no cross-domain pairs."""
        with _patch('integrate_entry.run_validate'), _patch('integrate_entry.git_commit'):
            integrate(str(self.entry_a), str(self.garden))
            integrate(str(self.entry_c), str(self.garden))

        # Full scan — no cross-domain pairs
        result = run_scanner(str(self.garden))
        # python has 1 entry, tools has 1 entry — no within-domain pairs
        self.assertIn('No unchecked pairs', result.stdout)

    def test_happy_path_drift_triggers_dedupe_check(self):
        """Integrate 3 entries → drift=3 ≥ threshold=3 → --dedupe-check warns."""
        with _patch('integrate_entry.run_validate'), _patch('integrate_entry.git_commit'):
            integrate(str(self.entry_a), str(self.garden))
            integrate(str(self.entry_b), str(self.garden))
            integrate(str(self.entry_c), str(self.garden))

        result = run_dedupe_check(self.garden)
        self.assertEqual(result.returncode, 2, result.stdout)
        self.assertIn('DEDUPE recommended', result.stdout)

    def test_happy_path_bulk_import_all_pairs_generated(self):
        """3 entries in same domain → 3 pairs generated."""
        # Add a third python entry
        entry_d = self.garden / 'python' / 'GE-20260414-dddd04.md'
        entry_d.write_text(YAML_ENTRY_A.replace('GE-20260414-aaaa01', 'GE-20260414-dddd04')
                                       .replace('silently skips CRLF', 'raises ValueError on invalid'))

        result_json = run_scanner(str(self.garden), '--domain', 'python', '--json')
        data = json.loads(result_json.stdout)
        # 3 entries: GE-aaaa01, GE-bbbb02, GE-dddd04 → 3 pairs
        self.assertEqual(len(data), 3)

    def test_top_flag_in_e2e_context(self):
        """--top 1 returns only the highest-scoring pair."""
        entry_d = self.garden / 'python' / 'GE-20260414-dddd04.md'
        entry_d.write_text(YAML_ENTRY_A.replace('GE-20260414-aaaa01', 'GE-20260414-dddd04'))

        result_all = run_scanner(str(self.garden), '--domain', 'python', '--json')
        result_top1 = run_scanner(str(self.garden), '--domain', 'python', '--json', '--top', '1')

        all_data = json.loads(result_all.stdout)
        top_data = json.loads(result_top1.stdout)

        self.assertEqual(len(top_data), 1)
        self.assertEqual(top_data[0]['score'], all_data[0]['score'])
```

- [ ] **Step 2: Run E2E tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integration.py::TestE2EDedupeScanner -v
```

Expected: All 5 E2E tests PASS.

- [ ] **Step 3: Run full test suite**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ -v 2>&1 | tail -30
```

Expected: All new tests pass; no regressions in existing tests.

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add tests/test_integration.py
git commit -m "test(integration): add E2E tests for full dedupe pipeline — integrate → drift → scan → record"
```

---

## Task 7: Update harvest SKILL.md

**Files:**
- Modify: `soredium/harvest/SKILL.md`

- [ ] **Step 1: Update DEDUPE Step 1 in harvest/SKILL.md**

In `soredium/harvest/SKILL.md`, find `**Step 1 — Load the index and pair log**` and replace it and its content with:

```markdown
**Step 1 — Run scanner to get pre-scored pair list**

```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py \
  ${HORTORA_GARDEN:-~/.hortora/garden} --top 50
```

The scanner reads domain directories directly for YAML-frontmatter entries, cross-references CHECKED.md to exclude already-checked pairs, computes Jaccard similarity for all unchecked within-domain pairs, and returns them sorted highest-score-first.

Review pairs from highest score down — highest-scoring pairs are most likely related or duplicate. If a domain has many entries, add `--domain <name>` to focus on one domain at a time.

To see all unchecked pairs as JSON (for scripting):
```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py \
  ${HORTORA_GARDEN:-~/.hortora/garden} --json
```
```

- [ ] **Step 2: Update DEDUPE Step 5 in harvest/SKILL.md**

Find `**Step 5 — Update CHECKED.md**` and replace it and its content with:

```markdown
**Step 5 — Record comparison results via scanner**

For each pair reviewed, record the result using the scanner's `--record` flag:

```bash
python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py \
  ${HORTORA_GARDEN:-~/.hortora/garden} \
  --record "GE-XXXX × GE-YYYY" distinct "brief note"
```

Results must be one of: `distinct`, `related`, `duplicate-discarded`.

The scanner enforces canonical ID ordering and is idempotent — recording the same pair twice produces one row only. Do NOT append to CHECKED.md manually.
```

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/soredium
git add harvest/SKILL.md
git commit -m "feat(harvest): update DEDUPE to use dedupe_scanner for pair prioritisation and CHECKED.md management"
```

---

## Self-Review

**Spec coverage:**

| Requirement | Task |
|---|---|
| Drift counter auto-incremented by integrate_entry.py | Task 1 |
| Counter starts from any value, increments by 1 | Task 1 |
| Missing counter field handled gracefully | Task 1 |
| --dedupe-check reads counter, exits 0/2 | Task 2 |
| --dedupe-check cross-checks git log | Task 2 |
| Git-log count wins when higher | Task 2 |
| tokenize: 3+ char, lowercase, dedup, punctuation, CRLF | Task 3 |
| jaccard: empty, identical, disjoint, partial, formula | Task 3 |
| canonical_pair: lexicographic, legacy before new format | Task 3 |
| load_checked_pairs: empty, missing, both orderings, malformed | Task 3 |
| record_pair: appends, canonical order, idempotent, validates result | Task 3 |
| load_entries: YAML only, skip control files, skip _summaries, domain filter, CRLF files | Task 4 |
| compute_pairs: 0/1/2/3 entries, exclude checked, cross-domain excluded, sorted, top limit | Task 4 |
| CLI: --domain, --top, --json, --record | Task 5 |
| record-then-rescan excludes pair | Task 5 |
| E2E: related pair (integrate → drift → scan → record → re-scan) | Task 6 |
| E2E: distinct pair (no cross-domain pairs) | Task 6 |
| E2E: drift triggers --dedupe-check warning | Task 6 |
| E2E: bulk import generates N×(N-1)/2 pairs | Task 6 |
| harvest SKILL.md updated (Step 1, Step 5) | Task 7 |

**No gaps. No placeholders.**

**Type consistency:** `canonical_pair()` returns `"GE-X × GE-Y"` format throughout. `load_checked_pairs()` returns `set[str]` of canonical pair strings. `compute_pairs()` accepts `set` from `load_checked_pairs()`. `record_pair()` reads from `load_checked_pairs()` for idempotency. All consistent.
