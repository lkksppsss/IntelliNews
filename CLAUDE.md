# 🧠 Intelligent Information Tracking & Visualization Platform

## ⚠️ AI 協作規則（優先閱讀）

- 嚴格遵守模組邊界，不混用跨層職責
- 優先擴展現有抽象（如 BaseCrawler / BaseSpider），不重造輪子
- Side project 導向：各模組以基本功能可運作為目標，不過度設計
- 避免服務間緊耦合
- Crawler 只做：fetch / parse / store，其他一律不碰
- **CQRS 原則：Command（寫入）一律透過 gRPC Server；Query（讀取）直接用 raw SQL，不透過 gRPC**
- gRPC Server 的存在是為了：跨語言統一寫入介面、集中管理資料異動、未來支援多個異質 DB
- **開發順序：先定義 proto → 其他服務實作 client 端 → gRPC Server 最後補實作**

---

## 🎯 當前工作焦點

**Phase 2 已完成 ✅（2026-03-30）**

- ✅ FastAPI 建立完成（GET /news、GET /news/{id}、POST /crawl/{spider} via RabbitMQ）
- ✅ FastAPI → RabbitMQ → Scrapy worker → gRPC Server → DB 整條鏈路打通
- ✅ C# Gateway 核心完成（YARP Proxy、JWT 驗證、Request Logging、CORS、401 保護）
- ✅ Django 改為 Poetry 環境、PostgreSQL、簽發 JWT（simplejwt）
- ✅ Django 新聞資料改透過 Gateway 取得（JWT service token）
- ✅ Django → Gateway(:5138) → FastAPI → PostgreSQL 完整鏈路打通（gateway_client.py）

整體目標：架構完整、各服務串聯、可部署展示，作為求職主力作品

**Phase 2.5（當前）— 補齊核心鏈路與前台功能**
- ✅ Scrapy：PublishArticleCreatedPipeline（detail 存好後發布到 article_created_queue）
- ✅ Laravel：ConsumeArticleCreated command（RabbitMQ consumer）
- ✅ Laravel：DiscordChannel 實作
- ✅ Laravel：ChannelDispatcher 加入 discord case
- ✅ Laravel：ArticleCreatedHandler channels 改為從 NotificationRule DB 讀
- ✅ Laravel：.env 加 RABBITMQ_* 和 DISCORD_WEBHOOK_URL（Webhook URL 待填）
- ✅ 整條鏈路驗證：FastAPI → RabbitMQ → Scrapy → gRPC → Scrapy → RabbitMQ → Laravel → DB
- ⏳ Django：搜尋 / 過濾功能
- ⏳ Django：UserSourcePreference（用戶追蹤特定來源，日後推播用）

Phase 2.5 完成後 → Dockerfile + 部署到 Railway（含 PostgreSQL + RabbitMQ）

**Phase 3a — 自動化 + 可觀測性**
- ⏳ n8n 排程觸發爬蟲（加分在視覺化 workflow，可用 cron 替代）
- ⏳ Log 蒐集（Go log collector → ELK）

**Phase 3b — Discord Bot + AI**
- ⏳ Discord Bot：指令觸發爬蟲 + 查新聞（不含 AI）
- ⏳ Discord Bot：查詢 ELK log（admin only，自然語言 → AI Agent / MCP Server → Elasticsearch query）
- ⏳ AI RAG：LangChain + Ollama（admin only，地端，處理機敏資料）

**Phase 4 — 收尾**
- ⏳ 用戶偏好推播（追蹤來源 → 每日 Discord 通知）
- ⏳ Scrapy 完整錯誤處理 + Incremental Crawling
- ⏳ Dockerfile + CI/CD 全服務補齊
- ⏳ MAUI（個人技能練習，不影響主線）

> 每次開工前更新此區塊

---

## 🛠️ Tech Stack

