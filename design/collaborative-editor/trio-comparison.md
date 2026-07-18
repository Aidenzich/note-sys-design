# 線上協作文件編輯器三系統深度設計比較:Google Docs vs Confluence vs Notion

> 來源:2026-07-18 的 read-only 調查(所有外部引文擷取日期 = 2026-07-18;皆為一手優先)。逐字引文 + URL 收錄於文末 Evidence Appendix,正文以 `[A#]` 錨點回指。
> 姊妹篇:[Google Docs vs Confluence 雙系統版](./README.md)(較精簡;本篇為獨立重新取證的三系統版,多了 Notion 與更完整的反證彙整)。

---

## 0. 一句話結論(先給答案)

三個系統名義上都是「多人協作文件」,但**文件模型(document model)決定了協作演算法,三者其實站在三個不同的架構象限**:

- **Google Docs = 字元級即時協作的標竿。** 文件本體就是一條**append-only 的操作日誌(op log,官方稱 revision log)**,靠 **OT(Operational Transformation,操作轉換)**在中央伺服器做字元級(character-by-character)合併。這是三者中唯一、且由官方一手文件明確證實的「真・即時字元級協作」。`[A1][A2]`
- **Notion = block 樹 + Postgres 分片的工作區。** 文件是**一棵 block(區塊)樹**,每個 block 是 PostgreSQL 裡的一列(row);線上編輯是「**operations 打包成 transaction 交給伺服器整批 commit/reject**」的模型,官方**從未宣稱線上用 OT 或 CRDT**;CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別)只在 2025-12 上線的**離線(offline)**路徑用到。其最硬的一手素材是公開的 **Postgres 分片史(480 邏輯分片、32→96 台實例)**。`[A7][A8][A9][A11]`
- **Confluence = 企業 wiki,DB 存整份 XHTML blob。** Server/Data Center 版把整頁內容當一份 **XHTML-based「storage format」字串**存進關聯式資料庫(BODYCONTENT 表),版本歷史是**每次編輯存一份完整新版本**;即時協作由獨立服務 **Synchrony** 提供,走 **WebSocket(失敗退回 XHR)**——但**「Synchrony 用 OT」這件事,Atlassian 一手文件並未寫明**,只能標 UNKNOWN。`[A12][A13][A14][A16][A17]`

**一句話取捨定位:** Google Docs 為「即時字元級」把整個儲存模型改成 op log 並吃下 OT 的複雜度;Notion 為「靈活的 block 工作區 + 巨量擴展」選了 block-per-row + 分片,協作精緻度退到 record/block 級整批交易;Confluence 為「企業自管、相容既有 DB」把文件當 blob 存進傳統 RDBMS,即時協作是後來(2016 Confluence 6.0)才用外掛服務 Synchrony 補上的能力層。

> **可信度分層提醒:** Google Docs 的協作演算法、Notion 的資料模型與分片數字、Confluence 的儲存/搜尋引擎都是**一手 FACT**;而三者的「內部資料庫選型/快取/搜尋引擎/傳輸協定」有相當一部分官方**從未公開**,本文一律標 **UNKNOWN + closure recipe**,不以二手部落格冒充事實。特別是網路盛傳的「Notion 用 CRDT 做即時協作」「Confluence Synchrony 是 OT」兩則,經反證後**都不是一手來源可證的**——詳見 §6。

---

## 1. 定義問題(Define the Problem)

### 1.1 這份 spike 為什麼要做

「線上協作文件編輯器」是系統設計面試與真實架構裡的經典題:同一份文件、多人同時改、還要即時看到彼此的游標與字元。但市面上被歸為同一類的三個代表產品——**Google Docs**(消費級即時協作標竿)、**Confluence**(企業 wiki)、**Notion**(block 式工作區)——在**儲存**與**協作**兩大面向的內部設計其實差異巨大。本 spike 要做的不是列功能,而是**像一份系統設計深度解析**:先立「這類系統該怎麼比」的設計面向(rubric,評比量尺),再逐系統用一手證據舉證,最後建三欄比較表並回答「三者在架構取捨上各站在什麼位置、為什麼」。

### 1.2 要回答的具體問題

1. **儲存(Storage):** 文件存進哪種資料庫?文件模型是整份 blob、操作日誌 op log、block 樹、還是字元序列?快取怎麼用?二進位(圖片/附件)是否與文字分離、是否放 object storage(如 S3)?全文檢索 FTS(Full-Text Search)用什麼引擎、怎麼建索引?
2. **協作(Collaboration):** 多人同編用什麼演算法/協定(OT、CRDT、鎖、存檔式 / last-write-wins)?粒度是字元級、區塊級還是整頁級?即時傳輸走 WebSocket、long-poll、SSE 還是純 REST?前端如何呈現多人同編(presence/游標、即時更新、還是存檔後才見)?衝突怎麼解?
3. **其他考題(酌情):** 版本歷史/diff/回溯、離線編輯與同步、擴展性(sharding)、以及「文件模型如何決定協作演算法」這條主線。

### 1.3 調查沿哪些面向展開(dimensions)

下表是本文從頭到尾使用的評比面向;每一項在 §4 三欄表中都有對應欄位。

