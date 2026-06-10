# SQLite Aggregate State Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace CHECKED.md, DISCARDED.md, and the GARDEN.md By Technology index with a single SQLite database (`garden.db`) providing O(log n) lookup and ACID writes, per ADR-0004.

**Architecture:** New module `garden_db.py` owns all SQLite schema and CRUD operations. `garden_db_migrate.py` does the one-time migration from existing markdown files. Five existing scripts are updated to use `garden_db.py` instead of flat-file access: `dedupe_scanner.py`, `integrate_entry.py`, `init_garden.py`, `validate_garden.py`, and `harvest SKILL.md`. The existing `GardenFixture` test helper is updated to create `garden.db` so existing tests continue to work. CHECKED.md and DISCARDED.md are retired (renamed to `.bak` post-migration).

**Tech Stack:** Python 3.11+ `sqlite3` (stdlib), WAL journal mode, pytest. No new runtime dependencies.

---

## File Map

| File | Status | Responsibility |
|------|--------|----------------|
| `scripts/garden_db.py` | Create | Schema, connection, all CRUD for checked_pairs / discarded_entries / entries_index |
| `scripts/garden_db_migrate.py` | Create | One-time migration from CHECKED.md / DISCARDED.md → garden.db |
| `tests/test_garden_db.py` | Create | Unit + integration tests for garden_db.py |
| `tests/test_garden_db_migrate.py` | Create | Unit + integration + CLI tests for garden_db_migrate.py |
| `scripts/dedupe_scanner.py` | Modify | Replace local load_checked_pairs / record_pair with garden_db imports |
| `tests/test_dedupe_scanner.py` | Modify | Update fixtures to use garden.db instead of CHECKED.md |
| `scripts/integrate_entry.py` | Modify | Add upsert_entry call after successful integration |
| `tests/test_integrate_entry.py` | Modify | Add TestEntriesIndexIntegration class |
| `scripts/init_garden.py` | Modify | Replace create_checked_md/create_discarded_md with garden_db.init_db(); add .gitattributes |
| `tests/test_init_garden.py` | Modify | Update assertions: garden.db created, CHECKED.md not created |
| `scripts/validate_garden.py` | Modify | Add --check-db flag; update CHECKED.md integrity checks to use garden_db |
| `tests/test_validate_garden.py` | Modify | Replace TestCheckedMdIntegrity with TestGardenDbIntegrity |
| `tests/garden_fixture.py` | Modify | Update GardenFixture to init garden.db instead of CHECKED.md/DISCARDED.md |
| `harvest/SKILL.md` | Modify | Update DEDUPE steps 5 and 6 to reference garden.db |

---

## Task 1: garden_db.py — core module (TDD)

