# Garden Engine — TDD Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement
> this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Full test suite for `garden-engine` — every component covered at every level
before production code is written. The Python pipeline's 838 tests are the acceptance
criteria for the Java port; each Python test has a Java equivalent.

**Iron law:** No production code without a failing test first.
Delete any code written before its test. No exceptions.

**Issue:** Refs #39

---

## Test Taxonomy

| Layer | What it tests | Real deps? | Count target |
|---|---|---|---|
| Unit | Single class, no I/O | No | ~120 |
| Integration | Component + real filesystem/git | filesystem + git | ~40 |
| AI service | LLM service interfaces | Mocked LLM | ~30 |
| @QuarkusTest | Full CDI context, Qdrant DevServices | Qdrant + CDI | ~25 |
| E2E / happy path | Full pipeline CLI commands | All real | ~15 |
| Correctness | Known input → exact output | Varies | ~30 |
| Robustness | Edge cases, failures, bad inputs | Varies | ~40 |

**Total target: ~300 tests**

---

## Test Infrastructure

### Shared fixtures

```java
// TestFixtures.java — synthetic source trees used across multiple tests
public class TestFixtures {

    /** Writes a minimal Java source tree to a temp dir. */
    public static Path javaProjectWithInterfaces(Path root, int interfaces, int beans) { ... }

    /** Creates a git repo with two tagged versions (v1.0 adds plain class, v2.0 adds interface). */
    public static Path gitRepoWithTwoVersions(Path root) { ... }

    /** Returns a minimal valid projects.yaml content string. */
    public static String projectsYaml(String... projectNames) { ... }

    /** Returns two garden entries that are semantically identical (duplicate). */
    public static String[] duplicateEntryPair() { ... }

    /** Returns two garden entries that are semantically related but distinct. */
    public static String[] relatedEntryPair() { ... }

    /** Returns two garden entries with no semantic relationship. */
    public static String[] distinctEntryPair() { ... }
}
```

### AI service mock

```java
// MockReasoningService.java — injectable mock for all @RegisterAiService interfaces
@Alternative @Priority(1)
@ApplicationScoped
public class MockReasoningService
    implements PatternNamingService, DeltaNarrativeService,
               DedupeClassifier, EntryMergeService {

    private PatternCandidate nextPattern;
    private DeltaNarrative nextNarrative;
    private DedupeDecision nextDecision;
    private String nextMerge;

    // Configurable canned responses — set in @BeforeEach
    public void willReturnPattern(PatternCandidate p) { this.nextPattern = p; }
    public void willReturnDecision(DedupeDecision d) { this.nextDecision = d; }
    // ...
}
```

---

## Task 1 — FeatureExtractor unit tests

**File:** `tests/java/FeatureExtractorTest.java`

### Unit tests

- [ ] **empty directory returns zero counts**
  ```java
  var fp = extractor.extract(emptyDir);
  assertThat(fp.interfaceCount()).isZero();
  assertThat(fp.fileCount()).isZero();
  assertThat(fp.abstractionDepth()).isEqualTo(0.0);
  ```

- [ ] **counts interface declarations in .java files**
  ```java
  write(root, "Foo.java", "public interface Foo {}");
  write(root, "Bar.java", "public interface Bar extends Foo {}");
  assertThat(extractor.extract(root).interfaceCount()).isEqualTo(2);
  ```

- [ ] **counts abstract class declarations**
  ```java
  write(root, "Base.java", "public abstract class Base {}");
  assertThat(extractor.extract(root).interfaceCount()).isEqualTo(1);
  ```

- [ ] **counts CDI injection annotations**
  ```java
  write(root, "A.java", "@ApplicationScoped\npublic class A {\n  @Inject Foo f;\n}");
  assertThat(extractor.extract(root).injectionPoints()).isEqualTo(2);
  ```

- [ ] **counts extends/implements signatures**
  ```java
  write(root, "A.java", "public class A extends B implements C, D {}");
  assertThat(extractor.extract(root).extensionSignatures()).isEqualTo(1);
  ```

- [ ] **counts META-INF/services SPI files**
  ```java
  write(root, "META-INF/services/com.example.Foo", "com.example.impl.FooImpl\n");
  assertThat(extractor.extract(root).spiPatterns()).isEqualTo(1);
  ```

- [ ] **ignores non-source files (.md, .xml, .txt)**
  ```java
  write(root, "README.md", "# interface Foo");
  write(root, "pom.xml", "<interface name='Foo'/>");
  assertThat(extractor.extract(root).interfaceCount()).isZero();
  ```

- [ ] **ignores .class and binary files** — no NPE, count stays 0

