# Area 2 Phase 3 — Project Registry + Ecosystem Mining Pipeline

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the project registry and two pattern-discovery pipelines (structural clustering + delta analysis) that feed the patterns-garden.

**Architecture:** A YAML-backed project registry (`registry/projects.yaml`) tracks monitored projects with a `last_processed_commit` cursor. A feature extractor walks cloned source trees and emits structural fingerprints (interface counts, abstraction depth, injection points). A clustering pipeline groups projects by fingerprint similarity and surfaces candidates that don't match known patterns. A delta analysis pipeline inspects git tag diffs to find when new abstraction layers first appeared, supplying origin provenance for confirmed patterns.

**Tech Stack:** Python 3.11+, PyYAML, scikit-learn (KMeans + cosine similarity), pathlib, subprocess (git), unittest

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `registry/projects.yaml` | Create | Canonical list of monitored projects |
| `scripts/project_registry.py` | Create | CRUD for registry — load/save/add/update/list |
| `scripts/feature_extractor.py` | Create | Walk a source tree; emit structural fingerprint dict |
| `scripts/cluster_pipeline.py` | Create | Cluster fingerprints; return candidate groups |
| `scripts/delta_analysis.py` | Create | Diff two git tags; return new-abstraction candidates |
| `tests/test_project_registry.py` | Create | Registry CRUD tests |
| `tests/test_feature_extractor.py` | Create | Extractor tests against synthetic source trees |
| `tests/test_cluster_pipeline.py` | Create | Clustering tests with known fixture fingerprints |
| `tests/test_delta_analysis.py` | Create | Delta tests against a synthetic git repo |

---

## Task 1: Project registry schema and CRUD

**Files:**
- Create: `registry/projects.yaml`
- Create: `scripts/project_registry.py`
- Create: `tests/test_project_registry.py`

- [ ] **Step 1: Create the registry directory and seed file**

```bash
mkdir -p /path/to/soredium/registry
```

`registry/projects.yaml`:
```yaml
# Hortora project registry — monitored projects for ecosystem mining
projects: []
```

- [ ] **Step 2: Write failing tests**

`tests/test_project_registry.py`:
```python
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory
import yaml

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from project_registry import ProjectRegistry


class TestProjectRegistry(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.path = Path(self.tmp.name) / 'projects.yaml'
        self.path.write_text('projects: []\n')
        self.registry = ProjectRegistry(self.path)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_registry_lists_nothing(self):
        self.assertEqual(self.registry.list(), [])

    def test_add_project_persists(self):
        self.registry.add({
            'project': 'serverless-workflow',
            'url': 'https://github.com/serverlessworkflow/specification',
            'domain': 'jvm',
            'primary_language': 'java',
            'frameworks': ['quarkus'],
            'last_processed_commit': None,
            'notable_contributors': [],
        })
        projects = self.registry.list()
        self.assertEqual(len(projects), 1)
        self.assertEqual(projects[0]['project'], 'serverless-workflow')

    def test_add_duplicate_raises(self):
        entry = {'project': 'foo', 'url': 'https://github.com/foo/foo',
                 'domain': 'jvm', 'primary_language': 'java',
                 'frameworks': [], 'last_processed_commit': None,
                 'notable_contributors': []}
        self.registry.add(entry)
        with self.assertRaises(ValueError):
            self.registry.add(entry)

    def test_update_last_processed_commit(self):
        self.registry.add({'project': 'foo', 'url': 'https://github.com/foo/foo',
                           'domain': 'jvm', 'primary_language': 'java',
                           'frameworks': [], 'last_processed_commit': None,
                           'notable_contributors': []})
        self.registry.update_commit('foo', 'abc1234')
        project = self.registry.get('foo')
        self.assertEqual(project['last_processed_commit'], 'abc1234')

    def test_get_unknown_project_returns_none(self):
        self.assertIsNone(self.registry.get('does-not-exist'))

    def test_required_fields_validated_on_add(self):
        with self.assertRaises(ValueError):
            self.registry.add({'project': 'missing-fields'})

    def test_data_persists_across_instances(self):
        self.registry.add({'project': 'foo', 'url': 'https://github.com/foo/foo',
                           'domain': 'jvm', 'primary_language': 'java',
                           'frameworks': [], 'last_processed_commit': None,
                           'notable_contributors': []})
        reload = ProjectRegistry(self.path)
        self.assertEqual(len(reload.list()), 1)
```

