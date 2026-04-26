# CLIbrary — Claude Code Context

> 這份文件給 Claude Code 看，讓它知道這個專案是什麼、現在在哪個階段、接下來要做什麼。
> 每次開新對話前先讀這份文件。

---

## 專案是什麼

CLIbrary 是兩層的平台：

**Layer 1 — CLIbrary（開源，MIT）**
一個標準化的 CLI 工具描述庫。每個 CLI 有一個 `manifest.json`，描述這個工具做什麼、什麼情況下用、接受什麼參數、回傳什麼結果。放在 GitHub 上，任何人都能用。

**Layer 2 — CLI Brain（閉源，AGPL + 商業授權）**
一個智慧路由系統。接收 AI agent 的自然語言意圖，找到最合適的 CLI，組好參數，回傳 tool call JSON。這是唯一的收費產品。

**一句話：** CLIbrary 是工具庫（開源），Brain 是智慧（收費）。

---

## 技術架構

### Phase 1 Brain（現在要做的）：RAG 版

```
Agent 意圖輸入
    ↓
sentence-transformer 向量化（multilingual-e5-base）
    ↓
DistilBERT 分類器 → 判斷類別（10 選 1）
    ↓
pgvector 在該類別內檢索 top-5 CLI
    ↓
Kimi API 小模型組參數（意圖 + manifest input_schema → params JSON）
    ↓
安全檢查
    ↓
寫 routing log（這是 Phase 2 LoRA 的訓練資料）
    ↓
回傳 tool call JSON 給 agent
```

### Phase 2 Brain（18 個月後）：LoRA 版

```
Agent 意圖輸入
    ↓
Mistral 7B + category LoRA adapter（跑在你的伺服器）
    ↓
直接輸出 tool call JSON（routing + 參數組裝一次完成）
```

Phase 1 和 Phase 2 對 agent 的介面完全一樣，升級是無縫的。

### 混合架構（本地 + 雲端）

```
用戶本地：manifest 快取（SQLite，~5MB）← 本地搜尋用，免費功能
雲端 Brain API：routing + 參數組裝 ← 付費功能
```

---

## 現在的進度

### 已完成
- [x] PoC 驗證：93.6% routing 準確率（1,380 筆測試，8 個類別）
- [x] 技術路線決定：Phase 1 用 RAG，Phase 2 用 LoRA
- [x] 授權決定：CLIbrary MIT，Brain AGPL + 商業授權雙軌
- [x] 收費模式：Free $0 / Developer $29 / Team $99 / Enterprise 議價
- [x] GitHub org 建立：github.com/clibrary-hub
- [x] 第一個 repo：clibrary-hub/manifests（5 個 manifest 已推上去）
- [x] README.md 完成並推上 GitHub
- [x] Kimi 批量生成完成：本地已有 69 個 manifest，尚未推上 GitHub

### 已知問題
- Kimi For Coding API 從 Python 直接呼叫會被 403 擋，需改用 Kimi 標準 API（platform.moonshot.cn）或直接在 Kimi 介面操作

### 進行中（Phase 0）
- [x] 把本地 69 個 manifest 推上 GitHub
- [x] 買域名 clibrary.dev
- [x] 建 landing page（Vercel）— repo: clibrary-hub/website
- [ ] 發 HackerNews Show HN
- [ ] 收集 500 個 email 等待清單

### 還沒開始（Phase 1，等 Phase 0 驗證後才做）
- [ ] PostgreSQL + pgvector Registry
- [ ] FastAPI Brain server（Fly.io）
- [ ] DistilBERT 分類器訓練
- [ ] `clibrary` pip 套件
- [ ] Stripe 計費
- [ ] Brain API 上線

---

## Manifest Schema（凍結，不能改）

```json
{
  "name": "string，kebab-case，3–50 字元",
  "version": "string，semver 格式",
  "category": "enum：ai-ml | devops | data | web | security | media | productivity | finance | science | networking",
  "description": "string，10–200 字元，一句話說清楚",
  "intent_triggers": [
    "至少 5 條，中英文都要有，說法要多樣化"
  ],
  "input_schema": {
    "參數名": {
      "type": "string | integer | boolean | enum",
      "required": true,
      "description": "說明"
    }
  },
  "output_schema": {
    "format": "json | csv | text",
    "fields": ["output 欄位名稱"]
  },
  "examples": [
    {
      "intent": "自然語言描述",
      "invocation": "cli-name --param value"
    }
  ],
  "error_codes": {
    "1": "最常見的錯誤"
  },
  "tags": ["相關標籤"]
}
```