- [ ] **abstraction depth is interface_count / file_count**
  ```java
  // 2 interfaces, 4 total files → depth = 0.5
  assertThat(fp.abstractionDepth()).isCloseTo(0.5, offset(0.001));
  ```

- [ ] **abstraction depth is 0.0 for zero files** — no division by zero

- [ ] **Kotlin .kt files counted as source**
  ```java
  write(root, "Foo.kt", "interface Foo");
  assertThat(extractor.extract(root).fileCount()).isEqualTo(1);
  ```

- [ ] **files with bad encoding are skipped without exception**
  ```java
  Files.write(root.resolve("Bad.java"), new byte[]{(byte)0xFF, (byte)0xFE});
  assertDoesNotThrow(() -> extractor.extract(root));
  ```

### Correctness tests

- [ ] **known Quarkus-like project produces expected ratio fingerprint** — a synthetic
  tree with 10 interfaces, 40 injection points, 50 files → exact ratio values match
  Python pipeline output for same input

- [ ] **injection ratio matches Python feature_extractor.py** — port the Python
  `test_counts_injection_points` fixture exactly; Java and Python must agree

### Robustness tests

- [ ] **symlinked directory does not cause infinite traversal** — symlink pointing
  to parent; walk terminates without StackOverflowError

- [ ] **permission-denied file is skipped** — chmod 000; no exception propagates

- [ ] **very large source tree (10k files) completes in < 10 seconds**

- [ ] **concurrent calls to extract() on different roots are thread-safe**

---

## Task 2 — ClusterPipeline unit tests

**File:** `tests/java/ClusterPipelineTest.java`

### Unit tests

- [ ] **fewer than 2 projects returns empty list**

- [ ] **identical fingerprints form a cluster**

- [ ] **dissimilar fingerprints do not cluster**

- [ ] **all clusters have at least 2 members**

- [ ] **centroid values are the mean of cluster member raw fingerprints**

- [ ] **similarity score is between 0 and 1**

- [ ] **known pattern match is tagged when centroid cosine ≥ threshold**

- [ ] **no known pattern match returns null for matches_known_pattern**

### Correctness tests (ratio normalisation — issue #36 fix must hold)

- [ ] **small project (6 files) and large project (21k files) do NOT cluster at 0.9 threshold**
  — the regression test for the normalisation bug. Without ratio conversion,
  these would score ~0.999. With ratios they score <0.5.

- [ ] **two projects with identical injection-to-file ratios DO cluster** — even if
  raw counts differ by 100x

- [ ] **quarkus-like vs hibernate-like fingerprints are NOT clustered** — quarkus has
  injection_ratio ~0.4, hibernate has ~0.002; they should not cluster at 0.85 threshold

### Robustness tests

- [ ] **all-zero fingerprint (empty project) does not cause division by zero**

- [ ] **single-project set returns empty list without exception**

- [ ] **100-project set completes in < 1 second** — clustering is O(n²), must not degrade

---

## Task 3 — DeltaAnalysis unit tests

**File:** `tests/java/DeltaAnalysisTest.java`  
Uses a synthetic git repo created in `@BeforeAll`.

### Unit tests

- [ ] **no tags returns empty candidate list**

- [ ] **single tag returns empty candidate list** — need at least 2 for a diff

- [ ] **new interface file between v1.0 and v2.0 is detected**

- [ ] **new abstract class between versions is detected**

- [ ] **pre-existing files not in the delta are excluded**

- [ ] **candidate has all required fields: file, kind, introduced_at, commit, author, date**

- [ ] **kind is exactly "interface" or "abstract_class"**

- [ ] **from_ref == to_ref returns empty list** — no self-diff

- [ ] **get_major_version_tags returns tags in version order**

### Correctness tests

- [ ] **synthetic repo: v1.0 has 1 class, v2.0 adds 1 interface, 1 abstract class**
  → exactly 2 candidates, correct kinds, correct file paths

- [ ] **git blame info matches the commit that added the file**

### Robustness tests

- [ ] **shallow clone detected and returns empty with warning logged** — the fix from #37

- [ ] **repo with 50 tags between two major versions completes in < 5 seconds**

- [ ] **non-Java files added between versions are not returned**

- [ ] **deleted files between versions do not appear as candidates**

---

## Task 4 — ProjectRegistry unit tests

**File:** `tests/java/ProjectRegistryTest.java`

### Unit tests (port from Python `test_project_registry.py`)

- [ ] **empty registry lists nothing**
- [ ] **add project persists across instances**
- [ ] **add duplicate raises exception**
- [ ] **update last_processed_commit persists**
- [ ] **get unknown project returns empty Optional**
- [ ] **required fields validated on add** — missing field throws

### Correctness tests

- [ ] **YAML round-trip preserves all fields** — write, reload, compare field by field

