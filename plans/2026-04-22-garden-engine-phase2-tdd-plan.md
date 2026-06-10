# Garden Engine Phase 2 — TDD Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement
> this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire Langchain4j 1.9.1 into the garden-engine for real LLM inference
(Ollama default, Anthropic Sonnet via profile), add Qdrant vector indexing for
fingerprints and semantic dedup, enable the 4 currently @Disabled Qdrant tests.

**Iron law:** No production code without a failing test first.

**Issue:** Refs #39

**Infrastructure constraints:**
- Podman 5.7.1 available → Quarkus DevServices uses it for Qdrant auto-start
- Ollama NOT installed → real inference tests are `@Disabled` with clear message
- Anthropic API → tests are `@Disabled` unless `ANTHROPIC_API_KEY` is set
- Langchain4j Quarkus extension: 1.9.1

**Baseline:** 136 tests passing, 6 @Disabled, 0 failures

---

## What changes in Phase 2

### AI service layer

Phase 1 `PatternNamingServiceImpl` etc. parse JSON passed directly as the "context"
(a test-only design). Phase 2 replaces them with real `@RegisterAiService` interfaces
where Langchain4j handles prompt dispatch and response parsing.

**Before (Phase 1):**
```java
// Caller passes JSON, impl parses it (test-only pattern)
public class PatternNamingServiceImpl implements PatternNamingService {
    public PatternCandidate namePattern(String clusterContext) {
        return JSON.readValue(clusterContext, PatternCandidate.class);
    }
}
```

**After (Phase 2):**
```java
// Langchain4j creates the impl via @RegisterAiService
@RegisterAiService
@SystemMessage("You are an expert in JVM framework architecture...")
public interface PatternNamingService {
    @UserMessage("Cluster context: {clusterContext}")
    PatternCandidate namePattern(String clusterContext);
}
```

