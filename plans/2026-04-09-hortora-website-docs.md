# Hortora Website Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a four-page documentation section to `hortora.github.io` covering Getting Started, How It Works, Skills Reference, and Design Spec, with a left sidebar navigation.

**Architecture:** Static HTML pages in a new `docs/` directory, sharing the existing `style.css` with new docs-specific classes appended. Each docs page is self-contained with an inline sidebar. The fixed nav bar on all pages gets a new "Docs" link.

**Tech Stack:** Plain HTML, CSS (no build tooling). Fonts via Google Fonts (already loaded). All work in `~/claude/hortora/hortora.github.io/`.

---

## File Map

**Modify:**
- `style.css` — append docs layout CSS classes
- `index.html` — add Docs nav link; update soredium status card
- `blog/index.html` — add Docs nav link
- `blog/2026-04-07-the-rootstock.html` — add Docs nav link
- `blog/2026-04-07-the-session.html` — add Docs nav link

**Create:**
- `docs/getting-started.html`
- `docs/how-it-works.html`
- `docs/skills-reference.html`
- `docs/design-spec.html`

---

## Task 1: Add docs CSS to style.css

**Files:**
- Modify: `style.css` (append to end)

- [ ] **Step 1: Append docs layout classes to style.css**

Add the following block at the end of `style.css`:

```css
/* ── Docs layout ── */
.docs-layout {
  display: flex;
  margin-top: 4.5rem;
  min-height: calc(100vh - 4.5rem);
}

.docs-sidebar {
  width: 220px;
  flex-shrink: 0;
  padding: 2.5rem 1.5rem;
  border-right: 1px solid var(--sage-light);
  background: var(--parchment-dark);
  position: sticky;
  top: 4.5rem;
  height: calc(100vh - 4.5rem);
  overflow-y: auto;
}

.docs-sidebar-label {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.65rem;
  font-weight: 500;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: var(--sage);
  margin-bottom: 1rem;
}

.docs-sidebar nav {
  display: flex;
  flex-direction: column;
  gap: 0.1rem;
  position: static;
  background: transparent;
  border: none;
  padding: 0;
}

.docs-sidebar a {
  font-family: 'Lora', Georgia, serif;
  font-size: 0.9rem;
  color: var(--ink-light);
  text-decoration: none;
  padding: 0.4rem 0.75rem;
  border-radius: 3px;
  transition: background 0.15s;
  letter-spacing: 0;
}

.docs-sidebar a:hover {
  background: var(--sage-light);
  color: var(--ink);
}

.docs-sidebar a.active {
  background: var(--forest);
  color: var(--parchment);
}

.docs-content {
  flex: 1;
  padding: 3rem 4rem;
  max-width: 760px;
}

.docs-content h1 {
  font-family: 'Cormorant Garamond', Georgia, serif;
  font-size: 2.4rem;
  font-weight: 400;
  color: var(--ink);
  line-height: 1.2;
  margin-bottom: 0.5rem;
  opacity: 1;
  animation: none;
}

.docs-content .lead {
  font-size: 1rem;
  color: var(--sage);
  font-style: italic;
  margin-bottom: 2.5rem;
  padding-bottom: 2rem;
  border-bottom: 1px solid var(--sage-light);
}

.docs-content h2 {
  font-family: 'Cormorant Garamond', Georgia, serif;
  font-size: 1.5rem;
  font-weight: 500;
  color: var(--forest);
  margin: 2.5rem 0 0.75rem;
}

.docs-content p {
  margin-bottom: 1rem;
  color: var(--ink-light);
  font-size: 0.95rem;
}

.docs-content ul, .docs-content ol {
  margin: 0.5rem 0 1rem 1.25rem;
}

.docs-content li {
  margin-bottom: 0.35rem;
  color: var(--ink-light);
  font-size: 0.95rem;
}

.docs-content a {
  color: var(--forest);
  text-decoration: underline;
  text-underline-offset: 2px;
}

.code-block {
  background: var(--ink);
  color: #a8d5a2;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.8rem;
  padding: 1.25rem 1.5rem;
  border-radius: 4px;
  margin: 1rem 0 1.5rem;
  line-height: 1.7;
  overflow-x: auto;
}

.code-block .comment { color: var(--sage); }
.code-block .prompt   { color: #c8dcca; user-select: none; }

.callout {
  border-left: 3px solid var(--sage);
  background: var(--parchment-dark);
  padding: 1rem 1.25rem;
  border-radius: 0 4px 4px 0;
  margin: 1.5rem 0;
  font-size: 0.9rem;
  color: var(--ink-light);
}

.callout strong { color: var(--forest); }

.docs-content code {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.82em;
  background: var(--parchment-dark);
  padding: 0.1em 0.4em;
  border-radius: 3px;
  color: var(--forest);
}

/* Skills reference table */
.skill-table {
  width: 100%;
  border-collapse: collapse;
  margin: 1rem 0 2rem;
  font-size: 0.88rem;
}

.skill-table th {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.65rem;
  letter-spacing: 0.15em;
  text-transform: uppercase;
  color: var(--sage);
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 2px solid var(--sage-light);
}

.skill-table td {
  padding: 0.65rem 0.75rem;
  border-bottom: 1px solid var(--sage-light);
  color: var(--ink-light);
  vertical-align: top;
}

.skill-table tr:last-child td { border-bottom: none; }

.skill-table td:first-child {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.8rem;
  color: var(--forest);
  white-space: nowrap;
}

/* Architecture diagram */
.arch-diagram {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 0;
  margin: 2rem 0;
  padding: 1.5rem;
  background: var(--parchment-dark);
  border-radius: 4px;
  border: 1px solid var(--sage-light);
  overflow-x: auto;
}

.arch-node {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.35rem;
  min-width: 90px;
}

.arch-node-box {
  background: var(--forest);
  color: var(--parchment);
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.65rem;
  padding: 0.5rem 0.75rem;
  border-radius: 3px;
  text-align: center;
  line-height: 1.4;
  white-space: nowrap;
}

.arch-node-box.arch-light {
  background: var(--parchment);
  color: var(--forest);
  border: 1px solid var(--sage);
}

.arch-node-label {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.55rem;
  color: var(--sage);
  letter-spacing: 0.08em;
  text-transform: uppercase;
  text-align: center;
}

.arch-arrow-group {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.2rem;
  flex-shrink: 0;
  padding: 0 0.25rem;
  margin-bottom: 1.35rem;
}

.arch-arrow {
  font-size: 1rem;
  color: var(--sage);
  line-height: 1;
}

.arch-arrow-label {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.5rem;
  color: var(--sage);
  letter-spacing: 0.06em;
  text-align: center;
  white-space: nowrap;
}

/* Docs footer nav */
.docs-footer {
  margin-top: 3rem;
  padding-top: 2rem;
  border-top: 1px solid var(--sage-light);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.docs-prev, .docs-next {
  font-family: 'Cormorant Garamond', serif;
  font-size: 1rem;
  color: var(--forest);
  text-decoration: none;
  letter-spacing: 0.02em;
}

.docs-prev::before { content: '← '; }
.docs-next::after  { content: ' →'; }
```

- [ ] **Step 2: Open index.html in browser and confirm no visual change**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/index.html
```

Expected: landing page looks identical to before.

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/hortora.github.io
git add style.css
git commit -m "feat: add docs layout CSS classes to style.css"
```

---

## Task 2: Add Docs nav link to index.html and update soredium card

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Add Docs link to nav**

Find this in `index.html`:
```html
      <li><a href="blog/index.html">Diary</a></li>
```

Replace with:
```html
      <li><a href="docs/getting-started.html">Docs</a></li>
      <li><a href="blog/index.html">Diary</a></li>
```

- [ ] **Step 2: Update soredium status card**

Find this in `index.html`:
```html
      <a class="status-card" href="https://github.com/Hortora/soredium" target="_blank">
        <div class="status-card-label">hortora /</div>
        <div class="status-card-name">soredium</div>
        <p class="status-card-desc">The starter kit — validators, CI, MCP server, Obsidian plugin, and Claude skill. Coming soon.</p>
        <span class="badge badge-soon">coming soon</span>
      </a>
```