- [ ] **projects.yaml with 5 entries loads all 5 in correct order**

### Robustness tests

- [ ] **corrupt YAML throws descriptive exception**, not NullPointerException

- [ ] **missing file throws FileNotFoundException**, not silent empty list

- [ ] **concurrent add() calls from two threads do not corrupt file** — last-write-wins
  is acceptable, but no partial write

---

## Task 5 — AI Service unit tests (mocked LLM)

**File:** `tests/java/PatternNamingServiceTest.java` etc.

These tests verify that:
1. The prompt construction is correct (right context passed to LLM)
2. The response parsing handles valid JSON
3. The response parsing handles malformed JSON gracefully

### PatternNamingService

- [ ] **prompt includes all cluster member project names**

- [ ] **prompt includes fingerprint values for each member**

- [ ] **valid JSON response is parsed into PatternCandidate record**

- [ ] **malformed JSON response throws descriptive exception**, not ClassCastException

- [ ] **empty cluster context returns null candidate**, not exception

### DedupeClassifier

- [ ] **prompt includes both entry titles and bodies**

- [ ] **DISTINCT classification parsed correctly**

- [ ] **RELATED classification parsed correctly**

- [ ] **DUPLICATE classification with keep_id parsed correctly**

- [ ] **malformed JSON response → classification defaults to DISTINCT** (safe fallback)

### EntryMergeService

- [ ] **prompt includes both full entry texts**

- [ ] **merged output contains YAML frontmatter block**

- [ ] **merged output contains required sections (title, stack, body)**

- [ ] **score in merged output is the maximum of both input scores**

### DeltaNarrativeService

- [ ] **prompt includes the git diff text**

- [ ] **valid JSON response parsed into DeltaNarrative record**

- [ ] **introduced_at field matches the to_ref passed in**

---

## Task 6 — @QuarkusTest integration tests (CDI + Qdrant DevServices)

**File:** `tests/java/PipelineQuarkusTest.java`

Quarkus DevServices auto-starts Qdrant. MockReasoningService replaces real AI services.

### Happy path tests

- [ ] **mine command with 1 synthetic project produces a CandidateReport**

- [ ] **fingerprint is indexed into Qdrant after mine completes**

- [ ] **Qdrant search finds indexed fingerprint by semantic similarity**

- [ ] **harvest command with 2 duplicate entries produces a merged entry**

- [ ] **harvest command with 2 distinct entries produces See Also links only**

- [ ] **harvest command with 2 unrelated entries produces no output**

### Integration correctness tests

- [ ] **mine → harvest → Qdrant contains merged entry** (full loop with DevServices)

- [ ] **re-running mine on same project updates last_processed_commit in registry**

- [ ] **known rejection suppresses duplicate cluster candidate**

### Robustness tests (QuarkusTest)

- [ ] **Qdrant unavailable at startup → circuit breaker opens, mine still runs (no crash)**

- [ ] **AI service returns error → harvest logs warning, skips pair, continues**

- [ ] **mine with 0 projects in registry produces empty report, no exception**

- [ ] **harvest on empty garden produces no output, no exception**

---

## Task 7 — CLI command tests

**File:** `tests/java/MineCommandTest.java`, `HarvestCommandTest.java`, `QECommandTest.java`

Use Quarkus `@QuarkusMainTest` / `@TestProfile` for CLI mode.

### mine command

- [ ] **`mine --all` with populated registry runs without error**

- [ ] **`mine --project unknown` prints "project not found" and exits 1**

- [ ] **`mine --all` writes candidate_report.json to configured path**

- [ ] **`mine --all --profile=sonnet` activates Anthropic AI service** (verify via
  mock that Anthropic client is called, not Ollama)

### harvest command

- [ ] **`harvest --sweep` processes all unchecked pairs**

- [ ] **`harvest --sweep` with no unchecked pairs exits 0 cleanly**

- [ ] **`harvest --sweep --dry-run` logs decisions without writing files**

### qe command

- [ ] **`qe --sample=5 --tasks=pattern_naming` produces report with 5 rows**

- [ ] **`qe` report shows per-task agreement rates**

- [ ] **`qe` report marks tasks above 90% threshold as "promote"**

- [ ] **`qe` with `--compare=sonnet` uses Anthropic QualityEvaluator**

---

## Task 8 — Correctness tests (Python equivalence)

**File:** `tests/java/PythonEquivalenceTest.java`

Each Python test fixture → identical Java test. These are the acceptance criteria
for the port. Both pipelines must produce the same output for the same input.

- [ ] **FeatureExtractor: all 7 Python test_feature_extractor.py scenarios pass**

- [ ] **ClusterPipeline: all 6 Python test_cluster_pipeline.py scenarios pass**