| Layer            | Tech                                          | 狀態         |
|------------------|-----------------------------------------------|--------------|
| Crawler (主)     | Scrapy, gRPC client, RabbitMQ                 | 70%          |
| Crawler (參考)   | Python, SQLAlchemy, BeautifulSoup, Playwright  | 75%          |
| gRPC DB Server   | Python, gRPC, SQLAlchemy, PostgreSQL          | 隨需同步開發  |
| Proto 定義        | Protocol Buffers (proto3)                     | 100%         |
| C# Gateway       | ASP.NET Core Web API（唯一對外端口）           | 進行中       |
| API              | FastAPI（Internal only）                      | 60%          |
| Frontend         | Django 5.2, Chart.js                          | 85%          |
| Notification     | Laravel 12, Queue, Email/Webhook              | 40%          |
| Message Queue    | RabbitMQ                                      | 整合中       |
| AI - RAG         | LangChain, Qdrant, Ollama（本地 LLM）         | 0%（計劃）   |
| AI - Agent       | LangChain Agent, Tool Calling                 | 0%（計劃）   |
| AI - Bot         | Discord Bot（discord.py）                     | 0%（計劃）   |
| MCP Server       | Python MCP SDK                                | 0%（計劃）   |
| Log 蒐集         | Go（log collector）→ ELK（Elasticsearch, Logstash, Kibana）| 0%（計劃）|
| Automation       | n8n                                           | 0%           |
| Mobile           | MAUI                                          | 0%           |
| DB               | PostgreSQL                                    | 使用中       |

---

## 🏗️ 系統架構

### 對外流量（External）

```
Browser / 外部 Client
        │
        ▼
[C# ASP.NET Core Gateway]  ← 唯一對外端口，負責 Auth / JWT / Logging
        │
        ├─ HTTP → [FastAPI]  ← 業務邏輯 API（不直接對外）
        │             │ gRPC
        │             ▼
        └─ gRPC → [crawler_grpc_server] ←→ [Shared Data DB]
```

### Internal 流量（繞過 Gateway）

```
[news_crawler_scrapy]
  │ gRPC（直連，不走 Gateway）
  ▼
[crawler_grpc_server] ←→ [Shared Data DB (PostgreSQL)]

[news_crawler_scrapy]（SaveNewsDetailPipeline 成功後）
  │ RabbitMQ publish → article_created_queue
  ▼
[RabbitMQ]
  │ consume
  ▼
[laravel_notify]  ← 通知職責，消費 MQ 事件，不走 Gateway
```

### Django 的位置

```
[intellinews-app (Django)]
  ├─ 自有 DB：User / Session / 用戶偏好 / 書籤（Django ORM）
  └─ 新聞資料：→ C# Gateway → FastAPI → gRPC Server（視為 external client）
```

### 爬蟲觸發流程（Phase 2 規劃）

```
n8n → POST /crawl/{spider} → FastAPI（internal）→ subprocess → scrapy crawl <spider>
```

- Scrapy 是 **run-and-exit** 模型，不作為常駐 container
- FastAPI 與 Scrapy 部署在同一台機器，FastAPI 用 subprocess 直接呼叫
- CD 目標：確保機器上有最新 image，不啟動常駐 container

### 職責邊界

| 服務 | 職責 | 對外？ |
|------|------|--------|
| C# Gateway | 唯一對外端口、Auth、Logging | ✅ |
| FastAPI | 業務邏輯 API、爬蟲控制 | ❌ Internal only |
| crawler_grpc_server | DB 存取微服務 | ❌ Internal only |
| Django | Frontend 渲染、用戶資料 | ✅（透過 Gateway 取資料）|
| Laravel Notify | 通知派送、MQ 消費 | ❌ Internal only |
| Scrapy | 爬蟲執行 | ❌ Internal only |

---

## 📁 Repo 結構

```
/
├── Python/
│   ├── news_crawler_scrapy/    ← 主力爬蟲（Scrapy + gRPC）60%
│   ├── news_crawler_mvp/       ← 學習用架構 / 設計參考     75%
│   ├── crawler_grpc_server/    ← DB 存取微服務（gRPC）      70%
│   ├── intellinews-app/        ← Django Frontend            85%
│   │   ├── intellinews_fastapi/    ← 業務邏輯 API（FastAPI）     進行中
└── intellinews-proto/      ← 共用 Proto 定義             100%
└── PHP/
    └── laravel_notify/         ← 通知服務                    60%
```

---

## 📦 各模組說明

### 🕷️ Crawler 主力：news_crawler_scrapy — 60%

