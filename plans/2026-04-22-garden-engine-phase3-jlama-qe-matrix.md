# Garden Engine Phase 3 — JLama Default + QE Model Comparison Matrix

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans
> to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Switch default inference from Ollama (external daemon) to JLama
(in-process, auto-downloads from HuggingFace). Extend QE command with a model
comparison matrix so you can run the same task through JLama-small, JLama-large,
Ollama, and Sonnet and compare speed + quality side by side.

**Iron law:** No production code without a failing test first.

**Issue:** Refs #39

**Baseline:** 151 tests passing, 5 skipped, 0 failures

---

## Why JLama as default

- No external daemon (no `brew services start ollama`, no `ollama pull`)
- Auto-downloads models from HuggingFace on first use, cached in `~/.jlama/`
- Pure Java + Panama FFM — ships as a Maven dependency
- Structured output via Langchain4j's schema layer (same as Ollama)
- Ollama and Sonnet kept as named profiles for QE and production GPU use

---

## Profile structure after Phase 3

| Profile | Provider | Use case |
|---|---|---|
| default | JLama `Qwen2.5-3B-Instruct-JQ4` | Development, CI (after first download) |
| `-Dquarkus.profile=ollama` | Ollama `qwen3.6:35b-a3b` | GPU-accelerated production |
| `-Dquarkus.profile=sonnet` | Anthropic Claude Sonnet | QE gold standard |

---

## QE matrix design

```
garden-engine qe --matrix --tasks=dedup,pattern [--sample=N]
```

Runs each task through all configured models, reports:

```
QE Matrix Report — 2026-04-22
Tasks: dedup_classification (20), pattern_naming (10)

Model                              | Avg ms | JSON ok | vs Sonnet
-----------------------------------|--------|---------|----------
JLama Qwen2.5-3B-Instruct-JQ4     |   2340 |  95%    |  82% agree
JLama Llama-3.2-1B-Instruct-JQ4   |    890 |  88%    |  74% agree
Ollama qwen3.6:35b-a3b (if avail) |  45000 |  99%    |  91% agree
Claude Sonnet (gold standard)      |   1200 |  100%   | 100%
```

---

## Task 1 — JLama dependency + wiring

### Tests to write first

Create `JlamaWiringTest.java`:

- [ ] **jlamaProviderIsConfiguredAsDefault**
  ```java
  @QuarkusTest
  class JlamaWiringTest {
      @ConfigProperty(name = "quarkus.langchain4j.chat-model.provider")
      String provider;

      @Test
      void jlamaProviderIsConfiguredAsDefault() {
          assertThat(provider).isEqualTo("jlama");
      }
  }
  ```

- [ ] **jlamaModelNameIsConfigured**
  ```java
  @ConfigProperty(name = "quarkus.langchain4j.jlama.chat-model.model-name")
  String modelName;

  @Test
  void jlamaModelNameIsConfigured() {
      assertThat(modelName).contains("Qwen2.5").contains("JQ4");
  }
  ```

- [ ] **mockReasoningServiceStillInterceptsInTests**
  ```java
  @Inject PatternNamingService service;
  @Inject MockReasoningService mock;

  @Test
  void mockReasoningServiceStillInterceptsInTests() {
      mock.willReturnPattern(new PatternCandidate("JLama-test", "d", "s", "w"));
      assertThat(service.namePattern("ctx").name()).isEqualTo("JLama-test");
  }
  ```

- [ ] **ollamaProfileConfigExists**
  ```java
  @Test
  void ollamaProfileConfigFileExists() {
      var path = Path.of("src/main/resources/application-ollama.properties");
      assertThat(path).exists();
      assertThat(Files.readString(path)).contains("ollama");
  }
  ```

### Implementation steps

- [ ] **Step 1: Add JLama dependency to pom.xml**
  ```xml
  <dependency>
      <groupId>io.quarkiverse.langchain4j</groupId>
      <artifactId>quarkus-langchain4j-jlama</artifactId>
  </dependency>
  ```
  Keep Ollama and Anthropic dependencies — they're still needed for profiles.

- [ ] **Step 2: Update application.properties (main)**
  ```properties
  # Default: JLama — in-process, no external daemon
  quarkus.langchain4j.chat-model.provider=jlama
  quarkus.langchain4j.jlama.chat-model.model-name=tjake/Qwen2.5-3B-Instruct-JQ4
  quarkus.langchain4j.jlama.chat-model.temperature=0.2
  quarkus.langchain4j.jlama.chat-model.max-new-tokens=512
  # JLama model cache directory
  quarkus.langchain4j.jlama.cache-directory=${user.home}/.jlama
  ```

