# 線上協作文件編輯器三系統深度設計比較:Google Docs vs Confluence vs Notion

> 來源:2026-07-19 的 read-only 調查(獨立取證;https-only、一手優先、URL + 逐字引文 + 擷取日期)。承重結論皆做過反證搜尋;事實 / 推論 / 未知分開標示。
> 延伸追問:[能不能直接拿 FTS 或 S3 當「可編輯文件」的主存放?](./fts-s3-as-primary-store.md)

---

## TL;DR

三個系統名義上都是「多人協作文件編輯器」,但**核心協作演算法完全不同,而且差異來自各自的文件模型(document model)**——這是本調查最重要的發現:

- **Google Docs** — 消費級即時協作標竿。文件存成一條**操作日誌(operation log / revision log)**,協作用 **Operational Transformation(OT,操作變換)**,**字元級(character-by-character)**即時同步。這是 Google 官方 2010 年三篇工程 blog 逐字自陳的[[1](#ref-1)][[2](#ref-2)][[3](#ref-3)]。兩人同時打同一個字,兩人的字都會保留且最終收斂。
- **Confluence(Server/Data Center)** — 企業 wiki。每頁內容以 **XHTML「storage format」整份 blob** 存進關聯式資料庫的 `BODYCONTENT` 表[[9](#ref-9)][[10](#ref-10)];即時協作由獨立微服務 **Synchrony** 經 **WebSocket** 驅動[[14](#ref-14)]。演算法**官方從未具名**——OT 是本文標記清楚的**推論**,不是 Atlassian 明述的事實。
- **Notion** — block 式工作區。每個 block 是 **Postgres 的一列(row)**,組成 block-tree[[21](#ref-21)][[22](#ref-22)];協作是**伺服器裁決的 last-write-wins(LWW,後寫覆蓋)**——由資料模型作者本人(Notion 工程師 Jake Teton-Landis)公開證實**「Notion 生產環境不用 OT 也不用 CRDT」**[[26](#ref-26)][[27](#ref-27)]。CRDT 直到 2025-12 才導入,且**僅限離線頁面的衝突解決**[[25](#ref-25)]。網路上盛傳「Notion 用 CRDT」是**誤傳**,本文用一手來源反駁。

**最大 caveat:** Google 與 Notion 的協作演算法都有官方一手逐字證據;**Confluence 的演算法識別是三者中證據最弱的一環**(Atlassian 只描述行為、不具名演算法),已標為 UNKNOWN 並附 closure recipe。此外三家的**儲存後端物理細節**(Google 用哪個資料庫、Notion 的 live search 引擎)多為未公開,凡屬推論皆逐處標示。

一句話定位:**Google Docs 把「即時、字元級、絕不衝突」做到極致(代價是不公開、不可自架);Confluence 把「企業級整頁 wiki + 事後可合併」做穩(代價是協作粒度粗、需維運 Synchrony);Notion 把「彈性 block 資料模型 + 水平分片」做大(代價是文字協作長年是 LWW,細粒度併發保證最弱)。**

---

## 1. Introduction

### 1.1 這份 spike 為什麼要做(purpose)

線上協作文件編輯器看起來是同一類產品,但「多人同時編輯一份文件」在工程上有天差地別的實作路線。本 spike 的目的,是**像一份系統設計深度解析**:不只列出三個系統各有什麼功能,而是先建立「這類系統該怎麼比較」的設計面向(rubric),再逐系統用一手證據取證,最後回答**三者在架構取捨(trade-off)上各站在什麼位置、為什麼**。

這對做技術選型、面試系統設計、或理解「為什麼 Google Docs 能字元級即時而 Notion 不能」的人,都是可直接使用的知識。

### 1.2 要回答的具體問題(questions)

1. 三者各自的**文件模型與儲存**是什麼?(整份 blob / op log / block-tree / 字元序列;資料庫;快取;二進位與文字是否分離;全文檢索引擎)
2. 三者各自的**協作機制**是什麼?(OT / CRDT / 鎖 / LWW;粒度是字元、區塊還是整頁;即時傳輸層是 WebSocket、long-poll 還是 REST;前端如何呈現多人編輯與解衝突)
3. 三者在**版本歷史、離線編輯、權限、擴展性(sharding)**上如何取捨?
4. **文件模型如何決定協作演算法**?(本文的貫穿論點)

### 1.3 調查涵蓋的面向(dimensions)

儲存(document model、DB、cache、object storage、FTS)、協作(algorithm、transport、granularity、conflict resolution、presence)、以及版本歷史 / 離線 / 權限 / 擴展性。完整 rubric 見 [§3 Methodology](#3-methodology)。

### 1.4 三個受測系統一句話定義(先給座標)

- **Google Docs** — Google 的雲端文書處理器,2010 年改版導入即時協作,消費級即時協作的公認標竿。
- **Confluence** — Atlassian 的企業 wiki / 知識庫;本文以有公開架構文件的 **Server / Data Center** 版為主,並在差異處標註 **Cloud** 版。
- **Notion** — 以「block(區塊)」為原子單位的模組化工作區,文件、資料庫、頁面皆是 block。

---

## 2. Prerequisites

本節把後續論證會用到的**承重名詞與結構性概念先教一遍**,每個結構性概念都附一個最小具體實例。已熟悉者可跳至 [§3](#3-methodology)。

### 2.1 四種「文件模型(document model)」——文件在後端到底長什麼樣

「文件模型」指**一份文件被持久化(persist)成什麼資料結構**。這是本文的地基,因為**文件模型幾乎決定了協作演算法**。四種原型:

**(a) 整份 blob(blob / serialized document)** — 整份文件序列化成一大塊字串/XML/JSON,存進資料庫的一個欄位。編輯 = 讀出整塊、改、整塊寫回。

```text
BODYCONTENT.body 欄位(單一 row):
"<p>Hello <strong>world</strong></p><p>second paragraph</p>"
      ↑ 整頁內容是這一格字串。改一個字 → 覆寫整格。
```

**(b) 操作日誌(operation log / op log)** — 不存「現在的內容」,而存**一連串「發生過的操作」**;「現在的內容」是把日誌從頭重播(replay)算出來的。這是 Google Docs 的模型。

```text
revision log(append-only,每行一個操作):
  rev1: {InsertText 'H'  @0}
  rev2: {InsertText 'i'  @1}
  rev3: {DeleteText       @1}   // 刪掉 'i'
目前內容 = 從 rev1 重播到 rev3 = "H"
新編輯不改舊資料,只在日誌尾端 append 一條。
```

op log 天生帶來版本歷史(重播到任一 rev 即得該時刻快照),也是 OT 能運作的前提。

**(c) block 樹(block-tree)** — 文件是一棵樹,每個節點(段落、標題、圖、清單項)是一個獨立「block」,各自是資料庫裡的一列(row),用指標串成父子關係。這是 Notion 的模型。

```text
Postgres 每個 block 一列:
{ id:"a1", type:"page",   content:["b2","b3"], parent:"root" }
{ id:"b2", type:"text",   properties:{title:"Hello"}, parent:"a1" }
{ id:"b3", type:"bullet", properties:{title:"world"}, parent:"a1" }
                          ↑ content 陣列 = 子 block 的有序 id 清單,構成「render tree」
編輯一個段落 = UPDATE 那一列;新增段落 = INSERT 一列 + 更新父的 content 陣列。
```

**(d) 字元序列(character sequence / CRDT 序列)** — 每個字元是帶唯一 id 的獨立元素,靠 id 排序合併。是 CRDT 文字型別的典型模型;三個受測系統的**線上主路徑都不用**它(Notion 2025 起僅在離線用 CRDT)。

### 2.2 協作演算法三種原型

**Operational Transformation(OT,操作變換)** — 每個編輯是一個「操作」;當兩人的操作基於不同版本併發送達時,伺服器/客戶端把後到的操作**依先到的操作「變換」座標**,使兩者都成立。Google 官方給的真實例子[[2](#ref-2)]:

```text
兩人同時對 "0123456789..." 操作:
  A: {ApplyStyle bold @10-20}   （把第10~20字設粗體）
  B: {InsertText 'ABC' @15}     （在第15字插入 "ABC",長度+3）
B 先套用後,A 的粗體範圍要順移 → 變換結果:
  {ApplyStyle Bold @10-23}      （終點從20推到23,涵蓋新插入的3字）
兩人最終看到一致內容。這個「把 @10-20 變換成 @10-23」就是 OT。
```

OT 需要一個中央權威(伺服器)排定全域順序;實作正確才保證收斂,歷史上以難實作著稱。

**CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別)** — 資料型別設計成「不管操作以什麼順序到達,合併結果都相同(可交換)」,因此**不需要中央變換**,天生適合離線/去中心。代價是每個元素要背 metadata(唯一 id、tombstone),資料會膨脹。

**Last-write-wins(LWW,後寫覆蓋)** — 最簡單:同一個欄位/block 有併發寫入時,**伺服器決定的「最後那筆」直接覆蓋先前的**,先前的內容遺失。Notion 的文字協作即此[[26](#ref-26)]。粒度夠細(每個 block 很小)時使用者才不易察覺遺失。

```text
A、B 幾乎同時編輯同一個 block b2 的文字:
  A 寫 "Hello there"   （伺服器記為 t=101）
  B 寫 "Hello world"   （伺服器記為 t=102）
LWW 結果:b2 = "Hello world"      ← A 的 "there" 直接消失,無合併。
但因為 b2 只是一個段落(不是整頁),衝擊面被 block 粒度侷限住。
```

### 2.3 其他名詞(一行即懂)

- **Presence / telepointer(臨場感 / 遠端游標)** — 畫面上顯示「誰在線上、游標/選取在哪」的即時小訊號,與內容編輯分開傳送。
- **WebSocket** — 瀏覽器與伺服器間的持久雙向連線,即時協作的常見傳輸層(相對於一問一答的 REST、或反覆輪詢的 long-poll)。
- **Object storage / Bucket(物件儲存,如 Amazon S3)** — 專門放大型二進位檔(圖片、附件)的儲存服務,與放文字的資料庫分離,便宜且可獨立擴展。
- **Full-text search / FTS(全文檢索)** — 讓使用者搜文件內文字;常用倒排索引引擎(如 **Apache Lucene**、**Elasticsearch**)。
- **Sharding(分片)/ partition key(分片鍵)** — 把資料水平切成多份放在多台資料庫,用某個鍵(如 workspace id)決定某列去哪一片,以突破單機容量。
- **OT / CRDT / LWW / blob / op log / block-tree** 已於上文定義,後續直接使用。

---

## 3. Methodology

本節先立**評分準則(rubric)**,再進證據。每條準則有三部分:**名稱 → 量尺(YARDSTICK:看什麼、什麼算好)→ 利害(STAKES:這個維度差異會對誰造成什麼實際後果)**。凡是講不出利害的維度就不列為準則。

### 3.1 準則清單

**準則一:文件模型(Document model)**
- 量尺:持久化的原子單位是整份 blob、op log、block-tree row 還是字元序列?
- 利害:**文件模型設下了協作演算法的天花板**。op log/字元序列才可能做無損字元級併發;整份 blob 只能整頁替換或事後合併;block-tree 讓 block 級併發便宜、字元級難。若三系統在此不同,意味著三者**能提供的最細併發保證上限不同**——直接決定「兩人打同一句話會不會有人的字消失」。

**準則二:儲存後端與快取(Storage backend & caching)**
- 量尺:用哪種資料庫?快取用什麼?是否可自架?
- 利害:決定**維運負擔與可自主性**。可自架關聯式 DB(Confluence)對企業是資料主權優勢但要自己維運;閉源雲端後端(Google)零維運但零掌控。這對「受法遵約束、必須自架」的組織是能否採用的硬門檻。

**準則三:二進位/文字分離與 object storage**
- 量尺:圖片/附件是否與文字分存?放檔案系統還是 S3?
- 利害:決定**主資料庫會不會被大檔撐爆、以及儲存成本曲線**。不分離會讓 DB 膨脹、備份變慢;分離到 S3 則便宜且可獨立擴展。對「大量夾檔的知識庫」是成本與擴展性問題。

**準則四:全文檢索(FTS)**
- 量尺:引擎是什麼?索引如何建與更新(即時 vs 批次)?
- 利害:決定**搜尋新鮮度與自架者要額外維運什麼**。批次索引(如 Confluence 每 ~5 秒)意味著「剛存的字可能搜不到」;獨立搜尋叢集是額外維運負擔。對「靠搜尋找回舊文件」的重度使用者是找得到 / 找不到的差別。

**準則五:協作演算法(Collaboration algorithm)**
- 量尺:OT、CRDT、LWW 還是鎖?是官方明述(fact)還是推論(inference)?
- 利害:**本比較的核心**。決定**併發編輯時會不會有人的內容被無聲覆蓋**。OT/CRDT 保證收斂不丟字;LWW 會丟(後寫覆蓋);鎖則犧牲併發性。對「多人同時趕同一份稿」是資料會不會遺失的差別。

**準則六:協作粒度(Granularity)**
- 量尺:併發解析的最小單位是字元、block 還是整頁?
- 利害:決定**「兩人能靠多近同時編輯而互不干擾」**。字元級 = 可同段同句;block 級 = 可同頁不同段;整頁級 = 實質互斥。對即時共筆體驗是「絲滑」還是「卡頓/互相踩」。

**準則七:即時傳輸層(Transport)**
- 量尺:WebSocket、long-poll、SSE(Server-Sent Events,伺服器單向推播)還是純 REST?
- 利害:決定**看到他人變更的延遲、以及基礎設施要求**(如 WebSocket 需要負載平衡器與代理支援)。對自架者是「LB/proxy 要不要特別設定」的維運事實。

**準則八:版本歷史 / diff / 回溯**
- 量尺:是否保留完整歷史?能否 diff、回溯到任一版?
- 利害:決定**誤刪/誤改的可回復性與稽核能力**。對受監管或多人協作的文件是合規與救援能力。

**準則九:離線編輯與同步**
- 量尺:能否離線編輯?重連如何同步、如何解衝突?
- 利害:決定**在飛機/地鐵/弱網下能不能繼續工作**。對行動與弱網使用者是可用性。

**準則十:權限模型**
- 量尺:分享/權限的粒度(工作區 / 空間 / 頁面 / block)與繼承方式。
- 利害:決定**能多精細地控制誰看得到什麼**。對企業是資安與治理能力。

**準則十一:擴展性 / sharding**
- 量尺:如何水平擴展?分片策略是否公開?
- 利害:決定**系統能長多大、以及多租戶隔離方式**。對平台方是規模天花板。

### 3.2 取證與判定紀律

每條「協作演算法」判定都做過**反證搜尋**(見 [Appendix B](#appendix-b--反證搜尋紀錄)):三家各自「到底是 OT、CRDT 還是都不是」都獨立查證,不憑印象。官方未公開者標 **UNKNOWN + closure recipe**,不臆測。二手系統設計部落格不得冒充 fact,凡引用皆標為 secondary。

---

## 4. Findings

本節逐系統呈現「查到什麼、代表什麼」(finding altitude);逐字引文與原始證據下沉到 [Appendix A](#appendix-a--原始證據原文引文與-references)。引用編號 [n] 對應 [References](#references引用清單一手為主二手已標示)。

### 4.1 Google Docs — op log + OT + 字元級,官方逐字自陳

**文件模型(fact)。** Google 官方 2010 三篇工程 blog 逐字說明:文件存成一條**變更日誌(revision log)**,不是字元 blob——「我們把你的文件存成一份由這些變更組成的 revision log……編輯者不是在修改代表文件的底層字元,而是把他的變更 append 到 revision log 的尾端。要顯示文件,我們從頭重播 revision log」[[2](#ref-2)]。變更有三種原型:`InsertText`、`DeleteText`、`ApplyStyle`,且帶座標運算元(例:`{InsertText 'T' @10}`)[[1](#ref-1)][[2](#ref-2)]。伺服器持久化三樣東西:未處理變更清單、已處理變更的完整歷史(revision log)、以及最後一個已處理變更後的文件現狀(快照)[[3](#ref-3)]。

**協作演算法 = OT(fact,官方多次具名)。** 「我們用來處理這些位移的演算法叫做 operational transformation(OT)。若 OT 實作正確,它保證一旦所有編輯者都收到所有變更,大家會看到同一版文件」[[2](#ref-2)]。Google 明說**鎖被考慮過但被否決**(「鎖住段落不是好方案……你是在用損害協作體驗來迴避技術挑戰」)[[1](#ref-1)]。粒度是**字元級**:「這些技術一起造就了 Google Docs 的 character-by-character 協作」[[1](#ref-1)],「編輯者可以逐字看到彼此的變更」[[3](#ref-3)]。

**協定(fact)。** 每個客戶端追蹤四樣:伺服器來的最新 revision 號、本地已改未送、已送未確認、以及該編輯者所見文件現狀;**且一次只送一個 pending 變更**,收到 ack 前新變更都排隊[[3](#ref-3)]。伺服器以 OT 把遲到的客戶端變更**變換到自上次同步以來已提交的所有變更之上**再存為新 revision[[3](#ref-3)]。本地編輯**樂觀立即套用**,不等伺服器,故「網路速度不影響你打字速度」[[3](#ref-3)]。這與 Google Wave OT 白皮書的「客戶端須等伺服器 ack 才送下一批」模型一致[[4](#ref-4)]。

**傳輸層(inference / UNKNOWN)。** 2010 三篇只說「變更在 client 與 server 間傳送」,**未具名 WebSocket 或 BrowserChannel**;2013 Realtime API 是同引擎的對外產品化,官方說「基於 OT,本地變更即時反映,即使高延遲網路」[[5](#ref-5)]。當前實際 wire transport 未公開(UNKNOWN,closure:對現行 Docs session 做網路擷取)。

**版本歷史 / 離線 / 擴展性。** 版本歷史是 op log 模型的直接副產品(重播到任一 rev)[[2](#ref-2)](fact)。離線:2019 官方說離線所做變更於重連後經 Drive 同步,需 Chrome 的 Google Docs Offline 擴充[[7](#ref-7)](fact);其複用 OT 重播調解是合理推論(inference)。擴展性:官方僅給**演算法層**主張「協作複雜度是常數,伺服器不需知道每個客戶端的狀態,故編輯者增加不會增加處理複雜度」[[3](#ref-3)](fact);**物理 sharding、後端資料庫、FTS 索引管線全未公開(UNKNOWN)**——二手宣稱 Spanner/Bigtable/Colossus 僅為 secondary 推論[[29](#ref-29)]。

**反證結果:** 試圖反駁「Google Docs 用 OT」的搜尋**落空**;最強獨立來源是一篇同儕審查論文,其主旨反而是「CRDT 優越性被誇大、多數商用共筆(含 Google Docs)仍是 OT」,並引 Google 自家 2010 貼文為證[[8](#ref-8)]。唯一誠實 caveat:官方陳述止於 2010–2013,2013 後引擎若改版屬未公開(無證據顯示有改)。

### 4.2 Confluence — 整頁 XHTML blob + Synchrony/WebSocket,演算法官方未具名

**文件模型與儲存(fact)。** 每頁內容以 **XHTML-based「storage format」**(官方澄清「技術上是 XML」)儲存[[9](#ref-9)];頁 body 存在 **`BODYCONTENT` 表**(「Confluence 頁面的內容。此處不存版本資訊或其他 metadata」),metadata 存在 `CONTENT` 表[[10](#ref-10)]。這是**整頁 blob 模型**(inference:官方未說「blob」,但「storage format 存頁面內容」+ 單表 body 強烈指向整份序列化)。支援 PostgreSQL/MySQL/Oracle/SQL Server,**可自架**[[10](#ref-10)]。快取:叢集用 **Hazelcast**(「叢集時 Confluence 用 Hazelcast 管理的 local/distributed/hybrid 快取組合」)[[19](#ref-19)](fact)。

**二進位/文字分離(fact)。** 附件**預設存檔案系統**(home 目錄的 `attachments`),**DB 儲存已不再支援**,**8.1 起可存 Amazon S3**[[11](#ref-11)];每個附件版本是獨立檔案(如 `12345678.1`、`12345678.2`)[[12](#ref-12)]。故**文字(DB)與二進位(檔案系統/S3)分離**。

**FTS(fact)。** 用 **Apache Lucene**,分 content index 與 change index,存於各節點 `<local-home>/index`;變更**不即時更新索引,而是入佇列每 ~5 秒批次處理**[[13](#ref-13)]。文字型附件內文會被抽取並索引[[13](#ref-13)]。

**協作演算法(inference — 三者中最弱的一環)。** Synchrony 是「同步任意資料模型的微服務,支援 HTML WYSIWYG(What You See Is What You Get,所見即所得)編輯器的特殊同步,含 telepointer(遠端選取)」[[14](#ref-14)]。傳輸是 **WebSocket**(JWT(JSON Web Token,簽章式存取憑證)+ `contentId` 開 WebSocket session),**Synchrony 是編輯期的 single source of truth**[[14](#ref-14)]。**共享草稿(shared draft)**模型:每頁一份草稿,所有人看到同一份[[14](#ref-14)]。**但沒有任何 Atlassian 一手文件具名演算法**——我親自核對官方 collaborative-editing 文件、6.0 blog[[17](#ref-17)]、administering 文件[[15](#ref-15)]皆**不含 "operational transformation"/"OT"/"CRDT" 字樣**。「Synchrony = OT」是**推論**(依據:中央 source-of-truth + 共享草稿 + 逐操作即時同步是典型 OT 拓撲;降級路徑退回「存檔時合併衝突」暗示逐操作調解而非 CRDT 可交換合併)。唯一曾具名的一手(AtlasCamp 2016 talk「How we built Synchrony」)其 URL 現已 302 轉址走[[20](#ref-20)],web.archive.org 被環境封鎖——列為首要 UNKNOWN。

**協作粒度與併發上限(fact)。** **最多 12 人同時編輯一頁**[[15](#ref-15)];降級(limited)模式下**一次只有一人能編**共享草稿,且「存檔時嘗試合併衝突(類似 Confluence 5 體驗)」[[15](#ref-15)][[14](#ref-14)]。歷史:**6.0(2016)前無即時協作**,併發是**偵測並存檔時合併**——「若變更不重疊自動合併;若重疊,顯示錯誤要使用者選 Continue editing / Overwrite / Cancel」[[16](#ref-16)]。

**版本歷史 / 擴展性(fact)。** 每次修改建新版,保留全部歷史;diff **綠=新增、紅=刪除、藍=格式變更**;回溯是**還原成一份複本**[[18](#ref-18)]。Data Center 叢集:共享 DB + 共享 home(附件/頭像/匯出)+ 各節點 local home(logs/快取/Lucene 索引);**LB 需支援 session affinity 與 WebSocket**[[19](#ref-19)]。離線編輯:Server/DC web 編輯器**無**真正離線(Synchrony 需活的 WebSocket)(inference,反證未找到離線功能)。

**反證結果:** 試圖在 Atlassian 一手來源找「CRDT 描述」或「鎖為常規演算法」皆**落空**——唯一的鎖行為明確是**降級模式**,唯一的 merge-on-save 明確是**6.0 前 / 降級路徑**。OT 假說未被反駁,但仍是**推論**,非 Atlassian 明述之事實。

### 4.3 Notion — block-tree(Postgres row)+ LWW,官方作者親口否認 CRDT

**文件模型(fact)。** 「你在 Notion 看到的一切都是 block。文字、圖片、清單、資料庫的一列,甚至頁面本身——都是 block」[[21](#ref-21)]。block 具體結構:UUID(Universally Unique Identifier,全域唯一識別碼)v4 的 **id**、**properties**(最常見是存文字的 `title`)、**type**、**content**(子 block id 的有序陣列,構成 render tree)、**parent**(「parent block 只用於權限」)[[21](#ref-21)]。**每個 block 是資料庫的一列**[[22](#ref-22)]。

**儲存與分片(fact)。** **Postgres**,**應用層分片(application-level sharding)**,**分片鍵 = workspace id**(「使用者通常一次只查一個 workspace 的資料,故避開多數跨片 join」)[[22](#ref-22)]。2021:**480 個邏輯分片(Postgres schema)平均分佈於 32 台實體 DB**(每台 15 schema),單表上限 500 GB、單機 10 TB[[22](#ref-22)]。2023:**擴到 96 台(3×),零停機**,用 Postgres 邏輯複製搬遷,PgBouncer 做連線池[[23](#ref-23)]。2024:另建 **S3 資料湖**(每 Postgres host 一個 Debezium CDC(Change Data Capture,變更資料擷取)connector → Kafka(每表一 topic,480 分片共寫)→ Apache Hudi 寫入 S3 → Spark 處理),規模 **2000 億+ blocks、數百 TB**,Snowflake 保留給其他工作負載;資料湖服務 **Search、AI 等產品需求**[[24](#ref-24)]。

**二進位/FTS(inference / UNKNOWN)。** 上傳檔案/圖片(二進位)存 S3、block 文字結構存 Postgres 是**合理推論**(未捕獲逐字一手確認具體上傳檔路徑)。**線上 live search 引擎未經一手確認**——二手(BigData Boutique)稱 Notion 在 EC2 跑 **Elasticsearch**[[28](#ref-28)],標為 secondary;資料湖一手確認「服務 Search」但未具名引擎[[24](#ref-24)](UNKNOWN,closure 見 Appendix C)。

**協作演算法 = LWW(fact,作者親口,已本人二次核對)。** Notion 資料模型 blog 的作者、Notion 工程師 **jitl(Jake Teton-Landis)**在 Hacker News 公開陳述:**「Notion 生產環境不用 OT 也不用 CRDT。多數東西是 last-write-wins,但有些操作會合併,如清單重排或權限變更」**[[26](#ref-26)];**「Notion 是『協作的』,但我們的文字不用 CRDT,全是伺服器裁決的 last-write-wins……這就是我們正在轉向 CRDT 做文字的原因」**[[27](#ref-27)]。我已透過 HN Algolia API 逐字核對兩則留言與作者身分。粒度是 **block 級**:因每個 block/段落很小且獨立,LWW 才可接受;資料模型 blog 亦說解耦 property 儲存「對協作至關重要,因為我們盡量保留使用者意圖」[[21](#ref-21)]。

**傳輸與離線(fact)。** 即時傳輸是 **WebSocket 的 pub/sub channel(每頁一個)**[[25](#ref-25)]。離線(2025-12 一手):多年來用 **SQLite** 在本地快取記錄,離線模式把它演進成持久儲存層 + 離線頁樹「forest」+ 每頁 `lastDownloadedTimestamp` 對比伺服器 `lastUpdatedTime` 的時間戳增量同步;**被標為離線的頁面動態遷移到新的 CRDT 資料模型做衝突解決**[[25](#ref-25)]。**故 CRDT 在 Notion 僅離線範疇、且到 2025-12 才有**——非一般線上協作演算法。

**版本歷史 / 權限 / 擴展性。** 權限沿 block 的 `parent` 鏈繼承(「parent 只用於權限」)[[21](#ref-21)](fact);完整 ACL(Access Control List,存取控制清單)(guest、公開連結)為使用者可見但未一手引用。版本歷史快照節奏與保留天數為 secondary/UNKNOWN(免費 ~7 天等,取自定價頁,未逐字捕獲)[[30](#ref-30)]。擴展性頭條即上述分片三部曲(全 fact)。

**反證結果(本文最重要的一次反駁):** 網路盛傳「Notion 用 CRDT」或「CRDT 結構 + OT 文字混合」——我做反證搜尋,找到的最強一手證據**直接反駁**該說法:資料模型作者本人 2023 年兩度明說不用 OT/CRDT、全是 LWW[[26](#ref-26)][[27](#ref-27)]。唯一一手確認的 CRDT 用途是 **2025-12 離線衝突解決**[[25](#ref-25)]。盛傳的「Notion = CRDT」是**誤傳**,很可能把 2025 離線功能與一般協作引擎混為一談。

### 4.4 三欄比較表(核心產物)

> 圖例:**fact** = 官方一手逐字證據;**inference** = 有據推論(已於 Findings 標明依據);**UNKNOWN** = 官方未公開。

| 設計面向 | **Google Docs** | **Confluence (Server/DC)** | **Notion** |
|---|---|---|---|
| **文件模型** | 操作日誌 op log(revision log),存 `{InsertText/DeleteText/ApplyStyle @座標}`,重播得內容 **(fact** [[1](#ref-1)][[2](#ref-2)][[3](#ref-3)]**)** | 整頁 XHTML「storage format」blob,存 `BODYCONTENT` 表 **(fact** [[9](#ref-9)][[10](#ref-10)]**;「blob」為 inference)** | block-tree:每 block 一列,`content[]` 子 id + `parent`(僅權限)**(fact** [[21](#ref-21)][[22](#ref-22)]**)** |
| **儲存後端 / 快取** | 未公開;二手稱 Spanner/Bigtable/Colossus **(UNKNOWN;二手** [[29](#ref-29)]**)** | PostgreSQL/MySQL/Oracle/SQL Server,**可自架**;Hazelcast 叢集快取 **(fact** [[19](#ref-19)]**)** | Postgres,應用層分片;SQLite 本地快取;server 端快取棧未公開 **(fact + UNKNOWN** [[22](#ref-22)][[25](#ref-25)]**)** |
| **二進位/文字分離** | 未公開(圖片後端未述) **(UNKNOWN)** | 文字在 DB;附件在**檔案系統(預設)/ S3(8.1+)**,DB 儲存已棄用 **(fact** [[11](#ref-11)][[12](#ref-12)]**)** | 文字在 Postgres;檔案在 S3 **(inference)**;另有 S3 資料湖 **(fact** [[24](#ref-24)]**)** |
| **全文檢索 FTS** | 有(可搜)但索引管線未公開 **(UNKNOWN)** | **Apache Lucene**,content+change index,**每 ~5 秒批次**更新,各節點本地索引 **(fact** [[13](#ref-13)]**)** | live search 引擎未經一手確認(二手稱 Elasticsearch);資料湖服務 Search **(UNKNOWN + 二手** [[28](#ref-28)]**;湖 fact** [[24](#ref-24)]**)** |
| **協作演算法** | **OT(Operational Transformation)(fact,官方多次具名** [[2](#ref-2)][[3](#ref-3)]**)** | **未具名**;結構上最像 OT **(inference;Atlassian 一手從不具名)** | **LWW(last-write-wins),部分操作可合併(fact,作者親口** [[26](#ref-26)][[27](#ref-27)]**)**;CRDT 僅離線(2025-12** [[25](#ref-25)]**)** |
| **協作粒度** | **字元級 character-by-character (fact** [[1](#ref-1)][[3](#ref-3)]**)** | 頁級共享草稿,最多 12 人;降級模式退回單人編輯 **(fact** [[15](#ref-15)]**)** | **block 級**(每段獨立)**(fact** [[26](#ref-26)]**)** |
| **即時傳輸層** | 未具名(2010 只說「傳送變更」);Realtime API 基於 OT **(inference/UNKNOWN** [[5](#ref-5)]**)** | **WebSocket**(JWT + contentId) **(fact** [[14](#ref-14)]**)** | **WebSocket** pub/sub channel(每頁) **(fact** [[25](#ref-25)]**)** |
| **版本歷史 / diff** | op log 的直接副產品,重播到任一 rev **(fact** [[2](#ref-2)]**)** | 每改建新版,保留全部;diff 綠/紅/藍;回溯=建複本 **(fact** [[18](#ref-18)]**)** | 有版本歷史;快照節奏/保留天數 **(secondary/UNKNOWN** [[30](#ref-30)]**)** |
| **離線編輯** | 支援,經 Chrome 擴充,重連經 Drive 同步 **(fact** [[7](#ref-7)]**)** | web 編輯器**無**真離線(Synchrony 需活 WebSocket) **(inference)** | 支援(2025-12);SQLite 持久層 + 時間戳增量同步 + **CRDT 解衝突 (fact** [[25](#ref-25)]**)** |
| **權限模型** | Drive 層 ACL(viewer/commenter/editor、連結分享),疊在編輯器之上 **(inference)** | 空間權限 + 頁面 restriction **(inference;本 run 未逐字取證)** | 沿 block `parent` 鏈繼承的頁/工作區 ACL **(fact 依據** [[21](#ref-21)]**)** |
| **擴展性 / sharding** | 僅演算法層「協作複雜度常數」主張;物理分片未公開 **(fact + UNKNOWN** [[3](#ref-3)]**)** | Data Center 叢集(共享 DB+home,各節點 local index);Synchrony 可獨立成叢集 **(fact** [[19](#ref-19)]**)** | 應用層分片 workspace id;480 邏輯分片 32→96 台,零停機;2000 億+ blocks **(fact** [[22](#ref-22)][[23](#ref-23)][[24](#ref-24)]**)** |

---

## 5. Discussion

### 5.1 重審 rubric——見過證據後,量尺還量對東西嗎?

大體成立,但有兩處需要明確修正:

**(a) 「協作演算法」準則的證據等級必須被平權看待,不能一律當 fact 比較。** 立準則時我假設三家的演算法都能查到官方明述。實際上**只有 Google(逐字具名 OT)與 Notion(作者親口 LWW)達到 fact 等級;Confluence 是 inference**。因此在下結論時,對 Confluence 的演算法定位必須降級措辭(「結構上最像 OT」),不能與另兩家的一手證據並列。這是證據紀律對修辭的約束:**Confluence 那一格永遠帶星號。**

**(b) 「協作演算法」與「協作粒度」高度耦合,應合看而非分看。** 原本列為兩條獨立準則,但證據顯示**粒度決定了演算法的可容忍度**:Notion 的 LWW 之所以「可用」,正因粒度是 block 級(每次覆蓋只影響一小段);若 Notion 是整頁 LWW 就會災難性丟稿。Google 的 OT 之所以「必要」,正因它要字元級——字元級併發沒有 OT/CRDT 就無法無損。所以評比時這兩條要**綁在一起解讀**(見 §5.2)。

其餘準則(儲存、二進位分離、FTS、版本、離線、權限、擴展性)量尺與利害在證據下都站得住,不需修改。

### 5.2 沿(修正後)量尺評比,並把差異翻譯成利害

**文件模型 → 協作演算法的因果鏈(本文貫穿論點,三家各自印證):**
- Google 選 **op log**,所以能做 **OT + 字元級**:op log 把編輯表達成可變換的操作序列,這正是 OT 的輸入。**利害翻譯:** 這讓「兩人同時打同一句話、兩人的字都保留且收斂」成為可能——這是消費級即時共筆體驗的天花板,也是為什麼 Google Docs 至今是標竿。代價是這套引擎**完全不公開、不可自架**。
- Confluence 選 **整頁 XHTML blob**,所以**至今無法在儲存層做細粒度併發**——才需要外掛一個 Synchrony 微服務在**編輯期**維護一份可協作的中介狀態,發佈時再轉回 storage format;6.0 之前根本只能**存檔時整頁合併**。**利害翻譯:** 對企業,好處是文件在關聯式 DB 裡、可自架、可 SQL 稽核、與既有備份體系相容;代價是協作是「頁級共享草稿 + 最多 12 人 + Synchrony 一掛就退回單人 merge-on-save」,細粒度併發保證弱於 Google。
- Notion 選 **block-tree(row)**,所以協作自然落在 **block 級 LWW**:每個 block 是獨立小 row,UPDATE 一列即編輯,伺服器裁決最後寫入者勝。**利害翻譯:** 好處是資料模型極彈性(任何東西都是 block、可自由搬移)且**天生好水平分片**(以 workspace id 切 480 片、擴到 96 台、撐 2000 億 blocks);代價是**同一個 block 內兩人併發打字,先寫者的字會被無聲覆蓋**——這是三者中細粒度併發保證**最弱**的,Notion 自己也因此「正在轉向 CRDT」。

**儲存後端與可自主性:** Google(閉源雲端,零維運零掌控)vs Confluence(可自架關聯式 DB,資料主權強但要自己維運 Hazelcast/Synchrony/Lucene)vs Notion(閉源雲端 Postgres 分片)。**利害:** 對「法遵要求必須自架」的組織,**只有 Confluence 是可行選項**——這一格的差異直接決定某些企業能不能採用。

**二進位/文字分離與 FTS:** Confluence 證據最完整且最「教科書」(文字進 DB、附件進檔案系統/S3、Lucene 每 5 秒批次索引)。**利害:** 這種透明度本身對自架者是價值——你知道要維運什麼、搜尋為何有 ~5 秒延遲。Google 與 Notion 的對應細節多為 UNKNOWN,**利害:** 使用者無從自行調優或除錯搜尋,只能信任供應商。

**即時傳輸層:** Confluence 與 Notion 都明確 **WebSocket**;Google 反而**未具名**(2010 太早、之後未更新)。**利害:** 對自架的 Confluence,這是「LB/proxy 必須支援 WebSocket」的硬維運事實[[19](#ref-19)];對 Google 使用者則無關(供應商代管)。

**離線編輯:** Notion(2025-12 起,SQLite 持久層 + CRDT 解衝突)與 Google Docs(Chrome 擴充 + Drive 同步)都支援;Confluence Server/DC web 編輯器**實質不支援**(Synchrony 需活連線)。**利害:** 對行動/弱網重度使用者,Confluence 是明顯短板。

**版本歷史:** 三家都有,但**成因不同**——Google 是 op log 的免費副產品(最細,理論上可回到任一操作);Confluence 是每次存檔建新版(頁級、附彩色 diff);Notion 是伺服器端快照(節奏未公開)。**利害:** Google 的回溯粒度最細,Confluence 的 diff 呈現最成熟(綠/紅/藍),Notion 的保留天數受方案綁定。

### 5.3 誠實的證據不對稱(不為對稱美化任何一方)

三家的協作能力差異**很大**,且證據強度**也不對稱**,本文據實呈現:
- **即時字元級**(Google,fact)> **即時 block 級 LWW**(Notion,fact)> **頁級共享草稿、掛了退回單人 merge**(Confluence,fact 行為 + inference 演算法)。
- 這不是「三個都很棒各有千秋」的對稱故事:**Notion 的文字併發保證確實最弱**(LWW 會丟字,官方承認並在補救);**Confluence 的演算法證據確實最薄**(官方從不具名);**Google 的協作最強但架構最不透明**(後端幾乎全 UNKNOWN)。每一格的星號都保留。

---

## 6. Conclusion

**回到原始問題:三者在架構取捨上各站在什麼位置、為什麼?**

**Google Docs — 「即時協作的上限」象限。** 因為選了 **op log** 文件模型,它能且必須用 **OT** 做**字元級**即時協作,並用「一次一個 pending 變更 + 樂觀本地套用 + 伺服器線性 revision」的協定把體驗做到「打字不受網路影響、絕不衝突、逐字可見」[[1](#ref-1)][[2](#ref-2)][[3](#ref-3)]。**取捨:** 換來標竿級體驗與內建版本歷史,代價是後端**完全不透明、不可自架**——對重視資料主權或想自行調優的組織,這是硬牆。演算法證據等級:**最強(官方逐字)**。

**Confluence — 「企業級整頁 wiki + 可自架」象限。** 因為選了**整頁 XHTML blob** 存進可自架關聯式 DB,它的協作是**外掛式**的:6.0 才用 Synchrony/WebSocket 把「頁級共享草稿」做成即時,掛掉就退回 6.0 前的**存檔合併**;細粒度併發不是它的強項[[14](#ref-14)][[15](#ref-15)][[16](#ref-16)]。**取捨:** 換來**資料主權、SQL 可稽核、與企業備份/叢集體系相容、透明的 Lucene 搜尋與 S3 附件**,代價是協作粒度粗、需自行維運 Synchrony/Hazelcast、web 端無離線。**它是三者中唯一可自架的**——這一點對特定企業直接決定「能不能用」,而非「好不好用」。演算法證據等級:**最弱(官方從不具名,本文標為 inference + UNKNOWN,closure recipe 見 Appendix B)**。

**Notion — 「彈性 block 模型 + 極致水平擴展」象限。** 因為選了 **block-tree(每 block 一 Postgres row)**,它換來無與倫比的資料模型彈性與**乾淨的 workspace 分片**(480 片、32→96 台零停機、2000 億+ blocks、獨立 S3 資料湖餵 Search/AI)[[22](#ref-22)][[23](#ref-23)][[24](#ref-24)]。**取捨:** 代價是文字協作長年是 **block 級 LWW**——**同一段落內併發會丟字**,這是三者中最弱的細粒度保證,Notion 自己承認並正轉向 CRDT(目前 CRDT 僅落地於 2025-12 的離線衝突解決)[[26](#ref-26)][[27](#ref-27)][[25](#ref-25)]。演算法證據等級:**強(資料模型作者親口,本文已二次核對)**,且順帶**反駁了網路盛傳的「Notion 用 CRDT」誤說**。

**可行動的一句話 wrap-up。** 若你要的是**極致即時共筆體驗**,Google Docs 的 op log + OT 路線是標竿;若你要的是**可自架、可稽核、與企業基建相容的知識庫**且能接受頁級協作,Confluence 是唯一可自架選項;若你要的是**高度彈性的 block 工作區與可長到很大的多租戶平台**且能接受(至今的)文字 LWW,Notion 是該象限之王。**三者的位置不是靠「差異數量」排出來的,而是各自的文件模型設下的協作天花板 × 可自主性 × 擴展性三個利害軸決定的**——這正是「文件模型決定協作演算法」這條主線在三個真實系統上的印證。

**最需要繼續追的一個缺口:** Confluence Synchrony 的演算法識別(是否真為 OT、何種 OT 變體)——一手證據缺失,closure recipe:取回 AtlasCamp 2016「How we built Synchrony」talk 錄影/逐字稿(見 [Appendix B](#appendix-b--反證搜尋紀錄))。

---

## Appendix A — 原始證據:原文引文與 References

本附錄承載 Findings 引用的逐字證據。每則標 [n] 對應下方 References。

### A.1 Google Docs 逐字引文

- **op log 模型**[[2](#ref-2)]:"Think of the history of a document as a series of changes... we save your document as a revision log consisting of a list of these changes. When someone edits a document, they're not modifying the underlying characters that represents the document. Instead they are appending their change to the end of the revision log. To display a document, we replay the revision log from the beginning."
- **變更帶座標**[[1](#ref-1)]:"a document is stored as a series of chronological changes. A change might be something like {InsertText 'T' @10}... instead of computing the changes by comparing document versions, we now compute the versions by playing forward the history of changes."
- **OT 具名 + 收斂保證**[[2](#ref-2)]:"The algorithm that we use to handle these shifts is called operational transformation (OT). If OT is implemented correctly, it guarantees that once all editors have received all changes, everyone will be looking at the same version of the document."
- **OT 變換實例**[[2](#ref-2)]:"{ApplyStyle bold @10-20} transformed against {InsertText 'ABC' @15} results in {ApplyStyle Bold @10-23}."
- **鎖被否決**[[1](#ref-1)]:"you could lock paragraphs so that only one editor was ever allowed to type in a single paragraph at a given time. But locking paragraphs isn't a great solution: you're sidestepping the technical challenges by hampering the collaborative editing experience."
- **字元級**[[1](#ref-1)]:"Together, these technologies create the character-by-character collaboration in Google Docs." [[3](#ref-3)]:"there are no more collaboration conflicts and editors can see each other's changes as they happen, character-by-character."
- **四項狀態 + 一次一個 pending**[[3](#ref-3)]:"each client keeps track of four pieces of information: 1. The number of the most recent revision sent from the server... 2. Any changes that have been made locally and not yet sent... 3. Any changes... sent... but not yet acknowledged... 4. The current state of the document as seen by that particular editor." + "we never send more than one pending change at a time."
- **伺服器再變換**[[3](#ref-3)]:"the server will use OT to transform John's change so that it can be stored as Revision 3... transform John's sent change against all the changes that have been committed since the last time John synced with the server."
- **樂觀本地 + 常數複雜度**[[3](#ref-3)]:"every editor can optimistically apply their own changes locally without waiting for the server... the complexity of processing changes does not increase as you add more editors."
- **伺服器持久化三樣**[[3](#ref-3)]:"The server remembers three things: 1. The list of all changes that it has received but not yet processed. 2. The complete history of all processed changes (called the revision log). 3. The current state of the document as of the last processed change."
- **Realtime API 基於 OT**[[5](#ref-5)]:"Because the Drive Realtime API is based on operational transformation (OT), local changes are reflected instantly, even on high-latency networks."
- **Wave OT 客戶端等 ack**[[4](#ref-4)]:"Wave OT modifies the basic theory of OT by requiring the client to wait for acknowledgement from the server before sending more operations."
- **離線**[[7](#ref-7)]:"Any changes made to files while offline will then sync in Drive once the user is connected again." + "The Google Docs Offline extension... is still required."

### A.2 Confluence 逐字引文

- **storage format**[[9](#ref-9)]:"This page describes the XHTML-based format that Confluence uses to store the content of pages... Technically, it's XML, since the storage format doesn't fully comply with the XHTML definition."
- **BODYCONTENT / CONTENT**[[10](#ref-10)]:CONTENT = "A persistence table for the ContentEntityObject class of objects."；BODYCONTENT = "The content of Confluence pages. No version information or other metadata is stored here."；"The Hibernate mapping files... are the *.hbm.xml files."
- **Hazelcast**[[19](#ref-19)]:"When clustered, Confluence uses a combination of local caches, distributed caches, and hybrid caches that are managed using Hazelcast."
- **附件儲存**[[11](#ref-11)]:"By default, Confluence stores attachments in the attachments directory within the configured Confluence home folder." + "you may still be storing attachments in your database or WebDAV. These storage methods are no longer supported." + "Starting from Confluence 8.1, you can also store your attachment data on Amazon S3 object storage."
- **附件版本檔名**[[12](#ref-12)]:"the attachment file names for versions 1, 2, and 6 will be 12345678.1, 12345678.2, 12345678.6 respectively."
- **Lucene + 批次索引**[[13](#ref-13)]:"a content index that contains content such as the text of pages..." + "a change index..." + "Changes... aren't updated in each index immediately. They're placed into queues and regularly processed in batches (as often as every 5 seconds)."
- **Synchrony + WebSocket + source of truth**[[14](#ref-14)]:"Synchrony is a service that allows the synchronisation of arbitrary data models in real time. It supports special synchronisation for HTML WYSIWYG editors, including telepointers (remote selections)." + "A WebSocket session is opened to Synchrony using the provided JWT and the contentId of the page being edited." + "the service will act as the source of truth for page content." + "A single shared draft is created for each page, and anyone editing will see the same draft."
- **12 人 + limited mode**[[15](#ref-15)]:"A maximum of 12 people can edit a page at the same time." + "your load balancer and proxies should allow WebSocket connections." + "When a site is in limited mode, only one person can edit a shared draft at one time" + "Confluence will attempt to merge any conflicts on save."
- **6.0 前偵測合併**[[16](#ref-16)]:"If the changes don't overlap... Bob's changes will be merged with Alice's automatically." + "If Bob's changes overlap with Alice's, Confluence will display an error message" + 選項 "Continue editing—Overwrite—Cancel"。
- **版本歷史 / diff**[[18](#ref-18)]:"restoring an older version creates a copy of that version." + "Green: Added content, Red: Deleted content, Blue: Changed formatting"。
- **叢集拓撲**[[19](#ref-19)]:"Each Confluence node has a local home that contains logs, caches, Lucene indexes and configuration files." + shared home 含 "attachments, avatars... export files..." + "The load balancer needs to support 'session affinity' and WebSockets."

### A.3 Notion 逐字引文

- **block 定義**[[21](#ref-21)]:"Everything you see in Notion is a block. Text, images, lists, a row in a database, even pages themselves—these are all blocks."
- **block 結構**[[21](#ref-21)]:"We use randomly-generated UUIDs (UUID v4) for IDs." + "Properties—a data structure... The most common property is title..." + "Content—an array (or ordered set) of block IDs..." + "Parent—the block ID of the block's parent. The parent block is only used for permissions."
- **block=row**[[22](#ref-22)]:"Notion's data model revolves around the concept of a block, each occupying a row in our database."
- **應用層分片 + workspace 鍵**[[22](#ref-22)]:"an approach known as application-level sharding." + "we used the workspace ID as the partition key."
- **480 分片 / 32 台**[[22](#ref-22)]:"480 logical shards evenly distributed across 32 physical databases." + "we set an upper bound of 500 GB per table and 10 TB per physical database."
- **re-shard 96 台零停機**[[23](#ref-23)]:"tripling the number of instances in our fleet from 32 to 96 machines." + "we went from having 15 schemas per shard to 5." + "we designed the process to not require downtime" + "a proxy layer (PgBouncer) pools connections."
- **資料湖**[[24](#ref-24)]:"use S3 as a data repository and lake..." + "one Debezium CDC connector per Postgres host..." + "one Kafka topic per Postgres table and let all connectors consuming from 480 shards write to the same topic" + "use Apache Hudi..." + "We chose Spark..." + "more than two hundred billion blocks—a data volume of hundreds of terabytes" + 服務 "AI, Search, and other product requirements"。
- **LWW(作者親口)**[[26](#ref-26)]:"Notion doesn't use OT or CRDT in production. Most things are last-write-wins, but we have operations that merge like list re-ordering or permission changes." [[27](#ref-27)]:"we don't use a CRDT for text, it's all last-write-wins decided by the server." + "That's why we're working on switching to CRDT for our texts."
- **離線 + CRDT + WebSocket channel**[[25](#ref-25)]:"For years, Notion has used SQLite to cache records locally..." + "we evolved our SQLite cache into a persistent storage layer" + "Each client tracks a lastDownloadedTimestamp for every offline page. On reconnect, we compare that timestamp with the server's lastUpdatedTime" + "pages that are marked as available offline are dynamically migrated to our new CRDT data model for conflict-resolution" + "Clients subscribe to these channels for each of their offline pages" + "Published December 11, 2025"。

### References(引用清單,一手為主,二手已標示)

<a id="ref-1"></a>[1] Google Drive Blog — *What's different about the new Google Docs: Working together, even apart*(2010-09-21,一手)。https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_21.html(逐字引文另交叉核對 UW IDL 存檔 PDF https://idl.uw.edu/future-scholarly-communication/files/2010-GoogleDocs-OT.pdf)。擷取 2026-07-19。

<a id="ref-2"></a>[2] Google Drive Blog — *What's different about the new Google Docs: Conflict resolution*(2010-09-22,一手)。https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_22.html。擷取 2026-07-19。

<a id="ref-3"></a>[3] Google Drive Blog — *What's different about the new Google Docs: Making collaboration fast*(2010-09-23,一手)。https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html。擷取 2026-07-19。

<a id="ref-4"></a>[4] Google Wave — *Operational Transformation* whitepaper(Apache Wave,一手)。https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html。擷取 2026-07-19。

<a id="ref-5"></a>[5] Google Developers Blog — *Build collaborative apps with Google Drive Realtime API*(2013-03-19,一手)。https://developers.googleblog.com/en/build-collaborative-apps-with-google-drive-realtime-api/。擷取 2026-07-19。

<a id="ref-6"></a>[6] Google Workspace Updates — *Committed to storage APIs, retiring the Realtime API*(2017-11-28,一手)。https://workspaceupdates.googleblog.com/2017/11/committed-to-storage-apis-retiring.html。擷取 2026-07-19。

<a id="ref-7"></a>[7] Google Workspace Updates — *Work anywhere with Google Docs, Sheets, and Slides in new offline mode*(2019-04,一手)。https://workspaceupdates.googleblog.com/2019/04/drive-offiline-mode.html。擷取 2026-07-19。

<a id="ref-8"></a>[8] David Sun, Chengzheng Sun, Agustina Ng, Weiwei Cai — *Real Differences between OT and CRDT in Building Co-Editing Systems and Real World Applications*, arXiv:1905.01517(學術一手)。https://arxiv.org/pdf/1905.01517。擷取 2026-07-19。

<a id="ref-9"></a>[9] Atlassian — *Confluence Storage Format*(一手)。https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html。擷取 2026-07-19。

<a id="ref-10"></a>[10] Atlassian — *Confluence Data Model*(CONTENT / BODYCONTENT / .hbm.xml、支援的資料庫,一手)。https://confluence.atlassian.com/doc/confluence-data-model-127369837.html。擷取 2026-07-19。

<a id="ref-11"></a>[11] Atlassian — *Attachment Storage Configuration*(檔案系統 / DB 棄用 / S3 8.1,一手)。https://confluence.atlassian.com/doc/attachment-storage-configuration-166876.html。擷取 2026-07-19。

<a id="ref-12"></a>[12] Atlassian — *Hierarchical File System Attachment Storage*(一手)。https://confluence.atlassian.com/doc/hierarchical-file-system-attachment-storage-704578486.html。擷取 2026-07-19。

<a id="ref-13"></a>[13] Atlassian — *Content Index Administration*(Lucene、content/change index、每 5 秒批次,一手)。https://confluence.atlassian.com/doc/content-index-administration-148844.html。擷取 2026-07-19。

<a id="ref-14"></a>[14] Atlassian Developer — *Collaborative editing for Confluence Data Center*(Synchrony、WebSocket、JWT、source-of-truth、telepointer,一手)。https://developer.atlassian.com/server/confluence/collaborative-editing-for-confluence-server/。擷取 2026-07-19。

<a id="ref-15"></a>[15] Atlassian — *Administering Collaborative Editing*(12 人、WebSocket、limited mode、merge on save,一手)。https://confluence.atlassian.com/doc/administering-collaborative-editing-858772086.html。擷取 2026-07-19。

<a id="ref-16"></a>[16] Atlassian — *Concurrent Editing and Merging Changes*(6.0 前 detect-and-merge,一手)。https://confluence.atlassian.com/spaces/CONF94/pages/1540721528/Concurrent+Editing+and+Merging+Changes。擷取 2026-07-19。

<a id="ref-17"></a>[17] Atlassian Blog — *Collaborative editing in Confluence 6.0*(一手;經本人核對,不含演算法名)。https://www.atlassian.com/blog/confluence/collaborative-editing-confluence-6-0。擷取 2026-07-19。

<a id="ref-18"></a>[18] Atlassian — *Page History and Page Comparison Views*(歷史 / 彩色 diff / 還原成複本,一手)。https://confluence.atlassian.com/doc/page-history-and-page-comparison-views-139379.html。擷取 2026-07-19。

<a id="ref-19"></a>[19] Atlassian — *Clustering with Confluence Data Center*(Hazelcast、shared/local home、Synchrony 叢集、LB session affinity + WebSocket,一手)。https://confluence.atlassian.com/doc/clustering-with-confluence-data-center-790795847.html。擷取 2026-07-19。

<a id="ref-20"></a>[20] Atlassian AtlasCamp 2016 — *How we built Synchrony, the engine behind collaborative editing in Confluence*(一手,但 URL 現 302 轉址走、web.archive.org 被環境封鎖 → UNKNOWN closure 目標)。https://www.atlassian.com/atlascamp/2016/archives/developer-best-practices/how-we-built-synchrony-the-engine-behind-collaborative-editing-in-confluence。擷取 2026-07-19(僅得轉址後的 landing page)。

<a id="ref-21"></a>[21] Notion Engineering — *The data model behind Notion's flexibility*(作者 Jake Teton-Landis,一手)。https://www.notion.com/blog/data-model-behind-notion。擷取 2026-07-19。

<a id="ref-22"></a>[22] Notion Engineering — *Herding elephants: Lessons learned from sharding Postgres at Notion*(2021,一手)。https://www.notion.com/blog/sharding-postgres-at-notion。擷取 2026-07-19。

<a id="ref-23"></a>[23] Notion Engineering — *The Great Re-shard: adding Postgres capacity (again) with zero downtime*(2023-07-17,一手)。https://www.notion.com/blog/the-great-re-shard。擷取 2026-07-19。

<a id="ref-24"></a>[24] Notion Engineering — *Building and scaling Notion's data lake*(2024,一手)。https://www.notion.com/blog/building-and-scaling-notions-data-lake。擷取 2026-07-19。

<a id="ref-25"></a>[25] Notion Engineering — *How we made Notion available offline*(2025-12-11,一手)。https://www.notion.com/blog/how-we-made-notion-available-offline。擷取 2026-07-19。

<a id="ref-26"></a>[26] Jake Teton-Landis(HN 用戶 jitl,Notion 工程師)— HN 留言 id 37767739(2023-10-04):"Notion doesn't use OT or CRDT in production..."(一手,經 HN Algolia API 逐字核對)。https://news.ycombinator.com/item?id=37767739。擷取 2026-07-19。

<a id="ref-27"></a>[27] Jake Teton-Landis(jitl)— HN 留言 id 38292180,thread 38289327(2023-11-16):"...it's all last-write-wins decided by the server" / "switching to CRDT for our texts"(一手,經 HN Algolia API 逐字核對)。https://news.ycombinator.com/item?id=38289327。擷取 2026-07-19。

<a id="ref-28"></a>[28] BigData Boutique — Notion Elasticsearch search 案例(**二手**,僅用於標示未經一手確認的 Elasticsearch 宣稱)。https://bigdataboutique.com/customers/notion。擷取 2026-07-19。

<a id="ref-29"></a>[29] Grokking the System Design — Google Docs 系統設計指南(**二手**,僅用於標示 Spanner/Bigtable/Colossus 之推論,無 Google 一手佐證)。https://grokkingthesystemdesign.com/guides/google-docs-system-design/。擷取 2026-07-19。

<a id="ref-30"></a>[30] Notion — 定價頁(**二手/未逐字捕獲**,版本歷史保留天數)。https://www.notion.com/pricing。擷取 2026-07-19。

---

## Appendix B — 反證搜尋紀錄

每條協作演算法判定的反證搜尋與結果:

**B.1 Google Docs「是否真為 OT?」** 反證查詢意在反駁「Google Docs 用 OT」:`Google Docs CRDT not operational transformation refute`。結果**落空**:最強獨立來源(arXiv:1905.01517[[8](#ref-8)])主旨反而是「CRDT 優越性被誇大、多數商用共筆含 Google Docs 仍是 OT」,且其參考文獻直接引 Google 2010 貼文。常被混淆的「Figma 用 CRDT」是 Figma、且明確與 Google Docs 對比,非反駁。唯一 caveat:官方陳述止於 2010–2013,之後若改版未公開(無證據顯示)。**判定:OT(fact),反證未動搖。**

**B.2 Confluence「Synchrony 是 OT、CRDT 還是鎖?」** 執行多次反證查詢(`Synchrony operational transformation`、`Synchrony CRDT`、`How we built Synchrony`),親自核對官方 collaborative-editing 文件[[14](#ref-14)]、6.0 blog[[17](#ref-17)]、administering 文件[[15](#ref-15)]:**皆不含 "operational transformation"/"OT"/"CRDT" 字樣**。找 CRDT 或「鎖為常規演算法」的描述**落空**——鎖只是降級模式、merge-on-save 只是 6.0 前/降級路徑。唯一曾具名的一手(AtlasCamp 2016 talk[[20](#ref-20)])URL 現 302 轉址走、web.archive.org 被環境封鎖。**判定:結構上最像 OT,但維持 inference,非 Atlassian 明述事實。Closure recipe:** 取回 AtlasCamp 2016「How we built Synchrony」talk 的 YouTube 錄影或逐字稿(搜 "AtlasCamp 2016 Synchrony collaborative editing"),或在非封鎖鏡像取回原頁;亦可反編譯 Synchrony jar 或檢視載入編輯器的 Synchrony JS bundle(靜態閱讀)。

**B.3 Notion「是否用 CRDT?」(本文最重要反駁)** 針對盛傳的「Notion 用 CRDT / CRDT+OT 混合」做反證:查詢 `Notion "not CRDT" last write wins Jake Teton-Landis`,經 HN Algolia API 取回並逐字核對兩則作者留言[[26](#ref-26)][[27](#ref-27)],確認作者 jitl = 資料模型 blog 作者 Jake Teton-Landis。**最強反證直接推翻盛傳說法:** 作者本人 2023 兩度明說「不用 OT/CRDT、全是伺服器裁決 LWW」。反向再試(找 Notion 一手說線上編輯器用 CRDT/OT)——**唯一一手 CRDT 用途是 2025-12 離線衝突解決**[[25](#ref-25)],非一般線上路徑。**判定:LWW(fact);「Notion=CRDT」為誤傳,已反駁。** 唯一 UNKNOWN:2025-12 後線上文字是否已從 LWW 遷往 CRDT(closure:找 2026 Notion 編輯器/即時工程貼文或新的作者陳述)。

---

## Appendix C — 未解問題(UNKNOWN)與 closure recipes 彙整

| # | UNKNOWN | Closure recipe(read-only) |
|---|---|---|
| U1 | Google Docs 後端資料庫、FTS 索引管線、物理 sharding、當前 wire transport | 檢視現行 Docs session 網路擷取(找 WebSocket upgrade 或 BrowserChannel `/bind?VER=`);查 Google Research 出版與 SREcon/Cloud Next 由 Workspace infra 團隊演講;查引用 2010 revision-log 模型的 Google 專利。 |
| U2 | Confluence Synchrony 演算法識別(OT?何種變體?中介模型) | 取回 AtlasCamp 2016 talk 錄影/逐字稿(見 B.2);檢視 Synchrony JS bundle 或反編譯 Synchrony jar(靜態閱讀)。 |
| U3 | Confluence BODYCONTENT 欄位名(`body`/`bodyType`)、單機快取引擎(EhCache?) | 取回完整 Data Model 表列[[10](#ref-10)],或讀 Confluence JAR 內 `*.hbm.xml` Hibernate mapping;搜 "Cache Statistics"/classpath 找 `ehcache`。 |
| U4 | Notion 線上 live search 引擎(Elasticsearch?)、server 端快取棧 | 找具名搜尋引擎的 Notion 一手貼文,或帶 Notion 具名/引述的 Elastic 案例;確認索引是由資料湖(Spark)還是 Postgres CDC 直建。 |
| U5 | Notion 版本歷史快照節奏/保留天數逐字、2025-12 後線上文字是否遷往 CRDT | 找現行 "Page history" help doc 正確 slug;找 2026 Notion 即時/編輯器工程貼文或新的 jitl/Notion-eng 陳述。 |
| U6 | 三家的 presence/游標協定細節 | 各自即時工程貼文;或客戶端逆向 + 一手佐證(read-only 網路擷取)。 |

*(以上 closure recipes 皆為唯讀查證,不涉及任何資料庫或產品變更。)*