**Files:**
- Create: `soredium/scripts/garden_db.py`
- Create: `soredium/tests/test_garden_db.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_garden_db.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for garden_db.py."""

import json
import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from garden_db import (
    init_db, get_connection, get_schema_version,
    is_pair_checked, get_pair_result, record_pair, load_checked_pairs,
    record_discarded, is_discarded,
    upsert_entry, get_entries_by_domain,
    VALID_RESULTS, SCHEMA_VERSION,
)


class TestInitDb(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_garden_db_file(self):
        init_db(self.garden)
        self.assertTrue((self.garden / 'garden.db').exists())

    def test_creates_checked_pairs_table(self):
        init_db(self.garden)
        conn = get_connection(self.garden)
        conn.execute("SELECT * FROM checked_pairs")
        conn.close()

    def test_creates_discarded_entries_table(self):
        init_db(self.garden)
        conn = get_connection(self.garden)
        conn.execute("SELECT * FROM discarded_entries")
        conn.close()

    def test_creates_entries_index_table(self):
        init_db(self.garden)
        conn = get_connection(self.garden)
        conn.execute("SELECT * FROM entries_index")
        conn.close()

    def test_schema_version_set(self):
        init_db(self.garden)
        self.assertEqual(get_schema_version(self.garden), SCHEMA_VERSION)

    def test_idempotent_second_init(self):
        init_db(self.garden)
        init_db(self.garden)  # must not raise
        self.assertEqual(get_schema_version(self.garden), SCHEMA_VERSION)

    def test_no_db_returns_none_schema_version(self):
        self.assertIsNone(get_schema_version(self.garden))

    def test_wal_mode_enabled(self):
        init_db(self.garden)
        conn = get_connection(self.garden)
        mode = conn.execute("PRAGMA journal_mode").fetchone()[0]
        conn.close()
        self.assertEqual(mode, 'wal')


class TestCheckedPairs(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        init_db(self.garden)

    def tearDown(self):
        self.tmp.cleanup()

    def test_new_pair_not_checked(self):
        self.assertFalse(is_pair_checked(self.garden, 'GE-0001 × GE-0002'))

    def test_recorded_pair_is_checked(self):
        record_pair(self.garden, 'GE-0001 × GE-0002', 'distinct')
        self.assertTrue(is_pair_checked(self.garden, 'GE-0001 × GE-0002'))

    def test_get_pair_result_returns_result(self):
        record_pair(self.garden, 'GE-0001 × GE-0002', 'related', 'cross-ref added')
        self.assertEqual(get_pair_result(self.garden, 'GE-0001 × GE-0002'), 'related')

    def test_get_pair_result_none_for_unknown(self):
        self.assertIsNone(get_pair_result(self.garden, 'GE-9999 × GE-8888'))

    def test_record_pair_idempotent(self):
        record_pair(self.garden, 'GE-0001 × GE-0002', 'distinct')
        record_pair(self.garden, 'GE-0001 × GE-0002', 'distinct')
        conn = get_connection(self.garden)
        count = conn.execute("SELECT COUNT(*) FROM checked_pairs").fetchone()[0]
        conn.close()
        self.assertEqual(count, 1)

    def test_canonical_pair_order_enforced(self):
        record_pair(self.garden, 'GE-0002 × GE-0001', 'distinct')
        self.assertTrue(is_pair_checked(self.garden, 'GE-0001 × GE-0002'))
        self.assertTrue(is_pair_checked(self.garden, 'GE-0002 × GE-0001'))

    def test_invalid_result_raises(self):
        with self.assertRaises(ValueError):
            record_pair(self.garden, 'GE-0001 × GE-0002', 'bogus')

    def test_all_valid_results_accepted(self):
        for i, result in enumerate(sorted(VALID_RESULTS)):
            record_pair(self.garden, f'GE-000{i} × GE-999{i}', result)
            self.assertEqual(get_pair_result(self.garden, f'GE-000{i} × GE-999{i}'), result)

    def test_load_checked_pairs_empty(self):
        self.assertEqual(load_checked_pairs(self.garden), set())

    def test_load_checked_pairs_returns_all(self):
        record_pair(self.garden, 'GE-0001 × GE-0002', 'distinct')
        record_pair(self.garden, 'GE-0003 × GE-0004', 'related')
        pairs = load_checked_pairs(self.garden)
        self.assertIn('GE-0001 × GE-0002', pairs)
        self.assertIn('GE-0003 × GE-0004', pairs)
        self.assertEqual(len(pairs), 2)

    def test_notes_stored_and_retrieved(self):
        record_pair(self.garden, 'GE-0001 × GE-0002', 'related', 'cross-referenced both')
        conn = get_connection(self.garden)
        row = conn.execute("SELECT notes FROM checked_pairs").fetchone()
        conn.close()
        self.assertEqual(row[0], 'cross-referenced both')


class TestDiscardedEntries(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        init_db(self.garden)

    def tearDown(self):
        self.tmp.cleanup()

    def test_new_entry_not_discarded(self):
        self.assertFalse(is_discarded(self.garden, 'GE-0001'))

    def test_recorded_entry_is_discarded(self):
        record_discarded(self.garden, 'GE-0001', 'GE-0002', 'subset of GE-0002')
        self.assertTrue(is_discarded(self.garden, 'GE-0001'))

    def test_idempotent(self):
        record_discarded(self.garden, 'GE-0001', 'GE-0002')
        record_discarded(self.garden, 'GE-0001', 'GE-0002')
        conn = get_connection(self.garden)
        count = conn.execute("SELECT COUNT(*) FROM discarded_entries").fetchone()[0]
        conn.close()
        self.assertEqual(count, 1)

    def test_reason_stored(self):
        record_discarded(self.garden, 'GE-0001', 'GE-0002', 'exact duplicate')
        conn = get_connection(self.garden)
        row = conn.execute("SELECT reason FROM discarded_entries").fetchone()
        conn.close()
        self.assertEqual(row[0], 'exact duplicate')


class TestEntriesIndex(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        init_db(self.garden)
        self.entry = {
            'ge_id': 'GE-20260414-aa0001',
            'title': 'Hibernate @PreUpdate fires at flush time',
            'domain': 'java',
            'type': 'gotcha',
            'score': 12,
            'submitted': '2026-04-14',
            'staleness_threshold': 730,
            'tags': ['hibernate', 'jpa', 'flush'],
            'verified_on': 'hibernate: 6.x',
            'last_reviewed': '',
            'file_path': 'java/GE-20260414-aa0001.md',
        }

    def tearDown(self):
        self.tmp.cleanup()

    def test_upsert_creates_row(self):
        upsert_entry(self.garden, self.entry)
        rows = get_entries_by_domain(self.garden, 'java')
        self.assertEqual(len(rows), 1)

    def test_upsert_idempotent(self):
        upsert_entry(self.garden, self.entry)
        upsert_entry(self.garden, self.entry)
        rows = get_entries_by_domain(self.garden, 'java')
        self.assertEqual(len(rows), 1)

    def test_get_entries_by_domain_filters_correctly(self):
        upsert_entry(self.garden, self.entry)
        other = {**self.entry, 'ge_id': 'GE-20260414-bb0001', 'domain': 'tools',
                 'file_path': 'tools/GE-20260414-bb0001.md'}
        upsert_entry(self.garden, other)
        java = get_entries_by_domain(self.garden, 'java')
        tools = get_entries_by_domain(self.garden, 'tools')
        self.assertEqual(len(java), 1)
        self.assertEqual(len(tools), 1)

    def test_tags_round_trip_as_list(self):
        upsert_entry(self.garden, self.entry)
        rows = get_entries_by_domain(self.garden, 'java')
        self.assertEqual(rows[0]['tags'], ['hibernate', 'jpa', 'flush'])

    def test_get_entries_ordered_by_score_desc(self):
        low = {**self.entry, 'ge_id': 'GE-20260414-low', 'score': 8,
               'file_path': 'java/GE-20260414-low.md'}
        upsert_entry(self.garden, self.entry)   # score 12
        upsert_entry(self.garden, low)           # score 8
        rows = get_entries_by_domain(self.garden, 'java')
        self.assertGreaterEqual(rows[0]['score'], rows[1]['score'])

    def test_empty_domain_returns_empty_list(self):
        self.assertEqual(get_entries_by_domain(self.garden, 'nonexistent'), [])

    def test_upsert_updates_existing(self):
        upsert_entry(self.garden, self.entry)
        updated = {**self.entry, 'title': 'Updated title'}
        upsert_entry(self.garden, updated)
        rows = get_entries_by_domain(self.garden, 'java')
        self.assertEqual(rows[0]['title'], 'Updated title')


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_db.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'garden_db'`

- [ ] **Step 3: Create garden_db.py**

Create `soredium/scripts/garden_db.py`:

