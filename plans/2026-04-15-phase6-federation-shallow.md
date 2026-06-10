# Phase 6 — Federation (Shallow) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the federation layer — client config, submission routing, upstream dedup enforcement, and child-garden augmentation — so child gardens interoperate with parents without modifying them.

**Architecture:** Four scripts. `garden_config.py` reads `~/.claude/garden-config.toml` (stdlib `tomllib`) and resolves domain→garden mappings by reading each garden's SCHEMA.md. `route_submission.py` wraps config lookup into a CLI. `validate_pr.py` gains `--upstream-garden` flag for cross-chain Jaccard dedup. `augment_entry.py` manages `_augment/` files in child gardens; `init_garden.py` extended to create `_augment/` for child roles. All stdlib only — no new dependencies.

**Tech Stack:** Python 3.11+ (`tomllib`, `pathlib`, `re`, `argparse`, `subprocess`). pytest.

---

## File Map

| File | Status | Responsibility |
|------|--------|----------------|
| `scripts/garden_config.py` | Create | Read `garden-config.toml`, resolve domain→garden, walk upstream chain |
| `scripts/route_submission.py` | Create | CLI: given domain, print which local garden owns it |
| `scripts/validate_pr.py` | Modify | Add `--upstream-garden` flag; check upstream gardens for duplicates |
| `scripts/init_garden.py` | Modify | Create `_augment/` dir for child gardens |
| `scripts/augment_entry.py` | Create | Create/list/validate `_augment/` files in child gardens |
| `tests/test_garden_config.py` | Create | Unit + integration + CLI tests for garden_config.py |
| `tests/test_route_submission.py` | Create | Unit + integration + CLI tests for route_submission.py |
| `tests/test_validate_pr.py` | Modify | Add upstream dedup tests |
| `tests/test_init_garden.py` | Modify | Add _augment/ tests for child gardens |
| `tests/test_augment_entry.py` | Create | Unit + integration + CLI tests for augment_entry.py |

---

## Config Format

`~/.claude/garden-config.toml`:
```toml
# Maps garden names to local clone paths.
# SCHEMA.md in each garden declares its role, domains, and upstream.

[[gardens]]
name = "jvm-garden"
path = "~/.hortora/jvm-garden"

[[gardens]]
name = "tools-garden"
path = "~/.hortora/tools-garden"

[[gardens]]
name = "my-private-garden"
path = "~/work/my-private-garden"
```

---

## Task 1: garden_config.py — client config reader (TDD)

