# Phase 5 — Ecosystem Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `validate_schema.py` (federation config validator) and `init_garden.py` (canonical garden initializer) — the tooling needed to stand up `jvm-garden` and `tools-garden` as properly structured canonical knowledge gardens.

**Architecture:** Two new Python scripts in `soredium/scripts/`. `validate_schema.py` parses and validates a garden's `SCHEMA.md` federation configuration (role, ge_prefix, domains, upstream). `init_garden.py` creates a new canonical, child, or peer garden from scratch. `validate_garden.py` is extended to call `validate_schema` when `SCHEMA.md` is present. All scripts follow existing patterns: regex-based parsing (no new runtime dependencies), subprocess-safe CLI, exit codes 0/1.

**Tech Stack:** Python 3.11+, re, subprocess, pathlib, pytest. No new dependencies.

---

## File Map

| File | Status | Responsibility |
|------|--------|----------------|
| `scripts/validate_schema.py` | Create | Parse and validate SCHEMA.md federation config |
| `scripts/init_garden.py` | Create | Initialize a new garden directory |
| `scripts/validate_garden.py` | Modify | Call validate_schema when SCHEMA.md present |
| `tests/test_validate_schema.py` | Create | Unit + integration + CLI tests |
| `tests/test_init_garden.py` | Create | Unit + integration + CLI tests |
| `tests/test_validate_garden.py` | Modify | Add TestSchemaValidation class |
| `tests/test_integration.py` | Modify | Add TestE2EInitGardenPipeline class |
| `tests/garden_fixture.py` | Modify | Add schema_md() builder method |

---

## SCHEMA.md Format

All garden repos contain a `SCHEMA.md` at root. YAML frontmatter file.

**Canonical:**
```
---
name: jvm-garden
description: "JVM ecosystem knowledge garden"
role: canonical
ge_prefix: JE-
schema_version: "1.0"
domains: [java, quarkus, spring, kotlin]
---
```

**Child:**
```
---
name: my-private-garden
description: "Private extension of jvm-garden"
role: child
ge_prefix: ME-
schema_version: "1.0"
domains: [java, quarkus]
upstream:
  - https://github.com/Hortora/jvm-garden
---
```

**Peer:**
```
---
name: tools-garden
description: "Cross-cutting tools knowledge"
role: peer
ge_prefix: TE-
schema_version: "1.0"
domains: [tools, cli, git]
---
```

### Validation rules

| Field | Rule |
|-------|------|
| `name` | required, non-empty string |
| `description` | required |
| `role` | required, one of: `canonical`, `child`, `peer` |
| `ge_prefix` | required, 1–3 uppercase letters + hyphen (e.g. `JE-`) |
| `schema_version` | required; unknown version → warning, not error |
| `domains` | required, non-empty list |
| `upstream` | required for `child`; forbidden for `canonical` and `peer` |

---

## Task 1: validate_schema.py — core functions (TDD, unit tests)

**Files:**
- Create: `soredium/scripts/validate_schema.py`
- Create: `soredium/tests/test_validate_schema.py`

- [x] **Step 1: Write failing unit tests**