```python
#!/usr/bin/env python3
"""garden_db.py — SQLite-backed storage for garden aggregate state.

Replaces CHECKED.md, DISCARDED.md, and the GARDEN.md By Technology index.
All mutable state lives in garden.db, committed to git alongside entry files.

Tables:
  checked_pairs    — deduplication pair comparison results (replaces CHECKED.md)
  discarded_entries — retired entry log (replaces DISCARDED.md)
  entries_index    — metadata index for fast domain queries (derived, rebuildable)
  schema_version   — migration tracking
"""

import hashlib
import json
import sqlite3
from datetime import date
from pathlib import Path

SCHEMA_VERSION = 1
VALID_RESULTS = {'distinct', 'related', 'duplicate-discarded'}

_CREATE_CHECKED_PAIRS = """
CREATE TABLE IF NOT EXISTS checked_pairs (
    pair_hash  TEXT PRIMARY KEY,
    pair       TEXT NOT NULL,
    result     TEXT NOT NULL,
    checked_at TEXT NOT NULL,
    notes      TEXT DEFAULT ''
)"""

_CREATE_DISCARDED = """
CREATE TABLE IF NOT EXISTS discarded_entries (
    ge_id          TEXT PRIMARY KEY,
    conflicts_with TEXT NOT NULL,
    discarded_at   TEXT NOT NULL,
    reason         TEXT DEFAULT ''
)"""

_CREATE_ENTRIES_INDEX = """
CREATE TABLE IF NOT EXISTS entries_index (
    ge_id               TEXT PRIMARY KEY,
    title               TEXT NOT NULL,
    domain              TEXT NOT NULL,
    type                TEXT NOT NULL,
    score               INTEGER NOT NULL,
    submitted           TEXT NOT NULL,
    staleness_threshold INTEGER NOT NULL DEFAULT 730,
    tags                TEXT DEFAULT '[]',
    verified_on         TEXT DEFAULT '',
    last_reviewed       TEXT DEFAULT '',
    file_path           TEXT NOT NULL
)"""

_CREATE_SCHEMA_VERSION = """
CREATE TABLE IF NOT EXISTS schema_version (
    version    INTEGER NOT NULL,
    applied_at TEXT NOT NULL
)"""

_INDICES = [
    "CREATE INDEX IF NOT EXISTS idx_entries_domain ON entries_index(domain)",
    "CREATE INDEX IF NOT EXISTS idx_entries_score  ON entries_index(score DESC)",
    "CREATE INDEX IF NOT EXISTS idx_entries_submitted ON entries_index(submitted)",
]


def _pair_hash(pair: str) -> str:
    return hashlib.sha256(pair.encode()).hexdigest()[:8]


def _canonical_pair(pair: str) -> str:
    """Return canonical 'lower × higher' form regardless of input order."""
    parts = [p.strip() for p in pair.replace('x', '×').split('×')]
    if len(parts) == 2:
        return f"{min(parts)} × {max(parts)}"
    return pair


def get_connection(garden: Path) -> sqlite3.Connection:
    """Return an open WAL-mode sqlite3 connection to garden.db."""
    conn = sqlite3.connect(str(Path(garden) / 'garden.db'))
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn


def init_db(garden: Path) -> None:
    """Create garden.db with all tables if they don't exist. Idempotent."""
    conn = get_connection(garden)
    with conn:
        for ddl in (_CREATE_CHECKED_PAIRS, _CREATE_DISCARDED,
                    _CREATE_ENTRIES_INDEX, _CREATE_SCHEMA_VERSION):
            conn.execute(ddl)
        for idx in _INDICES:
            conn.execute(idx)
        if not conn.execute("SELECT 1 FROM schema_version").fetchone():
            conn.execute("INSERT INTO schema_version VALUES (?, ?)",
                         (SCHEMA_VERSION, date.today().isoformat()))
    conn.close()


def get_schema_version(garden: Path) -> int | None:
    """Return current schema version, or None if garden.db does not exist."""
    db_path = Path(garden) / 'garden.db'
    if not db_path.exists():
        return None
    conn = get_connection(garden)
    row = conn.execute(
        "SELECT version FROM schema_version ORDER BY version DESC LIMIT 1"
    ).fetchone()
    conn.close()
    return row['version'] if row else None


# ── checked_pairs ──────────────────────────────────────────────────────────────

def is_pair_checked(garden: Path, pair: str) -> bool:
    """Return True if pair has been classified."""
    conn = get_connection(garden)
    row = conn.execute(
        "SELECT 1 FROM checked_pairs WHERE pair_hash = ?",
        (_pair_hash(_canonical_pair(pair)),)
    ).fetchone()
    conn.close()
    return row is not None


def get_pair_result(garden: Path, pair: str) -> str | None:
    """Return result for pair ('distinct'|'related'|'duplicate-discarded'), or None."""
    conn = get_connection(garden)
    row = conn.execute(
        "SELECT result FROM checked_pairs WHERE pair_hash = ?",
        (_pair_hash(_canonical_pair(pair)),)
    ).fetchone()
    conn.close()
    return row['result'] if row else None


def record_pair(garden: Path, pair: str, result: str, notes: str = '') -> None:
    """Record a pair comparison result. Idempotent — silently skips if already recorded."""
    if result not in VALID_RESULTS:
        raise ValueError(f"result must be one of {sorted(VALID_RESULTS)}, got {result!r}")
    canon = _canonical_pair(pair)
    conn = get_connection(garden)
    with conn:
        conn.execute(
            "INSERT OR IGNORE INTO checked_pairs "
            "(pair_hash, pair, result, checked_at, notes) VALUES (?, ?, ?, ?, ?)",
            (_pair_hash(canon), canon, result, date.today().isoformat(), notes)
        )
    conn.close()


def load_checked_pairs(garden: Path) -> set:
    """Return set of all canonical pair strings already classified."""
    conn = get_connection(garden)
    rows = conn.execute("SELECT pair FROM checked_pairs").fetchall()
    conn.close()
    return {row['pair'] for row in rows}


# ── discarded_entries ─────────────────────────────────────────────────────────

def record_discarded(garden: Path, ge_id: str, conflicts_with: str,
                     reason: str = '') -> None:
    """Record a discarded entry. Idempotent."""
    conn = get_connection(garden)
    with conn:
        conn.execute(
            "INSERT OR IGNORE INTO discarded_entries "
            "(ge_id, conflicts_with, discarded_at, reason) VALUES (?, ?, ?, ?)",
            (ge_id, conflicts_with, date.today().isoformat(), reason)
        )
    conn.close()


def is_discarded(garden: Path, ge_id: str) -> bool:
    """Return True if ge_id has been discarded."""
    conn = get_connection(garden)
    row = conn.execute(
        "SELECT 1 FROM discarded_entries WHERE ge_id = ?", (ge_id,)
    ).fetchone()
    conn.close()
    return row is not None


# ── entries_index ─────────────────────────────────────────────────────────────

def upsert_entry(garden: Path, entry: dict) -> None:
    """Insert or replace an entry in entries_index."""
    conn = get_connection(garden)
    with conn:
        conn.execute(
            "INSERT OR REPLACE INTO entries_index "
            "(ge_id, title, domain, type, score, submitted, staleness_threshold, "
            "tags, verified_on, last_reviewed, file_path) "
            "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
            (
                entry['ge_id'], entry.get('title', ''), entry.get('domain', ''),
                entry.get('type', ''), int(entry.get('score', 0)),
                entry.get('submitted', ''), int(entry.get('staleness_threshold', 730)),
                json.dumps(entry.get('tags', [])),
                entry.get('verified_on', ''), entry.get('last_reviewed', ''),
                entry.get('file_path', ''),
            )
        )
    conn.close()


def get_entries_by_domain(garden: Path, domain: str) -> list:
    """Return all entries_index rows for a domain, ordered by score DESC."""
    conn = get_connection(garden)
    rows = conn.execute(
        "SELECT * FROM entries_index WHERE domain = ? ORDER BY score DESC", (domain,)
    ).fetchall()
    conn.close()
    result = []
    for row in rows:
        d = dict(row)
        d['tags'] = json.loads(d['tags'])
        result.append(d)
    return result
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_db.py -v
```