- [ ] **DeltaAnalysis: all 7 Python test_delta_analysis.py scenarios pass**

- [ ] **ProjectRegistry: all 6 Python test_project_registry.py scenarios pass**

- [ ] **RejectionRegistry: all Python test_rejection_registry.py scenarios pass**

- [ ] **CandidateReport: all Python test_candidate_report.py scenarios pass**

For each: given the same synthetic input, Java and Python produce numerically
identical output (fingerprint values, cluster membership, delta candidates).

---

## Task 9 — End-to-end tests (real repos, real AI)

**File:** `tests/java/EndToEndTest.java`  
`@QuarkusIntegrationTest` (starts the native image). Requires:
- Ollama running with `qwen3.6:35b-a3b` (or `--profile=mock-ai` for CI)
- Real garden clone at `~/.hortora/garden`

### Happy path (with mock AI for CI)

- [ ] **mine incus-spawn → CandidateReport with non-null fingerprint**

- [ ] **mine 5 projects → cluster candidates present** (ratio-normalized, no spurious small-project clusters)

- [ ] **harvest on garden with 3 known duplicate pairs → 3 merged entries committed**

### Happy path (with real AI — manual/nightly only)

- [ ] **PatternNamingService names the CDI vs service-loader pattern correctly** — given
  the quarkus/hibernate-orm fingerprints, expect name containing "injection" or "service-loader"

- [ ] **DedupeClassifier correctly classifies the 3 entry pairs from TestFixtures**

- [ ] **QE report shows ≥ 80% agreement between Ollama and Sonnet** — baseline measurement

### E2E robustness

- [ ] **mine on shallow clone logs warning and skips delta analysis**, report still produced

- [ ] **mine when Ollama is down falls back gracefully** — cluster candidates present,
  no pattern names, no crash

- [ ] **harvest when Qdrant is down logs error and exits 1 with clear message**

---

## Task 10 — Robustness / edge case matrix

**File:** `tests/java/RobustnessTest.java`

### Input edge cases

- [ ] **Java file containing only comments — no counts, no exception**

- [ ] **Java file with interface in a string literal — not counted**
  ```java
  String s = "public interface Foo {}"; // should not match
  ```

- [ ] **Deeply nested directory structure (20 levels) — traversed correctly**

- [ ] **Project with 0 .java files but valid META-INF/services — spiPatterns=1, others=0**

- [ ] **Project where every file is an interface — abstraction depth = 1.0**

- [ ] **Registry with 100 projects — list() returns all 100**

### Failure recovery

- [ ] **git subprocess times out (ProcessBuilder with 30s limit) → exception with message,
  not hang indefinitely**

- [ ] **Qdrant index times out → CandidateReport still written with structural candidates**

- [ ] **AI merge produces invalid YAML → entry NOT written, error logged, original
  entries preserved**

- [ ] **AI classification returns unexpected JSON field → parsed with defaults, no NPE**

- [ ] **Registry file deleted mid-run → IOException caught, pipeline continues for other projects**

### Concurrency

- [ ] **Two mine processes for different projects run concurrently without registry corruption**

- [ ] **Harvest acquires file lock; second harvest process waits or exits cleanly**

---

## Test execution matrix

| Test class | Runs in CI | Requires Ollama | Requires real garden |
|---|---|---|---|
| FeatureExtractorTest | ✅ | ❌ | ❌ |
| ClusterPipelineTest | ✅ | ❌ | ❌ |
| DeltaAnalysisTest | ✅ | ❌ | ❌ |
| ProjectRegistryTest | ✅ | ❌ | ❌ |
| AI Service tests | ✅ (mocked) | ❌ | ❌ |
| PipelineQuarkusTest | ✅ (DevServices) | ❌ | ❌ |
| CLI command tests | ✅ (mock AI) | ❌ | ❌ |
| PythonEquivalenceTest | ✅ | ❌ | ❌ |
| EndToEndTest (mock AI) | ✅ | ❌ | ✅ |
| EndToEndTest (real AI) | ❌ nightly | ✅ | ✅ |
| RobustnessTest | ✅ | ❌ | ❌ |

CI runs all tests except "real AI" group. Nightly runs the full matrix.

---

## Definition of Done

The garden-engine is implementation-ready when:

- [ ] All ~300 tests are written and each watched to **fail** before implementation
- [ ] All tests pass with minimal production code
- [ ] Python equivalence tests pass — Java and Python agree on all shared fixtures  
- [ ] No test passes immediately after being written (would mean testing existing behavior)
- [ ] MockReasoningService covers all AI service interfaces
- [ ] Test execution matrix above is satisfied
- [ ] All commits reference `#39`