Create `soredium/tests/test_validate_schema.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for validate_schema.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from validate_schema import (
    parse_schema, validate_name, validate_role, validate_ge_prefix,
    validate_domains, validate_upstream, validate_schema,
)

VALIDATOR = Path(__file__).parent.parent / 'scripts' / 'validate_schema.py'


def run_validator(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(VALIDATOR)] + list(args),
        capture_output=True, text=True
    )


CANONICAL_SCHEMA = """\
---
name: jvm-garden
description: "JVM ecosystem knowledge garden"
role: canonical
ge_prefix: JE-
schema_version: "1.0"
domains: [java, quarkus, spring, kotlin]
---
"""

CHILD_SCHEMA = """\
---
name: my-private-garden
description: "Private extension of jvm-garden"
role: child
ge_prefix: ME-
schema_version: "1.0"
domains: [java, quarkus]
upstream:
  - https://github.com/Hortora/jvm-garden
---
"""

PEER_SCHEMA = """\
---
name: tools-garden
description: "Cross-cutting tools knowledge"
role: peer
ge_prefix: TE-
schema_version: "1.0"
domains: [tools, cli, git]
---
"""


# ── parse_schema ───────────────────────────────────────────────────────────────

class TestParseSchema(unittest.TestCase):

    def test_parses_canonical_schema(self):
        result = parse_schema(CANONICAL_SCHEMA)
        self.assertEqual(result['name'], 'jvm-garden')
        self.assertEqual(result['role'], 'canonical')
        self.assertEqual(result['ge_prefix'], 'JE-')
        self.assertEqual(result['domains'], ['java', 'quarkus', 'spring', 'kotlin'])

    def test_parses_child_schema_with_upstream(self):
        result = parse_schema(CHILD_SCHEMA)
        self.assertEqual(result['role'], 'child')
        self.assertIn('upstream', result)
        self.assertEqual(result['upstream'], ['https://github.com/Hortora/jvm-garden'])

    def test_parses_peer_schema(self):
        result = parse_schema(PEER_SCHEMA)
        self.assertEqual(result['role'], 'peer')
        self.assertEqual(result['domains'], ['tools', 'cli', 'git'])

    def test_missing_frontmatter_returns_none(self):
        self.assertIsNone(parse_schema('# No frontmatter\n'))

    def test_empty_string_returns_none(self):
        self.assertIsNone(parse_schema(''))

    def test_crlf_normalised(self):
        result = parse_schema(CANONICAL_SCHEMA.replace('\n', '\r\n'))
        self.assertIsNotNone(result)
        self.assertEqual(result['name'], 'jvm-garden')

    def test_inline_domain_list(self):
        result = parse_schema('---\nname: x\ndomains: [java, quarkus]\n---\n')
        self.assertEqual(result['domains'], ['java', 'quarkus'])

    def test_block_domain_list(self):
        result = parse_schema('---\nname: x\ndomains:\n  - java\n  - quarkus\n---\n')
        self.assertEqual(result['domains'], ['java', 'quarkus'])

    def test_block_upstream_list(self):
        schema = (
            '---\nrole: child\nupstream:\n'
            '  - https://github.com/Hortora/jvm-garden\n'
            '  - https://github.com/Hortora/tools-garden\n---\n'
        )
        result = parse_schema(schema)
        self.assertEqual(result['upstream'], [
            'https://github.com/Hortora/jvm-garden',
            'https://github.com/Hortora/tools-garden',
        ])


# ── validate_name ─────────────────────────────────────────────────────────────

class TestValidateName(unittest.TestCase):

    def test_valid_name(self):
        self.assertEqual(validate_name({'name': 'jvm-garden'}), [])

    def test_missing_name(self):
        errors = validate_name({})
        self.assertEqual(len(errors), 1)
        self.assertIn('name', errors[0])

    def test_empty_name(self):
        errors = validate_name({'name': ''})
        self.assertEqual(len(errors), 1)

    def test_name_with_spaces_valid(self):
        self.assertEqual(validate_name({'name': 'my garden'}), [])


# ── validate_role ─────────────────────────────────────────────────────────────

class TestValidateRole(unittest.TestCase):

    def test_canonical_valid(self):
        self.assertEqual(validate_role({'role': 'canonical'}), [])

    def test_child_valid(self):
        self.assertEqual(validate_role({'role': 'child'}), [])

    def test_peer_valid(self):
        self.assertEqual(validate_role({'role': 'peer'}), [])

    def test_missing_role(self):
        errors = validate_role({})
        self.assertEqual(len(errors), 1)
        self.assertIn('role', errors[0])

    def test_invalid_role(self):
        errors = validate_role({'role': 'master'})
        self.assertEqual(len(errors), 1)
        self.assertIn('canonical', errors[0])

    def test_case_sensitive(self):
        errors = validate_role({'role': 'Canonical'})
        self.assertEqual(len(errors), 1)


# ── validate_ge_prefix ────────────────────────────────────────────────────────

class TestValidateGePrefix(unittest.TestCase):

    def test_two_letter_prefix(self):
        self.assertEqual(validate_ge_prefix({'ge_prefix': 'JE-'}), [])

    def test_one_letter_prefix(self):
        self.assertEqual(validate_ge_prefix({'ge_prefix': 'J-'}), [])

    def test_three_letter_prefix(self):
        self.assertEqual(validate_ge_prefix({'ge_prefix': 'JVM-'}), [])

    def test_missing_ge_prefix(self):
        errors = validate_ge_prefix({})
        self.assertEqual(len(errors), 1)
        self.assertIn('ge_prefix', errors[0])

    def test_lowercase_rejected(self):
        errors = validate_ge_prefix({'ge_prefix': 'je-'})
        self.assertEqual(len(errors), 1)

    def test_missing_hyphen_rejected(self):
        errors = validate_ge_prefix({'ge_prefix': 'JE'})
        self.assertEqual(len(errors), 1)

    def test_four_letters_rejected(self):
        errors = validate_ge_prefix({'ge_prefix': 'JAVA-'})
        self.assertEqual(len(errors), 1)

    def test_empty_rejected(self):
        errors = validate_ge_prefix({'ge_prefix': ''})
        self.assertEqual(len(errors), 1)


# ── validate_domains ──────────────────────────────────────────────────────────

class TestValidateDomains(unittest.TestCase):

    def test_single_domain_valid(self):
        self.assertEqual(validate_domains({'domains': ['java']}), [])

    def test_multiple_domains_valid(self):
        self.assertEqual(validate_domains({'domains': ['java', 'quarkus', 'spring']}), [])

    def test_missing_domains(self):
        errors = validate_domains({})
        self.assertEqual(len(errors), 1)
        self.assertIn('domains', errors[0])

    def test_empty_list_rejected(self):
        errors = validate_domains({'domains': []})
        self.assertEqual(len(errors), 1)

    def test_non_list_rejected(self):
        errors = validate_domains({'domains': 'java'})
        self.assertEqual(len(errors), 1)


# ── validate_upstream ─────────────────────────────────────────────────────────

class TestValidateUpstream(unittest.TestCase):

    def test_child_with_upstream_valid(self):
        schema = {'role': 'child', 'upstream': ['https://github.com/Hortora/jvm-garden']}
        self.assertEqual(validate_upstream(schema), [])

    def test_child_without_upstream_rejected(self):
        errors = validate_upstream({'role': 'child'})
        self.assertEqual(len(errors), 1)
        self.assertIn('upstream', errors[0])

    def test_child_with_empty_upstream_rejected(self):
        errors = validate_upstream({'role': 'child', 'upstream': []})
        self.assertEqual(len(errors), 1)

    def test_canonical_with_upstream_rejected(self):
        errors = validate_upstream({'role': 'canonical', 'upstream': ['https://example.com']})
        self.assertEqual(len(errors), 1)
        self.assertIn('canonical', errors[0])

    def test_canonical_without_upstream_valid(self):
        self.assertEqual(validate_upstream({'role': 'canonical'}), [])

    def test_peer_without_upstream_valid(self):
        self.assertEqual(validate_upstream({'role': 'peer'}), [])

    def test_peer_with_upstream_rejected(self):
        errors = validate_upstream({'role': 'peer', 'upstream': ['https://example.com']})
        self.assertEqual(len(errors), 1)

    def test_upstream_items_must_be_strings(self):
        errors = validate_upstream({'role': 'child', 'upstream': [123]})
        self.assertEqual(len(errors), 1)


# ── validate_schema ───────────────────────────────────────────────────────────

class TestValidateSchema(unittest.TestCase):

    def test_valid_canonical_no_errors(self):
        schema = parse_schema(CANONICAL_SCHEMA)
        errors, warnings = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_valid_child_no_errors(self):
        schema = parse_schema(CHILD_SCHEMA)
        errors, warnings = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_valid_peer_no_errors(self):
        schema = parse_schema(PEER_SCHEMA)
        errors, warnings = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_multiple_missing_fields_all_reported(self):
        errors, warnings = validate_schema({'role': 'canonical'})
        self.assertGreaterEqual(len(errors), 3)

    def test_unknown_schema_version_warns_not_errors(self):
        schema = parse_schema(CANONICAL_SCHEMA.replace('"1.0"', '"99.0"'))
        errors, warnings = validate_schema(schema)
        self.assertEqual(errors, [])
        self.assertTrue(any('schema_version' in w for w in warnings))

    def test_empty_dict_returns_multiple_errors(self):
        errors, warnings = validate_schema({})
        self.assertGreater(len(errors), 0)


# ── Integration tests — full SCHEMA.md files ──────────────────────────────────

class TestSchemaIntegration(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def _write(self, content: str):
        (self.root / 'SCHEMA.md').write_text(content)

    def test_canonical_garden_valid(self):
        self._write(CANONICAL_SCHEMA)
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_child_garden_valid(self):
        self._write(CHILD_SCHEMA)
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_peer_garden_valid(self):
        self._write(PEER_SCHEMA)
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_child_without_upstream_fails(self):
        content = CHILD_SCHEMA.replace(
            'upstream:\n  - https://github.com/Hortora/jvm-garden\n', ''
        )
        self._write(content)
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertTrue(any('upstream' in e for e in errors))

    def test_invalid_ge_prefix_fails(self):
        self._write(CANONICAL_SCHEMA.replace('ge_prefix: JE-', 'ge_prefix: java-'))
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertTrue(any('ge_prefix' in e for e in errors))

    def test_crlf_schema_valid(self):
        self._write(CANONICAL_SCHEMA.replace('\n', '\r\n'))
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [])


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestValidateSchemaCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_valid_canonical_exits_0(self):
        (self.root / 'SCHEMA.md').write_text(CANONICAL_SCHEMA)
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertIn('✅', result.stdout)

    def test_valid_child_exits_0(self):
        (self.root / 'SCHEMA.md').write_text(CHILD_SCHEMA)
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 0)

    def test_invalid_exits_1(self):
        (self.root / 'SCHEMA.md').write_text('---\nname: broken\n---\n')
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 1)
        self.assertIn('❌', result.stdout)

    def test_missing_schema_md_exits_1(self):
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 1)
        self.assertIn('SCHEMA.md', result.stdout + result.stderr)

    def test_output_shows_role_and_prefix_on_success(self):
        (self.root / 'SCHEMA.md').write_text(CANONICAL_SCHEMA)
        result = run_validator(str(self.root))
        self.assertIn('canonical', result.stdout)
        self.assertIn('JE-', result.stdout)

    def test_all_errors_reported(self):
        (self.root / 'SCHEMA.md').write_text('---\nrole: canonical\n---\n')
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 1)
        self.assertGreaterEqual(result.stdout.count('ERROR'), 3)

    def test_unknown_version_warns_exits_0(self):
        (self.root / 'SCHEMA.md').write_text(
            CANONICAL_SCHEMA.replace('"1.0"', '"99.0"')
        )
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 0)
        self.assertIn('WARNING', result.stdout)

    def test_no_frontmatter_exits_1(self):
        (self.root / 'SCHEMA.md').write_text('# No frontmatter here\n')
        result = run_validator(str(self.root))
        self.assertEqual(result.returncode, 1)


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [x] **Step 2: Run tests — verify they FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_schema.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'validate_schema'`

