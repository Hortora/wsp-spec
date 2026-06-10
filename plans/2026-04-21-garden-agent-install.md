# Garden Agent Install Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `garden-agent-install.sh` — an idempotent bash script that installs the autonomous garden deduplication agent into any Hortora garden directory.

**Architecture:** A single self-contained bash installer in `soredium/scripts/` that embeds all agent content as heredocs (no template files). It checks each component before writing, appends rather than overwrites existing files where safe, and prints a status line per component. Tested via Python/subprocess against a temp git repo, following the existing soredium test pattern.

**Tech Stack:** Bash, Python 3 / unittest / subprocess, pytest, TemporaryDirectory

**Spec:** `spec/docs/superpowers/specs/2026-04-21-garden-agent-design.md`

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `soredium/scripts/garden-agent-install.sh` | Create | Idempotent installer — embeds and installs all agent components |
| `soredium/tests/test_garden_agent_install.py` | Create | Full test suite for installer behaviour |

---

### Task 1: Test fixture and skeleton test file

**Files:**
- Create: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the fixture helper and skeleton**

```python
#!/usr/bin/env python3
"""Tests for garden-agent-install.sh."""

import subprocess
import sys
import unittest
from pathlib import Path
from tempfile import TemporaryDirectory

INSTALLER = Path(__file__).parent.parent / 'scripts' / 'garden-agent-install.sh'


def make_garden() -> TemporaryDirectory:
    """Create a temp directory with a bare git repo (simulates a garden)."""
    tmp = TemporaryDirectory()
    subprocess.run(['git', 'init', tmp.name], check=True, capture_output=True)
    # Create an initial commit so HEAD~1 diffs work
    (Path(tmp.name) / 'GARDEN.md').write_text('# Garden\n')
    subprocess.run(['git', '-C', tmp.name, 'add', '.'], check=True, capture_output=True)
    subprocess.run(
        ['git', '-C', tmp.name, 'commit', '-m', 'init'],
        check=True, capture_output=True,
        env={**__import__('os').environ, 'GIT_AUTHOR_NAME': 'Test', 'GIT_AUTHOR_EMAIL': 'test@test.com',
             'GIT_COMMITTER_NAME': 'Test', 'GIT_COMMITTER_EMAIL': 'test@test.com'}
    )
    return tmp


def run_installer(garden_path: str) -> subprocess.CompletedProcess:
    return subprocess.run(
        ['bash', str(INSTALLER)],
        capture_output=True, text=True,
        cwd=garden_path
    )


class TestGardenAgentInstall(unittest.TestCase):
    pass


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 2: Verify the test file is valid Python**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_agent_install.py -v
```

Expected: `0 passed` (no tests yet), no import errors.

- [ ] **Step 3: Create empty installer so import path resolves**

```bash
touch ~/claude/hortora/soredium/scripts/garden-agent-install.sh
chmod +x ~/claude/hortora/soredium/scripts/garden-agent-install.sh
```

- [ ] **Step 4: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: scaffold garden-agent-install.sh and test file"
```

---

### Task 2: Installer installs `garden-agent.sh`

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the failing test**

Add to `TestGardenAgentInstall`:

```python
def test_installs_garden_agent_sh(self):
    with make_garden() as garden:
        result = run_installer(garden)
        agent_sh = Path(garden) / 'garden-agent.sh'
        self.assertTrue(agent_sh.exists(), f"garden-agent.sh not created. stderr: {result.stderr}")
        self.assertTrue(agent_sh.stat().st_mode & 0o111, "garden-agent.sh not executable")
        content = agent_sh.read_text()
        self.assertIn('claude --print', content)
        self.assertIn('HORTORA_GARDEN', content)
        self.assertIn('garden-agent.log', content)
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd ~/claude/hortora/soredium
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_garden_agent_sh -v
```

Expected: FAIL — `garden-agent.sh not created`

- [ ] **Step 3: Write installer with garden-agent.sh heredoc**

Replace `soredium/scripts/garden-agent-install.sh` with:

```bash
#!/usr/bin/env bash
# garden-agent-install.sh — idempotent installer for the garden dedup agent.
# Run from inside a garden directory (or set HORTORA_GARDEN).
# No tokens consumed. Safe to run multiple times.

