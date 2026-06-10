# Post-Phase-2: Three things that needed fixing

Phase 2 shipped. The CI pipeline validates submissions, integrates on merge,
and the garden is at 169 entries. What the next session revealed: three decisions
made for speed that deserved revisiting.

## The harvest — and the invisible entries

First, maintenance. Thirteen submissions had accumulated in `submissions/` — most
of them from the implementation sprint itself, capturing things we hit while
building. We merged them, ran DEDUPE, and found that four entries in
`tools/llm-testing.md` had been integrated without `**ID:**` headers.

Claude ran the validator to confirm. It also surfaced something stranger: an
existing entry (GE-0091, about `validate_garden.py` itself) contained the phrase
"the closing ` ``` ` fence marker" in its prose text. The validator's
`strip_code_fences` regex treats literal triple backtick as a fence delimiter —
so everything from that point to the next real fence in the file got stripped,
including all the `**ID:**` lines that followed. Those entries were invisible
to the validator.

The error was `ERROR: Index references GE-XXXX but no matching **ID:** in any garden file`. The entries existed. The fix was renaming "the closing ` ``` `" to "the closing fence marker" in one line of prose. One change, four IDs reappeared.

## The path problem — 62 occurrences

The garden lived at `~/claude/knowledge-garden/`. Every bash example in the forage
and harvest skills hardcoded it. Two assumptions embedded: that the path would
contain "claude", and that the directory would be called "knowledge-garden".

Neither is defensible. We introduced `HORTORA_GARDEN` as an env var, defaulting
to `~/.hortora/garden` — a clean, namespaced location with no tool-specific
assumptions baked in. Then migrated the actual garden there. `~/claude/knowledge-garden`
stays as a symlink for backward compat. Sixty-two occurrences replaced across
the skills and scripts.

## The ID scheme — from issues to birthday arithmetic

The original GE-ID design: GitHub Issues. Forage creates issue #124,
that becomes `GE-0124`. The problem surfaced when I asked the obvious question:
what happens at a million entries? GitHub doesn't scale as an ID counter.

We dropped issues. That still left the sequential counter in `GARDEN.md` —
which requires coordination. Two concurrent submissions both read
`Last assigned ID: GE-0172`, both try to claim `GE-0173`, one wins, the other
fails CI and rebases. Manageable at small-team scale. Friction at any meaningful
adoption.

The replacement: `GE-YYYYMMDD-xxxxxx` — today's date plus six random hex
characters. Claude ran the numbers: 16⁶ is 16.7 million possibilities per day,
50% collision probability at around 5,000 submissions on the same day. At
realistic single-garden scale the risk is negligible — under 0.003% at
1,000 submissions per day.

No counter to read. No coordination required. Generated locally:

```python
import secrets
from datetime import date
print(f"GE-{date.today().strftime('%Y%m%d')}-{secrets.token_hex(3)}")
```

The 172 existing sequential entries stay as-is. The validator now accepts both
formats. This is ADR-0003.

## The feedback I needed to hear

A reviewer raised something I hadn't framed clearly: entries capture WHAT
(symptom, fix) but not WHY this fix over alternatives, or WHEN the knowledge
expires. Without staleness enforcement, an AI agent acts on a three-year-old
gotcha with the same confidence as a fresh one. "corporate bureaucracy with agentic powers."

The `staleness_threshold` field has existed since Phase 1 but does nothing —
entries past threshold are indistinguishable from current ones.

I documented six solutions in `IDEAS.md`, each with an argument for, an argument
against, a success rating, and specific reasons it might not work. The two most
actionable: staleness annotations in forage SEARCH results (cheap, adds epistemic
humility at point of use) and a periodic harvest REVIEW mode (thorough, but
discipline-dependent). Neither is built. But the problem is named, which is
where fixing starts.

## What's still open

The CI workflows in `Hortora/garden` still reference the old issue-based flow.
Updating them is what unblocks end-to-end testing of GitHub mode.