- [x] **Step 3: Create validate_schema.py**

Create `soredium/scripts/validate_schema.py`:

```python
#!/usr/bin/env python3
"""validate_schema.py — Validates SCHEMA.md federation config for a garden repo.

Usage:
  validate_schema.py <garden_path>

Exit codes: 0 = valid, 1 = invalid
"""

import re
import sys
from pathlib import Path

KNOWN_SCHEMA_VERSIONS = {'1.0'}
VALID_ROLES = {'canonical', 'child', 'peer'}
GE_PREFIX_RE = re.compile(r'^[A-Z]{1,3}-$')
FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---', re.DOTALL)


# ── Parsing ────────────────────────────────────────────────────────────────────

def parse_schema(content: str) -> dict | None:
    """Parse SCHEMA.md content and return field dict, or None if no frontmatter."""
    content = content.replace('\r\n', '\n')
    m = FRONTMATTER_RE.match(content)
    if not m:
        return None
    return _parse_frontmatter(m.group(1))


def _parse_frontmatter(fm: str) -> dict:
    result = {}
    lines = fm.splitlines()
    i = 0
    while i < len(lines):
        line = lines[i]
        if not line.strip() or line.startswith('#'):
            i += 1
            continue
        if ':' not in line:
            i += 1
            continue
        key, _, rest = line.partition(':')
        key = key.strip()
        rest = rest.strip()
        if not rest:
            # Block list
            items = []
            i += 1
            while i < len(lines) and re.match(r'^\s+-\s+', lines[i]):
                items.append(re.sub(r'^\s+-\s+', '', lines[i]).strip().strip('"\''))
                i += 1
            if items:
                result[key] = items
            continue
        elif rest.startswith('[') and rest.endswith(']'):
            inner = rest[1:-1]
            result[key] = [v.strip().strip('"\'') for v in inner.split(',') if v.strip()]
        else:
            result[key] = rest.strip('"\'')
        i += 1
    return result


# ── Validators ─────────────────────────────────────────────────────────────────

def validate_name(schema: dict) -> list[str]:
    name = schema.get('name', '')
    if not name:
        return ["'name' is required and must be non-empty"]
    return []


def validate_role(schema: dict) -> list[str]:
    role = schema.get('role')
    if not role:
        return [f"'role' is required — must be one of: {', '.join(sorted(VALID_ROLES))}"]
    if role not in VALID_ROLES:
        return [f"'role' is {role!r} — must be one of: {', '.join(sorted(VALID_ROLES))}"]
    return []


def validate_ge_prefix(schema: dict) -> list[str]:
    prefix = schema.get('ge_prefix', '')
    if not prefix:
        return ["'ge_prefix' is required (e.g. 'JE-', 'TE-')"]
    if not GE_PREFIX_RE.match(prefix):
        return [
            f"'ge_prefix' is {prefix!r} — must be 1–3 uppercase letters followed by "
            "a hyphen (e.g. 'JE-', 'TE-', 'JVM-')"
        ]
    return []


def validate_domains(schema: dict) -> list[str]:
    domains = schema.get('domains')
    if domains is None:
        return ["'domains' is required — list of domain names (e.g. [java, quarkus])"]
    if not isinstance(domains, list):
        return ["'domains' must be a list"]
    if not domains:
        return ["'domains' must be non-empty"]
    return []


def validate_upstream(schema: dict) -> list[str]:
    role = schema.get('role')
    upstream = schema.get('upstream')
    if role == 'child':
        if not upstream:
            return ["'upstream' is required for child gardens — list one or more parent garden URLs"]
        if not isinstance(upstream, list) or not upstream:
            return ["'upstream' must be a non-empty list for child gardens"]
        for item in upstream:
            if not isinstance(item, str):
                return [f"'upstream' items must be strings (URLs), got: {item!r}"]
    elif role in ('canonical', 'peer') and upstream:
        return [f"'upstream' must not be set for {role} gardens — only child gardens have upstream parents"]
    return []


def validate_schema(schema: dict) -> tuple[list[str], list[str]]:
    """Validate schema dict. Returns (errors, warnings)."""
    errors: list[str] = []
    warnings: list[str] = []
    errors.extend(validate_name(schema))
    errors.extend(validate_role(schema))
    errors.extend(validate_ge_prefix(schema))
    errors.extend(validate_domains(schema))
    errors.extend(validate_upstream(schema))
    version = schema.get('schema_version', '')
    if not version:
        errors.append("'schema_version' is required")
    elif version not in KNOWN_SCHEMA_VERSIONS:
        warnings.append(
            f"'schema_version' is {version!r} — known versions: "
            f"{', '.join(sorted(KNOWN_SCHEMA_VERSIONS))}"
        )
    return errors, warnings


# ── CLI ────────────────────────────────────────────────────────────────────────

def main():
    if len(sys.argv) < 2 or sys.argv[1] in ('--help', '-h'):
        print(__doc__)
        sys.exit(0)
    garden = Path(sys.argv[1]).expanduser().resolve()
    schema_path = garden / 'SCHEMA.md'
    if not schema_path.exists():
        print(f"ERROR: SCHEMA.md not found in {garden}")
        sys.exit(1)
    content = schema_path.read_text(encoding='utf-8')
    schema = parse_schema(content)
    if schema is None:
        print("ERROR: SCHEMA.md has no YAML frontmatter")
        sys.exit(1)
    errors, warnings = validate_schema(schema)
    for w in warnings:
        print(f"WARNING: {w}")
    if errors:
        for e in errors:
            print(f"ERROR: {e}")
        print(f"\n❌ {len(errors)} error(s), {len(warnings)} warning(s)")
        sys.exit(1)
    print(
        f"✅ SCHEMA.md valid — role={schema.get('role')}, "
        f"ge_prefix={schema.get('ge_prefix')}, domains={schema.get('domains')}"
    )
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [x] **Step 4: Run all tests — verify they PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_schema.py -v
```