```
news_crawler_scrapy/
├── news_crawler_scrapy/
│   ├── spiders/
│   │   ├── base_detail_spider.py
│   │   ├── pts/         # pts_list_spider, pts_detail_spider
│   │   ├── cna/         # cna_rss_spider, cna_detail_spider
│   │   └── udn/         # udn_list_spider, udn_detail_spider
│   ├── items.py          # NewsItem
│   ├── pipelines.py      # SaveList / SaveDetail / PublishToRabbitMQ
│   ├── settings.py
│   ├── grpc_client/
│   │   └── grpc_client.py
│   └── providers/
│       └── rabbit_mq_provider.py
└── generated/            # Protobuf 產生的 Python 檔
```

支援來源：PTS ✅ / CNA（RSS）✅ / UDN ✅
Pipeline：gRPC 存 List → RabbitMQ 發布 → gRPC 存 Detail
本機已串接 crawler_grpc_server 可正常運行

待完成：
- ⏳ 完整錯誤處理
- ⏳ Dynamic spider（Playwright middleware）
- ⏳ Incremental crawling（status check before re-fetch）

---

### 📚 Crawler 參考：news_crawler_mvp — 75%

自行設計的爬蟲框架，作為學習成果與架構參考。可從中取用設計模式與 utils。

```
news_crawler_mvp/
├── core/
│   ├── base_crawler.py           # 抽象基底
│   ├── base_static_crawler.py
│   ├── base_dynamic_crawler.py   # Selenium / Playwright
│   ├── base_visual_crawler.py    # 截圖
│   ├── base_crawl_flow.py        # 流程編排
│   ├── db_handler.py
│   └── utils/                    # browser / log / argparse
├── models/                       # SQLAlchemy ORM
├── repositories/                 # DB 存取層
├── sites/                        # PTS / ETtoday / LTN
│   └── {site}/
│       ├── *_list_crawler.py
│       ├── *_detail_crawler.py
│       ├── *_visual_crawler.py
│       └── *_crawl_flow.py
└── main.py
```

支援：Static ✅ / JSON API ✅ / Visual ✅ / Dynamic ⏳ / Incremental ⏳

---

### ⚙️ gRPC DB Server：crawler_grpc_server — 隨需同步開發

**目的：** 跨語言（Python / C# / PHP / ...）統一 DB 存取介面。未來支援多個異質 DB（爬蟲資料 DB、AI 訓練資料 DB、AI 結果 DB 等），各服務只需呼叫對應的 gRPC 方法，不需關心底層 DB 類型。

爬蟲與上層服務的資料存取微服務，統一管理 News / Category / Source。

```
crawler_grpc_server/
├── models/       # News, Source, Category, NewsCategory, VisualSnapshot
├── repositories/ # CRUD 存取層
├── services/     # aggregated_service → news/category/source_service
├── generated/    # Protobuf 產生檔
├── utils/
│   └── proto_mapper.py   # ORM ↔ Protobuf 轉換
├── db.py         # SessionLocal factory
└── server.py     # gRPC server on :50051
```

RPC 方法：CreateNews / FetchNews / FindNews / ListNews / DeleteNews
         CreateCategory / ListCategory
         CreateSource / FindSource / ListSource

開發策略：隨其他專案需求同步擴充，不預先過度設計

---

### 📐 Proto 定義：intellinews-proto — 100%

所有服務共用的 gRPC contract，單一來源（single source of truth）。

```
intellinews-proto/
└── news_crawler_db_protos/
    ├── crawler_db.proto     # CrawlerDBService 定義
    ├── news.proto
    ├── category.proto
    ├── source.proto
    └── visual_snapshot.proto
```

---

### 🌐 C# Gateway — 進行中（Phase 2）

- **ASP.NET Core Web API**，系統唯一對外端口
- 職責：JWT 驗證、Request logging、CORS、Reverse proxy 到 FastAPI
- 不含業務邏輯，業務邏輯在 FastAPI

**Auth 架構決策（已定案）：**
- 方案：**JWT（選項 A）**— Django 負責登入驗帳密並簽發 JWT，Gateway 用 shared secret 驗簽章，不需回查 Django DB
- Access token 短效（15min）+ Refresh token rotation
- Refresh token 存 HttpOnly cookie
- Django 是 user data 的 authority，Gateway 是 request 驗證的 authority

