# 線上協作文件編輯器設計比較:Google Docs vs Confluence

> 來源:2026-07-16 的 read-only 調查(所有外部引文擷取日期 = 2026-07-16,除另註;皆為官方一手來源)。
> 正文維持 finding 高度;原始逐字引文全部收在附錄,以〔G#〕〔C#〕〔S#〕anchor 引用。
> 不熟悉這個領域?先讀 **§1.3 文件模型入門**(四種儲存形式各附具體例子),再回來看比較。
> 姊妹篇:[三系統版(+ Notion)](./trio-comparison.md) —— 獨立重新取證,含 Notion 的 block 樹/分片史,以及「Notion 用 CRDT 做即時協作」「Synchrony 是 OT」兩則流言的反證。

---

## 0. 先給答案(Answer-first)

| 系統 | 一句話定位 | 協作核心 | 儲存核心 |
|------|-----------|---------|---------|
| **Google Docs** | 字元級即時協作的工程極致:文件**本體就是一串操作日誌**(見 §1.3 例②),靠 OT 演算法在伺服器權威排序下即時合併 | **OT**(操作轉換,§1.3)+ 伺服器權威 | 操作日誌(op log)模型,實體後端未公開 |
| **Confluence** | 企業 wiki 的即時協作:獨立 Synchrony 服務做即時同步,但**文件落地仍是一份整包標記文字**(見 §1.3 例①)**存進關聯式 DB** | **Synchrony 即時服務**(演算法 OT/CRDT 官方未明說)+ 伺服器為 source of truth | XHTML 標記 blob 存 `BODYCONTENT` 關聯式資料表 |

**兩個核心取捨:**

1. **協作即時性**
   - Google Docs:字元級即時,衝突由演算法自動合併,使用者無感。
   - Confluence:共享草稿級即時,有 telepointer(遠端游標,§1.3);但即時層是**外掛的獨立服務**,不是文件模型的本質。
2. **文件模型決定協作演算法**(全文主軸)
   - Docs 把文件做成「操作序列」→ 字元級合併自然成立。
   - Confluence 保留「整份標記 blob 存 DB」→ 即時協作只能靠加掛服務補上。
   - 通則:**模型愈接近操作序列,即時協作愈自然;愈接近整份 blob,愈需要外掛即時層。**

---

## 1. 定義問題(Problem definition)

### 1.1 目的
把兩個線上協作編輯器標竿放在同一組設計面向下解析:同樣是「多人寫文件」,為什麼消費級的 Google Docs 與企業級的 Confluence 在**儲存、協作、檢索**上做了截然不同的取捨。

### 1.2 要回答的具體問題
- **Q1 儲存**:
  - 文件存進哪種資料庫?文件模型是哪一種(見 §1.3 的四種形式)?
  - 快取?有無 object storage(物件儲存,如 S3)分離二進位?
  - 全文檢索(FTS)引擎與索引怎麼管理?
- **Q2 協作**:
  - 演算法(OT / CRDT / save-based)?
  - 即時傳輸層(WebSocket / long-poll / XHR)?
  - 前端多人呈現(presence 游標 / 即時字元)?衝突怎麼解?
- **Q3 其他考題**:
  - 版本歷史 / diff / 回溯;離線編輯;
  - 文件模型如何反過來決定協作演算法。

### 1.3 文件模型入門:四種儲存形式,各看一個具體例子

「文件模型」= 系統把一份文件**實際存成什麼形狀**。這是全文最重要的概念,四種形式由粗到細:

**① 整份 blob(whole blob)—— Confluence 用這種**
blob = 一整包不透明的資料:資料庫只把它當一坨文字/位元組存在一個欄位裡,**看不見內部結構**。Confluence 存的是 XHTML(以 XML 語法規則書寫的 HTML)標記:

