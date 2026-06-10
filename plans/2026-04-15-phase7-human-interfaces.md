# Phase 7 — Human Interfaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Humans can browse, search, and contribute to the knowledge garden without touching the CLI. Deliver: a Python data builder (`garden_web_data.py`), a garden browser web page (`/docs/garden.html`) with Fuse.js search, and an Obsidian integration guide (`/docs/obsidian.html`).

**Architecture:** Three deliverables. `garden_web_data.py` (soredium/scripts) reads the local garden and outputs JSON — this is the TDD-testable backend. The garden browser page is vanilla JS + Fuse.js CDN hosted on the Jekyll site; it fetches garden data from the GitHub raw content API so no build step is needed. The Obsidian guide documents native compatibility — the garden format is already Obsidian-native (YAML frontmatter → tags, `See also` → wikilinks). No separate TypeScript plugin needed for initial implementation.

**Tech Stack:** Python 3.11+, pytest (garden_web_data.py); HTML/CSS/vanilla JS/Fuse.js CDN (garden browser); Jekyll (hosting).

---

## File Map

| File | Repo | Status | Responsibility |
|------|------|--------|----------------|
| `scripts/garden_web_data.py` | soredium | Create | Parse garden structure → JSON |
| `tests/test_garden_web_data.py` | soredium | Create | Unit + integration tests |
| `docs/garden.html` | hortora.github.io | Create | Garden browser web page |
| `docs/obsidian.html` | hortora.github.io | Create | Obsidian integration guide |

---

## Task 1: garden_web_data.py — data builder (TDD)

**Files:**
- Create: `soredium/scripts/garden_web_data.py`
- Create: `soredium/tests/test_garden_web_data.py`

The data builder reads a local garden clone and produces a JSON structure:

```json
{
  "domains": [
    {
      "name": "java",
      "entry_count": 42,
      "entries": [
        {
          "id": "GE-20260414-aa0001",
          "title": "Hibernate @PreUpdate fires at flush time",
          "type": "gotcha",
          "domain": "java",
          "stack": "Hibernate ORM 6.x",
          "tags": ["hibernate", "jpa", "flush"],
          "score": 12,
          "submitted": "2026-04-14",
          "staleness_threshold": 730
        }
      ]
    }
  ],
  "total_entries": 84,
  "generated": "2026-04-15T12:00:00"
}
```

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_garden_web_data.py`:

```python
#!/usr/bin/env python3
"""Unit and integration tests for garden_web_data.py."""

import json
import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from garden_web_data import (
    parse_entry_frontmatter, parse_domain_index,
    get_domain_entries, build_garden_data,
)

CLI = Path(__file__).parent.parent / 'scripts' / 'garden_web_data.py'


def run_cli(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(CLI)] + list(args),
        capture_output=True, text=True
    )


ENTRY_CONTENT = """\
---
id: GE-20260414-aa0001
title: "Hibernate @PreUpdate fires at flush time not at persist"
type: gotcha
domain: java
stack: "Hibernate ORM 6.x"
tags: [hibernate, jpa, lifecycle, flush]
score: 12
verified: true
staleness_threshold: 730
submitted: 2026-04-14
---

## Hibernate @PreUpdate fires at flush time not at persist

**ID:** GE-20260414-aa0001
**Stack:** Hibernate ORM 6.x
**Symptom:** Callback not firing.

### Root cause
Fires at flush.

### Fix
Force flush.
"""

DOMAIN_INDEX = """\
# Java Index

| GE-ID | Title | Type | Score | Submitted |
|-------|-------|------|-------|-----------|
| GE-20260414-aa0001 | Hibernate @PreUpdate fires at flush time | gotcha | 12 | 2026-04-14 |
| GE-20260414-aa0002 | H2 rejects key as a column name | gotcha | 10 | 2026-04-14 |
"""