| 面向 | 具體子問題 |
|---|---|
| **D1 文件模型** | blob / op log / block 樹 / 字元序列?current content 如何得出? |
| **D2 資料庫** | 哪種 DB?一列存什麼? |
| **D3 二進位/物件儲存** | 圖片附件放哪?是否與文字分離、是否 S3? |
| **D4 全文檢索 FTS** | 什麼引擎?怎麼索引? |
| **D5 版本/歷史** | 全版本快照 vs 差異 diff vs op log 重放? |
| **D6 協作演算法** | OT / CRDT / 鎖 / 存檔式 / 整批交易? |
| **D7 協作粒度** | 字元級 / 區塊級 / 整頁級? |
| **D8 即時傳輸** | WebSocket / long-poll / SSE / REST? |
| **D9 呈現與衝突** | presence/游標?即時 vs 存檔後?衝突如何解? |
| **D10 擴展性** | 分片 sharding?如何擴? |

---

## 2. 先教再用:名詞與文件模型 Primer

比較開始前,先把後面論證會靠的結構性概念用**最小具體實例**講清楚。這一節的概念都在 §4 的表格與 §6 的論證中被使用,故放在前面。

### 2.1 四種文件模型(document model)

「文件在資料庫裡長什麼樣」有四種典型形狀,是本文最核心的分類軸:

**(a) 整份 blob(whole-blob)** — 一份文件 = 一大串文字/標記,整包存進一個欄位或檔案。取一個版本就取整包。**Confluence Server/DC 屬此類**:整頁是一份 XHTML(Extensible HyperText Markup Language,XML 家族的標記語言)字串,叫「storage format(儲存格式)」。長這樣:

```xml
<!-- Confluence storage format:整頁 body 就是一份 XHTML/XML 字串,整包存進 BODYCONTENT 表的一列 -->
<p>Hello <strong>world</strong></p>
<ac:structured-macro ac:name="info">
  <ac:rich-text-body><p>注意事項</p></ac:rich-text-body>
</ac:structured-macro>
```
編輯 = 讀出整份字串、改、寫回一份新版本字串。`[A12]`

**(b) 操作日誌 op log(operation log)** — 文件本體**不是**當前文字,而是「一連串改動(操作)」的**append-only 清單**;「當前內容」是把 log**從頭重放(replay)**算出來的。**Google Docs 屬此類**,官方稱 revision log。最小實例:打「Hi」再退格刪掉「i」,op log 是:

```jsonc
// Google Docs revision log(示意):文件 = 一串按時間排序的改動,不是當前字串本身
{ "InsertText": "H", "at": 1 }   // 位置 1 插入 H      → 內容 "H"
{ "InsertText": "i", "at": 2 }   // 位置 2 插入 i      → 內容 "Hi"
{ "DeleteText": "@2-2" }         // 刪掉位置 2 的字元   → 內容 "H"
// 「當前內容」不是存下來的字串,而是把上面三筆從頭 replay 得到的結果 "H"
```
官方原文:文件「stored as a series of chronological changes」,而且「To display a document, we replay the revision log from the beginning」。`[A1][A2]` 所有編輯只歸為三種操作:InsertText、DeleteText、ApplyStyle。`[A2]`

**(c) block 樹(block tree)** — 文件是一棵樹:每個「區塊(block,一段文字、一張圖、一列資料庫、甚至頁面本身)」是一筆有 ID 的記錄,記著自己的 `content`(子 block 的 ID 陣列)與 `parent`(父 block ID)。**Notion 屬此類**。最小實例:一個頁面含一段文字和一個項目符號清單:

```jsonc
// Notion block 樹(示意):每個 block 是一列記錄,靠 content 陣列串成樹
{ "id": "pageA", "type": "page",     "content": ["b1", "b2"], "parent": "workspace" }
{ "id": "b1",    "type": "text",     "properties": { "title": "Hello" }, "parent": "pageA" }
{ "id": "b2",    "type": "bulleted_list_item", "properties": { "title": "item" }, "parent": "pageA" }
// 「整頁內容」= 從 pageA 沿 content 指標往下遞迴收集子 block(官方 API:loadPageChunk)
```
官方原文:「Everything you see in Notion is a block」;每個 block 有 `Content`(「an array (or ordered set) of block IDs representing the content inside this block」)與 `Parent`;載頁面的 API `loadPageChunk`「descends… down the content tree」。`[A7]`

**(d) 純字元序列(character sequence)** — 文件是一長串字元/位置索引。Google Docs 的 op log 內部即以字元位置索引運作(`InsertText 'T' @10` = 在第 10 位插入 T),可視為 (b)+(d) 的混合:op log 記錄的是對字元序列的操作。`[A1]`

> **一句話串起主線:** 文件模型直接決定了協作演算法能做多細——op log/字元序列讓 OT 能做**字元級**合併;block 樹讓協作退到 **block/record 級**整批交易;整份 blob 則天生難即時協作,要靠外掛服務(Synchrony)另闢即時層。這條「模型決定演算法」是全文論證主線。

### 2.2 協作演算法:OT vs CRDT vs 鎖 vs 存檔式