**Files:**
- Create: `soredium/scripts/garden_config.py`
- Create: `soredium/tests/test_garden_config.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_garden_config.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for garden_config.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from garden_config import (
    load_config, validate_config, resolve_paths,
    find_garden_for_domain, get_upstream_chain,
)

CLI = Path(__file__).parent.parent / 'scripts' / 'garden_config.py'


def run_cli(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(CLI)] + list(args),
        capture_output=True, text=True
    )


def write_config(path: Path, content: str):
    path.write_text(content)


def write_schema(garden_dir: Path, role: str, ge_prefix: str,
                 domains: list, name: str = 'test', upstream: list = None):
    garden_dir.mkdir(parents=True, exist_ok=True)
    lines = [
        '---', f'name: {name}', 'description: "test"',
        f'role: {role}', f'ge_prefix: {ge_prefix}',
        'schema_version: "1.0"', f'domains: [{", ".join(domains)}]',
    ]
    if upstream:
        lines.append('upstream:')
        for u in upstream:
            lines.append(f'  - {u}')
    lines.extend(['---', ''])
    (garden_dir / 'SCHEMA.md').write_text('\n'.join(lines))


VALID_TOML = """\
[[gardens]]
name = "jvm-garden"
path = "/tmp/jvm-garden"

[[gardens]]
name = "tools-garden"
path = "/tmp/tools-garden"
"""

CHILD_TOML = """\
[[gardens]]
name = "jvm-garden"
path = "/tmp/jvm-garden"

[[gardens]]
name = "my-garden"
path = "/tmp/my-garden"
"""


# ── load_config ───────────────────────────────────────────────────────────────

class TestLoadConfig(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_loads_valid_toml(self):
        p = self.root / 'garden-config.toml'
        write_config(p, VALID_TOML)
        result = load_config(p)
        self.assertEqual(len(result), 2)
        self.assertEqual(result[0]['name'], 'jvm-garden')
        self.assertEqual(result[1]['name'], 'tools-garden')

    def test_missing_file_raises(self):
        with self.assertRaises(FileNotFoundError):
            load_config(self.root / 'nonexistent.toml')

    def test_invalid_toml_raises(self):
        p = self.root / 'bad.toml'
        p.write_text('[[gardens\nbroken')
        with self.assertRaises(Exception):
            load_config(p)

    def test_empty_gardens_list(self):
        p = self.root / 'empty.toml'
        p.write_text('')
        result = load_config(p)
        self.assertEqual(result, [])

    def test_single_garden(self):
        p = self.root / 'single.toml'
        p.write_text('[[gardens]]\nname = "jvm-garden"\npath = "/tmp/jvm"\n')
        result = load_config(p)
        self.assertEqual(len(result), 1)
        self.assertEqual(result[0]['name'], 'jvm-garden')


# ── validate_config ───────────────────────────────────────────────────────────

class TestValidateConfig(unittest.TestCase):

    def test_valid_config_no_errors(self):
        gardens = [
            {'name': 'jvm-garden', 'path': '/tmp/jvm'},
            {'name': 'tools-garden', 'path': '/tmp/tools'},
        ]
        errors, warnings = validate_config(gardens)
        self.assertEqual(errors, [])

    def test_missing_name_error(self):
        errors, _ = validate_config([{'path': '/tmp/jvm'}])
        self.assertEqual(len(errors), 1)
        self.assertIn('name', errors[0])

    def test_missing_path_error(self):
        errors, _ = validate_config([{'name': 'jvm-garden'}])
        self.assertEqual(len(errors), 1)
        self.assertIn('path', errors[0])

    def test_duplicate_names_error(self):
        gardens = [
            {'name': 'jvm-garden', 'path': '/tmp/a'},
            {'name': 'jvm-garden', 'path': '/tmp/b'},
        ]
        errors, _ = validate_config(gardens)
        self.assertEqual(len(errors), 1)
        self.assertIn('duplicate', errors[0].lower())

    def test_empty_list_valid(self):
        errors, warnings = validate_config([])
        self.assertEqual(errors, [])

    def test_both_missing_fields_both_reported(self):
        errors, _ = validate_config([{}])
        self.assertGreaterEqual(len(errors), 2)


# ── resolve_paths ─────────────────────────────────────────────────────────────

class TestResolvePaths(unittest.TestCase):

    def test_tilde_expanded(self):
        gardens = [{'name': 'jvm', 'path': '~/.hortora/jvm-garden'}]
        result = resolve_paths(gardens)
        self.assertNotIn('~', result[0]['path'])
        self.assertIn('.hortora', result[0]['path'])

    def test_absolute_path_unchanged(self):
        gardens = [{'name': 'jvm', 'path': '/absolute/path'}]
        result = resolve_paths(gardens)
        self.assertEqual(result[0]['path'], '/absolute/path')

    def test_does_not_modify_original(self):
        gardens = [{'name': 'jvm', 'path': '~/.hortora/jvm'}]
        resolve_paths(gardens)
        self.assertEqual(gardens[0]['path'], '~/.hortora/jvm')

    def test_multiple_gardens_all_resolved(self):
        gardens = [
            {'name': 'a', 'path': '~/a'},
            {'name': 'b', 'path': '~/b'},
        ]
        result = resolve_paths(gardens)
        for g in result:
            self.assertNotIn('~', g['path'])


# ── find_garden_for_domain ────────────────────────────────────────────────────

class TestFindGardenForDomain(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def _make_garden(self, name: str, domains: list, role: str = 'canonical') -> dict:
        gdir = self.root / name
        write_schema(gdir, role=role, ge_prefix='GE-', domains=domains, name=name)
        return {'name': name, 'path': str(gdir)}

    def test_finds_correct_garden(self):
        jvm = self._make_garden('jvm-garden', ['java', 'quarkus'])
        tools = self._make_garden('tools-garden', ['tools', 'cli'])
        result = find_garden_for_domain([jvm, tools], 'java')
        self.assertEqual(result['name'], 'jvm-garden')

    def test_returns_none_when_not_found(self):
        jvm = self._make_garden('jvm-garden', ['java'])
        result = find_garden_for_domain([jvm], 'python')
        self.assertIsNone(result)

    def test_empty_list_returns_none(self):
        result = find_garden_for_domain([], 'java')
        self.assertIsNone(result)

    def test_domain_in_multiple_gardens_returns_first_with_warning(self):
        a = self._make_garden('garden-a', ['java'])
        b = self._make_garden('garden-b', ['java'])
        result, warnings = find_garden_for_domain([a, b], 'java', return_warnings=True)
        self.assertEqual(result['name'], 'garden-a')
        self.assertEqual(len(warnings), 1)

    def test_garden_without_schema_skipped(self):
        no_schema = {'name': 'no-schema', 'path': str(self.root / 'empty')}
        (self.root / 'empty').mkdir()
        jvm = self._make_garden('jvm-garden', ['java'])
        result = find_garden_for_domain([no_schema, jvm], 'java')
        self.assertEqual(result['name'], 'jvm-garden')


# ── get_upstream_chain ────────────────────────────────────────────────────────

class TestGetUpstreamChain(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def _make_garden(self, name: str, role: str, domains: list,
                     upstream_names: list = None) -> tuple:
        gdir = self.root / name
        upstream_urls = []
        if upstream_names:
            upstream_urls = [f'https://github.com/Hortora/{n}' for n in upstream_names]
        write_schema(gdir, role=role, ge_prefix='GE-', domains=domains,
                     name=name, upstream=upstream_urls)
        garden = {'name': name, 'path': str(gdir)}
        return gdir, garden

    def test_canonical_returns_empty_chain(self):
        gdir, garden = self._make_garden('jvm-garden', 'canonical', ['java'])
        gardens = [garden]
        chain = get_upstream_chain(gardens, Path(gdir))
        self.assertEqual(chain, [])

    def test_child_returns_parent(self):
        jvm_dir, jvm = self._make_garden('jvm-garden', 'canonical', ['java'])
        my_dir, my = self._make_garden('my-garden', 'child', ['java'],
                                       upstream_names=['jvm-garden'])
        gardens = [jvm, my]
        chain = get_upstream_chain(gardens, Path(my_dir))
        self.assertEqual(len(chain), 1)
        self.assertEqual(str(chain[0]), str(jvm_dir))

    def test_three_level_chain(self):
        root_dir, root = self._make_garden('root', 'canonical', ['java'])
        mid_dir, mid = self._make_garden('mid', 'child', ['java'],
                                         upstream_names=['root'])
        leaf_dir, leaf = self._make_garden('leaf', 'child', ['java'],
                                           upstream_names=['mid'])
        gardens = [root, mid, leaf]
        chain = get_upstream_chain(gardens, Path(leaf_dir))
        self.assertEqual(len(chain), 2)
        self.assertEqual(str(chain[0]), str(mid_dir))
        self.assertEqual(str(chain[1]), str(root_dir))

    def test_unknown_upstream_returns_empty(self):
        my_dir, my = self._make_garden('my-garden', 'child', ['java'],
                                       upstream_names=['nonexistent'])
        chain = get_upstream_chain([my], Path(my_dir))
        self.assertEqual(chain, [])

    def test_peer_returns_empty_chain(self):
        gdir, garden = self._make_garden('peer-garden', 'peer', ['tools'])
        chain = get_upstream_chain([garden], Path(gdir))
        self.assertEqual(chain, [])


# ── Integration tests ─────────────────────────────────────────────────────────

class TestGardenConfigIntegration(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.jvm_dir = self.root / 'jvm-garden'
        self.my_dir = self.root / 'my-garden'
        write_schema(self.jvm_dir, role='canonical', ge_prefix='JE-',
                     domains=['java', 'quarkus'], name='jvm-garden')
        write_schema(self.my_dir, role='child', ge_prefix='ME-',
                     domains=['java'], name='my-garden',
                     upstream=['https://github.com/Hortora/jvm-garden'])
        self.config_path = self.root / 'garden-config.toml'
        self.config_path.write_text(
            f'[[gardens]]\nname = "jvm-garden"\npath = "{self.jvm_dir}"\n\n'
            f'[[gardens]]\nname = "my-garden"\npath = "{self.my_dir}"\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_domain_routing_via_schema(self):
        gardens = load_config(self.config_path)
        result = find_garden_for_domain(gardens, 'quarkus')
        self.assertEqual(result['name'], 'jvm-garden')

    def test_upstream_chain_resolution(self):
        gardens = load_config(self.config_path)
        chain = get_upstream_chain(gardens, self.my_dir)
        self.assertEqual(len(chain), 1)
        self.assertEqual(str(chain[0]), str(self.jvm_dir))

    def test_domain_not_in_any_garden(self):
        gardens = load_config(self.config_path)
        result = find_garden_for_domain(gardens, 'python')
        self.assertIsNone(result)


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestGardenConfigCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.jvm_dir = self.root / 'jvm-garden'
        write_schema(self.jvm_dir, role='canonical', ge_prefix='JE-',
                     domains=['java', 'quarkus'], name='jvm-garden')
        self.config_path = self.root / 'garden-config.toml'
        self.config_path.write_text(
            f'[[gardens]]\nname = "jvm-garden"\npath = "{self.jvm_dir}"\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_status_exits_0(self):
        result = run_cli('status', '--config', str(self.config_path))
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_status_shows_garden_names(self):
        result = run_cli('status', '--config', str(self.config_path))
        self.assertIn('jvm-garden', result.stdout)

    def test_missing_config_exits_1(self):
        result = run_cli('status', '--config', '/nonexistent/config.toml')
        self.assertEqual(result.returncode, 1)

    def test_status_shows_role_and_domains(self):
        result = run_cli('status', '--config', str(self.config_path))
        self.assertIn('canonical', result.stdout)
        self.assertIn('java', result.stdout)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_config.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'garden_config'`