- [ ] **Step 3: Create application-ollama.properties**
  ```properties
  quarkus.langchain4j.chat-model.provider=ollama
  quarkus.langchain4j.ollama.base-url=http://localhost:11434
  quarkus.langchain4j.ollama.chat-model.model-id=qwen3.6:35b-a3b
  quarkus.langchain4j.ollama.chat-model.temperature=0.2
  ```

- [ ] **Step 4: Update test application.properties**
  ```properties
  # Tests use MockReasoningService — JLama model never actually loaded
  quarkus.langchain4j.chat-model.provider=jlama
  quarkus.langchain4j.jlama.chat-model.model-name=tjake/TinyLlama-1.1B-Chat-v1.0-Jlama-Q4
  # Disable devservices (no containers needed for basic tests)
  quarkus.langchain4j.devservices.enabled=false
  # Use existing test Ollama URL (connection refused — model never called)
  quarkus.langchain4j.ollama.base-url=http://localhost:1
  ```

  **Note on eager loading:** If JLama tries to download the model at startup
  (even with MockReasoningService in place), set:
  ```properties
  quarkus.langchain4j.jlama.devservices.enabled=false
  ```
  Watch for `FileNotFoundException` or download errors in the test output.
  If they appear, the test config needs adjustment — surface this in the report.

- [ ] **Step 5: Run JlamaWiringTest — all 4 must pass**
  ```bash
  mvn test -Dtest=JlamaWiringTest 2>&1 | grep -E "Tests run|BUILD"
  ```

- [ ] **Step 6: Run full suite — must not regress**
  ```bash
  mvn test 2>&1 | grep -E "Tests run:|BUILD" | tail -5
  ```
  Expected: 155+ passing, 0 failures (151 baseline + 4 new).

- [ ] **Step 7: Commit**
  ```bash
  git commit -m "feat(garden-engine): Phase 3 — JLama as default provider

  Add quarkus-langchain4j-jlama. Switch default from Ollama to JLama
  (in-process, auto-downloads from HuggingFace, no external daemon).
  Default model: tjake/Qwen2.5-3B-Instruct-JQ4.
  Create application-ollama.properties for GPU/production profile.
  MockReasoningService continues to intercept all test CDI calls.

  Refs #39"
  ```

---

## Task 2 — JLama model availability condition

Replaces `@RequiresOllama` in smoke tests. Checks if the model is cached
locally in `~/.jlama/` so tests skip gracefully if the model hasn't been
downloaded yet (avoids surprise 3GB downloads in CI).

### Tests to write first

Create `JlamaAvailabilityTest.java`:

- [ ] **jlamaAvailabilityConditionExistsAndIsAnnotation**
  ```java
  @Test
  void requiresJlamaModelAnnotationIsPresent() {
      assertThat(RequiresJlamaModel.class).isNotNull();
      assertThat(RequiresJlamaModel.class.isAnnotation()).isTrue();
  }
  ```

- [ ] **jlamaModelCachePathIsConfigurable**
  ```java
  @Test
  void jlamaCachePathUsesHomeDir() {
      var checker = new JlamaModelChecker();
      var cache = checker.cacheDir();
      assertThat(cache.toString()).contains(".jlama");
  }
  ```

- [ ] **jlamaModelCheckerReturnsFalseWhenModelNotCached**
  ```java
  @Test
  void checkerReturnsFalseForNonExistentModel() {
      var checker = new JlamaModelChecker();
      assertThat(checker.isModelCached("nonexistent/model-that-does-not-exist"))
          .isFalse();
  }
  ```

- [ ] **jlamaModelCheckerReturnsTrueWhenModelDirExists**
  ```java
  @Test
  void checkerReturnsTrueWhenModelDirExists(@TempDir Path cacheDir) throws IOException {
      Files.createDirectories(cacheDir.resolve("myorg").resolve("MyModel-JQ4"));
      var checker = new JlamaModelChecker(cacheDir);
      assertThat(checker.isModelCached("myorg/MyModel-JQ4")).isTrue();
  }
  ```

### Implementation

- [ ] Create `JlamaModelChecker.java`:
  ```java
  public class JlamaModelChecker {
      private final Path cacheDir;

      public JlamaModelChecker() {
          this(Path.of(System.getProperty("jlama.cache",
              System.getProperty("user.home") + "/.jlama")));
      }

      public JlamaModelChecker(Path cacheDir) { this.cacheDir = cacheDir; }

      public Path cacheDir() { return cacheDir; }

      public boolean isModelCached(String modelName) {
          // modelName format: "org/ModelName" → cache has "org/ModelName/" dir
          return Files.isDirectory(cacheDir.resolve(modelName.replace("/", File.separator)));
      }
  }
  ```