- **OT(Operational Transformation,操作轉換):** 每個人的改動是「操作」;當你的操作和別人的並發操作撞在一起,系統把對方的操作「轉換(transform)」成相對於你當前文件仍正確的版本。通常需要**中央伺服器當仲裁者**排序。Google Docs 用它。官方:「The algorithm that we use to handle these shifts is called operational transformation (OT)」。`[A3]`
  - 最小直覺:原文 `EASY`,你在最前面插了 `IT`(變 `ITEASY`),此時對方送來「刪除 @1」。若照字面刪你的第 1 位會刪錯字;OT 把對方的「刪除 @1」**平移 2 格**成「刪除 @3」,才正確。`[A3]`
- **CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別):** 資料結構本身數學上保證「不同順序套用同一組改動,最終收斂到同一結果」,理論上可去中心化、離線友善。Notion 只在**離線頁面**用到,且稱之為「new CRDT data model」(2025-12)。`[A11]`
- **鎖(locking):** 同一段(如整段落)同時只准一人改。Google 明確**評估後否決**了段落鎖:「you could lock paragraphs… But locking paragraphs isn't a great solution」。`[A2]`
- **存檔式 / last-write-wins(LWW,後寫覆蓋):** 沒有細粒度合併,誰最後存誰贏。Notion 線上模型的「operations 打包成 transaction、伺服器整批 commit/reject」在缺乏一手 OT/CRDT 說明的情況下,行為上偏向 record 級的伺服器仲裁(見 §6.3)。`[A10]`

### 2.3 其他縮寫(第一次出現即解釋)

- **FTS(Full-Text Search,全文檢索):** 讓使用者用關鍵字搜文件內文的索引系統(如 Apache Lucene、OpenSearch/Elasticsearch)。
- **CDC(Change Data Capture,變更資料捕捉):** 把資料庫的每一筆變更即時「串流」出來給下游(Notion 用 Debezium 做 Postgres CDC)。`[A10]`
- **VACUUM / TXID wraparound:** PostgreSQL 的維護機制。`VACUUM` 回收被刪列佔用的空間;`TXID`(transaction ID)是交易編號,若逼近上限會「wraparound(繞回)」而 Postgres 為安全會**停止所有寫入**——這正是 Notion 被迫分片的導火線。`[A8]`
- **presence / telepointer:** 「presence」= 顯示誰在線上一起編輯;「telepointer」= 對方的游標/選取在你畫面上的即時彩色游標。

---

## 3. 量測準則(Measurement Criteria)—— 判斷答案的量尺

在看證據前,先定義「什麼叫把某一面向講清楚了」。本文對每個系統、每個面向,用以下三檔標示證據強度,並要求承重結論可被一手來源反查:

| 標籤 | 定義 | 收錄要求 |
|---|---|---|
| **FACT** | 有一手來源(官方工程 blog / 官方文件 / 官方 API doc / 專利 / 官方影片)逐字佐證 | URL + 逐字引文 + 日期(見 Appendix) |
| **INFERENCE** | 由一手事實合理推導,但官方未直接明說 | 標明由哪條 FACT 推得 |
| **UNKNOWN** | 官方未公開,或只有二手來源 | 附 closure recipe(要用什麼查詢/在哪裡能關掉這個未知) |

**「好答案」長什麼樣(pass 條件):**
1. 每個面向都能落到 FACT / INFERENCE / UNKNOWN 其中一檔,不含糊。
2. 每個承重結論至少跑過一次**反證搜尋**(例:三家「到底 OT 還是 CRDT 還是都不是」都要查),並回報反證結果或說明反證查空。
3. 二手系統設計部落格**不得**冒充 FACT;snippet(搜尋結果摘要)不算證據,必須實際打開頁面讀過。
4. 三系統協作能力差異若很大,照證據寫,不為對稱美化任何一方。

---

## 4. 多來源舉證與三欄比較(核心產物)

以下每格都標了證據檔位並回指 Appendix 錨點。**這張三欄表是本 spike 的核心交付物。**

### 4.1 三欄比較表(Storage + Collaboration)

