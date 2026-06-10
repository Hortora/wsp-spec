# Area 2 Phase 4 — Human Validation Gate + End-to-End Pipeline Runner

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the pattern discovery loop with a human validation gate that accepts/rejects cluster candidates and an end-to-end pipeline runner that orchestrates the full Phase 3 stack.

**Architecture:** Candidates flow through `run_pipeline.py` (registry → extract → cluster → delta → JSON report), then through `validate_candidates.py` (accept → patterns-garden YAML skeleton, reject → `known_rejections.yaml`, skip → unchanged). The rejection registry feeds back into the clustering step to suppress already-reviewed noise. All modules are pure functions with injectable I/O so every layer is testable without subprocess or filesystem side-effects.

**Tech Stack:** Python 3.11+, PyYAML, json (stdlib), unittest, subprocess (git, for pipeline E2E only)

---

## Existing interfaces (Phase 3 — do not modify)

```python
# scripts/project_registry.py
class ProjectRegistry:
    def list() -> list[dict]        # each dict has: project, url, domain, primary_language,
                                    #   frameworks, last_processed_commit, notable_contributors
    def get(name: str) -> dict | None
    def update_commit(name: str, commit: str) -> None

# scripts/feature_extractor.py
def extract_features(root: Path) -> dict
# returns: {interface_count, abstraction_depth, injection_points,
#           extension_signatures, file_count, spi_patterns}

# scripts/cluster_pipeline.py
FEATURE_KEYS: list[str]   # ordered list of feature names
def cluster_projects(fingerprints: dict[str,dict], known_patterns: list[dict],
                     similarity_threshold: float = 0.75) -> list[dict]
# each result dict: {projects: list[str], centroid: dict, similarity_score: float,
#                    matches_known_pattern: str|None}

# scripts/delta_analysis.py
def get_major_version_tags(repo: Path) -> list[str]
def delta_analysis(repo: Path, from_ref: str, to_ref: str) -> list[dict]
# each result dict: {file, kind, introduced_at, commit, author, date}
```

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `known_rejections.yaml` | Create | Seed file for rejected cluster fingerprints |
| `scripts/rejection_registry.py` | Create | Add/check/list known rejections |
| `scripts/candidate_report.py` | Create | Serialize/deserialize cluster+delta candidates to/from JSON |
| `scripts/pattern_entry.py` | Create | Generate patterns-garden YAML entry skeleton from accepted candidate |
| `scripts/validate_candidates.py` | Create | Validation gate — accepts decision callback, returns session summary |
| `scripts/run_pipeline.py` | Create | Orchestrate: registry → extract → cluster → delta → write report |
| `tests/test_rejection_registry.py` | Create | Unit + correctness tests |
| `tests/test_candidate_report.py` | Create | Unit + correctness tests |
| `tests/test_pattern_entry.py` | Create | Unit + correctness tests |
| `tests/test_validate_candidates.py` | Create | Unit + correctness + integration tests |
| `tests/test_run_pipeline.py` | Create | Integration + E2E + happy path tests |

---

## Data formats (shared contract between all modules)

### Cluster candidate (from `cluster_projects`)
```python
{
    'projects': ['proj-a', 'proj-b'],
    'centroid': {'interface_count': 11.5, 'abstraction_depth': 0.3, ...},
    'similarity_score': 0.97,
    'matches_known_pattern': None,   # str name or None
}
```

### Delta candidate (from `delta_analysis`)
```python
{
    'file': 'src/Evaluator.java',
    'kind': 'interface',             # or 'abstract_class'
    'introduced_at': 'v2.0',
    'commit': 'abc1234',
    'author': 'dev@example.com',
    'date': '2026-04-21',
}
```

### Candidate report (JSON file)
```json
{
    "generated_at": "2026-04-21T12:00:00",
    "cluster_candidates": [ <cluster candidate dicts> ],
    "delta_candidates": [ <delta candidate dicts> ]
}
```

### Rejection record (in `known_rejections.yaml`)
```yaml
rejections:
  - centroid: {interface_count: 20.0, abstraction_depth: 0.6, ...}
    projects: [proj-a, proj-b]
    reason: "High interface count from test doubles, not a real pattern"
    rejected_at: "2026-04-21"
```

### Patterns-garden entry skeleton (YAML file)
```yaml
---
id: GP-20260421-ab1234
garden: patterns
title: ""
type: architectural
domain: jvm
tags: []
score: 10
verified: false
staleness_threshold: 730
submitted: 2026-04-21
observed_in:
  - project: proj-a
    url: ""
  - project: proj-b
    url: ""
---

### Pattern description
<!-- Describe the architectural pattern -->

### When to use
<!-- When is this pattern appropriate? -->

### When not to use
<!-- When should you avoid this pattern? -->

### Variants
<!-- Known variants of this pattern -->
```

---

## Task 1: Rejection registry

**Files:**
- Create: `known_rejections.yaml`
- Create: `scripts/rejection_registry.py`
- Create: `tests/test_rejection_registry.py`

- [ ] **Step 1: Create seed file**

`known_rejections.yaml`:
```yaml
rejections: []
```

- [ ] **Step 2: Write failing tests (unit + correctness)**

