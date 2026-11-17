# SemTrack MVP Implementation Plan

Goal: 把已驗證的 Notion 風原型落地為可运行的輕量 Web app（Assignment + Group Project 進度，單一平台）。
Scope: MVP 前後端；Phase 2（Moodle Extension + LLM）僅預留邊界，不實作。
Non-goals: 多人即時協作、通知推送、真實 Moodle 連線、付費/帳號矩陣。

Affected files（建議 repo 結構，Next.js + Prisma + SQLite 起步）:
- `app/`（或 `src/app/`）: 頁面與 API routes
  - `page.tsx`: 總覽（對應原型 overview）
  - `assignments/page.tsx`: 資料庫表格 + 篩選 + 新增表單
  - `projects/page.tsx`、`projects/[id]/page.tsx`: 卡片與詳情
  - `calendar/page.tsx`: 月視圖（union 推導）
  - `api/assignments/route.ts`、`api/projects/route.ts`、`api/tasks/route.ts`、`api/calendar/route.ts`、`api/auth/*`
- `lib/derive.ts`: **唯一推導邏輯**（逾期/剩 N 天/進度%/日曆 union），前後端共用，移植自原型 `semtrack-prototype.html` 的 diff()/pjProg()/calEvents()
- `prisma/schema.prisma`: 见 backend-components.md §3 MVP 表
- `components/`: Tag、Check、ProgressBar、EmptyState、Dialog（移植原型 CSS token）
- `tests/derive.test.ts`: 推導邏輯單元測試

Steps:
1. 初始化 Next.js + Prisma + SQLite；建 schema 與 seed（移植原型 12 筆 assignments、2 個 projects 作為 demo seed，標註示範）。
2. 移植 `lib/derive.ts` 並先寫測試（TDD）：逾期邊界（今天=due 不算逾期）、done 不計逾期、進度 0/100、日曆 union 含 milestone 與 project due。
3. API routes：assignments CRUD + done 切換；projects/tasks CRUD；calendar/overview 推導 endpoint。每個 route 聚焦測試。
4. 前端四視圖移植原型 markup/CSS（Notion token：#FFFFFF/#F7F7F5/#37352F/粉彩 tag），改接 API；保持單一 state 推導、空狀態與驗證行為與原型一致。
5. 簡易 auth（email+密碼 session）；MVP 可先 single-user demo mode 開關。
6. 響應式與 a11y 复查（390/1280、focus-visible、reduced-motion），對照原型 QA 清單。
7. Phase 2 預留：`lib/ingest/` 介面定義（FileRegistry/ReviewQueue contract）但不實作 worker。

Verification:
- `npm test`（derive 單元 + route 整合）全綠；
- `npm run build` 無誤；
- 手動/Playwright smoke：新增→計數聯動、勾選→總覽/日曆聯動、空篩選恢復（對照原型已通過的 QA 矩陣）；
- seed 數據與原型數字一致（逾期 1、7 天內 3、完成 2/12、專題 33%/40%）。

Risks:
- 手動輸入摩擦（R1）：表單極簡＋Phase 2 自動匯入為留存關鍵；
- 技術棧未定案（Q3）：Next.js/Prisma 為建議，若團隊偏好其他棧，component 邊界不變；
- 推導邏輯兩套實作風險：强制共用 lib/derive.ts，禁止前端另算。

Rollback: 每 step 独立 commit；schema migration 可回退；demo mode 開關可繞過 auth 故障。