| 面向 | **Google Docs** | **Confluence**(Server/DC 為主) | **Notion** |
|---|---|---|---|
| **D1 文件模型** | **op log(revision log)**:文件=一串按時間排序的改動;三種操作 Insert/Delete/ApplyStyle;current content 靠**從頭 replay** 算出 — **FACT** `[A1][A2]` | **整份 XHTML blob(storage format,技術上是 XML)**,存於 `BODYCONTENT` 表一列 — **FACT** `[A12][A13]`。Cloud 改用 ADF(Atlassian Document Format,JSON) — **FACT(有即時同編);內部細節 INFERENCE** `[A18]` | **block 樹**:每 block 一筆記錄,含 ID/Type/Properties/`content`(子 block ID 陣列)/`parent`;整頁靠 `loadPageChunk` 遞迴 content 樹重建 — **FACT** `[A7]` |
| **D2 資料庫 / 一列存什麼** | **UNKNOWN**:官方從未公開 Docs 用哪種 DB(Bigtable/Spanner 皆為二手臆測);Colossus 為 Drive 級依賴的 **INFERENCE**。伺服器持有「完整 revision log + 最新一次處理後的文件狀態」 — 後者為 **FACT** `[A1]` | **PostgreSQL / MySQL / Oracle / SQL Server**(Server/DC 支援矩陣;不支援 MariaDB/Percona) — **FACT** `[A16]`。一列 = 一個 `ContentEntityObject`(CONTENT 表存版本/中繼,BODYCONTENT 表存 body) — **FACT** `[A13]` | **PostgreSQL**,source-of-truth;**每個 block = 一列(row)** — **FACT** `[A7][A9][A10]`。`properties`/`content` 是否以 JSON 欄位存 = **UNKNOWN** |
| **D3 二進位 / 物件儲存** | **INFERENCE/UNKNOWN**:2010 模型只涵蓋文字/樣式三操作,未提圖片;圖片是否分離存 blob store 無一手佐證 | **檔案系統為預設;Confluence 8.1 起支援 Amazon S3;DB/WebDAV 儲存已不再支援** — **FACT** `[A15]`(文字與二進位**分離**) | **S3 用於 data lake** — **FACT** `[A10]`;使用者上傳檔案/圖片是否也走 S3 = **INFERENCE** |
| **D4 全文檢索 FTS** | **UNKNOWN**:Drive/Docs 內部搜尋引擎未公開;「首 100 頁/OCR」等數字僅見二手 | **Apache Lucene 為預設**(索引落在 `<home>/index`);**Data Center 8.9+ 可選 OpenSearch** — **FACT** `[A17]` | **UNKNOWN**:官方只預告「Search and AI Embedding RAG Infra」文章;Elasticsearch 為**二手臆測**,非一手 — **UNKNOWN** `[A10]` |
| **D5 版本 / 歷史** | op log 天然即完整歷史;可回放任一 revision — **FACT** `[A1][A2]` | **每次編輯建一份完整新版本、全部保留;還原舊版=複製該版成新的當前版** → 全版本快照(diff 只是顯示層) — **FACT(全版本);「非 diff 儲存」為 INFERENCE** `[A14]` | 200B+ blocks 規模下的頁面歷史機制官方未細述 — **UNKNOWN**;離線衝突解由 CRDT 處理 — **FACT** `[A11]` |
| **D6 協作演算法** | **OT(Operational Transformation)**,中央伺服器仲裁;明確否決「版本比對」與「段落鎖」 — **FACT** `[A2][A3]` | **Synchrony 服務即時同步**;每頁一份 Synchrony change log(「a graph of all edits」) — **FACT** `[A16]`。**演算法是否為 OT:Atlassian 一手文件未寫明 → UNKNOWN(僅二手稱 OT)** `[A16]` | 線上:**operations 打包成 transaction,伺服器整批 commit/reject** — **FACT** `[A10]`;**官方未宣稱線上用 OT 或 CRDT** → 演算法 **UNKNOWN**;**CRDT 僅用於離線頁面(2025-12 上線的「new CRDT data model」)** — **FACT** `[A11]` |
| **D7 協作粒度** | **字元級(character-by-character)** — **FACT** `[A4]` | 富文字即時同編(telepointers/遠端選取) — **FACT**;精確粒度演算法層未公開 — 連動 D6 UNKNOWN `[A16]` | **record/block 級**(操作以「create or update a single record」為單位)— **INFERENCE**(由 D6 FACT 推);離線「rich-text conflict resolution」暗示 CRDT 下有子 block 級合併 — **FACT(有此問題);粒度 INFERENCE** `[A10][A11]` |
| **D8 即時傳輸** | **UNKNOWN**:2010 文只描述邏輯協定(client 送一筆改動、server ack/broadcast),未點名 WebSocket/long-poll/SSE — **UNKNOWN** `[A5]` | **WebSocket,取不到時退回 XHR(XML HTTP Request)** — **FACT** `[A16]`;Cloud 亦以 WebSocket 對 Synchrony 開 session — **FACT** `[A18]` | 離線頁面走「channels」訂閱+收訊後 fetch — **FACT**;是否 WebSocket = **INFERENCE** `[A11]` |
| **D9 呈現 / 衝突解** | 即時、每個 editor 把收到的改動 transform 成相對本地版本正確;樂觀本地先套用、一次只送一筆 pending change — **FACT** `[A5]`;彩色游標 UI 機制 — **UNKNOWN** | 即時同編、顯示協作者 avatar 與遠端游標(telepointer);Synchrony 為 source of truth,離線緩衝、回線合併 — **FACT** `[A16]` | 離線:先本地改、回線再同步;衝突由 CRDT(離線)處理 — **FACT** `[A11]`;線上 presence/游標機制 — **UNKNOWN** |
| **D10 擴展性** | 未公開 — **UNKNOWN** | 自管;水平擴展靠 Data Center 叢集(細節本文未取一手) — **UNKNOWN** | **公開的一手素材**:**480 邏輯分片(Postgres schema)**,原 **32 台實例(每台 15 schema)**,2023 **re-shard 到 96 台(每台 5 schema)**,480 恆定;分片鍵=**workspace ID**;動機=VACUUM 停滯 / TXID wraparound;PgBouncer 零停機切換 — **FACT** `[A8][A9][A10]` |

### 4.2 D1「文件模型」並排(結構最能一眼看出取捨)

| | 一份文件在儲存層長什麼樣 | 「當前內容」怎麼得出 | 決定了什麼協作能力 |
|---|---|---|---|
| **Google Docs** | append-only op log:`[{Insert H@1},{Insert i@2},{Delete@2}]` | 從頭 replay | 字元級 OT(最細) |
| **Notion** | 一堆 block row,靠 `content`/`parent` 串成樹 | 從 page block 遞迴收子 block | block/record 級整批交易 |
| **Confluence (DC)** | 一份 XHTML 字串 `<p>Hello <strong>world</strong></p>` | 直接就是內容(整包讀出) | 需外掛 Synchrony 才有即時同編 |

