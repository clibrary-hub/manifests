# CLIbrary — Manifests

> A standardized manifest registry for agent-callable CLI tools.

---

## What is this?

CLIbrary is an open standard for describing CLI tools in a way that AI agents can understand and invoke them automatically — without wasting tokens on documentation.

Each manifest is a single JSON file that answers four questions:

| Question | Field |
|----------|-------|
| What does it do? | `description` |
| When should I use it? | `intent_triggers` |
| What does it need? | `input_schema` |
| What does it return? | `output_schema` |

---

## Why?

Today, AI agents waste thousands of tokens figuring out which tool to use and how to call it. CLIbrary solves this by separating **discovery** (manifest) from **execution** (the tool itself). The agent reads the manifest once; after that, routing is instant.

**Validated accuracy: 93.6% routing precision on 1,380 test cases, zero fine-tuning.**

---

## How it works

```
Agent: "Convert this MP4 to a GIF"
           ↓
   Embed intent as vector
           ↓
   Nearest-neighbor search across intent_triggers
           ↓
   Match: video-to-gif  (score: 0.97)
           ↓
   Assemble tool call with correct params
```

No LLM call needed for routing. Each manifest's `intent_triggers` act as the routing index.

---

## Structure

```
manifests/
├── ai-ml/          # Model inference, embeddings, transcription
├── data/           # SQL, CSV, databases, ETL
├── devops/         # Docker, git, CI/CD
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
| **Total** | **69** | Growing — PRs welcome |

---

## Contributing

We welcome contributions for any CLI tool that an AI agent might reasonably want to use.

**Steps:**

```bash
# 1. Fork this repo

# 2. Create your manifest
cp manifests/data/sql-runner.json manifests/<category>/<your-tool>.json

# 3. Validate
python packages/manifest-validator/validator.py manifests/<category>/<your-tool>.json

# 4. Open a PR
```

**Rules:**

- `intent_triggers` must have ≥ 5 entries
- `output_schema.format` must be `json`, `csv`, or `text`
- At least 1 example with a real, runnable `invocation`
- `error_codes` must define at least exit code `1`
- CI will auto-validate your manifest on every PR

---

## Roadmap

- [x] Manifest schema v1.0 (frozen)
- [x] Initial 5 manifests across 4 categories
- [x] PoC: 93.6% routing accuracy with zero training
- [ ] Expand to 50 manifests across all 10 categories
- [ ] `pip install clibrary` — local search CLI
- [ ] Manifest validator CI on every PR
- [ ] CLIbrary Brain API — intelligent routing as a service

---

## License

All manifests in this repository are licensed under **MIT**.  
Free to use, modify, and distribute — just keep the copyright notice.

The CLIbrary Brain (routing engine) is proprietary software — see [clibrary.dev](https://clibrary.dev) for API access.

---

*Built for the agentic era.*