Replace with:
```html
      <a class="status-card" href="https://github.com/Hortora/soredium" target="_blank">
        <div class="status-card-label">hortora /</div>
        <div class="status-card-name">soredium</div>
        <p class="status-card-desc">The starter kit — Claude skills (forage, harvest), validators, and the claude-skill installer. Growing.</p>
        <span class="badge badge-active">active</span>
      </a>
```

- [ ] **Step 3: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/index.html
```

Expected: "Docs" appears in the nav between "Repos" and "Diary". Soredium card shows "active" badge.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add Docs nav link and update soredium status"
```

---

## Task 3: Add Docs nav link to blog pages

**Files:**
- Modify: `blog/index.html`, `blog/2026-04-07-the-rootstock.html`, `blog/2026-04-07-the-session.html`

The blog pages use `../style.css` and have their own nav HTML. The pattern to find in each file:

- [ ] **Step 1: Add Docs link to blog/index.html**

Find the nav's Diary link. It will look like one of:
```html
<li><a href="index.html">Diary</a></li>
```
or
```html
<li><a href="../blog/index.html">Diary</a></li>
```

Add a Docs link immediately before it:
```html
<li><a href="../docs/getting-started.html">Docs</a></li>
```

To find the exact line: `grep -n "Diary\|diary\|blog" blog/index.html | head -5`

- [ ] **Step 2: Add Docs link to blog/2026-04-07-the-rootstock.html**

Same approach — find the Diary nav link, add Docs before it:
```html
<li><a href="../docs/getting-started.html">Docs</a></li>
```

- [ ] **Step 3: Add Docs link to blog/2026-04-07-the-session.html**

Same as Step 2.

- [ ] **Step 4: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/blog/index.html
```

Expected: "Docs" appears in the nav on the blog index page.

- [ ] **Step 5: Commit**

```bash
git add blog/index.html blog/2026-04-07-the-rootstock.html blog/2026-04-07-the-session.html
git commit -m "feat: add Docs nav link to blog pages"
```

---

## Task 4: Create docs/getting-started.html

**Files:**
- Create: `docs/getting-started.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Getting Started — Hortora</title>
  <meta name="description" content="Install the Hortora garden skills and capture your first entry.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,500;0,600;1,300;1,400&family=Lora:ital,wght@0,400;0,500;1,400&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="../style.css">