---

## 5. 重審準則(Re-confirm the Criteria)

看完真實證據後,回頭檢查 §3 的量尺是否仍恰當——結論:**大體成立,但需補一條刻度**。

- **原準則仍有效:** 「FACT/INFERENCE/UNKNOWN 三檔 + 反證搜尋」正是本 spike 最有價值處。真正把三家分開的,不是「誰功能多」,而是「哪些是官方證實、哪些是網路以訛傳訛」——例如若不做反證,幾乎必然會把「Notion=CRDT 協作」「Synchrony=OT」當成 FACT 寫進表裡,而兩者其實都不是一手可證的。
- **需補一條刻度——區分 Server/DC vs Cloud。** 證據顯示 Confluence 的儲存/搜尋事實幾乎全來自**自管的 Server/Data Center 版**文件;**Cloud 版**的 DB/搜尋/物件儲存內部大多未公開。故表格已對 Confluence 明確標註適用版本,避免拿 DC 的 blob 模型去代表 Cloud(Cloud 用 ADF/JSON,不同)。這條刻度是準則接觸證據後才長出來的,比事前空想更強。

---

## 6. 對照準則評估(Evaluate Against the Criteria)

把 §3 量尺套到三系統,逐面向給評:

### 6.1 Google Docs —— 三檔分布最「乾淨」的協作事實,但儲存後端全黑箱

- **協作(D6–D9)全綠:** OT、字元級、中央伺服器仲裁、樂觀本地套用+一次一筆 pending change,全部一手 FACT,且來自同一組 2010 官方三部曲(作者 John Day-Richter,Google Software Engineer),我親自讀 PDF 影像逐字核對(見附錄 note)。`[A1]-[A5]`
- **儲存後端(D2–D4)全黑箱:** Docs 用 Bigtable 還是 Spanner、搜尋用什麼、圖片放哪——**沒有一條 Google 一手來源證實**。網路上「Bigtable 存內容 chunk、Spanner 存 metadata」的說法全出自第三方系統設計站,經反證後降級 UNKNOWN。**評估:協作維度可信度極高;儲存維度誠實留白。**
- **反證結果:** 「Google Docs 用 CRDT 而非 OT」查空——所有比較文皆確認 Docs 用 OT(常拿來與改用 CRDT 的 Figma 對比)。OT 結論穩固。

### 6.2 Confluence —— 儲存事實最扎實(DC),協作演算法名稱是最大坑

- **儲存(D1–D5)在 Server/DC 上最扎實:** 支援的 DB、BODYCONTENT/CONTENT 表、XHTML storage format、附件檔案系統/S3、Lucene/OpenSearch、全版本歷史——每一項都有官方 doc 逐字佐證,是三家中儲存維度一手覆蓋最完整的。
- **最大坑在 D6:** 幾乎所有二手資料都說「Synchrony 用 OT」,但我實際打開 Atlassian 三個相關頁面(admin、server dev、cloud dev),**「operational transformation / OT / CRDT」字樣一次都沒出現**——官方只說「synchronizes data in real time」與「a graph of all edits」。**依準則第 3 條,這只能標 UNKNOWN**,closure recipe:轉錄 Atlassian 官方技術演講「How we built Synchrony」(YouTube,Haymo Meran)或找到官方工程 blog 明文。**評估:儲存維度 A 級;協作演算法維度誠實標 UNKNOWN,不隨大眾寫成 OT。**
- **反證結果:** (a)「Lucene 是唯一引擎」被**部分推翻**——DC 8.9+ 可選 OpenSearch,但 Lucene 仍為預設;(b)「Synchrony 是 OT」——一手查空。

### 6.3 Notion —— 一手素材最豐富(資料模型+分片),但「即時協作演算法」是全網最大誤解

- **儲存(D1–D2, D10)一手素材最豐富:** everything-is-a-block、block=row、480 邏輯分片恆定(32×15 = 96×5 = 480)、workspace ID 分片鍵、VACUUM/TXID 動機、PgBouncer 零停機——數字我親自打開 sharding 一手文逐字核對,全部 FACT。這是三家中擴展性(D10)唯一有一手證據的。
- **D6 是全網最大誤解:** 大量二手(系統設計部落格、廠商頁)斷言「Notion 用 CRDT(或 OT+CRDT hybrid)做百萬並發即時協作」。**經反證:Notion 唯一的一手 CRDT 陳述是 2025-12 離線文,且明確限定在「offline-marked pages」並稱之為「new」**——強烈暗示歷史線上路徑**不是** CRDT。線上官方只說「operations… batched into transactions that are committed (or rejected) by the server as a group」,既未提 OT 也未提 CRDT。**依準則,線上協作演算法只能標 UNKNOWN,行為上推為伺服器仲裁的整批交易(近 record 級 LWW)。**
- **反證結果:** (a)「Notion 用 CRDT 做線上協作」——一手推翻,CRDT 僅離線;(b)「搜尋用 Elasticsearch」——一手查空,只有廠商二手;(c) 分片數字 480/32/96 潛在矛盾——由 data lake 文逐字調和一致(見 §7)。

