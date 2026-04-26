# CLIbrary — Manifests

> A standardized manifest registry for agent-callable CLI tools.  
> 給 AI agent 使用的 CLI 工具標準化描述庫。

---

## What is this? / 這是什麼？

CLIbrary is an open standard for describing CLI tools in a way that AI agents can understand and invoke them automatically — without wasting tokens on documentation.

CLIbrary 是一套開放標準，讓 AI agent 能自動理解並調用 CLI 工具，不需要把工具說明書塞進 prompt。

Each manifest is a single JSON file that answers four questions:

每個 manifest 是一個 JSON 檔案，回答四個問題：

| Question | Field |
|----------|-------|
| What does it do? 它做什麼？ | `description` |
| When should I use it? 什麼時候用？ | `intent_triggers` |
| What does it need? 需要什麼參數？ | `input_schema` |
| What does it return? 回傳什麼？ | `output_schema` |

---

## Why? / 為什麼要做這個？

Today, AI agents waste thousands of tokens figuring out which tool to use and how to call it. CLIbrary solves this by separating **discovery** (manifest) from **execution** (the tool itself). The agent reads the manifest once; after that, routing is instant.

現在 AI agent 每次調用工具都要浪費大量 token 去理解工具怎麼用。CLIbrary 把「找工具」（manifest）和「跑工具」（執行）分開——agent 讀一次 manifest，之後路由不需要額外 token。

**Validated accuracy: 93.6% routing precision on 1,380 test cases, zero fine-tuning.**  
**驗證準確率：1,380 筆測試達 93.6% 路由精準度，無需任何模型訓練。**

---

## How it works / 運作原理

```
Agent: "幫我把這個 MP4 轉成 GIF"
           ↓
   Embed intent as vector
   意圖轉成向量
           ↓
   Nearest-neighbor search across intent_triggers
   在所有 intent_triggers 做向量最近鄰搜尋
           ↓
   Match: video-to-gif  (score: 0.97)
           ↓
   Assemble tool call with correct params
   組出正確的 tool call 參數
```

No LLM call needed for routing. Each manifest's `intent_triggers` act as the routing index.

路由本身不需要 LLM。每個 manifest 的 `intent_triggers` 就是路由索引。

---

## Structure / 目錄結構

```
manifests/
├── ai-ml/          # Model inference, embeddings, transcription / 模型推論、語音辨識
├── data/           # SQL, CSV, databases, ETL / 資料庫查詢、資料匯出
├── devops/         # Docker, git, CI/CD / 容器管理、版本控制
├── media/          # Video, image, audio / 影片、圖片、音訊處理
├── web/            # HTTP, scraping, APIs / 網路請求、爬蟲
├── security/       # Scanning, auditing, encryption / 掃描、加密、審計
├── productivity/   # Files, text, conversion / 檔案處理、文件轉換
├── finance/        # Market data, invoices / 金融資料、發票
├── science/        # Units, stats, computation / 單位換算、統計
└── networking/     # DNS, IP, diagnostics / 網路診斷
```

---

## Manifest Format / Manifest 格式

```json
{
  "name": "sql-runner",
  "version": "1.0.0",
  "category": "data",
  "description": "Execute SQL queries against a database and return structured results",
  "intent_triggers": [
    "query a database",
    "run a SQL statement",
    "查資料庫",
    "跑 SQL",
    "幫我查數據",
    "get records from a table",
    "執行資料庫查詢"
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
    },
    {
      "intent": "查上個月每個用戶的訂單數",
      "invocation": "sql-runner --query \"SELECT user_id, COUNT(*) FROM orders WHERE created_at > DATE_SUB(NOW(), INTERVAL 1 MONTH) GROUP BY user_id\" --output-format csv"
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

### Required fields / 必填欄位

| Field | Type | Rule |
|-------|------|------|
| `name` | string | lowercase, hyphen-separated / 小寫加連字號 |
| `version` | string | semver `x.y.z` |
| `category` | enum | one of 10 categories / 十個類別之一 |
| `description` | string | 10–200 characters / 10–200 字元 |
| `intent_triggers` | array | ≥ 5 items, mix of EN + ZH / 至少 5 條，中英文都要有 |
| `input_schema` | object | each param needs `type` |
| `output_schema` | object | needs `format` field |
| `examples` | array | ≥ 1 item with working `invocation` |

---

## Current Manifests / 目前收錄

| Category | Count | Tools |
|----------|-------|-------|
| `ai-ml` | 1 | whisper-transcribe |
| `data` | 2 | sql-runner, csv-export |
| `devops` | 1 | docker-cleanup |
| `media` | 1 | video-to-gif |
| **Total** | **5** | Growing — PRs welcome |

---

## Contributing / 如何貢獻

We welcome contributions for any CLI tool that an AI agent might reasonably want to use.

歡迎為任何 AI agent 可能用到的 CLI 工具貢獻 manifest。

**Steps / 步驟：**

```bash
# 1. Fork this repo / Fork 這個 repo

# 2. Create your manifest / 建立你的 manifest
cp manifests/data/sql-runner.json manifests/<category>/<your-tool>.json

# 3. Validate / 驗證格式
python packages/manifest-validator/validator.py manifests/<category>/<your-tool>.json

# 4. Open a PR / 開 PR
```

**Rules / 規則：**

- `intent_triggers` must have ≥ 5 entries, including both English and Chinese / 至少 5 條，中英文都要有
- `output_schema.format` must be `json`, `csv`, or `text`
- At least 1 example with a real, runnable `invocation` / 至少 1 個真的能跑的 invocation
- `error_codes` must define at least exit code `1`
- CI will auto-validate your manifest on PR / CI 會在 PR 時自動驗證

---

## Roadmap / 路線圖

- [x] Manifest schema v1.0 (frozen / 已凍結)
- [x] Initial 5 manifests across 4 categories
- [x] PoC: 93.6% routing accuracy with zero training
- [ ] Expand to 50 manifests across all 10 categories
- [ ] `pip install clibrary` — local search CLI
- [ ] Manifest validator CI on every PR
- [ ] CLIbrary Brain API — intelligent routing as a service
- [ ] LoRA-based adapter for sub-200ms routing (Plan A)

---

## License / 授權

All manifests in this repository are licensed under **MIT**.  
Free to use, modify, and distribute — just keep the copyright notice.

本 repo 的所有 manifest 採用 **MIT** 授權。  
可自由使用、修改、發布，保留版權聲明即可。

The CLIbrary Brain (routing engine) is proprietary software — see [clibrary.dev](https://clibrary.dev) for API access.  
CLIbrary Brain（路由引擎）為閉源商業軟體，API 存取請見 [clibrary.dev](https://clibrary.dev)。

---

*Built for the agentic era. / 為 AI agent 時代而生。*