</head>
<body>

  <nav>
    <a class="nav-mark" href="../index.html">Hortora</a>
    <ul class="nav-links">
      <li><a href="../index.html#what">About</a></li>
      <li><a href="../index.html#repos">Repos</a></li>
      <li><a href="getting-started.html" class="active">Docs</a></li>
      <li><a href="../blog/index.html">Diary</a></li>
      <li><a href="https://github.com/Hortora" target="_blank">GitHub</a></li>
    </ul>
  </nav>

  <div class="docs-layout">

    <aside class="docs-sidebar">
      <div class="docs-sidebar-label">Documentation</div>
      <nav>
        <a href="getting-started.html" class="active">Getting Started</a>
        <a href="how-it-works.html">How It Works</a>
        <a href="skills-reference.html">Skills Reference</a>
        <a href="design-spec.html">Design Spec</a>
      </nav>
    </aside>

    <main class="docs-content">

      <h1>Getting Started</h1>
      <p class="lead">Install the Hortora garden skills and capture your first entry.</p>

      <h2>Prerequisites</h2>
      <p>Hortora garden skills run inside <a href="https://claude.ai/code" target="_blank">Claude Code</a> — Anthropic's CLI for Claude. You'll need:</p>
      <ul>
        <li>Claude Code installed and authenticated</li>
        <li>Python 3.8 or later</li>
        <li>macOS or Linux</li>
      </ul>

      <h2>Install the skill manager</h2>
      <p>Download the <code>claude-skill</code> installer from the soredium repository:</p>
      <div class="code-block">
        <span class="comment"># Download the skill manager</span><br>
        <span class="prompt">$ </span>curl -fsSL https://raw.githubusercontent.com/Hortora/soredium/main/scripts/claude-skill -o claude-skill
      </div>

      <h2>Install forage and harvest</h2>
      <p><strong>forage</strong> handles session-time operations (capturing and searching). <strong>harvest</strong> handles maintenance (merging submissions and deduplication).</p>
      <div class="code-block">
        <span class="prompt">$ </span>python3 claude-skill install forage<br>
        <span class="prompt">$ </span>python3 claude-skill install harvest
      </div>
      <div class="callout">
        <strong>Tip:</strong> Claude Code auto-discovers skills placed in <code>~/.claude/skills/</code> — no registration step needed after install.
      </div>

      <h2>Capture your first entry</h2>
      <p>During your next Claude Code session, when something non-obvious surfaces — a bug that took an hour to find, a technique worth remembering — Claude will proactively offer to capture it. You can also trigger it directly at any time:</p>
      <div class="code-block">
        <span class="comment"># In any Claude Code session</span><br>
        <span class="prompt">&gt; </span>/forage CAPTURE
      </div>
      <p>Claude will ask for the entry details: what technology, what type (gotcha, technique, or undocumented behaviour), and a description. The entry queues as a submission until you run harvest.</p>

      <div class="docs-footer">
        <span></span>
        <a href="how-it-works.html" class="docs-next">How It Works</a>
      </div>

    </main>
  </div>

  <footer>
    <span class="footer-mark">Hortora — tend your knowledge</span>
    <ul class="footer-links">
      <li><a href="https://github.com/Hortora" target="_blank">github</a></li>
      <li><a href="https://github.com/Hortora/spec" target="_blank">spec</a></li>
      <li><a href="https://github.com/Hortora/soredium" target="_blank">soredium</a></li>
    </ul>
  </footer>

</body>
</html>
```

- [ ] **Step 2: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/docs/getting-started.html
```

Expected: parchment background, forest-green active sidebar link on "Getting Started", code blocks in dark with green text, Docs active in top nav.

- [ ] **Step 3: Commit**

```bash
git add docs/getting-started.html
git commit -m "feat: add Getting Started docs page"
```

---

## Task 5: Create docs/how-it-works.html