待做：
- ⏳ 專案建立（ASP.NET Core Web API）
- ⏳ JWT 驗證 middleware
- ⏳ Reverse proxy 到 FastAPI
- ⏳ Request logging middleware
- ⏳ CORS 設定
- ⏳ Django 改為簽發 JWT（配合 Gateway）

### ⚡ FastAPI — 60%（Phase 2 進行中）

- Internal only，不直接對外暴露
- 職責：業務邏輯 API、爬蟲控制（POST /crawl/{spider}）
- **CQRS Read side：直接用 raw SQL 查詢 PostgreSQL，不透過 gRPC**
- **CQRS Write side：透過 gRPC client 呼叫 crawler_grpc_server**
- 資料夾位置：`Python/intellinews_fastapi/`
- Poetry 環境，src layout（`src/intellinews_fastapi/`）
- 爬蟲觸發：FastAPI → RabbitMQ → Scrapy worker（worker.py 常駐監聽）

完成：
- ✅ 專案建立（Poetry + src layout）
- ✅ PostgreSQL connection pool（psycopg3）
- ✅ `GET /news`（分頁列表，raw SQL）
- ✅ `GET /news/{id}`（單篇詳情，404 處理）
- ✅ `POST /crawl/{spider}`（發 message 到 RabbitMQ）

待做：
- ⏳ 與 C# Gateway 串接

---

### 🖥️ Frontend：intellinews-app（Django）— 85%

```
intellinews-app/intellinews/
├── accounts/     # UserProfile, UserCategoryPreference, UserSavedNews
├── news/         # Source, Category, News models + list/detail views
└── templates/    # Django Templates + Chart.js
```

完成：用戶認證 / 新聞列表與詳情 / 分類訂閱（interest level）/ 書籤功能 / Admin panel

待做：
- ⏳ 通知中心 UI
- ⏳ 搜尋 / 過濾功能
- ⏳ 用戶設定頁面
- ⏳ 與 API Layer 串接（目前直連 Django DB）

---

### 🔔 Notification：laravel_notify（Laravel）— 60%

```
laravel_notify/app/
├── Models/      # Notification, NotificationRule
├── Channels/    # EmailChannel, WebhookChannel, LogChannel
├── Services/    # NotificationService, NotifyDispatcher, ChannelDispatcher
├── Handlers/    # ArticleCreatedHandler, CrawlerFailedHandler
└── Jobs/        # SendNotification（Queue job）
```

完成：Email channel / 事件枚舉 / Queue job / DB models / Request 驗證

待做：
- ⏳ Webhook channel 完整實作
- ⏳ Discord Webhook channel（取代 LINE，對接 Discord）
- ⏳ Email template 系統
- ⏳ 通知頻率控制 / 退訂
- ⏳ 與 RabbitMQ / n8n 串接

---

### 🤖 AI Module — 0%（Phase 3）

#### 功能一：RAG 新聞問答
- 新聞資料向量化 → 存入 Qdrant（支援 gRPC API）
- 使用者提問 → 搜尋相關文章 → Ollama（本地 LLM）生成回答
- 不需要訓練模型，透過 RAG 讓 LLM 基於真實新聞資料回答

#### 功能二：LLM Agent 維運查詢
自然語言查詢系統狀態，Agent 自動判斷要呼叫哪個 Tool：

```
你問："user alvin 今天登入失敗，怎麼了？"
  ↓
Agent 判斷需要的 Tools
  ├─ Tool: gRPC → crawler_grpc_server（查用戶資料）
  └─ Tool: Elasticsearch query（查 auth log）
  ↓
Ollama 整理成自然語言回答
```

#### 功能三：Discord Bot 介面
- discord.py 建立 Bot
- 統一入口：新聞問答 + 維運查詢都透過 Discord 指令觸發
- 部署後可當輕量維運工具使用

#### Tech Stack
- **LLM**：Ollama（本地，免費）— 推薦模型：Llama 3 / Mistral
- **RAG 框架**：LangChain
- **Vector DB**：Qdrant（有 gRPC API，符合架構設計）
- **Bot**：discord.py
- **Log 查詢**：Elasticsearch（由 Log 蒐集專案提供）

#### 架構