The `MockReasoningService` continues to work via `@Alternative @Priority(1)` for all
non-AI tests. The Impl classes are deleted (replaced by Langchain4j's proxy).

### Qdrant integration

Add a `FingerprintIndexer` service that indexes project fingerprints into Qdrant
collection `mining-fingerprints`. The `ClusterPipeline` gains an optional Qdrant
search path: when Qdrant is available, use semantic nearest-neighbour; fall back to
cosine similarity when it's not.

---

## Task 1 — Dependencies + Langchain4j wiring

**Files:** `pom.xml`, AI service interfaces, delete Impl classes

### Tests to write first

Create `LangchainWiringTest.java` — tests that CDI wiring is correct after adding
the Langchain4j extensions. These use `MockReasoningService` (no real LLM needed).

- [ ] **CDI alternative is active — MockReasoningService resolves as PatternNamingService**
  ```java
  @QuarkusTest
  class LangchainWiringTest {
      @Inject PatternNamingService namingService;
      @Inject DedupeClassifier classifier;
      @Inject EntryMergeService mergeService;
      @Inject DeltaNarrativeService narrativeService;

      @Test
      void mockReasoningServiceResolvesAsPatternNamingService() {
          // MockReasoningService is @Alternative @Priority(1) — should be injected
          assertThat(namingService).isNotNull();
          var result = namingService.namePattern("any context");
          assertThat(result.name()).isEqualTo("mock-pattern"); // from MockReasoningService
      }
  }
  ```

- [ ] **All four AI services are injectable and return mock values**
  ```java
  @Test
  void allAiServicesInjectableWithMock() {
      assertThat(classifier.classify("any pair").classification())
          .isEqualTo(DedupeDecision.Classification.DISTINCT);
      assertThat(mergeService.mergeEntries("any pair")).contains("---");
      assertThat(narrativeService.explainDelta("any diff").decision()).isNotBlank();
  }
  ```

- [ ] **MockReasoningService configurable values persist across calls**
  ```java
  @Inject MockReasoningService mock;

  @Test
  void mockReasoningServiceConfigurableValues() {
      mock.willReturnPattern(new PatternCandidate("custom", "d", "s", "w"));
      assertThat(namingService.namePattern("ctx").name()).isEqualTo("custom");
  }
  ```

### Implementation steps

- [ ] **Step 1: Add dependencies to pom.xml**

```xml
<!-- Langchain4j BOM — manages all quarkiverse.langchain4j versions -->
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-bom</artifactId>
    <version>1.9.1</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

Add in `<dependencyManagement>` and then add individual artifacts:
```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-ollama</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-anthropic</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-qdrant</artifactId>
</dependency>
```

- [ ] **Step 2: Annotate AI service interfaces with @RegisterAiService**

PatternNamingService:
```java
@RegisterAiService
public interface PatternNamingService {

    @SystemMessage("""
        You are an expert in JVM framework architecture. Given a cluster of structurally
        similar projects and their fingerprints, identify the shared architectural pattern.
        Respond as JSON: {"name":"...","description":"...","structural_signal":"...","why_it_exists":"..."}
        """)
    PatternCandidate namePattern(@UserMessage String clusterContext);
}
```

DedupeClassifier:
```java
@RegisterAiService
public interface DedupeClassifier {

    @SystemMessage("""
        You are reviewing two knowledge garden entries for duplication.
        Classify as DISTINCT, RELATED, or DUPLICATE.
        Respond as JSON: {"classification":"...","reasoning":"...","keep_id":null,"preserve_from_other":null}
        """)
    DedupeDecision classify(@UserMessage String entryPair);
}
```

EntryMergeService:
```java
@RegisterAiService
public interface EntryMergeService {

    @SystemMessage("""
        Merge two duplicate knowledge garden entries into one enriched entry.
        Preserve the best elements of both. Output full YAML frontmatter + body.
        """)
    String mergeEntries(@UserMessage String entryPair);
}
```

DeltaNarrativeService:
```java
@RegisterAiService
public interface DeltaNarrativeService {

    @SystemMessage("""
        You are an expert in JVM framework architecture. Given a git diff showing
        new interface or abstract class files, explain the architectural decision made.
        Respond as JSON: {"decision":"...","pattern_name":"...","motivation":"...","introduced_at":"..."}
        """)
    DeltaNarrative explainDelta(@UserMessage String diffContext);
}
```

- [ ] **Step 3: Delete the Phase 1 Impl classes**

Delete:
- `ai/PatternNamingServiceImpl.java`
- `ai/DedupeClassifierImpl.java`
- `ai/EntryMergeServiceImpl.java`
- `ai/DeltaNarrativeServiceImpl.java`

These are replaced by Langchain4j's `@RegisterAiService` proxy. **Check that all
17 Phase 1 AI service tests still pass** — they test behaviour through the interface,
not the impl, and MockReasoningService provides the backing.

Actually: the Phase 1 AI service tests instantiate the Impl directly with `new`.
Those tests must be updated to use the interface via CDI (@QuarkusTest + @Inject)
or the tests must be rewritten. See Task 3 for test updates.

- [ ] **Step 4: application.properties — Ollama configuration**

```properties
# Default reasoning model — local, free, no API key
quarkus.langchain4j.ollama.base-url=http://localhost:11434
quarkus.langchain4j.ollama.chat-model.model-id=qwen3.6:35b-a3b
quarkus.langchain4j.ollama.chat-model.temperature=0.2
quarkus.langchain4j.ollama.log-requests=true
quarkus.langchain4j.ollama.log-responses=true

# Disable timeout for large models
quarkus.langchain4j.ollama.timeout=120s
```

`application-sonnet.properties` (activated with -Dquarkus.profile=sonnet):
```properties
quarkus.langchain4j.anthropic.api-key=${ANTHROPIC_API_KEY}
quarkus.langchain4j.anthropic.model-name=claude-sonnet-4-6
quarkus.langchain4j.anthropic.max-tokens=4096
```

`src/test/resources/application.properties`:
```properties
# In tests — disable Ollama connection (MockReasoningService takes over via CDI)
quarkus.langchain4j.ollama.base-url=http://localhost:1
quarkus.langchain4j.devservices.enabled=false
```

- [ ] **Step 5: Run LangchainWiringTest — all 3 tests pass**

```bash
mvn test -Dtest=LangchainWiringTest 2>&1 | grep -E "Tests run|BUILD"
```

- [ ] **Step 6: Run full suite — 139 tests passing** (136 + 3 new)

```bash
mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
```

- [ ] **Step 7: Commit**
```bash
git commit -m "feat(garden-engine): Phase 2 — Langchain4j wiring + @RegisterAiService

Add Langchain4j 1.9.1 (ollama, anthropic, qdrant). Convert AI service
interfaces to @RegisterAiService. Remove manual JSON Impl classes —
replaced by Langchain4j proxy. MockReasoningService @Alternative
continues to satisfy all CDI injection in tests. Ollama configured
as default; Anthropic via -Dquarkus.profile=sonnet.

Refs #39"
```

---

## Task 2 — Qdrant indexing service + DevServices tests

**Files:** New `FingerprintIndexer.java`, updated `PipelineQuarkusTest.java`

### Tests to write first (Podman DevServices — CI compatible)

Enable the 4 currently @Disabled Qdrant tests in `PipelineQuarkusTest.java` and
add new indexing tests. DevServices auto-starts Qdrant via Podman.

- [ ] **fingerprintIsIndexedIntoQdrantAfterMineCompletes**
  ```java
  @Inject FingerprintIndexer indexer;
  @Inject QdrantClient qdrant;  // or EmbeddingStore

  @Test
  void fingerprintIsIndexedIntoQdrantAfterMineCompletes(@TempDir Path dir) throws IOException {
      TestFixtures.javaProjectWithInterfaces(dir, 5, 10);
      var fp = extractor.extract(dir);
      indexer.index("test-project", fp);

      var results = indexer.findSimilar(fp, 5);
      assertThat(results).isNotEmpty();
      assertThat(results.get(0).projectName()).isEqualTo("test-project");
  }
  ```

- [ ] **qdrantSearchFindsSimilarFingerprints**
  ```java
  @Test
  void qdrantSearchFindsSimilarFingerprints() {
      var fp1 = new Fingerprint(10, 0.2, 8, 12, 50, 2);
      var fp2 = new Fingerprint(11, 0.22, 9, 11, 55, 2); // similar to fp1
      var fp3 = new Fingerprint(1, 0.01, 0, 1, 200, 0); // dissimilar

      indexer.index("project-a", fp1);
      indexer.index("project-b", fp2);
      indexer.index("project-c", fp3);

      var similar = indexer.findSimilar(fp1, 2);
      // project-a and project-b should be most similar
      assertThat(similar.stream().map(SimilarProject::projectName).toList())
          .contains("project-a", "project-b");
  }
  ```

- [ ] **mineAndHarvestWithKnownDuplicates** (previously @Disabled)
  ```java
  @Test
  @Disabled("Requires semantic dedup pipeline — Phase 4")
  void harvestWithDuplicateEntriesProducesMergedEntry() { }
  ```
  Keep @Disabled but update the message — this moves to Phase 4 (semantic harvest).

- [ ] **mineToHarvestFullLoop** (previously @Disabled)
  ```java
  @Test
  @Disabled("Requires semantic dedup pipeline — Phase 4")
  void mineToHarvestFullLoop() { }
  ```

- [ ] **qdrantUnavailableAtStartupCircuitBreakerOpens**
  ```java
  @Test
  void mineCompletesEvenWhenQdrantIndexingFails(@TempDir Path dir) throws IOException {
      // indexer gracefully handles Qdrant failures
      TestFixtures.javaProjectWithInterfaces(dir, 3, 5);
      var fp = extractor.extract(dir);
      // Simulate Qdrant unavailable — indexer should log warning, not throw
      assertDoesNotThrow(() -> indexer.indexSafely("project", fp));
  }
  ```

### FingerprintIndexer implementation

```java
@ApplicationScoped
public class FingerprintIndexer {

    record SimilarProject(String projectName, double score) {}

    // Inject Qdrant EmbeddingStore when available
    // For now: in-memory fallback when Qdrant is unavailable
    
    public void index(String projectName, Fingerprint fp) {
        // Convert fingerprint to text for embedding
        var text = buildFingerprintText(projectName, fp);
        // Index into Qdrant collection mining-fingerprints
        // Payload: {project: projectName, ...fp fields}
    }

    public void indexSafely(String projectName, Fingerprint fp) {
        try { index(projectName, fp); }
        catch (Exception e) {
            Logger.getLogger(FingerprintIndexer.class).warnf(
                "Failed to index %s — Qdrant may be unavailable: %s", projectName, e.getMessage());
        }
    }

    public List<SimilarProject> findSimilar(Fingerprint fp, int limit) {
        var query = buildFingerprintText("query", fp);
        // Search Qdrant, return top-N by similarity
    }

    private String buildFingerprintText(String name, Fingerprint fp) {
        return String.format(
            "Project %s: %d interfaces per %d files, %d injection points, %d SPI files, " +
            "abstraction depth %.3f, extension signatures %d",
            name, fp.interfaceCount(), fp.fileCount(), fp.injectionPoints(),
            fp.spiPatterns(), fp.abstractionDepth(), fp.extensionSignatures()
        );
    }
}
```

- [ ] **Run Qdrant tests** (Podman must be running):
  ```bash
  mvn test -Dtest=PipelineQuarkusTest 2>&1 | grep -E "Tests run|DISABLED|BUILD"
  ```

- [ ] **Commit** after all Qdrant tests pass.

---

## Task 3 — Update Phase 1 AI service tests for Langchain4j

Phase 1 tests instantiated Impl classes with `new`. Since Impl classes are deleted,
update the 17 Phase 1 tests to either:
A. Use CDI injection via `@QuarkusTest` + `@Inject` (preferred), or
B. Remove tests that only tested Impl-specific JSON parsing (now irrelevant)

### Tests that need updating

- [ ] `PatternNamingServiceTest` — 5 tests that called `new PatternNamingServiceImpl()`
  → Rewrite as `@QuarkusTest` with `@Inject PatternNamingService` + `MockReasoningService`
  → The prompt-construction test becomes: verify `buildClusterContext()` is a static
     method on `PatternNamingService` or `FingerprintIndexer`, not the deleted Impl

- [ ] `DedupeClassifierTest` — 5 tests
  → Rewrite using CDI + MockReasoningService

- [ ] `EntryMergeServiceTest` — 4 tests  
  → The `mergeEntries(null)` and frontmatter validation tests move to MockReasoningService
     configuration tests: "when mock returns value without ---, service behavior is X"

- [ ] `DeltaNarrativeServiceTest` — 3 tests
  → Rewrite using CDI + MockReasoningService

**Net test count after Task 3:** Should be ≥ 136 (may drop slightly if some
Impl-specific tests become irrelevant, but CDI injection tests replace them 1:1)

---

## Task 4 — Ollama availability detection + real AI smoke test

- [ ] **Create `OllamaAvailabilityCondition.java`**

```java
public class OllamaAvailabilityCondition implements ExecutionCondition {

    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext ctx) {
        try {
            var conn = new java.net.URL("http://localhost:11434/api/tags").openConnection();
            conn.setConnectTimeout(1000);
            conn.connect();
            return ConditionEvaluationResult.enabled("Ollama is running");
        } catch (Exception e) {
            return ConditionEvaluationResult.disabled(
                "Ollama not running at localhost:11434 — install and run: ollama pull qwen3.6:35b-a3b");
        }
    }
}
```

Create annotation `@RequiresOllama`:
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(OllamaAvailabilityCondition.class)
public @interface RequiresOllama {}
```

