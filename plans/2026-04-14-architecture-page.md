# Architecture Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a dedicated "Architecture" page to hortora.github.io explaining Hortora's six core systems for platform architects evaluating enterprise deployment.

**Architecture:** One new static HTML page (`architecture.html`) matching the existing Jekyll docs pattern, plus sidebar nav updates in four existing pages and a footer next-link added to design-spec.html. No new CSS, no new Jekyll config — all existing styles are reused.

**Tech Stack:** Jekyll (GitHub Pages), HTML, existing CSS classes (`callout`, `code-block`, `lead`, `docs-footer`, `docs-prev`, `docs-next`).

---

## Task 1: Add Architecture to nav and design-spec footer

**Files:**
- Modify: `hortora.github.io/docs/getting-started.html`
- Modify: `hortora.github.io/docs/how-it-works.html`
- Modify: `hortora.github.io/docs/skills-reference.html`
- Modify: `hortora.github.io/docs/design-spec.html`

- [ ] **Step 1: Add Architecture link to getting-started.html nav**

Find this block in `hortora.github.io/docs/getting-started.html`:
```html
        <a href="design-spec.html">Design Spec</a>
      </nav>
```

Replace with:
```html
        <a href="design-spec.html">Design Spec</a>
        <a href="architecture.html">Architecture</a>
      </nav>
```

- [ ] **Step 2: Add Architecture link to how-it-works.html nav**

Find this block in `hortora.github.io/docs/how-it-works.html`:
```html
        <a href="design-spec.html">Design Spec</a>
      </nav>
```

Replace with:
```html
        <a href="design-spec.html">Design Spec</a>
        <a href="architecture.html">Architecture</a>
      </nav>
```

- [ ] **Step 3: Add Architecture link to skills-reference.html nav**

Find this block in `hortora.github.io/docs/skills-reference.html`:
```html
        <a href="design-spec.html">Design Spec</a>
      </nav>
```

Replace with:
```html
        <a href="design-spec.html">Design Spec</a>
        <a href="architecture.html">Architecture</a>
      </nav>
```

- [ ] **Step 4: Add Architecture link to design-spec.html nav and footer**

In `hortora.github.io/docs/design-spec.html`, find:
```html
        <a href="design-spec.html" class="active">Design Spec</a>
      </nav>
```

Replace with:
```html
        <a href="design-spec.html" class="active">Design Spec</a>
        <a href="architecture.html">Architecture</a>
      </nav>
```

Then find:
```html
      <div class="docs-footer">
        <a href="skills-reference.html" class="docs-prev">Skills Reference</a>
        <span></span>
      </div>
```