```xml
<!-- Confluence 的一頁 = 一份完整 XHTML 標記,整包存進 BODYCONTENT 表的一個欄位 -->
<p>會議記錄</p>
<ac:structured-macro ac:name="info">      <!-- ac: 開頭是 Confluence 自訂的 macro 元素 -->
  <ac:rich-text-body><p>決議:採用方案 B</p></ac:rich-text-body>
</ac:structured-macro>
<!-- 編輯 = 讀出整份 → 改 → 整份寫回。DB 不知道你「改了哪個字」 -->
```

**② 操作日誌(op log)—— Google Docs 用這種**
文件本體不是「目前的內容」,而是**從開頭到現在的每一筆操作**;目前內容是把日誌從頭「播放」出來的:

```jsonc
// 使用者打了 "Hi" 然後按退格,日誌長這樣:
{ "op": "InsertText", "text": "H", "pos": 0 }
{ "op": "InsertText", "text": "i", "pos": 1 }
{ "op": "DeleteText", "pos": 1, "len": 1 }
// 「目前的文件」= 播放整串 → "H"。版本回溯 = 播放到某一筆為止。
// 兩人同時編輯 = 兩串操作交錯到達,由 OT 演算法轉換後合併(見下)。
```

**③ 區塊樹(block-tree)—— Notion 這類系統用這種**(本文比較對象之外,列出供對照)
文件 = 一棵「區塊」樹,每個區塊(段落、標題、清單項…)是**一筆獨立記錄**:

```jsonc
{ "id": "page-1", "type": "page",    "children": ["b1", "b2"] }
{ "id": "b1", "type": "heading",     "text": "會議記錄", "parent": "page-1" }
{ "id": "b2", "type": "bulleted_li", "text": "決議:採用方案 B", "parent": "page-1" }
// 編輯 = 只改被碰到的那個 block 記錄。
// 粒度介於「整份 blob」與「字元」之間:協作衝突以 block 為單位,不同人改不同 block 就不衝突。
```

**④ 字元序列(CRDT 式)—— Figma 等採 CRDT 的系統用這種**(對照用)
每個字元帶一個**全域唯一 ID**,排序靠 ID 而非位置,因此不同人並發插入的合併結果天然一致:

```text
[ {id:"7@alice", ch:"H"}, {id:"3@bob", ch:"i"} ]
// 兩人同時插字,各自的 ID 決定最終順序,不需要中央伺服器仲裁。
// 代價:每個字元都背著 metadata,文件體積與記憶體開銷變大。
```

**這四種形式直接決定了協作能力**:①只能整份存檔比對(save-based)、②天然支撐字元級即時合併、③支撐 block 級協作、④支撐去中心的字元級合併。這就是 §0 主軸的由來。

### 1.4 其餘名詞(首現全名 + 用途)
- **OT(Operational Transformation,操作轉換)**:多人協作演算法。兩人同時改、操作到達順序不同時,把後到的操作「轉換」成基於已套用操作的等價操作,讓所有端收斂到同一份文件。**通常需要一台伺服器決定操作的權威順序**。
- **CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別)**:另一種協作演算法,即 §1.3 例④的資料結構 —— 不管操作以什麼順序到達、合併結果都一樣,**理論上不需要中央伺服器仲裁**。
- **save-based / 樂觀並發**:不做即時同步。各自編輯、按存檔;伺服器用「你是基於第幾版改的」判斷撞版,撞了報衝突要人工處理。最簡單、最不即時。
- **FTS(Full-Text Search,全文檢索)**:對文件正文建索引以支援關鍵字/語意搜尋。
- **Lucene / OpenSearch**:Lucene 是 Java 的全文檢索引擎庫(Elasticsearch/OpenSearch 的核心);OpenSearch 是可獨立部署的檢索叢集。
- **telepointer / presence**:即時顯示「其他人游標/選取在哪、誰在線上」的機制。
- **WebSocket / XHR**:WebSocket 是瀏覽器與伺服器之間的持久雙向連線(適合即時推送);XHR(XMLHttpRequest)是傳統的單次 HTTP 請求(可輪詢模擬即時,效率較差)。
- **JWT(JSON Web Token)**:簽名過的身分憑證,Confluence 用它讓瀏覽器向 Synchrony 開連線時證明身分。
- **mutation**:一筆「變更操作」記錄(Google 專利用語,即 §1.3 例②日誌裡的一行)。
- **ETag(entity tag)**:HTTP 的「內容版本指紋」;內容一變 ETag 就變,可拿來做「我改的還是不是我讀到的那一版?」檢查(§7 會用到)。