- [ ] **Create `RealAiSmokeTest.java`** — real Ollama inference, skipped if not available

```java
@QuarkusTest
class RealAiSmokeTest {

    @Inject PatternNamingService namingService; // real Langchain4j proxy, not mock

    @Test
    @RequiresOllama
    void patternNamingProducesSensibleOutput() {
        var context = PatternNamingService.buildClusterContext(
            List.of("quarkus", "micronaut"),
            List.of(
                new Fingerprint(5457, 0.253, 8696, 10106, 21570, 192),
                new Fingerprint(1200, 0.24, 1800, 2200, 5000, 45)
            )
        );
        var result = namingService.namePattern(context);
        assertThat(result).isNotNull();
        assertThat(result.name()).isNotBlank();
        assertThat(result.name().length()).isGreaterThan(3);
    }

    @Test
    @RequiresOllama
    void dedupeClassifierCorrectlyClassifiesDuplicates() {
        var pair = String.join("\n\n---\n\n", TestFixtures.duplicateEntryPair());
        var classifier = // inject real DedupeClassifier without mock override
        // Note: MockReasoningService takes over in @QuarkusTest — need a separate test profile
        // Use @TestProfile(NoMockProfile.class) to disable the mock
    }
}
```

Note on the real inference test: `MockReasoningService` is `@Alternative @Priority(1)`
so it always wins in normal `@QuarkusTest`. To test real inference, create a
`@TestProfile(RealAiProfile.class)` that disables the mock:

