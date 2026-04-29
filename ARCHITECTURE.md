---
name: dataflow-builder — architecture
description: Why the v0.1 skill is structured the way it is. Captures the design constraints from the 2026-04-29 design walk so future-you doesn't reverse-engineer them out.
---

# dataflow-builder — architecture (v0.1)

## Five structural choices, in order of how much they constrain everything else

### 1. Composition-first at the data layer

Every other choice in this skill follows from this one: **the Acquire stage MUST call existing fleet desks instead of reinventing yfinance integration.**

The fleet already has 5 production-quality desks that wrap yfinance with logging, freshness checks, error handling, and session-aware extended-hours adjustment:

- `price-desk.get_history()` — historical OHLCV with `adjust_aftermarket=True` default (added 2026-04-29 to fill the gap dataflow-builder needed)
- `price-desk.get_price()` — live price (incl. pre/post-market)
- `fundamentals-desk` — fundamentals snapshot
- `technicals-desk.get_technicals()` — RSI/ADX/ATR/Fib
- `macro-desk` — macro context
- `options-desk.get_options()` — vol surface

**Why this matters:** if the Acquire stage drilled raw `yfinance.download()` lines, the user would learn the WRONG pattern (direct API access bypassing the freshness/quality layer). The lesson IS the composition — that's what makes pipelines maintainable as the data layer evolves.

**Implication:** before scaffolding any stanza in Acquire, check that the desk function exists. If it doesn't, extend the desk (one new function, alongside existing helpers). NEVER bypass with raw yfinance. This rule is locked in SKILL.md as Rule 1 + Anti-pattern #1.

### 2. coderecall mechanic, not Socratic questions

The DATAFLOW concept is already taught Socratically by `dataflow-tutor` (zero code, all WHY-questions). Building a SECOND Socratic skill would duplicate that. Instead, dataflow-builder uses the **coderecall first-letter mechanic** — the user types code, blanks become muscle memory, the project file fills as drills succeed.

This means: by the end of one session, the user has BOTH (a) a real working project AND (b) reflex-level recall on every keyword they typed. Two outcomes, one workflow.

**Implication:** dataflow-builder DEPENDS on coderecall as a fleet primitive (composes_with relationship). If coderecall changes its keyword set, dataflow-builder inherits that change. Keep them in sync.

### 3. Real project dir, not a tutorial transcript

The output is files on disk, not a chat session log. After each successful drill, the line is appended to the actual `.py` / `.sql` / `.ipynb` file in `~/Desktop/CLAUDE CODE/<project-name>/`.

**Why files on disk:**

1. The artifact survives the chat session
2. The user can run it immediately (`python3 acquire.py`)
3. Git tracking (Launch stage) gets a real diff to commit
4. Future sessions can resume by reading current file state, not just save card text
5. Fleet patterns (poly-repo per project) integrate naturally — project goes to GitHub when user pushes

**Implication:** every stanza must produce executable code, not pseudocode or comments. If a stanza can't run, the design is wrong — fix the data layer first.

### 4. Collaborative Observe, not autonomous insights

The Observe stage breaks the line-by-line drill rhythm intentionally. Skill proposes 3–5 candidate analytical angles based on the data shape and Define-stage objective; user picks/refines; together they sketch a dashboard wireframe in ASCII; THEN drill the viz code.

**Why this is non-negotiable:**

- "Claude shows you 5 charts" produces visualizations without intent
- "User-only-picks" without proposals leaves the user staring at data with no leads
- The wireframe step is where the OBJECTIVE becomes a CONCRETE DASHBOARD — that translation is the highest-value cognitive step in the whole pipeline

**Implication:** the Observe stage takes longer than the others. Don't compress it. The wireframe IS the contract; viz code is just transcription of the wireframe.

### 5. Save card persistence, not session-bound state

Pipelines take more than one chat session. dataflow-builder uses the same save card pattern as coderecall + dataflow-millionaire — paste-mediated, no disk dependency.

