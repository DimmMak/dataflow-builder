---
name: dataflow-builder — schemas
description: Save card format, project-state JSON, drilled-stanza tracking, and shared weak-spots schema with coderecall + examiner.
---

# dataflow-builder — schemas (v0.1)

`schema_version: 0.1.0`

## 1. Save card (text block, paste-mediated)

Emitted at session pause / stage end. User pastes at next session start.

```
--- DATAFLOW BUILDER SAVE ---
Project: [absolute path to project dir]
Project Name: [string]
Tickers: [comma-separated uppercase, e.g. TSLA,XLY]
Objective: [one sentence in user's words]
Current Stage: [D|A|T|A|F|L|O]
Stage Progress: [stanza N of M]
Composed Desks: [comma list — price-desk,fundamentals-desk,...]
Total Stanzas Drilled: [integer]
Weak Spots: [comma list of keywords missed]
Mastered Keywords: [comma list — 5+ correct in a row]
Last Session: [ISO 8601 date]
Save Lineage Hash: [8-char hex — parent save card hash]
--- END ---
```

**Field rules:**

| Field | Type | Validation |
|---|---|---|
| `Project` | absolute path | must exist on disk; relative paths rejected |
| `Project Name` | string | max 64 chars, kebab-case preferred |
| `Tickers` | uppercase list | 1–5 chars each, comma-separated |
| `Objective` | string | max 280 chars, must be ≥ 10 chars |
| `Current Stage` | enum | one of `D / A / T / A / F / L / O` (DATAFLOW letters) |
| `Stage Progress` | string | format: `<int> of <int>` |
| `Composed Desks` | string list | each must be a real fleet skill name |
| `Last Session` | date | ISO 8601 (`YYYY-MM-DD`) |
| `Save Lineage Hash` | hex | exactly 8 chars |

**Sanity checks (refuses to load):**
- If `Project` path doesn't exist on disk → user moved it; ask for new path
- If `Stage Progress` exceeds total expected stanzas for current stage → save card was hand-edited; warn before loading
- If `Composed Desks` lists a skill that's not in `~/.claude/skills/` → desk was deleted; surface error

## 2. Project state — `<project>/.dataflow-builder/state.json`

Optional on-disk state inside the project directory itself. Persists across save-card cycles.

```json
{
  "schema_version": "0.1.0",
  "project_name": "tsla-divergence-pipeline-20260429",
  "tickers": ["TSLA", "XLY"],
  "objective": "Find the days TSLA close diverged most from sector ETF XLY.",
  "stages_completed": ["D", "A", "T"],
  "current_stage": "A",
  "current_stanza_index": 2,
  "drilled_stanzas": [
    {
      "stage": "A",
      "stanza_index": 0,
      "file": "acquire.py",
      "line_range": [1, 4],
      "keywords_drilled": ["import", "as", "from", "import"],
      "completed_at": "2026-04-29T15:30:00Z"
    }
  ],
  "weak_spots": [
    {"keyword": "groupby", "miss_count": 2, "last_missed": "2026-04-29T15:42:00Z"}
  ],
  "composed_desks": ["price-desk", "fundamentals-desk"],
  "save_lineage": ["abc12345", "def67890"]
}
```

**Field rules:**

| Field | Type | Notes |
|---|---|---|
| `schema_version` | string | must match this file |
| `stages_completed` | string list | DATAFLOW letters in order |
| `drilled_stanzas` | object list | append-only; one entry per drill completion |
| `weak_spots` | object list | aggregated, deduplicated by `keyword` |
| `save_lineage` | hex list | full chain of save card hashes |