`tests/test_rejection_registry.py`:
```python
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from rejection_registry import RejectionRegistry


_CENTROID_A = {'interface_count': 20.0, 'abstraction_depth': 0.6,
               'injection_points': 15.0, 'extension_signatures': 18.0,
               'file_count': 33.0, 'spi_patterns': 4.0}
_CENTROID_B = {'interface_count': 1.0, 'abstraction_depth': 0.02,
               'injection_points': 2.0, 'extension_signatures': 1.0,
               'file_count': 50.0, 'spi_patterns': 0.0}


class TestRejectionRegistryUnit(unittest.TestCase):
    """Unit tests — RejectionRegistry in isolation."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.path = Path(self.tmp.name) / 'rejections.yaml'
        self.path.write_text('rejections: []\n')
        self.reg = RejectionRegistry(self.path)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_registry_has_no_rejections(self):
        self.assertEqual(self.reg.list(), [])

    def test_add_rejection_persists(self):
        self.reg.add(_CENTROID_A, ['proj-a', 'proj-b'], 'test doubles')
        records = self.reg.list()
        self.assertEqual(len(records), 1)
        self.assertEqual(records[0]['projects'], ['proj-a', 'proj-b'])
        self.assertEqual(records[0]['reason'], 'test doubles')

    def test_add_rejection_stores_centroid(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        self.assertEqual(self.reg.list()[0]['centroid'], _CENTROID_A)

    def test_add_rejection_records_date(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        self.assertIn('rejected_at', self.reg.list()[0])

    def test_data_persists_across_instances(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        reload = RejectionRegistry(self.path)
        self.assertEqual(len(reload.list()), 1)


class TestRejectionRegistryCorrectness(unittest.TestCase):
    """Correctness tests — rejection suppression behaviour."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.path = Path(self.tmp.name) / 'rejections.yaml'
        self.path.write_text('rejections: []\n')
        self.reg = RejectionRegistry(self.path)

    def tearDown(self):
        self.tmp.cleanup()

    def test_is_rejected_returns_false_for_unknown_centroid(self):
        self.assertFalse(self.reg.is_rejected(_CENTROID_A))

    def test_is_rejected_returns_true_after_add(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        self.assertTrue(self.reg.is_rejected(_CENTROID_A))

    def test_is_rejected_uses_similarity_not_exact_match(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        # Near-identical centroid — should still match
        near = dict(_CENTROID_A)
        near['interface_count'] = 20.1
        self.assertTrue(self.reg.is_rejected(near))

    def test_dissimilar_centroid_not_rejected(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        self.assertFalse(self.reg.is_rejected(_CENTROID_B))

    def test_multiple_rejections_all_checked(self):
        self.reg.add(_CENTROID_A, ['proj-a'], 'noise')
        self.reg.add(_CENTROID_B, ['proj-c'], 'noise2')
        self.assertTrue(self.reg.is_rejected(_CENTROID_A))
        self.assertTrue(self.reg.is_rejected(_CENTROID_B))
```

- [ ] **Step 3: Run tests — verify they fail**

```bash
cd /Users/mdproctor/claude/hortora/soredium-area2-phase4 && python3 -m pytest tests/test_rejection_registry.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'rejection_registry'`

- [ ] **Step 4: Implement `scripts/rejection_registry.py`**

```python
"""Registry of rejected pattern candidates — prevents re-surfacing known noise."""
import math
from datetime import date
from pathlib import Path
import yaml

from cluster_pipeline import FEATURE_KEYS, fingerprint_to_vector

_REJECTION_SIMILARITY_THRESHOLD = 0.98


def _cosine_similarity(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    mag_a = math.sqrt(sum(x * x for x in a))
    mag_b = math.sqrt(sum(x * x for x in b))
    if mag_a == 0 or mag_b == 0:
        return 0.0
    return dot / (mag_a * mag_b)


class RejectionRegistry:
    def __init__(self, path: Path):
        self.path = Path(path)

    def _load(self) -> list:
        data = yaml.safe_load(self.path.read_text()) or {}
        return data.get('rejections', [])

    def _save(self, rejections: list) -> None:
        self.path.write_text(
            yaml.dump({'rejections': rejections}, default_flow_style=False, sort_keys=False)
        )

    def list(self) -> list:
        return self._load()

    def add(self, centroid: dict, projects: list[str], reason: str) -> None:
        rejections = self._load()
        rejections.append({
            'centroid': centroid,
            'projects': projects,
            'reason': reason,
            'rejected_at': str(date.today()),
        })
        self._save(rejections)

    def is_rejected(self, centroid: dict) -> bool:
        vec = fingerprint_to_vector(centroid)
        for record in self._load():
            known_vec = fingerprint_to_vector(record['centroid'])
            if _cosine_similarity(vec, known_vec) >= _REJECTION_SIMILARITY_THRESHOLD:
                return True
        return False
```

- [ ] **Step 5: Run tests — verify all 10 pass**

```bash
python3 -m pytest tests/test_rejection_registry.py -v
```
Expected: 10 tests pass.

- [ ] **Step 6: Commit**

```bash
git add known_rejections.yaml scripts/rejection_registry.py tests/test_rejection_registry.py
git commit -m "feat(gate): rejection registry — suppress re-surfacing of known noise

Refs #35"
```

---

## Task 2: Candidate report serialization

**Files:**
- Create: `scripts/candidate_report.py`
- Create: `tests/test_candidate_report.py`

- [ ] **Step 1: Write failing tests (unit + correctness)**

