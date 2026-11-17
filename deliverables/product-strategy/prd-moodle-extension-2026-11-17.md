# PRD：SemTrack Moodle Extension — 一鍵 Scrap & Upload，作業截止自動上日曆

> 作者：小浣熊肖恩 · 版本 v0.1（草案，供研發評審）
> 語義徽章：【事实】可核驗｜【假设】待驗證推斷｜【判断】基於前兩者的建議｜【待确认】需拍板｜【本期不做】明確排除

## TL;DR 首屏結論

【判断】本需求值得做且可開工：MVP 手動輸入摩擦是留存最大風險，一鍵匯入是下一段價值槓桿。
- 依據 1：【事实】用戶痛點明確——逐個下載繁瑣、重複下載、deadline 埋在 slides；
- 依據 2：【事实】MVP 日曆/資料模型已就緒，匯入只需增量管線，架構復用度高；
- 依據 3：【判断】單站 adapter＋信心分流可把 LLM 污染風險控制在可回滾範圍。
決策建議：採「高信心自動入曆＋低信心人工確認」（D1 選項 A），評審會拍板 D1–D5 後即可拆 Sprint。
岗位就緒度：後端 Conditional Ready（契约已定，待 D1）；前端 Conditional Ready（待 D2 取樣）；測試 Not Ready（缺 D5 真實樣本，P0：AC-005 不可測）。

## 1. 文檔資訊

| 字段 | 內容 |
|---|---|
| 產品/功能 | SemTrack Moodle Connector（Chrome Extension MV3）＋ Ingest/抽取管線＋日曆呈現 |
| 版本 | v0.1 |
| 文檔狀態 | 草案（假設版初稿，決策清單見 §13.2） |
| 作者/負責人 | 小浣熊肖恩（PM 視角代筆） |
| 評審對象 | 前端（Extension＋Web）/ 後端 / 算法（LLM 抽取）/ 測試 |
| 上線範圍 | 內測：1 所目標學校 Moodle 站、受邀學生【假设】 |
| 依賴項 | SemTrack MVP（帳號、courses/assignments、推導式日曆）已上線【假设】；LLM API 可用 |
| 待決策項 | D1–D5（見 §13.2） |

## 2. 背景、問題與目標

- 當前業務背景：【事实】SemTrack MVP 已交付 Notion 風 Web 原型與架構設計（`docs/specs/product-spec.md`、`docs/architecture/backend-components.md`），MVP 資料靠手動輸入，日曆由 assignments/milestones 推導。
- 用戶痛點：【事实】（用戶原話收斂）① 課件要逐個下載，繁瑣；② 同一檔案重複下載、散落各處，檔案管理混亂；③ deadline 埋在 slides 裡，手動整理成本高，導致跟不上課程進度。
- 業務痛點：【判断】手動輸入摩擦是 MVP 留存的最大風險（spec R1）；沒有自動匯入，「單一平台看學期進度」的承諾難以兌現。
- 為什麼現在做：【判断】MVP 核心視圖已驗證，匯入自動化是下一段價值槓桿；先支援單站降低 adapter 成本。
- 本次北極星目標：【判断】學生在 Moodle 課程頁按一次「Scrap and upload to SemTrack」，該課本學期已知 assignment/考試截止**無需手動建檔**即出現在 SemTrack 日曆。
- 非目標 /【本期不做】：多校 adapter 矩陣、Moodle 以外的 LMS（Canvas/Blackboard）、PPT/DOCX 解析（僅預留）、到期提醒推送、多人共享、成績預測、代替使用者登入 Moodle。

## 3. 成功指標與反指標

> 【假设】以下目標值為內測校準用初始值，非實測數據；上線兩週後以埋點回填基線。不得以虛構數字對外宣稱。

| 類型 | 指標 | 口徑 | 目標值 | 數據來源 |
|---|---|---|---|---|
| 北極星 | 一鍵入曆完成率 | 按下按鈕的会话中，≥1 筆 deadline 成功入曆的比例 | ≥80%（初始） | ext_scrape_started / cal_event_created |
| 過程 | 按鍵到入曆耗時 P90 | 按鈕點擊 → 首筆事件可見 | ≤120s | 事件時間差 |
| 過程 | 重複檔案跳過率 | 同 hash 再次同步被跳過的比例 | 觀測項（無目標） | ingest_dedup_hit |
| 質量 | 抽取日期準確率 | 入曆事件日期與原文一致（抽樣人工核對） | ≥95%（高信心段） | 人工抽樣＋correction 埋點 |
| 質量 | 解析失敗率 | ingest job failed / 全部 job | ≤5% | ingest_jobs.status |
| 反指標 | 誤入曆修正率 | 用戶刪除/改期自動入曆事件的比例 | ≤5%，超標即回退全自动 | cal_event_corrected |
| 反指標 | 手動建檔佔比回升 | 匯入上線後手動新增佔比不升 | 觀測項 | assignment.source |