Expected: All tests PASS.

- [ ] **Step 5: Full suite — no regressions**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden_db.py tests/test_garden_db.py
git commit -m "feat(db): add garden_db.py — SQLite module for checked_pairs, discarded_entries, entries_index

Refs #29"
```

---

## Task 2: garden_db_migrate.py — migration script (TDD)

**Files:**
- Create: `soredium/scripts/garden_db_migrate.py`
- Create: `soredium/tests/test_garden_db_migrate.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_garden_db_migrate.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for garden_db_migrate.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from garden_db import init_db, load_checked_pairs, is_discarded, get_schema_version
from garden_db_migrate import migrate_checked_md, migrate_discarded_md, run_migration

CLI = Path(__file__).parent.parent / 'scripts' / 'garden_db_migrate.py'


def run_cli(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(CLI)] + list(args),
        capture_output=True, text=True
    )


CHECKED_MD_CONTENT = """\
# Garden Duplicate Check Log

| Pair | Result | Date | Notes |
|------|--------|------|-------|
| GE-0001 × GE-0002 | distinct | 2026-04-10 | |
| GE-0003 × GE-0004 | related | 2026-04-11 | cross-referenced |
| GE-0005 × GE-0006 | duplicate-discarded | 2026-04-12 | GE-0005 kept |
| invalid-row | bogus | 2026-04-13 | should be skipped |
"""

DISCARDED_MD_CONTENT = """\
# Discarded Submissions

| Discarded | Conflicts With | Date | Reason |
|-----------|----------------|------|--------|
| GE-0006 | GE-0005 | 2026-04-12 | exact duplicate |
| GE-0007 | GE-0008 | 2026-04-13 | subset |
"""


class TestMigrateCheckedMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        init_db(self.garden)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_file_returns_zero(self):
        (self.garden / 'CHECKED.md').write_text('| Pair | Result | Date | Notes |\n|---|---|---|---|\n')
        self.assertEqual(migrate_checked_md(self.garden), 0)

    def test_missing_file_returns_zero(self):
        self.assertEqual(migrate_checked_md(self.garden), 0)

    def test_migrates_three_valid_rows(self):
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        count = migrate_checked_md(self.garden)
        self.assertEqual(count, 3)

    def test_invalid_result_skipped(self):
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        migrate_checked_md(self.garden)
        pairs = load_checked_pairs(self.garden)
        self.assertEqual(len(pairs), 3)  # 3 valid, 1 invalid skipped

    def test_pairs_accessible_after_migration(self):
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        migrate_checked_md(self.garden)
        pairs = load_checked_pairs(self.garden)
        self.assertIn('GE-0001 × GE-0002', pairs)
        self.assertIn('GE-0003 × GE-0004', pairs)

    def test_dry_run_does_not_write(self):
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        count = migrate_checked_md(self.garden, dry_run=True)
        self.assertEqual(count, 3)  # counts rows
        self.assertEqual(load_checked_pairs(self.garden), set())  # nothing written


class TestMigrateDiscardedMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        init_db(self.garden)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_file_returns_zero(self):
        (self.garden / 'DISCARDED.md').write_text('| Discarded | Conflicts With | Date | Reason |\n|---|---|---|---|\n')
        self.assertEqual(migrate_discarded_md(self.garden), 0)

    def test_missing_file_returns_zero(self):
        self.assertEqual(migrate_discarded_md(self.garden), 0)

    def test_migrates_two_valid_rows(self):
        (self.garden / 'DISCARDED.md').write_text(DISCARDED_MD_CONTENT)
        count = migrate_discarded_md(self.garden)
        self.assertEqual(count, 2)

    def test_entries_accessible_after_migration(self):
        (self.garden / 'DISCARDED.md').write_text(DISCARDED_MD_CONTENT)
        migrate_discarded_md(self.garden)
        self.assertTrue(is_discarded(self.garden, 'GE-0006'))
        self.assertTrue(is_discarded(self.garden, 'GE-0007'))

    def test_dry_run_does_not_write(self):
        (self.garden / 'DISCARDED.md').write_text(DISCARDED_MD_CONTENT)
        migrate_discarded_md(self.garden, dry_run=True)
        self.assertFalse(is_discarded(self.garden, 'GE-0006'))


