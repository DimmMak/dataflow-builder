---
name: dataflow-builder
description: >
  Pipeline-build coach for the DATAFLOW framework (Define → Acquire → Tidy → Archive → File → Launch → Observe). Walks the user through writing a real end-to-end stock-data pipeline as a code-along drill: each line of code is rendered with first-letter blanks (coderecall mechanic), user types first letters, system reveals the matching keyword, code is appended to the project file. Composes existing fleet desks at the Acquire stage — price-desk (history + live), fundamentals-desk, technicals-desk, macro-desk, options-desk. Output = a real working project directory, not a tutorial transcript. Observe stage is collaborative — Claude proposes candidate angles, user picks/refines, dashboard wireframe emerges before viz code is drilled.
  NOT for: Socratic WHY-questions on existing pipelines without writing code (use dataflow-tutor — zero code, conceptual). NOT for: gamified DATAFLOW pipeline-stage concept quiz (use dataflow-millionaire — game format, conceptual). NOT for: drilling raw SQL/pandas keywords as muscle memory without building a project (use coderecall — single-line drills, no project output). NOT for: orchestrating existing skills into one-shot execution (use mewtwo — executor, not teacher).
metadata:
  version: 0.1.0
  last_reviewed: 2026-04-29
  spec_version: 0.2.0
composable_with:
  - coderecall
  - price-desk
  - fundamentals-desk
  - technicals-desk
  - macro-desk
  - options-desk
---

# dataflow-builder — pipeline coach

## Purpose

Walks the user through building a real end-to-end stock-data pipeline, line by line, drilling each line as a coderecall first-letter sequence. By the end of one session, the user has:

1. A working project directory with `acquire.py`, `tidy.py`, `db.sql`, `analyze.ipynb`, `README.md`, `requirements.txt`
2. Muscle memory on every keyword they typed during the drill
3. A clear understanding of which fleet desks own which data primitives

