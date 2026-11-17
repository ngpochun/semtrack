# SemTrack 後端 Component 設計

> 狀態：草案 v1 · 2026-11-17 · 原則：**MVP 輕量單體先落地，Phase 2 才加抓取與 AI 管線**。
> 技術棧建議（待確認）：MVP 用 Next.js（API routes）＋ SQLite/Postgres（Prisma）；Phase 2 解析 worker 用 Python（FastAPI + 任務隊列）。以下以 component 邊界描述，不綁死框架。

## 0. 總覽圖

```
┌───────────── MVP ─────────────┐      ┌──────────────── Phase 2 ────────────────┐
│  Web Frontend (Notion 風 SPA)  │      │  Browser Extension (MV3)                │
│   │ REST/JSON                  │      │   content script: Moodle adapter        │
│  API Server (單體)             │      │   background: 收集/去重/上傳             │
│   ├─ Auth（簡易 session）      │◄─────┤   │ POST /ingest (hash, metadata, file) │
│   ├─ Courses / Assignments    │      │  Ingest API ─► Object Storage (S3/本地)  │
│   ├─ Projects / Tasks         │      │   │ enqueue                              │
│   ├─ Calendar (推導視圖)      │      │  Parse Worker (PDF→text/layout)          │
│   └─ DB (Postgres/SQLite)     │      │   │ enqueue                              │
└───────────────────────────────┘      │  Extract Worker (LLM + 信心分數)         │
                                       │   │                                      │
                                       │  Review Queue API ─► 人工確認 ─► 寫入 DB │
                                       └──────────────────────────────────────────┘
```

## 1. MVP Components（本輪落地範圍）

| Component | 職責 | 關鍵設計 |
|---|---|---|
| **API Server** | 單一服務提供 REST/JSON；無微服務 | 一個 deployable；route 按資源分模組 |
| **Auth** | 註冊/登入/session；MVP 可先 email+密碼或學校 SSO 預留 | 只存 hash；session cookie；不做 OAuth 矩陣 |
| **Course Service** | 課程 CRUD、學期起訖設定 | 學期起始日供 Phase 2 相對日期解析复用 |
| **Assignment Service** | 作業/小測/考試 CRUD、done 切換 | 狀態（逾期/剩 N 天）**由 date 推導，不存狀態欄**，避免不一致 |
| **Project Service** | 專題、成員、任務、里程碑 CRUD | 進度 = done tasks / total tasks，**查詢時推導** |
| **Calendar View** | 月視圖事件聚合 | 不另存事件表；由 assignments + milestones + project due **union 推導**（與原型同一邏輯） |
| **DB** | Postgres（或 MVP SQLite） | schema 見 §3 |

MVP 刻意**沒有**：隊列、worker、object storage、LLM——減少运维面，先驗證核心價值。

## 2. Phase 2 Components（Extension ＋ AI 管線）

### 2.1 Browser Extension（客戶端，非後端但為管線入口）
- **Moodle Adapter（per-site）**：content script 解析課程頁資源列表（DOM selector 按校配置）；只讀使用者已登入頁面；
- **Collector/Background**：收集 file URL + metadata（課程、週次、檔名、last-modified）；
- **Dedup Client**：先算 hash 或送 URL 指紋，server 回「已存在」則跳過上傳 → 解決重複下載；
- **Uploader**：使用者點「同步」才批次上傳（明確授權），附 session token（非 Moodle 帳密）。

### 2.2 Ingest API
- `POST /ingest`：接收 metadata + file（或先 presigned URL）；
- **File Registry**：以 SHA-256 為主鍵去重；同 hash 多次出現只存一份、記錄多個來源（課程/頁面/時間）→ 集中課件索引的基礎；
- 寫入 Object Storage；enqueue parse job。

### 2.3 Parse Worker
- PDF → text + 版面（python: pdfplumber/PyMuPDF；掃描件走 OCR 預留）；
- 產出 page-level text 供抽取與「原文引用」回溯；
- 失敗（加密/損壞）標記 job 狀態，不靜默丟棄。

### 2.4 Extract Worker（LLM）
- 輸入 page text + 學期起始日 + 課程代碼；
- 輸出結構化 `{type, title, date, time, confidence, source_page, quote}`；
- 相對日期解析器：「下週五」→ 以學期週曆換算；模糊（「十二月初」）→ 低 confidence + 推測標記；
- 每筆抽取一律進 Review Queue，**不自動寫入日曆**。

### 2.5 Review Queue API ＋ 確認流程
- `GET /review`、`PATCH /review/:id`（改標題/日期/類型）、`POST /review/:id/confirm`、`/dismiss`；
- confirm 才 transaction 寫入 assignments/milestones；保留 source file + page + quote 供稽核；
- 與原型「待確認抽取」UI 同一契約。

### 2.6 Courseware Index（課件管理）
- `GET /courses/:id/files`：課程 → 週次 → 檔案（hash 去重後）→ 來源版本列表；
- 解決「一個軟件下載很多次」：同一 hash 只有一筆，UI 顯示「已收錄、來自 N 個頁面」。

### 2.7 提醒（可再往後）
- 到期前 N 天 email/web push；由 assignment date 推導排程，MVP 不做。

## 3. 資料模型（MVP 核心表）

```
users(id, email, pw_hash, created_at)
semesters(id, user_id, name, start_date, end_date)
courses(id, semester_id, code, name, color)
assignments(id, course_id, type[assignment|quiz|exam], title, due_date, due_time, done, source[manual|extract], created_at)
projects(id, course_id, title, desc, due_date)
project_members(project_id, user_id|null, display_name, color)   -- MVP 成員可僅標記
tasks(id, project_id, title, owner_name, due_date, done)
milestones(id, project_id, title, date)
-- Phase 2 追加：
files(hash PK, size, mime, storage_key, first_seen_at)
file_sources(file_hash, course_id, moodle_url, page_title, seen_at)  -- 同檔多來源
ingest_jobs(id, file_hash, status[pending|parsed|extracted|failed], error)
extractions(id, job_id, course_id, type, title, date, confidence, source_page, quote, status[pending|confirmed|dismissed])
```

重點：**done 是唯一事實來源**；逾期/剩餘天數/進度%/日曆事件全部推導，與前端原型同一規則，杜绝兩套邏輯。

## 4. API 草案（MVP）

```
POST /auth/register | /auth/login | /auth/logout
GET/POST /courses            PATCH/DELETE /courses/:id
GET/POST /assignments        PATCH /assignments/:id   (含 done 切換)
GET/POST /projects           GET /projects/:id (含 tasks/members/miles)
POST /projects/:id/tasks     PATCH /tasks/:id
GET /calendar?month=YYYY-MM  (union 推導)
GET /overview                (統計推導)
```

## 5. 邊界與取捨

- MVP 單體 vs 微服務：單體。元件以模組邊界劃分，Phase 2 需要時才把 parse/extract 拆成独立 worker 進程；
- 推導 vs 存狀態：全部推導（日曆、進度、逾期），減少同步 bug；
- LLM 信任邊界：抽取≠事實，confirm 才是寫入點；
- 隱私：Extension 不傳帳密；file 內容僅用於該使用者之抽取；提供一鍵刪除課件。