## 4. 用戶、場景與完整流程

- 目標用戶：【事实】大學生，修 3–6 門課，已登入學校 Moodle；已註冊 SemTrack。
- 核心場景：學期初/新课件發佈後，學生打開 Moodle 課程頁，想一次把這門課的 deadline 收進 SemTrack。
- 觸發條件：瀏覽器處於目標 Moodle 站的課程頁（`/course/view.php?id=…`）【假設：目標站 URL 形態，D2 確認】。
- 當前替代方案：逐個下載 PDF → 人工翻找日期 → 手動輸入 SemTrack 或記事本。

### 4.1 主流程（Happy Path）

1. 入口：學生在 Moodle 課程頁看到 Extension 注入的「Scrap and upload to SemTrack」按鈕（課程標題旁）；
2. 授權檢查：Extension 已綁定 SemTrack 帳號（首次使用走配對授權，見 C-002）；
3. 點擊按鈕 → content script 掃描本頁課程資源（PDF 連結＋所屬週次/section＋檔名），popup 顯示「找到 N 個檔案」清單；
4. 去重預檢：批次送 URL 指紋到 `POST /ingest/precheck`，已收錄檔案標「已存在，跳過」；
5. 上傳：新檔案以使用者既有 Moodle 登入態 fetch（同源 cookie），送 `POST /ingest/batch`；server 以 SHA-256 去重入 File Registry＋Object Storage；
6. 解析＋抽取：Parse Worker（PDF→頁級文字）→ Extract Worker（LLM 抽 `{type,title,date,time,confidence,source_page,quote}`，相對日期以學期起始日換算）；
7. 入曆【判斷，D1 待確認】：confidence ≥ 閾值（暫定 0.85）→ **自動寫入 assignments（source=extract）→ 日曆即時可見**；confidence < 閾值或日期模糊 → 進 Web「待確認」queue，通知角標；
8. 結果交付：popup 顯示「已同步 N 檔、新增 M 個 deadline、K 筆待確認」，附「在 SemTrack 查看」連結；
9. 後續：學生在 Web 日曆看到該課全部截止；待確認項一鍵確認/修改/忽略；自動入曆項可修改或刪除（記入修正埋點）。

### 4.2 分支與異常流程

| 場景 | 前置條件 | 系統行為 | 用戶可做動作 | 兜底策略 |
|---|---|---|---|---|
| 非課程頁 | URL 不匹配 adapter | 按鈕不注入；popup 顯示「請打開 Moodle 課程頁」 | 導航到課程頁 | 不誤抓無關頁面 |
| 未綁定帳號 | 無 SemTrack token | 按鈕可見但點擊引導配對（C-002） | 完成配對後重試 | 不上傳任何資料 |
| Moodle 未登入/登入過期 | fetch 回 401/登入頁 | 中止該檔，標「需重新登入 Moodle」 | 重新登入後重試 | 絕不代填帳密 |
| 頁面無 PDF | 掃描 0 檔 | popup 空狀態「本頁未找到課件」 | 手動上傳 fallback（Web 端） | 不產生空 job |
| 檔案已收錄 | hash 命中 | 跳過上傳，來源表追加一筆 | 可強制重新抽取（P1） | 解決重複下載 |
| 加密/損壞 PDF | 解析失敗 | job=failed，popup 列出檔名與原因 | 手動建檔該課 deadline | 不靜默丟棄 |
| LLM 抽不到日期 | 0 筆抽取 | 檔案仍入課件索引；提示「未找到日期」 | 手動建檔 | 檔案價值保留 |
| 日期模糊/低信心 | conf<閾值 | 進待確認 queue，標「推測」＋原文引用 | 確認/修改/忽略 | 日曆不被污染 |
| 重複同步同課程 | 再次點擊 | 檔案去重；抽取以 (course,title,type,date) 相似度去重 | 變更項提示「日期已更新？」（P1） | 不產生重複日曆事件【待確認 D4】 |
| 上傳中斷/斷網 | 網路錯誤 | 已完成部分保留；popup 顯示進度與重試 | 一鍵續傳（僅未完成檔） | 幂等：hash 去重保證重試安全 |
| 撤銷授權 | 用戶解除綁定 | token 失效，停止一切上傳 | 重新配對 | 已入庫檔案可一鍵刪除 |## 5. 範圍與優先級