**Concurrency:** dataflow-builder writes state.json after each successful drill. If two sessions ever ran simultaneously (shouldn't happen — one project, one session), last-writer-wins.

## 3. Stanza definition

Each drillable code block has a fixed shape:

```json
{
  "stanza_id": "A.1.imports",
  "stage": "A",
  "stage_step_index": 1,
  "file": "acquire.py",
  "purpose": "Import pandas and the price-desk history helper.",
  "code": [
    "import pandas as pd",
    "from price import get_history"
  ],
  "blanks": [
    {"line": 0, "tokens": [{"text": "import", "blank": true}, {"text": "pandas", "blank": false}, {"text": "as", "blank": true}, {"text": "pd", "blank": false}]},
    {"line": 1, "tokens": [{"text": "from", "blank": true}, {"text": "price", "blank": false}, {"text": "import", "blank": true}, {"text": "get_history", "blank": false}]}
  ]
}
```

**Tokenizer rules** (inherited from coderecall, extended):

- Standard Python keywords blank: `import`, `from`, `as`, `def`, `return`, `if`, `else`, `elif`, `for`, `in`, `while`, `try`, `except`, `with`, `lambda`, `class`, `pass`, `yield`
- Pandas/numpy verbs blank: `pd`, `df`, `np`, `read_csv`, `read_sql`, `groupby`, `agg`, `merge`, `concat`, `pivot_table`, `fillna`, `dropna`, `sort_values`, `reset_index`, `set_index`, `apply`, `map`, `value_counts`, `head`, `tail`, `iloc`, `loc`, `rolling`, `mean`, `sum`, `count`, `std`
- SQL keywords blank (when in `db.sql` files): inherited from coderecall's SQL set
- Desk function names blank: `get_history`, `get_price`, `get_fundamentals`, `get_technicals`, `get_options`, `get_macro`
- Identifiers (project names, ticker strings, column names, table names) — NOT blanked, auto-revealed
- String literals — NOT blanked, auto-revealed
- Numbers / operators / punctuation — NOT blanked, auto-revealed

## 4. Weak-spots schema (shared with coderecall + examiner)

Append-only JSONL at `data/weak-spots.jsonl` (project-local).

**One JSON object per line:**

```json
{"ts": "2026-04-29T15:42:11Z", "skill": "dataflow-builder", "stage": "T", "stanza_id": "T.3.fillna", "language": "python", "keyword": "fillna", "context_before": "df.", "context_after": "(", "miss_type": "recall"}
```

| Field | Type | Notes |
|---|---|---|
| `ts` | ISO 8601 | UTC |
| `skill` | string | always `dataflow-builder` |
| `stage` | enum | DATAFLOW letter |
| `stanza_id` | string | unique within project |
| `language` | enum | `python` / `sql` |
| `keyword` | string | the missed keyword |
| `context_before` | string | preceding token |
| `context_after` | string | following token |
| `miss_type` | enum | `recall` / `clause-order` / `desk-mismatch` |

**`desk-mismatch` miss type** — new in dataflow-builder, not in coderecall:

When the user types a desk function name (e.g. `g` for `get_history`) but the wrong desk is implied by context (e.g. they're in fundamentals territory but typed history's first letter), score as `desk-mismatch`. Different remedy from plain recall miss: drill the composition map (which desk owns what), not the keyword.

## 5. Question / stanza hash (lineage tracking)

8-char hex of the canonical code string (whitespace-collapsed):

```python
import hashlib
def stanza_hash(code_lines: list[str]) -> str:
    canonical = " ".join(" ".join(line.split()) for line in code_lines)
    return hashlib.sha1(canonical.encode()).hexdigest()[:8]
```

Used to: prevent re-drilling already-drilled stanzas during resume.

## 6. Migration notes

**v0.1 → v0.2:**

- Add `levels` field to drilled_stanzas (when L2/L3 unlock): `{"l1": true, "l2": false, "l3": false}` per stanza
- Add `language` field at project root if multi-language pipelines emerge
- Bump schema_version to `0.2.0`
- v0.1 state.json files load with default `levels: {l1: true, l2: false, l3: false}`

**v0.1 → v0.2 (multi-ticker):**

- `tickers` field already a list — no migration needed
- Add `per_ticker_state` map if state diverges per ticker

Schema bumps explicit. No silent additions.
