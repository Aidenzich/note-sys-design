# 線上協作文件編輯器三系統深度設計比較:Google Docs vs Confluence vs Notion

> 來源:2026-07-19 的 read-only 調查(獨立重新取證)。所有外部主張皆附 URL + 逐字引文 + 擷取日期。
> 姊妹篇:[Google Docs vs Confluence 雙系統版](./README.md)(較精簡的入門版)。
> 證據等級標記:**FACT**(一手來源逐字佐證)/ **INFERENCE**(推論,弱佐證)/ **UNKNOWN**(官方未揭露,附 closure recipe)。
> 承重的原始證據(逐字引文、來源清單)下沉到 Appendix,本文以循序數字引用 [1]、[2]… 連結文末 References(引用只承載出處,句子不讀它也通順)。

---

## TL;DR

三個系統看起來都是「多人一起編一份文件」,但**文件模型(document model)決定了一切下游取捨**——存哪種資料庫、能不能字元級即時協作、能不能離線、怎麼分片。一句話定位:

- **Google Docs = 字元級 op log + Operational Transformation(OT)的即時協作標竿。** 文件本質是一串「變更操作(operation)」的 append-only 日誌;協作用 OT(Jupiter client-server 模型)把並發操作互相變換到收斂;可見到「逐字」即時更新與彩色游標;支援離線。代價是後端儲存細節官方幾乎全不公開(UNKNOWN)。[1](#ref-1)[2](#ref-2)[3](#ref-3)
- **Confluence = 傳統關聯式資料庫(RDBMS)存整段 XHTML 字串 + Synchrony 中央權威 rebase 協作。** 頁面正文是一大段 XHTML/XML 字串,存在關聯式資料庫的 `BODYCONTENT` 文字欄位;協作靠 Synchrony 微服務(WebSocket、telepointer 游標),機制是**伺服器仲裁的 edit-script/rebase**(OT 家族,**明確不是 CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別 —— 免中央仲裁即可合併的資料結構)**),但 Atlassian 官方從不用「OT」或「CRDT」這兩個詞自稱。全文檢索用 Lucene。不支援離線編輯。[4](#ref-4)[5](#ref-5)[6](#ref-6)[7](#ref-7)
- **Notion = 一切皆 block、以指標串成樹、存分片 Postgres;線上是 operation→transaction、離線才明確用 CRDT。** 每個東西(文字、圖、頁面本身)都是一筆 block 記錄,頁面是靠 `content`(子 block id 陣列)與 `parent` 指標組出來的樹;編輯是「operation 批次成 transaction 由伺服器整組 commit/reject」;**離線頁面**才被明確遷移到 **CRDT** 做衝突解決。資料庫是 workspace 分片的 Postgres(480 邏輯分片、32→96 台實體機)。搜尋用 Elasticsearch。[8](#ref-8)[9](#ref-9)[10](#ref-10)[11](#ref-11)

**最大 caveat:** 三家「揭露程度」極不對稱。Google 公開了協作演算法理論(OT/Jupiter)卻幾乎不談儲存;Notion 公開了儲存/分片一手好料卻**極少談線上協作演算法**(除了離線 CRDT);Confluence 的協作演算法官方**從未指名**,只能靠一份 Atlassian 專利與 ProseMirror 編輯器旁證。凡官方未公開者本文一律標 UNKNOWN,不為了三欄對稱而美化任何一格。

---

## 1. Introduction

### 1.1 這份 spike 的目的(Purpose)

把三個代表性的線上協作文件/wiki 系統攤開做**深度設計解析與比較**,像一份系統設計面試等級的架構解讀:不是列功能,而是**回答「三者在架構取捨上各站在什麼位置、為什麼」**,並讓讀者理解「文件模型 → 協作演算法 → 儲存/擴展」這條因果鏈。三個對象各自代表一種典型路線:

- **Google Docs** — 消費級即時協作標竿(character-level real-time)。
- **Confluence** — 企業 wiki(RDBMS-backed、macro 生態、後進場的即時協作)。
- **Notion** — block 式工作區(把「文件」與「資料庫」統一成 block)。

### 1.2 要回答的具體問題(Questions)

1. 每個系統的**文件模型**是什麼(整份 blob / 操作日誌 op log / block 樹 / 字元序列 / XHTML 字串)?存進哪種資料庫?文字與二進位如何分離?全文檢索用什麼?
2. **多人同編**用什麼演算法/協定(OT、CRDT、鎖、save-based、中央 rebase)?粒度是字元、區塊還是整頁?即時傳輸層是什麼?前端如何呈現多人同時編輯、衝突怎麼解?
3. **版本歷史、離線、權限、擴展性(分片)**各是什麼取捨?最終:**文件模型如何決定協作演算法與擴展路線?**

### 1.3 調查涵蓋的面向(Dimensions)

儲存(資料庫、文件模型、快取、物件儲存、全文檢索)、協作(演算法、粒度、傳輸、臨場感/游標、衝突解決)、以及其他(版本歷史、離線、權限、擴展/分片)。完整量尺見 §3。

---

## 2. Prerequisites

以下是後文論證會反覆倚賴的名詞與**結構性概念**。承重的結構性概念都附最小可寫實例(fenced code block),先教一次,後面才用。

### 2.1 文件模型的四種形狀(Document model)

「文件在資料庫裡長什麼樣」有幾種截然不同的做法,這是全篇最重要的分野:

**(a) 整份 blob(whole-document blob)** — 把整份文件當一個大字串/大物件存一欄。編輯=覆寫整段。最簡單,但兩人同時改就會互相蓋掉。Confluence 的頁面正文接近這型(一段 XHTML 字串)。

**(b) 操作日誌 / op log(operation log)** — 不存「現在的內容」,而存「一連串變更操作」,current content = 從空白重放(replay)所有 op。這是即時協作的關鍵地基。以打「Hi」再退格刪掉「i」為例,op log 長這樣:

```jsonc
// 一份 op log:三筆 append-only 操作(pos=游標位置, rev=伺服器序號)
{ "op": "insert", "pos": 0, "char": "H", "rev": 1 }
{ "op": "insert", "pos": 1, "char": "i", "rev": 2 }
{ "op": "delete", "pos": 1, "len": 1,     "rev": 3 }   // 退格,刪掉 "i"
// current content = 從 "" 依序重放 => "H"
```

好處:每個 keystroke 是一筆小 op,不必每次重寫整份文件;而且「操作」正是 OT/CRDT 能互相變換的單位。Google Docs 屬此型(官方專利稱為 "changes log")。實務上會週期性存 snapshot,避免每次都從頭重放。[1](#ref-1)

**(c) block 樹(block tree)** — 文件不是一段文字,而是一堆**獨立的 block 記錄**,每筆 block 用指標(id 陣列)指向子 block,組成一棵樹。Notion 屬此型。一個「有一段文字 + 一個待辦」的頁面長這樣:

```jsonc
// 三筆 block 記錄,靠 content(向下指標)+ parent(向上指標)串成樹
{ "id": "page-1", "type": "page",
  "content": ["blk-A", "blk-B"], "parent": "workspace" }              // 頁面本身也是 block
{ "id": "blk-A",  "type": "paragraph",
  "properties": { "title": [["Hello"]] }, "content": [], "parent": "page-1" }
{ "id": "blk-B",  "type": "to_do",
  "properties": { "title": [["buy milk"]], "checked": [["No"]] },
  "content": [], "parent": "page-1" }
// 「渲染 page-1」= 解析 page-1.content -> 抓 blk-A、blk-B -> 各自再抓其 content...(樹走訪)
```

代價:渲染一頁要遞迴解析一堆 id(典型 N+1 讀取);好處:move/reparent/巢狀天然就是 block 級操作。[8](#ref-8)[12](#ref-12)

**(d) 字元序列(character sequence)** — 把文件看成一串有身分的字元,每個字元可被定位、變換。這是 OT/CRDT 在文字層真正操作的抽象;Google Docs 的「item」(每個字元、每個標籤都是一個 item)就是這層。[13](#ref-13)

### 2.2 兩種並發控制演算法:OT 與 CRDT

多人同時改同一份文件,核心難題是「兩個並發操作怎麼合併成大家一致的結果」。兩大流派:

**OT(Operational Transformation,操作變換)** — 有一個**中央伺服器**當唯一定序點。當你的操作跟別人並發時,伺服器/客戶端把操作**互相變換(transform)**位置後再套用,使大家收斂到同一狀態。範例:文件是 `"AC"`,User1 在位置 1 插 `"B"`(→`"ABC"`),User2 同時刪位置 1 的 `"C"`。若不變換,兩邊套用順序不同會得到不同結果;OT 把「後到的操作」相對「先到的操作」調整索引:

```text
初始 "AC"                    (索引: A=0, C=1)
User1: insert(pos=1,"B")      User2: delete(pos=1,"C")   ← 兩者並發
伺服器定序後 transform:
  先 U1 -> "ABC";  再把 U2 的 delete(pos=1) 變換成 delete(pos=2)  ← 因為 U1 在前面插了一字
結果兩邊都收斂到 "AB"
```

OT 需要一個中央權威決定順序;Google Docs、以及(OT 家族的)Confluence Synchrony 屬此路線。

**CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別)** — 不需要中央定序。每個字元/元素帶一個**全域唯一、可排序的身分**(例如夾在兩個鄰居之間的分數位置 id),合併時取聯集、按 id 排序即得一致結果,順序無關。適合離線/去中心:兩份離線副本各改各的,重連後直接 merge 不需伺服器仲裁。Notion 的**離線**衝突解決明確採 CRDT。[10](#ref-10)

> 一句話對比:**OT = 中央伺服器把並發操作互相變換;CRDT = 每個元素自帶身分,合併與順序無關、免中央仲裁。**

### 2.3 其他名詞(一行即懂,不需 code block)

- **XHTML 儲存格式(storage format)** — Confluence 把頁面正文存成一段類 XHTML 的 XML 字串,巨集(macro,可嵌入頁面的功能小程式)用自訂 `ac:` 標籤內嵌其中,例:`<ac:structured-macro ac:name="info">…</ac:structured-macro>`;整段字串放在資料庫一個文字欄位裡。[4](#ref-4)[5](#ref-5)
- **telepointer(遠端游標/臨場感 presence)** — 在畫面上顯示「別人的游標/選取範圍」,讓你看到協作者此刻在哪、在改什麼。[6](#ref-6)
- **Lucene** — Apache 的全文檢索(full-text search)函式庫,把文件內容建成倒排索引(inverted index)存在磁碟上供關鍵字查詢;Confluence 內建用它。[14](#ref-14)
- **sharding(分片)** — 資料量大到單一資料庫撐不住時,把資料水平切成多份放到多台機器。Notion 依 workspace id 分片 Postgres。[11](#ref-11)
- **operation / transaction** — Notion 把 UI 動作表達成 operation(建立/更新單筆記錄),再把多個 operation 批次成一個 transaction 由伺服器整組 commit 或 reject。[9](#ref-9)
- **WebSocket** — 瀏覽器與伺服器間的全雙工長連線,適合把別人的即時變更「推」給你;Confluence Synchrony、Notion 即時通道都用推送式連線。[15](#ref-15)

---

## 3. Methodology

「這類編輯系統該怎麼比」的 rubric。以下每個面向都給出**判準**(要看什麼、什麼算好),先立在證據之前;§5 會在看過真實證據後重新檢視這把尺。

**A. 儲存與文件模型**
1. **文件模型** — blob / op log / block 樹 / XHTML 字串;判準:是否天生支援細粒度並發(op log/block 佳,整段 blob 差)。
2. **底層資料庫** — RDBMS / 分片 Postgres / 專有;判準:能否隨資料量水平擴展。
3. **二進位分離** — 文字與圖片/附件是否分離到物件儲存(S3 等);判準:大二進位不塞進主資料庫。
4. **快取** — 有哪些快取層。
5. **全文檢索** — 引擎、索引如何建/更新。

**B. 協作**
6. **並發演算法** — OT / CRDT / 鎖 / save-based-LWW(last-write-wins,後寫入者直接覆蓋前者)/ 中央 rebase;判準:能否無鎖即時多人同編。
7. **協作粒度** — 字元 / block / 整頁;判準:越細,並發衝突越少、體驗越即時。
8. **即時傳輸層** — WebSocket / long-poll / SSE(Server-Sent Events,伺服器單向推播的長連線)/ 純 REST。
9. **臨場感呈現** — 是否有 presence/游標、是否逐字即時、還是存檔後才見。
10. **衝突解決** — transform / merge / 覆寫(overwrite)/ abort。

**C. 其他**
11. **版本歷史** — 粒度與儲存方式(整份快照 vs diff)。
12. **離線編輯** — 是否支援、如何同步。
13. **權限模型** — 全域/空間/頁面、繼承規則。
14. **擴展性/分片** — 如何水平擴展。
15. **貫穿主題** — 文件模型如何決定了協作演算法與擴展路線(這是最終要回答的因果)。

**評分表達:** 每格標 FACT / INFERENCE / UNKNOWN。若官方未揭露,寧可寫 UNKNOWN + closure recipe,不臆測、不為對稱補齊。

---

## 4. Findings

本節於 finding 高度陳述;逐字引文與來源清單見 Appendix A,反證日誌見 Appendix B。

### 4.1 Google Docs — 字元級 op log × OT/Jupiter

**文件模型是 op log(FACT)。** Google 專利明述文件同步/儲存單位是 append-only 的 "changes log",且「文件資料的真實來源是伺服器」;維基百科(引一手鏈)也稱「the document is stored as a list of changes」。這正是 §2.1(b) 的形狀。[16](#ref-16)[1](#ref-1)

**協作演算法是 OT,源自 Jupiter client-server 模型(FACT)。** Google 自 published 的 Wave OT 白皮書逐字寫明「Google Wave uses the Operational Transformation (OT) framework」、且「the starting point for Wave OT was … the Jupiter collaboration system」、並「requires the client to wait for acknowledgement from the server before sending more operations」——即**中央伺服器定序、一次一個在途操作**。Wave 是 Docs 現代即時引擎的直系理論來源。粒度是**字元/item 級**(「every character, start tag or end tag … is called an item」),前端因此能「逐字」看到他人編輯,並以每人專屬顏色的游標呈現臨場感。[2](#ref-2)[13](#ref-13)[3](#ref-3)[17](#ref-17)

**離線支援(FACT)。** 官方 Help 說明離線可建立/檢視/編輯,行動 App 原生支援;專利描述離線可持續編輯、重連後以同一套 change-log/OT 對帳。版本歷史自動保存、可檢視可回復、每位作者以顏色區分;但 Drive API 明言「頻繁編輯的 Google Docs 版本清單可能不完整」——暗示底層細粒度 op log 會被壓縮/合併成較粗的具名版本。[18](#ref-18)[19](#ref-19)

**儲存後端幾乎全 UNKNOWN。** 「Docs 用 Bigtable/Spanner/Colossus」這類敘述**只**出現在會自我 hedge(「likely」)的二手系統設計部落格,無任何 Google 一手佐證;快取層、二進位/圖片分離、全文檢索引擎與索引管線同樣未公開。吞吐/延遲數字(如「200k ops/sec」「<200ms」)亦僅見二手,**不得當作 Google 官方數據**。closure recipe 見 Appendix C。[20](#ref-20)

### 4.2 Confluence — RDBMS 存 XHTML 字串 × Synchrony 中央 rebase

**文件模型是「一段 XHTML 字串存在 RDBMS 文字欄位」(FACT)。** 官方說明頁面正文用 XHTML-based 儲存格式(「Technically, it's XML」),巨集以 `ac:` 標籤內嵌;此正文字串存在 `BODYCONTENT` 表(內容 metadata 在 `CONTENT` 表)。支援的資料庫是 PostgreSQL / MySQL / Oracle / SQL Server(H2 僅測試用)。這是 §2.1(a) 近似「整段 blob」的路線——渲染時還要處理內嵌巨集。[21](#ref-21)[4](#ref-4)[5](#ref-5)

**二進位分離(FACT)。** 附件預設存本機檔案系統的 `attachments/` 目錄;Confluence 8.1 起(Data Center)支援 S3 物件儲存(prefix `/confluence/attachments/v4`,且 S3「僅供附件資料」);舊版曾存資料庫/WebDAV 但已不支援。**文字進 RDBMS、二進位進檔案系統/S3**,分離明確。[22](#ref-22)[23](#ref-23)

**全文檢索用 Lucene(FACT)。** 官方:「By default, Confluence uses Lucene for indexing」;索引檔在 `<home>/index`;變更進佇列、背景每約 5 秒批次處理;叢集中每節點各存一份完整索引,靠 journal service 同步。快取在叢集用 Hazelcast(EhCache 為歷史單機層,列 INFERENCE)。[14](#ref-14)[24](#ref-24)

**協作:Synchrony 微服務,WebSocket + telepointer(FACT);演算法為 OT 家族的伺服器仲裁 edit-script/rebase,但官方從不指名(見反證)。** 官方:「Synchrony is a micro service that allows the synchronization of arbitrary data models in real time」、「will act as the source of truth for page content」、定期存 snapshot;傳輸用 WebSocket(JWT + contentId,叢集需 session affinity + WebSocket 的負載平衡);支援 telepointer 遠端選取;斷線變更緩衝、重連合併。單一 shared draft、持續存檔,但**未發佈的變更不另存版本**。發佈時 `conflictPolicy` 目前僅支援 `abort`——中央權威直接拒衝突,而非 CRDT 自動合併。[15](#ref-15)[6](#ref-6)[25](#ref-25)[7](#ref-7)

> **關於演算法的精確定性(這是 Confluence 最需要誠實的一格):** Atlassian 的**文件/部落格從未使用「operational transformation / OT / CRDT」任何一詞**自稱。唯一描述機制的一手 Atlassian 產物是**專利 US9355083B1**(發明人 Haymo Meran、Tobias Steiner,即 Synchrony 團隊,來自 Atlassian 收購的 Wikidocs),描述的是**伺服器產生並轉發 "edit script"** 的**中央仲裁模型**——屬 OT 家族、**明確不是 CRDT**,但專利本身也不自稱「OT」。故「Confluence 用 OT」是**二手詮釋**,本文定性為「**伺服器仲裁的 edit-script/rebase(OT 家族),非 CRDT,官方未指名**」。[26](#ref-26)[27](#ref-27)

**版本歷史為整份快照(FACT/INFERENCE)。** 官方:每次修改建一個新版本,還原=複製該舊版本向前,刪版本不重新編號——用字是「creates a copy」,指向**整份快照**而非 diff 儲存(二手宣稱 diff 儲存與此一手用語矛盾,列 UNKNOWN)。權限三層(全域→空間→頁面限制),view 限制向下繼承、edit 限制不繼承,限制只能收窄不能放寬。**不支援離線編輯**(Synchrony 為 source of truth 且需 live WebSocket;無任何官方離線頁面)——列 INFERENCE。擴展性:Data Center 為 active-active 叢集,共享 DB、共享 home 放附件、local home 放索引/快取。[28](#ref-28)[29](#ref-29)[30](#ref-30)

### 4.3 Notion — 一切皆 block × 分片 Postgres × 線上 operation/離線 CRDT

**文件模型是 block 樹,一切皆 block(FACT)。** 官方 engineering blog:「Everything you see in Notion is a block … even pages themselves」;每個 block 有 ID / Properties / Type / **Content(子 block id 的有序陣列,即向下指標)** / **Parent(向上指標,僅供權限用)**;block 可無限巢狀,頁面就是靠這些指標組出的樹(§2.1(c))。[8](#ref-8)[12](#ref-12)

**線上編輯是 operation→transaction(FACT);離線才明確用 CRDT(FACT)。** 官方:UI 動作「are expressed as operations that create or update a single record」、「operations are batched into transactions that are committed (or rejected) by the server as a group」——**伺服器整組 commit/reject**,是中央定序的 block/record 級模型,**非字元級 OT**。另一篇官方離線文明述:「pages that are marked as available offline are dynamically migrated to our new **CRDT** data model for conflict-resolution」,並點名要解決「reference tracking、background syncing、rich-text conflict resolution」。本機快取/離線儲存用 **SQLite**;重連以 `lastDownloadedTimestamp` 對比伺服器 `lastUpdatedTime` 只抓較新頁面;即時更新走「訂閱 channel、收到訊息就抓最新變更」的推送模型。[9](#ref-9)[10](#ref-10)[31](#ref-31)[32](#ref-32)

> **Notion 協作最需誠實的一格:** 官方**只**明確了兩件事——線上是 operation/transaction、離線是 CRDT。**線上並發的精確合併語意(是否 block 級 LWW?字元級?)官方未揭露**,二手流傳的「OT+CRDT hybrid / 純 CRDT / LWW」互相矛盾且無一手佐證,一律**不採信、列 UNKNOWN**。

**資料庫是 workspace 分片的 Postgres(FACT,罕見的一手好料)。** 2020 從單體 Postgres 轉分片:**480 個邏輯分片(每個是一個 Postgres schema)平均分布於 32 台實體機**,依 **workspace id** 分片(「each block belongs to exactly one workspace」)。2023「The Great Re-shard」把 32→96 台、零停機(用 Postgres logical replication,同步時間從 3 天降到 12 小時,PgBouncer 在棧內)。分析用資料湖:Postgres→(Debezium CDC(Change Data Capture,變更資料擷取 —— 把資料庫的每筆變更即時串流出去))→Kafka→(Apache Hudi/Spark)→**S3**,下游含 **Elasticsearch**、向量庫、KV;規模「兩千億以上 blocks、數百 TB,每 6–12 個月翻倍」。[11](#ref-11)[33](#ref-33)[34](#ref-34)

**全文檢索用 Elasticsearch(FACT 為下游存在;live search 細節 INFERENCE)。** 資料湖官方文把 Elasticsearch 列為 product-facing 下游存在(一手);「跑在 EC2、從 managed 遷移」等細節僅第三方 vendor case study(二手,附 caveat)。物件儲存(圖片/檔案)= S3 屬 INFERENCE(資料湖用 S3 為一手,但無「圖片存 S3」的逐字一手)。權限沿 parent 指標鏈評估,分析時「has to be constructed on the fly via expensive tree traversal computation」——block 樹導致 N+1 讀取與權限樹走訪成本(FACT)。頁面歷史的儲存格式屬 UNKNOWN。[34](#ref-34)[35](#ref-35)[36](#ref-36)

### 4.4 三欄比較表(核心產物)

> 每格標 FACT / INF(inference)/ UNK(unknown)(末列「揭露程度(meta)」為對三家揭露不對稱的整體評註,不逐格標記)。錨點回指 Appendix A。

| 面向 | **Google Docs** | **Confluence** | **Notion** |
|---|---|---|---|
| **文件模型** | 操作日誌 op log(專利稱 "changes log");current=重放操作 · **FACT**[1](#ref-1) | 一段 XHTML/XML 字串(近整段 blob),巨集內嵌 · **FACT**[4](#ref-4)[5](#ref-5) | block 樹:一切皆 block,`content`/`parent` 指標串成樹 · **FACT**[8](#ref-8) |
| **底層資料庫** | 未公開(二手猜 Bigtable/Spanner)· **UNK**[20](#ref-20) | RDBMS:PostgreSQL/MySQL/Oracle/SQL Server · **FACT**[21](#ref-21) | 分片 Postgres:480 邏輯分片 / 32→96 實體機,依 workspace 分片 · **FACT**[11](#ref-11)[33](#ref-33) |
| **二進位分離 / 物件儲存** | 未公開(推測 Colossus/Drive)· **UNK/INF**[20](#ref-20) | 附件→本機檔案系統,或 S3(8.1+,DC);S3 僅附件 · **FACT**[22](#ref-22)[23](#ref-23) | 資料湖用 S3(一手);圖片/檔案存 S3 · **INF**[34](#ref-34) |
| **快取** | 未公開 · **UNK** | Hazelcast(叢集);EhCache(歷史單機)· **FACT/INF**[24](#ref-24) | 用戶端 SQLite 本機快取;伺服器端快取層未公開 · **FACT/UNK**[31](#ref-31) |
| **全文檢索** | 未公開 · **UNK**[20](#ref-20) | Lucene,索引在磁碟,背景每 ~5s 批次;每節點各一份 · **FACT**[14](#ref-14) | Elasticsearch(下游 product-facing)· **FACT**;live 細節 **INF**[34](#ref-34)[35](#ref-35) |
| **並發演算法** | **OT**(Jupiter client-server),中央定序 · **FACT**[2](#ref-2)[3](#ref-3) | 伺服器仲裁 edit-script / rebase(**OT 家族,非 CRDT**);官方未指名 · **FACT(機制)/官方未指名**[7](#ref-7)[26](#ref-26) | 線上:operation→transaction 伺服器整組 commit;**離線:CRDT** · **FACT**[9](#ref-9)[10](#ref-10) |
| **協作粒度** | 字元 / item 級 · **FACT**[13](#ref-13) | 文件變更 / step 級(細節 INF)· **INF**[7](#ref-7) | block / record 級(字元級語意未揭露)· **FACT/UNK**[9](#ref-9) |
| **即時傳輸層** | 現行未公開(史上 BrowserChannel/long-poll)· **INF/UNK**[37](#ref-37) | **WebSocket**(JWT+contentId;叢集需 session affinity)· **FACT**[15](#ref-15) | 訂閱 channel 的推送模型;傳輸技術未指名 · **FACT/UNK**[32](#ref-32) |
| **臨場感 / 游標** | 每人專屬顏色游標,**逐字**即時 · **FACT**[17](#ref-17) | telepointer 遠端選取 · **FACT**[6](#ref-6) | 產品有 presence/游標;機制未公開 · **FACT(功能)/UNK**[38](#ref-38) |
| **衝突解決** | OT transform 到收斂,無鎖 · **FACT**[3](#ref-3) | 中央 source of truth;發佈 `conflictPolicy=abort`(拒衝突);舊模型曾提示/合併/覆寫 · **FACT**[25](#ref-25)[30](#ref-30) | 伺服器 commit/reject transaction;離線 CRDT merge · **FACT**[9](#ref-9)[10](#ref-10) |
| **版本歷史** | 自動保存、可回復、作者顏色區分;細粒度 op 壓縮成具名版本 · **FACT**[18](#ref-18)[19](#ref-19) | 每次修改建整份快照版本;還原=複製舊版前移 · **FACT**[28](#ref-28) | 由 operation/transaction 導出(儲存格式未公開)· **INF/UNK**[9](#ref-9) |
| **離線編輯** | **支援**(桌面快取 + 行動原生),OT 對帳 · **FACT**[18](#ref-18) | **不支援**(Synchrony 需 live WebSocket)· **INF**[30](#ref-30) | **支援**(標記離線頁 → SQLite + CRDT + 時戳對帳)· **FACT**[10](#ref-10)[31](#ref-31) |
| **權限模型** | Drive 分享角色(owner/writer/…)· **FACT**[19](#ref-19) | 三層:全域→空間→頁面限制;view 繼承、edit 不繼承;只能收窄 · **FACT**[29](#ref-29) | 沿 parent 指標鏈,樹走訪評估 · **FACT**[36](#ref-36) |
| **擴展性 / 分片** | 未公開(per-doc 定序推測)· **UNK**[37](#ref-37) | Data Center active-active 叢集,共享 DB/home · **FACT**[30](#ref-30) | workspace 分片 Postgres(32→96 零停機)+ S3 資料湖 · **FACT**[11](#ref-11)[33](#ref-33) |
| **揭露程度(meta)** | 協作理論全公開、儲存幾乎不談 | 協作演算法從不指名、儲存/schema 文件齊全 | 儲存/分片一手詳盡、線上協作演算法極少談 |

---

## 5. Discussion

### 5.1 重審 rubric(看過證據後,這把尺還量對東西嗎?)

三處需要修正/強化:

1. **「並發演算法」不能只填 OT/CRDT 二選一。** 證據顯示真實世界是**光譜**:Google Docs 是教科書 OT;Confluence 是「OT 家族的中央仲裁 edit-script/rebase」(專利佐證,但官方不指名,且**非**去中心 CRDT);Notion 是「線上 operation/transaction 中央 commit + **離線** CRDT」的**混合**。硬塞成 OT vs CRDT 會失真——修正為記錄「中央 vs 去中心」「線上 vs 離線各用什麼」。這正是反證搜尋救回的精確度(Appendix B)。

2. **新增「揭露程度(meta)」這一列是必要的。** 三家公開的部位極不對稱(見表末列),不記錄的話讀者會誤把「Google 儲存欄=UNKNOWN」當成 Google 比較差,其實只是不公開。量尺必須把「這格是真差,還是只是沒揭露」分開——本文以 FACT/INF/UNK 標記達成。

3. **「文件模型」應升格為貫穿主軸而非並列面向之一。** 證據顯示它是因,其餘多是果(見 §5.2)。原 rubric 把它與其他 15 項並列,低估了它。

其餘判準(儲存、傳輸、離線、擴展)經證據檢驗仍成立,不變。

### 5.2 依(修正後)量尺評估:文件模型如何決定下游

把三欄表縱讀,一條因果鏈浮現——**文件模型決定了協作演算法能有多細、以及擴展怎麼走**:

- **op log(Google Docs)→ 天生適合字元級 OT。** 因為文件本來就是「一串操作」,操作正是 OT 能互相 transform 的單位,於是能做到**無鎖、逐字、即時**、還能離線後用同一套 change-log 對帳。代價:這套即時引擎複雜且高度專有,Google 選擇不公開儲存細節。

- **整段 XHTML 字串 in RDBMS(Confluence)→ 即時協作是「後加的」且需中央仲裁。** 正文是一大段字串,本質不利細粒度並發(§2.1(a));Confluence 早年只能用「提示他人正在編輯 + 合併/覆寫」的 save-based 模型,直到 6.0 才用 Synchrony 這個**外掛的中央權威服務**把「任意資料模型」即時同步上去。RDBMS 也決定了擴展走「Data Center 叢集」而非水平分片。全文檢索用 Lucene(貼著檔案系統索引)也是這個 on-prem/DC 世界觀的自然結果。

- **block 樹(Notion)→ 協作與擴展都以 block 為單位。** 因為一切皆 block、且每 block 屬於恰好一個 workspace,**分片鍵天然就是 workspace id**(整個 workspace 的 block 落在同一分片,避免跨分片 join);協作也落在 block/record 級的 operation/transaction;離線能用 CRDT 是因為 block 的 move/reparent/巢狀天然是可合併的結構操作。代價:渲染一頁要走訪一棵指標樹(N+1),權限要沿 parent 鏈算,分析要「tree traversal + 去正規化」——這些成本 Notion 的一手文章都直說了。

- **一句話貫穿:** **文件被「存成什麼」,就決定了它能被「一起改成什麼樣」。** op log→字元級 OT;字串 blob→後加中央仲裁;block 樹→block 級 operation + 離線 CRDT + workspace 分片。

### 5.3 誠實的不確定性

三個載重結論的證據強度不同:Google=OT、Notion=block 樹/分片/離線 CRDT 皆**一手強證**;Confluence 的協作**機制**有專利佐證但官方**不指名**(定性為推得的家族歸屬);三家的「線上並發精確語意」「儲存後端」多處 UNKNOWN(見 Appendix C)。凡表中 UNK/INF 者,不應被讀成能力差,而是**未公開或需進一步取證**。

---

## 6. Conclusion

**原問題:三者在架構取捨上各站在什麼位置、為什麼?** 依 §4.4 三欄表與 §5 的評估,直接回答:

- **Google Docs 站在「即時協作深度」的極端。** 因為文件模型是**字元級 op log**,它是唯一能**無鎖、逐字、即時**多人同編又支援離線的系統,協作演算法是教科書 **OT(Jupiter)**。取捨:把工程重心壓在即時引擎與 OT,**犧牲了儲存架構的公開性**(後端幾乎全 UNKNOWN)。適合「一份文件、多人此刻一起打字」。

- **Confluence 站在「企業 wiki 的結構化與可維運」一端。** 文件模型是**存進關聯式資料庫的 XHTML 字串**,即時協作(Synchrony)是**後加的中央仲裁 rebase**(OT 家族、非 CRDT、官方不指名),傳輸用 WebSocket、有 telepointer,但**不支援離線**,擴展靠 Data Center 叢集而非分片,全文檢索用 Lucene。取捨:換來 macro 生態、細緻的空間/頁面權限、企業可自管的 schema 與索引,協作即時性與離線能力則不是它的強項。適合「受治理的企業知識庫」。

- **Notion 站在「文件即資料、萬物皆 block」一端。** 文件模型是**指標串成的 block 樹**,線上協作是 **block 級 operation→transaction 中央 commit**、**離線**才明確用 **CRDT**;儲存是**依 workspace 分片的 Postgres** + S3 資料湖 + Elasticsearch。取捨:block 樹換來極高的內容彈性與乾淨的 workspace 分片鍵,但付出**渲染 N+1、權限樹走訪**的讀取成本,且**線上並發的精確語意官方未揭露**。適合「結構化工作區/知識庫,兼顧離線」。

**wrap-up(可帶走的一句):** 這三者不是「同一題的三個好壞答案」,而是**三種文件模型各自的邏輯終點**。要選型時,先問「我的內容天生是一串操作、一段文件、還是一棵 block 樹?」——答案基本上就決定了你能做字元級 OT、只能後加中央 rebase、還是走 block 級 operation+CRDT+分片這三條路的哪一條。表中所有 UNK/INF 格請讀成「官方未公開或需再取證」,而非能力落差;要把某格從 UNK 升為 FACT,依 Appendix C 的 closure recipe 執行。

---

## Appendix A — References(逐字引文;provenance-only)

> 全部擷取於 2026-07-19,https-only。編號依正文首次引用順序排列;括號標記所屬系統。

- <a id="ref-1"></a>**[1]**(Google Docs) https://patents.google.com/patent/US10382547B2/en (assignee Google)—「Each OT pair keeps track of only one change at a time, and these changes are recorded in a changes log.」/「the source of truth for the document data is the server.」
- <a id="ref-2"></a>**[2]**(Google Docs) https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html (Google Wave OT 白皮書)—「Google Wave uses the Operational Transformation (OT) framework of concurrency control.」
- <a id="ref-3"></a>**[3]**(Google Docs) https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html —「The starting point for Wave OT was the paper 'High-latency, low-bandwidth windowing in the Jupiter collaboration system'.」/「Wave OT modifies the basic theory of OT by requiring the client to wait for acknowledgement from the server before sending more operations.」
- <a id="ref-4"></a>**[4]**(Confluence) https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html —「This page describes the XHTML-based format that Confluence uses to store the content of pages …」/「Technically, it's XML …」/巨集例 `<ac:emoticon ac:name="smile" />`。
- <a id="ref-5"></a>**[5]**(Confluence) https://confluence.atlassian.com/doc/confluence-data-model-127369837.html — BODYCONTENT=「The content of Confluence pages. No version information or other metadata is stored here.」;CONTENT=「A persistence table for the ContentEntityObject …」。
- <a id="ref-6"></a>**[6]**(Confluence) https://developer.atlassian.com/server/confluence/collaborative-editing-for-confluence-server/ —「It supports special synchronisation for HTML WYSIWYG editors, including telepointers (remote selections).」
- <a id="ref-7"></a>**[7]**(Confluence) https://developer.atlassian.com/server/confluence/collaborative-editing-for-confluence-server/ —「All changes are buffered; once Synchrony comes back online, buffered changes will be merged and synchronised between clients.」(機制描述;演算法名未見於任何 Atlassian 頁)。
- <a id="ref-8"></a>**[8]**(Notion) https://www.notion.com/blog/data-model-behind-notion —「Everything you see in Notion is a block. Text, images, lists, a row in a database, even pages themselves—these are all blocks …」
- <a id="ref-9"></a>**[9]**(Notion) https://www.notion.com/blog/data-model-behind-notion —「these changes are expressed as operations that create or update a single record.」/「operations are batched into transactions that are committed (or rejected) by the server as a group.」
- <a id="ref-10"></a>**[10]**(Notion) https://www.notion.com/blog/how-we-made-notion-available-offline —「pages that are marked as available offline are dynamically migrated to our new CRDT data model for conflict-resolution」/「reference tracking, background syncing, and rich-text conflict resolution」。
- <a id="ref-11"></a>**[11]**(Notion) https://www.notion.com/blog/sharding-postgres-at-notion —「we settled on an architecture consisting of 480 logical shards evenly distributed across 32 physical databases.」/「we decided to partition by workspace ID.」/「each block belongs to exactly one workspace.」
- <a id="ref-12"></a>**[12]**(Notion) https://www.notion.com/blog/data-model-behind-notion —「blocks can be nested inside of other blocks … The content attribute of a block is what stores the array of block IDs (or pointers) …」/「Instead, we use an 'upward pointer'—the parent attribute—for the permission system.」;API:https://developers.notion.com/reference/block(block 物件欄位 object/id/parent/type/has_children…)。
- <a id="ref-13"></a>**[13]**(Google Docs) https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html —「In Google Wave, every character, start tag or end tag in a document is called an item.」
- <a id="ref-14"></a>**[14]**(Confluence) https://confluence.atlassian.com/doc/content-index-administration-148844.html —「By default, Confluence uses Lucene for indexing.」/「You can find the index in the <home-directory>/index directory.」/批次「as often as every 5 seconds」;叢集每節點各存完整索引 + journal service 同步。
- <a id="ref-15"></a>**[15]**(Confluence) https://developer.atlassian.com/server/confluence/collaborative-editing-for-confluence-server/ —「A WebSocket session is opened to Synchrony using the provided JWT and the contentId of the page being edited」;叢集 LB「needs to support 'session affinity' and WebSockets」([24](#ref-24))。
- <a id="ref-16"></a>**[16]**(Google Docs) https://en.wikipedia.org/wiki/Google_Docs — 「the document is stored as a list of changes」(維基,引一手鏈;僅用於已被一手佐證處)。
- <a id="ref-17"></a>**[17]**(Google Docs) https://en.wikipedia.org/wiki/Google_Docs —「An editor's current position is represented with an editor-specific color/cursor …」/「users can see character-by-character changes as other collaborators make edits」。
- <a id="ref-18"></a>**[18]**(Google Docs) https://support.google.com/docs/answer/6388102 —「If you aren't connected to the internet, you can still create, view, and edit files on Google Docs, Sheets, Google Slides」;維基:「The Android and iOS apps natively support offline editing.」;[1](#ref-1)專利:「Offline editing can continue so long as the user deems.」
- <a id="ref-19"></a>**[19]**(Google Docs) https://developers.google.com/workspace/drive/api/guides/manage-revisions —「The list of revisions returned … might be incomplete for files with a large revision history, including frequently edited Google Docs …」;維基:「a revision history is automatically kept so past edits may be viewed and reverted.」
- <a id="ref-20"></a>**[20]**(Google Docs) https://www.systemdesignhandbook.com/guides/google-docs-system-design/ —(二手,自我 hedge)「In Google's infrastructure, this **likely** involves Bigtable for storing document content and operation logs … Spanner … Colossus …」→ 本文列 UNKNOWN/INFERENCE,非 FACT。
- <a id="ref-21"></a>**[21]**(Confluence) https://confluence.atlassian.com/doc/database-configuration-159764.html —「The embedded H2 database is only supported for testing …」;支援 Oracle/MySQL/PostgreSQL/SQL Server,經 JDBC 連線。
- <a id="ref-22"></a>**[22]**(Confluence) https://confluence.atlassian.com/doc/attachment-storage-configuration-166876.html —「By default, Confluence stores attachments in the attachments directory within the configured Confluence home folder.」/「These storage methods [database/WebDAV] are no longer supported.」
- <a id="ref-23"></a>**[23]**(Confluence) https://confluence.atlassian.com/spaces/DOC/pages/1206794554/Configuring+S3+object+storage —「attachment data is stored in S3 using the root prefix /confluence/attachments/v4」/「S3 object storage is for attachment data only.」
- <a id="ref-24"></a>**[24]**(Confluence) https://confluence.atlassian.com/doc/clustering-with-confluence-data-center-790795847.html —「Confluence uses a combination of local caches, distributed caches, and hybrid caches that are managed using Hazelcast.」
- <a id="ref-25"></a>**[25]**(Confluence) https://developer.atlassian.com/cloud/confluence/collaborative-editing/ —「Synchrony is a micro service that allows the synchronization of arbitrary data models in real time.」/「will act as the source of truth for page content.」/「snapshot … at regular intervals.」/「Currently only the abort policy is supported.」
- <a id="ref-26"></a>**[26]**(Confluence) https://patents.google.com/patent/US9355083B1/en (assignee **Atlassian Pty Ltd**;inventors Haymo Meran, Tobias Steiner)—「generating an edit script defining operations that may be performed on dataset A to generate dataset B」/「Server 104 processes edit script 1 … Server 104 then transmits the edit script to clients.」→ 伺服器仲裁 edit-script(OT 家族),不自稱 OT,明確非 CRDT。
- <a id="ref-27"></a>**[27]**(Confluence) https://marijnhaverbeke.nl/blog/collaborative-editing.html (ProseMirror 作者;Atlaskit 編輯器底層)—「ProseMirror's algorithm is centralized …」/「Unlike OT, it does not try to guarantee that applying changes in a different order will produce the same document.」→ 佐證「中央權威 rebase、非去中心 CRDT」的家族歸屬(旁證,非 Atlassian 一手)。
- <a id="ref-28"></a>**[28]**(Confluence) https://confluence.atlassian.com/doc/page-history-and-page-comparison-views-139379.html —「Confluence tracks the history … by creating a new version of the page each time it's modified.」/「restoring an older version creates a copy of that version」。
- <a id="ref-29"></a>**[29]**(Confluence) https://confluence.atlassian.com/doc/page-restrictions-139414.html —「View restrictions are inherited … edit restrictions are not inherited.」/「Restrictions don't override a person's space permission.」
- <a id="ref-30"></a>**[30]**(Confluence) https://confluence.atlassian.com/doc/clustering-with-confluence-data-center-790795847.html —「All application nodes are active and process requests.」/ shared home=「attachments …」、local home=「Lucene indexes, configuration files …」;離線:無任何官方離線編輯頁(→ INFERENCE 不支援)。舊並發模型:https://confluence.atlassian.com/doc/concurrent-editing-and-merging-changes-144719.html —「If another user is editing the same page … Confluence will display a message …」/「If there are no conflicting changes, Confluence will merge the changes.」
- <a id="ref-31"></a>**[31]**(Notion) https://www.notion.com/blog/how-we-made-notion-available-offline —「For years, Notion has used SQLite to cache records locally and speed up page loads.」
- <a id="ref-32"></a>**[32]**(Notion) https://www.notion.com/blog/how-we-made-notion-available-offline —「Each client tracks a lastDownloadedTimestamp for every offline page. On reconnect, we compare that timestamp with the server's lastUpdatedTime and only fetch pages where the server version is newer.」/「Clients subscribe to these channels … when they receive a message, they fetch the latest changes for the page.」
- <a id="ref-33"></a>**[33]**(Notion) https://www.notion.com/blog/the-great-re-shard —「tripling the number of instances in our fleet from 32 to 96 machines」/「We used built-in Postgres logical replication …」/「Total time to synchronize the machines decreased from 3 days to 12 hours!」/「no report or observation … of perceived user downtime」;「connection limits scaling our PgBouncer deployment」。
- <a id="ref-34"></a>**[34]**(Notion) https://www.notion.com/blog/building-and-scaling-notions-data-lake —「use S3 as a data repository and lake … position data warehouse and other product-facing data stores such as ElasticSearch, Vector Database, Key-Value store … as its downstream」/「Debezium CDC connectors … Apache Hudi … Spark」/「more than two hundred billion blocks—a data volume of hundreds of terabytes」。
- <a id="ref-35"></a>**[35]**(Notion) https://bigdataboutique.com/customers/notion(第三方 vendor case study,**二手**)—「migrating from a managed solution to an Elasticsearch deployment running on AWS EC2 instances」→ 僅作 live search 細節的弱佐證,附 caveat。
- <a id="ref-36"></a>**[36]**(Notion) [34](#ref-34)—權限「has to be constructed on the fly via expensive tree traversal computation」;「complex data processing logics like tree traversal and block data denormalization」。
- <a id="ref-37"></a>**[37]**(Google Docs) 傳輸/擴展:現行 wire protocol、per-doc 定序、op-log 壓縮頻率官方未公開(史上為 BrowserChannel/long-poll,屬 INFERENCE)。Jupiter 一手理論:Nichols/Curtis/Dixon/Lamping, UIST 1995, DOI 10.1145/215585.215706(摘要付費牆,metadata 經 researchr.org 確認)。
- <a id="ref-38"></a>**[38]**(Notion) presence/游標為產品功能但機制未公開;頁面歷史儲存格式未公開;線上並發精確語意未公開。
- <a id="ref-39"></a>**[39]**(Confluence) Synchrony 血緣:Haymo Meran 為被收購公司 Wikidocs 創辦人(二手 biographical);其一手技術演講「How we built Synchrony …」(YouTube,無法伺服器端擷取逐字,列為最佳待補一手來源)。
- <a id="ref-40"></a>**[40]**(Notion) https://www.notion.com/blog/data-model-behind-notion —「Every block has the following attributes: ID … Properties … Type …」
- <a id="ref-41"></a>**[41]**(Notion) https://www.notion.com/blog/data-model-behind-notion — Content=「an array (or ordered set) of block IDs representing the content inside this block」;Parent=「the block ID of the block's parent. The parent block is only used for permissions.」

## Appendix B — 反證搜尋日誌(Refutation)

每個載重的協作結論都做過一次以上「刻意找反例」的搜尋:

- **Google Docs「是 OT 還是 CRDT?」** — 明確反證查詢。所有一手(Wave 白皮書[2](#ref-2)、Google 專利[1](#ref-1))與二手一致指向 **OT**;無任一來源宣稱 Docs 用 CRDT。最強反面只是「OT 被批評易出 bug」(二手,批評複雜度而非改變分類)。**反證 Docs=OT 的嘗試落空,結論維持 OT。**
- **Confluence「Synchrony 是 OT / CRDT / 都不是?」** — 同時做確認與反證。**確認失敗**:遍查 Atlassian 一手文件/部落格,**無一使用「operational transformation / OT / CRDT」自稱**;WebSearch 摘要宣稱「Synchrony 用 OT」無法回溯到任何開啟過的 Atlassian 頁,降級為 LEAD。**機制的最強一手是專利[26](#ref-26)**(伺服器 edit-script,OT 家族、非 CRDT)。`conflictPolicy=abort`[25](#ref-25)與「central source of truth」皆與去中心 CRDT 相反。**結論:OT 家族中央仲裁、非 CRDT、官方未指名——依證據而非印象。**
- **Notion「是 OT 還是 CRDT?」** — 一手只明確兩點:線上 operation/transaction[9](#ref-9)、離線 CRDT[10](#ref-10)。二手流傳的「OT+CRDT hybrid / 純 CRDT / LWW」互相矛盾且無一手佐證,**全部拒採並列 UNKNOWN**,不為填格而選一個。

---

## Appendix C — UNKNOWN 清單與 closure recipe

| 項目 | 系統 | 為何 UNKNOWN | closure recipe(read-only) |
|---|---|---|---|
| 底層資料庫 / 二進位分離 / 快取 / 全文檢索引擎 | Google Docs | Google 從未公開 Docs 儲存棧 | 搜 Google 專利(assignee Google, 關鍵字 storage/Bigtable/Spanner);Google Cloud Next / SREcon Workspace 工程師演講;檢視 Drive API `files.get` 欄位與 `exportLinks` 看二進位分離訊號 |
| 現行 wire protocol / per-doc 定序 / op-log 壓縮頻率 | Google Docs | 專有 | 對 docs.google.com 做網路擷取(DevTools WS/XHR 面板);比對開源 Closure `goog.net.BrowserChannel`;專利 US20160173594A1 / US10880372B2 |
| Synchrony 演算法官方指名 | Confluence | Atlassian 文件從不指名 | 觀看 Haymo Meran「How we built Synchrony」演講逐字;搜 atlassian.com/engineering 的 Synchrony 架構文;細讀專利 US9355083B1 claims |
| Cloud 資料庫/搜尋引擎、版本為快照或 diff | Confluence | Cloud 內部與 DB 欄位細節未全公開 | developer.atlassian.com Cloud platform architecture;開啟 Confluence Data Model 版本表段(CONTENT 版本列 / PREVVER 鏈) |
| 線上並發精確語意(block 級 LWW?字元級?)、傳輸技術、CRDT 演算法、頁面歷史儲存 | Notion | 官方僅揭露 operation/transaction 與離線 CRDT | 追 notion.com/blog 後續協作/搜尋專文;網路擷取即時通道;查 Notion 是否公開 rich-text CRDT(Peritext 血緣)一手聲明 |
| 圖片/檔案是否存 S3、伺服器端快取(Redis?) | Notion | 資料湖用 S3 為一手,但無「檔案存 S3」逐字 | 搜 notion.com/blog 快取/媒體專文;檢視 Notion 附件 URL 網域;Notion infra 工程師會議演講 |

---

*本文獨立取證,未沿用任何既有筆記;所有外部主張均附 URL + 逐字引文 + 擷取日期(2026-07-19),證據等級以 FACT / INFERENCE / UNKNOWN 標記,承重協作結論均附反證日誌(Appendix B)。*