class TestRunMigration(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        (self.garden / 'DISCARDED.md').write_text(DISCARDED_MD_CONTENT)

    def tearDown(self):
        self.tmp.cleanup()

    def test_returns_counts(self):
        result = run_migration(self.garden)
        self.assertEqual(result['checked'], 3)
        self.assertEqual(result['discarded'], 2)

    def test_creates_garden_db(self):
        run_migration(self.garden)
        self.assertTrue((self.garden / 'garden.db').exists())

    def test_renames_checked_md_to_bak(self):
        run_migration(self.garden)
        self.assertFalse((self.garden / 'CHECKED.md').exists())
        self.assertTrue((self.garden / 'CHECKED.md.bak').exists())

    def test_renames_discarded_md_to_bak(self):
        run_migration(self.garden)
        self.assertFalse((self.garden / 'DISCARDED.md').exists())
        self.assertTrue((self.garden / 'DISCARDED.md.bak').exists())

    def test_dry_run_no_db_created(self):
        run_migration(self.garden, dry_run=True)
        self.assertFalse((self.garden / 'garden.db').exists())

    def test_dry_run_no_rename(self):
        run_migration(self.garden, dry_run=True)
        self.assertTrue((self.garden / 'CHECKED.md').exists())
        self.assertTrue((self.garden / 'DISCARDED.md').exists())

    def test_schema_version_set_after_migration(self):
        run_migration(self.garden)
        from garden_db import SCHEMA_VERSION
        self.assertEqual(get_schema_version(self.garden), SCHEMA_VERSION)


class TestGardenDbMigrateCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name)
        (self.garden / 'CHECKED.md').write_text(CHECKED_MD_CONTENT)
        (self.garden / 'DISCARDED.md').write_text(DISCARDED_MD_CONTENT)

    def tearDown(self):
        self.tmp.cleanup()

    def test_exits_0_on_success(self):
        result = run_cli(str(self.garden))
        self.assertEqual(result.returncode, 0, result.stderr)

    def test_output_shows_counts(self):
        result = run_cli(str(self.garden))
        self.assertIn('3', result.stdout)
        self.assertIn('2', result.stdout)

    def test_dry_run_flag(self):
        result = run_cli(str(self.garden), '--dry-run')
        self.assertEqual(result.returncode, 0)
        self.assertIn('DRY RUN', result.stdout.upper())
        self.assertFalse((self.garden / 'garden.db').exists())

    def test_invalid_path_exits_nonzero(self):
        result = run_cli('/nonexistent/path')
        self.assertNotEqual(result.returncode, 0)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_db_migrate.py -v 2>&1 | head -10
```

- [ ] **Step 3: Create garden_db_migrate.py**

Create `soredium/scripts/garden_db_migrate.py`:

```python
#!/usr/bin/env python3
"""garden_db_migrate.py — One-time migration from CHECKED.md / DISCARDED.md to garden.db.

Usage:
  garden_db_migrate.py <garden_path> [--dry-run]

--dry-run: print what would be migrated without writing anything.

After migration: CHECKED.md → CHECKED.md.bak, DISCARDED.md → DISCARDED.md.bak
"""

import argparse
import re
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))
from garden_db import init_db, record_pair, record_discarded, VALID_RESULTS

_PAIR_RE = re.compile(
    r'\|\s*(GE-[\w-]+)\s*[×x]\s*(GE-[\w-]+)\s*\|\s*([\w-]+)\s*\|[^|]*\|([^|]*)\|'
)
_DISCARDED_RE = re.compile(
    r'\|\s*(GE-[\w-]+)\s*\|\s*(GE-[\w-]+)\s*\|[^|]*\|([^|]*)\|'
)


def migrate_checked_md(garden: Path, dry_run: bool = False) -> int:
    """Read CHECKED.md rows into garden.db. Returns number of rows migrated."""
    checked_md = Path(garden) / 'CHECKED.md'
    if not checked_md.exists():
        return 0
    count = 0
    for line in checked_md.read_text(encoding='utf-8').splitlines():
        m = _PAIR_RE.search(line)
        if not m:
            continue
        id_a, id_b = m.group(1), m.group(2)
        result = m.group(3).strip()
        notes = m.group(4).strip()
        if result not in VALID_RESULTS:
            continue
        pair = f"{min(id_a, id_b)} × {max(id_a, id_b)}"
        if not dry_run:
            record_pair(garden, pair, result, notes)
        count += 1
    return count


def migrate_discarded_md(garden: Path, dry_run: bool = False) -> int:
    """Read DISCARDED.md rows into garden.db. Returns number of rows migrated."""
    discarded_md = Path(garden) / 'DISCARDED.md'
    if not discarded_md.exists():
        return 0
    count = 0
    for line in discarded_md.read_text(encoding='utf-8').splitlines():
        m = _DISCARDED_RE.search(line)
        if not m:
            continue
        ge_id, conflicts_with, reason = m.group(1), m.group(2), m.group(3).strip()
        if ge_id in ('Discarded', 'GE-ID'):
            continue
        if not dry_run:
            record_discarded(garden, ge_id, conflicts_with, reason)
        count += 1
    return count


def run_migration(garden: Path, dry_run: bool = False) -> dict:
    """Full migration: init db, import CHECKED.md and DISCARDED.md, rename sources."""
    garden = Path(garden)
    if not garden.exists():
        raise FileNotFoundError(f"Garden not found: {garden}")
    if not dry_run:
        init_db(garden)
    checked = migrate_checked_md(garden, dry_run)
    discarded = migrate_discarded_md(garden, dry_run)
    if not dry_run:
        for fname in ('CHECKED.md', 'DISCARDED.md'):
            src = garden / fname
            if src.exists():
                src.rename(garden / f'{fname}.bak')
    return {'checked': checked, 'discarded': discarded}


def main():
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('garden_path', type=Path)
    parser.add_argument('--dry-run', action='store_true',
                        help='Preview migration without writing anything')
    args = parser.parse_args()

    try:
        result = run_migration(args.garden_path, dry_run=args.dry_run)
    except FileNotFoundError as e:
        print(f"ERROR: {e}", file=sys.stderr)
        sys.exit(1)

    prefix = '[DRY RUN] ' if args.dry_run else ''
    print(f"{prefix}Migrated {result['checked']} checked pairs, "
          f"{result['discarded']} discarded entries")
    if args.dry_run:
        print("Run without --dry-run to write to garden.db and rename source files.")
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_db_migrate.py -v
```

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden_db_migrate.py tests/test_garden_db_migrate.py
git commit -m "feat(db): add garden_db_migrate.py — one-time migration from CHECKED.md to garden.db

Refs #29"
```

---

## Task 3: Update GardenFixture + dedupe_scanner.py (TDD)

