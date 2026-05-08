# CLIbrary — Manifests

> A standardized manifest registry for agent-callable CLI tools.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Manifests](https://img.shields.io/badge/manifests-69-00e5a0)](#current-manifests)

---

## What is this?

CLIbrary is an open standard for describing CLI tools so that AI agents can discover and invoke them — without stuffing every tool's documentation into the prompt.

Each manifest is a single JSON file that answers four questions:

| Question | Field |
|----------|-------|
| What does it do? | `description` |
| When should I use it? | `intent_triggers` |
| What does it need? | `input_schema` |
| What does it return? | `output_schema` |

---

## Why?

Today, when an agent has 100+ tools available, the LLM has to read every tool's description to pick one. That wastes thousands of tokens per call and accuracy collapses ("lost in the middle").

CLIbrary separates **discovery** (manifest) from **execution** (the tool itself). A small embedding-based router reads the manifests once, then routes natural-language intents to the right CLI in ~36 ms with ~150 tokens.

### Validated routing performance

| Eval set | Strategy | N | Top-1 | Top-3 |
|----------|----------|---|-------|-------|
| in_domain | clibrary | 500 | 86.80% | 88.00% |
| paraphrase | clibrary | 1,500 | 81.80% | 83.40% |
| cross_domain | clibrary | 328 | 68.90% | — |
| adversarial | clibrary_top3 (post R3) | 204 | 86.27% | **100%** |

Adversarial set was iteratively patched (3 rounds of `intent_triggers` improvement) — no model fine-tuning involved.

### Model size stops mattering

**paraphrase, N=1,500**

| Model | CLI Acc | Top-3 | Pick | Params | E2E |
|-------|---------|-------|------|--------|-----|
| qwen2.5:3b  | 81.80% | 83.40% | 98.08% | 69.00% | 61.27% |
| qwen2.5:14b | 81.67% | 83.27% | 98.08% | 68.87% | 61.20% |
| **Δ** | +0.13pp | +0.13pp | ±0 | +0.13pp | +0.07pp |

**in_domain, N=500** — even sharper, because ~98.6% of queries hit Path A (template fill, no LLM):

| Model | CLI Acc | Top-3 | A-path hit |
|-------|---------|-------|------------|
| qwen2.5:3b  | 86.80% | 88.00% | 98.64% |
| qwen2.5:14b | 86.80% | 88.00% | 98.64% |

Both models are **identical to four decimal places** — when the router skips the LLM 98.6% of the time, model size literally cannot matter.

Once retrieval is handled by the router, scaling the LLM 4.7× gives essentially zero accuracy gain. Compute spent on a bigger model is wasted — invest it in better `intent_triggers`. Full breakdown: [clibrary.dev/benchmark.html](https://clibrary.dev/benchmark.html).

---

## Use it

The reference router is on PyPI:

```bash
pip install clibrary-hub
clibrary-build-index --manifest-dir ./manifests
```

```python
from clibrary_hub import router

print(router.route("convert this MP4 to a GIF"))
# {"action": "route", "cli": "video-to-gif", "params": {...}, "latency_ms": 36}
```

Source: [clibrary-hub/CLIbrary](https://github.com/clibrary-hub/CLIbrary)

---

## How routing works

```
Agent: "Convert this MP4 to a GIF"
           ↓
   Embed intent (multilingual-e5-base, 768-d)
           ↓
   FAISS cli_index → top-3 candidates
           ↓
   MaxSim re-rank over trigger_index
           ↓
   Stage 2: example_index → best matching example
           ↓
   sim ≥ 0.85 → Path A (template fill, no LLM)
   sim < 0.85 → Path B (LLM extracts params)
           ↓
   {"cli": "video-to-gif", "params": {...}}
```

No LLM call needed for ~80% of queries. Each manifest's `intent_triggers` and `examples` act as the routing index.

---

## Structure

```
manifests/
├── ai-ml/          # Model inference, embeddings, transcription
├── data/           # SQL, CSV, databases, ETL
├── devops/         # Docker, git, CI/CD, code analysis
├── media/          # Video, image, audio processing
├── web/            # HTTP, scraping, APIs
├── security/       # Scanning, auditing, encryption
├── productivity/   # Files, text, conversion
├── finance/        # Market data, invoices
├── science/        # Units, stats, computation
└── networking/     # DNS, IP, diagnostics
```

---

## Manifest Format

```json
{
  "name": "sql-runner",
  "version": "1.0.0",
  "category": "data",
  "description": "Execute SQL queries against a database and return structured results",
  "intent_triggers": [
    "query a database",
    "run a SQL statement",
    "get records from a table",
    "fetch data from postgres",
    "execute a database query",
    "查資料庫",
    "跑 SQL"
  ],
  "input_schema": {
    "query": {
      "type": "string",
      "required": true,
      "description": "SQL query to execute"
    },
    "db_url": {
      "type": "string",
      "required": false,
      "description": "Database connection URL (defaults to $DATABASE_URL)"
    },
    "output_format": {
      "type": "enum",
      "required": false,
      "default": "json",
      "description": "Output format: json, csv, or table"
    }
  },
  "output_schema": {
    "format": "json",
    "fields": ["rows", "columns", "row_count"]
  },
  "examples": [
    {
      "intent": "Get total sales from last week",
      "invocation": "sql-runner --query \"SELECT SUM(amount) FROM orders WHERE created_at > NOW() - INTERVAL 7 DAY\""
    }
  ],
  "error_codes": {
    "1": "Connection failed",
    "2": "Query syntax error",
    "3": "Permission denied"
  },
  "tags": ["sql", "database", "query", "etl"]
}
```

### Required fields

| Field | Type | Rule |
|-------|------|------|
| `name` | string | lowercase, hyphen-separated |
| `version` | string | semver `x.y.z` |
| `category` | enum | one of 10 categories |
| `description` | string | 10–200 characters |
| `intent_triggers` | array | ≥ 5 items, mix of languages welcome |
| `input_schema` | object | each param needs `type` |
| `output_schema` | object | needs `format` field |
| `examples` | array | ≥ 1 item with working `invocation` |

---

## Current Manifests

| Category | Count | Examples |
|----------|-------|---------|
| `devops` | 35 | docker-cleanup, lint-guard, repo-map, sandbox-run… |
| `productivity` | 15 | doc-to-md, i18n-sync, task-prioritizer, zip… |
| `ai-ml` | 10 | whisper-transcribe, rag-forge, ocr-feed, token-meter… |
| `media` | 5 | video-to-gif, tts, img-batch, html-voice… |
| `data` | 3 | sql-runner, csv-export, schema-brief |
| `web` | 3 | api-compat, api-slim, web-clean |
| `security` | 2 | dep-audit, sec-scan |
| `finance` | 1 | finmind |
| **Total** | **69** | More being added — PRs welcome |

---

## Contributing

PRs are welcome for any CLI tool an AI agent might reasonably want to use.

**Steps:**

```bash
# 1. Fork this repo

# 2. Create your manifest
cp ai-ml/whisper-transcribe.json <category>/<your-tool>.json

# 3. Edit and validate locally
pip install clibrary-hub
clibrary-build-index --manifest-dir .

# 4. Open a PR
```

**Rules:**

- `intent_triggers` must have ≥ 5 entries (mixed languages encouraged)
- `output_schema.format` must be `json`, `csv`, or `text`
- At least 1 example with a real, runnable `invocation`
- `error_codes` must define at least exit code `1`

---

## License

All manifests in this repository are licensed under **MIT**.
Free to use, modify, and distribute — just keep the copyright notice.

---

## Related

- **Reference router** (Python package): [clibrary-hub/CLIbrary](https://github.com/clibrary-hub/CLIbrary) — `pip install clibrary-hub`
- **Project site**: [clibrary.dev](https://clibrary.dev)
