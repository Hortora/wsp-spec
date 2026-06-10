# Homepage Quick Start Section Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a Quick Start section to the homepage showing install-from-marketplace, forage, and harvest in three steps with a git graphic — and simplify the Getting Started docs page to match.

**Architecture:** Three targeted file changes: append CSS classes to the shared stylesheet, insert an HTML section into the homepage between the hero and the first divider, and simplify the Getting Started docs page to replace the complex curl/Python install with a single `/install-skills` command.

**Tech Stack:** Plain HTML, CSS. No build tooling. All work in `~/claude/hortora/hortora.github.io/`.

---

## File Map

**Modify:**
- `style.css` — append `.qs-*` CSS classes (Quick Start section styles)
- `index.html` — insert `<section class="quickstart">` between lines 553 and 555
- `docs/getting-started.html` — simplify install section; remove Python prerequisite

---

## Task 1: Add Quick Start CSS to style.css

**Files:**
- Modify: `style.css` (append to end, after line 400)

- [ ] **Step 1: Append Quick Start CSS block to style.css**

Add the following block at the very end of `/Users/mdproctor/claude/hortora/hortora.github.io/style.css`:

```css
/* ── Quick Start section ── */
.quickstart {
  background: var(--parchment-dark);
  border-top: 1px solid var(--sage-light);
  border-bottom: 1px solid var(--sage-light);
  padding: 5rem 2rem;
}

.quickstart-inner {
  max-width: 1100px;
  margin: 0 auto;
}

.qs-steps {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 2.5rem;
  margin-bottom: 3rem;
  align-items: start;
}

.qs-step {
  border-top: 2px solid var(--sage-light);
  padding-top: 1.25rem;
}

.qs-step-num {
  font-family: 'Cormorant Garamond', serif;
  font-size: 2.5rem;
  font-weight: 300;
  color: var(--sage-light);
  line-height: 1;
  margin-bottom: 0.5rem;
}

.qs-step h3 {
  font-family: 'Cormorant Garamond', serif;
  font-size: 1.3rem;
  font-weight: 500;
  color: var(--forest);
  margin-bottom: 0.6rem;
}

.qs-step p {
  font-size: 0.85rem;
  color: var(--ink-light);
  line-height: 1.7;
  margin-bottom: 0.75rem;
}

.qs-code {
  background: var(--ink);
  color: #a8d5a2;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.75rem;
  padding: 0.9rem 1rem;
  border-radius: 4px;
  line-height: 1.8;
}

.qs-code .dim    { color: var(--sage); }
.qs-code .prompt { color: var(--sage-light); user-select: none; }

.qs-tag {
  display: inline-block;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.6rem;
  letter-spacing: 0.08em;
  padding: 0.15rem 0.5rem;
  border-radius: 2px;
  margin-bottom: 0.5rem;
  background: #E8F4E8;
  color: var(--forest);
}

.qs-graphic {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
}

.qs-terminal {
  background: var(--ink);
  border-radius: 6px;
  overflow: hidden;
}

.qs-terminal-bar {
  background: #2d4a2d;
  padding: 0.5rem 0.75rem;
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.qs-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--sage);
  opacity: 0.4;
}

.qs-terminal-label {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.62rem;
  color: var(--sage);
  letter-spacing: 0.08em;
  margin-left: 0.25rem;
}

.qs-terminal-body {
  padding: 1rem 1.25rem;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.72rem;
  color: #a8d5a2;
  line-height: 1.9;
}

.qs-terminal-body .cmd    { color: #f4efe4; }
.qs-terminal-body .dim    { color: var(--sage); }
.qs-terminal-body .file   { color: var(--sage-light); }
.qs-terminal-body .prompt { color: var(--sage); user-select: none; }

.qs-git-tree {
  grid-column: 1 / -1;
  background: var(--ink);
  border-radius: 6px;
  padding: 0.9rem 1.25rem;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.72rem;
  color: #a8d5a2;
  line-height: 1.9;
  display: flex;
  gap: 3rem;
}

.qs-git-tree .dim  { color: var(--sage); }
.qs-git-tree .sha  { color: var(--sage-light); }
.qs-git-tree .label {
  font-size: 0.6rem;
  color: var(--sage);
  letter-spacing: 0.1em;
  text-transform: uppercase;
  margin-bottom: 0.25rem;
}
```