The skill never re-implements yfinance pulls, SQL primitives, or pandas verbs that another skill or library already provides. The Acquire stage composes fleet desks (price-desk's `get_history`, fundamentals-desk, technicals-desk, etc.) instead of duplicating them.

## When to trigger

- User types `.dataflow-builder` / `.dfb` → start a new pipeline build
- Natural language: "build me a pipeline for X", "let's build the TSLA pipeline", "code-along through DATAFLOW", "scaffold a stock-data project"
- User pastes a save card → resume from saved DATAFLOW stage

## When NOT to trigger

- User wants conceptual WHY-questions on an EXISTING pipeline → `dataflow-tutor` (Socratic, zero code)
- User wants gamified DATAFLOW concept recall (Millionaire format) → `dataflow-millionaire`
- User wants single-line SQL/pandas keyword drills with no project → `coderecall`
- User wants daily question drill graded cold → `examiner`
- User wants to orchestrate existing skills in one shot (no teaching) → `mewtwo`
- User wants to RUN a finished pipeline, not BUILD one → directly call the underlying desks
- User wants conceptual SQL/pandas teaching via metaphor → `chef` / `.5` / lens skills

## The locked design (v0.1)

These rules came out of the design walk on 2026-04-29. **Do not bypass.**

### Rule 1 — Composition, not reinvention

The Acquire stage MUST use the existing fleet desks for data pulls. Specifically:

| Data need | Desk | Function |
|---|---|---|
| Historical OHLCV (1d/5d/1mo/.../max) | `price-desk` | `get_history(ticker, period, interval, adjust_aftermarket=True)` |
| Live current price (incl. extended hours) | `price-desk` | `get_price(ticker)` |
| Snapshot fundamentals | `fundamentals-desk` | `get_fundamentals(ticker)` |
| Computed technicals (RSI/ADX/ATR/Fib) | `technicals-desk` | `get_technicals(ticker)` |
| Macro indicators (VIX/TNX/DXY) | `macro-desk` | scan public API |
| Options snapshot (IV/OI/skew) | `options-desk` | `get_options(ticker)` |

If a desk doesn't expose what's needed, the right move is to extend the desk (one new function), not to bypass it with raw `yfinance.download()` inside dataflow-builder. Reinventing data layers is an explicit anti-pattern (see Anti-patterns).

### Rule 2 — Code-along via first-letter drill

Every line of code that goes into a project file is rendered as a coderecall-style drill before being saved. Process per line:

1. Claude announces the **goal** of the line in 1 sentence.
2. Renders the line with keywords blanked (`_` × len), non-keywords visible.
3. User types first-letter sequence (space-separated OR no-space).
4. Claude scores, reveals, and appends the line to the project file.

Multi-line blocks bundle 3-6 lines per drill (a "stanza"). Stanza is the atomic save unit.

### Rule 3 — Real project, real files

Output is a real directory structure on disk. Default scaffold:

```
~/Desktop/CLAUDE CODE/<project-name>/
  acquire.py          # data pulls via desks
  tidy.py             # pandas cleaning
  db.sql              # SQLite schema
  ingest.py           # SQLite loader
  analyze.ipynb       # analysis notebook (Observe stage output)
  dashboard.py        # plotly viz (Observe stage output)
  README.md           # objective + how to run
  requirements.txt    # pinned deps
  .gitignore
  data/raw/           # cached pulls
  data/processed/     # tidy parquet outputs
```

User picks the project name at Define. Default if skipped: `<ticker>-pipeline-<YYYYMMDD>`.

### Rule 4 — Collaborative Observe

Observe stage is NOT "Claude shows you 5 charts." It's:

1. Claude proposes 3–5 candidate analytical angles based on the data shape and the Define-stage objective
2. User picks the 1–3 most interesting (or types a different angle in plain English)
3. Together, sketch a dashboard wireframe (sections, what each shows, why)
4. THEN drill the viz code line-by-line per the standard mechanic

Skipping the wireframe step means visualizing data without intent — explicit anti-pattern.

### Rule 5 — Save card persistence

Long sessions can pause between stages. Save card matches dataflow-millionaire/coderecall format. State persisted: project path, current stage, drill progress within stage, weak spots, save-card lineage hashes.

## DATAFLOW stage contracts

Each stage has a fixed contract — what it consumes, what it produces, what files it writes.

### D — Define (no drill, text only)

| Input | Output |
|---|---|
| Ticker(s), objective sentence | `README.md` skeleton, project dir created |

Skill asks for ticker + objective in plain English. No code drilled.

### A — Acquire (composes desks)

| Input | Output |
|---|---|
| Ticker(s), period, fields needed | `acquire.py` |

Drilled stanzas — a typical session writes 3–5 stanzas:
- Stanza 1: imports (`import pandas`, `from price import get_history`, etc.)
- Stanza 2: history pull(s) via `get_history`
- Stanza 3: fundamentals pull (if objective requires)
- Stanza 4: technicals pull (if objective requires)
- Stanza 5: cache to `data/raw/<ticker>.parquet`

Composition rule: if the objective doesn't NEED a data type, don't pull it. (Don't drill fundamentals code if the objective is purely price-action.)

### T — Tidy (pandas)

| Input | Output |
|---|---|
| Raw DataFrames from Acquire | `tidy.py` |

Drilled stanzas:
- Stanza 1: load from `data/raw/`
- Stanza 2: handle missing values (`fillna` / `dropna` based on objective)
- Stanza 3: type coercion (dates, numerics)
- Stanza 4: derive features (rolling means, returns, etc.) per objective
- Stanza 5: save to `data/processed/<ticker>.parquet`

### A — Archive (SQLite)

| Input | Output |
|---|---|
| Tidy parquet, schema | `db.sql` + `ingest.py` |

Drilled stanzas:
- Stanza 1: `db.sql` — `CREATE TABLE` for prices, fundamentals, etc.
- Stanza 2: `ingest.py` — open SQLite, read parquet, `.to_sql()` insert
- Stanza 3: index creation for query patterns the Observe stage will use

### F — File (export)

| Input | Output |
|---|---|
| SQLite tables | `data/exports/*.csv` or `*.parquet` |

Drilled stanzas:
- Stanza 1: query the SQLite for the canonical analytical slice
- Stanza 2: export to CSV/parquet for portability

### L — Launch (git)

| Input | Output |
|---|---|
| Project dir | git repo + first commit |

Drilled stanzas:
- Stanza 1: `.gitignore` keywords (`__pycache__`, `data/raw/`, `*.parquet`)
- Stanza 2: `git init` + `git add` + `git commit -m "..."` (terminal commands rendered in shell drill style)

NO `git push` in v0.1 — that's a remote action requiring user confirmation.

### O — Observe (collaborative)

| Input | Output |
|---|---|
| Tidy data, objective | `analyze.ipynb` + `dashboard.py` |

This stage breaks the standard line-by-line drill rhythm. Subphases:

1. **Angle proposal** (Claude → user). Claude reads the data shape + objective, proposes 3–5 candidate analytical angles in plain English. Example: *"Looking at TSLA's 2-year price history with macro context, I see four angles worth exploring: (1) divergence days vs sector ETF, (2) gap-up/gap-down clustering around earnings, (3) post-earnings drift comparison to peers, (4) volatility regime shifts vs VIX. Which 1–3 do you want to chase?"*
2. **User selection** (user → Claude). User picks angles or proposes new ones in plain English.
3. **Wireframe** (Claude + user). Sketch the dashboard layout in ASCII: rows, columns, what each cell shows. User signs off.
4. **Code drill** (back to mechanic). Line-by-line drill of the pandas analysis + plotly viz code that produces the wireframed dashboard.

## Save card format

```
--- DATAFLOW BUILDER SAVE ---
Project: [project-dir-path]
Tickers: [comma list]
Objective: [one sentence]
Current Stage: [D|A|T|A|F|L|O]
Stage Progress: [stanza N of M]
Weak Spots: [comma list of keywords missed]
Composed Desks: [comma list — price-desk, fundamentals-desk, etc.]
Last Session: [ISO date]
--- END ---
```

## Anti-patterns

- **Reinventing data pulls.** If `price-desk.get_history()` exists, the Acquire stage MUST use it. Drilling raw `yfinance.download()` lines = anti-pattern. Extend the desk if needed.
- **Skipping the wireframe in Observe.** Going straight to viz code without a wireframe = data theater. The wireframe IS the contract.
- **Drilling code that won't run.** Every drilled stanza must produce code that the user can immediately execute against their real environment. If a stanza references a function that doesn't exist, the design is broken — fix the data layer, then drill.
- **Mid-stage stage-switching.** If user wants to skip from Acquire to Observe before Tidy is done, the data isn't ready. Skill refuses with: "Tidy hasn't run; the Observe queries will fail. Finish Tidy or save and resume."
- **Over-drilling identifiers.** Project names, table names, column names, ticker strings = auto-revealed (not drilled). Only standard Python/SQL/pandas keywords + desk-API function names get blanked. Drilling user-chosen names is noise.
- **Auto-pushing to git.** Launch stage drills `git init`/`add`/`commit` — never `push`. Push is user-authorized.
- **Pre-building L2/L3 of the drill mechanic.** Drill stays at coderecall L1 (first-letter cued recall) only in v0.1 — partial scaffolding (L2) and blind recall (L3) deferred per `feedback_wait_time_decay.md`.

## Exit conditions

- All 7 stages complete → save card emitted, project dir final, README updated
- User types `pause` / `quit` / `stop` → save card emitted at current stage, exit
- 60-min inactivity → auto-pause with save card
- User invokes another skill → auto-pause
- Composition failure (a required desk function missing) → STOP, surface the gap, ask user whether to extend the desk before continuing

## Non-goals (deferred)

- **L2/L3 drill modes.** Per wait-time-decay rule, defer until L1 reps prove insufficient.
- **Auto-running the pipeline end-to-end after build.** v0.1 builds; running is user-driven.
- **GitHub push automation.** Launch stage stops at first commit. Pushing is user's call.
- **Multi-ticker pipelines.** v0.1 is one ticker per project. Multi-ticker batch scaffold = v0.2.
- **Live trading integration.** This skill teaches pipeline construction. Trade execution is firmly out of scope.
- **Custom DSL / non-Python languages.** v0.1 is Python + SQLite + pandas. R / Rust / Julia ports = v0.2 if requested.
- **Webapp/dashboard hosting.** v0.1 produces local Plotly + notebook. Streamlit/Vercel deployment = v0.2.

## Future modes (deferred — add only when triggered)

| Feature | Trigger condition |
|---|---|
| L2 partial-letter scaffolding mode | User reports L1 feels reflex-tight after 30+ stanzas drilled |
| Blind L3 (full recall mode for a stanza) | User completes L2 with ≥80% accuracy across 3 sessions |
| Multi-ticker batch | User builds 3+ single-ticker pipelines and asks "can it do all of these?" |
| Streamlit/Vercel deploy stage | User asks for shareable dashboards |
| R/Julia language packs | User explicitly requests |
| Web-app companion (browser drill) | Chat round-trip latency limits stanza throughput |

## Composes with

- **coderecall** — drill mechanic source. dataflow-builder applies coderecall's L1 first-letter blanking to multi-line code stanzas instead of single-line SQL/pandas snippets.
- **price-desk** — `get_history()`, `get_price()` for the Acquire stage.
- **fundamentals-desk** — fundamentals snapshot for the Acquire stage.
- **technicals-desk** — RSI/ADX/ATR/Fib for the Acquire stage.
- **macro-desk** — VIX/TNX/DXY for macro-context pipelines.
- **options-desk** — options snapshot for vol-aware pipelines.
- **examiner** (optional) — weak-spots from the build can feed examiner's drill bias.

## Trigger phrase reference

| User says | dataflow-builder does |
|---|---|
| `.dataflow-builder` / `.dfb` | start new pipeline build |
| `.dfb resume` | scan for save card and resume |
| `next` mid-stanza | advance to next line |
| `same` mid-stanza | redo current line drill |
| `stage <letter>` | jump to stage (D/A/T/A/F/L/O) — refuses if dependencies unmet |
| `pause` / `quit` / `stop` | emit save card, exit |

## Failure modes guarded against

- **Vapor data references** (e.g. drilling code that calls `price_desk.foo()` when no such function exists) — Anti-pattern #3 + Exit condition on missing function
- **Pipeline-without-objective drift** (jumping to code without Define) — D stage is mandatory
- **Wireframe-skipping in Observe** — Anti-pattern + locked rule #4
- **Reinventing yfinance** — Anti-pattern + locked rule #1
- **Memory dependency** — all rules + scoring + save format live in this SKILL.md
- **Cross-skill drift** — collision fence in description names 4 sibling skills explicitly
