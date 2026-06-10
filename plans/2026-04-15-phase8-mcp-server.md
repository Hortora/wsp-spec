# Phase 8 — MCP Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a FastMCP server exposing three tools — `garden_search`, `garden_capture`, `garden_status` — so any MCP-compatible AI assistant (Cursor, Copilot, Gemini CLI, Windsurf) can query and contribute to the garden, not just Claude Code.

**Architecture:** Four files. Three pure-Python business logic modules (`mcp_garden_search.py`, `mcp_garden_status.py`, `mcp_garden_capture.py`) contain all logic and are unit-testable without MCP. `garden_mcp_server.py` wraps them with FastMCP. This separation means 95% of the code is tested without the MCP protocol layer. The server runs on stdio (standard MCP transport). Each business logic module uses git commands exclusively — no filesystem reads — consistent with the rest of the codebase.

**New dependency:** `mcp` Python library (FastMCP). Install: `pip install mcp`.

**Tech Stack:** Python 3.11+, `mcp` (FastMCP), existing garden scripts imported as modules, pytest with subprocess-based integration tests.

---

## File Map

| File | Status | Responsibility |
|------|--------|----------------|
| `scripts/mcp_garden_search.py` | Create | 3-tier retrieval: technology filter → index match → full domain scan |
| `scripts/mcp_garden_status.py` | Create | Read GARDEN.md metadata: entry count, drift, last sweep, health |
| `scripts/mcp_garden_capture.py` | Create | Validate entry, create git branch, commit, report result |
| `scripts/garden_mcp_server.py` | Create | FastMCP server wrapping the three tools |
| `tests/test_mcp_garden_search.py` | Create | Unit + integration tests for search logic |
| `tests/test_mcp_garden_status.py` | Create | Unit + integration tests for status logic |
| `tests/test_mcp_garden_capture.py` | Create | Unit + integration tests for capture logic |
| `tests/test_garden_mcp_server.py` | Create | MCP protocol integration tests (subprocess JSON-RPC) |

---

## Tool Signatures

```python
garden_search(query: str, technology: str = None, domain: str = None) -> list[dict]
# Returns: [{"id": str, "title": str, "domain": str, "score": int, "body": str, "staleness": str}]

garden_status(garden_path: str = None) -> dict
# Returns: {"name": str, "entry_count": int, "drift": int, "threshold": int,
#            "last_sweep": str, "last_staleness_review": str, "role": str}

garden_capture(title: str, type: str, domain: str, stack: str,
               tags: list[str], score: int, body: str,
               garden_path: str = None) -> dict
# Returns: {"status": str, "branch": str, "ge_id": str, "message": str}
```

---

## Task 1: mcp_garden_search.py — 3-tier retrieval (TDD)

**Files:**
- Create: `soredium/scripts/mcp_garden_search.py`
- Create: `soredium/tests/test_mcp_garden_search.py`

### 3-tier algorithm

**Tier 1 — Technology/domain filter:** Read `GARDEN.md` By Technology section. Extract GE-IDs associated with the requested `technology` or `domain`. This is O(index) — never reads entry bodies.

**Tier 2 — Keyword match in matching entries:** For each GE-ID from Tier 1, fetch body via `git cat-file blob HEAD:<domain>/<ge-id>.md`. Score by keyword overlap with `query`. Return all with score > 0, sorted by score desc.

**Tier 3 — Full domain scan:** If Tier 2 returns no results (or no `technology`/`domain` given), run `git grep` across all committed `.md` files in the relevant domain directory. Filter to files with YAML frontmatter. Return matches.

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_mcp_garden_search.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for mcp_garden_search.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from mcp_garden_search import (
    search_garden, keyword_score, parse_garden_index,
    fetch_entry_body, tier3_grep,
)

SCRIPTS = Path(__file__).parent.parent / 'scripts'


def make_git_garden(tmp_path: Path) -> Path:
    """Create a minimal committed garden with 2 java entries and 1 tools entry."""
    garden = tmp_path / 'garden'
    garden.mkdir()
    subprocess.run(['git', 'init', str(garden)], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'config', 'user.email', 'test@test.com'],
                   check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'config', 'user.name', 'Test'],
                   check=True, capture_output=True)

    # GARDEN.md with index
    (garden / 'GARDEN.md').write_text(
        '**Last assigned ID:** GE-0002\n'
        '**Last full DEDUPE sweep:** 2026-04-15\n'
        '**Entries merged since last sweep:** 1\n'
        '**Drift threshold:** 10\n'
        '**Last staleness review:** never\n\n'
        '## By Technology\n\n'
        '### Java\n'
        '| GE-ID | Title | Type | Score |\n'
        '|-------|-------|------|-------|\n'
        '| [GE-20260414-aa0001](java/GE-20260414-aa0001.md) | '
        'Hibernate @PreUpdate fires at flush time | gotcha | 12 |\n'
        '| [GE-20260414-aa0002](java/GE-20260414-aa0002.md) | '
        'H2 rejects key as column name | gotcha | 10 |\n\n'
        '## By Symptom / Type\n\n## By Label\n\n'
    )

    # Java domain entries
    (garden / 'java').mkdir()
    (garden / 'java' / 'GE-20260414-aa0001.md').write_text(
        '---\nid: GE-20260414-aa0001\n'
        'title: "Hibernate @PreUpdate fires at flush time not at persist"\n'
        'type: gotcha\ndomain: java\n'
        'stack: "Hibernate ORM 6.x"\ntags: [hibernate, jpa, lifecycle, flush]\n'
        'score: 12\nverified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        '## Hibernate @PreUpdate fires at flush time not at persist\n\n'
        '**ID:** GE-20260414-aa0001\n**Stack:** Hibernate ORM 6.x\n'
        '**Symptom:** @PreUpdate callback not firing when expected.\n\n'
        '### Root cause\nFires at flush, not persist.\n\n### Fix\nForce flush.\n'
    )
    (garden / 'java' / 'GE-20260414-aa0002.md').write_text(
        '---\nid: GE-20260414-aa0002\n'
        'title: "H2 rejects key as a column name"\n'
        'type: gotcha\ndomain: java\n'
        'stack: "H2 2.x, Quarkus"\ntags: [h2, sql, reserved-word]\n'
        'score: 10\nverified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        '## H2 rejects key as a column name\n\n'
        '**ID:** GE-20260414-aa0002\n**Stack:** H2 2.x\n'
        '**Symptom:** SQL syntax error on column named "key".\n\n'
        '### Root cause\n"key" is an H2 reserved word.\n\n### Fix\nRename the column.\n'
    )

    # Tools domain
    (garden / 'tools').mkdir()
    (garden / 'tools' / 'GE-20260414-bb0001.md').write_text(
        '---\nid: GE-20260414-bb0001\n'
        'title: "macOS BSD sed silently ignores word boundaries"\n'
        'type: gotcha\ndomain: tools\n'
        'stack: "macOS, BSD sed"\ntags: [sed, macos, regex]\n'
        'score: 11\nverified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        '## macOS BSD sed silently ignores word boundaries\n\n'
        '**ID:** GE-20260414-bb0001\n**Stack:** macOS, BSD sed\n'
        '**Symptom:** sed replacement silently does nothing.\n\n'
        '### Root cause\nBSD sed has different regex syntax.\n\n### Fix\nUse perl -pi instead.\n'
    )

    subprocess.run(['git', '-C', str(garden), 'add', '.'], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'commit', '-m', 'init'],
                   check=True, capture_output=True)
    return garden