Expected: All tests PASS.

- [x] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_schema.py tests/test_validate_schema.py
git commit -m "feat(schema): add validate_schema.py with full test suite — federation config validation

Refs #12"
```

---

## Task 2: validate_garden.py + GardenFixture — check SCHEMA.md (TDD)

**Files:**
- Modify: `soredium/tests/garden_fixture.py`
- Modify: `soredium/tests/test_validate_garden.py`
- Modify: `soredium/scripts/validate_garden.py`

- [x] **Step 1: Add schema_md() to GardenFixture**

Read `soredium/tests/garden_fixture.py` first to understand the existing builder pattern. Then add this method to the `GardenFixture` class:

```python
def schema_md(self, role: str = 'canonical', ge_prefix: str = 'GE-',
              domains: list = None, name: str = 'test-garden',
              upstream: list = None) -> 'GardenFixture':
    """Write a valid SCHEMA.md to the garden root."""
    domains = domains or ['tools']
    lines = [
        '---',
        f'name: {name}',
        'description: "Test garden"',
        f'role: {role}',
        f'ge_prefix: {ge_prefix}',
        'schema_version: "1.0"',
        f'domains: [{", ".join(domains)}]',
    ]
    if upstream:
        lines.append('upstream:')
        for url in upstream:
            lines.append(f'  - {url}')
    lines.extend(['---', ''])
    (self.root / 'SCHEMA.md').write_text('\n'.join(lines))
    return self
```

- [x] **Step 2: Write failing tests**

Add this class to `soredium/tests/test_validate_garden.py` (after existing imports — `GardenFixture` is already imported there):

```python
class TestSchemaValidation(unittest.TestCase):
    """validate_garden.py calls validate_schema when SCHEMA.md is present."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_garden_without_schema_passes(self):
        GardenFixture(self.root).garden_md('GE-0001')
        result = run_validator(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_garden_with_valid_schema_passes(self):
        GardenFixture(self.root).garden_md('GE-0001').schema_md()
        result = run_validator(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_garden_with_invalid_schema_fails(self):
        GardenFixture(self.root).garden_md('GE-0001')
        (self.root / 'SCHEMA.md').write_text('---\nrole: canonical\n---\n')
        result = run_validator(self.root)
        self.assertEqual(result.returncode, 1, result.stdout)
        output = result.stdout + result.stderr
        self.assertTrue(
            any(term in output for term in ('SCHEMA', 'ge_prefix', 'name', 'domains')),
            f"Expected schema error in output, got: {output}"
        )

    def test_garden_with_child_schema_and_upstream_passes(self):
        GardenFixture(self.root).garden_md('GE-0001').schema_md(
            role='child',
            upstream=['https://github.com/Hortora/jvm-garden'],
        )
        result = run_validator(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_child_schema_missing_upstream_fails(self):
        GardenFixture(self.root).garden_md('GE-0001')
        (self.root / 'SCHEMA.md').write_text(
            '---\nname: x\nrole: child\nge_prefix: JE-\n'
            'domains: [java]\nschema_version: "1.0"\n---\n'
        )
        result = run_validator(self.root)
        self.assertEqual(result.returncode, 1)
        self.assertIn('upstream', result.stdout + result.stderr)
```

- [x] **Step 3: Run — verify they FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py::TestSchemaValidation -v
```

Expected: All 5 FAIL — `validate_garden.py` doesn't call `validate_schema` yet.

- [x] **Step 4: Implement SCHEMA.md check in validate_garden.py**

Read `soredium/scripts/validate_garden.py` to find where `GARDEN_MD` is defined and where the main validation body begins. Then add this block immediately after the GARDEN.md existence check (keep it near the top of the validation body, before entry-level checks):

```python
    # 0. Validate SCHEMA.md if present (federation config)
    _schema_path = GARDEN_MD.parent / 'SCHEMA.md'
    if _schema_path.exists():
        import sys as _sys
        import importlib.util as _ilu
        _spec = _ilu.spec_from_file_location(
            'validate_schema',
            Path(__file__).parent / 'validate_schema.py'
        )
        _vs = _ilu.module_from_spec(_spec)
        _spec.loader.exec_module(_vs)
        _schema_content = _schema_path.read_text(encoding='utf-8')
        _schema = _vs.parse_schema(_schema_content)
        if _schema is None:
            log_error("SCHEMA.md has no YAML frontmatter")
        else:
            _s_errors, _s_warnings = _vs.validate_schema(_schema)
            for _e in _s_errors:
                log_error(f"SCHEMA.md: {_e}")
            for _w in _s_warnings:
                log_info(f"SCHEMA.md: {_w}")
```

**Note:** Using `importlib` avoids sys.path manipulation and is consistent with the existing validate_garden.py pattern of self-contained execution. Read the file carefully to find `log_error`, `log_info`, and `GARDEN_MD` — use the exact names already in the file.

- [x] **Step 5: Run all validate_garden tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_validate_garden.py -v
```

Expected: All tests PASS including the 5 new `TestSchemaValidation` tests.

- [x] **Step 6: Run full suite — no regressions**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 260+ tests, 0 failures.

- [x] **Step 7: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/validate_garden.py tests/test_validate_garden.py tests/garden_fixture.py
git commit -m "feat(validate): check SCHEMA.md federation config when present in garden

Refs #12"
```

---

## Task 3: init_garden.py (TDD)

**Files:**
- Create: `soredium/scripts/init_garden.py`
- Create: `soredium/tests/test_init_garden.py`

- [x] **Step 1: Write failing tests**

Create `soredium/tests/test_init_garden.py`:

```python
#!/usr/bin/env python3
"""Unit, integration, and CLI tests for init_garden.py."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from init_garden import (
    create_garden_md, create_schema_md, create_checked_md,
    create_discarded_md, create_domain, create_ci_workflow, init_garden,
)
from validate_schema import parse_schema, validate_schema