```
Discord Bot
  │
  ▼
AI Module (Python)
  ├─ RAG pipeline → Qdrant ← 新聞向量索引
  ├─ Agent tools:
  │    ├─ gRPC → crawler_grpc_server
  │    └─ Elasticsearch → Log DB
  └─ Ollama (本地 LLM)
```

---

### 📋 Log 蒐集專案 — 0%（Phase 3 前置）

獨立專案，蒐集所有服務的 log，統一整理後存入 ELK。

```
各服務 log（Scrapy / FastAPI / C# Gateway / Django / Laravel / ...）
  │ structured log（JSON）
  ▼
Log Collector（Go，負責接收 + 正規化）
  │
  ▼
Logstash（轉換 + 過濾）
  │
  ▼
Elasticsearch（儲存 + 索引）← AI Agent 從這裡查維運資料
  │
  ▼
Kibana（視覺化 dashboard）
```

AI Agent 的維運查詢功能依賴此專案提供結構化 log 資料。

### 🔌 MCP Server — 0%（Phase 3c）

獨立專案 `Python/intellinews_mcp/`，讓 Claude Desktop / Claude Code 直接操作 IntelliNews。

**分階段實作：**
- Phase 2.5 後：`query_docs`（讀 README / CLAUDE.md）、`search_news`（直連 FastAPI）
- Phase 3a 後：`get_system_status`、`query_logs`（依賴 ELK）
- Phase 3b 後：`trigger_crawl`（依賴 Gateway API Key middleware）

**權限設計：**
- 同一個 MCP，API Key 控制層級
- readonly key：query_docs、search_news、get_news_detail
- admin key：trigger_crawl、get_system_status、query_logs

**Gateway 端：**
- 新增 `X-Api-Key` header 驗證 middleware，平行於現有 JWT middleware
- API Key 存 appsettings.json / 環境變數

---

### 🔄 Automation（n8n）— 0%

負責：排程爬蟲、觸發 AI 流程、派送通知
串接方式：n8n HTTP Request → FastAPI `/crawl/{spider}`（不直接呼叫 Scrapy）

### 📱 MAUI App — 0%

負責：監控 dashboard、控制爬蟲與系統狀態

---

## 🗄️ 資料庫策略

### CQRS 分離策略

| 操作 | 路徑 | 說明 |
|------|------|------|
| Command（寫入）| gRPC Server | 跨語言一致、集中管理異動 |
| Query（讀取）| Raw SQL 直連 PostgreSQL | 彈性、輕量，複雜查詢不受 proto 限制 |

- **Django Frontend：** 獨立 DB，直接使用 Django ORM（用戶資料、書籤、偏好）
- **新聞資料讀取（FastAPI / Django 未來）：** raw SQL 直連 Shared DB（psycopg2）
- **新聞資料寫入（Scrapy / FastAPI）：** 一律透過 gRPC Server
- **現在唯一 DB：** PostgreSQL（gRPC Server 已使用）
- 未來可能有多個異質 DB（爬蟲資料、AI 訓練資料、AI 結果），gRPC Server 管理 Command side

APScheduler 只用於：log 清理、快取管理、統計生成
**❌ 不用 APScheduler 排程爬蟲**

---

## 🚀 Roadmap

| Phase | 目標 | 狀態 |
|-------|------|------|
| 1 | Scrapy Pipeline 串接 + gRPC Server 基礎 RPC | ✅ 完成 |
| 2 | FastAPI（internal）+ C# Gateway（對外）+ Django 改接 Gateway | ✅ 完成 |
| 2.5（當前）| Laravel RabbitMQ+Discord串通、Django搜尋、UserSourcePreference | 🔄 進行中 |
| 2.5 後    | Dockerfile + 部署到 Railway | ⏳ |
| 3a        | n8n 排程 + Go Log Collector → ELK | ⏳ 計劃中 |
| 3b        | Discord Bot（指令+查新聞）+ AI Agent log查詢（admin only）+ AI RAG | ⏳ 計劃中 |
| 3c        | MCP Server（query_docs + search_news MVP → trigger_crawl + query_logs） | ⏳ 計劃中 |
| 4         | 用戶偏好推播、Scrapy 錯誤處理、CI/CD 全服務 | ⏳ 計劃中 |
| 5         | MAUI（個人技能練習） | ⏳ 計劃中 |