set -euo pipefail

GARDEN="${HORTORA_GARDEN:-$(pwd)}"
PASS="✓"
SKIP="–"

echo "Garden Agent Installer"
echo "Garden: $GARDEN"
echo ""

# ── garden-agent.sh ──────────────────────────────────────────────────────────
AGENT_SH="$GARDEN/garden-agent.sh"
if [[ -f "$AGENT_SH" ]]; then
    echo "$SKIP  garden-agent.sh            already present"
else
    cat > "$AGENT_SH" << 'AGENT_EOF'
#!/usr/bin/env bash
# garden-agent.sh — invoke Claude dedup agent (hook or manual mode).
GARDEN_ROOT="${HORTORA_GARDEN:-$(pwd)}"
LOG="$GARDEN_ROOT/garden-agent.log"
TASK="You are the Hortora garden deduplication agent. Run the dedup sweep as described in CLAUDE.md."

if [[ "$1" == "--hook" ]] || [[ ! -t 0 ]]; then
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] garden-agent starting" >> "$LOG"
    claude --print "$TASK" >> "$LOG" 2>&1
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] garden-agent done" >> "$LOG"
else
    claude "$TASK"
fi
AGENT_EOF
    chmod +x "$AGENT_SH"
    echo "$PASS  garden-agent.sh            installed"
fi
```

- [ ] **Step 4: Run test to verify it passes**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_garden_agent_sh -v
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer creates garden-agent.sh"
```

---

### Task 3: Installer installs `.claude/settings.json`

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the failing test**

Add to `TestGardenAgentInstall`:

```python
def test_installs_settings_json(self):
    import json
    with make_garden() as garden:
        result = run_installer(garden)
        settings = Path(garden) / '.claude' / 'settings.json'
        self.assertTrue(settings.exists(), f"settings.json not created. stderr: {result.stderr}")
        data = json.loads(settings.read_text())
        self.assertEqual(data.get('defaultMode'), 'acceptEdits')
        allowed = data['permissions']['allow']
        self.assertTrue(any('dedupe_scanner.py' in r for r in allowed))
        self.assertTrue(any('git commit' in r for r in allowed))
```

- [ ] **Step 2: Run test to verify it fails**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_settings_json -v
```

Expected: FAIL — `settings.json not created`

- [ ] **Step 3: Add settings.json block to installer**

Append to `garden-agent-install.sh` (after the garden-agent.sh block):

```bash
# ── .claude/settings.json ────────────────────────────────────────────────────
SETTINGS_DIR="$GARDEN/.claude"
SETTINGS="$SETTINGS_DIR/settings.json"
if [[ -f "$SETTINGS" ]]; then
    echo "$SKIP  .claude/settings.json      already present"
else
    mkdir -p "$SETTINGS_DIR"
    cat > "$SETTINGS" << 'SETTINGS_EOF'
{
  "defaultMode": "acceptEdits",
  "permissions": {
    "allow": [
      "Bash(git show *)",
      "Bash(git log *)",
      "Bash(git status *)",
      "Bash(git add *)",
      "Bash(git commit *)",
      "Bash(git diff *)",
      "Bash(python3 */dedupe_scanner.py *)",
      "Bash(python3 */validate_garden.py *)"
    ],
    "deny": []
  }
}
SETTINGS_EOF
    echo "$PASS  .claude/settings.json      installed"
fi
```

- [ ] **Step 4: Run test to verify it passes**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_settings_json -v
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer creates .claude/settings.json"
```

---

### Task 4: Installer installs `CLAUDE.md`

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the failing test**

Add to `TestGardenAgentInstall`:

```python
def test_installs_claude_md(self):
    with make_garden() as garden:
        result = run_installer(garden)
        claude_md = Path(garden) / 'CLAUDE.md'
        self.assertTrue(claude_md.exists(), f"CLAUDE.md not created. stderr: {result.stderr}")
        content = claude_md.read_text()
        self.assertIn('garden deduplication agent', content)
        self.assertIn('dedupe_scanner.py', content)
        self.assertIn('duplicate-discarded', content)
        self.assertIn('git show HEAD:', content)
```