**Files:**
- Modify: `soredium/tests/garden_fixture.py` (or wherever GardenFixture is defined — check test_validate_garden.py)
- Modify: `soredium/scripts/dedupe_scanner.py`
- Modify: `soredium/tests/test_dedupe_scanner.py`

**Context:** `dedupe_scanner.py` currently defines its own `load_checked_pairs()` and `record_pair()` functions that read/write CHECKED.md. These are replaced by imports from `garden_db.py`. The `GardenFixture` test helper currently creates `CHECKED.md` and `DISCARDED.md` — it must now call `init_db()` instead so existing tests continue to work.

- [ ] **Step 1: Read the existing files**

Read these files before touching anything:
- `soredium/tests/test_validate_garden.py` — find GardenFixture definition (it may be inline here, not in garden_fixture.py)
- `soredium/scripts/dedupe_scanner.py` — find `load_checked_pairs()` and `record_pair()`

- [ ] **Step 2: Update GardenFixture**

In whichever file defines `GardenFixture`, find the `__init__` method that creates `CHECKED.md` and `DISCARDED.md`. Replace those two writes with a call to `init_db`:

```python
# Remove these lines:
# (self.root / "CHECKED.md").write_text(
#     "# Garden Duplicate Check Log\n\n"
#     "| Pair | Result | Date | Notes |\n"
#     "|------|--------|------|-------|\n"
# )
# (self.root / "DISCARDED.md").write_text(
#     "# Discarded Submissions\n\n"
#     "| Discarded | Conflicts With | Date | Reason |\n"
#     "|-----------|---------------|------|--------|\n"
# )

# Add at the top of the file:
import sys as _sys
_sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from garden_db import init_db as _init_db

# Replace the removed lines with:
_init_db(self.root)
```

Also add `schema_md` to keep the existing method if it writes CHECKED.md — it should not (CHECKED.md is gone). Verify all existing tests still pass after this change before proceeding.

- [ ] **Step 3: Write failing tests for dedupe_scanner changes**

In `soredium/tests/test_dedupe_scanner.py`, add to the existing `TestLoadCheckedPairs` class and `TestRecordPair` class — these currently test the dedupe_scanner internal implementation. After this change, they will test that dedupe_scanner correctly delegates to garden_db.

Add a new class that specifically tests the garden_db delegation:

```python
class TestDedupeUsesGardenDb(unittest.TestCase):
    """Verify dedupe_scanner uses garden_db instead of CHECKED.md."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        from garden_db import init_db
        init_db(self.root)

    def tearDown(self):
        self.tmp.cleanup()

    def test_record_pair_writes_to_garden_db_not_checked_md(self):
        from dedupe_scanner import record_pair as ds_record_pair
        ds_record_pair(self.root, 'GE-0001 × GE-0002', 'distinct', '')
        # garden.db has the entry
        from garden_db import get_pair_result
        self.assertEqual(get_pair_result(self.root, 'GE-0001 × GE-0002'), 'distinct')
        # CHECKED.md does NOT exist
        self.assertFalse((self.root / 'CHECKED.md').exists())

    def test_load_checked_pairs_reads_from_garden_db(self):
        from garden_db import record_pair as db_record
        db_record(self.root, 'GE-0001 × GE-0002', 'distinct')
        from dedupe_scanner import load_checked_pairs as ds_load
        pairs = ds_load(self.root)
        self.assertIn('GE-0001 × GE-0002', pairs)
```

- [ ] **Step 4: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py::TestDedupeUsesGardenDb -v
```

Expected: FAIL — dedupe_scanner still writes to CHECKED.md.

- [ ] **Step 5: Update dedupe_scanner.py**

In `soredium/scripts/dedupe_scanner.py`:

1. Add at the top of the file (after existing imports):
```python
import sys as _sys
_sys.path.insert(0, str(Path(__file__).parent))
from garden_db import (
    load_checked_pairs as _db_load_checked_pairs,
    record_pair as _db_record_pair,
    init_db as _db_init,
)
```

2. Replace the existing `load_checked_pairs(garden)` function body:
```python
def load_checked_pairs(garden: Path) -> set:
    """Return set of canonical pair strings already in garden.db."""
    db_path = garden / 'garden.db'
    if not db_path.exists():
        _db_init(garden)
    return _db_load_checked_pairs(garden)
```

3. Replace the existing `record_pair(garden, pair, result, note)` function body:
```python
def record_pair(garden: Path, pair: str, result: str, note: str = '') -> None:
    """Append a pair comparison result to garden.db. Idempotent."""
    db_path = garden / 'garden.db'
    if not db_path.exists():
        _db_init(garden)
    _db_record_pair(garden, pair, result, note)
```

- [ ] **Step 6: Run all dedupe_scanner tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_dedupe_scanner.py -v
```

Expected: All tests PASS.

- [ ] **Step 7: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 8: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/dedupe_scanner.py tests/test_dedupe_scanner.py tests/test_validate_garden.py
git commit -m "feat(db): dedupe_scanner delegates to garden_db; GardenFixture uses init_db

Refs #29"
```

---

## Task 4: integrate_entry.py — upsert entries_index (TDD)

**Files:**
- Modify: `soredium/scripts/integrate_entry.py`
- Modify: `soredium/tests/test_integrate_entry.py`

After a successful integration, upsert the entry's metadata into `entries_index`. This enables fast domain queries without re-scanning git ls-tree.

- [ ] **Step 1: Write failing test**

Add to `soredium/tests/test_integrate_entry.py`:

```python
class TestEntriesIndexIntegration(unittest.TestCase):
    """integrate_entry upserts into entries_index after successful merge."""

    def _run(self, entry, garden):
        from garden_db import init_db
        init_db(garden)
        with patch('integrate_entry.run_validate'), \
             patch('integrate_entry.git_commit'):
            return integrate(str(entry), str(garden))

    def test_entry_appears_in_entries_index(self, tmp_path):
        garden = _garden_with_drift(tmp_path, drift=0)
        from garden_db import init_db
        init_db(garden)
        entry = garden / 'quarkus' / 'cdi' / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        with patch('integrate_entry.run_validate'), \
             patch('integrate_entry.git_commit'):
            integrate(str(entry), str(garden))
        from garden_db import get_entries_by_domain
        rows = get_entries_by_domain(garden, 'quarkus')
        self.assertGreater(len(rows), 0)

    def test_no_garden_db_no_crash(self, tmp_path):
        """If garden.db doesn't exist, integrate still succeeds (creates db)."""
        garden = _garden_with_drift(tmp_path, drift=0)
        entry = garden / 'quarkus' / 'cdi' / 'GE-0123.md'
        entry.write_text(VALID_ENTRY)
        with patch('integrate_entry.run_validate'), \
             patch('integrate_entry.git_commit'):
            result = integrate(str(entry), str(garden))
        self.assertEqual(result['status'], 'ok')