- [ ] Create `RequiresJlamaModel.java` annotation:
  ```java
  @Target(ElementType.METHOD)
  @Retention(RetentionPolicy.RUNTIME)
  @ExtendWith(JlamaModelAvailabilityCondition.class)
  public @interface RequiresJlamaModel {
      String value() default "tjake/Qwen2.5-3B-Instruct-JQ4";
  }
  ```

- [ ] Create `JlamaModelAvailabilityCondition.java`:
  ```java
  public class JlamaModelAvailabilityCondition implements ExecutionCondition {
      @Override
      public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext ctx) {
          var annotation = ctx.getElement()
              .flatMap(e -> Optional.ofNullable(e.getAnnotation(RequiresJlamaModel.class)))
              .orElse(null);
          var modelName = annotation != null ? annotation.value()
              : "tjake/Qwen2.5-3B-Instruct-JQ4";
          var checker = new JlamaModelChecker();
          if (checker.isModelCached(modelName)) {
              return ConditionEvaluationResult.enabled("JLama model cached: " + modelName);
          }
          return ConditionEvaluationResult.disabled(
              "JLama model not cached: " + modelName +
              " — download with: garden-engine download --model " + modelName);
      }
  }
  ```

- [ ] Update `RealAiSmokeTest.java`:
  - Replace `@RequiresOllama` with `@RequiresJlamaModel`
  - Remove Ollama HTTP client — use JLama via Langchain4j instead
  - The smoke test can now use the CDI path (JLama runs in-process)
  - Use `@TestProfile(RealJlamaProfile.class)` to disable MockReasoningService

- [ ] Create `RealJlamaProfile.java` (disables MockReasoningService for real inference):
  ```java
  public class RealJlamaProfile implements QuarkusTestProfile {
      @Override
      public Map<String, String> getConfigOverrides() {
          // Override with real JLama URL (the default, not localhost:1)
          return Map.of("quarkus.langchain4j.jlama.cache-directory",
              System.getProperty("user.home") + "/.jlama");
      }
      // MockReasoningService is @Alternative @Priority(1) — always wins.
      // To test real inference, inject the specific Impl directly by type,
      // bypassing CDI selection. Or accept that smoke tests are Ollama-based.
      // See note below.
  }
  ```

  **Note on CDI override problem:** `MockReasoningService` with `@Alternative @Priority(1)`
  cannot be disabled per-test in Quarkus (no `getDisabledAlternatives()`). The cleanest
  path for real-inference smoke tests is to inject the concrete JLama-generated
  implementation class directly via a qualifier, OR skip CDI and call Langchain4j
  `ChatModel` directly. For Phase 3, inject `ChatModel` directly:

  ```java
  @QuarkusTest
  @TestProfile(RealJlamaProfile.class)
  @Tag("ai-smoke")
  class RealAiSmokeTest {
      @Inject ChatModel chatModel;  // injects JLama ChatModel, not MockReasoningService

      @Test
      @RequiresJlamaModel("tjake/Qwen2.5-3B-Instruct-JQ4")
      void jlamaRespondsToPatternNamingPrompt() {
          var ctx = PatternNamingService.buildClusterContext(List.of("quarkus"), List.of(fp));
          var response = chatModel.chat(ctx);
          assertThat(response).isNotBlank().hasSizeGreaterThan(10);
      }
  }
  ```

- [ ] **Run JlamaAvailabilityTest — all 4 pass**

---

## Task 3 — QE model comparison matrix

### Data model tests (write first)

Create `ModelComparisonResultTest.java`:

- [ ] **resultHasAllRequiredFields**
  ```java
  @Test
  void modelComparisonResultHasAllFields() {
      var result = new ModelComparisonResult(
          "JLama Qwen2.5-3B", "dedup", "DISTINCT", "DISTINCT",
          true, true, 1234L
      );
      assertThat(result.modelName()).isEqualTo("JLama Qwen2.5-3B");
      assertThat(result.taskType()).isEqualTo("dedup");
      assertThat(result.modelOutput()).isEqualTo("DISTINCT");
      assertThat(result.goldStandardOutput()).isEqualTo("DISTINCT");
      assertThat(result.jsonParseSuccess()).isTrue();
      assertThat(result.agreesWithGoldStandard()).isTrue();
      assertThat(result.inferenceMs()).isEqualTo(1234L);
  }
  ```