INIT = Path(__file__).parent.parent / 'scripts' / 'init_garden.py'
SCHEMA_V = Path(__file__).parent.parent / 'scripts' / 'validate_schema.py'


def run_init(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(INIT)] + list(args),
        capture_output=True, text=True
    )


def run_schema_validator(garden: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(SCHEMA_V), str(garden)],
        capture_output=True, text=True
    )


# ── create_garden_md ──────────────────────────────────────────────────────────

class TestCreateGardenMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_file(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        self.assertTrue((self.root / 'GARDEN.md').exists())

    def test_contains_last_assigned_id(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('Last assigned ID', content)

    def test_contains_dedupe_sweep_field(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('Last full DEDUPE sweep', content)

    def test_drift_counter_starts_at_zero(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('Entries merged since last sweep: 0', content)

    def test_contains_drift_threshold(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('Drift threshold', content)

    def test_contains_by_technology_section(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('## By Technology', content)

    def test_contains_by_symptom_section(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('## By Symptom', content)

    def test_contains_by_label_section(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        content = (self.root / 'GARDEN.md').read_text()
        self.assertIn('## By Label', content)

    def test_idempotent_does_not_overwrite(self):
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        # Manually mutate
        p = self.root / 'GARDEN.md'
        p.write_text(p.read_text() + '\n<!-- sentinel -->\n')
        create_garden_md(self.root, name='jvm-garden', ge_prefix='JE-')
        self.assertIn('sentinel', p.read_text())


# ── create_schema_md ──────────────────────────────────────────────────────────

class TestCreateSchemaMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_file(self):
        create_schema_md(self.root, name='jvm-garden', description='JVM garden',
                         role='canonical', ge_prefix='JE-', domains=['java', 'quarkus'])
        self.assertTrue((self.root / 'SCHEMA.md').exists())

    def test_created_schema_validates(self):
        create_schema_md(self.root, name='jvm-garden', description='JVM garden',
                         role='canonical', ge_prefix='JE-', domains=['java', 'quarkus'])
        content = (self.root / 'SCHEMA.md').read_text()
        schema = parse_schema(content)
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [], f"Created SCHEMA.md has errors: {errors}")

    def test_canonical_has_no_upstream(self):
        create_schema_md(self.root, name='jvm-garden', description='JVM garden',
                         role='canonical', ge_prefix='JE-', domains=['java'])
        content = (self.root / 'SCHEMA.md').read_text()
        self.assertNotIn('upstream', content)

    def test_child_includes_upstream(self):
        create_schema_md(self.root, name='my-garden', description='Private',
                         role='child', ge_prefix='ME-', domains=['java'],
                         upstream=['https://github.com/Hortora/jvm-garden'])
        content = (self.root / 'SCHEMA.md').read_text()
        self.assertIn('upstream', content)
        self.assertIn('https://github.com/Hortora/jvm-garden', content)
        schema = parse_schema(content)
        errors, _ = validate_schema(schema)
        self.assertEqual(errors, [])

    def test_domains_appear_in_schema(self):
        create_schema_md(self.root, name='jvm-garden', description='JVM',
                         role='canonical', ge_prefix='JE-',
                         domains=['java', 'quarkus', 'spring'])
        schema = parse_schema((self.root / 'SCHEMA.md').read_text())
        self.assertEqual(sorted(schema['domains']), ['java', 'quarkus', 'spring'])

    def test_idempotent_does_not_overwrite(self):
        create_schema_md(self.root, name='jvm-garden', description='JVM garden',
                         role='canonical', ge_prefix='JE-', domains=['java'])
        p = self.root / 'SCHEMA.md'
        p.write_text(p.read_text() + '\n# sentinel\n')
        create_schema_md(self.root, name='jvm-garden', description='JVM garden',
                         role='canonical', ge_prefix='JE-', domains=['java'])
        self.assertIn('sentinel', p.read_text())


# ── create_checked_md ─────────────────────────────────────────────────────────

class TestCreateCheckedMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_file(self):
        create_checked_md(self.root)
        self.assertTrue((self.root / 'CHECKED.md').exists())

    def test_has_table_header(self):
        create_checked_md(self.root)
        content = (self.root / 'CHECKED.md').read_text()
        self.assertIn('| Pair |', content)
        self.assertIn('| Result |', content)

    def test_no_data_rows(self):
        create_checked_md(self.root)
        content = (self.root / 'CHECKED.md').read_text()
        data_rows = [
            l for l in content.splitlines()
            if l.startswith('|') and '---' not in l and 'Pair' not in l
        ]
        self.assertEqual(data_rows, [])


# ── create_discarded_md ───────────────────────────────────────────────────────

class TestCreateDiscardedMd(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_file(self):
        create_discarded_md(self.root)
        self.assertTrue((self.root / 'DISCARDED.md').exists())

    def test_has_table_header(self):
        create_discarded_md(self.root)
        content = (self.root / 'DISCARDED.md').read_text()
        self.assertIn('| Discarded |', content)
        self.assertIn('| Conflicts With |', content)


# ── create_domain ─────────────────────────────────────────────────────────────

class TestCreateDomain(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_directory(self):
        create_domain(self.root, 'java')
        self.assertTrue((self.root / 'java').is_dir())

    def test_creates_index_md(self):
        create_domain(self.root, 'java')
        self.assertTrue((self.root / 'java' / 'INDEX.md').exists())

    def test_index_md_has_table_header(self):
        create_domain(self.root, 'java')
        content = (self.root / 'java' / 'INDEX.md').read_text()
        self.assertIn('| GE-ID |', content)
        self.assertIn('| Title |', content)

    def test_multiple_domains_independent(self):
        create_domain(self.root, 'java')
        create_domain(self.root, 'quarkus')
        self.assertTrue((self.root / 'java' / 'INDEX.md').exists())
        self.assertTrue((self.root / 'quarkus' / 'INDEX.md').exists())

    def test_idempotent(self):
        create_domain(self.root, 'java')
        idx = self.root / 'java' / 'INDEX.md'
        idx.write_text(idx.read_text() + '\n<!-- sentinel -->\n')
        create_domain(self.root, 'java')
        self.assertIn('sentinel', idx.read_text())


# ── create_ci_workflow ────────────────────────────────────────────────────────

class TestCreateCiWorkflow(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_workflow_file(self):
        create_ci_workflow(self.root)
        self.assertTrue((self.root / '.github' / 'workflows' / 'validate_pr.yml').exists())

    def test_workflow_references_validate_pr(self):
        create_ci_workflow(self.root)
        content = (self.root / '.github' / 'workflows' / 'validate_pr.yml').read_text()
        self.assertIn('validate_pr.py', content)

    def test_workflow_triggers_on_pull_request(self):
        create_ci_workflow(self.root)
        content = (self.root / '.github' / 'workflows' / 'validate_pr.yml').read_text()
        self.assertIn('pull_request', content)

    def test_workflow_has_required_yaml_keys(self):
        create_ci_workflow(self.root)
        content = (self.root / '.github' / 'workflows' / 'validate_pr.yml').read_text()
        self.assertIn('on:', content)
        self.assertIn('jobs:', content)


# ── init_garden orchestrator ──────────────────────────────────────────────────

class TestInitGarden(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_creates_all_required_files(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java', 'quarkus'])
        self.assertTrue((self.root / 'GARDEN.md').exists())
        self.assertTrue((self.root / 'SCHEMA.md').exists())
        self.assertTrue((self.root / 'CHECKED.md').exists())
        self.assertTrue((self.root / 'DISCARDED.md').exists())
        self.assertTrue((self.root / '.github' / 'workflows' / 'validate_pr.yml').exists())

    def test_creates_all_domain_directories(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java', 'quarkus', 'spring'])
        for domain in ['java', 'quarkus', 'spring']:
            self.assertTrue((self.root / domain / 'INDEX.md').exists(),
                            f"Missing {domain}/INDEX.md")

    def test_idempotent_preserves_existing_entries(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        (self.root / 'java' / 'GE-0001.md').write_text('# Custom entry\n')
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java'])
        self.assertTrue((self.root / 'java' / 'GE-0001.md').exists())

    def test_created_schema_passes_validate_schema(self):
        init_garden(self.root, name='jvm-garden', description='JVM garden',
                    role='canonical', ge_prefix='JE-', domains=['java', 'quarkus'])
        result = run_schema_validator(self.root)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_child_garden_includes_upstream(self):
        init_garden(self.root, name='my-garden', description='Private',
                    role='child', ge_prefix='ME-', domains=['java'],
                    upstream=['https://github.com/Hortora/jvm-garden'])
        content = (self.root / 'SCHEMA.md').read_text()
        self.assertIn('https://github.com/Hortora/jvm-garden', content)
        result = run_schema_validator(self.root)
        self.assertEqual(result.returncode, 0)

    def test_returns_list_of_created_paths(self):
        created = init_garden(self.root, name='jvm-garden', description='JVM',
                              role='canonical', ge_prefix='JE-', domains=['java'])
        self.assertIsInstance(created, list)
        self.assertGreater(len(created), 0)
        self.assertTrue(any('GARDEN.md' in p for p in created))
        self.assertTrue(any('SCHEMA.md' in p for p in created))


# ── CLI tests ─────────────────────────────────────────────────────────────────

class TestInitGardenCLI(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name) / 'new-garden'

    def tearDown(self):
        self.tmp.cleanup()

    def test_basic_canonical_garden(self):
        result = run_init(
            str(self.root),
            '--name', 'jvm-garden',
            '--description', 'JVM garden',
            '--role', 'canonical',
            '--ge-prefix', 'JE-',
            '--domains', 'java', 'quarkus',
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)
        self.assertTrue((self.root / 'GARDEN.md').exists())
        self.assertTrue((self.root / 'SCHEMA.md').exists())

    def test_child_garden_with_upstream(self):
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
        content = (self.root / 'SCHEMA.md').read_text()
        self.assertIn('https://github.com/Hortora/jvm-garden', content)

    def test_missing_required_arg_exits_nonzero(self):
        result = run_init(str(self.root), '--role', 'canonical')
        self.assertNotEqual(result.returncode, 0)

    def test_output_confirms_created_files(self):
        result = run_init(
            str(self.root),
            '--name', 'test-garden',
            '--description', 'Test',
            '--role', 'canonical',
            '--ge-prefix', 'TE-',
            '--domains', 'tools',
        )
        self.assertEqual(result.returncode, 0)
        self.assertIn('GARDEN.md', result.stdout)
        self.assertIn('SCHEMA.md', result.stdout)

    def test_second_run_reports_nothing_created(self):
        args = [
            str(self.root), '--name', 'test-garden', '--description', 'Test',
            '--role', 'canonical', '--ge-prefix', 'TE-', '--domains', 'tools',
        ]
        run_init(*args)
        result = run_init(*args)
        self.assertEqual(result.returncode, 0)
        self.assertIn('nothing created', result.stdout.lower())


if __name__ == '__main__':
    unittest.main(verbosity=2)
```

- [x] **Step 2: Run — verify they FAIL**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_init_garden.py -v 2>&1 | head -10
```

Expected: `ModuleNotFoundError: No module named 'init_garden'`

- [x] **Step 3: Create init_garden.py**

Create `soredium/scripts/init_garden.py`:

```python
#!/usr/bin/env python3
"""init_garden.py — Initialize a new Hortora knowledge garden.

Creates GARDEN.md, SCHEMA.md, CHECKED.md, DISCARDED.md, domain directories
with INDEX.md, and .github/workflows/validate_pr.yml.

Usage:
  init_garden.py <garden_path> --name NAME --description DESC \\
    --role canonical|child|peer --ge-prefix PREFIX \\
    --domains DOMAIN [DOMAIN ...] \\
    [--upstream URL [URL ...]]
"""

import argparse
import sys
from datetime import date
from pathlib import Path

TODAY = date.today().isoformat()


def create_garden_md(root: Path, name: str, ge_prefix: str) -> None:
    path = root / 'GARDEN.md'
    if path.exists():
        return
    path.write_text(
        f'# {name}\n\n'
        f'**Last assigned ID:** GE-0000\n'
        f'**Last full DEDUPE sweep:** {TODAY}\n'
        f'**Entries merged since last sweep:** 0\n'
        f'**Drift threshold:** 10\n'
        f'**Last staleness review:** never\n'
        f'\n'
        f'## By Technology\n\n---\n\n'
        f'## By Symptom / Type\n\n---\n\n'
        f'## By Label\n\n---\n',
        encoding='utf-8',
    )


def create_schema_md(root: Path, name: str, description: str, role: str,
                     ge_prefix: str, domains: list, upstream: list = None) -> None:
    path = root / 'SCHEMA.md'
    if path.exists():
        return
    lines = [
        '---',
        f'name: {name}',
        f'description: "{description}"',
        f'role: {role}',
        f'ge_prefix: {ge_prefix}',
        'schema_version: "1.0"',
        f'domains: [{", ".join(domains)}]',
    ]
    if upstream:
        lines.append('upstream:')
        for url in upstream:
            lines.append(f'  - {url}')
    lines.extend(['---', ''])
    path.write_text('\n'.join(lines), encoding='utf-8')


def create_checked_md(root: Path) -> None:
    path = root / 'CHECKED.md'
    if path.exists():
        return
    path.write_text(
        '# Garden Duplicate Check Log\n\n'
        '| Pair | Result | Date | Notes |\n'
        '|------|--------|------|-------|\n',
        encoding='utf-8',
    )


def create_discarded_md(root: Path) -> None:
    path = root / 'DISCARDED.md'
    if path.exists():
        return
    path.write_text(
        '# Discarded Submissions\n\n'
        '| Discarded | Conflicts With | Date | Reason |\n'
        '|-----------|----------------|------|--------|\n',
        encoding='utf-8',
    )


def create_domain(root: Path, domain: str) -> None:
    domain_dir = root / domain
    domain_dir.mkdir(parents=True, exist_ok=True)
    index = domain_dir / 'INDEX.md'
    if index.exists():
        return
    index.write_text(
        f'# {domain.capitalize()} Index\n\n'
        '| GE-ID | Title | Type | Score | Submitted |\n'
        '|-------|-------|------|-------|-----------|\n',
        encoding='utf-8',
    )


def create_ci_workflow(root: Path) -> None:
    workflows_dir = root / '.github' / 'workflows'
    workflows_dir.mkdir(parents=True, exist_ok=True)
    workflow = workflows_dir / 'validate_pr.yml'
    if workflow.exists():
        return
    workflow.write_text(
        'name: Validate Garden Entry PR\n'
        '\n'
        'on:\n'
        '  pull_request:\n'
        '    branches: [main]\n'
        '\n'
        'jobs:\n'
        '  validate:\n'
        '    runs-on: ubuntu-latest\n'
        '    steps:\n'
        '      - uses: actions/checkout@v4\n'
        '        with:\n'
        '          fetch-depth: 0\n'
        '\n'
        '      - name: Set up Python\n'
        '        uses: actions/setup-python@v5\n'
        '        with:\n'
        '          python-version: "3.11"\n'
        '\n'
        '      - name: Validate PR\n'
        '        run: |\n'
        '          curl -sSL https://raw.githubusercontent.com/Hortora/soredium/main/scripts/validate_pr.py \\\n'
        '            -o validate_pr.py\n'
        '          python3 validate_pr.py\n',
        encoding='utf-8',
    )


def init_garden(root: Path, name: str, description: str, role: str,
                ge_prefix: str, domains: list, upstream: list = None) -> list:
    """Initialize a garden. Idempotent — skips files that already exist.
    Returns list of created file paths relative to root."""
    root.mkdir(parents=True, exist_ok=True)
    created = []

    for fn, args, kwargs in [
        (create_garden_md,   [root, name, ge_prefix], {}),
        (create_schema_md,   [root, name, description, role, ge_prefix, domains],
                             {'upstream': upstream}),
        (create_checked_md,  [root], {}),
        (create_discarded_md,[root], {}),
        (create_ci_workflow, [root], {}),
    ]:
        before = set(root.rglob('*'))
        fn(*args, **kwargs)
        after = set(root.rglob('*'))
        created.extend(str(p.relative_to(root)) for p in sorted(after - before))

    for domain in domains:
        before = set(root.rglob('*'))
        create_domain(root, domain)
        after = set(root.rglob('*'))
        created.extend(str(p.relative_to(root)) for p in sorted(after - before))

    return created


def main():
    parser = argparse.ArgumentParser(
        description=__doc__,
        formatter_class=argparse.RawDescriptionHelpFormatter
    )
    parser.add_argument('garden_path')
    parser.add_argument('--name', required=True)
    parser.add_argument('--description', required=True)
    parser.add_argument('--role', required=True, choices=['canonical', 'child', 'peer'])
    parser.add_argument('--ge-prefix', required=True, dest='ge_prefix')
    parser.add_argument('--domains', required=True, nargs='+')
    parser.add_argument('--upstream', nargs='+', default=None)

    args = parser.parse_args()
    root = Path(args.garden_path).expanduser().resolve()

    created = init_garden(
        root=root, name=args.name, description=args.description,
        role=args.role, ge_prefix=args.ge_prefix,
        domains=args.domains, upstream=args.upstream,
    )

    if created:
        print(f"Initialized garden at {root}:")
        for f in created:
            print(f"  Created: {f}")
    else:
        print(f"Garden already exists at {root} — nothing created.")

    print(f"\nNext: cd {root} && git init && git add . && git commit -m 'init: {args.name}'")
    sys.exit(0)


if __name__ == '__main__':
    main()
```

- [x] **Step 4: Run all init_garden tests — verify they PASS**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_init_garden.py -v
```

Expected: All tests PASS.

- [x] **Step 5: Run full suite — no regressions**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [x] **Step 6: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/init_garden.py tests/test_init_garden.py
git commit -m "feat(init): add init_garden.py with full test suite — canonical garden initializer

Refs #12"
```

---

## Task 4: E2E pipeline test

**Files:**
- Modify: `soredium/tests/test_integration.py`

Full pipeline: `init_garden` → validate schema → validate garden → add entry on PR branch → `validate_pr`.

- [x] **Step 1: Read test_integration.py first**

Read `soredium/tests/test_integration.py` to find: existing imports, existing path constants (`VALIDATE_PR`, `VALIDATOR`, etc.), the `textwrap` import, and `subprocess`/`sys` imports. Add only what is missing.

- [x] **Step 2: Write failing E2E tests**

Add to the END of `soredium/tests/test_integration.py`:

```python
# ── Phase 5 E2E: init_garden pipeline ─────────────────────────────────────────

import textwrap as _tw

_INIT    = Path(__file__).parent.parent / 'scripts' / 'init_garden.py'
_SCHEMA_V = Path(__file__).parent.parent / 'scripts' / 'validate_schema.py'


def _run_init_garden(*args) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(_INIT)] + list(args),
        capture_output=True, text=True
    )


def _run_schema_validator(garden: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, str(_SCHEMA_V), str(garden)],
        capture_output=True, text=True
    )


_E2E_ENTRY = _tw.dedent("""\
    ---
    id: GE-20260414-e2e001
    title: "E2E test entry for init_garden pipeline"
    type: gotcha
    domain: java
    stack: "Java (all versions)"
    tags: [java, testing]
    score: 10
    verified: true
    staleness_threshold: 730
    submitted: 2026-04-14
    ---

    ## E2E test entry for init_garden pipeline

    **ID:** GE-20260414-e2e001
    **Stack:** Java (all versions)
    **Symptom:** Test symptom.
    **Context:** E2E test.

    ### What was tried (didn't work)
    - tried X — failed

    ### Root cause
    E2E root cause.

    ### Fix
    E2E fix.

    ### Why this is non-obvious
    Used in E2E tests.

    *Score: 10/15 · Included because: test coverage · Reservation: none*
""")


class TestE2EInitGardenPipeline(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.garden = Path(self.tmp.name) / 'test-jvm-garden'

    def tearDown(self):
        self.tmp.cleanup()

    def _init(self, role='canonical', ge_prefix='JE-',
              domains=None, upstream=None):
        domains = domains or ['java', 'quarkus']
        args = [
            str(self.garden),
            '--name', 'test-jvm-garden',
            '--description', 'Test JVM garden',
            '--role', role,
            '--ge-prefix', ge_prefix,
            '--domains', *domains,
        ]
        if upstream:
            args += ['--upstream', *upstream]
        return _run_init_garden(*args)

    def _git_init(self):
        for cmd in [
            ['git', 'init', str(self.garden)],
            ['git', '-C', str(self.garden), 'config', 'user.email', 'test@test.com'],
            ['git', '-C', str(self.garden), 'config', 'user.name', 'Test'],
            ['git', '-C', str(self.garden), 'add', '.'],
            ['git', '-C', str(self.garden), 'commit', '-m', 'init'],
        ]:
            subprocess.run(cmd, check=True, capture_output=True)

    def test_init_exits_0(self):
        result = self._init()
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_schema_validates_after_init(self):
        self._init()
        result = _run_schema_validator(self.garden)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_garden_structure_complete_after_init(self):
        self._init()
        for path in [
            'GARDEN.md', 'SCHEMA.md', 'CHECKED.md', 'DISCARDED.md',
            '.github/workflows/validate_pr.yml',
            'java/INDEX.md', 'quarkus/INDEX.md',
        ]:
            self.assertTrue((self.garden / path).exists(), f"Missing {path}")

    def test_validate_pr_accepts_entry_on_pr_branch(self):
        """Full pipeline: init → git init → add entry on branch → validate_pr passes."""
        self._init()
        self._git_init()

        # Add entry on a PR branch
        subprocess.run(
            ['git', '-C', str(self.garden), 'checkout', '-b', 'submit/GE-20260414-e2e001'],
            check=True, capture_output=True
        )
        entry_path = self.garden / 'java' / 'GE-20260414-e2e001.md'
        entry_path.write_text(_E2E_ENTRY)
        subprocess.run(['git', '-C', str(self.garden), 'add', '.'],
                       check=True, capture_output=True)
        subprocess.run(
            ['git', '-C', str(self.garden), 'commit', '-m', 'submit: e2e test entry'],
            check=True, capture_output=True
        )

        # validate_pr.py must be run from the garden directory
        VALIDATE_PR_SCRIPT = Path(__file__).parent.parent / 'scripts' / 'validate_pr.py'
        result = subprocess.run(
            [sys.executable, str(VALIDATE_PR_SCRIPT), str(self.garden)],
            capture_output=True, text=True, cwd=str(self.garden)
        )
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_child_garden_schema_validates(self):
        self._init(
            role='child', ge_prefix='ME-',
            domains=['java'],
            upstream=['https://github.com/Hortora/jvm-garden'],
        )
        result = _run_schema_validator(self.garden)
        self.assertEqual(result.returncode, 0, result.stdout + result.stderr)

    def test_idempotent_second_init_does_not_corrupt(self):
        self._init()
        self._init()  # second run — should be no-op
        result = _run_schema_validator(self.garden)
        self.assertEqual(result.returncode, 0)
```

- [x] **Step 3: Run E2E tests**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_integration.py::TestE2EInitGardenPipeline -v
```

Expected: All 6 tests PASS.

- [x] **Step 4: Run full suite**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/ --tb=short 2>&1 | tail -5
```

Expected: 0 failures.

- [x] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add tests/test_integration.py
git commit -m "test(e2e): add E2E pipeline tests for Phase 5 — init→validate→submit

Refs #12"
```

---

## Self-Review

**Spec coverage:**

| Phase 5 requirement | Task |
|---|---|
| SCHEMA.md format defined and validated | Task 1 |
| Role (canonical/child/peer) enforced | Task 1 |
| ge_prefix format enforced (1–3 uppercase + hyphen) | Task 1 |
| Upstream required for child, forbidden for canonical/peer | Task 1 |
| Integration + CLI tests for validate_schema | Task 1 |
| validate_garden.py checks SCHEMA.md when present | Task 2 |
| GardenFixture.schema_md() builder | Task 2 |
| Garden initializer creates all required files | Task 3 |
| Idempotent init (does not overwrite existing files) | Task 3 |
| CI workflow template | Task 3 |
| CLI for init_garden with all arguments | Task 3 |
| E2E: init → schema validate → garden validate → PR validate | Task 4 |

**No gaps. No placeholders.**

**Type consistency:** `parse_schema()` → `dict | None`. `validate_schema()` → `tuple[list[str], list[str]]`. All `validate_*()` functions → `list[str]`. `init_garden()` → `list[str]`. Consistent throughout.