### 6.4 主線驗證:文件模型 → 協作演算法

三系統正好構成一條「模型決定演算法」的光譜,證據完全支持:

- **op log / 字元序列(Google)→ OT 字元級**:模型把文件拆到字元操作,OT 才有辦法做最細合併。
- **block 樹(Notion)→ record/block 級整批交易**:操作單位是「create/update a single record」,協作精緻度自然停在 block 級。
- **整份 blob(Confluence DC)→ 天生難即時,故外掛 Synchrony**:blob 模型無法原生字元級合併,即時能力是 2016 才用獨立服務補上的分離層。

---

## 7. 直接回答與收尾(Answer & Wrap-up)

**問:三者在架構取捨上各站在什麼位置、為什麼?**

1. **Google Docs —— 為「即時字元級協作」不惜重寫儲存模型。** 它把文件本體改成 append-only op log(revision log),吃下 OT 的全部複雜度,換來三家中唯一由官方證實的字元級即時協作。代價/風險是:協作邏輯集中在中央伺服器仲裁,且其**儲存後端、搜尋、傳輸幾乎全不公開**——你能確知「它怎麼合併字元」,卻不能確知「它把 bytes 存哪」。取捨定位:**協作精緻度優先,儲存細節黑箱。**

2. **Notion —— 為「靈活 block 工作區 + 巨量水平擴展」選了 block-per-row + Postgres 分片。** 「everything is a block、每 block 一列」讓產品極度靈活(文字、資料庫、頁面同構),但也讓它在 200B+ 列規模撞上 Postgres 的 VACUUM/TXID 天花板,於是有了公開的 480 邏輯分片史。協作上,它**沒有**走 Google 式字元級 OT;線上是整批交易的伺服器仲裁,CRDT 遲至 2025-12 才為**離線**補上。取捨定位:**模型靈活度與擴展性優先,即時協作精緻度讓位(近 block/record 級)。**

3. **Confluence —— 為「企業自管、相容既有 RDBMS」把文件當 blob 存。** Server/DC 版把整頁當一份 XHTML storage format 字串存進 PostgreSQL/Oracle/SQL Server 等傳統關聯式資料庫,版本是全快照,搜尋是自管 Lucene(可選 OpenSearch),附件走檔案系統或 S3。即時協作是後來(Confluence 6.0, 2016)用**獨立服務 Synchrony** 外掛補上的能力層,走 WebSocket。取捨定位:**企業可自管、與傳統 DB/運維相容優先,即時協作是後補的分離層;其合併演算法官方甚至未公開名稱。**

**一圖總結取捨象限:**
- 協作即時/精緻度: **Google Docs(字元級,強) > Confluence(富文字即時,中) ≈ Notion 線上(block 級整批,中/弱)**;離線: **Notion(CRDT)** 反而最新最強。
- 儲存/擴展一手透明度: **Notion(分片全公開)> Confluence(DC 儲存全公開)> Google Docs(後端全黑箱)**。
- 三者無「誰全面最好」,只有**沿各自產品定位(消費級即時 / 企業自管 wiki / 靈活擴展工作區)做出的不同取捨**。

**可行動收尾(給讀者):**
- 若要引用「Notion 用 CRDT」或「Confluence Synchrony 用 OT」——**別引**,兩者非一手可證;要嘛標 UNKNOWN,要嘛先關掉下方 closure recipe。
- 若後續要把某個 UNKNOWN 關掉,最高價值三條:(1) 轉錄「How we built Synchrony」演講以定 Confluence 演算法;(2) 等 Notion 承諾的 Search/RAG infra 一手文以定其搜尋引擎;(3) 抓 live Docs 網路流量(`/bind` 端點)以定 Google Docs 傳輸協定。皆為 read-only。

---

## 附錄 — Evidence Appendix(取證下沉:URL + 逐字引文 + 日期)

> 取證日期一律 **2026-07-18**。本 spike 親自打開並讀過的一手頁面清單如下,body 以 `[A#]` 回指。

### Google Docs(2010 官方三部曲,作者 John Day-Richter, Google Software Engineer)

> **來源可信度 note:** 三篇 2010 Google Drive Blog 原文在 `drive.googleblog.com` 以 JS 殼呈現,WebFetch 取不到 body。我改讀 University of Washington Interactive Data Lab 託管的**逐字重製 PDF**,並**親自以 Read 工具讀取 PDF 頁面影像逐字核對**:PDF 內含「Google Drive Blog」刊頭、原標題、日期(2010-09-21 / 09-22)、署名「Posted by: John Day-Richter, Software Engineer」,確認為原文忠實重製。PDF URL:`https://idl.uw.edu/future-scholarly-communication/files/2010-GoogleDocs-OT.pdf`。原始 canonical:`https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html`(及 `_21`、`_22` 兩續篇)。