- [ ] **Step 2: Verify no existing styles broken**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/index.html
```

Expected: landing page looks identical (new CSS classes are unused until Task 2).

- [ ] **Step 3: Commit**

```bash
cd ~/claude/hortora/hortora.github.io
git add style.css
git commit -m "feat: add Quick Start section CSS classes"
```

---

## Task 2: Add Quick Start section to index.html

**Files:**
- Modify: `index.html` (insert between lines 553 and 555)

The hero `</section>` is at line 553. The first divider `<div class="divider">` is at line 555. Insert the new section between them.

- [ ] **Step 1: Insert Quick Start section into index.html**

Find this exact text in `index.html` (lines 553–555):

```html
  </section>

  <div class="divider"><span class="divider-mark">⁂</span></div>
```

Replace with:

```html
  </section>

  <!-- ── Quick Start ── -->
  <section class="quickstart">
    <div class="quickstart-inner">

      <p class="section-label">Get started in three steps</p>

      <div class="qs-steps">

        <div class="qs-step">
          <div class="qs-step-num">01</div>
          <h3>Install from the marketplace</h3>
          <p>In any Claude Code session, point it at the Hortora marketplace. Skills install automatically — no scripts, no config.</p>
          <div class="qs-code">
            <span class="dim"># in any Claude Code session</span><br>
            <span class="prompt">&gt; </span>/install-skills https://github.com/Hortora/soredium
          </div>
        </div>

        <div class="qs-step">
          <div class="qs-step-num">02</div>
          <span class="qs-tag">session-time</span>
          <h3>forage — capture</h3>
          <p>During any project session, invoke forage when something non-obvious surfaces. It queues the entry as a git-tracked submission — nothing touches the live garden yet.</p>
          <div class="qs-code">
            <span class="dim"># in any project session</span><br>
            <span class="prompt">&gt; </span>/forage
          </div>
        </div>

        <div class="qs-step">
          <div class="qs-step-num">03</div>
          <span class="qs-tag">maintenance</span>
          <h3>harvest — merge</h3>
          <p>In a dedicated maintenance session, harvest integrates pending submissions — assigning stable GE-IDs, deduplicating, and committing the result to your garden.</p>
          <div class="qs-code">
            <span class="dim"># in a maintenance session</span><br>
            <span class="prompt">&gt; </span>/harvest
          </div>
        </div>

      </div>

      <div class="qs-graphic">

        <div class="qs-terminal">
          <div class="qs-terminal-bar">
            <div class="qs-dot"></div>
            <div class="qs-dot"></div>
            <div class="qs-dot"></div>
            <span class="qs-terminal-label">forage — project session</span>
          </div>
          <div class="qs-terminal-body">
            <span class="prompt">&gt; </span><span class="cmd">/forage</span><br>
            <span class="dim">Captured: Maven skips tests silently</span><br>
            <span class="dim">  type: gotcha · tech: maven</span><br>
            <br>
            <span class="prompt">$ </span><span class="cmd">git status</span><br>
            <span class="file">M  submissions/GE-pending-042.md</span><br>
            <span class="prompt">$ </span><span class="cmd">git log --oneline -1</span><br>
            <span class="dim">9e3d8a1 forage: capture GE-042 pending</span>
          </div>
        </div>

        <div class="qs-terminal">
          <div class="qs-terminal-bar">
            <div class="qs-dot"></div>
            <div class="qs-dot"></div>
            <div class="qs-dot"></div>
            <span class="qs-terminal-label">harvest — maintenance session</span>
          </div>
          <div class="qs-terminal-body">
            <span class="prompt">&gt; </span><span class="cmd">/harvest</span><br>
            <span class="dim">Merged GE-0042 — maven-surefire-skip.md</span><br>
            <span class="dim">  1 submission · 0 duplicates</span><br>
            <br>
            <span class="prompt">$ </span><span class="cmd">git status</span><br>
            <span class="file">A  tools/maven/maven-surefire-skip.md</span><br>
            <span class="prompt">$ </span><span class="cmd">git log --oneline -1</span><br>
            <span class="dim">4a2f1bc harvest: merge GE-0042</span>
          </div>
        </div>

        <div class="qs-git-tree">
          <div>
            <div class="label">git log — knowledge-garden</div>
            <div><span class="sha">4a2f1bc</span> harvest: merge GE-0042 — maven-surefire-skip</div>
            <div><span class="sha">9e3d8a1</span> forage: capture GE-042 pending</div>
            <div><span class="sha">2b7f4e2</span> forage: capture GE-041 pending</div>
            <div><span class="sha">c1d3f0e</span> harvest: merge GE-0041 — git-rebase-conflict</div>
          </div>
          <div>
            <div class="label">garden after harvest</div>
            <div>tools/maven/maven-surefire-skip.md</div>
            <div>tools/git/git-rebase-conflict.md</div>
            <div class="dim">submissions/ — empty</div>
          </div>
        </div>

      </div>
    </div>
  </section>

  <div class="divider"><span class="divider-mark">⁂</span></div>
