---
date: 2026-04-09
type: phase-update
---

# Hortora — Designing the Engine

## What I was trying to design: the GitHub backend

Phase 2 in the roadmap was clear enough: GitHub Issues as GE-IDs, CI validation on every PR, index maintenance on merge, sparse blobless clone for retrieval. What the spec doesn't settle is where the tooling lives and who calls it.

I knew the goal. The interesting part was working out the architecture.

## One engine, two callers

The central question was where `validate_pr.py` and `integrate_entry.py` live — and therefore how CI gets them.

Three options. Claude and I worked through each one: scripts in the garden repo, scripts in soredium referenced by CI, soredium published to PyPI. The garden repo option looked simple until I looked at it squarely: soredium already owns `validate_garden.py` and its test suite. Putting new scripts in the garden repo would mean the tests live in one place and the scripts they test in another. That's the kind of thing that drifts.

Then Claude made the case for soredium — and in the next message made the case for PyPI. I called it out: "one minute you say C is the right call, now you say B?" The actual position was B now, C later — but the framing had been sloppy.

Once that was straight: soredium owns all scripts. CI checks out soredium alongside the garden. `GITHUB_TOKEN` covers it because soredium is public. Two checkouts, zero secrets complexity. PyPI when soredium stabilises.

The insight that made everything click: the same scripts run in CI and locally. In CI, GitHub Actions calls them on PR events. Locally, the submitter Claude calls them directly before pushing — or before committing, in local-only mode. Three operation modes, one set of scripts:

| Mode | Who calls the scripts | ID source |
|------|----------------------|-----------|
| CI | GitHub Actions | GitHub issue number |
| GitHub-local | Submitter Claude, before PR | GitHub issue number |
| Local-only | Submitter Claude, directly | Sequential counter in GARDEN.md |

The harvester stays local and maintainer-triggered for Phase 2. Claude-in-CI for borderline PRs is in the roadmap.

## Four phases to get there

We planned the delivery as four sub-phases, each leaving the system working before the next starts.

2.1 is the thin slice: minimal `validate_pr.py` with CRITICAL checks only, GitHub Actions `validate-on-pr`, forage CAPTURE in GitHub mode. A real submission PR gets validated by CI — end-to-end, shallow.

2.2 deepens: full Jaccard duplicate scanning, injection patterns, `integrate_entry.py` wiring, automatic index maintenance on merge.

2.3 is local mode parity. Same scripts, no GitHub, same validation. Mode auto-detected from the git remote.

2.4 is the retrieval upgrade: `git cat-file --batch` for on-demand entry reads, sparse blobless clone, session-start pull that fetches only index changes.

The spec is written — 347 lines, GitHub Actions YAML sketched out, script contracts defined, Jaccard computed over tokenised title + tags + summary. The scripts don't exist yet. That's next session.