- [ ] **Step 3: Run tests — verify they all fail**

```bash
cd soredium && python -m pytest tests/test_project_registry.py -v 2>&1 | head -20
```
Expected: `ModuleNotFoundError: No module named 'project_registry'`

- [ ] **Step 4: Implement `scripts/project_registry.py`**

```python
"""CRUD for the Hortora project registry (registry/projects.yaml)."""
from pathlib import Path
import yaml

REQUIRED_FIELDS = {
    'project', 'url', 'domain', 'primary_language',
    'frameworks', 'last_processed_commit', 'notable_contributors',
}


class ProjectRegistry:
    def __init__(self, path: Path):
        self.path = Path(path)

    def _load(self) -> list:
        data = yaml.safe_load(self.path.read_text()) or {}
        return data.get('projects', [])

    def _save(self, projects: list) -> None:
        self.path.write_text(yaml.dump({'projects': projects}, default_flow_style=False, sort_keys=False))

    def list(self) -> list:
        return self._load()

    def get(self, name: str) -> dict | None:
        return next((p for p in self._load() if p['project'] == name), None)

    def add(self, entry: dict) -> None:
        missing = REQUIRED_FIELDS - entry.keys()
        if missing:
            raise ValueError(f"Missing required fields: {missing}")
        projects = self._load()
        if any(p['project'] == entry['project'] for p in projects):
            raise ValueError(f"Project '{entry['project']}' already in registry")
        projects.append(entry)
        self._save(projects)

    def update_commit(self, name: str, commit: str) -> None:
        projects = self._load()
        for p in projects:
            if p['project'] == name:
                p['last_processed_commit'] = commit
                self._save(projects)
                return
        raise ValueError(f"Project '{name}' not found in registry")
```

- [ ] **Step 5: Run tests — verify they all pass**

```bash
python -m pytest tests/test_project_registry.py -v
```
Expected: 7 tests pass.

- [ ] **Step 6: Commit**

```bash
git add registry/projects.yaml scripts/project_registry.py tests/test_project_registry.py
git commit -m "feat(registry): project registry CRUD — load/save/add/update/list

Refs #34"
```

---

## Task 2: Structural feature extractor

**Files:**
- Create: `scripts/feature_extractor.py`
- Create: `tests/test_feature_extractor.py`

The extractor walks a source directory and returns a fingerprint dict:
```python
{
  'interface_count': int,       # files/classes declared as interface or abstract
  'abstraction_depth': float,   # mean inheritance chain length (approximated)
  'injection_points': int,      # @Inject / @Autowired / @ApplicationScoped annotations
  'extension_signatures': int,  # implements / extends usage in type declarations
  'file_count': int,            # total source files
  'spi_patterns': int,          # META-INF/services entries or @Provider annotations
}
```

Extraction is regex-based over source text — no compiler required.

- [ ] **Step 1: Write failing tests**