---

## 2. 定義比較 rubric(先立後量)

| # | Rubric 面向 | 判準 |
|---|-------------|------|
| R1 | **資料庫種類與文件模型** | §1.3 四種形式中的哪一種?模型是否支撐所選協作演算法? |
| R2 | **協作演算法** | OT / CRDT / save-based?字元級還是草稿級?伺服器權威還是去中心? |
| R3 | **即時傳輸層** | WebSocket / long-poll / XHR?有無專用即時服務/程序? |
| R4 | **前端多人呈現與衝突處理** | presence 游標?即時字元?衝突怎麼解? |
| R5 | **版本歷史 / diff / 回溯** | 全量快照還是增量?能否比對、回溯?回溯語意? |
| R6 | **快取** | 有無快取層?快取什麼? |
| R7 | **Object storage / 附件** | 附件走檔案系統 / DB / 物件儲存(S3)?文字與二進位是否分離? |
| R8 | **全文檢索 FTS** | 引擎(Lucene / OpenSearch…)?索引怎麼管理? |

貫穿主軸:**R2(協作演算法)與 R1(文件模型)互相決定**。

---

## 3. 兩系統取證(公開一手來源;逐字引文見附錄)

### 3.1 Google Docs

- **R2 協作演算法 —— OT,文件 = 操作日誌(定論)**
  - Google 工程一手來源:2010 年官方部落格說明文件被存成「一串按時序的變更」(如 `{InsertText 'T' @10}`,即 §1.3 例②),版本是「把變更歷史往前播放」算出來的,合併演算法就是 OT〔G1〕。
  - Google 專利描述伺服器接收 mutations、以 OT 解衝突〔G2〕;2013 年 Drive Realtime API 官方文章直說「based on operational transformation (OT)」〔G3〕。
  - **反證搜尋**(刻意找「Google Docs 其實用 CRDT」)**回空** —— 業界指出改用 CRDT 的是 *Figma* 而非 Google。→ OT 定論成立。
- **R3 傳輸 + R4 presence**
  - presence 是一等公民:Realtime API 追蹤誰連線,join/leave/change 都有事件〔G3〕;衝突由 OT 自動合併,使用者無感。
  - **傳輸線材:UNKNOWN**(歷史上用 BrowserChannel/long-poll,現代觀察傾向 WebSocket,但無 Google 一手來源坐實)→ 不臆測。
- **R1/R5 儲存/版本**
  - 模型層即「播放變更歷史」= op log〔G1〕;Drive API 佐證 revision 歷史連續累積、purgeable revision 保留約 30 天〔G4〕。
  - **實體後端(Spanner/Bigtable/Colossus):UNKNOWN** —— 只有二手部落格聲稱,不採信為 fact。
- **R6/R7/R8**:內部快取/儲存後端/FTS 管線均未公開 → **UNKNOWN**(closure recipe 見 §4)。**離線**:官方支援(Chrome/Edge + 擴充)〔G5〕,reconcile 走 OT(推論)。

### 3.2 Confluence(Server / Data Center)

- **R2 協作演算法 —— Synchrony 即時服務;演算法官方未明說**
  - Atlassian 官方:Synchrony 即時同步任意資料模型、對 WYSIWYG 編輯器有特化支援(含 telepointers);「content data 存在 Synchrony 服務、由它作為頁面內容的 source of truth」;使用者共編一份 **shared draft**〔C1、C2〕。
  - **反證搜尋**(「Synchrony 是 OT 還是 CRDT」):官方**沒有明說** → 標 **UNKNOWN**;「伺服器權威 + buffered-then-merge」形態偏**集中式 OT 風格**,但這是 inference。