**Files:**
- Create: `docs/how-it-works.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>How It Works — Hortora</title>
  <meta name="description" content="The garden model, entry lifecycle, and the local-first design of Hortora.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,500;0,600;1,300;1,400&family=Lora:ital,wght@0,400;0,500;1,400&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="../style.css">
</head>
<body>

  <nav>
    <a class="nav-mark" href="../index.html">Hortora</a>
    <ul class="nav-links">
      <li><a href="../index.html#what">About</a></li>
      <li><a href="../index.html#repos">Repos</a></li>
      <li><a href="getting-started.html" class="active">Docs</a></li>
      <li><a href="../blog/index.html">Diary</a></li>
      <li><a href="https://github.com/Hortora" target="_blank">GitHub</a></li>
    </ul>
  </nav>

  <div class="docs-layout">

    <aside class="docs-sidebar">
      <div class="docs-sidebar-label">Documentation</div>
      <nav>
        <a href="getting-started.html">Getting Started</a>
        <a href="how-it-works.html" class="active">How It Works</a>
        <a href="skills-reference.html">Skills Reference</a>
        <a href="design-spec.html">Design Spec</a>
      </nav>
    </aside>

    <main class="docs-content">

      <h1>How It Works</h1>
      <p class="lead">The garden model, entry lifecycle, and why Hortora is local-first.</p>

      <h2>The garden model</h2>
      <p>A Hortora garden is a machine-wide library of hard-won technical knowledge stored at <code>~/claude/knowledge-garden/</code>. Every Claude Code session on your machine can read from it and contribute to it.</p>
      <p>Entries come in three kinds:</p>
      <ul>
        <li><strong>Gotchas</strong> — bugs that silently fail, behaviours that contradict documentation, workarounds that took hours to find</li>
        <li><strong>Techniques</strong> — non-obvious approaches a skilled developer wouldn't naturally reach for, but would immediately value</li>
        <li><strong>Undocumented</strong> — behaviours, options, or features that exist and work but aren't written down anywhere</li>
      </ul>
      <p>Each entry targets a specific technology and describes exactly one thing. The editorial bar is intentionally high — that's what makes the garden worth consulting.</p>

      <h2>Entry lifecycle</h2>
      <p>Entries move through a governed pipeline from raw capture to live garden:</p>
      <ol>
        <li><strong>Capture</strong> — during a session, forage queues a submission file in <code>~/claude/knowledge-garden/submissions/</code></li>
        <li><strong>Merge</strong> — in a dedicated maintenance session, harvest MERGE assigns a GE-ID and integrates the submission into the live garden</li>
        <li><strong>Deduplicate</strong> — harvest DEDUPE finds overlapping entries and resolves them, keeping the garden clean</li>
      </ol>
      <p>The separation between capture (lightweight, session-time) and merge (thorough, maintenance-time) is intentional. It keeps forage fast and keeps the live garden authoritative.</p>

      <h2>GE-IDs</h2>
      <p>Every merged entry gets a stable identifier — <code>GE-0001</code>, <code>GE-0042</code>, and so on. IDs are assigned sequentially at merge time and never reused. They let Claude reference specific entries across sessions and make deduplication tractable.</p>

      <h2>Local-first</h2>
      <p>The garden is plain files on your filesystem. No server, no account, no network required after installing the skills. Any Claude Code session that can read <code>~/claude/knowledge-garden/</code> can use the garden.</p>
      <p>Federation — sharing gardens across machines and teams via GitHub — is planned for a future phase. The local garden you build now will migrate cleanly.</p>

      <div class="docs-footer">
        <a href="getting-started.html" class="docs-prev">Getting Started</a>
        <a href="skills-reference.html" class="docs-next">Skills Reference</a>
      </div>

    </main>
  </div>

  <footer>
    <span class="footer-mark">Hortora — tend your knowledge</span>
    <ul class="footer-links">
      <li><a href="https://github.com/Hortora" target="_blank">github</a></li>
      <li><a href="https://github.com/Hortora/spec" target="_blank">spec</a></li>
      <li><a href="https://github.com/Hortora/soredium" target="_blank">soredium</a></li>
    </ul>
  </footer>

</body>
</html>
```

- [ ] **Step 2: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/docs/how-it-works.html
```

Expected: "How It Works" is the active sidebar link. Prev/next footer nav shows Getting Started ← and → Skills Reference.

- [ ] **Step 3: Commit**

```bash
git add docs/how-it-works.html
git commit -m "feat: add How It Works docs page"
```

---

## Task 6: Create docs/skills-reference.html

**Files:**
- Create: `docs/skills-reference.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Skills Reference — Hortora</title>
  <meta name="description" content="Complete reference for the forage and harvest Claude Code skills.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,500;0,600;1,300;1,400&family=Lora:ital,wght@0,400;0,500;1,400&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="../style.css">