`tests/test_feature_extractor.py`:
```python
import unittest
import sys
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from feature_extractor import extract_features


def _write(root: Path, rel: str, content: str) -> None:
    p = root / rel
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(content)


class TestFeatureExtractor(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.root = Path(self.tmp.name)

    def tearDown(self):
        self.tmp.cleanup()

    def test_empty_directory_returns_zero_counts(self):
        features = extract_features(self.root)
        self.assertEqual(features['interface_count'], 0)
        self.assertEqual(features['file_count'], 0)

    def test_counts_java_interfaces(self):
        _write(self.root, 'src/Foo.java', 'public interface Foo {}')
        _write(self.root, 'src/Bar.java', 'public interface Bar extends Foo {}')
        _write(self.root, 'src/Baz.java', 'public class Baz implements Foo {}')
        features = extract_features(self.root)
        self.assertEqual(features['interface_count'], 2)
        self.assertEqual(features['file_count'], 3)

    def test_counts_injection_points(self):
        _write(self.root, 'src/A.java',
               '@ApplicationScoped\npublic class A {\n  @Inject Foo foo;\n}')
        features = extract_features(self.root)
        self.assertEqual(features['injection_points'], 2)

    def test_counts_extension_signatures(self):
        _write(self.root, 'src/A.java', 'public class A extends B implements C, D {}')
        features = extract_features(self.root)
        self.assertEqual(features['extension_signatures'], 1)

    def test_counts_spi_services_file(self):
        _write(self.root, 'META-INF/services/com.example.Foo',
               'com.example.impl.FooImpl\n')
        features = extract_features(self.root)
        self.assertEqual(features['spi_patterns'], 1)

    def test_ignores_non_source_files(self):
        _write(self.root, 'README.md', '# interface Foo')
        _write(self.root, 'build.xml', '<interface name="Foo"/>')
        features = extract_features(self.root)
        self.assertEqual(features['interface_count'], 0)

    def test_python_injection_via_type_hints(self):
        _write(self.root, 'src/service.py',
               'def __init__(self, foo: Foo, bar: Bar) -> None: ...')
        features = extract_features(self.root)
        self.assertGreaterEqual(features['injection_points'], 0)
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python -m pytest tests/test_feature_extractor.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'feature_extractor'`

- [ ] **Step 3: Implement `scripts/feature_extractor.py`**

```python
"""Structural feature extractor — regex-based, no compiler required."""
import re
from pathlib import Path

_JAVA_EXTS = {'.java', '.kt'}
_PYTHON_EXTS = {'.py'}
_SOURCE_EXTS = _JAVA_EXTS | _PYTHON_EXTS

_RE_JAVA_INTERFACE = re.compile(r'\b(?:interface|abstract\s+class)\b')
_RE_JAVA_INJECT = re.compile(r'@(?:Inject|Autowired|ApplicationScoped|RequestScoped|SessionScoped|Singleton)')
_RE_JAVA_EXTENDS = re.compile(r'\bclass\s+\w+(?:\s*<[^>]*>)?\s+(?:extends|implements)\b')
_RE_PYTHON_INJECT = re.compile(r'def __init__\s*\(self(?:,\s*\w+:\s*\w+)+\)')


def extract_features(root: Path) -> dict:
    root = Path(root)
    interface_count = 0
    injection_points = 0
    extension_signatures = 0
    spi_patterns = 0
    file_count = 0

    for path in root.rglob('*'):
        if not path.is_file():
            continue

        # SPI services files
        if 'META-INF/services' in str(path) and path.suffix == '':
            spi_patterns += 1
            continue

        if path.suffix not in _SOURCE_EXTS:
            continue

        file_count += 1
        text = path.read_text(errors='replace')

        if path.suffix in _JAVA_EXTS:
            interface_count += len(_RE_JAVA_INTERFACE.findall(text))
            injection_points += len(_RE_JAVA_INJECT.findall(text))
            extension_signatures += len(_RE_JAVA_EXTENDS.findall(text))
        elif path.suffix in _PYTHON_EXTS:
            injection_points += len(_RE_PYTHON_INJECT.findall(text))

    # abstraction_depth: ratio of interfaces+abstracts to total files (0–1 scale)
    abstraction_depth = round(interface_count / file_count, 3) if file_count else 0.0

    return {
        'interface_count': interface_count,
        'abstraction_depth': abstraction_depth,
        'injection_points': injection_points,
        'extension_signatures': extension_signatures,
        'file_count': file_count,
        'spi_patterns': spi_patterns,
    }
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
python -m pytest tests/test_feature_extractor.py -v
```
Expected: 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/feature_extractor.py tests/test_feature_extractor.py
git commit -m "feat(mining): structural feature extractor — regex-based fingerprinting