- **R3 傳輸**
  - WebSocket 為主(以 JWT + contentId 開 session),連不上**退回 XHR**〔C1、C2〕。
  - Synchrony 是**獨立程序/獨立 JVM**,預設 port 8091〔C1、C3〕。
- **R1/R5 儲存/版本**
  - 頁面內容 = XHTML 標記 blob(§1.3 例①,技術上是 XML、含 macro 自訂元素),存關聯式 DB 的 `BODYCONTENT` 表;版本資訊在 `CONTENT` 表;支援 PostgreSQL / MySQL / Oracle / SQL Server〔C4、C5〕。
  - 每次修改建新版本、可 diff、可回溯;「回溯 = 複製舊版成新的當前版」〔C6〕。
- **R7 附件**:預設存 home 的 `attachments` 檔案系統;Data Center 8.1+ 可改存 **Amazon S3**(僅附件);DB/WebDAV 是 legacy 不再支援〔C7〕。**文字/二進位分離。**
- **R8 FTS**:預設**內嵌 Lucene**(索引在本機 home);Data Center 9.0+ 可選 **OpenSearch**(外部叢集、藍綠重建)〔C8〕。
- **R6 快取**:Data Center 有 Hazelcast 類叢集快取(Hazelcast:Java 的分散式記憶體快取框架),本次未取一手引文 → 未取證。

---

## 4. 依證據重新確認 rubric

1. **R2 與 R1 的耦合被證據強化,升為主軸** —— Docs 的 op log 讓字元級 OT 成立;Confluence 的 blob 模型使即時協作只能外掛(Synchrony),停在草稿級。
2. **R6 快取對兩者皆不可 close** —— 內部未公開,誠實標 unknown,不硬填。
3. 其餘 R3/R4/R5/R7/R8 量測有效。

**未能關閉的 unknown + closure recipe:**
- **Google Docs 傳輸/儲存後端/FTS 內部**:唯一可靠關閉方式是 Google 官方一手發文;在此之前任何「WebSocket / Spanner / Bigtable」宣稱只能標 inference/secondary。
- **Confluence Synchrony 的 OT/CRDT**:反編譯 Synchrony JAR / Atlassian 一手聲明;在此之前維持 UNKNOWN。

---

## 5. 依 rubric 建比較表(核心產物)

