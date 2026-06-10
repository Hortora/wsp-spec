# Hortora — Project Handoff

*Last updated: 2026-06-10 (session 20 — phase 2 unblocked, context sync)*

---

## What Hortora Is / Local Folder Structure

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## The Repos — delta only

### CaseHub peer repos

*Unchanged — `git show HEAD~2:HANDOFF.md`*

### `casehubio/neural-text` — C1–C7 complete (confirmed)

*Unchanged — `git show HEAD~1:HANDOFF.md`*

### `casehubio/garden` + `casehubio/parent` — Mutiny Tier 1 protocol update

`casehubio/garden#3` closed: `module-tier-structure.md` updated — Mutiny `provided` is
acceptable in Tier 1 api modules (same reasoning as CDI annotations; inert without a
runtime). `casehubio/parent` PLATFORM.md updated (commit `a9bd5af`) to reflect this
inline. Companion issue `parent#213` was already closed.

### Other repos

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## Open Design Questions

*Unchanged — `git show HEAD~2:HANDOFF.md`*

---

## What To Do Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

**Immediate:** garden-engine phase 2 — SPLADE hybrid search + cross-encoder reranker.
Gate passed. `SparseEmbedder` (`casehub-inference-splade`) and `CrossEncoderReranker`
(`casehub-inference-tasks`) are published.

---

## Key References

- ARC42STORIES.MD: `git show HEAD~1:HANDOFF.md`
- Blog: `../hortora.github.io/_posts/2026-06-10-mdp01-phase-2-unblocked.md`
- Garden: GE-20260610-eb673a (Claude Code Agent tool sub-agents don't inherit MCP connections)