| 優先級 | 模組/能力 | 用戶價值 | 本期範圍 | 不做內容 | 驗收口徑 |
|---|---|---|---|---|---|
| P0 | Extension 按鈕＋掃描＋上傳 | 一鍵代替逐個下載 | 單站 adapter、PDF、去重預檢、批次上傳、進度與結果 popup | 多校、非 PDF | AC-001/002/003 |
| P0 | Ingest＋File Registry | 同檔只存一份 | SHA-256 去重、file_sources 多來源、job 狀態 | 版本 diff | AC-004 |
| P0 | 解析＋LLM 抽取 | deadline 自動歸納 | 頁級文字、結構化抽取、信心分數、原文引用、相對日期換算 | OCR 掃描件 | AC-005/006 |
| P0 | 入曆＋待確認兜底 | 日曆即時可見且不被污染 | 高信心自動入曆（D1）、低信心 queue、修改/刪除 | 自動改期覆盖手動項 | AC-007/008 |
| P1 | 配對授權 UX | 首次 30 秒完成綁定 | Web 產生配對碼→popup 輸入 | OAuth 矩陣 | AC-009 |
| P1 | 強制重抽/更新偵測 | 課件改版跟得上 | 同 hash 重抽入口；新 hash 提示日期變更 | 自動覆盖 | AC-010 |
| P2 | 課件索引頁 | 集中檔案管理 | Web 課程→週次→檔案列表 | 預覽/全文搜索 | AC-011 |
| 【本期不做】 | 提醒推送、多 LMS、共享協作 | — | — | 全部 | — |

## 6. 組件級需求

| 編號 | 頁面區域 | 組件 | 觸發條件 | 顯示規則 | 交互規則 | 異常規則 | 驗收標準 |
|---|---|---|---|---|---|---|---|
| C-001 | Moodle 課程頁 | 注入按鈕「Scrap and upload to SemTrack」 | adapter URL 匹配 | 課程標題旁，SemTrack 品牌色，不遮擋原頁功能 | 點擊開 popup 並開始掃描 | 非課程頁不注入 | 注入/不注入各 1 例通過 |
| C-002 | Extension popup | 配對授權面板 | 無有效 token | 輸入配對碼＋「到 SemTrack 取得配對碼」連結 | 驗證成功存 token（chrome.storage.local） | 碼錯誤保留輸入＋提示；過期碼提示重取 | AC-009 |
| C-003 | popup | 掃描結果清單 | 掃描完成 | 每檔：檔名/週次/狀態（新檔/已存在/失敗） | 可取消勾選個別檔案（P1）；預設全選新檔 | 0 檔空狀態＋手動上傳引導 | 清單與頁面實際資源一致 |
| C-004 | popup | 同步進度條＋結果摘要 | 上傳開始 | x/N 檔、即時狀態；完成後「新增 M 筆、K 筆待確認」 | 「在 SemTrack 查看」開 Web 對應課程頁 | 中斷顯示重試按鈕 | AC-003 |
| C-005 | Web 日曆 | 自動入曆事件 | source=extract 寫入 | 與手動事件同視圖；帶「自動匯入」小標 | 點開可看來源檔/頁碼/原文引用；可改/刪 | 修改與刪除記入修正埋點 | AC-007 |
| C-006 | Web 待確認頁 | Review queue | 低信心抽取 | 標題/日期/類型可編輯＋信心標＋quote | 確認/修改後確認/忽略 | 確認前不進日曆 | AC-008 |
| C-007 | Web 課程頁 | 課件索引（P2） | 檔案入庫 | 課程→週次→檔案（hash 唯一）＋「來自 N 個來源」 | 點擊開來源 Moodle 連結 | 刪除課件需二次確認 | AC-011 |

## 7. 任務/配置規格（抽取管線）