- [ ] **Step 2: Run test to verify it fails**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_claude_md -v
```

Expected: FAIL — `CLAUDE.md not created`

- [ ] **Step 3: Add CLAUDE.md block to installer**

Append to `garden-agent-install.sh`:

```bash
# ── CLAUDE.md ─────────────────────────────────────────────────────────────────
CLAUDE_MD="$GARDEN/CLAUDE.md"
if [[ -f "$CLAUDE_MD" ]]; then
    echo "$SKIP  CLAUDE.md                  already present"
else
    cat > "$CLAUDE_MD" << 'CLAUDE_EOF'
# Garden Deduplication Agent

You are the Hortora garden deduplication agent. When invoked, run a full
dedup sweep and commit the results without asking for confirmation.

## Environment

- Garden root: current working directory
- Scanner: `python3 ${SOREDIUM_PATH:-~/claude/hortora/soredium}/scripts/dedupe_scanner.py .`
- All reads via `git show HEAD:<path>` — never read files directly

## Workflow

1. Run `dedupe_scanner.py . --top 50` to get unchecked pairs, highest score first
2. For each pair, read both entries: `git show HEAD:<domain>/<id>.md | head -35`
3. Classify and act:

| Classification | Action |
|---|---|
| **Distinct** | `--record` as `distinct` |
| **Related** | Append `**See also:**` line to both files, `--record` as `related` |
| **Duplicate** | Apply duplicate rules below |

4. Commit: `git add -A && git commit -m "dedupe: sweep N pairs — M related, K duplicates resolved"`

## Duplicate Rules

Keep the entry with the higher `score:` in frontmatter. If tied, keep the
newer `submitted:` date. If still tied, keep the longer entry (line count).

Delete the discarded file. Append to `DISCARDED.md`. Remove from `GARDEN.md`
index if present. Record as `duplicate-discarded`.

## Tiebreaker order

Score → submitted date → line count (keep longer).
CLAUDE_EOF
    echo "$PASS  CLAUDE.md                  installed"
fi
```

- [ ] **Step 4: Run test to verify it passes**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_claude_md -v
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer creates CLAUDE.md"
```

---

### Task 5: Installer installs `post-commit` hook

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

The hook may already exist (the garden already has pre-commit logic). The installer
appends only if the agent block is not already present, guarded by a sentinel comment.

- [ ] **Step 1: Write the failing tests**

Add to `TestGardenAgentInstall`:

```python
def test_installs_post_commit_hook(self):
    with make_garden() as garden:
        result = run_installer(garden)
        hook = Path(garden) / '.git' / 'hooks' / 'post-commit'
        self.assertTrue(hook.exists(), f"post-commit hook not created. stderr: {result.stderr}")
        self.assertTrue(hook.stat().st_mode & 0o111, "post-commit hook not executable")
        content = hook.read_text()
        self.assertIn('garden-agent.sh', content)
        self.assertIn('GE-', content)  # grep pattern for new entries

def test_post_commit_hook_idempotent(self):
    with make_garden() as garden:
        run_installer(garden)
        run_installer(garden)  # second run
        hook = Path(garden) / '.git' / 'hooks' / 'post-commit'
        content = hook.read_text()
        # Agent block should appear exactly once
        self.assertEqual(content.count('garden-agent.sh'), 1)

def test_post_commit_hook_appends_to_existing(self):
    with make_garden() as garden:
        # Pre-existing hook content
        hook = Path(garden) / '.git' / 'hooks' / 'post-commit'
        hook.write_text('#!/bin/bash\n# existing hook\necho "existing"\n')
        hook.chmod(0o755)
        run_installer(garden)
        content = hook.read_text()
        self.assertIn('existing', content)
        self.assertIn('garden-agent.sh', content)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_post_commit_hook tests/test_garden_agent_install.py::TestGardenAgentInstall::test_post_commit_hook_idempotent tests/test_garden_agent_install.py::TestGardenAgentInstall::test_post_commit_hook_appends_to_existing -v
```