- [ ] **Step 3: Create garden_config.py**

Create `soredium/scripts/garden_config.py`:

```python
#!/usr/bin/env python3
"""garden_config.py — Reads ~/.claude/garden-config.toml and resolves garden routing.

Usage:
  garden_config.py status [--config PATH]

Exit codes: 0 = ok, 1 = error
"""

import argparse
import re
import sys
import tomllib
from pathlib import Path

DEFAULT_CONFIG = Path.home() / '.claude' / 'garden-config.toml'
FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---', re.DOTALL)


# ── Config I/O ─────────────────────────────────────────────────────────────────

def load_config(path: Path) -> list:
    """Load garden-config.toml. Returns list of {name, path} dicts."""
    if not path.exists():
        raise FileNotFoundError(f"Config not found: {path}")
    with open(path, 'rb') as f:
        data = tomllib.load(f)
    return data.get('gardens', [])


def validate_config(gardens: list) -> tuple:
    """Validate garden list. Returns (errors, warnings)."""
    errors = []
    warnings = []
    seen_names = set()
    for i, g in enumerate(gardens):
        prefix = f"Garden {i+1}"
        if not g.get('name'):
            errors.append(f"{prefix}: 'name' is required")
        if not g.get('path'):
            errors.append(f"{prefix}: 'path' is required")
        name = g.get('name', '')
        if name and name in seen_names:
            errors.append(f"Duplicate garden name: {name!r}")
        if name:
            seen_names.add(name)
    return errors, warnings


def resolve_paths(gardens: list) -> list:
    """Return new list with ~ expanded in path fields."""
    return [
        {**g, 'path': str(Path(g['path']).expanduser())}
        for g in gardens
    ]


# ── SCHEMA.md reading ──────────────────────────────────────────────────────────

def _read_schema(garden_path) -> dict | None:
    """Read SCHEMA.md from a garden directory. Returns parsed frontmatter or None."""
    schema_path = Path(garden_path) / 'SCHEMA.md'
    if not schema_path.exists():
        return None
    content = schema_path.read_text(encoding='utf-8').replace('\r\n', '\n')
    m = FRONTMATTER_RE.match(content)
    if not m:
        return None
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
            items = []
            i += 1
            while i < len(lines) and re.match(r'^\s+-\s+', lines[i]):
                items.append(re.sub(r'^\s+-\s+', '', lines[i]).strip().strip('"\''))
                i += 1
            if items:
                result[key] = items
            continue
        elif rest.startswith('[') and rest.endswith(']'):
            result[key] = [v.strip().strip('"\'') for v in rest[1:-1].split(',') if v.strip()]
        else:
            result[key] = rest.strip('"\'')
        i += 1
    return result


# ── Routing ────────────────────────────────────────────────────────────────────

def find_garden_for_domain(gardens: list, domain: str,
                            return_warnings: bool = False):
    """Find the garden that owns the given domain.

    Returns garden dict (or None), and optionally a list of warnings.
    Reads SCHEMA.md from each configured garden to find domain membership.
    """
    matches = []
    for g in gardens:
        schema = _read_schema(g.get('path', ''))
        if schema is None:
            continue
        domains = schema.get('domains', [])
        if domain in domains:
            matches.append(g)

    warnings = []
    if len(matches) > 1:
        names = [m['name'] for m in matches]
        warnings.append(
            f"Domain {domain!r} claimed by multiple gardens: {', '.join(names)} — using first"
        )

    result = matches[0] if matches else None
    if return_warnings:
        return result, warnings
    return result


def get_upstream_chain(gardens: list, garden_path: Path) -> list:
    """Return ordered list of upstream garden paths [immediate_parent, ..., canonical].

    Reads target garden's SCHEMA.md to find upstream URL list, then maps each
    URL to a local path via the garden name (URL suffix match). Recursively
    walks the chain. Returns [] for canonical or peer gardens.
    """
    schema = _read_schema(garden_path)
    if schema is None or schema.get('role') != 'child':
        return []

    upstream_urls = schema.get('upstream', [])
    if not upstream_urls:
        return []

    name_to_path = {g['name']: Path(g['path']) for g in gardens}
    chain = []

    for url in upstream_urls:
        # Match by garden name: last path segment of URL (without .git)
        name = url.rstrip('/').split('/')[-1].removesuffix('.git')
        parent_path = name_to_path.get(name)
        if parent_path is None:
            continue
        chain.append(parent_path)
        # Recurse
        chain.extend(get_upstream_chain(gardens, parent_path))

    return chain


# ── CLI ────────────────────────────────────────────────────────────────────────

def main():
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('command', choices=['status'], help='Command to run')
    parser.add_argument('--config', type=Path, default=DEFAULT_CONFIG,
                        help=f'Path to garden-config.toml (default: {DEFAULT_CONFIG})')
    args = parser.parse_args()

    try:
        gardens = load_config(args.config)
    except FileNotFoundError as e:
        print(f"ERROR: {e}")
        sys.exit(1)

    errors, warnings = validate_config(gardens)
    if errors:
        for e in errors:
            print(f"ERROR: {e}")
        sys.exit(1)

    gardens = resolve_paths(gardens)

    if args.command == 'status':
        if not gardens:
            print("No gardens configured.")
            sys.exit(0)
        print(f"Configured gardens ({len(gardens)}):")
        for g in gardens:
            schema = _read_schema(g['path'])
            role = schema.get('role', 'unknown') if schema else 'no SCHEMA.md'
            domains = schema.get('domains', []) if schema else []
            domain_str = ', '.join(domains) if domains else '(none)'
            print(f"  {g['name']}")
            print(f"    path:    {g['path']}")
            print(f"    role:    {role}")
            print(f"    domains: {domain_str}")
        sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_config.py -v
```

Expected: All tests PASS.

**Note on `find_garden_for_domain` tests:** The `test_domain_in_multiple_gardens_returns_first_with_warning` test calls `find_garden_for_domain(..., return_warnings=True)`. The implementation supports this via the `return_warnings` parameter.

- [ ] **Step 5: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 371+ passing, 0 failures.

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden_config.py tests/test_garden_config.py
git commit -m "feat(federation): add garden_config.py — client config reader and domain routing

Refs #29"
```

---

## Task 2: route_submission.py — submission routing CLI (TDD)

**Files:**
- Create: `soredium/scripts/route_submission.py`
- Create: `soredium/tests/test_route_submission.py`

- [ ] **Step 1: Write failing tests**

Create `soredium/tests/test_route_submission.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for route_submission.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from route_submission import route

CLI = Path(__file__).parent.parent / 'scripts' / 'route_submission.py'


def run_cli(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(CLI)] + list(args),
        capture_output=True, text=True
    )


def write_schema(garden_dir: Path, role: str, ge_prefix: str,
                 domains: list, name: str = 'test'):
    garden_dir.mkdir(parents=True, exist_ok=True)
    lines = [
        '---', f'name: {name}', 'description: "test"',
        f'role: {role}', f'ge_prefix: {ge_prefix}',
        'schema_version: "1.0"', f'domains: [{", ".join(domains)}]',
        '---', '',
    ]
    (garden_dir / 'SCHEMA.md').write_text('\n'.join(lines))