| 任務 | 輸入要求 | 輸出物 | 默認策略 | 執行步驟 | 風險提示 | 後續動作 |
|---|---|---|---|---|---|---|
| Parse | PDF bytes＋mime 校驗 | 頁級 text＋頁碼映射 | PyMuPDF；>30MB 或加密→failed | 驗證→抽取→存 text | 掃描件無文字層→標 no_text，不硬抽 | 觸發 Extract 或標記失敗 |
| Extract | 頁級 text＋semester.start_date＋course.code | extractions[]（type/title/date/time/confidence/source_page/quote） | LLM 結構化輸出（JSON schema 強校驗）；溫度 0；每檔一次呼叫＋超長分段 | prompt→schema 校驗→相對日期換算→信心評分 | 幻覺日期：quote 必須能在原文定位，否則丟棄該筆 | 高信心→入曆；低→queue |
| 相對日期換算 | 「下週五」「Week 12 Friday」等 | 絕對日期 | 以學期起始日＋課表週曆；無法定位→conf≤0.5 | 規則優先，LLM 兜底 | 學期起始日未設定→全部進 queue＋提示設定 | D3 |
| 去重 | 新抽取 vs 既有 assignments | 新建/疑似重複標記 | (course_id,type,title 相似度≥0.9,date 相同)→跳過；date 不同→標「疑似改期」進 queue | 比對→分流 | 誤判重複→漏事件；相似度閾值內測校準 | D4 |

## 8. 狀態機與狀態流轉（單次同步會話）

| 狀態 | 進入條件 | 展示內容 | 可操作項 | 下一狀態 | 埋點 |
|---|---|---|---|---|---|
| idle | popup 開啟且已綁定 | 「開始掃描」 | 點擊按鈕 | scanning | ext_popup_open |
| needs_auth | 無/失效 token | 配對面板 | 輸入配對碼 | idle / auth_failed | ext_auth_attempt |
| scanning | 用戶點擊 | 掃描中 spinner | 取消 | scanned / scan_empty / scan_failed | ext_scrape_started |
| scanned | ≥1 檔 | 檔案清單＋預檢結果 | 開始同步/取消 | uploading | ext_scrape_done |
| uploading | 用戶確認 | x/N 進度 | 取消（保留已完成） | uploaded / partial_failed | ext_upload_progress |
| processing | 上傳完成 | 「解析中，可關閉 popup」 | 關閉（背景繼續） | done | ingest_job_status |
| done | 全部 job 終態 | 新增 M/待確認 K/失敗 F＋連結 | 開 Web/重試失敗檔 | idle | ext_sync_done |
| partial_failed / scan_failed | 任一失敗 | 原因＋重試 | 重試/手動 fallback | uploading / scanning | ext_sync_error |

> 【判断】processing 非同步：LLM 抽取可達數十秒，popup 不阻塞等待；Web 端角標通知完成，避免使用者掛在 popup。## 9. 埋點與指標字段

| 事件名 | 觸發時機 | 字段 | 示例值 | 用途 | 隱私邊界 |
|---|---|---|---|---|---|
| ext_scrape_started | 按鈕點擊 | course_moodle_id, adapter_ver | 4821, v1 | 漏斗入口 | 不記 URL 全文/帳密 |
| ext_scrape_done | 掃描完成 | files_found, files_new | 12, 5 | 去重效果 | 同上 |
| ingest_dedup_hit | precheck 命中 | file_hash_prefix | a1b2c3d4 | 重複下載量化 | hash 前 8 位即可 |
| ext_upload_progress | 每檔完成 | ok, failed, bytes | 5, 0, 8388608 | 成功率/耗時 | 不記檔名原文（可選 hash） |
| ingest_job_status | job 終態 | status, fail_reason_code | failed, encrypted | 解析質量 | 錯誤碼枚举，不含內容 |
| cal_event_created | 自動入曆 | extraction_id, confidence_band | 901, high | 北極星分子 | 不記課程成績類內容 |
| review_confirmed / review_dismissed | queue 操作 | extraction_id, edited(bool) | 902, true | 抽取質量校準 | — |
| cal_event_corrected | 自動入曆項被改/刪 | field_changed | due_date | 反指標：誤入曆率 | — |
| ext_sync_done | 會話結束 | duration_ms, created, pending, failed | 74000, 6, 1, 0 | P90 耗時 | — |

## 10. 接口、配置與協作建議

### 10.1 前端（Extension＋Web）
- Extension：MV3；content script（adapter 掃描＋注入按鈕）／service worker（fetch 上傳、token 管理）／popup（清單/進度/配對）；permissions 最小化：`storage`＋目標站 host permission【判断】不申請 `<all_urls>`、不申請 tabs 讀取；
- Adapter 以配置下發（JSON：URL pattern＋resource selector＋section selector），站點改版可熱修不發版【判断】；
- Web：沿用 MVP Notion token；新增「待確認」queue 頁與事件來源詳情（quote/頁碼）；日曆事件加「自動匯入」標；
- 可訪問性：popup 全鍵盤可達、狀態 aria-live；注入按鈕不與 Moodle 原控件重疊。