Expected: all FAIL

- [ ] **Step 3: Add post-commit hook block to installer**

Append to `garden-agent-install.sh`:

```bash
# ── .git/hooks/post-commit ───────────────────────────────────────────────────
HOOK="$GARDEN/.git/hooks/post-commit"
SENTINEL="# garden-agent: auto-installed"
if [[ -f "$HOOK" ]] && grep -q "$SENTINEL" "$HOOK"; then
    echo "$SKIP  .git/hooks/post-commit     already present"
else
    cat >> "$HOOK" << 'HOOK_EOF'
# garden-agent: auto-installed
# Fire garden dedup agent when a commit adds new GE-*.md entries.
_GARDEN_ROOT="$(git rev-parse --show-toplevel)"
_LOG="$_GARDEN_ROOT/garden-agent.log"
_new_entries=$(git diff --name-only HEAD~1 HEAD 2>/dev/null \
  | grep -E "^[^/]+/GE-[0-9]{8}-[0-9a-f]{6}\.md$")
if [[ -n "$_new_entries" ]]; then
    echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] new entries detected, starting agent:" >> "$_LOG"
    echo "$_new_entries" | sed 's/^/  /' >> "$_LOG"
    nohup "$_GARDEN_ROOT/garden-agent.sh" --hook >> "$_LOG" 2>&1 &
fi
HOOK_EOF
    chmod +x "$HOOK"
    echo "$PASS  .git/hooks/post-commit     installed"
fi
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_post_commit_hook tests/test_garden_agent_install.py::TestGardenAgentInstall::test_post_commit_hook_idempotent tests/test_garden_agent_install.py::TestGardenAgentInstall::test_post_commit_hook_appends_to_existing -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer appends post-commit hook block"
```

---

### Task 6: Installer adds `.gitignore` entry

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the failing tests**

Add to `TestGardenAgentInstall`:

```python
def test_installs_gitignore_entry(self):
    with make_garden() as garden:
        result = run_installer(garden)
        gitignore = Path(garden) / '.gitignore'
        self.assertTrue(gitignore.exists(), f".gitignore not created. stderr: {result.stderr}")
        self.assertIn('garden-agent.log', gitignore.read_text())

def test_gitignore_idempotent(self):
    with make_garden() as garden:
        run_installer(garden)
        run_installer(garden)
        gitignore = Path(garden) / '.gitignore'
        content = gitignore.read_text()
        self.assertEqual(content.count('garden-agent.log'), 1)

def test_gitignore_appends_to_existing(self):
    with make_garden() as garden:
        gitignore = Path(garden) / '.gitignore'
        gitignore.write_text('*.pyc\n__pycache__/\n')
        run_installer(garden)
        content = gitignore.read_text()
        self.assertIn('*.pyc', content)
        self.assertIn('garden-agent.log', content)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_gitignore_entry tests/test_garden_agent_install.py::TestGardenAgentInstall::test_gitignore_idempotent tests/test_garden_agent_install.py::TestGardenAgentInstall::test_gitignore_appends_to_existing -v
```

Expected: all FAIL

- [ ] **Step 3: Add .gitignore block to installer**

Append to `garden-agent-install.sh`:

```bash
# ── .gitignore ────────────────────────────────────────────────────────────────
GITIGNORE="$GARDEN/.gitignore"
if [[ -f "$GITIGNORE" ]] && grep -q "garden-agent.log" "$GITIGNORE"; then
    echo "$SKIP  .gitignore                 already present"
else
    echo "garden-agent.log" >> "$GITIGNORE"
    echo "$PASS  .gitignore                 updated"
fi
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_installs_gitignore_entry tests/test_garden_agent_install.py::TestGardenAgentInstall::test_gitignore_idempotent tests/test_garden_agent_install.py::TestGardenAgentInstall::test_gitignore_appends_to_existing -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer adds garden-agent.log to .gitignore"
```