> 圖例:**Fact** = 官方一手引文可稽(附錄〔G#〕〔C#〕);*Inference* = 有據推論;**UNKNOWN** = 官方未公開。

| Rubric 面向 | **Google Docs** | **Confluence**(Server/Data Center) |
|---|---|---|
| **R1 資料庫 + 文件模型** | **操作日誌**(§1.3 例②),如 `{InsertText 'T' @10}`(Fact〔G1〕)。實體後端 **UNKNOWN**。 | **XHTML 標記 blob**(§1.3 例①)存**關聯式 DB** `BODYCONTENT` 表(Fact〔C4、C5〕);支援 PG/MySQL/Oracle/SQL Server。 |
| **R2 協作演算法** | **OT**,伺服器權威排序(Fact〔G1、G2、G3〕)。字元級。反 CRDT 搜尋回空。 | **Synchrony 即時服務**,伺服器為 source of truth、buffered-then-merge(Fact〔C1、C2〕)。演算法 **UNKNOWN**;形態偏 *集中式 OT-style*(Inference)。草稿級。 |
| **R3 即時傳輸層** | **UNKNOWN**(歷史 long-poll,現代疑似 WebSocket,未一手證實)。 | **WebSocket 為主、XHR 退回**;獨立 JVM 程序、預設 port 8091(Fact〔C1、C2、C3〕)。 |
| **R4 前端多人呈現 + 衝突** | **字元級即時**;presence 追蹤 join/leave/change(Fact〔G3〕)。衝突 OT 自動合併,無感。 | **共享草稿 + telepointers**(Fact〔C1、C2〕)。衝突由 Synchrony 同步 + buffered merge。 |
| **R5 版本歷史 / diff / 回溯** | 連續 revision(播放歷史);purgeable revision ~30 天(Fact〔G1、G4〕)。 | 每次修改建新版本,可 diff、可回溯;回溯 = 複製舊版為新當前版(Fact〔C6〕)。 |
| **R6 快取** | **UNKNOWN**。 | 叢集快取(Hazelcast 類),未取一手引文。 |
| **R7 Object storage / 附件** | **UNKNOWN**(Drive 底層未由 Docs 文件公開)。 | 附件預設檔案系統;**S3**(DC 8.1+,僅附件)(Fact〔C7〕)。**文字/二進位分離。** |
| **R8 全文檢索 FTS** | **UNKNOWN**。 | **Lucene 內嵌**(預設);**OpenSearch**(DC 9.0+)(Fact〔C8〕)。 |

---

## 6. 架構取捨定位 + wrapup

### 6.1 各站在什麼位置、為什麼

**Google Docs —— 「協作即時性」這一軸的工程極致,代價是把文件重新定義成操作日誌。**
它的核心洞察〔G1〕:**別把文件當 blob 比對版本,把文件當成「一串可重放的操作」**。一旦文件本體就是 op log,字元級 OT 合併、無感衝突解決、即時游標都自然成立。代價:需要伺服器做操作權威排序,且實體儲存/傳輸/搜尋內部從不公開 —— 協作能力拉滿、內部細節封閉的消費級標竿。

**Confluence —— 「企業 wiki + 即時協作」的折衷,把即時層與落地層拆開。**
它保留「一份 XHTML blob 存關聯式表」的傳統 wiki 落地方式,另掛**獨立程序 Synchrony** 做即時同步與 telepointer。得到:企業級可部署性(自選 DB、Lucene/OpenSearch、S3 附件、完整版本 diff)+ 即時共編。代價:即時協作是**加掛服務**而非模型本質,停在草稿級,演算法甚至不公開。

### 6.2 貫穿主軸的驗證:文件模型決定協作演算法
- **op log(Docs)→ 字元級 OT** 自然成立,衝突解決內建於模型。
- **整份 blob(Confluence)→ 外掛即時服務** 才有即時協作,且停在草稿級。
**即時協作不是加功能,是選文件模型。**

### 6.3 讀者可帶走的一句話
設計協作編輯器先問**文件模型**:要字元級即時就得像 Docs 把文件做成操作日誌(工程最重);要企業 wiki + 即時但落地簡單,就學 Confluence 把即時層做成獨立服務、與 blob 儲存解耦。

---

## 7. 延伸追問:能不能直接拿 FTS 或 S3 當「可編輯文件」的主存放?

這是上面比較的自然延伸題。答案的關鍵在一個共通點:**S3 物件與 Lucene 索引段(segment)都是 immutable(不可就地修改)的基底** —— 而「可編輯」的本質是「能便宜地改一小塊」。

### 7.1 S3:可以當 save-based 主存,但即時協作要自己鋪 log

- **S3 物件不可部分更新**:每次編輯 = 整份重新 PUT。所以它天然只支撐 **save-based**(§1.3 例①的整份 blob 模式),不可能直接做字元級即時協作。
- **但 save-based 所需的兩塊拼圖,S3 現在原生都有**(Fact〔S1、S2〕):
  - **版本歷史**:S3 物件版本化(object versioning)免費給你「每次存檔一版」。
  - **樂觀鎖**:2024-11 起 S3 支援 conditional write —— PUT 帶 `If-Match: <ETag>`,若物件已被別人改過(ETag 不符)回 **412 Precondition Failed**。這就是 compare-and-swap(CAS),與 save-based 編輯器「baseVersion 不符回 409」是同一個模式,只是狀態碼不同。
  - → 一個「wiki 頁 = 一個 S3 物件」的 save-based 編輯器,**現在可以不需要資料庫**就做出版本史+撞版偵測(*Inference:可行性推論,主流產品尚未見此架構*)。
- **要在 S3 上做到「可編輯」的進階招:在 immutable 物件上鋪一層日誌** —— 這正是 lakehouse 表格式(Delta Lake / Apache Iceberg)的做法:資料檔 immutable,「編輯」寫進 transaction log,讀取時播放 log 得到現況。**結構上就是 Google Docs 的 op-log 招式搬到 S3 上**(*Inference:類比*)。
- 實務定位:S3 適合當**附件/二進位層**(Confluence DC 8.1+ 正是如此〔C7〕)或 save-based 文件的 blob 層;**熱編輯路徑**(頻繁小改)放 S3 會付出整份重寫 + 請求延遲的代價。

### 7.2 FTS:不要當主存,當 derived index

- **Lucene 的索引段 immutable**:「更新一份文件」實際是把整份文件重新分析、寫進新 segment,舊的標記刪除、之後合併清理。對「頻繁小改」的編輯工作負載,這是**最大化的寫入放大**。
- 再加上:近即時(refresh 間隔)的可見性延遲、無交易語意、mapping 變更要全量重建索引 —— 把**可編輯的正本**放在 Elasticsearch/OpenSearch 是公認的反模式;它們的設計定位是**衍生索引(derived index)**。
- 業界慣例正是本文的兩個案例:**正本在 DB,FTS 是索引** —— Confluence 正本在 `BODYCONTENT`(關聯式),Lucene/OpenSearch 只是索引〔C4、C8〕。
- 例外情境(非編輯器):log/事件這類**一寫不改**的資料,以 ES 當唯一存放是可行的 —— 因為那正好避開了「編輯」。

### 7.3 一句話收束
**S3 = 可以當 save-based 的文件 blob 主存(版本化 + If-Match 樂觀鎖都原生),但字元級即時協作得自己在上面鋪 op log;FTS = 永遠當衍生索引,不當正本。** 兩者的 immutable 基底把你推回本文主軸:要嘛整份重寫(save-based),要嘛鋪一層操作日誌(Docs 的招)。

---

## 附錄 A — 一手來源逐字引文(擷取 2026-07-16;〔S#〕為 2026-07-18)

### Google Docs

- **〔G1〕Google Drive Blog,John Day-Richter(Google 工程師),2010-09-21**(https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_21.html):
  - 「The new version of Google documents does things differently. In the new editor, **a document is stored as a series of chronological changes.**」
  - 「A change might be something like `{InsertText 'T' @10}`. That particular change was to insert the letter T at the 10th position in the document.」
  - 「instead of computing the changes by comparing document versions, we now **compute the versions by playing forward the history of changes.**」
  - 「Tomorrow's post will give an overview of the algorithm for merging changes — **operational transformation.**」
- **〔G2〕Google 專利 US 9,720,897 B2**(受讓人 Google Inc.;https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9720897):「Mutations representing spreadsheet edit operations are received at a server …」「…**operational transform** techniques that can be used to **resolve conflicts** between such mutations…」
- **〔G3〕Google Developers Blog(Brian Cairns,2013),Drive Realtime API**(https://developers.googleblog.com/build-collaborative-apps-with-google-drive-realtime-api/):「Because the Drive Realtime API **is based on operational transformation (OT)**, local changes are reflected instantly, even on high-latency networks.」;「the Drive Realtime API also **keeps track of who is connected** … provides your app with events for when collaborators **join, leave, or make changes.**」
- **〔G4〕Drive API「Manage file revisions」**(https://developers.google.com/workspace/drive/api/guides/manage-revisions):Docs 累積連續 revision 歷史;purgeable revision 保留約 30 天。
- **〔G5〕Docs 離線**(https://support.google.com/docs/answer/6388102):官方支援 Chrome/Edge + Docs Offline 擴充。

### Confluence

- **〔C1〕Collaborative editing for Confluence Server**(https://developer.atlassian.com/server/confluence/collaborative-editing-for-confluence-server/):「Synchrony is a service that allows the synchronisation of arbitrary data models in real time. It supports special synchronisation for HTML WYSIWYG editors, **including telepointers (remote selections).**」;「content data will be stored on the Synchrony service and the service will **act as the source of truth for page content.**」;「opens a **WebSocket** session to Synchrony using the provided JWT and the `contentId`」;Synchrony「executed as a **separate process**」「runs in a **separate JVM**」。
- **〔C2〕Administering collaborative editing**(https://confluence.atlassian.com/doc/administering-collaborative-editing-858772086.html):「Collaborative editing is powered by Synchrony which **synchronizes data in real time.**」;「edit a **shared draft** of a page at the same time, and see each others' changes in real time」;「If your users cannot get a WebSocket connection, Confluence will **fall back to an XML HTTP Request (XHR)**」。
- **〔C3〕Configuring Synchrony**(https://confluence.atlassian.com/doc/configuring-synchrony-858772125.html):預設 **port 8091**。
- **〔C4〕Confluence storage format**(https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html):「the **XHTML-based format** that Confluence uses to store the content of pages …」「Technically, it's **XML**, since the storage format doesn't fully comply with the XHTML definition. For example, Confluence includes custom elements for macros」。
- **〔C5〕Confluence data model**(https://confluence.atlassian.com/doc/confluence-data-model-127369837.html):`BODYCONTENT`「The content of Confluence pages. No version information … That is all in the `content` table.」;`CONTENT`「persistence table for the `ContentEntityObject`」。支援 DB:PostgreSQL / MySQL / Oracle / SQL Server(Atlassian supported-platforms doc,2026-07-16)。
- **〔C6〕Page history & comparison**(https://confluence.atlassian.com/doc/page-history-and-page-comparison-views-139379.html):「creating a **new version** of the page each time it's modified」;可 diff、可回溯;「restoring an older version **creates a copy** of that version」。
- **〔C7〕Attachment storage**(https://confluence.atlassian.com/doc/attachment-storage-configuration-166876.html + Configuring S3 doc):「By default, Confluence stores attachments in the `attachments` directory within the … Confluence home folder.」;「Starting from Confluence 8.1, you can also **store your attachment data on Amazon S3** object storage.」;「S3 object storage is **for attachment data only.**」;DB/WebDAV 儲存「no longer supported」。
- **〔C8〕OpenSearch for Confluence Data Center**(https://confluence.atlassian.com/enterprise/opensearch-for-confluence-data-center-1653834676.html):「By default, Confluence uses **Lucene**.」「You can use **OpenSearch** starting from Confluence 9.0.」Lucene 索引在本機 home `index` 目錄;OpenSearch 由外部叢集管理、藍綠重建。

### S3 conditional writes(§7 用;擷取 2026-07-18)

- **〔S1〕AWS What's New,2024-11「Amazon S3 adds new functionality for conditional writes」**(https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-functionality-conditional-writes/):S3 可在寫入前檢查物件是否未被修改 —— PutObject / CompleteMultipartUpload 帶 ETag 條件。
- **〔S2〕AWS S3 官方文件「How to prevent object overwrites with conditional writes」**(https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html):`If-Match` 帶 ETag,不符則寫入失敗回 **412 Precondition Failed** —— 即 compare-and-swap 式的樂觀並發控制。

## 附錄 B — 誠實聲明(honest negatives)
- Google Docs 的**傳輸線材、實體儲存後端、FTS 內部**,以及 Confluence Synchrony 的 **OT/CRDT 演算法**,官方未公開一手 → 一律標 UNKNOWN + closure recipe(§4),未以二手部落格冒充 fact。
- §7 中「S3-only 的 save-based 編輯器」是可行性推論(未見主流產品採此架構);Delta/Iceberg 與 op-log 的類比是結構類比,非產品比較。