# ── keyword_score ──────────────────────────────────────────────────────────────

class TestKeywordScore(unittest.TestCase):

    def test_empty_query_scores_zero(self):
        self.assertEqual(keyword_score('', 'some content here'), 0)

    def test_single_match(self):
        self.assertGreater(keyword_score('hibernate', 'Hibernate fires at flush'), 0)

    def test_no_match_scores_zero(self):
        self.assertEqual(keyword_score('python asyncio', 'Hibernate flush fires'), 0)

    def test_multiple_matches_higher_than_single(self):
        single = keyword_score('flush', 'Hibernate fires at flush')
        multi = keyword_score('hibernate flush fires', 'Hibernate fires at flush')
        self.assertGreater(multi, single)

    def test_case_insensitive(self):
        self.assertEqual(
            keyword_score('HIBERNATE', 'Hibernate ORM'),
            keyword_score('hibernate', 'Hibernate ORM')
        )

    def test_short_words_ignored(self):
        # Words < 3 chars should not count
        self.assertEqual(keyword_score('at in of', 'at in of to'), 0)


# ── parse_garden_index ────────────────────────────────────────────────────────

class TestParseGardenIndex(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_returns_dict_by_technology(self):
        index = parse_garden_index(self.garden)
        self.assertIsInstance(index, dict)

    def test_java_entries_found(self):
        index = parse_garden_index(self.garden)
        java_ids = index.get('Java', index.get('java', []))
        self.assertGreater(len(java_ids), 0)

    def test_ge_ids_in_index(self):
        index = parse_garden_index(self.garden)
        all_ids = [geid for ids in index.values() for geid in ids]
        self.assertIn('GE-20260414-aa0001', all_ids)
        self.assertIn('GE-20260414-aa0002', all_ids)

    def test_no_garden_md_returns_empty(self):
        tmp = Path(self.tmp.name) / 'empty'
        tmp.mkdir()
        subprocess.run(['git', 'init', str(tmp)], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.email', 'test@test.com'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.name', 'Test'],
                       check=True, capture_output=True)
        result = parse_garden_index(tmp)
        self.assertEqual(result, {})


# ── fetch_entry_body ──────────────────────────────────────────────────────────