</head>
<body>

  <nav>
    <a class="nav-mark" href="../index.html">Hortora</a>
    <ul class="nav-links">
      <li><a href="../index.html#what">About</a></li>
      <li><a href="../index.html#repos">Repos</a></li>
      <li><a href="getting-started.html" class="active">Docs</a></li>
      <li><a href="../blog/index.html">Diary</a></li>
      <li><a href="https://github.com/Hortora" target="_blank">GitHub</a></li>
    </ul>
  </nav>

  <div class="docs-layout">

    <aside class="docs-sidebar">
      <div class="docs-sidebar-label">Documentation</div>
      <nav>
        <a href="getting-started.html">Getting Started</a>
        <a href="how-it-works.html">How It Works</a>
        <a href="skills-reference.html" class="active">Skills Reference</a>
        <a href="design-spec.html">Design Spec</a>
      </nav>
    </aside>

    <main class="docs-content">

      <h1>Skills Reference</h1>
      <p class="lead">forage handles session-time operations. harvest handles maintenance. They are installed separately and used in different contexts.</p>

      <h2>forage</h2>
      <p>Use during active Claude Code sessions when non-obvious knowledge surfaces. forage is lightweight by design — it queues submissions without touching the live garden.</p>

      <table class="skill-table">
        <thead>
          <tr>
            <th>Operation</th>
            <th>When to use</th>
            <th>What it does</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>CAPTURE</td>
            <td>A specific gotcha, technique, or undocumented behaviour surfaces</td>
            <td>Records a single entry as a submission file, queued for harvest</td>
          </tr>
          <tr>
            <td>SWEEP</td>
            <td>At the end of a session</td>
            <td>Systematically scans the full session for anything worth capturing, proposes entries for review</td>
          </tr>
          <tr>
            <td>SEARCH</td>
            <td>Looking for prior knowledge before solving a problem</td>
            <td>Retrieves matching entries from the live garden using a three-tier search strategy</td>
          </tr>
          <tr>
            <td>REVISE</td>
            <td>An existing entry needs enrichment or correction</td>
            <td>Updates a previously captured entry with new information</td>
          </tr>
        </tbody>
      </table>

      <div class="code-block">
        <span class="comment"># Trigger forage operations in any Claude Code session</span><br>
        <span class="prompt">&gt; </span>/forage CAPTURE<br>
        <span class="prompt">&gt; </span>/forage SWEEP<br>
        <span class="prompt">&gt; </span>/forage SEARCH maven dependency resolution<br>
        <span class="prompt">&gt; </span>/forage REVISE GE-0042
      </div>

      <h2>harvest</h2>
      <p>Run in a dedicated maintenance session — not during active project work. harvest reads many files and is intentionally thorough.</p>

      <table class="skill-table">
        <thead>
          <tr>
            <th>Operation</th>
            <th>When to use</th>
            <th>What it does</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>MERGE</td>
            <td>Submissions are pending in <code>submissions/</code></td>
            <td>Assigns GE-IDs, validates, and integrates queued submissions into the live garden</td>
          </tr>
          <tr>
            <td>DEDUPE</td>
            <td>After bulk additions or periodically</td>
            <td>Finds duplicate or overlapping entries using three-level Jaccard similarity, resolves with review</td>
          </tr>
        </tbody>
      </table>

      <div class="code-block">
        <span class="comment"># Trigger harvest operations in a dedicated Claude Code session</span><br>
        <span class="prompt">&gt; </span>/harvest MERGE<br>
        <span class="prompt">&gt; </span>/harvest DEDUPE
      </div>

      <div class="callout">
        <strong>Full reference:</strong> The complete skill specifications — trigger conditions, operation details, and submission format — are in the <a href="https://github.com/Hortora/soredium/blob/main/forage/SKILL.md" target="_blank">forage SKILL.md</a> and <a href="https://github.com/Hortora/soredium/blob/main/harvest/SKILL.md" target="_blank">harvest SKILL.md</a> on GitHub.
      </div>

      <div class="docs-footer">
        <a href="how-it-works.html" class="docs-prev">How It Works</a>
        <a href="design-spec.html" class="docs-next">Design Spec</a>
      </div>

    </main>
  </div>

  <footer>
    <span class="footer-mark">Hortora — tend your knowledge</span>
    <ul class="footer-links">
      <li><a href="https://github.com/Hortora" target="_blank">github</a></li>
      <li><a href="https://github.com/Hortora/spec" target="_blank">spec</a></li>
      <li><a href="https://github.com/Hortora/soredium" target="_blank">soredium</a></li>
    </ul>
  </footer>

</body>
</html>
```

- [ ] **Step 2: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/docs/skills-reference.html
```

Expected: "Skills Reference" active in sidebar. Two tables with monospace operation names in forest green. Code block with slash commands. Callout with GitHub links.

- [ ] **Step 3: Commit**

```bash
git add docs/skills-reference.html
git commit -m "feat: add Skills Reference docs page"
```