`tests/test_candidate_report.py`:
```python
import json
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory
from datetime import datetime

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from candidate_report import CandidateReport, load_report, save_report


_CLUSTER = {
    'projects': ['proj-a', 'proj-b'],
    'centroid': {'interface_count': 11.5, 'abstraction_depth': 0.3,
                 'injection_points': 7.0, 'extension_signatures': 8.0,
                 'file_count': 31.5, 'spi_patterns': 1.5},
    'similarity_score': 0.97,
    'matches_known_pattern': None,
}
_DELTA = {
    'file': 'src/Evaluator.java',
    'kind': 'interface',
    'introduced_at': 'v2.0',
    'commit': 'abc1234',
    'author': 'dev@example.com',
    'date': '2026-04-21',
}


class TestCandidateReportUnit(unittest.TestCase):
    """Unit tests — CandidateReport construction."""

    def test_empty_report_has_no_candidates(self):
        report = CandidateReport(cluster_candidates=[], delta_candidates=[])
        self.assertEqual(report.cluster_candidates, [])
        self.assertEqual(report.delta_candidates, [])

    def test_report_stores_candidates(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER], delta_candidates=[_DELTA])
        self.assertEqual(len(report.cluster_candidates), 1)
        self.assertEqual(len(report.delta_candidates), 1)

    def test_report_has_generated_at_timestamp(self):
        report = CandidateReport(cluster_candidates=[], delta_candidates=[])
        self.assertIsInstance(report.generated_at, str)
        # Should parse as ISO datetime
        datetime.fromisoformat(report.generated_at)

    def test_total_count(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER], delta_candidates=[_DELTA])
        self.assertEqual(report.total_count(), 2)


class TestCandidateReportCorrectness(unittest.TestCase):
    """Correctness tests — round-trip serialization."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.path = Path(self.tmp.name) / 'report.json'

    def tearDown(self):
        self.tmp.cleanup()

    def test_save_creates_valid_json(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER], delta_candidates=[_DELTA])
        save_report(report, self.path)
        with open(self.path) as f:
            data = json.load(f)
        self.assertIn('generated_at', data)
        self.assertIn('cluster_candidates', data)
        self.assertIn('delta_candidates', data)

    def test_load_restores_cluster_candidates(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER], delta_candidates=[])
        save_report(report, self.path)
        loaded = load_report(self.path)
        self.assertEqual(loaded.cluster_candidates[0]['projects'], ['proj-a', 'proj-b'])

    def test_load_restores_delta_candidates(self):
        report = CandidateReport(cluster_candidates=[], delta_candidates=[_DELTA])
        save_report(report, self.path)
        loaded = load_report(self.path)
        self.assertEqual(loaded.delta_candidates[0]['file'], 'src/Evaluator.java')

    def test_round_trip_preserves_all_fields(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER], delta_candidates=[_DELTA])
        save_report(report, self.path)
        loaded = load_report(self.path)
        self.assertEqual(loaded.cluster_candidates, [_CLUSTER])
        self.assertEqual(loaded.delta_candidates, [_DELTA])

    def test_load_nonexistent_raises(self):
        with self.assertRaises(FileNotFoundError):
            load_report(Path('/nonexistent/report.json'))
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_candidate_report.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'candidate_report'`

- [ ] **Step 3: Implement `scripts/candidate_report.py`**

```python
"""Serialize/deserialize cluster+delta candidate reports to/from JSON."""
import json
from dataclasses import dataclass, field
from datetime import datetime
from pathlib import Path


@dataclass
class CandidateReport:
    cluster_candidates: list
    delta_candidates: list
    generated_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())

    def total_count(self) -> int:
        return len(self.cluster_candidates) + len(self.delta_candidates)


def save_report(report: CandidateReport, path: Path) -> None:
    path = Path(path)
    data = {
        'generated_at': report.generated_at,
        'cluster_candidates': report.cluster_candidates,
        'delta_candidates': report.delta_candidates,
    }
    path.write_text(json.dumps(data, indent=2))


def load_report(path: Path) -> CandidateReport:
    path = Path(path)
    if not path.exists():
        raise FileNotFoundError(f"Report not found: {path}")
    data = json.loads(path.read_text())
    return CandidateReport(
        cluster_candidates=data['cluster_candidates'],
        delta_candidates=data['delta_candidates'],
        generated_at=data['generated_at'],
    )
```

- [ ] **Step 4: Run tests — verify all 9 pass**

```bash
python3 -m pytest tests/test_candidate_report.py -v
```
Expected: 9 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/candidate_report.py tests/test_candidate_report.py
git commit -m "feat(gate): candidate report — JSON serialization for cluster+delta candidates

Refs #35"
```

---

## Task 3: Patterns-garden entry skeleton generator

**Files:**
- Create: `scripts/pattern_entry.py`
- Create: `tests/test_pattern_entry.py`

- [ ] **Step 1: Write failing tests (unit + correctness)**

`tests/test_pattern_entry.py`:
```python
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory
import yaml

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from pattern_entry import generate_skeleton, write_skeleton, SKELETON_SECTIONS


_CLUSTER = {
    'projects': ['proj-a', 'proj-b'],
    'centroid': {'interface_count': 11.5, 'abstraction_depth': 0.3,
                 'injection_points': 7.0, 'extension_signatures': 8.0,
                 'file_count': 31.5, 'spi_patterns': 1.5},
    'similarity_score': 0.97,
    'matches_known_pattern': None,
}
_REGISTRY_PROJECTS = [
    {'project': 'proj-a', 'url': 'https://github.com/org/proj-a',
     'domain': 'jvm', 'primary_language': 'java',
     'frameworks': [], 'last_processed_commit': None, 'notable_contributors': []},
    {'project': 'proj-b', 'url': 'https://github.com/org/proj-b',
     'domain': 'jvm', 'primary_language': 'java',
     'frameworks': [], 'last_processed_commit': None, 'notable_contributors': []},
]