- **[A1]** 文件=時序改動、伺服器持有完整 log + 當前狀態 — PDF 同上 — *"In the new editor, a document is stored as a series of chronological changes. A change might be something like {InsertText 'T' @10}."* / *"A fundamental difference between the new editor and the old one is that instead of computing the changes by comparing document versions, we now compute the versions by playing forward the history of changes."*
- **[A2]** 三種操作 + append + replay;否決段落鎖 — 同上 — *"In Google documents, all edits boil down to three basic types of changes: inserting text, deleting text, and applying styles to a range of text. We save your document as a revision log consisting of a list of these changes… they are appending their change to the end of the revision log. To display a document, we replay the revision log from the beginning."* / *"you could lock paragraphs so that only one editor was ever allowed to type in a single paragraph at a given time. But locking paragraphs isn't a great solution…"*
- **[A3]** OT 為演算法 + 轉換直覺 — 同上 — *"The algorithm that we use to handle these shifts is called operational transformation (OT). If OT is implemented correctly, it guarantees that once all editors have received all changes, everyone will be looking at the same version of the document."*
- **[A4]** 字元級 — 同上 — *"Together, these technologies create the character-by-character collaboration in Google Docs."*
- **[A5]** client-server、樂觀本地套用、每個 editor transform 收到的改動 — 同上 — *"Collaboration in Google Docs consists of sending changes from one editor to the server, and then to the other editors. Each editor transforms incoming changes so that they make sense relative to the local version of the document."*
- **[A6]** (INFERENCE 佐證)Colossus 為 Google 產品普遍依賴 — `https://cloud.google.com/blog/products/storage-data-transfer/how-colossus-optimizes-data-placement-for-performance` — *"From YouTube and Gmail to BigQuery and Cloud Storage, almost all of Google's products depend on Colossus, our foundational distributed storage system."*(未點名 Docs,故 Docs 落於 Colossus 為 INFERENCE。)

### Notion(全部 notion.com/blog 一手,親自打開)

- **[A7]** block 模型 / 屬性 / content 陣列 / parent / loadPageChunk — `https://www.notion.com/blog/data-model-behind-notion`(canonical;`/data-model-behind-notions-flexibility` 會 404)— *"Everything you see in Notion is a block. Text, images, lists, a row in a database, even pages themselves—these are all blocks"* / *"an array (or ordered set) of block IDs representing the content inside this block"* / *"the block ID of the block's parent"* / *"loadPageChunk—it descends from a starting point (likely the block ID of a page block) down the content tree, and returns the blocks"*。頁內**未出現** "operational transformation" 或 "CRDT"(親自核對)。
- **[A8]** 分片:480/32、VACUUM/TXID、五分鐘停機 — `https://www.notion.com/blog/sharding-postgres-at-notion` — *"480 logical shards evenly distributed across 32 physical databases"* / *"Physical database (32 total) → Logical shard, represented as a Postgres schema (15 per database, 480 total) → block table (1 per logical shard, 480 total)"* / *"Each block belongs to exactly one workspace, we used the workspace ID as the partition key."* / *"the Postgres VACUUM process began to stall consistently, preventing the database from reclaiming disk space from dead tuples"* / *"transaction ID (TXID) wraparound… Postgres would stop processing all writes"* / *"we took Notion down for five minutes of scheduled maintenance."*
- **[A9]** re-shard 32→96、每 shard schema 15→5 — `https://www.notion.com/blog/the-great-re-shard` — *"tripling the number of instances in our fleet from 32 to 96 machines"* / *"we went from having 15 schemas per shard to 5"*(發佈 2023-07-17)。
- **[A10]** PostgreSQL source-of-truth / 整批交易 / S3 data lake / Debezium-Kafka-Hudi-Spark / 480 恆定調和 / 200B blocks — `https://www.notion.com/blog/building-and-scaling-notions-data-lake` & `[A7]` — *"operations are batched into transactions that are committed (or rejected) by the server as a group"* `[A7]`;*"SaveTransaction's main job is to get your data into our source-of-truth databases, which store all block data"* `[A7]`;data lake 文:*"horizontally sharding our Postgres database into 32 physical instances"* / *"increasing the number of physical instances to 96, with five logical shards per instance. Thus we maintained a total of 480 logical shards"* / *"use S3 as a data repository and lake to store all raw and processed data"* / *"Notion's in-house data lake is built on Debezium CDC connector, Kafka, Hudi, Spark, and S3"* / *"more than 20 billion block rows… grown to more than two hundred billion blocks"* / (搜尋)*"Stay tuned for a detailed post on our Search and AI Embedding RAG Infra…"*(→ 搜尋引擎 UNKNOWN)。
- **[A11]** CRDT 僅離線 / SQLite 本地快取 / channels 訂閱 — `https://www.notion.com/blog/how-we-made-notion-available-offline`(2025-12-11)— *"pages that are marked as available offline are dynamically migrated to our new CRDT data model for conflict-resolution"* / *"For years, Notion has used SQLite to cache records locally and speed up page loads."* / *"Notion's unique block architecture meant we had to solve several challenging problems around reference tracking, background syncing, and rich-text conflict resolution."* / *"Clients subscribe to these channels for each of their offline pages, and when they receive a message, they fetch the latest changes for the page."* 頁內**未出現** "operational transformation"(親自核對)。

### Confluence(親自打開;標明版本 DC 10.2)