# ── route() unit tests ────────────────────────────────────────────────────────

class TestRoute(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def _make_garden(self, name: str, domains: list) -> dict:
        gdir = self.root / name
        write_schema(gdir, role='canonical', ge_prefix='GE-',
                     domains=domains, name=name)
        return {'name': name, 'path': str(gdir)}

    def test_routes_to_correct_garden(self):
        jvm = self._make_garden('jvm-garden', ['java', 'quarkus'])
        tools = self._make_garden('tools-garden', ['tools', 'cli'])
        result = route('java', [jvm, tools])
        self.assertEqual(result, Path(jvm['path']))

    def test_returns_none_for_unknown_domain(self):
        jvm = self._make_garden('jvm-garden', ['java'])
        result = route('python', [jvm])
        self.assertIsNone(result)

    def test_empty_garden_list_returns_none(self):
        result = route('java', [])
        self.assertIsNone(result)

    def test_returns_path_object(self):
        jvm = self._make_garden('jvm-garden', ['java'])
        result = route('java', [jvm])
        self.assertIsInstance(result, Path)

    def test_garden_without_schema_skipped(self):
        empty = {'name': 'empty', 'path': str(self.root / 'empty')}
        (self.root / 'empty').mkdir()
        jvm = self._make_garden('jvm-garden', ['java'])
        result = route('java', [empty, jvm])
        self.assertEqual(result, Path(jvm['path']))


# ── Integration tests ─────────────────────────────────────────────────────────

class TestRouteIntegration(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.jvm_dir = self.root / 'jvm-garden'
        self.tools_dir = self.root / 'tools-garden'
        write_schema(self.jvm_dir, role='canonical', ge_prefix='JE-',
                     domains=['java', 'quarkus'], name='jvm-garden')
        write_schema(self.tools_dir, role='canonical', ge_prefix='TE-',
                     domains=['tools', 'cli', 'git'], name='tools-garden')
        self.config = self.root / 'garden-config.toml'
        self.config.write_text(
            f'[[gardens]]\nname = "jvm-garden"\npath = "{self.jvm_dir}"\n\n'
            f'[[gardens]]\nname = "tools-garden"\npath = "{self.tools_dir}"\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_java_routes_to_jvm_garden(self):
        result = run_cli('java', '--config', str(self.config))
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn(str(self.jvm_dir), result.stdout)

    def test_tools_routes_to_tools_garden(self):
        result = run_cli('tools', '--config', str(self.config))
        self.assertEqual(result.returncode, 0)
        self.assertIn(str(self.tools_dir), result.stdout)

    def test_unknown_domain_exits_1(self):
        result = run_cli('python', '--config', str(self.config))
        self.assertEqual(result.returncode, 1)
        self.assertIn('python', result.stdout + result.stderr)


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestRouteSubmissionCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.jvm_dir = self.root / 'jvm-garden'
        write_schema(self.jvm_dir, role='canonical', ge_prefix='JE-',
                     domains=['java', 'quarkus'], name='jvm-garden')
        self.config = self.root / 'garden-config.toml'
        self.config.write_text(
            f'[[gardens]]\nname = "jvm-garden"\npath = "{self.jvm_dir}"\n'
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_found_exits_0_and_prints_path(self):
        result = run_cli('java', '--config', str(self.config))
        self.assertEqual(result.returncode, 0)
        self.assertIn(str(self.jvm_dir), result.stdout)

    def test_not_found_exits_1_and_prints_error(self):
        result = run_cli('python', '--config', str(self.config))
        self.assertEqual(result.returncode, 1)
        self.assertIn('No garden', result.stdout + result.stderr)

    def test_missing_config_exits_1(self):
        result = run_cli('java', '--config', '/nonexistent.toml')
        self.assertEqual(result.returncode, 1)

    def test_missing_domain_arg_exits_nonzero(self):
        result = run_cli('--config', str(self.config))
        self.assertNotEqual(result.returncode, 0)

    def test_output_is_a_path(self):
        result = run_cli('java', '--config', str(self.config))
        self.assertEqual(result.returncode, 0)
        # Output should be a valid path string
        output_path = Path(result.stdout.strip())
        self.assertTrue(output_path.is_absolute() or result.stdout.strip().startswith('~'))


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_route_submission.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'route_submission'`

- [ ] **Step 3: Create route_submission.py**

Create `soredium/scripts/route_submission.py`:

```python
#!/usr/bin/env python3
"""route_submission.py — Find the correct local garden for a given domain.

Usage:
  route_submission.py <domain> [--config PATH]

Prints the local path of the garden that owns the domain.
Exit codes: 0 = found, 1 = not found or error
"""

import argparse
import sys
from pathlib import Path

# Import from same scripts/ directory
sys.path.insert(0, str(Path(__file__).parent))
from garden_config import load_config, resolve_paths, find_garden_for_domain

DEFAULT_CONFIG = Path.home() / '.claude' / 'garden-config.toml'


def route(domain: str, gardens: list) -> Path | None:
    """Return local Path of the garden owning this domain, or None."""
    result = find_garden_for_domain(gardens, domain)
    if result is None:
        return None
    return Path(result['path'])


def main():
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('domain', help='Domain to route (e.g. java, quarkus, tools)')
    parser.add_argument('--config', type=Path, default=DEFAULT_CONFIG)
    args = parser.parse_args()

    try:
        gardens = load_config(args.config)
    except FileNotFoundError as e:
        print(f"ERROR: {e}")
        sys.exit(1)

    gardens = resolve_paths(gardens)
    result = route(args.domain, gardens)

    if result is None:
        print(f"No garden found for domain: {args.domain!r}")
        sys.exit(1)

    print(str(result))
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 4: Run all tests — verify PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_route_submission.py -v
```

Expected: All tests PASS.

- [ ] **Step 5: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [ ] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/route_submission.py tests/test_route_submission.py
git commit -m "feat(federation): add route_submission.py — domain-to-garden routing CLI

Refs #29"
```

---

## Task 3: validate_pr.py — upstream chain dedup (TDD)

**Files:**
- Modify: `soredium/scripts/validate_pr.py`
- Modify: `soredium/tests/test_validate_pr.py`

When validating a PR against a child garden, also run L1 Jaccard dedup against all upstream gardens. Add `--upstream-garden PATH` flag (repeatable) to specify upstream gardens explicitly.

- [ ] **Step 1: Read validate_pr.py first**

Read `soredium/scripts/validate_pr.py` to find:
- How the existing Jaccard/L1 dedup function is named and called
- How `GARDEN_ROOT` / the target garden path is resolved
- Where to add the `--upstream-garden` argument
- The `main()` structure

- [ ] **Step 2: Write failing tests**

Read `soredium/tests/test_validate_pr.py` to understand the existing test fixture pattern (git repo setup, entry creation). Then add this class after all existing test classes:

```python
class TestUpstreamGardenDedup(unittest.TestCase):
    """validate_pr.py --upstream-garden rejects entries duplicating upstream content."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        # Target (child) garden — minimal git repo
        self.child = self.root / 'child-garden'
        self.child.mkdir()
        for cmd in [
            ['git', 'init', str(self.child)],
            ['git', '-C', str(self.child), 'config', 'user.email', 'test@test.com'],
            ['git', '-C', str(self.child), 'config', 'user.name', 'Test'],
        ]:
            subprocess.run(cmd, check=True, capture_output=True)
        (self.child / 'GARDEN.md').write_text(
            '**Last assigned ID:** GE-0000\n**Last full DEDUPE sweep:** 2026-04-15\n'
            '**Entries merged since last sweep:** 0\n**Drift threshold:** 10\n'
        )
        (self.child / 'java').mkdir()
        subprocess.run(['git', '-C', str(self.child), 'add', '.'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.child), 'commit', '-m', 'init'],
                       check=True, capture_output=True)
        # Upstream (parent) garden — has an existing entry
        self.parent = self.root / 'parent-garden'
        self.parent.mkdir()
        for cmd in [
            ['git', 'init', str(self.parent)],
            ['git', '-C', str(self.parent), 'config', 'user.email', 'test@test.com'],
            ['git', '-C', str(self.parent), 'config', 'user.name', 'Test'],
        ]:
            subprocess.run(cmd, check=True, capture_output=True)
        (self.parent / 'GARDEN.md').write_text(
            '**Last assigned ID:** GE-0000\n**Last full DEDUPE sweep:** 2026-04-15\n'
            '**Entries merged since last sweep:** 0\n**Drift threshold:** 10\n'
        )
        (self.parent / 'java').mkdir()
        # Add a real entry to the parent
        self.parent_entry = self.parent / 'java' / 'GE-20260414-parent1.md'
        self.parent_entry.write_text(UPSTREAM_ENTRY)
        subprocess.run(['git', '-C', str(self.parent), 'add', '.'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.parent), 'commit', '-m', 'add parent entry'],
                       check=True, capture_output=True)

    def tearDown(self):
        self.tmp.cleanup()

    def _submit_entry(self, content: str, entry_name: str = 'GE-20260414-test001.md'):
        """Add entry on a PR branch in child garden."""
        subprocess.run(
            ['git', '-C', str(self.child), 'checkout', '-b', f'submit/{entry_name[:-3]}'],
            check=True, capture_output=True
        )
        (self.child / 'java' / entry_name).write_text(content)
        subprocess.run(['git', '-C', str(self.child), 'add', '.'],
                       check=True, capture_output=True)
        subprocess.run(['git', '-C', str(self.child), 'commit', '-m', f'submit: {entry_name}'],
                       check=True, capture_output=True)

    def _run_validator(self, entry_path: str, *extra_args):
        VALIDATE_PR = Path(__file__).parent.parent / 'scripts' / 'validate_pr.py'
        return subprocess.run(
            [sys.executable, str(VALIDATE_PR), entry_path, str(self.child)]
            + list(extra_args),
            capture_output=True, text=True, cwd=str(self.child)
        )

    def test_distinct_entry_passes_without_upstream_flag(self):
        self._submit_entry(DISTINCT_ENTRY)
        entry = str(self.child / 'java' / 'GE-20260414-test001.md')
        result = self._run_validator(entry)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_distinct_entry_passes_with_upstream_flag(self):
        self._submit_entry(DISTINCT_ENTRY)
        entry = str(self.child / 'java' / 'GE-20260414-test001.md')
        result = self._run_validator(
            entry, '--upstream-garden', str(self.parent)
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_duplicate_of_upstream_entry_rejected(self):
        """Entry with identical title/tags to upstream entry should be rejected."""
        self._submit_entry(UPSTREAM_DUPLICATE_ENTRY)
        entry = str(self.child / 'java' / 'GE-20260414-test001.md')
        result = self._run_validator(
            entry, '--upstream-garden', str(self.parent)
        )
        self.assertEqual(result.returncode, 1, result.stdout + result.stderr)
        self.assertIn('upstream', result.stdout.lower() + result.stderr.lower())

    def test_multiple_upstream_gardens_all_checked(self):
        """Passing two --upstream-garden flags checks both."""
        parent2 = self.root / 'parent2-garden'
        parent2.mkdir()
        for cmd in [
            ['git', 'init', str(parent2)],
            ['git', '-C', str(parent2), 'config', 'user.email', 'test@test.com'],
            ['git', '-C', str(parent2), 'config', 'user.name', 'Test'],
        ]:
            subprocess.run(cmd, check=True, capture_output=True)
        (parent2 / 'GARDEN.md').write_text('**Last assigned ID:** GE-0000\n')
        (parent2 / 'java').mkdir()
        subprocess.run(['git', '-C', str(parent2), 'add', '.'], check=True, capture_output=True)
        subprocess.run(['git', '-C', str(parent2), 'commit', '-m', 'init'], check=True, capture_output=True)

        self._submit_entry(DISTINCT_ENTRY)
        entry = str(self.child / 'java' / 'GE-20260414-test001.md')
        result = self._run_validator(
            entry,
            '--upstream-garden', str(self.parent),
            '--upstream-garden', str(parent2),
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
```

Add these constants near the top of the test file (after existing constants/imports):

```python
UPSTREAM_ENTRY = """\
---
id: GE-20260414-parent1
title: "Hibernate @PreUpdate fires at flush time not at persist"
type: gotcha
domain: java
stack: "Hibernate ORM 6.x, Quarkus"
tags: [hibernate, jpa, java, lifecycle, flush]
score: 12
verified: true
staleness_threshold: 730
submitted: 2026-04-14
---

## Hibernate @PreUpdate fires at flush time not at persist

**ID:** GE-20260414-parent1
**Stack:** Hibernate ORM 6.x, Quarkus
**Symptom:** @PreUpdate not firing when expected.
**Context:** Entity lifecycle callbacks.

### What was tried (didn't work)
- Relied on @PreUpdate to fire at persist() time.

### Root cause
Fires at flush, not persist.

### Fix
Force flush or restructure logic.

### Why this is non-obvious
Naming implies otherwise.

*Score: 12/15 · Included because: common trap · Reservation: none*
"""

UPSTREAM_DUPLICATE_ENTRY = """\
---
id: GE-20260414-test001
title: "Hibernate @PreUpdate fires at flush time not at persist"
type: gotcha
domain: java
stack: "Hibernate ORM 6.x"
tags: [hibernate, jpa, java, lifecycle, flush]
score: 10
verified: true
staleness_threshold: 730
submitted: 2026-04-15
---

## Hibernate @PreUpdate fires at flush time not at persist

**ID:** GE-20260414-test001
**Stack:** Hibernate ORM 6.x
**Symptom:** Same symptom as upstream.
**Context:** Same context.

### What was tried (didn't work)
- Same approaches.

### Root cause
Same root cause.

### Fix
Same fix.

### Why this is non-obvious
Same reason.

*Score: 10/15 · Included because: test · Reservation: none*
"""

DISTINCT_ENTRY = """\
---
id: GE-20260414-test001
title: "CompletableFuture.get() blocks the carrier thread in virtual thread context"
type: gotcha
domain: java
stack: "Java 21+, Virtual Threads"
tags: [java, virtual-threads, async, blocking]
score: 11
verified: true
staleness_threshold: 730
submitted: 2026-04-15
---

## CompletableFuture.get() blocks the carrier thread in virtual thread context

**ID:** GE-20260414-test001
**Stack:** Java 21+, Virtual Threads
**Symptom:** Performance degradation under virtual threads.
**Context:** Java 21 virtual thread migration.

### What was tried (didn't work)
- Used CompletableFuture.get() expecting cooperative scheduling.

### Root cause
get() is a blocking call that pins the carrier thread.

### Fix
Use .join() or structured concurrency instead.

### Why this is non-obvious
Virtual threads appear to handle blocking transparently but don't for all methods.

*Score: 11/15 · Included because: Java 21 migration gotcha · Reservation: none*
"""
```

- [ ] **Step 3: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_pr.py::TestUpstreamGardenDedup -v
```

Expected: 4 tests FAIL — `--upstream-garden` flag not implemented.

- [ ] **Step 4: Implement --upstream-garden in validate_pr.py**

Read `validate_pr.py` carefully. Find:
- The existing L1 Jaccard similarity function (likely using `compute_title_jaccard` or similar)
- Where `sys.argv` is parsed or `argparse` is used
- The main validation flow

Add `--upstream-garden` support:

1. Add argument parsing at the top of the file (after existing arg parsing, before main logic):

```python
import argparse as _argparse
_parser = _argparse.ArgumentParser(add_help=False)
_parser.add_argument('--upstream-garden', action='append', dest='upstream_gardens',
                     metavar='PATH', default=[])
_known, _remaining = _parser.parse_known_args()
UPSTREAM_GARDENS = [_Path(_p).expanduser().resolve() for _p in _known.upstream_gardens]
# Remove --upstream-garden args from sys.argv so existing parsing isn't affected
import sys as _sys
_sys.argv = [_sys.argv[0]] + _remaining
```

2. After existing L1 Jaccard check passes, add the upstream check. Find where the L1 dedup result is used and add:

```python
    # Upstream chain dedup check
    for _upstream_path in UPSTREAM_GARDENS:
        _up_score = _check_jaccard_against_garden(entry_tokens, _upstream_path, same_domain=True)
        if _up_score is not None and _up_score >= DUPLICATE_THRESHOLD:
            log_error(
                f"L1 upstream duplicate: Jaccard={_up_score:.2f} against entry in "
                f"{_upstream_path.name} — too similar to an upstream entry"
            )
```

**Important:** Read the actual validate_pr.py to understand:
- The exact name of the Jaccard function and how to call it
- What `entry_tokens` is called (may be `title_tokens`, `tag_tokens`, etc.)
- The duplicate threshold value
- How `log_error` works

Adapt the above pseudocode to match the actual code. The key contract: if `--upstream-garden PATH` is passed, the entry's title+tags Jaccard is checked against all YAML-frontmatter entries in `PATH/<domain>/` and if any match is above the threshold, validation fails with a message containing "upstream".

- [ ] **Step 5: Run all validate_pr tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_pr.py -v
```

Expected: All tests PASS including the 4 new TestUpstreamGardenDedup tests.

- [ ] **Step 6: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [ ] **Step 7: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_pr.py tests/test_validate_pr.py
git commit -m "feat(federation): validate_pr --upstream-garden checks upstream chain for duplicates

Refs #29"
```

---

## Task 4: _augment/ layer — init_garden.py + augment_entry.py (TDD)

**Files:**
- Modify: `soredium/scripts/init_garden.py`
- Modify: `soredium/tests/test_init_garden.py`
- Create: `soredium/scripts/augment_entry.py`
- Create: `soredium/tests/test_augment_entry.py`

Child gardens can annotate parent entries privately without modifying the parent. `_augment/` holds these files.

**Augmentation file format:**
```yaml
---
target: GE-20260414-aabbcc
target_garden: jvm-garden
augment_type: context
submitted: 2026-04-15
---

## Private context for GE-20260414-aabbcc

[Private notes extending the parent entry locally]
```

- [ ] **Step 1: Write failing tests for init_garden.py _augment/**

Add to `soredium/tests/test_init_garden.py` (in `TestInitGarden` class):

```python
    def test_child_garden_creates_augment_dir(self):
        init_garden(self.root, name='my-garden', description='Private',
                    role='child', ge_prefix='ME-', domains=['java'],
                    upstream=['https://github.com/Hortora/jvm-garden'])
        self.assertTrue((self.root / '_augment').is_dir(),
                        "Child garden should have _augment/ directory")

    def test_canonical_garden_no_augment_dir(self):
        init_garden(self.root, name='jvm-garden', description='JVM',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        self.assertFalse((self.root / '_augment').exists(),
                         "Canonical garden should NOT have _augment/")

    def test_peer_garden_no_augment_dir(self):
        init_garden(self.root, name='tools-garden', description='Tools',
                    role='peer', ge_prefix='TE-', domains=['tools'])
        self.assertFalse((self.root / '_augment').exists(),
                         "Peer garden should NOT have _augment/")

    def test_augment_dir_has_readme(self):
        init_garden(self.root, name='my-garden', description='Private',
                    role='child', ge_prefix='ME-', domains=['java'],
                    upstream=['https://github.com/Hortora/jvm-garden'])
        readme = self.root / '_augment' / 'README.md'
        self.assertTrue(readme.exists(), "_augment/ should have README.md")
        self.assertIn('augment', readme.read_text().lower())
```

Also add to `TestInitGardenCLI`:

```python
    def test_child_role_creates_augment_dir(self):
        result = run_init(
            str(self.root),
            '--name', 'my-garden',
            '--description', 'Private garden',
            '--role', 'child',
            '--ge-prefix', 'ME-',
            '--domains', 'java',
            '--upstream', 'https://github.com/Hortora/jvm-garden',
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertTrue((self.root / '_augment').is_dir())
```

- [ ] **Step 2: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_init_garden.py::TestInitGarden::test_child_garden_creates_augment_dir -v
```

Expected: FAIL — `_augment/` not created.

- [ ] **Step 3: Implement create_augment_dir in init_garden.py**

In `soredium/scripts/init_garden.py`, add:

```python
def create_augment_dir(root: Path) -> None:
    """Create _augment/ directory with README.md for child gardens."""
    augment_dir = root / '_augment'
    augment_dir.mkdir(exist_ok=True)
    readme = augment_dir / 'README.md'
    if readme.exists():
        return
    readme.write_text(
        '# _augment/\n\n'
        'Private annotations on parent garden entries.\n\n'
        'Each file augments a single parent entry without modifying it.\n\n'
        '## Format\n\n'
        '```yaml\n'
        '---\n'
        'target: GE-20260414-aabbcc\n'
        'target_garden: jvm-garden\n'
        'augment_type: context   # context | correction | update\n'
        'submitted: YYYY-MM-DD\n'
        '---\n\n'
        '## Private context for GE-20260414-aabbcc\n\n'
        '[Your private notes here]\n'
        '```\n',
        encoding='utf-8',
    )
```

In `init_garden()`, add after the main file creation loop:

```python
    if role == 'child':
        before = set(root.rglob('*'))
        create_augment_dir(root)
        after = set(root.rglob('*'))
        created.extend(str(p.relative_to(root)) for p in sorted(after - before))
```

Also update the `create_augment_dir` call site — make sure `role` is available in `init_garden()` (it already is as a parameter).

- [ ] **Step 4: Run init_garden tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_init_garden.py -v
```

Expected: All tests PASS including the 5 new _augment/ tests.

- [ ] **Step 5: Write failing tests for augment_entry.py**

Create `soredium/tests/test_augment_entry.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for augment_entry.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from augment_entry import (
    create_augmentation, list_augmentations, validate_augmentation,
)

CLI = Path(__file__).parent.parent / 'scripts' / 'augment_entry.py'
VALID_TYPES = ('context', 'correction', 'update')


def run_cli(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(CLI)] + list(args),
        capture_output=True, text=True
    )


# ── create_augmentation ────────────────────────────────────────────────────────

class TestCreateAugmentation(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.augment_dir = Path(self.tmp.name) / '_augment'
        self.augment_dir.mkdir()

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_file(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Some private context'
        )
        self.assertTrue(path.exists())

    def test_file_contains_target_id(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Some private context'
        )
        content = path.read_text()
        self.assertIn('GE-20260414-aabbcc', content)

    def test_file_contains_target_garden(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Some private context'
        )
        content = path.read_text()
        self.assertIn('jvm-garden', content)

    def test_file_contains_augment_type(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'correction', 'Fix to parent entry'
        )
        content = path.read_text()
        self.assertIn('correction', content)

    def test_file_contains_content(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Secret internal note'
        )
        content = path.read_text()
        self.assertIn('Secret internal note', content)

    def test_filename_includes_target_id(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Note'
        )
        self.assertIn('GE-20260414-aabbcc', path.name)

    def test_all_valid_augment_types_accepted(self):
        for i, aug_type in enumerate(VALID_TYPES):
            path = create_augmentation(
                self.augment_dir, f'GE-20260414-id{i:04d}',
                'jvm-garden', aug_type, f'Content for {aug_type}'
            )
            self.assertTrue(path.exists())

    def test_has_yaml_frontmatter(self):
        path = create_augmentation(
            self.augment_dir, 'GE-20260414-aabbcc',
            'jvm-garden', 'context', 'Note'
        )
        content = path.read_text()
        self.assertTrue(content.startswith('---\n'))
        self.assertIn('\n---\n', content)


# ── list_augmentations ────────────────────────────────────────────────────────

class TestListAugmentations(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.augment_dir = Path(self.tmp.name) / '_augment'
        self.augment_dir.mkdir()

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_dir_returns_empty_list(self):
        result = list_augmentations(self.augment_dir)
        self.assertEqual(result, [])

    def test_single_augmentation(self):
        create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                            'jvm-garden', 'context', 'Note')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(len(result), 1)

    def test_returns_target_field(self):
        create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                            'jvm-garden', 'context', 'Note')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(result[0]['target'], 'GE-20260414-aabbcc')

    def test_returns_target_garden_field(self):
        create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                            'jvm-garden', 'context', 'Note')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(result[0]['target_garden'], 'jvm-garden')

    def test_returns_augment_type_field(self):
        create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                            'jvm-garden', 'correction', 'Note')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(result[0]['augment_type'], 'correction')

    def test_multiple_augmentations(self):
        for i in range(3):
            create_augmentation(self.augment_dir, f'GE-20260414-id{i:04d}',
                                'jvm-garden', 'context', f'Note {i}')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(len(result), 3)

    def test_skips_readme(self):
        (self.augment_dir / 'README.md').write_text('# README\n')
        result = list_augmentations(self.augment_dir)
        self.assertEqual(result, [])


# ── validate_augmentation ─────────────────────────────────────────────────────

class TestValidateAugmentation(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.augment_dir = Path(self.tmp.name) / '_augment'
        self.augment_dir.mkdir()

    def tearDown(self):
        self.tmp.cleanup()

    def test_valid_file_no_errors(self):
        path = create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                                   'jvm-garden', 'context', 'Valid note')
        errors, _ = validate_augmentation(path)
        self.assertEqual(errors, [])

    def test_missing_target_field(self):
        p = self.augment_dir / 'bad.md'
        p.write_text('---\ntarget_garden: jvm-garden\naugment_type: context\n---\n\nNote\n')
        errors, _ = validate_augmentation(p)
        self.assertTrue(any('target' in e for e in errors))

    def test_missing_target_garden_field(self):
        p = self.augment_dir / 'bad2.md'
        p.write_text('---\ntarget: GE-20260414-aabbcc\naugment_type: context\n---\n\nNote\n')
        errors, _ = validate_augmentation(p)
        self.assertTrue(any('target_garden' in e for e in errors))

    def test_invalid_augment_type(self):
        p = self.augment_dir / 'bad3.md'
        p.write_text('---\ntarget: GE-20260414-aabbcc\ntarget_garden: jvm-garden\n'
                     'augment_type: bogus\n---\n\nNote\n')
        errors, _ = validate_augmentation(p)
        self.assertTrue(any('augment_type' in e for e in errors))

    def test_no_frontmatter(self):
        p = self.augment_dir / 'bad4.md'
        p.write_text('Just plain text, no frontmatter\n')
        errors, _ = validate_augmentation(p)
        self.assertGreater(len(errors), 0)


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestAugmentEntryCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.augment_dir = Path(self.tmp.name) / '_augment'
        self.augment_dir.mkdir()

    def tearDown(self):
        self.tmp.cleanup()

    def test_create_subcommand(self):
        result = run_cli(
            'create', 'GE-20260414-aabbcc',
            '--garden', 'jvm-garden',
            '--type', 'context',
            '--content', 'Private note here',
            '--augment-dir', str(self.augment_dir),
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        files = list(self.augment_dir.glob('*.md'))
        self.assertEqual(len(files), 1)

    def test_list_subcommand_empty(self):
        result = run_cli('list', '--augment-dir', str(self.augment_dir))
        self.assertEqual(result.returncode, 0)
        self.assertIn('0', result.stdout)

    def test_list_subcommand_after_create(self):
        create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                            'jvm-garden', 'context', 'Note')
        result = run_cli('list', '--augment-dir', str(self.augment_dir))
        self.assertEqual(result.returncode, 0)
        self.assertIn('GE-20260414-aabbcc', result.stdout)

    def test_validate_subcommand_valid(self):
        path = create_augmentation(self.augment_dir, 'GE-20260414-aabbcc',
                                   'jvm-garden', 'context', 'Note')
        result = run_cli('validate', str(path))
        self.assertEqual(result.returncode, 0)
        self.assertIn('valid', result.stdout.lower())

    def test_validate_subcommand_invalid(self):
        p = self.augment_dir / 'bad.md'
        p.write_text('no frontmatter\n')
        result = run_cli('validate', str(p))
        self.assertEqual(result.returncode, 1)

    def test_invalid_type_exits_nonzero(self):
        result = run_cli(
            'create', 'GE-20260414-aabbcc',
            '--garden', 'jvm-garden',
            '--type', 'bogus-type',
            '--content', 'Note',
            '--augment-dir', str(self.augment_dir),
        )
        self.assertNotEqual(result.returncode, 0)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [ ] **Step 6: Run — confirm FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_augment_entry.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'augment_entry'`

- [ ] **Step 7: Create augment_entry.py**

Create `soredium/scripts/augment_entry.py`:

```python
#!/usr/bin/env python3
"""augment_entry.py — Create and manage private augmentations of parent garden entries.

Usage:
  augment_entry.py create GE-XXXX --garden GARDEN --type TYPE --content TEXT [--augment-dir DIR]
  augment_entry.py list [--augment-dir DIR]
  augment_entry.py validate PATH

augment_type: context | correction | update
"""

import argparse
import re
import sys
from datetime import date
from pathlib import Path

VALID_TYPES = {'context', 'correction', 'update'}
FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---', re.DOTALL)
TODAY = date.today().isoformat()


def create_augmentation(augment_dir: Path, target_id: str, target_garden: str,
                        augment_type: str, content: str) -> Path:
    """Write an augmentation file. Returns the created path."""
    path = augment_dir / f'{target_id}.md'
    path.write_text(
        f'---\n'
        f'target: {target_id}\n'
        f'target_garden: {target_garden}\n'
        f'augment_type: {augment_type}\n'
        f'submitted: {TODAY}\n'
        f'---\n\n'
        f'## Private context for {target_id}\n\n'
        f'{content}\n',
        encoding='utf-8',
    )
    return path


def _parse_frontmatter(content: str) -> dict | None:
    content = content.replace('\r\n', '\n')
    m = FRONTMATTER_RE.match(content)
    if not m:
        return None
    result = {}
    for line in m.group(1).splitlines():
        if ':' in line:
            key, _, val = line.partition(':')
            result[key.strip()] = val.strip().strip('"\'')
    return result


def list_augmentations(augment_dir: Path) -> list:
    """Return list of {target, target_garden, augment_type, submitted, path} for all augmentations."""
    results = []
    for p in sorted(augment_dir.glob('*.md')):
        if p.name == 'README.md':
            continue
        fm = _parse_frontmatter(p.read_text(encoding='utf-8'))
        if fm is None:
            continue
        results.append({
            'target': fm.get('target', ''),
            'target_garden': fm.get('target_garden', ''),
            'augment_type': fm.get('augment_type', ''),
            'submitted': fm.get('submitted', ''),
            'path': p,
        })
    return results


def validate_augmentation(path: Path) -> tuple:
    """Validate an augmentation file. Returns (errors, warnings)."""
    errors = []
    warnings = []
    content = path.read_text(encoding='utf-8')
    fm = _parse_frontmatter(content)
    if fm is None:
        errors.append("No YAML frontmatter found")
        return errors, warnings
    if not fm.get('target'):
        errors.append("'target' is required (the parent entry GE-ID)")
    if not fm.get('target_garden'):
        errors.append("'target_garden' is required (name of the parent garden)")
    aug_type = fm.get('augment_type', '')
    if not aug_type:
        errors.append("'augment_type' is required")
    elif aug_type not in VALID_TYPES:
        errors.append(f"'augment_type' is {aug_type!r} — must be one of: {', '.join(sorted(VALID_TYPES))}")
    return errors, warnings


def main():
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = parser.add_subparsers(dest='command', required=True)

    create_p = sub.add_parser('create', help='Create an augmentation')
    create_p.add_argument('target_id', help='Parent entry GE-ID (e.g. GE-20260414-aabbcc)')
    create_p.add_argument('--garden', required=True, dest='target_garden')
    create_p.add_argument('--type', required=True, dest='augment_type',
                          choices=sorted(VALID_TYPES))
    create_p.add_argument('--content', required=True)
    create_p.add_argument('--augment-dir', type=Path, default=Path('_augment'))

    list_p = sub.add_parser('list', help='List augmentations')
    list_p.add_argument('--augment-dir', type=Path, default=Path('_augment'))

    validate_p = sub.add_parser('validate', help='Validate an augmentation file')
    validate_p.add_argument('path', type=Path)

    args = parser.parse_args()

    if args.command == 'create':
        path = create_augmentation(
            args.augment_dir, args.target_id,
            args.target_garden, args.augment_type, args.content
        )
        print(f"Created: {path}")
        sys.exit(0)

    elif args.command == 'list':
        augmentations = list_augmentations(args.augment_dir)
        if not augmentations:
            print(f"0 augmentations in {args.augment_dir}")
        else:
            print(f"{len(augmentations)} augmentation(s):")
            for a in augmentations:
                print(f"  {a['target']} ({a['augment_type']}) ← {a['target_garden']}")
        sys.exit(0)

    elif args.command == 'validate':
        errors, warnings = validate_augmentation(args.path)
        for w in warnings:
            print(f"WARNING: {w}")
        if errors:
            for e in errors:
                print(f"ERROR: {e}")
            sys.exit(1)
        print(f"✅ {args.path.name} is valid")
        sys.exit(0)


if __name__ == '__main__':
    main()
```

- [ ] **Step 8: Run all augment_entry and init_garden tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_augment_entry.py tests/test_init_garden.py -v
```

Expected: All tests PASS.

- [ ] **Step 9: Run full suite**

```bash
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [ ] **Step 10: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/init_garden.py scripts/augment_entry.py \
        tests/test_init_garden.py tests/test_augment_entry.py
git commit -m "feat(federation): add _augment/ layer for child gardens + augment_entry.py

init_garden creates _augment/ for child gardens only. augment_entry.py
manages private annotations on parent entries (create/list/validate).

Refs #29"
```

- [ ] **Step 11: Push all Phase 6 commits**

```bash
cd ~/claude/hortora/soredium
git push
```

---

## Self-Review

**Spec coverage:**

| Phase 6 requirement | Task |
|---|---|
| Client config (`garden-config.toml`) | Task 1 |
| Domain ownership resolution | Task 1 |
| Upstream chain resolution | Task 1 |
| Submission routing: domain → garden | Task 2 |
| Non-duplication enforcement across upstream chain | Task 3 |
| `_augment/` layer — private child augmentation | Task 4 |
| Multi-garden retrieval (skill update) | Out of scope for this plan — forage SKILL.md update |

**Out of scope (future):** forage SKILL.md update for multi-garden SEARCH, peer retrieval, `_watch/` CI monitoring (Phase 9).

**No gaps in codeable scope. No placeholders.**

**Type consistency:**
- `load_config()` → `list[dict]`
- `validate_config()` → `tuple[list[str], list[str]]`
- `find_garden_for_domain()` → `dict | None` (or `tuple[dict|None, list[str]]` with `return_warnings=True`)
- `get_upstream_chain()` → `list[Path]`
- `route()` → `Path | None`
- `create_augmentation()` → `Path`
- `list_augmentations()` → `list[dict]`
- `validate_augmentation()` → `tuple[list[str], list[str]]`

All consistent across tasks.