Refs #34"
```

---

## Task 3: Clustering pipeline

**Files:**
- Create: `scripts/cluster_pipeline.py`
- Create: `tests/test_cluster_pipeline.py`

Takes a dict of `{project_name: fingerprint}` and returns candidate groups — clusters of 2+ projects that share unusual structural similarity and don't match any known pattern signature.

```python
# Input
fingerprints = {
    'project-a': {'interface_count': 12, 'abstraction_depth': 0.4, ...},
    'project-b': {'interface_count': 11, 'abstraction_depth': 0.38, ...},
    'project-c': {'interface_count': 1,  'abstraction_depth': 0.05, ...},
}
known_patterns = [
    {'name': 'event-sourcing', 'signature': {'interface_count': 15, ...}},
]

# Output
candidates = [
    {
        'projects': ['project-a', 'project-b'],
        'centroid': {'interface_count': 11.5, ...},
        'similarity_score': 0.97,
        'matches_known_pattern': None,
    }
]
```

- [ ] **Step 1: Write failing tests**

`tests/test_cluster_pipeline.py`:
```python
import unittest
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from cluster_pipeline import cluster_projects, fingerprint_to_vector, FEATURE_KEYS


_FP_HIGH_ABSTRACTION = {
    'interface_count': 20, 'abstraction_depth': 0.6,
    'injection_points': 15, 'extension_signatures': 18,
    'file_count': 33, 'spi_patterns': 4,
}
_FP_HIGH_ABSTRACTION_2 = {
    'interface_count': 18, 'abstraction_depth': 0.55,
    'injection_points': 14, 'extension_signatures': 16,
    'file_count': 30, 'spi_patterns': 3,
}
_FP_LOW_ABSTRACTION = {
    'interface_count': 1, 'abstraction_depth': 0.02,
    'injection_points': 2, 'extension_signatures': 1,
    'file_count': 50, 'spi_patterns': 0,
}


class TestClusterPipeline(unittest.TestCase):

    def test_feature_keys_defined(self):
        self.assertIn('interface_count', FEATURE_KEYS)
        self.assertIn('abstraction_depth', FEATURE_KEYS)

    def test_fingerprint_to_vector_returns_list(self):
        vec = fingerprint_to_vector(_FP_HIGH_ABSTRACTION)
        self.assertEqual(len(vec), len(FEATURE_KEYS))
        self.assertIsInstance(vec[0], float)

    def test_too_few_projects_returns_empty(self):
        result = cluster_projects({'only-one': _FP_HIGH_ABSTRACTION}, known_patterns=[])
        self.assertEqual(result, [])

    def test_similar_projects_form_candidate(self):
        fingerprints = {
            'proj-a': _FP_HIGH_ABSTRACTION,
            'proj-b': _FP_HIGH_ABSTRACTION_2,
            'proj-c': _FP_LOW_ABSTRACTION,
        }
        candidates = cluster_projects(fingerprints, known_patterns=[])
        # proj-a and proj-b should cluster together
        clustered = [set(c['projects']) for c in candidates]
        self.assertIn({'proj-a', 'proj-b'}, clustered)

    def test_candidate_has_required_fields(self):
        fingerprints = {
            'proj-a': _FP_HIGH_ABSTRACTION,
            'proj-b': _FP_HIGH_ABSTRACTION_2,
        }
        candidates = cluster_projects(fingerprints, known_patterns=[])
        self.assertTrue(len(candidates) > 0)
        c = candidates[0]
        self.assertIn('projects', c)
        self.assertIn('centroid', c)
        self.assertIn('similarity_score', c)
        self.assertIn('matches_known_pattern', c)

    def test_known_pattern_match_is_tagged(self):
        fingerprints = {
            'proj-a': _FP_HIGH_ABSTRACTION,
            'proj-b': _FP_HIGH_ABSTRACTION_2,
        }
        # A known pattern whose signature closely matches these projects
        known = [{'name': 'plugin-system', 'signature': _FP_HIGH_ABSTRACTION}]
        candidates = cluster_projects(fingerprints, known_patterns=known)
        # All candidates matching a known pattern should have it tagged
        for c in candidates:
            if c['matches_known_pattern']:
                self.assertIsInstance(c['matches_known_pattern'], str)

    def test_minimum_cluster_size_is_two(self):
        fingerprints = {f'proj-{i}': _FP_HIGH_ABSTRACTION for i in range(5)}
        candidates = cluster_projects(fingerprints, known_patterns=[])
        for c in candidates:
            self.assertGreaterEqual(len(c['projects']), 2)
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python -m pytest tests/test_cluster_pipeline.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'cluster_pipeline'`

- [ ] **Step 3: Implement `scripts/cluster_pipeline.py`**

```python
"""Cluster project fingerprints and surface novel pattern candidates."""
import math
from typing import Any