### 10.2 後端 / 配置（沿用既有架構，增量如下）
```
POST /ext/pair            {pair_code} → {ext_token}          # 配對，碼 10 分鐘有效、一次性
POST /ingest/precheck     {course_id, items:[{url_fingerprint,name,size}]} → [{status:new|exists, hash?}]
POST /ingest/batch        multipart：files[] + {course_id, moodle_course_id, sources[]}
                          → {job_ids[], dedup_skipped[]}      # 幂等：同 hash 重送安全
GET  /ingest/jobs?ids=    → [{id,status,fail_reason_code}]    # popup/Web 輪詢或 SSE（P1）
GET  /review              → 低信心 extractions（含 quote/source_page）
POST /review/:id/confirm  {title?,date?,type?} → 寫入 assignments(source=extract)
POST /review/:id/dismiss
GET  /courses/:id/files   → 課件索引（P2）
```
- 权限：ext_token 綁 user＋scope=ingest only，可獨立撤銷，不用主 session；server 端所有寫入校驗 course 歸屬該 user；
- 自動入曆（D1 通過後）：Extract Worker 對 conf≥threshold 直接 transaction 寫 assignments＋發 cal_event_created；threshold 為後端配置可灰度調整；
- 資料模型：沿用 `files / file_sources / ingest_jobs / extractions`（backend-components.md §3），assignments.source 增加 `extract`；新增 `ext_tokens(id,user_id,token_hash,created_at,revoked_at)`、`pair_codes(code_hash,user_id,expires_at,used_at)`；
- 檔案處理：mime 白名單（application/pdf）、單檔 ≤30MB、單次 ≤50 檔【假設，內測校準】；存 Object Storage，DB 只存 metadata；
- LLM 成本控制：每檔抽取結果快取於 job；同 hash 不重抽（P1 強制重抽除外）。

### 10.3 降級與回滾
- 降級鏈：自動入曆開關（feature flag）→ 關閉後全部進 queue（回到原架構保守模式）→ 抽取服務不可用時檔案仍入索引＋提示手動建檔；
- 回滾開關：`auto_calendar_enabled`（全局＋per-user）；adapter 配置可回退上一版；
- 灰度：內測 1 校 → 觀察反指標（誤入曆修正率 ≤5%）兩週 → 擴站；
- 一鍵刪除：使用者可刪課程全部匯入檔案與 source=extract 事件（手動項不受影響）。

## 11. 驗收用例

| 用例 ID | 優先級 | Given | When | Then | 驗收方式 |
|---|---|---|---|---|---|
| AC-001 | P0 | 已綁定＋目標 Moodle 課程頁（含 5 個 PDF，其中 2 個已收錄） | 點按鈕→確認同步 | 清單顯示 5 檔、2 檔標「已存在」；僅上傳 3 檔 | 內測站實測＋precheck 單測 |
| AC-002 | P0 | 非課程頁（Moodle 首頁/他站） | 打開頁面 | 按鈕不注入；popup 顯示引導 | adapter 單測（URL fixture） |
| AC-003 | P0 | 上傳中斷網 | 恢復後點重試 | 僅續傳未完成檔；無重複入庫 | 幂等整合測試 |
| AC-004 | P0 | 同 PDF 出现在兩門課 | 兩課各同步一次 | files 表 1 筆；file_sources 2 筆 | DB 斷言 |
| AC-005 | P0 | PDF 內含「Assignment 5 due 11 Dec 23:59」 | 管線完成 | extraction date=2026-12-11、quote 可定位原文 | 標注 fixture 集（≥20 份真實格式樣本）【待確認 D5 取樣】 |
| AC-006 | P0 | 內含「下週五交」＋學期起始日已設 | 抽取 | 換算為絕對日期；未設起始日→進 queue＋提示 | 規則單測＋fixture |
| AC-007 | P0 | conf≥0.85 抽取（D1 通過） | job 完成 | 日曆出現該事件（source=extract 標記）；改期後修正埋點觸發 | E2E＋埋點斷言 |
| AC-008 | P0 | conf<0.85 或模糊日期 | job 完成 | 不進日曆；queue 可見 quote；確認後才入曆 | E2E |
| AC-009 | P1 | 未綁定 | popup 輸入有效配對碼 | 30 秒內完成綁定；錯碼保留輸入 | 可用性測試 |
| AC-010 | P1 | 已收錄檔案強制重抽 | 點重抽 | 新 extraction 進 queue（不自動覆盖既有事件） | 整合測試 |
| AC-011 | P2 | 課程已同步 | 開課件索引 | 同 hash 只列 1 筆＋「來自 N 個來源」 | DB＋UI 斷言 |
| AC-012 | P0 | 任意同步會話 | 全程 | 無 Moodle 帳密離開瀏覽器；ext_token 撤銷後上傳被拒 | 安全測試（攔截請求斷言） |