```java
public class RealAiProfile implements QuarkusTestProfile {
    @Override
    public Set<Class<?>> getEnabledAlternatives() {
        return Set.of(); // don't enable MockReasoningService
    }
}
```

- [ ] **RealAiSmokeTest tests** (all skipped if Ollama not running):
  - `patternNamingProducesSensibleOutput`
  - `dedupeClassifierReturnsDISTINCTForUnrelatedEntries`
  - `entryMergeReturnsMergedYamlWithBothTitles`

---

## Task 5 — Profile switching validation

- [ ] **Test that profile config exists and is loadable**

```java
@QuarkusTest
class ProfileSwitchingTest {

    @Test
    void ollamaIsDefaultProvider() {
        // Verify default config points to Ollama
        var config = ConfigProvider.getConfig();
        var baseUrl = config.getOptionalValue(
            "quarkus.langchain4j.ollama.base-url", String.class);
        // In test profile, URL is localhost:1 (disabled)
        // In production profile, it would be localhost:11434
        assertThat(baseUrl).isPresent();
    }

    @Test
    @DisabledIfEnvironmentVariable(named = "ANTHROPIC_API_KEY", matches = "")
    @DisabledIfSystemProperty(named = "skip.anthropic", matches = "true")
    void sonnetProfileConfiguredWhenApiKeyPresent() {
        // Only runs when ANTHROPIC_API_KEY is set
        var apiKey = System.getenv("ANTHROPIC_API_KEY");
        assertThat(apiKey).isNotBlank();
    }
}
```

---

## Test execution matrix after Phase 2

| Test class | Runs in CI | Requires Ollama | Requires Podman |
|---|---|---|---|
| LangchainWiringTest | ✅ | ❌ (mock) | ❌ |
| PipelineQuarkusTest (Qdrant) | ✅ | ❌ (mock) | ✅ |
| Phase 1 AI tests (updated) | ✅ | ❌ (mock) | ❌ |
| OllamaAvailabilityCondition | ✅ | ❌ | ❌ |
| RealAiSmokeTest | ❌ skipped | ✅ | ❌ |
| ProfileSwitchingTest | ✅ | ❌ | ❌ |

---

## Definition of Done

- [ ] 139+ tests passing (136 baseline + new), 0 failures
- [ ] All Impl classes deleted, replaced by @RegisterAiService
- [ ] Qdrant DevServices tests enabled (Podman required in CI)
- [ ] `@RequiresOllama` annotation exists; real AI tests skip gracefully
- [ ] `--profile=sonnet` config present and loadable
- [ ] All commits reference `#39`
