# dataflow-builder — changelog

## v0.1.0 — 2026-04-29

**Initial release.** Pipeline-build coach that walks the user through writing a real end-to-end stock-data project, drilling each line as a coderecall first-letter recall.

### What shipped

- **DATAFLOW stage flow**: Define → Acquire → Tidy → Archive → File → Launch → Observe
- **coderecall mechanic** applied per code line: blanks-only display (`_` × len), user types first-letter sequence, system reveals
- **Composition-first at Acquire**: calls existing fleet desks (`price-desk.get_history`, `fundamentals-desk`, `technicals-desk`, `macro-desk`, `options-desk`) instead of reinventing yfinance integration
- **Real project output**: scaffolds `~/Desktop/CLAUDE CODE/<project-name>/` with `acquire.py`, `tidy.py`, `db.sql`, `ingest.py`, `analyze.ipynb`, `dashboard.py`, `README.md`, `requirements.txt`, `.gitignore`
- **Collaborative Observe stage**: Claude proposes 3–5 candidate analytical angles, user picks/refines, ASCII dashboard wireframe sketched together, viz code drilled per wireframe
- **Save card persistence** (matches `coderecall` + `dataflow-millionaire` pattern)
- **Project state JSON** at `<project>/.dataflow-builder/state.json` — durable, survives sessions
- **Weak-spots schema** shared with `coderecall` + `examiner` (`data/weak-spots.jsonl`)
- **`desk-mismatch` miss class** — new in dataflow-builder; tracks "knew the keyword, picked the wrong desk for the context"

### What did not ship (deferred per `feedback_wait_time_decay.md`)

- L2/L3 drill modes (partial scaffolding, blind recall) — coderecall is L1-only too in v0.1
- Multi-ticker pipelines (single ticker per project for v0.1)
- Streamlit / Vercel / Plotly Dash deploy stage
- R / Julia language packs
- Webapp companion (chat-only for v0.1)
- Auto-running the built pipeline end-to-end after Launch
- `git push` (Launch stops at first commit; user authorizes pushes)

### Why this skill exists (origin)

`dataflow-tutor` (sibling) teaches the WHY of pipelines via Socratic questions but writes ZERO code. It's a conceptual tutor, not a build tutor. Throughout the fleet there was no skill that walked a user through actually WRITING a pipeline — fund work (blue-hill-capital, royal-rumble, stock-analyzer projects) had been done manually each time, repeated pain.

The 2026-04-29 design walk surfaced two locked decisions:

1. **Use coderecall's mechanic** rather than building a second Socratic skill. By drilling each line, the user gets muscle memory + a real artifact in one workflow.
2. **Compose existing desks at Acquire** rather than teaching raw `yfinance.download()`. The desks are the canonical data layer; teaching anything else trains the wrong pattern.

A precondition: `price-desk` did not have a historical-pull function before this skill was designed. `get_history(ticker, period, interval, adjust_aftermarket=True)` was added to `price-desk` on 2026-04-29 to fill that gap. That helper now serves dataflow-builder + future history-needing skills (royal-rumble historicals, backtests, etc.) without duplication.

### Files

| File | Purpose |
|---|---|
| `SKILL.md` | Skill entry point — locked rules, stage contracts, save-card spec |
| `ARCHITECTURE.md` | Why these structural choices (composition-first, coderecall mechanic, real files, collaborative Observe) |
| `SCHEMA.md` | Save card, project-state JSON, drilled-stanza shape, weak-spots, hash format |
| `CHANGELOG.md` | This file |