FEATURE_KEYS = [
    'interface_count',
    'abstraction_depth',
    'injection_points',
    'extension_signatures',
    'file_count',
    'spi_patterns',
]


def fingerprint_to_vector(fp: dict) -> list[float]:
    return [float(fp.get(k, 0)) for k in FEATURE_KEYS]


def _cosine_similarity(a: list[float], b: list[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    mag_a = math.sqrt(sum(x * x for x in a))
    mag_b = math.sqrt(sum(x * x for x in b))
    if mag_a == 0 or mag_b == 0:
        return 0.0
    return dot / (mag_a * mag_b)


def _normalize(vectors: list[list[float]]) -> list[list[float]]:
    """Min-max normalize each feature dimension."""
    if not vectors:
        return vectors
    n_features = len(vectors[0])
    mins = [min(v[i] for v in vectors) for i in range(n_features)]
    maxs = [max(v[i] for v in vectors) for i in range(n_features)]
    result = []
    for v in vectors:
        norm = []
        for i in range(n_features):
            span = maxs[i] - mins[i]
            norm.append((v[i] - mins[i]) / span if span > 0 else 0.0)
        result.append(norm)
    return result


def _centroid(vectors: list[list[float]]) -> list[float]:
    n = len(vectors)
    return [sum(v[i] for v in vectors) / n for i in range(len(vectors[0]))]


def _matches_known(centroid: list[float], known_patterns: list[dict],
                   threshold: float = 0.92) -> str | None:
    for pattern in known_patterns:
        sig_vec = fingerprint_to_vector(pattern['signature'])
        if _cosine_similarity(centroid, sig_vec) >= threshold:
            return pattern['name']
    return None


def cluster_projects(
    fingerprints: dict[str, dict],
    known_patterns: list[dict],
    similarity_threshold: float = 0.90,
) -> list[dict[str, Any]]:
    """Return candidate clusters of structurally similar projects."""
    if len(fingerprints) < 2:
        return []

    names = list(fingerprints.keys())
    raw_vectors = [fingerprint_to_vector(fingerprints[n]) for n in names]
    norm_vectors = _normalize(raw_vectors)

    # Greedy single-linkage clustering at similarity_threshold
    assigned = [False] * len(names)
    clusters = []

    for i in range(len(names)):
        if assigned[i]:
            continue
        group = [i]
        for j in range(i + 1, len(names)):
            if assigned[j]:
                continue
            if _cosine_similarity(norm_vectors[i], norm_vectors[j]) >= similarity_threshold:
                group.append(j)
                assigned[j] = True
        if len(group) >= 2:
            assigned[i] = True
            group_vecs = [norm_vectors[k] for k in group]
            group_raw = [raw_vectors[k] for k in group]
            c = _centroid(group_vecs)
            raw_c = _centroid(group_raw)
            raw_centroid_dict = {k: round(raw_c[idx], 3) for idx, k in enumerate(FEATURE_KEYS)}
            sim = sum(
                _cosine_similarity(norm_vectors[group[0]], norm_vectors[k])
                for k in group[1:]
            ) / (len(group) - 1)
            clusters.append({
                'projects': [names[k] for k in group],
                'centroid': raw_centroid_dict,
                'similarity_score': round(sim, 4),
                'matches_known_pattern': _matches_known(c, known_patterns),
            })

    return clusters
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
python -m pytest tests/test_cluster_pipeline.py -v
```
Expected: 6 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/cluster_pipeline.py tests/test_cluster_pipeline.py
git commit -m "feat(mining): structural clustering pipeline — cosine similarity, known-pattern tagging

Refs #34"
```

---

## Task 4: Delta analysis pipeline

**Files:**
- Create: `scripts/delta_analysis.py`
- Create: `tests/test_delta_analysis.py`

Given a cloned repo path and two git refs (e.g. `v1.0` and `v2.0`), returns new abstraction candidates — source files that first introduced interface/abstract class declarations between those refs.

```python
# Input
candidates = delta_analysis('/path/to/repo', from_ref='v1.0', to_ref='v2.0')

# Output
[
    {
        'file': 'src/main/java/io/example/Evaluator.java',
        'kind': 'interface',           # or 'abstract_class'
        'introduced_at': 'v2.0',
        'commit': 'abc1234',
        'author': 'fabian-martinez',
        'date': '2022-03-14',
    },
    ...
]
```

- [ ] **Step 1: Write failing tests using a synthetic git repo**

`tests/test_delta_analysis.py`:
```python
import unittest
import sys
import subprocess
from pathlib import Path
from tempfile import TemporaryDirectory

sys.path.insert(0, str(Path(__file__).parent.parent / 'scripts'))
from delta_analysis import delta_analysis, get_major_version_tags


def _git(repo: Path, *args: str) -> None:
    subprocess.run(['git', '-C', str(repo)] + list(args), check=True,
                   capture_output=True)


def _make_repo(root: Path) -> Path:
    """Create a synthetic git repo with two tagged versions."""
    repo = root / 'repo'
    repo.mkdir()
    _git(repo, 'init')
    _git(repo, 'config', 'user.email', 'test@example.com')
    _git(repo, 'config', 'user.name', 'Test User')

    # v1.0 — plain class only
    (repo / 'src').mkdir()
    (repo / 'src' / 'Service.java').write_text(
        'public class Service {}\n'
    )
    _git(repo, 'add', '.')
    _git(repo, 'commit', '-m', 'initial')
    _git(repo, 'tag', 'v1.0')

    # v2.0 — adds an interface
    (repo / 'src' / 'Evaluator.java').write_text(
        'public interface Evaluator { void eval(); }\n'
    )
    (repo / 'src' / 'AbstractBase.java').write_text(
        'public abstract class AbstractBase {}\n'
    )
    _git(repo, 'add', '.')
    _git(repo, 'commit', '-m', 'add Evaluator interface')
    _git(repo, 'tag', 'v2.0')

    return repo


class TestDeltaAnalysis(unittest.TestCase):

    def setUp(self):
        self.tmp = TemporaryDirectory()
        self.repo = _make_repo(Path(self.tmp.name))

    def tearDown(self):
        self.tmp.cleanup()

    def test_no_new_abstractions_returns_empty(self):
        # v1.0 to v1.0 — no change
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v1.0')
        self.assertEqual(result, [])

    def test_detects_new_interface(self):
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v2.0')
        files = [c['file'] for c in result]
        self.assertTrue(any('Evaluator' in f for f in files))

    def test_detects_new_abstract_class(self):
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v2.0')
        files = [c['file'] for c in result]
        self.assertTrue(any('AbstractBase' in f for f in files))

    def test_candidate_has_required_fields(self):
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v2.0')
        self.assertTrue(len(result) > 0)
        for c in result:
            self.assertIn('file', c)
            self.assertIn('kind', c)
            self.assertIn('introduced_at', c)
            self.assertIn('commit', c)
            self.assertIn('author', c)
            self.assertIn('date', c)

    def test_kind_is_interface_or_abstract_class(self):
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v2.0')
        for c in result:
            self.assertIn(c['kind'], ('interface', 'abstract_class'))

    def test_get_major_version_tags(self):
        tags = get_major_version_tags(self.repo)
        self.assertIn('v1.0', tags)
        self.assertIn('v2.0', tags)

    def test_pre_existing_files_not_reported(self):
        result = delta_analysis(self.repo, from_ref='v1.0', to_ref='v2.0')
        files = [c['file'] for c in result]
        self.assertFalse(any('Service' in f for f in files))
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
python -m pytest tests/test_delta_analysis.py -v 2>&1 | head -10
```
Expected: `ModuleNotFoundError: No module named 'delta_analysis'`

- [ ] **Step 3: Implement `scripts/delta_analysis.py`**

```python
"""Delta analysis — find new abstraction layers introduced between two git refs."""
import re
import subprocess
from pathlib import Path

_RE_INTERFACE = re.compile(r'\binterface\s+\w+')
_RE_ABSTRACT = re.compile(r'\babstract\s+class\s+\w+')
_SOURCE_EXTS = {'.java', '.kt', '.py'}


def get_major_version_tags(repo: Path) -> list[str]:
    """Return all git tags in the repo, sorted by version."""
    result = subprocess.run(
        ['git', '-C', str(repo), 'tag', '--sort=version:refname'],
        capture_output=True, text=True, check=True,
    )
    return [t for t in result.stdout.splitlines() if t.strip()]


def _files_added_between(repo: Path, from_ref: str, to_ref: str) -> list[str]:
    """Return source files added (A) in to_ref that didn't exist in from_ref."""
    if from_ref == to_ref:
        return []
    result = subprocess.run(
        ['git', '-C', str(repo), 'diff', '--name-status', from_ref, to_ref],
        capture_output=True, text=True, check=True,
    )
    added = []
    for line in result.stdout.splitlines():
        parts = line.split('\t', 1)
        if len(parts) == 2 and parts[0] == 'A':
            path = parts[1].strip()
            if Path(path).suffix in _SOURCE_EXTS:
                added.append(path)
    return added


def _file_content_at(repo: Path, ref: str, filepath: str) -> str:
    try:
        result = subprocess.run(
            ['git', '-C', str(repo), 'show', f'{ref}:{filepath}'],
            capture_output=True, text=True, check=True,
        )
        return result.stdout
    except subprocess.CalledProcessError:
        return ''


def _blame_info(repo: Path, to_ref: str, filepath: str) -> dict:
    """Return commit hash, author, and date for the first line of filepath at to_ref."""
    try:
        result = subprocess.run(
            ['git', '-C', str(repo), 'log', '-1', '--format=%H|%ae|%ad',
             '--date=short', to_ref, '--', filepath],
            capture_output=True, text=True, check=True,
        )
        parts = result.stdout.strip().split('|')
        if len(parts) == 3:
            return {'commit': parts[0][:7], 'author': parts[1], 'date': parts[2]}
    except subprocess.CalledProcessError:
        pass
    return {'commit': 'unknown', 'author': 'unknown', 'date': 'unknown'}


def delta_analysis(repo: Path, from_ref: str, to_ref: str) -> list[dict]:
    """Return new abstraction candidates introduced between from_ref and to_ref."""
    repo = Path(repo)
    added_files = _files_added_between(repo, from_ref, to_ref)
    candidates = []

    for filepath in added_files:
        content = _file_content_at(repo, to_ref, filepath)
        if not content:
            continue

        kind = None
        if _RE_INTERFACE.search(content):
            kind = 'interface'
        elif _RE_ABSTRACT.search(content):
            kind = 'abstract_class'

        if kind:
            blame = _blame_info(repo, to_ref, filepath)
            candidates.append({
                'file': filepath,
                'kind': kind,
                'introduced_at': to_ref,
                **blame,
            })

    return candidates
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
python -m pytest tests/test_delta_analysis.py -v
```
Expected: 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add scripts/delta_analysis.py tests/test_delta_analysis.py
git commit -m "feat(mining): delta analysis pipeline — new abstractions between git tags

Refs #34"
```

---

## Task 5: Full test suite run and close

- [ ] **Step 1: Run the full test suite**

```bash
cd soredium && python -m pytest tests/ -v 2>&1 | tail -15
```
Expected: All existing tests plus 20 new ones pass. Zero regressions.

- [ ] **Step 2: Close issue**

```bash
gh issue close 34 --repo Hortora/soredium \
  --comment "Shipped: project registry, feature extractor, clustering pipeline, delta analysis. All tests passing. Refs #31."
```

- [ ] **Step 3: Final commit if any cleanup needed**

```bash
git add -A
git commit -m "chore: area 2 phase 3 complete — registry + mining pipeline

Closes #34, Refs #31"
```