def make_git_garden(tmp: Path) -> Path:
    """Create a committed garden with java and tools domains."""
    garden = tmp / 'garden'
    garden.mkdir()
    for cmd in [
        ['git', 'init', str(garden)],
        ['git', '-C', str(garden), 'config', 'user.email', 'test@test.com'],
        ['git', '-C', str(garden), 'config', 'user.name', 'Test'],
    ]:
        subprocess.run(cmd, check=True, capture_output=True)

    (garden / 'GARDEN.md').write_text(
        '**Last assigned ID:** GE-0002\n'
        '**Last full DEDUPE sweep:** 2026-04-15\n'
        '**Entries merged since last sweep:** 0\n'
        '**Drift threshold:** 10\n\n'
        '## By Technology\n\n'
        '### Java\n'
        '| GE-ID | Title | Type | Score |\n'
        '|-------|-------|------|-------|\n'
        '| [GE-20260414-aa0001](java/GE-20260414-aa0001.md) | Hibernate test | gotcha | 12 |\n'
        '| [GE-20260414-aa0002](java/GE-20260414-aa0002.md) | H2 test | gotcha | 10 |\n\n'
        '### Tools\n'
        '| GE-ID | Title | Type | Score |\n'
        '|-------|-------|------|-------|\n'
        '| [GE-20260414-bb0001](tools/GE-20260414-bb0001.md) | sed test | gotcha | 11 |\n\n'
        '## By Symptom / Type\n\n## By Label\n\n'
    )
    (garden / 'java').mkdir()
    (garden / 'java' / 'INDEX.md').write_text(DOMAIN_INDEX)
    for gid, title, score in [
        ('aa0001', 'Hibernate @PreUpdate fires at flush time', 12),
        ('aa0002', 'H2 rejects key as a column name', 10),
    ]:
        (garden / 'java' / f'GE-20260414-{gid}.md').write_text(
            f'---\nid: GE-20260414-{gid}\ntitle: "{title}"\ntype: gotcha\n'
            f'domain: java\nstack: "Hibernate ORM"\ntags: [java, hibernate]\n'
            f'score: {score}\nverified: true\nstaleness_threshold: 730\n'
            f'submitted: 2026-04-14\n---\n\n## {title}\n**ID:** GE-20260414-{gid}\n'
        )
    (garden / 'tools').mkdir()
    (garden / 'tools' / 'GE-20260414-bb0001.md').write_text(
        '---\nid: GE-20260414-bb0001\ntitle: "macOS BSD sed ignores word boundaries"\n'
        'type: gotcha\ndomain: tools\nstack: "macOS, BSD sed"\ntags: [sed, macos, regex]\n'
        'score: 11\nverified: true\nstaleness_threshold: 730\nsubmitted: 2026-04-14\n---\n\n'
        '## macOS BSD sed ignores word boundaries\n**ID:** GE-20260414-bb0001\n'
    )
    subprocess.run(['git', '-C', str(garden), 'add', '.'], check=True, capture_output=True)
    subprocess.run(['git', '-C', str(garden), 'commit', '-m', 'init'],
                   check=True, capture_output=True)
    return garden


# ── parse_entry_frontmatter ───────────────────────────────────────────────────