```

Note: `_garden_with_drift` and `VALID_ENTRY` are already defined in `test_integrate_entry.py`. Read the file first to confirm.

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integrate_entry.py::TestEntriesIndexIntegration -v
```

- [ ] **Step 3: Update integrate_entry.py**

Read `soredium/scripts/integrate_entry.py` to find where the integration result is returned. Then add, after `increment_drift_counter(garden)` and before `run_validate(garden)`:

```python
    # Upsert into entries_index
    try:
        import sys as _sys
        _sys.path.insert(0, str(Path(__file__).parent))
        from garden_db import upsert_entry, init_db
        db_path = Path(garden) / 'garden.db'
        if not db_path.exists():
            init_db(Path(garden))
        fm = _parse_frontmatter_for_index(entry_path, domain)
        if fm:
            upsert_entry(Path(garden), fm)
    except Exception:
        pass  # index failure never blocks integration
```

Also add this helper function to `integrate_entry.py` (after existing imports):

```python
def _parse_frontmatter_for_index(entry_path: Path, domain: str) -> dict | None:
    """Extract frontmatter fields needed for entries_index."""
    import re as _re
    _FM_RE = _re.compile(r'^---\n(.*?)\n---', _re.DOTALL)
    try:
        content = Path(entry_path).read_text(encoding='utf-8').replace('\r\n', '\n')
        m = _FM_RE.match(content)
        if not m:
            return None
        fm_text = m.group(1)
        def _get(key, default=''):
            mm = _re.search(rf'^{key}:\s*(.+)$', fm_text, _re.MULTILINE)
            return mm.group(1).strip().strip('"\'') if mm else default
        tags_raw = _get('tags', '[]')
        tags = [t.strip().strip('"\'') for t in tags_raw.strip('[]').split(',') if t.strip()]
        ge_id = _get('id') or Path(entry_path).stem
        score_raw = _get('score', '0')
        thresh_raw = _get('staleness_threshold', '730')
        return {
            'ge_id': ge_id,
            'title': _get('title'),
            'domain': _get('domain', domain),
            'type': _get('type', 'gotcha'),
            'score': int(score_raw) if score_raw.isdigit() else 0,
            'submitted': _get('submitted'),
            'staleness_threshold': int(thresh_raw) if thresh_raw.isdigit() else 730,
            'tags': tags,
            'verified_on': _get('verified_on'),
            'last_reviewed': _get('last_reviewed'),
            'file_path': f"{domain}/{Path(entry_path).name}",
        }
    except Exception:
        return None
```

- [ ] **Step 4: Run all integrate_entry tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integrate_entry.py -v
```

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/integrate_entry.py tests/test_integrate_entry.py
git commit -m "feat(db): integrate_entry upserts into entries_index after merge

Refs #29"
```

---

## Task 5: init_garden.py — create garden.db + .gitattributes (TDD)

**Files:**
- Modify: `soredium/scripts/init_garden.py`
- Modify: `soredium/tests/test_init_garden.py`

Replace `create_checked_md()` and `create_discarded_md()` with `init_db()`. Add `create_gitattributes()` that writes the sqlite3 textconv config.

- [ ] **Step 1: Write failing tests**

Add to `soredium/tests/test_init_garden.py` in `TestInitGarden`:

```python
    def test_garden_db_created_not_checked_md(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        self.assertTrue((self.root / 'garden.db').exists())
        self.assertFalse((self.root / 'CHECKED.md').exists())
        self.assertFalse((self.root / 'DISCARDED.md').exists())

    def test_garden_db_has_valid_schema(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        from garden_db import get_schema_version, SCHEMA_VERSION
        self.assertEqual(get_schema_version(self.root), SCHEMA_VERSION)

    def test_gitattributes_created(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        gitattributes = self.root / '.gitattributes'
        self.assertTrue(gitattributes.exists())
        self.assertIn('garden.db diff=sqlite3', gitattributes.read_text())
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
python3 -m pytest tests/test_init_garden.py::TestInitGarden::test_garden_db_created_not_checked_md -v
```

- [ ] **Step 3: Update init_garden.py**

Read `soredium/scripts/init_garden.py` to find the `create_checked_md()` and `create_discarded_md()` functions and their call sites in `init_garden()`.

Add these functions (replacing the old ones):

```python
def create_garden_db(root: Path) -> None:
    """Initialise garden.db with SQLite schema. Replaces CHECKED.md + DISCARDED.md."""
    import sys as _sys
    _sys.path.insert(0, str(Path(__file__).parent))
    from garden_db import init_db
    if not (root / 'garden.db').exists():
        init_db(root)


def create_gitattributes(root: Path) -> None:
    """Create .gitattributes with SQLite textconv diff driver config."""
    path = root / '.gitattributes'
    if path.exists():
        return
    path.write_text(
        '# SQLite diff driver — shows human-readable SQL dump in git diff\n'
        'garden.db diff=sqlite3\n',
        encoding='utf-8',
    )
```

In `init_garden()`, replace calls to `create_checked_md` and `create_discarded_md` with calls to `create_garden_db` and `create_gitattributes`. Also update the functions list in the `for fn, args, kwargs in [...]` loop.

- [ ] **Step 4: Run all init_garden tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_init_garden.py -v
```

Some old tests (`test_creates_all_required_files`) will now FAIL because they assert CHECKED.md exists. Update those assertions: replace `(self.root / 'CHECKED.md').exists()` with `(self.root / 'garden.db').exists()`.

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/init_garden.py tests/test_init_garden.py
git commit -m "feat(db): init_garden creates garden.db + .gitattributes instead of CHECKED.md

Refs #29"
```