## 12. 測試、灰度與上線計劃

- 測試範圍：adapter URL/DOM fixture 單測；ingest 幂等與去重整合；抽取 fixture 集（含中英混排、相對日期、表格型 syllabus、無日期課件）；E2E（內測 Moodle 站或本地 Moodle docker【判断】本地 docker 起站可脫離真實校網測試）；
- 灰度人群：1 校 10–20 名受邀學生【假设】；
- 觀測週期：2 週；每週人工抽樣 ≥30 筆入曆事件核對日期；
- 回滾條件：誤入曆修正率 >5%、或解析失敗率 >10%、或任何隱私事件 → 關 auto flag／停用 adapter；
- 上線檢查清單：權限最小化審核、token 撤銷路徑、一鍵刪除、埋點驗收、fixture 準確率報告、LLM 成本單檔上限告警。

## 13. 風險、依賴與開放問題

### 13.1 風險與依賴

| 類型 | 風險/問題 | 等級 | 影響 | 建議處理 | Owner | 截止時間 |
|---|---|---|---|---|---|---|
| 技術 | Moodle 版本/主題 DOM 差異 | P1 | 掃描漏檔/誤抓 | adapter 配置化＋fixture 回歸；先單站 | 前端 | 開發前取樣 |
| 技術 | LLM 幻覺日期污染日曆 | P0 | 學生錯過/誤信 deadline | quote 原文定位校驗＋信心分流＋反指標回滾 | 算法 | 上線前 |
| 技術 | 相對日期/學期曆法換算錯誤 | P1 | 日期偏移一週 | 規則優先＋未設起始日強制 queue | 後端 | 上線前 |
| 合規 | 抓取涉及校方平台條款與個人資料 | P1 | 校規/隱私爭議 | 僅使用者自有登入態、明示授權、最小收集；**校方條款與合規結論須法務/校方確認，本 PRD 不代下結論** | PM＋法務 | 內測前 |
| 產品 | 自動入曆與既有人工確認設計張力 | P0（決策） | 架構與信任模型 | D1 拍板後統一兩份文檔 | PM | 評審會 |
| 依賴 | SemTrack MVP 帳號/課程/日曆 API 就緒 | P0 | 無寫入點 | 與 MVP 開發並行對齊契約 | 後端 | Sprint 0 |

### 13.2 決策清單（評審會需拍板，3–7 個）

| # | 決策 | 選項與建議 | 影響 |
|---|---|---|---|
| D1【待确认】 | 高信心抽取是否自動入曆？ | A. conf≥0.85 自動＋低信心 queue（**建議**，貼合「按下按鈕就出現在日曆」的用戶要求，反指標兜底）；B. 全部人工確認（原架構保守模式，多一步摩擦） | 北極星體驗 vs 污染風險；§7/§10/AC-007 依此定稿 |
| D2【待确认】 | 首發目標學校與 Moodle 版本？ | 需 1 所可取樣學校（URL 形態、主題、資源類型） | adapter 開發與 fixture 取樣前提 |
| D3【待确认】 | 學期起始日來源？ | A. 用戶在 Web 手動設定（**建議** MVP 沿用）；B. 從 Moodle 校曆抓取 | 相對日期換算準確率 |
| D4【待确认】 | 疑似改期（同題不同日）處理？ | A. 進 queue 讓用戶裁決（**建議**）；B. 自動更新＋通知 | 漏事件 vs 覆蓋手動修改的風險 |
| D5【待确认】 | 抽取準確率驗收樣本誰提供？ | 需 ≥20 份目標校真實格式課件（去識別化） | AC-005 可測性；缺樣本則先用合成 fixture＋標注侷限 |

## 14. 版本記錄

| 版本 | 日期 | 變更 | 作者 |
|---|---|---|---|
| v0.1 | 2026-11-17 | 假設版初稿：主流程/異常/組件/狀態機/契約/驗收/決策清單 D1–D5 | 小浣熊肖恩 |