class TestParseEntryFrontmatter(unittest.TestCase):

    def test_parses_id(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['id'], 'GE-20260414-aa0001')

    def test_parses_title(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertIn('Hibernate', result['title'])

    def test_parses_type(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['type'], 'gotcha')

    def test_parses_domain(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['domain'], 'java')

    def test_parses_tags_as_list(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertIsInstance(result['tags'], list)
        self.assertIn('hibernate', result['tags'])

    def test_parses_score_as_int(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['score'], 12)

    def test_parses_staleness_threshold_as_int(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['staleness_threshold'], 730)

    def test_parses_submitted(self):
        result = parse_entry_frontmatter(ENTRY_CONTENT)
        self.assertEqual(result['submitted'], '2026-04-14')

    def test_empty_content_returns_empty_dict(self):
        self.assertEqual(parse_entry_frontmatter(''), {})

    def test_no_frontmatter_returns_empty_dict(self):
        self.assertEqual(parse_entry_frontmatter('## Just a heading\n\nNo YAML\n'), {})

    def test_crlf_content_parsed(self):
        crlf = ENTRY_CONTENT.replace('\n', '\r\n')
        result = parse_entry_frontmatter(crlf)
        self.assertEqual(result['id'], 'GE-20260414-aa0001')


# ── parse_domain_index ────────────────────────────────────────────────────────

class TestParseDomainIndex(unittest.TestCase):

    def test_returns_list(self):
        result = parse_domain_index(DOMAIN_INDEX)
        self.assertIsInstance(result, list)

    def test_finds_both_entries(self):
        result = parse_domain_index(DOMAIN_INDEX)
        self.assertEqual(len(result), 2)

    def test_entry_has_ge_id(self):
        result = parse_domain_index(DOMAIN_INDEX)
        ids = [e['id'] for e in result]
        self.assertIn('GE-20260414-aa0001', ids)
        self.assertIn('GE-20260414-aa0002', ids)

    def test_entry_has_title(self):
        result = parse_domain_index(DOMAIN_INDEX)
        titles = [e['title'] for e in result]
        self.assertTrue(any('Hibernate' in t for t in titles))

    def test_empty_index_returns_empty_list(self):
        self.assertEqual(parse_domain_index('# Java Index\n\nNo table.\n'), [])

    def test_header_rows_not_included(self):
        result = parse_domain_index(DOMAIN_INDEX)
        ids = [e['id'] for e in result]
        self.assertNotIn('GE-ID', ids)  # header row must not appear


# ── get_domain_entries ────────────────────────────────────────────────────────

class TestGetDomainEntries(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_returns_entries_for_java_domain(self):
        entries = get_domain_entries(self.garden, 'java')
        self.assertEqual(len(entries), 2)

    def test_returns_entries_for_tools_domain(self):
        entries = get_domain_entries(self.garden, 'tools')
        self.assertEqual(len(entries), 1)

    def test_entries_have_id(self):
        entries = get_domain_entries(self.garden, 'java')
        for e in entries:
            self.assertIn('id', e)
            self.assertRegex(e['id'], r'^GE-\d{8}-[0-9a-f]{6}$')

    def test_entries_have_title(self):
        entries = get_domain_entries(self.garden, 'java')
        for e in entries:
            self.assertIn('title', e)
            self.assertTrue(len(e['title']) > 0)

    def test_entries_have_required_fields(self):
        entries = get_domain_entries(self.garden, 'java')
        for e in entries:
            for field in ('id', 'title', 'type', 'domain', 'score', 'tags'):
                self.assertIn(field, e, f"Missing field: {field}")

    def test_nonexistent_domain_returns_empty(self):
        entries = get_domain_entries(self.garden, 'nonexistent')
        self.assertEqual(entries, [])


# ── build_garden_data ─────────────────────────────────────────────────────────

class TestBuildGardenData(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_returns_dict(self):
        data = build_garden_data(self.garden)
        self.assertIsInstance(data, dict)

    def test_has_domains_list(self):
        data = build_garden_data(self.garden)
        self.assertIn('domains', data)
        self.assertIsInstance(data['domains'], list)

    def test_finds_both_domains(self):
        data = build_garden_data(self.garden)
        names = [d['name'] for d in data['domains']]
        self.assertIn('java', names)
        self.assertIn('tools', names)

    def test_domain_has_entry_count(self):
        data = build_garden_data(self.garden)
        java = next(d for d in data['domains'] if d['name'] == 'java')
        self.assertEqual(java['entry_count'], 2)

    def test_domain_has_entries_list(self):
        data = build_garden_data(self.garden)
        java = next(d for d in data['domains'] if d['name'] == 'java')
        self.assertIn('entries', java)
        self.assertEqual(len(java['entries']), 2)

    def test_has_total_entries(self):
        data = build_garden_data(self.garden)
        self.assertIn('total_entries', data)
        self.assertEqual(data['total_entries'], 3)

    def test_has_generated_timestamp(self):
        data = build_garden_data(self.garden)
        self.assertIn('generated', data)
        self.assertIsInstance(data['generated'], str)

    def test_output_is_json_serializable(self):
        data = build_garden_data(self.garden)
        serialized = json.dumps(data)
        self.assertIsInstance(serialized, str)

    def test_nonexistent_garden_raises(self):
        with self.assertRaises(Exception):
            build_garden_data(Path('/nonexistent/path'))


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestGardenWebDataCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = make_git_garden(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_outputs_valid_json(self):
        result = run_cli(str(self.garden))
        self.assertEqual(result.returncode, 0, result.stderr)
        data = json.loads(result.stdout)
        self.assertIn('domains', data)

    def test_json_contains_both_domains(self):
        result = run_cli(str(self.garden))
        data = json.loads(result.stdout)
        names = [d['name'] for d in data['domains']]
        self.assertIn('java', names)
        self.assertIn('tools', names)

    def test_json_contains_correct_total(self):
        result = run_cli(str(self.garden))
        data = json.loads(result.stdout)
        self.assertEqual(data['total_entries'], 3)

    def test_invalid_path_exits_1(self):
        result = run_cli('/nonexistent/garden')
        self.assertEqual(result.returncode, 1)

    def test_no_args_uses_default_or_exits(self):
        # With no args, either uses HORTORA_GARDEN env or exits with error
        result = run_cli()
        # Either exit 0 (found default garden) or exit 1 (no default) — not crash
        self.assertIn(result.returncode, [0, 1])


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_web_data.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'garden_web_data'`

- [ ] **Step 3: Create garden_web_data.py**

Create `soredium/scripts/garden_web_data.py`:

```python
#!/usr/bin/env python3
"""garden_web_data.py — Build JSON data from a knowledge garden for the web app.

Reads a local garden clone and outputs a JSON structure containing all domains,
entries with metadata, and a search index suitable for Fuse.js.

Usage:
  garden_web_data.py [garden_path]  # defaults to $HORTORA_GARDEN or ~/.hortora/garden

Output: JSON to stdout
"""

import json
import os
import re
import sys
from datetime import datetime
from pathlib import Path

FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---', re.DOTALL)
GE_ID_RE = re.compile(r'\bGE-\d{8}-[0-9a-f]{6}\b|\bGE-\d{4}\b')
SKIP_FILES = {'INDEX.md', 'README.md', 'GARDEN.md', 'CHECKED.md',
              'DISCARDED.md', 'SCHEMA.md'}
SKIP_DIRS = {'.git', 'submissions', '_augment', '_summaries',
             '_index', 'labels', 'scripts'}


def parse_entry_frontmatter(content: str) -> dict:
    """Extract YAML frontmatter fields from an entry file."""
    if not content:
        return {}
    content = content.replace('\r\n', '\n')
    m = FRONTMATTER_RE.match(content)
    if not m:
        return {}
    result = {}
    lines = m.group(1).splitlines()
    i = 0
    while i < len(lines):
        line = lines[i]
        if not line.strip() or ':' not in line:
            i += 1
            continue
        key, _, rest = line.partition(':')
        key = key.strip()
        rest = rest.strip()
        if not rest:
            i += 1
            continue
        if rest.startswith('[') and rest.endswith(']'):
            vals = [v.strip().strip('"\'') for v in rest[1:-1].split(',') if v.strip()]
            result[key] = vals
        else:
            val = rest.strip('"\'')
            # Try int conversion for numeric fields
            if key in ('score', 'staleness_threshold') and val.isdigit():
                result[key] = int(val)
            else:
                result[key] = val
        i += 1
    return result


def parse_domain_index(content: str) -> list:
    """Parse a domain INDEX.md table. Returns list of {id, title, ...} dicts."""
    entries = []
    for line in content.splitlines():
        if not line.startswith('|'):
            continue
        cells = [c.strip() for c in line.split('|') if c.strip()]
        if not cells:
            continue
        # Skip separator row
        if all(c.startswith('-') for c in cells):
            continue
        # Find cell containing a GE-ID
        ge_id = None
        title = None
        for cell in cells:
            ids = GE_ID_RE.findall(cell)
            if ids:
                ge_id = ids[0]
            elif ge_id and not title and cell and not cell.startswith('GE-'):
                title = cell
        if ge_id and ge_id != 'GE-ID':  # skip header row
            entries.append({'id': ge_id, 'title': title or ''})
    return entries


def get_domain_entries(garden: Path, domain: str) -> list:
    """Read all YAML-frontmatter entries from a domain directory."""
    domain_dir = garden / domain
    if not domain_dir.is_dir():
        return []
    entries = []
    for path in sorted(domain_dir.glob('*.md')):
        if path.name in SKIP_FILES:
            continue
        if not GE_ID_RE.search(path.stem):
            continue
        try:
            content = path.read_text(encoding='utf-8')
        except OSError:
            continue
        fm = parse_entry_frontmatter(content)
        if not fm.get('id'):
            continue
        entries.append({
            'id': fm.get('id', path.stem),
            'title': fm.get('title', ''),
            'type': fm.get('type', ''),
            'domain': fm.get('domain', domain),
            'stack': fm.get('stack', ''),
            'tags': fm.get('tags', []),
            'score': fm.get('score', 0),
            'submitted': fm.get('submitted', ''),
            'staleness_threshold': fm.get('staleness_threshold', 730),
            'verified_on': fm.get('verified_on', ''),
        })
    return entries


def build_garden_data(garden: Path) -> dict:
    """Build complete garden data dict for JSON output."""
    garden = Path(garden).expanduser().resolve()
    if not garden.exists():
        raise FileNotFoundError(f"Garden not found: {garden}")

    domains = []
    total = 0

    # Discover domains from non-reserved directories
    for d in sorted(garden.iterdir()):
        if not d.is_dir() or d.name in SKIP_DIRS or d.name.startswith('.'):
            continue
        entries = get_domain_entries(garden, d.name)
        if entries:
            domains.append({
                'name': d.name,
                'entry_count': len(entries),
                'entries': entries,
            })
            total += len(entries)

    return {
        'domains': domains,
        'total_entries': total,
        'generated': datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ'),
    }


def main():
    default = Path(
        os.environ.get('HORTORA_GARDEN', str(Path.home() / '.hortora' / 'garden'))
    ).expanduser().resolve()

    garden = Path(sys.argv[1]).expanduser().resolve() if len(sys.argv) > 1 else default

    try:
        data = build_garden_data(garden)
    except FileNotFoundError as e:
        print(f"ERROR: {e}", file=sys.stderr)
        sys.exit(1)

    print(json.dumps(data, indent=2))
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_web_data.py -v
```

Expected: All tests PASS.

- [ ] **Step 5: Run full suite — no regressions**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden_web_data.py tests/test_garden_web_data.py
git commit -m "feat(web): add garden_web_data.py — JSON data builder for garden browser

Refs #29"
```

---

## Task 2: Garden browser web page

**Files:**
- Create: `hortora.github.io/docs/garden.html`
- Update: `hortora.github.io/docs/*.html` (add Garden link to sidebar nav)

The garden browser fetches data directly from the Hortora/garden GitHub repo using the GitHub raw content API. No build step required — pure static Jekyll page.

Features:
- Left sidebar: domain listing with entry counts
- Main area: entries for selected domain (card view)
- Search bar: Fuse.js client-side search across all entries
- Entry detail panel: full metadata on click

- [ ] **Step 1: Create `docs/garden.html`**

Create `/Users/mdproctor/claude/hortora/hortora.github.io/docs/garden.html`.

**Design requirements:**
- Match existing site: parchment/forest/sage palette, Lora serif font, same layout shell as other docs pages
- Left sidebar: domain cards showing name and entry count; clicking filters the main panel
- Main area: grid of entry cards; each card shows title, type badge (gotcha/technique/undocumented), domain, score, tags
- Search bar above the grid: Fuse.js search across all entries (title + tags + stack)
- Entry detail: sliding panel from right on card click, showing full metadata, entry body (fetched on demand), staleness indicator
- Loading state while fetching from GitHub API
- Graceful fallback: if GitHub API fails, show a friendly error message

**Garden data source** (public garden repo on GitHub):
```javascript
const GARDEN_RAW_BASE = 'https://raw.githubusercontent.com/Hortora/garden/main';
const GARDEN_API_BASE = 'https://api.github.com/repos/Hortora/garden/contents';
```

**Fetch strategy:**
1. `GET ${GARDEN_RAW_BASE}/GARDEN.md` — parse By Technology section to get domain list
2. For each domain: `GET ${GARDEN_RAW_BASE}/${domain}/INDEX.md` — parse entry table
3. On entry click: `GET ${GARDEN_RAW_BASE}/${domain}/${ge_id}.md` — show full entry

**Staleness indicator logic (in JS):**
```javascript
function stalenessLabel(submittedStr, thresholdDays) {
    const submitted = new Date(submittedStr);
    const ageDays = Math.floor((Date.now() - submitted) / 86400000);
    if (ageDays > thresholdDays) return { level: 'stale', days: ageDays };
    if (ageDays > thresholdDays * 0.75) return { level: 'approaching', days: ageDays };
    return { level: 'fresh', days: ageDays };
}
```

The page should include:
```html
<script src="https://cdn.jsdelivr.net/npm/fuse.js@7/dist/fuse.min.js"></script>
```

Full implementation: write a complete, production-quality `garden.html` that matches the existing site's visual language. The page must:
- Use `layout: default` Jekyll frontmatter
- Include the standard sidebar navigation (with Garden as active)
- Load data from GitHub, parse it, build Fuse.js index
- Render domain tabs, entry cards, search results, and entry detail panel
- Handle loading, error, and empty states gracefully

- [ ] **Step 2: Update sidebar nav in all docs pages**

Add `<a href="garden.html">Garden Browser</a>` to the sidebar nav in these files, in the correct position (after Skills Reference, before Design Spec):
- `docs/getting-started.html`
- `docs/how-it-works.html`
- `docs/skills-reference.html`
- `docs/design-spec.html`
- `docs/architecture.html`

- [ ] **Step 3: Commit to hortora.github.io**

```bash
cd ~/claude/hortora/hortora.github.io
git add docs/garden.html docs/getting-started.html docs/how-it-works.html \
         docs/skills-reference.html docs/design-spec.html docs/architecture.html
git commit -m "feat(garden-browser): add garden browser web page with Fuse.js search

Fetches entries from Hortora/garden via GitHub raw content API.
Domain listing, entry cards, client-side search, entry detail panel.
Staleness indicator based on staleness_threshold frontmatter field."
```

---

## Task 3: Obsidian integration guide

**Files:**
- Create: `hortora.github.io/docs/obsidian.html`
- Update: sidebar nav in all docs pages (add Obsidian)

The garden format is already Obsidian-native:
- YAML frontmatter → Obsidian reads natively
- `tags: [hibernate, jpa, flush]` → Obsidian tags
- `See also: GE-XXXX` in body → becomes a wikilink manually or via template

The guide covers:
1. **Setup**: Obsidian Git plugin + sparse checkout command
2. **Native features**: YAML frontmatter properties panel, tag search
3. **Dataview queries**: for staleness view, domain listing, score filtering
4. **Templates**: entry template for capturing from Obsidian
5. **Limitations**: writes from Obsidian bypass validate_pr.py — use PR workflow instead

- [ ] **Step 1: Create `docs/obsidian.html`**

Create a complete guide page matching the site's design. The page should include:

**Section 1 — Setup:**
```bash
# Clone the garden sparsely (only index files, fetch bodies on demand)
git clone --filter=blob:none --no-checkout https://github.com/Hortora/garden ~/.hortora/garden
cd ~/.hortora/garden
git sparse-checkout init
git sparse-checkout set GARDEN.md SCHEMA.md "*/INDEX.md" "*/GE-*.md"
git checkout main
```
Then open `~/.hortora/garden` as an Obsidian vault.

**Section 2 — Native compatibility:**
- `tags:` field → Obsidian's native tag panel
- `score:` → sortable property
- `staleness_threshold:` → queryable with Dataview
- `submitted:` → queryable date field
- `verified_on:` → version context

**Section 3 — Dataview queries:**

Staleness view (entries past threshold):
```dataview
TABLE title, score, staleness_threshold, submitted
FROM ""
WHERE contains(file.frontmatter.id, "GE-")
WHERE (date(today) - date(submitted)).days > staleness_threshold
SORT submitted ASC
```

Domain listing:
```dataview
TABLE length(rows) AS "Entries"
FROM ""
WHERE contains(file.frontmatter.id, "GE-")
GROUP BY domain
```

High-value entries (score ≥ 12):
```dataview
TABLE title, score, submitted
FROM ""
WHERE contains(file.frontmatter.id, "GE-") AND score >= 12
SORT score DESC
```

**Section 4 — Limitations:**
- Writes from Obsidian bypass CI — always submit via PR
- `See also:` cross-references are plain text, not wikilinks (future plugin work)
- Full-text search: use Obsidian's built-in search or the web app

- [ ] **Step 2: Add Obsidian to sidebar nav in all docs pages**

Add after the Garden Browser link.

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/hortora.github.io
git add docs/obsidian.html docs/getting-started.html docs/how-it-works.html \
         docs/skills-reference.html docs/design-spec.html docs/architecture.html \
         docs/garden.html
git commit -m "feat(obsidian): add Obsidian integration guide + update sidebar nav"
git push
```

---

## Self-Review

**Spec coverage:**

| Phase 7 requirement | Task |
|---|---|
| Web app: domain view | Task 2 (domain sidebar + entry cards) |
| Web app: search | Task 2 (Fuse.js client-side search) |
| Web app: entry detail | Task 2 (detail panel on click) |
| Obsidian: native YAML → tags | Task 3 |
| Obsidian: staleness view | Task 3 (Dataview queries) |
| garden_web_data.py data builder | Task 1 |

**Out of scope (Phase 7b or 8):** Graph view (`_graph/edges.md`), Export (PDF/slides), TypeScript Obsidian plugin (Dataview covers the initial use case).

**No gaps. No placeholders.**