- **[A12]** storage format 是 XHTML/XML — `https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html` — *"This page describes the XHTML-based format that Confluence uses to store the content of pages, page templates, blueprints, blog posts, and comments."* / *"Technically, it's XML, since the storage format doesn't fully comply with the XHTML definition."*
- **[A13]** BODYCONTENT / CONTENT 資料模型 — `https://confluence.atlassian.com/doc/confluence-data-model-127369837.html` — BODYCONTENT: *"The content of Confluence pages. No version information or other metadata is stored here. That is all in the content table."*;CONTENT: *"A persistence table for the ContentEntityObject class of objects."*
- **[A14]** 全版本 + 還原=複製 — `https://confluence.atlassian.com/doc/page-history-and-page-comparison-views-139379.html` — *"Confluence tracks the history of changes to each page by creating a new version of the page each time it's modified."* / *"restoring an older version creates a copy of that version."* / *"if you restore version 39, Confluence will create a copy of version 39 and the copy will become the new, current version."*
- **[A15]** 附件:檔案系統預設 / S3(8.1)/ DB-WebDAV 停用 — `https://confluence.atlassian.com/doc/attachment-storage-configuration-166876.html` — *"By default, Confluence stores attachments in the attachments directory within the configured Confluence home folder."* / *"Starting from Confluence 8.1, you can also store your attachment data on Amazon S3 object storage."* / *"These storage methods are no longer supported."*(指 DB/WebDAV)
- **[A16]** Synchrony / change log graph / WebSocket→XHR — `https://confluence.atlassian.com/doc/administering-collaborative-editing-858772086.html` — *"Collaborative editing is powered by Synchrony which synchronizes data in real time."* / *"Each page and blog post has its own Synchrony change log, which contains a graph of all edits to that page or blog post."* / *"For best results, your load balancer and proxies should allow WebSocket connections. If your users cannot get a WebSocket connection, Confluence will fall back to an XML HTTP Request (XHR)…"* **頁內未出現 "operational transformation"/"OT"/"CRDT"(親自核對)→ D6 演算法 UNKNOWN。**
- **[A17]** Lucene 預設 + OpenSearch opt-in — `https://confluence.atlassian.com/doc/content-index-administration-148844.html`(*"By default, Confluence uses Lucene for indexing."* / index 落於 `<home-directory>/index`)& `https://confluence.atlassian.com/doc/configuring-opensearch-for-confluence-1387594125.html`(*"Confluence Data Center now offers an alternative search engine as an opt-in feature — OpenSearch"*;預設 `search.platform=lucene`)。
- **[A18]** Cloud:WebSocket 對 Synchrony 開 session + 定期存 snapshot;Cloud 編輯器即時同編 — `https://developer.atlassian.com/cloud/confluence/collaborative-editing/`(*"A web socket session is opened to Synchrony using the provided JWT and the contentId of the page being edited"* / *"Confluence will continue storing a snapshot of the most recent state of the document being edited at regular intervals"*)& `https://support.atlassian.com/confluence-cloud/docs/learn-about-the-atlassian-cloud-editor/`(*"The cloud editor allows you and other users to edit a page or live doc simultaneously…"*)。

### 反證搜尋(Adversarial)彙整

| # | 反證查詢 | 結果 |
|---|---|---|
| R1 | Google Docs 用 CRDT 而非 OT? | **查空**;一手/比較文皆確認 OT。OT 結論穩固。 |
| R2 | Google Docs 後端是 Bigtable/Spanner? | **一手查空**;僅二手臆測 → D2 UNKNOWN。 |
| R3 | Confluence 搜尋 Lucene vs Elasticsearch/OpenSearch? | **部分推翻**:Lucene 仍預設,但 DC 8.9+ 可選 OpenSearch。 |
| R4 | Synchrony 是 OT 還是 CRDT? | **一手查空**;三個 Atlassian 頁皆未提演算法名 → D6 UNKNOWN。 |
| R5 | Notion 線上用 CRDT/OT? | **一手推翻**:CRDT 僅離線(2025-12「new」);線上僅「整批交易」。 |
| R6 | Notion 搜尋用 Elasticsearch? | **一手查空**;僅廠商二手 → D4 UNKNOWN。 |
| R7 | Notion 分片數字 480/32/96 是否矛盾? | **調和一致**:32×15 = 96×5 = 480(data lake 文逐字確認)。 |

### UNKNOWN 清單與 closure recipe(read-only)

- **Google Docs D2/D3/D4/D8/D9(後端 DB、圖片儲存、搜尋引擎、傳輸協定、游標 UI):** 查 Google Research/OSDI/SOSP 有無 Docs/Workspace 一手論文;抓 live `docs.google.com` 網路流量看 `/bind`、`/save` 端點判傳輸;找 Workspace 工程演講。
- **Confluence D6(Synchrony 演算法):** 轉錄官方演講「How we built Synchrony」(YouTube, Haymo Meran)或找官方工程 blog 明文;在此之前 OT 只能當二手 inference。
- **Confluence Cloud 儲存/搜尋/物件儲存:** 找 developer.atlassian.com 的 Media API / Cloud 資料架構文(多半未公開至列級)。
- **Notion D4(搜尋引擎)/ D2 欄位編碼 / 伺服器快取 / 線上傳輸:** 等官方承諾的 Search & RAG infra 一手文;或官方 schema/infra 演講。

---

*Facts / Inferences / Unknowns 已於每格標示;凡標 UNKNOWN 者皆附 closure recipe。本 spike 未修改任何原始碼、未 commit、未開 PR。*