---

### Task 7: Status report output and full idempotency test

**Files:**
- Modify: `soredium/scripts/garden-agent-install.sh`
- Modify: `soredium/tests/test_garden_agent_install.py`

- [ ] **Step 1: Write the failing tests**

Add to `TestGardenAgentInstall`:

```python
def test_status_output_on_fresh_install(self):
    with make_garden() as garden:
        result = run_installer(garden)
        self.assertEqual(result.returncode, 0)
        output = result.stdout
        self.assertIn('garden-agent.sh', output)
        self.assertIn('settings.json', output)
        self.assertIn('CLAUDE.md', output)
        self.assertIn('post-commit', output)
        self.assertIn('.gitignore', output)

def test_full_idempotency(self):
    with make_garden() as garden:
        first = run_installer(garden)
        second = run_installer(garden)
        self.assertEqual(second.returncode, 0)
        # Second run: all lines should show skip marker, none should show install marker
        for line in second.stdout.splitlines():
            if any(name in line for name in ['garden-agent.sh', 'settings.json', 'CLAUDE.md', 'post-commit', '.gitignore']):
                self.assertIn('already present', line, f"Expected 'already present' in: {line}")
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m pytest tests/test_garden_agent_install.py::TestGardenAgentInstall::test_status_output_on_fresh_install tests/test_garden_agent_install.py::TestGardenAgentInstall::test_full_idempotency -v
```

Expected: `test_status_output_on_fresh_install` may pass (output already exists), `test_full_idempotency` FAIL — `.gitignore` line says "updated" not "already present"

- [ ] **Step 3: Fix .gitignore status message to match**

The `.gitignore` block currently prints "updated" on first install but the test expects "already present" on second run. Verify the grep guard works:

```bash
# In garden-agent-install.sh — .gitignore block should read:
if [[ -f "$GITIGNORE" ]] && grep -q "garden-agent.log" "$GITIGNORE"; then
    echo "$SKIP  .gitignore                 already present"
else
    echo "garden-agent.log" >> "$GITIGNORE"
    echo "$PASS  .gitignore                 updated"
fi
```

This is already correct from Task 6 — run the tests to confirm.

- [ ] **Step 4: Run all tests to verify they pass**

```bash
python3 -m pytest tests/test_garden_agent_install.py -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: installer status output and full idempotency verified"
```

---

### Task 8: Run installer against the real garden

This is a manual verification step — no new code, no new tests. Confirms the installer works end-to-end against `~/.hortora/garden`.

- [ ] **Step 1: Run the installer against the live garden**

```bash
cd ~/.hortora/garden
~/claude/hortora/soredium/scripts/garden-agent-install.sh
```

Expected output (all five components reported as installed):
```
Garden Agent Installer
Garden: /Users/mdproctor/.hortora/garden

✓  garden-agent.sh            installed
✓  .claude/settings.json      installed
✓  CLAUDE.md                  installed
✓  .git/hooks/post-commit     installed
✓  .gitignore                 updated
```

- [ ] **Step 2: Run it again — verify all show as already present**

```bash
~/claude/hortora/soredium/scripts/garden-agent-install.sh
```

Expected output (all skip):
```
Garden Agent Installer
Garden: /Users/mdproctor/.hortora/garden

–  garden-agent.sh            already present
–  .claude/settings.json      already present
–  CLAUDE.md                  already present
–  .git/hooks/post-commit     already present
–  .gitignore                 already present
```

- [ ] **Step 3: Commit the installed files to the garden repo**

```bash
cd ~/.hortora/garden
git add garden-agent.sh .claude/settings.json CLAUDE.md .gitignore
git commit -m "feat: install garden dedup agent"
```

Note: `.git/hooks/post-commit` is not committed — git hooks are machine-local.

- [ ] **Step 4: Commit installer to soredium**

```bash
cd ~/claude/hortora/soredium
git add scripts/garden-agent-install.sh tests/test_garden_agent_install.py
git commit -m "feat: add garden-agent-install.sh to soredium scripts"
```