class TestPatternEntryUnit(unittest.TestCase):
    """Unit tests — generate_skeleton in isolation."""

    def test_returns_string(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        self.assertIsInstance(skeleton, str)

    def test_contains_yaml_frontmatter_delimiters(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        self.assertTrue(skeleton.startswith('---\n'))
        self.assertIn('\n---\n', skeleton)

    def test_id_has_gp_prefix(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        frontmatter = yaml.safe_load(skeleton.split('---\n')[1])
        self.assertTrue(frontmatter['id'].startswith('GP-'))

    def test_garden_is_patterns(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        frontmatter = yaml.safe_load(skeleton.split('---\n')[1])
        self.assertEqual(frontmatter['garden'], 'patterns')

    def test_observed_in_contains_cluster_projects(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        frontmatter = yaml.safe_load(skeleton.split('---\n')[1])
        observed_projects = [o['project'] for o in frontmatter['observed_in']]
        self.assertIn('proj-a', observed_projects)
        self.assertIn('proj-b', observed_projects)

    def test_observed_in_includes_url_from_registry(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        frontmatter = yaml.safe_load(skeleton.split('---\n')[1])
        proj_a = next(o for o in frontmatter['observed_in'] if o['project'] == 'proj-a')
        self.assertEqual(proj_a['url'], 'https://github.com/org/proj-a')

    def test_skeleton_sections_defined(self):
        self.assertIn('Pattern description', SKELETON_SECTIONS)
        self.assertIn('When to use', SKELETON_SECTIONS)
        self.assertIn('When not to use', SKELETON_SECTIONS)
        self.assertIn('Variants', SKELETON_SECTIONS)

    def test_body_contains_all_sections(self):
        skeleton = generate_skeleton(_CLUSTER, _REGISTRY_PROJECTS)
        for section in SKELETON_SECTIONS:
            self.assertIn(f'### {section}', skeleton)


class TestPatternEntryCorrectness(unittest.TestCase):
    """Correctness tests — file writing and YAML validity."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.out_dir = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_write_creates_file(self):
        path = write_skeleton(_CLUSTER, _REGISTRY_PROJECTS, self.out_dir)
        self.assertTrue(path.exists())

    def test_written_file_is_valid_yaml_frontmatter(self):
        path = write_skeleton(_CLUSTER, _REGISTRY_PROJECTS, self.out_dir)
        content = path.read_text()
        parts = content.split('---\n', 2)
        self.assertEqual(len(parts), 3)   # '', frontmatter, body
        frontmatter = yaml.safe_load(parts[1])
        self.assertIsInstance(frontmatter, dict)

    def test_filename_uses_gp_id(self):
        path = write_skeleton(_CLUSTER, _REGISTRY_PROJECTS, self.out_dir)
        self.assertTrue(path.name.startswith('GP-'))
        self.assertTrue(path.name.endswith('.md'))

    def test_project_not_in_registry_gets_empty_url(self):
        cluster = dict(_CLUSTER)
        cluster['projects'] = ['proj-a', 'unknown-proj']
        skeleton = generate_skeleton(cluster, _REGISTRY_PROJECTS)
        frontmatter = yaml.safe_load(skeleton.split('---\n')[1])
        unknown = next(o for o in frontmatter['observed_in'] if o['project'] == 'unknown-proj')
        self.assertEqual(unknown['url'], '')
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_pattern_entry.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'pattern_entry'`

- [ ] **Step 3: Implement `scripts/pattern_entry.py`**

```python
"""Generate patterns-garden entry skeleton from an accepted cluster candidate."""
import secrets
from datetime import date
from pathlib import Path
import yaml

SKELETON_SECTIONS = [
    'Pattern description',
    'When to use',
    'When not to use',
    'Variants',
]


def _make_id() -> str:
    today = date.today().strftime('%Y%m%d')
    return f"GP-{today}-{secrets.token_hex(3)}"


def generate_skeleton(cluster: dict, registry_projects: list[dict]) -> str:
    project_index = {p['project']: p for p in registry_projects}

    observed_in = []
    for name in cluster['projects']:
        reg = project_index.get(name, {})
        observed_in.append({
            'project': name,
            'url': reg.get('url', ''),
        })

    frontmatter = {
        'id': _make_id(),
        'garden': 'patterns',
        'title': '',
        'type': 'architectural',
        'domain': 'jvm',
        'tags': [],
        'score': 10,
        'verified': False,
        'staleness_threshold': 730,
        'submitted': str(date.today()),
        'observed_in': observed_in,
    }

    body_sections = '\n\n'.join(
        f'### {s}\n<!-- {s} -->' for s in SKELETON_SECTIONS
    )

    return f"---\n{yaml.dump(frontmatter, default_flow_style=False, sort_keys=False)}---\n\n{body_sections}\n"


def write_skeleton(cluster: dict, registry_projects: list[dict], out_dir: Path) -> Path:
    out_dir = Path(out_dir)
    out_dir.mkdir(parents=True, exist_ok=True)
    skeleton = generate_skeleton(cluster, registry_projects)
    # Extract id from generated skeleton
    entry_id = yaml.safe_load(skeleton.split('---\n')[1])['id']
    path = out_dir / f"{entry_id}.md"
    path.write_text(skeleton)
    return path
```

- [ ] **Step 4: Run tests — verify all 12 pass**

```bash
python3 -m pytest tests/test_pattern_entry.py -v
```
Expected: 12 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/pattern_entry.py tests/test_pattern_entry.py
git commit -m "feat(gate): pattern entry skeleton generator — GP-ID, observed_in, section placeholders

Refs #35"
```

---

## Task 4: Validation gate

**Files:**
- Create: `scripts/validate_candidates.py`
- Create: `tests/test_validate_candidates.py`

The gate is designed for testability: `validate_candidates` takes a `decide_fn` callback instead of reading stdin. This makes every path fully unit-testable. The CLI wrapper in `__main__` handles actual stdin.

- [ ] **Step 1: Write failing tests (unit + correctness + integration)**

`tests/test_validate_candidates.py`:
```python
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory
import yaml

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from candidate_report import CandidateReport
from validate_candidates import validate_candidates, Decision, SessionSummary


_CLUSTER_A = {
    'projects': ['proj-a', 'proj-b'],
    'centroid': {'interface_count': 20.0, 'abstraction_depth': 0.6,
                 'injection_points': 15.0, 'extension_signatures': 18.0,
                 'file_count': 33.0, 'spi_patterns': 4.0},
    'similarity_score': 0.97,
    'matches_known_pattern': None,
}
_CLUSTER_B = {
    'projects': ['proj-c', 'proj-d'],
    'centroid': {'interface_count': 5.0, 'abstraction_depth': 0.1,
                 'injection_points': 3.0, 'extension_signatures': 2.0,
                 'file_count': 50.0, 'spi_patterns': 0.0},
    'similarity_score': 0.91,
    'matches_known_pattern': None,
}
_REGISTRY_PROJECTS = [
    {'project': 'proj-a', 'url': 'https://github.com/org/proj-a',
     'domain': 'jvm', 'primary_language': 'java',
     'frameworks': [], 'last_processed_commit': None, 'notable_contributors': []},
    {'project': 'proj-b', 'url': 'https://github.com/org/proj-b',
     'domain': 'jvm', 'primary_language': 'java',
     'frameworks': [], 'last_processed_commit': None, 'notable_contributors': []},
]


class TestDecisionEnum(unittest.TestCase):
    """Unit tests — Decision values."""

    def test_decision_values_defined(self):
        self.assertEqual(Decision.ACCEPT, 'accept')
        self.assertEqual(Decision.REJECT, 'reject')
        self.assertEqual(Decision.SKIP, 'skip')


class TestSessionSummaryUnit(unittest.TestCase):
    """Unit tests — SessionSummary construction."""

    def test_summary_tracks_counts(self):
        s = SessionSummary(accepted=2, rejected=1, skipped=3)
        self.assertEqual(s.accepted, 2)
        self.assertEqual(s.rejected, 1)
        self.assertEqual(s.skipped, 3)

    def test_summary_total(self):
        s = SessionSummary(accepted=2, rejected=1, skipped=3)
        self.assertEqual(s.total(), 6)


class TestValidateCandidatesCorrectness(unittest.TestCase):
    """Correctness tests — accept/reject/skip state transitions."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.out_dir = Path(self.tmp.name) / 'patterns'
        self.rejections_path = Path(self.tmp.name) / 'rejections.yaml'
        self.rejections_path.write_text('rejections: []\n')
        self.report = CandidateReport(
            cluster_candidates=[_CLUSTER_A, _CLUSTER_B],
            delta_candidates=[],
        )

    def tearDown(self):
        self.tmp.cleanup()

    def test_accept_creates_skeleton_file(self):
        # Accept first, reject second
        decisions = iter([
            (Decision.ACCEPT, None),
            (Decision.REJECT, 'noise'),
        ])
        validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: next(decisions),
        )
        files = list(self.out_dir.glob('GP-*.md'))
        self.assertEqual(len(files), 1)

    def test_reject_records_in_rejection_registry(self):
        validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: (Decision.REJECT, 'test doubles'),
        )
        import yaml as _yaml
        data = _yaml.safe_load(self.rejections_path.read_text())
        self.assertEqual(len(data['rejections']), 2)

    def test_skip_creates_no_files_and_no_rejections(self):
        validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: (Decision.SKIP, None),
        )
        self.assertEqual(list(self.out_dir.glob('GP-*.md')), [])
        import yaml as _yaml
        data = _yaml.safe_load(self.rejections_path.read_text())
        self.assertEqual(len(data['rejections']), 0)

    def test_summary_reflects_decisions(self):
        decisions = iter([
            (Decision.ACCEPT, None),
            (Decision.REJECT, 'noise'),
        ])
        summary = validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: next(decisions),
        )
        self.assertEqual(summary.accepted, 1)
        self.assertEqual(summary.rejected, 1)
        self.assertEqual(summary.skipped, 0)

    def test_reject_stores_reason_in_registry(self):
        validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: (Decision.REJECT, 'my reason'),
        )
        import yaml as _yaml
        data = _yaml.safe_load(self.rejections_path.read_text())
        self.assertEqual(data['rejections'][0]['reason'], 'my reason')

    def test_already_rejected_candidate_is_skipped_automatically(self):
        # Pre-populate rejection registry with _CLUSTER_A's centroid
        from rejection_registry import RejectionRegistry
        reg = RejectionRegistry(self.rejections_path)
        reg.add(_CLUSTER_A['centroid'], _CLUSTER_A['projects'], 'pre-existing')

        called_with = []
        def decide(c):
            called_with.append(c)
            return (Decision.ACCEPT, None)

        validate_candidates(
            self.report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=decide,
        )
        # decide_fn only called for _CLUSTER_B — _CLUSTER_A was auto-suppressed
        self.assertEqual(len(called_with), 1)
        self.assertEqual(called_with[0]['projects'], ['proj-c', 'proj-d'])


class TestValidateCandidatesIntegration(unittest.TestCase):
    """Integration tests — validate_candidates using real rejection_registry + pattern_entry."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.out_dir = Path(self.tmp.name) / 'patterns'
        self.rejections_path = Path(self.tmp.name) / 'rejections.yaml'
        self.rejections_path.write_text('rejections: []\n')

    def tearDown(self):
        self.tmp.cleanup()

    def test_accepted_entry_is_valid_yaml(self):
        report = CandidateReport(cluster_candidates=[_CLUSTER_A], delta_candidates=[])
        validate_candidates(
            report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: (Decision.ACCEPT, None),
        )
        files = list(self.out_dir.glob('GP-*.md'))
        self.assertEqual(len(files), 1)
        content = files[0].read_text()
        parts = content.split('---\n', 2)
        frontmatter = yaml.safe_load(parts[1])
        self.assertEqual(frontmatter['garden'], 'patterns')

    def test_accept_then_reject_correct_state(self):
        report = CandidateReport(
            cluster_candidates=[_CLUSTER_A, _CLUSTER_B],
            delta_candidates=[],
        )
        decisions = iter([(Decision.ACCEPT, None), (Decision.REJECT, 'low signal')])
        summary = validate_candidates(
            report, _REGISTRY_PROJECTS,
            self.rejections_path, self.out_dir,
            decide_fn=lambda c: next(decisions),
        )
        self.assertEqual(len(list(self.out_dir.glob('GP-*.md'))), 1)
        data = yaml.safe_load(self.rejections_path.read_text())
        self.assertEqual(len(data['rejections']), 1)
        self.assertEqual(summary.total(), 2)
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_validate_candidates.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'validate_candidates'`

- [ ] **Step 3: Implement `scripts/validate_candidates.py`**

```python
"""Human validation gate for pattern candidates.

validate_candidates() accepts a decide_fn callback for testability.
The CLI (__main__) wires it to stdin.
"""
from dataclasses import dataclass
from pathlib import Path
from typing import Callable

from candidate_report import CandidateReport
from rejection_registry import RejectionRegistry
from pattern_entry import write_skeleton


class Decision:
    ACCEPT = 'accept'
    REJECT = 'reject'
    SKIP = 'skip'


@dataclass
class SessionSummary:
    accepted: int
    rejected: int
    skipped: int

    def total(self) -> int:
        return self.accepted + self.rejected + self.skipped


def validate_candidates(
    report: CandidateReport,
    registry_projects: list[dict],
    rejections_path: Path,
    out_dir: Path,
    decide_fn: Callable[[dict], tuple[str, str | None]],
) -> SessionSummary:
    """Process all cluster candidates through decide_fn.

    decide_fn(candidate) -> (Decision.ACCEPT|REJECT|SKIP, reason_or_None)
    Already-rejected candidates are auto-skipped without calling decide_fn.
    """
    rejection_reg = RejectionRegistry(rejections_path)
    accepted = rejected = skipped = 0

    for candidate in report.cluster_candidates:
        if rejection_reg.is_rejected(candidate['centroid']):
            skipped += 1
            continue

        decision, reason = decide_fn(candidate)

        if decision == Decision.ACCEPT:
            write_skeleton(candidate, registry_projects, out_dir)
            accepted += 1
        elif decision == Decision.REJECT:
            rejection_reg.add(candidate['centroid'], candidate['projects'], reason or '')
            rejected += 1
        else:
            skipped += 1

    return SessionSummary(accepted=accepted, rejected=rejected, skipped=skipped)


if __name__ == '__main__':
    import json, sys
    from pathlib import Path
    from candidate_report import load_report

    if len(sys.argv) < 4:
        print('Usage: validate_candidates.py <report.json> <rejections.yaml> <out_dir>')
        sys.exit(1)

    report = load_report(Path(sys.argv[1]))
    rejections_path = Path(sys.argv[2])
    out_dir = Path(sys.argv[3])
    registry_projects: list[dict] = []  # populated by caller via --registry flag in a future iteration

    def _stdin_decide(candidate: dict) -> tuple[str, str | None]:
        print(f"\nCandidate: {candidate['projects']}")
        print(f"  Similarity: {candidate['similarity_score']}")
        print(f"  Centroid: {candidate['centroid']}")
        choice = input('  [a]ccept / [r]eject / [s]kip: ').strip().lower()
        if choice == 'a':
            return Decision.ACCEPT, None
        elif choice == 'r':
            reason = input('  Reason: ').strip()
            return Decision.REJECT, reason
        return Decision.SKIP, None

    summary = validate_candidates(report, registry_projects, rejections_path, out_dir, _stdin_decide)
    print(f'\nDone — accepted: {summary.accepted}, rejected: {summary.rejected}, skipped: {summary.skipped}')
```

- [ ] **Step 4: Run tests — verify all tests pass**

```bash
python3 -m pytest tests/test_validate_candidates.py -v
```
Expected: all tests pass (Decision, SessionSummary, correctness, integration).

- [ ] **Step 5: Commit**

```bash
git add scripts/validate_candidates.py tests/test_validate_candidates.py
git commit -m "feat(gate): validation gate — accept/reject/skip with auto-suppression of known rejections

Refs #35"
```

---

## Task 5: End-to-end pipeline runner

**Files:**
- Create: `scripts/run_pipeline.py`
- Create: `tests/test_run_pipeline.py`

- [ ] **Step 1: Write failing tests (integration + E2E + happy path)**

`tests/test_run_pipeline.py`:
```python
import json
import subprocess
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from run_pipeline import run_pipeline, PipelineConfig
from project_registry import ProjectRegistry
from candidate_report import load_report


def _git(repo: Path, *args: str) -> None:
    subprocess.run(['git', '-C', str(repo)] + list(args), check=True, capture_output=True)


def _make_synthetic_project(root: Path, name: str, with_interface: bool = False) -> Path:
    """Create a minimal git repo with two tags to exercise delta analysis."""
    repo = root / name
    repo.mkdir()
    _git(repo, 'init')
    _git(repo, 'config', 'user.email', 'test@example.com')
    _git(repo, 'config', 'user.name', 'Test User')

    src = repo / 'src'
    src.mkdir()
    (src / 'Service.java').write_text('public class Service {}\n')
    _git(repo, 'add', '.')
    _git(repo, 'commit', '-m', 'initial')
    _git(repo, 'tag', 'v1.0')

    if with_interface:
        (src / 'Evaluator.java').write_text(
            'public interface Evaluator { void eval(); }\n'
        )
        _git(repo, 'add', '.')
        _git(repo, 'commit', '-m', 'add interface')
    _git(repo, 'tag', 'v2.0')

    return repo


class TestPipelineConfigUnit(unittest.TestCase):
    """Unit tests — PipelineConfig construction."""

    def test_pipeline_config_stores_paths(self):
        cfg = PipelineConfig(
            registry_path=Path('/tmp/projects.yaml'),
            rejections_path=Path('/tmp/rejections.yaml'),
            report_path=Path('/tmp/report.json'),
            project_roots={},
        )
        self.assertEqual(cfg.registry_path, Path('/tmp/projects.yaml'))

    def test_pipeline_config_project_roots_default_empty(self):
        cfg = PipelineConfig(
            registry_path=Path('/tmp/projects.yaml'),
            rejections_path=Path('/tmp/rejections.yaml'),
            report_path=Path('/tmp/report.json'),
        )
        self.assertEqual(cfg.project_roots, {})


class TestRunPipelineIntegration(unittest.TestCase):
    """Integration tests — pipeline stages connected."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.registry_path = self.root / 'projects.yaml'
        self.registry_path.write_text('projects: []\n')
        self.rejections_path = self.root / 'rejections.yaml'
        self.rejections_path.write_text('rejections: []\n')
        self.report_path = self.root / 'report.json'

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_registry_produces_empty_report(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={},
        )
        run_pipeline(cfg)
        report = load_report(self.report_path)
        self.assertEqual(report.cluster_candidates, [])
        self.assertEqual(report.delta_candidates, [])

    def test_report_file_created(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={},
        )
        run_pipeline(cfg)
        self.assertTrue(self.report_path.exists())

    def test_report_is_valid_json(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={},
        )
        run_pipeline(cfg)
        data = json.loads(self.report_path.read_text())
        self.assertIn('cluster_candidates', data)
        self.assertIn('delta_candidates', data)
        self.assertIn('generated_at', data)


class TestRunPipelineE2E(unittest.TestCase):
    """E2E tests — full pipeline against synthetic project fixtures."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

        # Create two structurally similar projects
        self.proj_a = _make_synthetic_project(self.root, 'proj-a', with_interface=True)
        self.proj_b = _make_synthetic_project(self.root, 'proj-b', with_interface=True)

        self.registry_path = self.root / 'projects.yaml'
        self.registry_path.write_text('projects: []\n')
        reg = ProjectRegistry(self.registry_path)
        reg.add({'project': 'proj-a', 'url': 'https://github.com/org/proj-a',
                 'domain': 'jvm', 'primary_language': 'java',
                 'frameworks': [], 'last_processed_commit': None,
                 'notable_contributors': []})
        reg.add({'project': 'proj-b', 'url': 'https://github.com/org/proj-b',
                 'domain': 'jvm', 'primary_language': 'java',
                 'frameworks': [], 'last_processed_commit': None,
                 'notable_contributors': []})

        self.rejections_path = self.root / 'rejections.yaml'
        self.rejections_path.write_text('rejections: []\n')
        self.report_path = self.root / 'report.json'

    def tearDown(self):
        self.tmp.cleanup()

    def test_e2e_pipeline_produces_report_with_delta_candidates(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={'proj-a': self.proj_a, 'proj-b': self.proj_b},
        )
        run_pipeline(cfg)
        report = load_report(self.report_path)
        # Both projects added an interface in v2.0 — should appear as delta candidates
        self.assertGreater(len(report.delta_candidates), 0)

    def test_e2e_delta_candidates_have_required_fields(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={'proj-a': self.proj_a, 'proj-b': self.proj_b},
        )
        run_pipeline(cfg)
        report = load_report(self.report_path)
        for candidate in report.delta_candidates:
            self.assertIn('file', candidate)
            self.assertIn('kind', candidate)
            self.assertIn('introduced_at', candidate)
            self.assertIn('project', candidate)  # pipeline adds project name

    def test_e2e_updates_last_processed_commit(self):
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={'proj-a': self.proj_a, 'proj-b': self.proj_b},
        )
        run_pipeline(cfg)
        reg = ProjectRegistry(self.registry_path)
        proj_a = reg.get('proj-a')
        self.assertIsNotNone(proj_a['last_processed_commit'])


class TestRunPipelineHappyPath(unittest.TestCase):
    """Happy path — complete session: candidates in, patterns-garden entries out."""

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)
        self.proj_a = _make_synthetic_project(self.root, 'proj-a', with_interface=True)
        self.proj_b = _make_synthetic_project(self.root, 'proj-b', with_interface=True)

        self.registry_path = self.root / 'projects.yaml'
        self.registry_path.write_text('projects: []\n')
        reg = ProjectRegistry(self.registry_path)
        for name, path in [('proj-a', self.proj_a), ('proj-b', self.proj_b)]:
            reg.add({'project': name, 'url': f'https://github.com/org/{name}',
                     'domain': 'jvm', 'primary_language': 'java',
                     'frameworks': [], 'last_processed_commit': None,
                     'notable_contributors': []})

        self.rejections_path = self.root / 'rejections.yaml'
        self.rejections_path.write_text('rejections: []\n')
        self.report_path = self.root / 'report.json'
        self.patterns_dir = self.root / 'patterns'

    def tearDown(self):
        self.tmp.cleanup()

    def test_happy_path_pipeline_to_accepted_entry(self):
        """Full journey: run pipeline → load report → validate (accept all) → GP entry on disk."""
        # Step 1: run pipeline
        cfg = PipelineConfig(
            registry_path=self.registry_path,
            rejections_path=self.rejections_path,
            report_path=self.report_path,
            project_roots={'proj-a': self.proj_a, 'proj-b': self.proj_b},
        )
        run_pipeline(cfg)

        # Step 2: load the report
        report = load_report(self.report_path)
        self.assertGreater(report.total_count(), 0)

        # Step 3: validate — accept everything
        from validate_candidates import validate_candidates, Decision
        registry = ProjectRegistry(self.registry_path)
        summary = validate_candidates(
            report,
            registry.list(),
            self.rejections_path,
            self.patterns_dir,
            decide_fn=lambda c: (Decision.ACCEPT, None),
        )

        # Step 4: at least one GP entry was written
        gp_files = list(self.patterns_dir.glob('GP-*.md'))
        self.assertGreater(len(gp_files), 0)
        self.assertGreater(summary.accepted, 0)
        self.assertEqual(summary.rejected, 0)
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python3 -m pytest tests/test_run_pipeline.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'run_pipeline'`

- [ ] **Step 3: Implement `scripts/run_pipeline.py`**

```python
"""Orchestrate: registry → feature extraction → clustering → delta analysis → candidate report."""
import subprocess
from dataclasses import dataclass, field
from pathlib import Path

from project_registry import ProjectRegistry
from feature_extractor import extract_features
from cluster_pipeline import cluster_projects, FEATURE_KEYS
from delta_analysis import delta_analysis, get_major_version_tags
from rejection_registry import RejectionRegistry
from candidate_report import CandidateReport, save_report


@dataclass
class PipelineConfig:
    registry_path: Path
    rejections_path: Path
    report_path: Path
    project_roots: dict[str, Path] = field(default_factory=dict)
    known_patterns: list[dict] = field(default_factory=list)
    similarity_threshold: float = 0.75


def _head_commit(repo: Path) -> str:
    result = subprocess.run(
        ['git', '-C', str(repo), 'rev-parse', 'HEAD'],
        capture_output=True, text=True, check=True,
    )
    return result.stdout.strip()


def run_pipeline(config: PipelineConfig) -> CandidateReport:
    registry = ProjectRegistry(config.registry_path)
    rejection_reg = RejectionRegistry(config.rejections_path)
    projects = registry.list()

    # Step 1: extract features for each project with a known local path
    fingerprints: dict[str, dict] = {}
    for project in projects:
        name = project['project']
        root = config.project_roots.get(name)
        if root is None:
            continue
        fingerprints[name] = extract_features(Path(root))

    # Step 2: cluster fingerprints, filter known rejections
    raw_clusters = cluster_projects(
        fingerprints, config.known_patterns, config.similarity_threshold
    )
    cluster_candidates = [
        c for c in raw_clusters
        if not rejection_reg.is_rejected(c['centroid'])
    ]

    # Step 3: delta analysis — compare consecutive major version tags per project
    delta_candidates: list[dict] = []
    for project in projects:
        name = project['project']
        root = config.project_roots.get(name)
        if root is None:
            continue
        repo = Path(root)
        tags = get_major_version_tags(repo)
        if len(tags) < 2:
            continue
        for from_ref, to_ref in zip(tags[:-1], tags[1:]):
            for candidate in delta_analysis(repo, from_ref, to_ref):
                delta_candidates.append({**candidate, 'project': name})

        # Step 4: update last_processed_commit
        try:
            commit = _head_commit(repo)
            registry.update_commit(name, commit)
        except (subprocess.CalledProcessError, ValueError):
            pass

    report = CandidateReport(
        cluster_candidates=cluster_candidates,
        delta_candidates=delta_candidates,
    )
    save_report(report, Path(config.report_path))
    return report
```

- [ ] **Step 4: Run all tests — verify they pass**

```bash
python3 -m pytest tests/test_run_pipeline.py -v
```
Expected: all tests pass (unit, integration, E2E, happy path).

- [ ] **Step 5: Full suite — verify no regressions**

```bash
python3 -m pytest tests/ -q 2>&1 | tail -5
```
Expected: 775 + new tests pass, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add scripts/run_pipeline.py tests/test_run_pipeline.py
git commit -m "feat(pipeline): end-to-end runner — registry → extract → cluster → delta → report

Refs #35"
```

---

## Task 6: Close and push

- [ ] **Step 1: Run full test suite one final time**

```bash
python3 -m pytest tests/ -v 2>&1 | tail -10
```
Expected: all tests pass, 0 failures.

- [ ] **Step 2: Close issue**

```bash
gh issue close 35 --repo Hortora/soredium \
  --comment "Shipped: rejection registry, candidate report, pattern entry generator, validation gate, pipeline runner. Full test pyramid: unit, correctness, integration, E2E, happy path. Refs #31."
```