Replace with:
```html
      <div class="docs-footer">
        <a href="skills-reference.html" class="docs-prev">Skills Reference</a>
        <a href="architecture.html" class="docs-next">Architecture</a>
      </div>
```

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/hortora.github.io
git add docs/getting-started.html docs/how-it-works.html docs/skills-reference.html docs/design-spec.html
git commit -m "feat(docs): add Architecture to sidebar nav and design-spec footer"
```

---

## Task 2: Create architecture.html

**Files:**
- Create: `hortora.github.io/docs/architecture.html`

- [ ] **Step 1: Create the file**

Create `hortora.github.io/docs/architecture.html` with the following complete content:

```html
---
layout: default
title: "Architecture — Hortora"
permalink: /docs/architecture.html
---

  <div class="docs-layout">

    <aside class="docs-sidebar">
      <div class="docs-sidebar-label">Documentation</div>
      <nav aria-label="Documentation">
        <a href="getting-started.html">Getting Started</a>
        <a href="how-it-works.html">How It Works</a>
        <a href="skills-reference.html">Skills Reference</a>
        <a href="design-spec.html">Design Spec</a>
        <a href="architecture.html" class="active">Architecture</a>
      </nav>
    </aside>

    <main class="docs-content">

      <h1>Architecture</h1>
      <p class="lead">How Hortora's core systems work — for platform architects evaluating enterprise deployment.</p>

      <h2>Knowledge never goes stale silently</h2>
      <p>An entry from two years ago looks identical to one written yesterday — same formatting, same authority, same surfacing. Without enforcement, AI agents confidently advise fixes that were correct for an earlier library version but break on current ones, with no indication anything is amiss.</p>
      <p>Hortora enforces staleness in five layers:</p>
      <ul>
        <li><strong>Always-on annotation:</strong> Every SEARCH result shows the entry's age. Past-threshold entries get a ⚠️ stale warning with verification date and version (if captured). Approaching-threshold entries get an ℹ️ advisory. This fires even if no maintenance has ever been run.</li>
        <li><strong>Version capture at write time:</strong> Library and tool-specific entries prompt for <code>verified_on</code> (e.g. <code>quarkus: 3.34.2</code>) at CAPTURE time. SEARCH uses this to produce a concrete version-gap flag rather than a generic age warning.</li>
        <li><strong>Rationale capture for high-scoring entries:</strong> Entries scoring ≥12/15 prompt for "why this fix over the obvious alternative" — preserving reasoning that would otherwise be lost with the session.</li>
        <li><strong>Domain-filtered session review:</strong> At SWEEP end, forage checks entries in the domains worked in that session against their <code>staleness_threshold</code>. Expired entries surface for Confirm/Revise/Retire while the developer has fresh context.</li>
        <li><strong>Systematic backstop:</strong> <code>validate_garden.py --freshness</code> scans all entries and reports the overdue count. <code>harvest REVIEW</code> processes all overdue entries across all domains — the guarantee that no entry escapes review indefinitely.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees</strong>
        <ul>
          <li>Every search result shows age — staleness is never invisible</li>
          <li>Entries past <code>staleness_threshold</code> are flagged before influencing behaviour</li>
          <li><code>last_reviewed</code> resets the staleness clock when a human explicitly confirms validity</li>
          <li><code>harvest REVIEW</code> provides full-garden coverage across all domains</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>SWEEP domain filtering requires session activity to identify domains — cold-start sessions skip the spot-check, but the always-on annotation still fires</li>
          <li><code>harvest REVIEW</code> requires deliberate scheduling; it is not triggered automatically by CI</li>
        </ul>
      </div>

      <h2>No entry enters the garden without passing validation</h2>
      <p>AI-generated content can be plausible-sounding but wrong, duplicate, or malformed. An unguarded knowledge store accumulates noise that erodes trust faster than it accumulates value — and bad entries are harder to find than good ones.</p>
      <p>Every submission passes a multi-gate validation pipeline (<code>validate_pr.py</code>):</p>
      <ul>
        <li><strong>Format validation:</strong> YAML frontmatter must contain all required fields (<code>id</code>, <code>title</code>, <code>type</code>, <code>domain</code>, <code>stack</code>, <code>tags</code>, <code>score</code>, <code>verified</code>, <code>staleness_threshold</code>, <code>submitted</code>). Malformed entries fail immediately.</li>
        <li><strong>Score threshold:</strong> Entries are rated across five dimensions — non-obviousness, discoverability, breadth, pain/impact, longevity — and must reach ≥8/15. The dimensions are defined in the protocol spec and enforced by CI.</li>
        <li><strong>Injection detection:</strong> Entry content is scanned for prompt-injection patterns before it enters the garden. An entry that tries to modify Claude's behaviour at retrieval time is rejected.</li>
        <li><strong>L1 deduplication:</strong> Jaccard similarity is checked against existing entries in the same domain at submission time. Obvious duplicates are rejected before a human reviewer sees them.</li>
        <li><strong>CI enforcement:</strong> GitHub Actions runs <code>validate_pr.py</code> on every PR against the garden repo. Merge is blocked on any failure. No bypass path exists in the standard workflow.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees</strong>
        <ul>
          <li>No entry merges without passing format validation and the ≥8 score threshold</li>
          <li>CI blocks merge on failure — there is no silent pass</li>
          <li>Every entry carries a permanent, collision-resistant GE-ID (<code>GE-YYYYMMDD-xxxxxx</code>) assigned at write time with no central counter</li>
          <li>All entry history is preserved in git — nothing is silently overwritten</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>L1 Jaccard catches obvious duplicates; close variants can still enter — <code>harvest DEDUPE</code> provides L2/L3 semantic deduplication as a periodic maintenance sweep</li>
          <li>Score is self-reported by the submitter; CI enforces the valid range (1–15) but not accuracy — periodic re-scoring improves signal over time</li>
        </ul>
      </div>

      <h2>Concurrent sessions cannot corrupt garden state</h2>
      <p>Multiple AI sessions writing to a shared knowledge store simultaneously can produce partial writes, conflicting commits, and filesystem races — inconsistencies that are hard to detect and harder to recover from. A knowledge garden used across a team is always under concurrent write pressure.</p>
      <p>Hortora uses git as the consistency layer:</p>
      <ul>
        <li><strong>Read discipline:</strong> All reads use <code>git show HEAD:&lt;path&gt;</code> — never the filesystem directly. A session mid-write to a file does not affect what another session reads; they always see the last committed state.</li>
        <li><strong>Write discipline:</strong> Every file write is followed immediately by <code>git add</code> and <code>git commit</code>. No uncommitted garden state persists between skill operations.</li>
        <li><strong>Conflict recovery:</strong> When two sessions commit simultaneously, the second receives a rejected push. <code>git rebase HEAD</code> resolves non-conflicting concurrent writes automatically, without data loss.</li>
        <li><strong>Human-readable format:</strong> YAML frontmatter + markdown — diffs are readable, history is auditable with <code>git log</code> and <code>git blame</code>, no binary formats.</li>
        <li><strong>Sparse blobless clone:</strong> Large gardens use <code>--filter=blob:none --sparse</code> at clone time. Clone size stays bounded as the garden grows to thousands of entries.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees</strong>
        <ul>
          <li>Concurrent sessions cannot produce partially-visible entries — the commit is the write</li>
          <li>Every read sees a complete, committed state regardless of concurrent write activity</li>
          <li>Full audit trail: git history records who submitted what, when, from which session</li>
          <li>Entries are never deleted — only deprecated or retired, with content preserved</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>Rebase recovery handles non-conflicting concurrent writes automatically; two sessions editing the same entry simultaneously require manual conflict resolution (rare — entries are append-only by convention)</li>
          <li>Sparse blobless clone requires a git remote with partial clone support (GitHub, GitLab); local-only gardens fetch all blobs at clone time</li>
        </ul>
      </div>

      <h2>Relevant entries surface; irrelevant ones don't</h2>
      <p>A flat keyword search across hundreds of entries produces two failure modes: false positives (cross-domain noise that wastes context budget and erodes trust) and false negatives (relevant entries missed because symptom keywords don't align). Both degrade the garden's value faster than new entries can recover it.</p>
      <p>Hortora uses a three-tier retrieval algorithm with bounded context cost:</p>
      <ul>
        <li><strong>Tier 1 — Technology filter:</strong> Every entry is assigned a <code>domain</code> at write time (e.g. <code>quarkus</code>, <code>java</code>, <code>tools</code>). Retrieval scopes to the relevant domain first — cross-domain entries are never surfaced for an unrelated query.</li>
        <li><strong>Tier 2 — Symptom and label match:</strong> Within the domain, the index is searched by symptom type and label across three dimensions — By Technology, By Symptom/Type, By Label.</li>
        <li><strong>Tier 3 — Full domain scan:</strong> If tiers 1 and 2 produce no match, a full scan of the domain is performed. No entry in the relevant domain is unreachable regardless of keyword alignment.</li>
        <li><strong>Index pre-load, bodies on demand:</strong> The <code>GARDEN.md</code> index is loaded at session start. Entry bodies are only fetched when an entry is selected — context budget cost is proportional to entries read, not garden size.</li>
        <li><strong>Git-only reads:</strong> Index and entry bodies are always read from <code>git show HEAD:</code> — no partial-write races, no stale filesystem cache.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees</strong>
        <ul>
          <li>Technology scoping eliminates cross-domain false positives at query time</li>
          <li>Three tiers ensure no entry in the relevant domain is unreachable</li>
          <li>Context budget cost is bounded — the index pre-load is a single file regardless of garden size</li>
          <li>Index is always committed state — no read can observe a partial write</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>Tier 3 (full domain scan) degrades in precision as a domain grows beyond ~200 entries — RAPTOR cluster summaries (Phase 8) will address this at scale</li>
          <li>Keyword effectiveness is not automatically measured — entries with thin keywords may be missed until a maintainer updates them; <code>harvest REVIEW</code> is the natural point to catch this</li>
        </ul>
      </div>

      <h2>The protocol scales beyond a single garden</h2>
      <p>A single canonical garden becomes a bottleneck at enterprise scale. Organisation-specific knowledge cannot sit alongside public community knowledge. Domain gardens need independent curation cadences. Yet independent forks fragment the protocol and eliminate sharing.</p>
      <p>Hortora uses a three-tier federation model:</p>
      <ul>
        <li><strong>Canonical garden:</strong> Curated, CI-enforced, publicly reviewed. Source of truth for cross-domain entries. Maintained under the Hortora GitHub organisation.</li>
        <li><strong>Child garden:</strong> Inherits from a canonical garden and extends it with domain-specific or organisation-specific entries. Validation prevents duplication of canonical entries — a child garden adds without repeating.</li>
        <li><strong>Peer gardens:</strong> Independent gardens sharing the Hortora protocol. No inheritance — entries are entirely distinct — but the entry format, GE-ID scheme, and retrieval algorithm are compatible.</li>
        <li><strong>SchemaVer:</strong> Each entry declares its schema version. Mismatches between gardens are detectable at integration time rather than silent.</li>
        <li><strong>Open protocol:</strong> Entry format, validation pipeline, and retrieval algorithm are not Claude-specific. Any AI assistant implementing the forage/harvest skill convention can participate. The spec is published and versioned independently of any model or vendor.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees (current)</strong>
        <ul>
          <li>Entry format and GE-ID scheme are stable and versioned — entries written today will be valid in future schema versions or carry an explicit migration path</li>
          <li>The protocol is vendor-neutral — no lock-in to a specific AI provider or model family</li>
          <li>Enterprise air-gapped deployment is supported today via local-only mode — no GitHub remote required</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Guarantees <span style="font-family:'JetBrains Mono',monospace;font-size:0.7rem;color:var(--sage);font-style:normal;margin-left:0.5rem">Phase 5+</span></strong>
        <ul>
          <li>Child garden validation prevents duplication of canonical entries</li>
          <li>Schema version mismatches fail loudly at integration time, not silently at retrieval time</li>
          <li>Compliant gardens can participate in federated retrieval — forage SEARCH can query across gardens in a single session</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>Federation is Phase 5+ — current production implementation is single-garden. The spec is published; multi-garden tooling is not yet deployed.</li>
        </ul>
      </div>

      <h2>Duplicate knowledge is eliminated in layers, not all at once</h2>
      <p>A single dedup gate at submission time is either too strict (rejecting valid near-duplicates) or too permissive (letting close variants through). A knowledge garden under active contribution accumulates semantic drift that point-in-time checks cannot catch.</p>
      <p>Hortora uses three-level deduplication with a drift counter:</p>
      <ul>
        <li><strong>L1 — Jaccard at PR time (<code>validate_pr.py</code>):</strong> Runs automatically on every submission. Compares the incoming entry against all existing entries in the same domain. Catches obvious duplicates and rejects them before any human review. Fires in CI — no operator action required.</li>
        <li><strong>L2 — Related entry detection (<code>harvest DEDUPE</code>):</strong> Periodic maintenance operation that compares all within-domain entry pairs not yet checked. Related entries — similar but legitimately distinct — are cross-referenced: <em>See also: GE-XXXX</em> is added to both. This improves retrieval coherence without discarding valid knowledge.</li>
        <li><strong>L3 — Duplicate consolidation (<code>harvest DEDUPE</code>):</strong> Within the same sweep, true duplicates are surfaced to the maintainer for consolidation. The less complete entry is retired; the more complete entry is preserved. Discarded entries are recorded in <code>DISCARDED.md</code> for audit.</li>
        <li><strong>Drift counter:</strong> <code>GARDEN.md</code> tracks <em>Entries merged since last sweep</em>. When it reaches the configured threshold (default: 10), <code>harvest DEDUPE</code> should be run. The counter is incremented by the CI integration step on every merge — dedup debt is always visible.</li>
      </ul>
      <div class="callout">
        <strong>Guarantees</strong>
        <ul>
          <li>L1 catches obvious duplicates before merge — no obviously redundant entry enters through the standard workflow</li>
          <li>Every dedup comparison is logged in <code>CHECKED.md</code> — pairs are never compared twice, and the full comparison history is auditable</li>
          <li>Discarded entries are recorded in <code>DISCARDED.md</code>, not silently deleted — the decision to discard is always traceable</li>
          <li>The drift counter makes dedup debt visible — maintainers always know how many entries have been added since the last sweep</li>
        </ul>
      </div>
      <div class="callout">
        <strong>Graceful degradation</strong>
        <ul>
          <li>L2/L3 (<code>harvest DEDUPE</code>) require a dedicated session with full context budget — they are not run automatically by CI</li>
          <li>The drift counter triggers an obligation, not an enforcement gate — a garden can operate above threshold, but the debt is always visible</li>
          <li>Cross-domain duplicates are not checked — entries in different domains are assumed distinct by definition</li>
        </ul>
      </div>

      <div class="callout">
        <strong>Full specification:</strong> The complete design — nine implementation phases, federation protocol, deduplication algorithm, and governance model — is in <a href="https://github.com/Hortora/spec/blob/main/docs/design/2026-04-07-garden-rag-redesign-design.md" target="_blank" rel="noopener noreferrer">spec/docs/design/2026-04-07-garden-rag-redesign-design.md</a> on GitHub.
      </div>

      <div class="docs-footer">
        <a href="design-spec.html" class="docs-prev">Design Spec</a>
        <span></span>
      </div>

    </main>
  </div>
```

- [ ] **Step 2: Verify the file exists**

```bash
ls -la ~/claude/hortora/hortora.github.io/docs/architecture.html
```

Expected: file listed, non-zero size.

- [ ] **Step 3: Run Jekyll locally and verify the page**

```bash
cd ~/claude/hortora/hortora.github.io
bundle exec jekyll serve --port 4000
```

Open `http://localhost:4000/docs/architecture.html` in a browser. Verify:
- Page title shows "Architecture — Hortora"
- Sidebar nav shows all 5 links with "Architecture" active
- All 6 section headings are present
- Callout boxes render with the sage left-border style
- Inline `<code>` elements render in forest green
- Footer shows ← Design Spec with no next link

Then open `http://localhost:4000/docs/design-spec.html` and verify:
- Sidebar nav shows Architecture as the 5th link
- Footer shows ← Skills Reference and Architecture →

Stop the server with Ctrl+C.

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/hortora.github.io
git add docs/architecture.html
git commit -m "feat(docs): add Architecture page for platform/enterprise architects"
```

---

## Self-Review

**Spec coverage:**

| Requirement | Task |
|---|---|
| New page at `/docs/architecture.html` | Task 2 |
| Architecture link in nav of all 4 existing pages | Task 1 |
| design-spec.html footer → Architecture next link | Task 1 |
| Section 1: Knowledge decay (5-layer staleness) | Task 2 |
| Section 2: Submission integrity (validation pipeline) | Task 2 |
| Section 3: Storage reliability (git consistency) | Task 2 |
| Section 4: Retrieval accuracy (3-tier algorithm) | Task 2 |
| Section 5: Federation & governance | Task 2 |
| Section 6: Deduplication (L1/L2/L3 + drift counter) | Task 2 |
| Guarantees callout per section | Task 2 |
| Graceful degradation callout per section | Task 2 |
| Phase 5+ badge on federation guarantees | Task 2 |
| Full spec link callout at bottom | Task 2 |
| architecture.html sidebar has Architecture active | Task 2 |
| architecture.html footer ← Design Spec | Task 2 |

**No gaps found. No placeholders.**