```

- [ ] **Step 2: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/index.html
```

Expected: new parchment-dark section appears between the hero and "What it is". Three equal columns (01 Install / 02 forage / 03 harvest). Below: two dark terminals side by side with a full-width git log panel beneath them.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Quick Start section to homepage"
```

---

## Task 3: Simplify docs/getting-started.html

**Files:**
- Modify: `docs/getting-started.html`

Two changes: remove Python from prerequisites; replace the two-step install (curl + python3) with a single `/install-skills` command.

- [ ] **Step 1: Remove Python prerequisite**

Find in `docs/getting-started.html`:

```html
      <ul>
        <li>Claude Code installed and authenticated</li>
        <li>Python 3.8 or later</li>
        <li>macOS or Linux</li>
      </ul>
```

Replace with:

```html
      <ul>
        <li>Claude Code installed and authenticated</li>
        <li>macOS or Linux</li>
      </ul>
```

- [ ] **Step 2: Replace the two install sections with one**

Find in `docs/getting-started.html` (lines 51–67):

```html
      <h2>Install the skill manager</h2>
      <p>Download the <code>claude-skill</code> installer from the soredium repository:</p>
      <div class="code-block">
        <span class="comment"># Download the skill manager</span><br>
        <span class="prompt">$ </span>curl -fsSL https://raw.githubusercontent.com/Hortora/soredium/main/scripts/claude-skill -o claude-skill
      </div>
      <p>The downloaded file is a Python script — invoke it with <code>python3 claude-skill</code> (no chmod needed).</p>

      <h2>Install forage and harvest</h2>
      <p><strong>forage</strong> handles session-time operations (capturing and searching). <strong>harvest</strong> handles maintenance (merging submissions and deduplication).</p>
      <div class="code-block">
        <span class="prompt">$ </span>python3 claude-skill install forage<br>
        <span class="prompt">$ </span>python3 claude-skill install harvest
      </div>
      <div class="callout">
        <strong>Tip:</strong> Claude Code auto-discovers skills placed in <code>~/.claude/skills/</code> — no registration step needed after install.
      </div>
```

Replace with:

```html
      <h2>Install from the marketplace</h2>
      <p>In any Claude Code session, give it the Hortora marketplace URL. Claude Code discovers and installs the skills automatically.</p>
      <div class="code-block">
        <span class="comment"># in any Claude Code session</span><br>
        <span class="prompt">&gt; </span>/install-skills https://github.com/Hortora/soredium
      </div>
      <div class="callout">
        <strong>Tip:</strong> Claude Code places skills in <code>~/.claude/skills/</code> and makes them immediately available in any session — no further setup needed.
      </div>
```

- [ ] **Step 3: Verify in browser**

```bash
open /Users/mdproctor/claude/hortora/hortora.github.io/docs/getting-started.html
```

Expected: Prerequisites now has two items (no Python). Install section has one h2 "Install from the marketplace" with a single code block showing `/install-skills https://github.com/Hortora/soredium`. Callout tip below.

- [ ] **Step 4: Commit**

```bash
git add docs/getting-started.html
git commit -m "fix: simplify Getting Started install to single marketplace command"
```

---

## Task 4: Push and verify

- [ ] **Step 1: Push to GitHub Pages**

```bash
cd ~/claude/hortora/hortora.github.io
git push origin main
```

- [ ] **Step 2: Verify live**

GitHub Pages deploys in ~1 minute.

```bash
open https://hortora.github.io
```

Expected: Quick Start section visible on homepage after the hero. Three columns with dark code blocks. Git graphic with two terminals and commit log. Navigate to Docs → Getting Started and confirm simplified install.

---

## Self-Review

- `.qs-terminal-body .cmd` targets `class="cmd"` inside `.qs-terminal-body` — confirmed correct CSS selector pattern
- `.qs-code .prompt` uses `var(--sage-light)` (consistent with the fix applied to `.code-block .prompt` in the docs build)
- `.qs-git-tree .sha` and `.qs-terminal-body .file` both use `var(--sage-light)` — consistent
- All three tasks are independent file changes; no ordering dependency between Task 2 and Task 3
- Spec coverage: ✅ three-step grid, ✅ git graphic, ✅ getting-started simplification, ✅ Python prerequisite removed