---

## Task 6: validate_garden.py — --check-db flag (TDD)

**Files:**
- Modify: `soredium/scripts/validate_garden.py`
- Modify: `soredium/tests/test_validate_garden.py`

Add `--check-db` flag that verifies garden.db integrity: schema version present, no checked pairs referencing non-existent GE-IDs. Replace the existing `TestCheckedMdIntegrity` tests with `TestGardenDbIntegrity`.

- [ ] **Step 1: Write failing tests**

Add to `soredium/tests/test_validate_garden.py`:

```python
def run_check_db(garden_root: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(VALIDATOR), '--check-db', str(garden_root)],
        capture_output=True, text=True
    )


class TestGardenDbIntegrity(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_valid_garden_db_exits_0(self):
        from garden_db import init_db
        init_db(self.root)
        result = run_check_db(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_missing_garden_db_exits_1(self):
        result = run_check_db(self.root)
        self.assertEqual(result.returncode, 1)
        self.assertIn('garden.db', result.stdout + result.stderr)

    def test_schema_version_shown_in_output(self):
        from garden_db import init_db
        init_db(self.root)
        result = run_check_db(self.root)
        self.assertIn('schema', result.stdout.lower())
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
python3 -m pytest tests/test_validate_garden.py::TestGardenDbIntegrity -v
```

- [ ] **Step 3: Add --check-db to validate_garden.py**

Read `validate_garden.py` to find where the existing `--freshness` and `--dedupe-check` flags are implemented (at the top of the file as early-exit blocks). Add a similar block:

```python
if '--check-db' in sys.argv:
    import sys as _sys2
    _idx = sys.argv.index('--check-db')
    if _idx + 1 < len(sys.argv) and not sys.argv[_idx + 1].startswith('--'):
        _garden = Path(sys.argv[_idx + 1]).expanduser().resolve()
    else:
        _garden = GARDEN_ROOT if 'GARDEN_ROOT' in dir() else Path.home() / '.hortora' / 'garden'

    _db_path = _garden / 'garden.db'
    if not _db_path.exists():
        print(f"ERROR: garden.db not found in {_garden}", file=sys.stderr)
        sys.exit(1)

    import sqlite3 as _sqlite3
    _conn = _sqlite3.connect(str(_db_path))
    _version_row = _conn.execute(
        "SELECT version FROM schema_version ORDER BY version DESC LIMIT 1"
    ).fetchone()
    if not _version_row:
        print("ERROR: schema_version table empty — garden.db may be corrupt")
        sys.exit(1)
    _checked_count = _conn.execute("SELECT COUNT(*) FROM checked_pairs").fetchone()[0]
    _discarded_count = _conn.execute("SELECT COUNT(*) FROM discarded_entries").fetchone()[0]
    _index_count = _conn.execute("SELECT COUNT(*) FROM entries_index").fetchone()[0]
    _conn.close()
    print(f"garden.db OK — schema version {_version_row[0]}")
    print(f"  checked pairs:    {_checked_count}")
    print(f"  discarded entries: {_discarded_count}")
    print(f"  entries index:    {_index_count}")
    sys.exit(0)
```

- [ ] **Step 4: Run all validate_garden tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py -v
```

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Push all Phase SQLite commits**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_garden.py tests/test_validate_garden.py
git commit -m "feat(db): validate_garden --check-db flag — verify garden.db schema and integrity

Refs #29"
git push
```

---

## Task 7: Update harvest SKILL.md (no tests needed)

**Files:**
- Modify: `soredium/harvest/SKILL.md`

Update DEDUPE Steps 5 and 6 to reference `garden.db` instead of `CHECKED.md`.

- [ ] **Step 1: Read harvest/SKILL.md**

Find Step 5 (Record comparison results) and Step 6 (Reset drift counter). Step 5 currently says "Do NOT append to CHECKED.md manually." Update it to:

In Step 5, replace the sentence about not appending to CHECKED.md:
```
The scanner enforces canonical ID ordering and is idempotent — recording the same pair twice produces one row only. Do NOT modify garden.db directly; always use dedupe_scanner --record.
```

Add a note at the top of DEDUPE workflow:
```
**Requires garden.db:** Run `garden_db_migrate.py <garden>` first if CHECKED.md still exists.
```

- [ ] **Step 2: Verify skill structure tests still pass**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_skill_structure.py -v
```

- [ ] **Step 3: Commit and push**

```bash
cd ~/claude/hortora/soredium
git add harvest/SKILL.md
git commit -m "feat(db): update harvest DEDUPE to reference garden.db instead of CHECKED.md

Refs #29"
git push
```

---

## Self-Review

**Spec coverage (ADR-0004):**

| ADR requirement | Task |
|---|---|
| `checked_pairs` table with pair_hash primary key | Task 1 |
| `discarded_entries` table | Task 1 |
| `entries_index` table with domain/score indices | Task 1 |
| `schema_version` table | Task 1 |
| WAL journal mode | Task 1 |
| Migration from CHECKED.md / DISCARDED.md | Task 2 |
| `--dry-run` migration preview | Task 2 |
| Source files renamed to .bak after migration | Task 2 |
| `dedupe_scanner.py` uses garden_db | Task 3 |
| GardenFixture uses init_db | Task 3 |
| `integrate_entry.py` upserts entries_index | Task 4 |
| `init_garden.py` creates garden.db not CHECKED.md | Task 5 |
| `.gitattributes` with sqlite3 textconv | Task 5 |
| `--check-db` validation flag | Task 6 |
| `harvest SKILL.md` updated | Task 7 |

**No gaps. No placeholders.**

**Type consistency:**
- `garden_db.record_pair(garden: Path, pair: str, result: str, notes: str = '')` — used identically in dedupe_scanner Task 3 and garden_db_migrate Task 2
- `garden_db.load_checked_pairs(garden: Path) -> set` — returns `set[str]` throughout
- `garden_db.upsert_entry(garden: Path, entry: dict)` — `entry` always has `ge_id` key as required
- All functions take `garden: Path` as first argument — consistent throughout