**最重要的欄位是 `intent_triggers`**，這是 Brain routing 的訓練原料。每條要不同說法，不能只是同義詞重複。

---

## 目錄結構

```
clibrary-hub/manifests/          ← 這個 repo
├── README.md
├── CONTRIBUTING.md
├── CLAUDE.md                    ← 這份文件
├── ai-ml/
│   └── whisper-transcribe.json
├── data/
│   ├── sql-runner.json
│   └── csv-export.json
├── devops/
│   └── docker-cleanup.json
└── media/
    └── video-to-gif.json
（本地另有 69 個 manifest 待推上 GitHub）

clibrary-brain/                  ← 另一個 private repo（還沒建）
├── api/                         ← FastAPI server
├── classifier/                  ← DistilBERT
├── retriever/                   ← pgvector
├── assembler/                   ← Kimi API 參數組裝
└── safety/
```

---

## 給 Claude Code 的操作規則

### 寫新 manifest 時

1. 永遠先寫 `manifest.json`，不要寫程式碼
2. `intent_triggers` 至少 5 條，中英文各至少 2 條
3. 說法要多樣化：口語、正式、問句、陳述句都要有
4. `output_schema.format` 只能是 `json`、`csv`、`text` 其中之一
5. `examples` 至少 1 條，`invocation` 要是真的能跑的指令語法

### 寫 Python 程式碼時（Phase 1 才開始）

- Python 3.11+
- 必須有 type hints
- 不能 hardcode 任何 secret，用環境變數
- FastAPI 做 API server
- exit 0 = 成功，非零 = 失敗
- 結果只能寫 stdout，錯誤只能寫 stderr

### 關於 Brain 的程式碼

- Brain 的程式碼放在 private repo `clibrary-brain`，不放在 `manifests` repo
- 每次 Brain API 調用都必須寫 routing log，格式：
  ```json
  {
    "intent": "用戶意圖",
    "category": "選到的類別",
    "cli_name": "選到的 CLI",
    "params": {},
    "success": true,
    "timestamp": "ISO 8601"
  }
  ```
- 這份 log 是 Phase 2 LoRA 訓練的原料，不能省略

---

## 重要決策記錄

| 決策 | 選擇 | 原因 |
|------|------|------|
| Brain 授權 | AGPL + 商業授權 | 商業使用必須付費，Elastic 模式 |
| CLIbrary 授權 | MIT | 最大化社群採用 |
| Phase 1 技術 | RAG（sentence-transformer + Kimi） | 快速上線，3 個月可收費 |
| Phase 2 技術 | LoRA adapter on Mistral 7B | 跑在你的伺服器，用戶不需要本地模型 |
| 套件架構 | 混合（本地 manifest 快取 + 雲端 Brain API） | 本地搜尋免費，routing 付費 |
| 小模型方案 | Kimi API（Phase 1）→ 自己的 Mistral（Phase 2） | 現在不用維運，之後有錢有資料再自己跑 |

---

## 商業里程碑

| 階段 | 時間 | 目標 |
|------|------|------|
| Phase 0 | 現在 | GitHub star 200+，email 500+ |
| Phase 1 | 3–5 個月 | MRR $5K，80 個付費用戶 |
| Phase 2 | 12 個月 | MRR $50K，LoRA Brain Pro 上線 |
| Phase 3 | 24 個月 | MRR $300K，Series A 或併購對話 |
| 退出 | 36 個月 | Anthropic / Google / Microsoft 併購 |

---

## 已驗證的數據

- PoC 準確率：**93.6%**（計畫門檻 88%）
- 測試樣本：1,380 筆
- 最弱類別：finance 90%、web 90%
- 最強類別：security 95%、ai-ml 94.5%
- 結論：技術方向成立，可以進入 Phase 0

---

*最後更新：2026-04-26*
*目前狀態：Phase 0 進行中*
*下一個行動：發 HackerNews Show HN，開始收集 email 等待清單*