---

## Task 7: Create docs/design-spec.html

**Files:**
- Create: `docs/design-spec.html`

- [ ] **Step 1: Create the file**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Design Spec — Hortora</title>
  <meta name="description" content="Architecture overview: entry format, retrieval, quality lifecycle, and the federation vision.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,500;0,600;1,300;1,400&family=Lora:ital,wght@0,400;0,500;1,400&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="../style.css">
</head>
<body>

  <nav>
    <a class="nav-mark" href="../index.html">Hortora</a>
    <ul class="nav-links">
      <li><a href="../index.html#what">About</a></li>
      <li><a href="../index.html#repos">Repos</a></li>
      <li><a href="getting-started.html" class="active">Docs</a></li>
      <li><a href="../blog/index.html">Diary</a></li>
      <li><a href="https://github.com/Hortora" target="_blank">GitHub</a></li>
    </ul>
  </nav>

  <div class="docs-layout">

    <aside class="docs-sidebar">
      <div class="docs-sidebar-label">Documentation</div>
      <nav>
        <a href="getting-started.html">Getting Started</a>
        <a href="how-it-works.html">How It Works</a>
        <a href="skills-reference.html">Skills Reference</a>
        <a href="design-spec.html" class="active">Design Spec</a>
      </nav>
    </aside>

    <main class="docs-content">

      <h1>Design Spec</h1>
      <p class="lead">Architecture overview — entry format, retrieval strategy, quality lifecycle, and the federation vision.</p>

      <h2>The loop</h2>
      <p>Hortora is a closed loop between Claude sessions and a governed garden:</p>

      <div class="arch-diagram">
        <div class="arch-node">
          <div class="arch-node-box">Claude<br>session</div>
          <div class="arch-node-label">any project</div>
        </div>
        <div class="arch-arrow-group">
          <div class="arch-arrow">→</div>
          <div class="arch-arrow-label">forage<br>CAPTURE</div>
        </div>
        <div class="arch-node">
          <div class="arch-node-box arch-light">submissions/</div>
          <div class="arch-node-label">queue</div>
        </div>
        <div class="arch-arrow-group">
          <div class="arch-arrow">→</div>
          <div class="arch-arrow-label">harvest<br>MERGE</div>
        </div>
        <div class="arch-node">
          <div class="arch-node-box">knowledge-<br>garden/</div>
          <div class="arch-node-label">live garden</div>
        </div>
        <div class="arch-arrow-group">
          <div class="arch-arrow">→</div>
          <div class="arch-arrow-label">forage<br>SEARCH</div>
        </div>
        <div class="arch-node">
          <div class="arch-node-box">Claude<br>session</div>
          <div class="arch-node-label">any project</div>
        </div>
      </div>

      <h2>Entry format</h2>
      <p>Each entry is a markdown file with YAML frontmatter. The frontmatter carries structured fields for retrieval; the body carries the human-readable content.</p>
      <div class="code-block">
        <span class="comment">---</span><br>
        id: GE-0042<br>
        title: Maven skips tests silently when Surefire version is missing<br>
        type: gotcha<br>
        technology: maven<br>
        tags: [testing, surefire, build]<br>
        status: active<br>
        <span class="comment">---</span><br><br>
        <span class="comment">## Symptom</span><br>
        `mvn test` completes with BUILD SUCCESS but no tests run...<br><br>
        <span class="comment">## Root cause</span><br>
        ...<br><br>
        <span class="comment">## Fix</span><br>
        ...
      </div>

      <h2>Three-tier retrieval</h2>
      <p>forage SEARCH finds entries in three passes, stopping when results are sufficient:</p>
      <ol>
        <li><strong>By technology</strong> — scan entries matching the exact technology tag. Fast, targeted.</li>
        <li><strong>By symptom</strong> — full-text match on symptom and title fields across all entries. Catches cross-technology patterns.</li>
        <li><strong>Full scan</strong> — read all entry bodies. Used when the first two passes return nothing. Budget-limited to keep token cost bounded.</li>
      </ol>

      <h2>Quality lifecycle</h2>
      <p>Every entry has a status field that moves through a defined lifecycle:</p>
      <ul>
        <li><strong>Active</strong> — current, verified, should be consulted</li>
        <li><strong>Suspected</strong> — may be outdated or partially incorrect; flagged for review</li>
        <li><strong>Superseded</strong> — replaced by a newer entry; kept for historical reference</li>
        <li><strong>Retired</strong> — no longer applicable (e.g. for a deprecated library)</li>
      </ul>

      <h2>GitHub backend <span style="font-family:'JetBrains Mono',monospace;font-size:0.7rem;color:var(--sage);font-style:normal;margin-left:0.5rem">planned</span></h2>
      <p>The next phase replaces the local submissions queue with a GitHub PR workflow. Each submission becomes a pull request; CI runs validation and deduplication checks; merging the PR integrates the entry. GitHub Issues serve as stable entry IDs.</p>

      <h2>Federation <span style="font-family:'JetBrains Mono',monospace;font-size:0.7rem;color:var(--sage);font-style:normal;margin-left:0.5rem">planned</span></h2>
      <p>Gardens federate without centralisation. A <em>canonical garden</em> sets the standard for a domain. <em>Child gardens</em> enrich it with local context. <em>Peer gardens</em> share across domains. Each garden remains sovereign — federation is opt-in and additive.</p>

      <div class="callout">
        <strong>Full specification:</strong> The complete design — nine implementation phases, federation protocol, deduplication algorithm, and governance model — is in <a href="https://github.com/Hortora/spec/blob/main/docs/design/2026-04-07-garden-rag-redesign-design.md" target="_blank">spec/docs/design/2026-04-07-garden-rag-redesign-design.md</a> on GitHub.
      </div>

      <div class="docs-footer">
        <a href="skills-reference.html" class="docs-prev">Skills Reference</a>
        <span></span>
      </div>

    </main>
  </div>

  <footer>
    <span class="footer-mark">Hortora — tend your knowledge</span>
    <ul class="footer-links">
      <li><a href="https://github.com/Hortora" target="_blank">github</a></li>
      <li><a href="https://github.com/Hortora/spec" target="_blank">spec</a></li>
      <li><a href="https://github.com/Hortora/soredium" target="_blank">soredium</a></li>
    </ul>
  </footer>