State persisted in save card:

- Project directory path (so resume opens the right files)
- Current stage + stanza progress
- Tickers + objective
- Composed-desks list (which desks were called)
- Weak spots
- Save-card lineage hashes (to prevent stanza repetition across resumes)

**Implication:** sessions can be days apart. Save card content must be self-sufficient — no Claude-side memory needed to resume. (Project directory state on disk is the source of truth for "what's already drilled.")

## Why these compositions (and not others)

- **`coderecall`** — drill mechanic source. dataflow-builder is, in one sense, "coderecall scaled up to multi-line stanzas with project file persistence." Hard composition.
- **`price-desk`** — owns historical OHLCV + live price. The function `get_history()` was added 2026-04-29 specifically for dataflow-builder's needs (it filled a real gap, not skill-specific bloat).
- **`fundamentals-desk`** / **`technicals-desk`** / **`macro-desk`** / **`options-desk`** — composed at Acquire when objective requires their data. Soft composition — only fired if objective needs that data type.
- **`examiner`** (optional) — weak-spots from drilled stanzas can feed examiner's drill bias. One-way data flow only.

**NOT composed with:**

- `dataflow-tutor` — same domain (DATAFLOW framework) but opposite mechanic (Socratic vs code-along). They are siblings, not a parent-child stack.
- `dataflow-millionaire` — game format on DATAFLOW concepts. Different cognitive task entirely.
- `mewtwo` — orchestrator. dataflow-builder is the artifact; mewtwo could orchestrate dataflow-builder + dataflow-tutor + coderecall in one sequence, but dataflow-builder doesn't reach into mewtwo.

## State surface

| State | Lives in | Lifetime |
|---|---|---|
| Project directory | `~/Desktop/CLAUDE CODE/<project>/` | persistent — survives sessions |
| Current stanza in current stage | session memory | one stage |
| Save card | paste-block | persistent — across sessions |
| Drill score / streak / weak spots | save card + project's `.dataflow-builder/state.json` | persistent |
| Stanza-line lineage (for resume) | derived from current file state on disk | persistent |

The project directory IS the durable state. Save card is the index INTO the project directory.

## Stage-flow rationale (DATAFLOW order)

The order isn't arbitrary. Each stage's output feeds the next:

| Stage | Produces | Why this order |
|---|---|---|
| Define | objective, ticker, project name | nothing else can start without these |
| Acquire | raw data | tidy needs raw to tidy |
| Tidy | clean DataFrames | archive needs clean for sane schemas |
| Archive | SQLite | file/observe need queryable storage |
| File | exports | survives if SQLite gets corrupted; portable handoff |
| Launch | git-versioned project | project becomes shareable artifact |
| Observe | dashboards + insights | answers the Define-stage objective |

**Strict ordering rule:** stage skipping refused (per Anti-pattern: mid-stage stage-switching). User can BACKTRACK to fix (e.g. revisit Acquire to add another data source) but can't SKIP forward over an unmet dependency.

## What v0.2+ might add (deferred, with trigger conditions)

| Feature | Trigger condition |
|---|---|
| L2/L3 drill modes (partial scaffolding, blind recall) | User reports 30+ stanzas drilled and L1 feels reflex-tight |
| Multi-ticker pipelines | User builds 3+ single-ticker projects and asks "all at once?" |
| Streamlit/Vercel deploy at Launch | User asks for shareable dashboards |
| R/Julia language packs | User explicitly requests |
| Webapp companion (browser drill) | Chat round-trip latency limits stanza throughput |
| Auto-running the built pipeline | User asks "now run it" repeatedly |
| Integration with project-manager skill | User reports forgetting which projects are in flight |

Each row is a parked design — don't pre-build, don't pre-design. When the trigger hits, design from current need, not stale intent (per `feedback_wait_time_decay.md`).
