# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-04-10 — Garden staleness: entries decay without context or review mechanism

**Priority:** high  
**Status:** active

Garden entries capture WHAT (symptom, root cause, fix) but not WHY this fix over alternatives, or WHEN the knowledge expires. The `staleness_threshold` field exists but is never enforced — entries past threshold look identical to current ones. The risk: AI agents act on stale, decontextualized knowledge with full confidence, amplifying bad practices across sessions silently. "Red hat bureaucracy with agentic powers."

**Context:** Feedback from external reviewer. Discussed 2026-04-10 in hortora session. Core tension: a knowledge garden is only as good as its decay management.

**Proposed solutions:**

---

### Solution 1 — Active staleness enforcement in forage SWEEP

When sweeping a session, flag entries past their `staleness_threshold` and prompt the user to confirm, revise, or retire them. Implementation: ~1 day change to forage SWEEP mode.

**Argument for:** Triggers human review at the right moment — end of a session, when the user has just been working in the relevant domain and context is freshest. Closes the loop naturally without a separate maintenance obligation.

**Argument against:** SWEEP happens infrequently and not always in the domain of the expiring entry. A user finishing a Java session won't have useful context to review a stale tmux entry surfaced at sweep time. Review quality may be poor.

**Success rating: 7/10**

*Why it might not succeed:* Users under time pressure will confirm without verifying ("still valid"). SWEEP is optional — users who skip it never see the flags. Doesn't help with entries that were stale from inception because the original submitter misjudged the staleness_threshold.

---

### Solution 2 — Skepticism annotation in forage SEARCH results

When surfacing a garden entry in SEARCH, append a visible age marker: "⚠️ entry dated YYYY-MM-DD ({N} months old) — verify still applies to your stack version." Entries past threshold get a stronger flag. Implementation: hours.

**Argument for:** Zero friction on submission — no schema change, no new fields. Acts at the exact moment Claude would use the knowledge, inserting epistemic humility before the entry influences behaviour. Cheapest possible intervention with immediate effect.

**Argument against:** Warning fatigue is real. Claude displays the annotation but continues acting on the entry anyway because the annotation doesn't change the underlying recommendation. "Verify still applies" is easy to dismiss under task pressure.

**Success rating: 6/10**

*Why it might not succeed:* Annotations without teeth become wallpaper. If Claude always shows a warning but acts on the entry regardless, users learn to ignore the warning. Doesn't address the "why this fix" gap at all — just flags age.

---

### Solution 3 — Require "rationale" field on high-scoring entries

Entries scoring 12+ require a brief `rationale` field: why this fix over the obvious alternative. Enforced by the validator; forage prompts for it during CAPTURE. Implementation: validator change + skill update, ~1 day.

**Argument for:** Captures the WHY at the moment when the submitter has maximum context — right after discovering the fix. Forces conscious articulation of the decision, which also improves the entry quality overall.

**Argument against:** Most gotchas don't have meaningful alternatives. "macOS sed silently empties files with ** — why not use a different approach?" The alternative is obvious (use Python), making the field redundant. For techniques the field is more valuable, but applying it only to 12+ scores is an arbitrary threshold.

**Success rating: 5/10**

*Why it might not succeed:* Adds friction that may reduce submission rate — the marginal submitter skips the entry rather than filling in the rationale. Claude may generate plausible-sounding but vacuous rationale to satisfy the validator. Doesn't fix the 172 existing entries with no rationale field.

---

### Solution 4 — Provenance links

Submission format includes optional `source_adr` and `source_session` fields. Where an entry originated from a formal decision, it links back. Garden entries display provenance when present. Implementation: convention change, no code required.

**Argument for:** Lets a future reader or agent trace the reasoning back to its origin. An entry that links to ADR-0003 can be evaluated in the context of that decision. Makes the knowledge graph more navigable.

**Argument against:** Most entries originate from ephemeral debugging sessions, not ADRs or tracked decisions. Sessions don't persist. The "source" for 90% of entries is "I hit this bug and found the fix" — there is no persistent artifact to link to. Links to handover docs rot when sessions end.

**Success rating: 3/10**

*Why it might not succeed:* The infrastructure for persistent session artifacts doesn't exist. Requiring ADR coverage for garden entries would massively raise the bar and collapse the submission rate. Optional fields get left blank — which is fine, but then the feature has no coverage.

---

### Solution 5 — Periodic review harvest mode

A dedicated harvest operation — `harvest REVIEW` — lists all entries past their `staleness_threshold`, shows their current content, and prompts for CONFIRM / REVISE / RETIRE. Implementation: ~2 days.

**Argument for:** Systematically catches ALL expired entries, not just the ones that happen to surface during a session. The only solution that guarantees full coverage of the garden's staleness state. Modelled on the existing DEDUPE sweep — same discipline, same cadence.

**Argument against:** Requires a dedicated full-context session. Nobody will run it regularly without external pressure. Reviewing 170+ entries is a significant time investment. Becomes another "should do" that competes with actual work.

**Success rating: 7/10 if run regularly, 3/10 in practice**

*Why it might not succeed:* Discipline problem — the drift counter triggers DEDUPE, but there's no equivalent automatic trigger for staleness review. Without a hard gate ("harvest refuses to merge until overdue reviews are cleared"), it gets deferred indefinitely. A garden can accumulate 50 expired entries and still function, so the pressure to review never feels urgent.

---

### Solution 6 — Version-pinned confidence decay

YAML frontmatter gains a `verified_on` field (e.g. `quarkus: 3.34.2`). forage SEARCH checks the current project's declared stack version against the entry's verified version and flags divergence. Implementation: schema change + skill update, ~2 days.

**Argument for:** Makes knowledge-currency explicitly tied to the tool version it was discovered on. A Quarkus CDI gotcha verified on 3.34.2 that the user is now running on 3.40.0 gets a concrete, actionable flag rather than a generic age warning.

**Argument against:** Most important entries — patterns, methodology, cross-tool techniques — have no meaningful version. Even library-specific entries often apply across many versions. Claude has no reliable way to know what version the user is currently running unless the project declares it explicitly.

**Success rating: 4/10**

*Why it might not succeed:* Version detection is unreliable — most projects don't declare their full stack in a machine-readable way Claude can compare at query time. The `verified_on` field will routinely be left blank or set vaguely ("quarkus: 3.x"). Methodology entries — which are arguably the most important — can't use this at all.

---

**Overall assessment:** Solutions 1 and 5 address the root problem most directly (stale entries get reviewed and retired). Solution 2 is the cheapest intervention and worth doing regardless. Solutions 3, 4, and 6 address the "why" gap but have poor coverage or high friction. The most realistic near-term path: implement solution 2 immediately (hours of work), design solution 5 as a formal harvest mode with a staleness drift counter analogous to the DEDUPE drift counter.

**Promoted to:**