</body>
</html>
```

- [ ] **Step 2: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/docs/design-spec.html
```

Expected: "Design Spec" active in sidebar. Architecture diagram shows 5 boxes with arrows and labels. Code block shows YAML entry format. "planned" labels appear in muted mono next to GitHub backend and Federation headings.

- [ ] **Step 3: Commit**

```bash
git add docs/design-spec.html
git commit -m "feat: add Design Spec docs page with architecture diagram"
```

---

## Task 8: Push and verify on GitHub Pages

**Files:** none (git operation only)

- [ ] **Step 1: Push to origin**

```bash
cd ~/claude/hortora/hortora.github.io
git push origin main
```

- [ ] **Step 2: Verify live site**

GitHub Pages typically deploys within 1–2 minutes.

```bash
open https://hortora.github.io/docs/getting-started.html
```

Expected: live site shows the Getting Started page with sidebar. Navigate through all four docs pages and confirm prev/next links work.

- [ ] **Step 3: Check nav works from main landing page**

```bash
open https://hortora.github.io
```

Expected: "Docs" appears in the nav and links to Getting Started.

---

## Self-Review Notes

- All four docs pages share identical sidebar HTML with only the `.active` class differing — no abstraction available in plain HTML, this is intentional.
- The `docs-sidebar nav` overrides the global `nav` CSS (position: fixed, etc.) — this is handled by the more-specific `.docs-sidebar nav` selector.
- The architecture diagram uses `flex-wrap: wrap` so it doesn't break on narrow viewports, though full mobile responsiveness is out of scope.
- Blog pages need a Docs nav link pointing to `../docs/getting-started.html` (one directory up from blog/); index.html uses `docs/getting-started.html` (relative from root).