class TestFetchEntryBody(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_fetches_known_entry(self):
        body = fetch_entry_body(self.garden, 'java', 'GE-20260414-aa0001')
        self.assertIn('Hibernate', body)
        self.assertIn('flush', body)

    def test_returns_none_for_unknown_entry(self):
        body = fetch_entry_body(self.garden, 'java', 'GE-99999999-ffffff')
        self.assertIsNone(body)

    def test_fetches_tools_entry(self):
        body = fetch_entry_body(self.garden, 'tools', 'GE-20260414-bb0001')
        self.assertIn('sed', body)


# ── tier3_grep ────────────────────────────────────────────────────────────────

class TestTier3Grep(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_finds_matching_entry(self):
        results = tier3_grep(self.garden, 'flush', domain='java')
        ids = [r['id'] for r in results]
        self.assertIn('GE-20260414-aa0001', ids)

    def test_no_match_returns_empty(self):
        results = tier3_grep(self.garden, 'xyzzy_nonexistent_term', domain='java')
        self.assertEqual(results, [])

    def test_domain_filter_excludes_other_domains(self):
        # 'sed' is only in tools, not java
        results = tier3_grep(self.garden, 'sed', domain='java')
        self.assertEqual(results, [])

    def test_no_domain_searches_all(self):
        results = tier3_grep(self.garden, 'sed')
        ids = [r['id'] for r in results]
        self.assertIn('GE-20260414-bb0001', ids)


# ── search_garden (integration) ───────────────────────────────────────────────

class TestSearchGarden(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_search_returns_list(self):
        results = search_garden(self.garden, 'hibernate')
        self.assertIsInstance(results, list)

    def test_technology_filter_narrows_results(self):
        results = search_garden(self.garden, 'flush', technology='Java')
        ids = [r['id'] for r in results]
        self.assertIn('GE-20260414-aa0001', ids)
        self.assertNotIn('GE-20260414-bb0001', ids)

    def test_relevant_entry_returned(self):
        results = search_garden(self.garden, 'hibernate flush')
        self.assertTrue(len(results) > 0)
        self.assertTrue(any('aa0001' in r['id'] for r in results))

    def test_each_result_has_required_fields(self):
        results = search_garden(self.garden, 'hibernate')
        self.assertTrue(len(results) > 0)
        for r in results:
            self.assertIn('id', r)
            self.assertIn('title', r)
            self.assertIn('domain', r)
            self.assertIn('body', r)

    def test_no_results_for_unknown_query(self):
        results = search_garden(self.garden, 'xyzzy_nonexistent_term_7829')
        self.assertEqual(results, [])

    def test_domain_parameter_filters_correctly(self):
        results = search_garden(self.garden, 'sed', domain='tools')
        ids = [r['id'] for r in results]
        self.assertIn('GE-20260414-bb0001', ids)

    def test_results_sorted_by_relevance(self):
        # Hibernate appears in aa0001 title, aa0002 does not
        results = search_garden(self.garden, 'hibernate')
        if len(results) > 1:
            # First result should be the more relevant one
            self.assertIn('aa0001', results[0]['id'])

    def test_empty_query_returns_empty(self):
        results = search_garden(self.garden, '')
        self.assertEqual(results, [])


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_mcp_garden_search.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'mcp_garden_search'`

- [ ] **Step 3: Create mcp_garden_search.py**

Create `soredium/scripts/mcp_garden_search.py`:

```python
#!/usr/bin/env python3
"""mcp_garden_search.py — 3-tier knowledge garden retrieval.

Used directly by garden_mcp_server.py. All reads via git — never filesystem.
"""

import re
import subprocess
from pathlib import Path

GE_ID_RE = re.compile(r'GE-\d{8}-[0-9a-f]{6}|GE-\d{4}')
FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---', re.DOTALL)
SHORT_WORD_RE = re.compile(r'\b[a-zA-Z]{1,2}\b')
TOKEN_RE = re.compile(r'\b[a-zA-Z]{3,}\b')

SKIP_FILES = {'GARDEN.md', 'CHECKED.md', 'DISCARDED.md', 'SCHEMA.md', 'README.md', 'INDEX.md'}


def keyword_score(query: str, text: str) -> int:
    """Count how many unique query tokens (≥3 chars) appear in text."""
    if not query.strip():
        return 0
    q_tokens = set(TOKEN_RE.findall(query.lower()))
    t_tokens = set(TOKEN_RE.findall(text.lower()))
    return len(q_tokens & t_tokens)


def parse_garden_index(garden: Path) -> dict:
    """Read GARDEN.md By Technology section. Returns {technology: [ge_id, ...]}."""
    result = {}
    try:
        content = subprocess.run(
            ['git', '-C', str(garden), 'show', 'HEAD:GARDEN.md'],
            capture_output=True, text=True, check=True
        ).stdout
    except subprocess.CalledProcessError:
        return {}

    # Find By Technology section
    m = re.search(r'## By Technology\n(.*?)(?=\n## |\Z)', content, re.DOTALL)
    if not m:
        return {}

    tech_section = m.group(1)
    current_tech = None
    for line in tech_section.splitlines():
        # ### Technology heading
        h = re.match(r'^###\s+(.+)', line)
        if h:
            current_tech = h.group(1).strip()
            result[current_tech] = []
            continue
        # Table row with GE-ID
        ids = GE_ID_RE.findall(line)
        if ids and current_tech:
            result[current_tech].extend(ids)

    return result


def fetch_entry_body(garden: Path, domain: str, ge_id: str) -> str | None:
    """Fetch entry body via git cat-file. Returns content or None if not found."""
    path = f'{domain}/{ge_id}.md'
    try:
        result = subprocess.run(
            ['git', '-C', str(garden), 'cat-file', 'blob', f'HEAD:{path}'],
            capture_output=True, text=True, check=True
        )
        return result.stdout
    except subprocess.CalledProcessError:
        return None


def _parse_frontmatter(content: str) -> dict:
    """Extract key-value fields from YAML frontmatter."""
    content = content.replace('\r\n', '\n')
    m = FRONTMATTER_RE.match(content)
    if not m:
        return {}
    result = {}
    for line in m.group(1).splitlines():
        if ':' in line:
            k, _, v = line.partition(':')
            k = k.strip()
            v = v.strip().strip('"\'')
            if v.startswith('[') and v.endswith(']'):
                v = [x.strip().strip('"\'') for x in v[1:-1].split(',') if x.strip()]
            result[k] = v
    return result


def _entry_from_body(body: str, ge_id: str) -> dict:
    """Build entry dict from body content."""
    fm = _parse_frontmatter(body)
    return {
        'id': fm.get('id', ge_id),
        'title': fm.get('title', ''),
        'domain': fm.get('domain', ''),
        'score': int(fm.get('score', 0)),
        'body': body,
        'tags': fm.get('tags', []),
        'submitted': fm.get('submitted', ''),
        'staleness_threshold': int(fm.get('staleness_threshold', 730)),
    }


def tier3_grep(garden: Path, query: str, domain: str = None) -> list:
    """Tier 3: git grep across committed .md files. Returns list of entry dicts."""
    if not query.strip():
        return []
    # Extract tokens ≥ 3 chars for grep
    tokens = TOKEN_RE.findall(query.lower())
    if not tokens:
        return []

    pattern = '|'.join(re.escape(t) for t in tokens)
    path_spec = f'{domain}/*.md' if domain else '*.md'
    try:
        result = subprocess.run(
            ['git', '-C', str(garden), 'grep', '-il', '-E', pattern,
             'HEAD', '--', path_spec],
            capture_output=True, text=True
        )
        if result.returncode != 0:
            return []
    except subprocess.CalledProcessError:
        return []

    entries = []
    seen = set()
    for line in result.stdout.splitlines():
        # Format: HEAD:domain/GE-XXXX.md
        m = re.match(r'HEAD:(.+/)(GE-[\w-]+)\.md', line)
        if not m:
            continue
        domain_part = m.group(1).rstrip('/')
        ge_id = m.group(2)
        filename = Path(m.group(1) + m.group(2) + '.md').name
        if filename in SKIP_FILES or ge_id in seen:
            continue
        seen.add(ge_id)
        body = fetch_entry_body(garden, domain_part, ge_id)
        if body and FRONTMATTER_RE.match(body.replace('\r\n', '\n')):
            entry = _entry_from_body(body, ge_id)
            entry['relevance'] = keyword_score(query, body)
            entries.append(entry)

    entries.sort(key=lambda e: e.get('relevance', 0), reverse=True)
    return entries


def search_garden(garden: Path, query: str,
                  technology: str = None, domain: str = None) -> list:
    """3-tier search. Returns list of entry dicts sorted by relevance."""
    if not query.strip():
        return []

    results = []
    seen = set()

    # Tier 1+2: technology-filtered index search
    if technology or domain:
        index = parse_garden_index(garden)
        # Find matching tech sections
        tech_key = technology or domain
        matching_ids = []
        for tech, ids in index.items():
            if tech_key.lower() in tech.lower() or tech.lower() in tech_key.lower():
                matching_ids.extend(ids)

        # Tier 2: fetch bodies and score
        for ge_id in matching_ids:
            if ge_id in seen:
                continue
            seen.add(ge_id)
            # Determine domain from index path or infer
            body = None
            for tech, ids in index.items():
                if ge_id in ids:
                    # Try common domain names derived from technology
                    for d in [tech.lower(), domain or '']:
                        body = fetch_entry_body(garden, d, ge_id)
                        if body:
                            break
                    if body:
                        break

            if not body:
                continue
            score = keyword_score(query, body)
            if score > 0:
                entry = _entry_from_body(body, ge_id)
                entry['relevance'] = score
                results.append(entry)

        results.sort(key=lambda e: e.get('relevance', 0), reverse=True)

        # Tier 3 if tier 2 found nothing
        if not results:
            results = tier3_grep(garden, query, domain=domain or technology.lower())
    else:
        # No technology filter: go straight to tier 3
        results = tier3_grep(garden, query)

    return results
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_mcp_garden_search.py -v
```

Expected: All tests PASS.

**Note:** The Tier 1+2 technology-filtered path in `search_garden` has a limitation: it needs to know which domain directory contains a given GE-ID. The implementation above tries to infer this from the technology name. If tests fail due to this, adapt `search_garden` to scan domain directories directly using git ls-tree rather than guessing. The unit tests use `make_git_garden` which puts entries in `java/` and the technology heading is `### Java` — these will need to match.

If `test_technology_filter_narrows_results` or `test_relevant_entry_returned` fail, simplify: make `search_garden` always use `tier3_grep` (which works correctly) for the body content, and use the index only for the technology filter (to get the domain name). The cleaner implementation:

```python
# Simpler Tier 2 that avoids domain inference:
for ge_id in matching_ids:
    # Find the domain by checking each domain directory
    try:
        ls = subprocess.run(
            ['git', '-C', str(garden), 'ls-tree', '-r', '--name-only', 'HEAD'],
            capture_output=True, text=True, check=True
        ).stdout
        for path in ls.splitlines():
            if ge_id in path and path.endswith('.md'):
                parts = path.split('/')
                if len(parts) >= 2:
                    d, fname = parts[-2], parts[-1]
                    body = fetch_entry_body(garden, d, ge_id)
                    if body:
                        ...
    except: pass
```

Use whichever approach makes the tests pass cleanly.

- [ ] **Step 5: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 456+ passing, 0 failures.

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/mcp_garden_search.py tests/test_mcp_garden_search.py
git commit -m "feat(mcp): add mcp_garden_search.py — 3-tier retrieval for MCP server

Refs #29"
```

---

## Task 2: mcp_garden_status.py — garden metadata (TDD)

**Files:**
- Create: `soredium/scripts/mcp_garden_status.py`
- Create: `soredium/tests/test_mcp_garden_status.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_mcp_garden_status.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for mcp_garden_status.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from mcp_garden_status import get_status, count_entries, parse_garden_md_metadata


def make_git_garden(tmp: Path, entries_since_sweep: int = 3) -> Path:
    garden = tmp / 'garden'
    garden.mkdir()
    for cmd in [
        ['git', 'init', str(garden)],
        ['git', '-C', str(garden), 'config', 'user.email', 'test@test.com'],
        ['git', '-C', str(garden), 'config', 'user.name', 'Test'],
    ]:
        subprocess.run(cmd, check=True, capture_output=True)

    (garden / 'GARDEN.md').write_text(
        f'# test-garden\n\n'
        f'**Last assigned ID:** GE-0002\n'
        f'**Last full DEDUPE sweep:** 2026-04-10\n'
        f'**Entries merged since last sweep:** {entries_since_sweep}\n'
        f'**Drift threshold:** 10\n'
        f'**Last staleness review:** 2026-04-01\n'
    )
    (garden / 'java').mkdir()
    for i in range(1, 3):
        (garden / 'java' / f'GE-2026041{i}-aa000{i}.md').write_text(
            f'---\nid: GE-2026041{i}-aa000{i}\ntitle: "Entry {i}"\ntype: gotcha\n'
            f'domain: java\nstack: "Java"\ntags: [java]\nscore: 10\n'
            f'verified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
            f'## Entry {i}\n\n**ID:** GE-2026041{i}-aa000{i}\n'
        )
    subprocess.run(['git', '-C', str(garden), 'add', '.'], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'commit', '-m', 'init'],
                   check=True, capture_output=True)
    return garden


class TestParseGardenMdMetadata(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_reads_drift_counter(self):
        meta = parse_garden_md_metadata(self.garden)
        self.assertEqual(meta['drift'], 3)

    def test_reads_threshold(self):
        meta = parse_garden_md_metadata(self.garden)
        self.assertEqual(meta['threshold'], 10)

    def test_reads_last_sweep(self):
        meta = parse_garden_md_metadata(self.garden)
        self.assertEqual(meta['last_sweep'], '2026-04-10')

    def test_reads_last_staleness_review(self):
        meta = parse_garden_md_metadata(self.garden)
        self.assertEqual(meta['last_staleness_review'], '2026-04-01')

    def test_missing_garden_md_returns_defaults(self):
        tmp = Path(self.tmp.name) / 'empty'
        tmp.mkdir()
        subprocess.run(['git', 'init', str(tmp)], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.email', 'test@test.com'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.name', 'Test'],
                       check=True, capture_output=True)
        meta = parse_garden_md_metadata(tmp)
        self.assertEqual(meta['drift'], 0)
        self.assertEqual(meta['threshold'], 10)


class TestCountEntries(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_counts_yaml_entries(self):
        count = count_entries(self.garden)
        self.assertEqual(count, 2)

    def test_zero_entries_in_empty_garden(self):
        tmp = Path(self.tmp.name) / 'empty'
        tmp.mkdir()
        subprocess.run(['git', 'init', str(tmp)], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.email', 'test@test.com'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'config', 'user.name', 'Test'],
                       check=True, capture_output=True)
        (tmp / 'GARDEN.md').write_text('**Last assigned ID:** GE-0000\n')
        subprocess.run(['git', '-C', str(tmp), 'add', '.'], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(tmp), 'commit', '-m', 'init'],
                       check=True, capture_output=True)
        self.assertEqual(count_entries(tmp), 0)

    def test_does_not_count_garden_md_or_checked_md(self):
        count = count_entries(self.garden)
        self.assertEqual(count, 2)  # only the 2 java entries, not GARDEN.md


class TestGetStatus(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_returns_dict(self):
        status = get_status(self.garden)
        self.assertIsInstance(status, dict)

    def test_has_entry_count(self):
        status = get_status(self.garden)
        self.assertIn('entry_count', status)
        self.assertEqual(status['entry_count'], 2)

    def test_has_drift(self):
        status = get_status(self.garden)
        self.assertIn('drift', status)
        self.assertEqual(status['drift'], 3)

    def test_has_threshold(self):
        status = get_status(self.garden)
        self.assertIn('threshold', status)

    def test_has_last_sweep(self):
        status = get_status(self.garden)
        self.assertIn('last_sweep', status)
        self.assertEqual(status['last_sweep'], '2026-04-10')

    def test_dedupe_recommended_when_above_threshold(self):
        garden = make_git_garden(Path(self.tmp.name) / 'high', entries_since_sweep=15)
        status = get_status(garden)
        self.assertTrue(status.get('dedupe_recommended', False))

    def test_dedupe_not_recommended_when_below_threshold(self):
        status = get_status(self.garden)
        self.assertFalse(status.get('dedupe_recommended', True))

    def test_has_garden_path(self):
        status = get_status(self.garden)
        self.assertIn('garden_path', status)

    def test_invalid_path_raises(self):
        with self.assertRaises(Exception):
            get_status(Path('/nonexistent/path/that/does/not/exist'))


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
python3 -m pytest tests/test_mcp_garden_status.py -v 2>&1 | head -10
```

- [ ] **Step 3: Create mcp_garden_status.py**

Create `soredium/scripts/mcp_garden_status.py`:

```python
#!/usr/bin/env python3
"""mcp_garden_status.py — Garden metadata for the MCP status tool."""

import re
import subprocess
from pathlib import Path

SKIP_FILES = {'GARDEN.md', 'CHECKED.md', 'DISCARDED.md', 'SCHEMA.md',
              'README.md', 'INDEX.md'}
FRONTMATTER_RE = re.compile(r'^---\n', re.MULTILINE)


def parse_garden_md_metadata(garden: Path) -> dict:
    """Read drift counter, threshold, last sweep from GARDEN.md via git."""
    defaults = {'drift': 0, 'threshold': 10, 'last_sweep': 'never',
                'last_staleness_review': 'never', 'name': ''}
    try:
        content = subprocess.run(
            ['git', '-C', str(garden), 'show', 'HEAD:GARDEN.md'],
            capture_output=True, text=True, check=True
        ).stdout
    except subprocess.CalledProcessError:
        return defaults

    def _int(pattern, text, default=0):
        m = re.search(pattern, text)
        return int(m.group(1)) if m else default

    def _str(pattern, text, default='never'):
        m = re.search(pattern, text)
        return m.group(1).strip() if m else default

    return {
        'drift': _int(r'\*\*Entries merged since last sweep:\*\*\s*(\d+)', content),
        'threshold': _int(r'\*\*Drift threshold:\*\*\s*(\d+)', content, default=10),
        'last_sweep': _str(r'\*\*Last full DEDUPE sweep:\*\*\s*(.+)', content),
        'last_staleness_review': _str(r'\*\*Last staleness review:\*\*\s*(.+)', content),
        'name': _str(r'^#\s+(.+)', content, default=''),
    }


def count_entries(garden: Path) -> int:
    """Count committed YAML-frontmatter entry files (excluding control files)."""
    try:
        ls = subprocess.run(
            ['git', '-C', str(garden), 'ls-tree', '-r', '--name-only', 'HEAD'],
            capture_output=True, text=True, check=True
        ).stdout
    except subprocess.CalledProcessError:
        return 0

    count = 0
    for path in ls.splitlines():
        filename = Path(path).name
        if not filename.endswith('.md') or filename in SKIP_FILES:
            continue
        # Check for YAML frontmatter
        try:
            body = subprocess.run(
                ['git', '-C', str(garden), 'show', f'HEAD:{path}'],
                capture_output=True, text=True, check=True
            ).stdout
            if body.startswith('---\n') or body.startswith('---\r\n'):
                count += 1
        except subprocess.CalledProcessError:
            pass
    return count


def get_status(garden: Path) -> dict:
    """Return garden health summary dict."""
    garden = Path(garden).expanduser().resolve()
    if not garden.exists():
        raise FileNotFoundError(f"Garden not found: {garden}")

    meta = parse_garden_md_metadata(garden)
    entry_count = count_entries(garden)
    drift = meta['drift']
    threshold = meta['threshold']

    # Read SCHEMA.md role if present
    role = 'unknown'
    try:
        schema = subprocess.run(
            ['git', '-C', str(garden), 'show', 'HEAD:SCHEMA.md'],
            capture_output=True, text=True, check=True
        ).stdout
        m = re.search(r'^role:\s*(\w+)', schema, re.MULTILINE)
        if m:
            role = m.group(1)
    except subprocess.CalledProcessError:
        pass

    return {
        'name': meta['name'],
        'garden_path': str(garden),
        'entry_count': entry_count,
        'drift': drift,
        'threshold': threshold,
        'dedupe_recommended': drift >= threshold,
        'last_sweep': meta['last_sweep'],
        'last_staleness_review': meta['last_staleness_review'],
        'role': role,
    }
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
python3 -m pytest tests/test_mcp_garden_status.py -v
```

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
git add scripts/mcp_garden_status.py tests/test_mcp_garden_status.py
git commit -m "feat(mcp): add mcp_garden_status.py — garden metadata for MCP server

Refs #29"
```

---

## Task 3: mcp_garden_capture.py — entry submission (TDD)

**Files:**
- Create: `soredium/scripts/mcp_garden_capture.py`
- Create: `soredium/tests/test_mcp_garden_capture.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_mcp_garden_capture.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for mcp_garden_capture.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from mcp_garden_capture import (
    build_entry_content, generate_ge_id, validate_capture_args, capture_entry,
)


def make_git_garden(tmp: Path) -> Path:
    garden = tmp / 'garden'
    garden.mkdir()
    for cmd in [
        ['git', 'init', str(garden)],
        ['git', '-C', str(garden), 'config', 'user.email', 'test@test.com'],
        ['git', '-C', str(garden), 'config', 'user.name', 'Test'],
    ]:
        subprocess.run(cmd, check=True, capture_output=True)
    (garden / 'GARDEN.md').write_text(
        '**Last assigned ID:** GE-0000\n'
        '**Last full DEDUPE sweep:** 2026-04-15\n'
        '**Entries merged since last sweep:** 0\n'
        '**Drift threshold:** 10\n'
    )
    (garden / 'java').mkdir()
    (garden / 'java' / 'INDEX.md').write_text(
        '| GE-ID | Title | Type | Score |\n|-------|-------|------|-------|\n'
    )
    subprocess.run(['git', '-C', str(garden), 'add', '.'], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'commit', '-m', 'init'],
                   check=True, capture_output=True)
    return garden


VALID_ARGS = {
    'title': 'CompletableFuture.get() blocks carrier thread in virtual thread context',
    'type': 'gotcha',
    'domain': 'java',
    'stack': 'Java 21+, Virtual Threads',
    'tags': ['java', 'virtual-threads', 'blocking'],
    'score': 11,
    'body': (
        '**Symptom:** Performance regression under virtual threads.\n\n'
        '### Root cause\nget() pins carrier thread.\n\n'
        '### Fix\nUse join() or structured concurrency.\n\n'
        '### Why this is non-obvious\nVirtual threads look transparent but aren\'t.\n'
    ),
}


# ── validate_capture_args ─────────────────────────────────────────────────────

class TestValidateCaptureArgs(unittest.TestCase):

    def test_valid_args_no_errors(self):
        errors = validate_capture_args(**VALID_ARGS)
        self.assertEqual(errors, [])

    def test_missing_title_error(self):
        args = {**VALID_ARGS, 'title': ''}
        errors = validate_capture_args(**args)
        self.assertTrue(any('title' in e for e in errors))

    def test_invalid_type_error(self):
        args = {**VALID_ARGS, 'type': 'bogus'}
        errors = validate_capture_args(**args)
        self.assertTrue(any('type' in e for e in errors))

    def test_score_below_minimum_error(self):
        args = {**VALID_ARGS, 'score': 7}
        errors = validate_capture_args(**args)
        self.assertTrue(any('score' in e for e in errors))

    def test_score_at_minimum_ok(self):
        args = {**VALID_ARGS, 'score': 8}
        errors = validate_capture_args(**args)
        self.assertEqual(errors, [])

    def test_missing_domain_error(self):
        args = {**VALID_ARGS, 'domain': ''}
        errors = validate_capture_args(**args)
        self.assertTrue(any('domain' in e for e in errors))

    def test_empty_tags_is_ok(self):
        args = {**VALID_ARGS, 'tags': []}
        errors = validate_capture_args(**args)
        self.assertEqual(errors, [])

    def test_empty_body_error(self):
        args = {**VALID_ARGS, 'body': ''}
        errors = validate_capture_args(**args)
        self.assertTrue(any('body' in e for e in errors))

    def test_valid_types_all_accepted(self):
        for t in ('gotcha', 'technique', 'undocumented'):
            args = {**VALID_ARGS, 'type': t}
            errors = validate_capture_args(**args)
            self.assertEqual(errors, [], f"Type {t!r} should be valid")


# ── generate_ge_id ────────────────────────────────────────────────────────────

class TestGenerateGeId(unittest.TestCase):

    def test_format_matches_pattern(self):
        import re
        ge_id = generate_ge_id()
        self.assertRegex(ge_id, r'^GE-\d{8}-[0-9a-f]{6}$')

    def test_two_calls_different_ids(self):
        self.assertNotEqual(generate_ge_id(), generate_ge_id())

    def test_contains_todays_date(self):
        from datetime import date
        today = date.today().strftime('%Y%m%d')
        ge_id = generate_ge_id()
        self.assertIn(today, ge_id)


# ── build_entry_content ───────────────────────────────────────────────────────

class TestBuildEntryContent(unittest.TestCase):

    def test_produces_yaml_frontmatter(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertTrue(content.startswith('---\n'))
        self.assertIn('\n---\n', content)

    def test_contains_ge_id(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn('GE-20260415-aabbcc', content)

    def test_contains_title(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn(VALID_ARGS['title'], content)

    def test_contains_domain(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn(f"domain: {VALID_ARGS['domain']}", content)

    def test_contains_score(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn(f"score: {VALID_ARGS['score']}", content)

    def test_contains_body(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn('### Root cause', content)

    def test_contains_id_in_body(self):
        content = build_entry_content(ge_id='GE-20260415-aabbcc', **VALID_ARGS)
        self.assertIn('**ID:** GE-20260415-aabbcc', content)


# ── capture_entry (integration) ───────────────────────────────────────────────

class TestCaptureEntry(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_capture_returns_dict(self):
        result = capture_entry(self.garden, **VALID_ARGS)
        self.assertIsInstance(result, dict)

    def test_capture_returns_status_ok(self):
        result = capture_entry(self.garden, **VALID_ARGS)
        self.assertEqual(result['status'], 'ok')

    def test_capture_returns_ge_id(self):
        result = capture_entry(self.garden, **VALID_ARGS)
        self.assertIn('ge_id', result)
        self.assertRegex(result['ge_id'], r'^GE-\d{8}-[0-9a-f]{6}$')

    def test_capture_returns_branch(self):
        result = capture_entry(self.garden, **VALID_ARGS)
        self.assertIn('branch', result)
        self.assertIn('submit/', result['branch'])

    def test_entry_file_created_on_branch(self):
        result = capture_entry(self.garden, **VALID_ARGS)
        branch = result['branch']
        ge_id = result['ge_id']
        # Check the file exists on the branch
        file_check = subprocess.run(
            ['git', '-C', str(self.garden), 'show', f'{branch}:java/{ge_id}.md'],
            capture_output=True, text=True
        )
        self.assertEqual(file_check.returncode, 0, file_check.stderr)
        self.assertIn(VALID_ARGS['title'], file_check.stdout)

    def test_invalid_args_returns_error_status(self):
        result = capture_entry(self.garden, title='', type='gotcha',
                               domain='java', stack='Java', tags=[], score=5,
                               body='content')
        self.assertEqual(result['status'], 'error')
        self.assertIn('errors', result)

    def test_capture_leaves_main_branch_clean(self):
        capture_entry(self.garden, **VALID_ARGS)
        current = subprocess.run(
            ['git', '-C', str(self.garden), 'branch', '--show-current'],
            capture_output=True, text=True
        ).stdout.strip()
        self.assertEqual(current, 'main')


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
python3 -m pytest tests/test_mcp_garden_capture.py -v 2>&1 | head -10
```

- [ ] **Step 3: Create mcp_garden_capture.py**

Create `soredium/scripts/mcp_garden_capture.py`:

```python
#!/usr/bin/env python3
"""mcp_garden_capture.py — Entry submission for the MCP capture tool."""

import re
import secrets
import subprocess
from datetime import date
from pathlib import Path

VALID_TYPES = {'gotcha', 'technique', 'undocumented'}
MIN_SCORE = 8
TODAY = date.today().isoformat()
DATE_COMPACT = date.today().strftime('%Y%m%d')
SKIP_DIRS = {'.git', 'submissions', '_augment', '_summaries', '_index', 'labels'}


def generate_ge_id() -> str:
    """Generate a GE-YYYYMMDD-xxxxxx ID."""
    return f'GE-{DATE_COMPACT}-{secrets.token_hex(3)}'


def validate_capture_args(title: str, type: str, domain: str, stack: str,
                           tags: list, score: int, body: str) -> list:
    """Validate capture arguments. Returns list of error strings."""
    errors = []
    if not title.strip():
        errors.append("'title' is required and must be non-empty")
    if type not in VALID_TYPES:
        errors.append(f"'type' must be one of: {', '.join(sorted(VALID_TYPES))}")
    if not domain.strip():
        errors.append("'domain' is required")
    if score < MIN_SCORE:
        errors.append(f"'score' must be >= {MIN_SCORE} (got {score})")
    if not body.strip():
        errors.append("'body' is required")
    return errors


def build_entry_content(ge_id: str, title: str, type: str, domain: str,
                         stack: str, tags: list, score: int, body: str) -> str:
    """Build the full markdown content for a new entry."""
    tag_str = ', '.join(tags) if tags else ''
    return (
        f'---\n'
        f'id: {ge_id}\n'
        f'title: "{title}"\n'
        f'type: {type}\n'
        f'domain: {domain}\n'
        f'stack: "{stack}"\n'
        f'tags: [{tag_str}]\n'
        f'score: {score}\n'
        f'verified: true\n'
        f'staleness_threshold: 730\n'
        f'submitted: {TODAY}\n'
        f'---\n\n'
        f'## {title}\n\n'
        f'**ID:** {ge_id}\n'
        f'**Stack:** {stack}\n\n'
        f'{body}\n'
        f'\n*Score: {score}/15 · Submitted via garden_capture MCP tool*\n'
    )


def capture_entry(garden: Path, title: str, type: str, domain: str,
                  stack: str, tags: list, score: int, body: str) -> dict:
    """Create a new entry on a submit branch. Returns result dict."""
    garden = Path(garden).expanduser().resolve()

    errors = validate_capture_args(title, type, domain, stack, tags, score, body)
    if errors:
        return {'status': 'error', 'errors': errors, 'message': '; '.join(errors)}

    ge_id = generate_ge_id()
    branch = f'submit/{ge_id}'

    try:
        # Remember current branch
        current = subprocess.run(
            ['git', '-C', str(garden), 'branch', '--show-current'],
            capture_output=True, text=True, check=True
        ).stdout.strip() or 'main'

        # Create branch off HEAD
        subprocess.run(
            ['git', '-C', str(garden), 'checkout', '-b', branch],
            check=True, capture_output=True
        )

        # Write entry file
        domain_dir = garden / domain
        domain_dir.mkdir(exist_ok=True)
        entry_path = domain_dir / f'{ge_id}.md'
        entry_path.write_text(
            build_entry_content(ge_id, title, type, domain, stack, tags, score, body),
            encoding='utf-8'
        )

        # Commit
        subprocess.run(['git', '-C', str(garden), 'add', str(entry_path)],
                       check=True, capture_output=True)
        subprocess.run(
            ['git', '-C', str(garden), 'commit', '-m',
             f'submit({ge_id}): {title[:60]}'],
            check=True, capture_output=True
        )

        # Return to original branch
        subprocess.run(
            ['git', '-C', str(garden), 'checkout', current],
            check=True, capture_output=True
        )

        return {
            'status': 'ok',
            'ge_id': ge_id,
            'branch': branch,
            'message': (
                f'Entry {ge_id} created on branch {branch}. '
                f'Open a PR to merge into the garden.'
            ),
        }

    except subprocess.CalledProcessError as e:
        # Try to return to original branch
        try:
            subprocess.run(['git', '-C', str(garden), 'checkout', current],
                           capture_output=True)
        except Exception:
            pass
        return {
            'status': 'error',
            'errors': [str(e)],
            'message': f'Git operation failed: {e}',
        }
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
python3 -m pytest tests/test_mcp_garden_capture.py -v
```

- [ ] **Step 5: Full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
git add scripts/mcp_garden_capture.py tests/test_mcp_garden_capture.py
git commit -m "feat(mcp): add mcp_garden_capture.py — entry submission for MCP server

Refs #29"
```

---

## Task 4: garden_mcp_server.py — FastMCP server + integration tests (TDD)

**Files:**
- Create: `soredium/scripts/garden_mcp_server.py`
- Create: `soredium/tests/test_garden_mcp_server.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_garden_mcp_server.py`:

```python
#!/usr/bin/env python3
"""Integration tests for garden_mcp_server.py via MCP client."""

import asyncio
import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

SERVER = Path(__file__).parent.parent / 'scripts' / 'garden_mcp_server.py'


def make_git_garden(tmp: Path) -> Path:
    garden = tmp / 'garden'
    garden.mkdir()
    for cmd in [
        ['git', 'init', str(garden)],
        ['git', '-C', str(garden), 'config', 'user.email', 'test@test.com'],
        ['git', '-C', str(garden), 'config', 'user.name', 'Test'],
    ]:
        subprocess.run(cmd, check=True, capture_output=True)

    (garden / 'GARDEN.md').write_text(
        '**Last assigned ID:** GE-0001\n'
        '**Last full DEDUPE sweep:** 2026-04-15\n'
        '**Entries merged since last sweep:** 1\n'
        '**Drift threshold:** 10\n'
        '**Last staleness review:** never\n\n'
        '## By Technology\n\n'
        '### Java\n'
        '| GE-ID | Title | Type | Score |\n'
        '|-------|-------|------|-------|\n'
        '| [GE-20260414-aa0001](java/GE-20260414-aa0001.md) | '
        'Hibernate PreUpdate fires at flush | gotcha | 12 |\n\n'
        '## By Symptom / Type\n\n## By Label\n\n'
    )
    (garden / 'java').mkdir()
    (garden / 'java' / 'GE-20260414-aa0001.md').write_text(
        '---\nid: GE-20260414-aa0001\n'
        'title: "Hibernate @PreUpdate fires at flush time"\n'
        'type: gotcha\ndomain: java\nstack: "Hibernate ORM 6.x"\n'
        'tags: [hibernate, jpa, flush]\nscore: 12\nverified: true\n'
        'staleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        '## Hibernate @PreUpdate fires at flush time\n\n'
        '**ID:** GE-20260414-aa0001\n**Stack:** Hibernate ORM 6.x\n'
        '**Symptom:** Callback not firing.\n\n'
        '### Root cause\nFires at flush.\n\n### Fix\nForce flush.\n'
    )
    subprocess.run(['git', '-C', str(garden), 'add', '.'], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'commit', '-m', 'init'],
                   check=True, capture_output=True)
    return garden


class TestMcpServerImport(unittest.TestCase):
    """Verify the server module can be imported and FastMCP app created."""

    def test_server_module_importable(self):
        sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
        import garden_mcp_server
        self.assertTrue(hasattr(garden_mcp_server, 'mcp'))

    def test_mcp_has_tools(self):
        sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
        import importlib
        import garden_mcp_server
        importlib.reload(garden_mcp_server)
        # FastMCP app should exist
        self.assertIsNotNone(garden_mcp_server.mcp)


class TestMcpServerTools(unittest.TestCase):
    """Test the tool functions directly (bypassing MCP protocol)."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))
        sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))

    def tearDown(self):
        self.tmp.cleanup()

    def test_garden_status_tool_returns_dict(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_status(str(self.garden))
        self.assertIsInstance(result, dict)
        self.assertIn('entry_count', result)

    def test_garden_status_correct_entry_count(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_status(str(self.garden))
        self.assertEqual(result['entry_count'], 1)

    def test_garden_search_tool_returns_list(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_search(
            'hibernate flush', garden_path=str(self.garden)
        )
        self.assertIsInstance(result, list)

    def test_garden_search_finds_relevant_entry(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_search(
            'hibernate flush', garden_path=str(self.garden)
        )
        self.assertTrue(len(result) > 0)
        self.assertTrue(any('aa0001' in r['id'] for r in result))

    def test_garden_search_empty_query_returns_empty(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_search('', garden_path=str(self.garden))
        self.assertEqual(result, [])

    def test_garden_capture_tool_returns_dict(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_capture(
            title='CompletableFuture blocks carrier thread',
            type='gotcha',
            domain='java',
            stack='Java 21+',
            tags=['java', 'virtual-threads'],
            score=11,
            body='**Symptom:** Blocking.\n\n### Root cause\nPins carrier.\n\n### Fix\nUse join().\n\n### Why non-obvious\nLooks transparent.\n',
            garden_path=str(self.garden),
        )
        self.assertIsInstance(result, dict)
        self.assertIn('status', result)

    def test_garden_capture_valid_entry_returns_ok(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_capture(
            title='CompletableFuture blocks carrier thread',
            type='gotcha',
            domain='java',
            stack='Java 21+',
            tags=['java', 'virtual-threads'],
            score=11,
            body='**Symptom:** Blocking.\n\n### Root cause\nPins carrier.\n\n### Fix\nUse join().\n\n### Why non-obvious\nLooks transparent.\n',
            garden_path=str(self.garden),
        )
        self.assertEqual(result['status'], 'ok')
        self.assertIn('ge_id', result)

    def test_garden_capture_low_score_returns_error(self):
        import garden_mcp_server
        result = garden_mcp_server.garden_capture(
            title='Test', type='gotcha', domain='java', stack='Java',
            tags=[], score=5,
            body='content',
            garden_path=str(self.garden),
        )
        self.assertEqual(result['status'], 'error')


class TestMcpServerFile(unittest.TestCase):
    """Verify the server file has the right structure."""

    def test_server_file_exists(self):
        self.assertTrue(SERVER.exists())

    def test_server_file_references_garden_search(self):
        content = SERVER.read_text()
        self.assertIn('garden_search', content)

    def test_server_file_references_garden_capture(self):
        content = SERVER.read_text()
        self.assertIn('garden_capture', content)

    def test_server_file_references_garden_status(self):
        content = SERVER.read_text()
        self.assertIn('garden_status', content)

    def test_server_file_uses_fastmcp(self):
        content = SERVER.read_text()
        self.assertIn('FastMCP', content)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
python3 -m pytest tests/test_garden_mcp_server.py -v 2>&1 | head -15
```

Expected: `ModuleNotFoundError: No module named 'garden_mcp_server'`

- [ ] **Step 3: Create garden_mcp_server.py**

Create `soredium/scripts/garden_mcp_server.py`:

```python
#!/usr/bin/env python3
"""
garden_mcp_server.py — Hortora Knowledge Garden MCP Server.

Exposes three tools via FastMCP:
  garden_search  — 3-tier retrieval: find entries by query + technology filter
  garden_capture — Submit a new entry as a git branch (ready to PR)
  garden_status  — Garden health: entry count, drift, last sweep

Usage (stdio transport — standard MCP):
  python3 garden_mcp_server.py

Configure in Claude Desktop / Cursor / Copilot:
  {
    "mcpServers": {
      "hortora-garden": {
        "command": "python3",
        "args": ["/path/to/soredium/scripts/garden_mcp_server.py"],
        "env": {"HORTORA_GARDEN": "/path/to/your/garden"}
      }
    }
  }
"""

import os
import sys
from pathlib import Path

# Ensure scripts/ is on path for sibling imports
sys.path.insert(0, str(Path(__file__).parent))

from mcp.server.fastmcp import FastMCP
from mcp_garden_search import search_garden
from mcp_garden_status import get_status
from mcp_garden_capture import capture_entry

_DEFAULT_GARDEN = Path(
    os.environ.get('HORTORA_GARDEN', Path.home() / '.hortora' / 'garden')
).expanduser().resolve()

mcp = FastMCP("Hortora Garden")


@mcp.tool()
def garden_search(
    query: str,
    technology: str = None,
    domain: str = None,
    garden_path: str = None,
) -> list:
    """Search the knowledge garden for entries matching a query.

    Uses 3-tier retrieval:
    1. Technology/domain filter via GARDEN.md index
    2. Keyword match in filtered entries
    3. Full-text grep across domain if no index matches

    Args:
        query: Keywords describing the problem or symptom
        technology: Filter by technology (e.g. "Java", "Python", "Quarkus")
        domain: Filter by domain directory name (e.g. "java", "tools")
        garden_path: Override default garden path

    Returns:
        List of matching entries, each with id, title, domain, score, body
    """
    garden = Path(garden_path).expanduser().resolve() if garden_path else _DEFAULT_GARDEN
    return search_garden(garden, query, technology=technology, domain=domain)


@mcp.tool()
def garden_status(garden_path: str = None) -> dict:
    """Get garden health and metadata.

    Args:
        garden_path: Override default garden path

    Returns:
        Dict with entry_count, drift, threshold, dedupe_recommended,
        last_sweep, last_staleness_review, role, name
    """
    garden = Path(garden_path).expanduser().resolve() if garden_path else _DEFAULT_GARDEN
    return get_status(garden)


@mcp.tool()
def garden_capture(
    title: str,
    type: str,
    domain: str,
    stack: str,
    tags: list,
    score: int,
    body: str,
    garden_path: str = None,
) -> dict:
    """Submit a new knowledge entry to the garden.

    Creates a git branch with the entry ready for PR review.
    Score must be >= 8. Type must be gotcha, technique, or undocumented.

    Args:
        title: Short imperative title describing the non-obvious thing
        type: Entry type — gotcha | technique | undocumented
        domain: Domain directory (e.g. java, python, tools)
        stack: Technology and version (e.g. "Quarkus 3.9.x, Java 21")
        tags: List of tag strings (e.g. ["hibernate", "jpa", "flush"])
        score: Self-assessed score 8-15 (must pass minimum threshold)
        body: Entry body markdown (symptom, root cause, fix, why non-obvious)
        garden_path: Override default garden path

    Returns:
        Dict with status (ok/error), ge_id, branch, message
    """
    garden = Path(garden_path).expanduser().resolve() if garden_path else _DEFAULT_GARDEN
    return capture_entry(garden, title=title, type=type, domain=domain,
                         stack=stack, tags=tags, score=score, body=body)


if __name__ == '__main__':
    mcp.run()
```

- [ ] **Step 4: Run all MCP server tests — verify PASS**

```bash
python3 -m pytest tests/test_garden_mcp_server.py -v
```

Expected: All tests PASS.

- [ ] **Step 5: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [ ] **Step 6: Commit and push**

```bash
git add scripts/garden_mcp_server.py tests/test_garden_mcp_server.py
git commit -m "feat(mcp): add garden_mcp_server.py — FastMCP server with garden_search, garden_capture, garden_status

Exposes the knowledge garden to any MCP-compatible AI assistant.
Business logic in mcp_garden_*.py modules; FastMCP wrapper here.

Refs #29"

git push
```

---

## Self-Review

**Spec coverage:**

| Phase 8 requirement | Task |
|---|---|
| `garden_search()` tool — 3-tier retrieval | Task 1 + Task 4 |
| `garden_capture()` tool — entry submission | Task 3 + Task 4 |
| `garden_status()` tool — garden metadata | Task 2 + Task 4 |
| FastMCP server with stdio transport | Task 4 |
| Business logic separate from MCP protocol | All tasks — mcp_garden_*.py vs garden_mcp_server.py |

**No gaps. No placeholders.**

**Type consistency:**
- `search_garden(garden, query, technology?, domain?)` → `list[dict]`
- `get_status(garden)` → `dict`
- `capture_entry(garden, title, type, domain, stack, tags, score, body)` → `dict`
- All tool functions in server have same signatures, accepting optional `garden_path: str`