- [ ] **matrixReportAggregatesCorrectly**
  ```java
  @Test
  void matrixReportComputesAverageInferenceMs() {
      var results = List.of(
          new ModelComparisonResult("Model-A", "dedup", "DISTINCT", "DISTINCT", true, true, 1000L),
          new ModelComparisonResult("Model-A", "dedup", "RELATED", "RELATED", true, true, 2000L),
          new ModelComparisonResult("Model-A", "dedup", "DISTINCT", "DUPLICATE", true, false, 3000L)
      );
      var report = QEMatrixReport.from(results);
      var row = report.rowFor("Model-A", "dedup");
      assertThat(row.avgInferenceMs()).isEqualTo(2000L);
      assertThat(row.jsonParseSuccessRate()).isCloseTo(1.0, offset(0.001));
      assertThat(row.agreementRate()).isCloseTo(0.667, offset(0.001));
  }
  ```

- [ ] **matrixReportFormatsAsTable**
  ```java
  @Test
  void matrixReportRendersTableWithHeaders() {
      var report = QEMatrixReport.from(List.of(...));
      var table = report.renderTable();
      assertThat(table).contains("Model").contains("Avg ms")
                       .contains("JSON ok").contains("vs Sonnet");
  }
  ```

### QEMatrixService tests

Create `QEMatrixServiceTest.java`:

- [ ] **matrixServiceAcceptsModelListAndTaskList**
  ```java
  @Test
  void matrixServiceRunsConfiguredModels() {
      var service = new QEMatrixService(List.of("model-a", "model-b"),
                                         List.of("dedup"), 2);
      assertThat(service.modelNames()).containsExactly("model-a", "model-b");
      assertThat(service.taskTypes()).containsExactly("dedup");
      assertThat(service.sampleSize()).isEqualTo(2);
  }
  ```

- [ ] **matrixServiceReturnsResultsForEachModelTaskCombination**
  - With mock task runners, 2 models × 1 task × 2 samples = 4 results

### QE CLI matrix flag tests

Update `CLICommandTest.java`:

- [ ] **qeMatrixFlagProducesComparisonReport**
  ```java
  @Test
  @Launch({"qe", "--matrix", "--tasks=dedup", "--sample=2"})
  void qeMatrixProducesComparisonReport(LaunchResult result) {
      assertThat(result.exitCode()).isEqualTo(0);
      assertThat(result.getOutput()).contains("Matrix").contains("Model");
  }
  ```

- [ ] **qeMatrixWithoutSonnetShowsPlaceholder**
  ```java
  @Test
  @Launch({"qe", "--matrix", "--sample=1"})
  void qeMatrixRunsWithoutSonnet(LaunchResult result) {
      assertThat(result.exitCode()).isEqualTo(0);
      // Should not crash even without ANTHROPIC_API_KEY
  }
  ```

### Implementation

- [ ] Create `ModelComparisonResult.java` record

- [ ] Create `QEMatrixReport.java` with `from()`, `rowFor()`, `renderTable()`

- [ ] Create `QEMatrixService.java`

- [ ] Update `QECommand.java`:
  ```java
  @CommandLine.Option(names = "--matrix", description = "Compare multiple models side by side")
  boolean matrix;
  ```

- [ ] Update `ModelComparisonResult` and `QEMatrixReport` to include:
  - Model name, task type, model output, gold standard output
  - JSON parse success (boolean), agreement (boolean), inference time (ms)
  - `rowFor(modelName, taskType)` aggregates across samples
  - `renderTable()` produces the aligned table

- [ ] **Run all tests — must not regress**

---

## Task 4 — Update ProfileSwitchingTest for JLama

The existing `ProfileSwitchingTest` checks Ollama as default. Update it for JLama.

- [ ] Update assertions: `defaultProviderIsJlama` (rename from `defaultProviderIsOllama`)
- [ ] Add: `jlamaModelNameContainsQwen`
- [ ] Add: `ollamaProfileConfigFileExists`
- [ ] Verify existing `anthropicApiKeyConfigPropertyExists` test still passes

---

## Definition of Done

- [ ] 155+ tests passing (151 baseline + JlamaWiring + JlamaAvailability + ModelComparison + QEMatrix)
- [ ] Default provider is JLama in `application.properties`
- [ ] `application-ollama.properties` exists
- [ ] `@RequiresJlamaModel` annotation skips tests when model not cached
- [ ] `RealAiSmokeTest` uses `@RequiresJlamaModel` (not `@RequiresOllama`)
- [ ] `qe --matrix` command produces comparison table
- [ ] All commits reference `#39`
- [ ] Full suite still passes with `mvn test` (no model downloads triggered)
