# 線上協作文件編輯器設計比較:Google Docs vs Confluence

> 來源:2026-07-16 的 read-only 調查(所有外部引文擷取日期 = 2026-07-16;皆為官方一手來源)。
> 正文維持 finding 高度;原始逐字引文全部收在附錄,以〔G#〕〔C#〕anchor 引用。

---

## 0. 先給答案(Answer-first)

| 系統 | 一句話定位 | 協作核心 | 儲存核心 |
|------|-----------|---------|---------|
| **Google Docs** | 字元級即時協作的工程極致:文件**本體就是一串操作日誌**,靠 OT 演算法在伺服器權威排序下即時合併 | **OT**(Operational Transformation,操作轉換)+ 伺服器權威 | 變更日誌(mutation/changelog)模型,實體後端未公開 |
| **Confluence** | 企業 wiki 的即時協作:獨立 Synchrony 服務做即時同步,但**文件落地仍是一份 XHTML 標記存進關聯式 DB** | **Synchrony 即時服務**(演算法 OT/CRDT 官方未明說)+ 伺服器為 source of truth | XHTML 標記存 `BODYCONTENT` 關聯式資料表 |

**兩句話總結取捨:**
1. **協作即時性**:Google Docs 是字元級 OT、衝突無感自動合併;Confluence 是共享草稿級的 Synchrony 服務、有 telepointer(遠端游標)但即時層是**外掛的獨立程序**,不是文件模型的本質。
2. **文件模型決定協作演算法**:Docs 把文件做成「操作序列」(op log),字元級合併因此自然成立;Confluence 保留「整份 XHTML blob 存關聯式 DB」的傳統 wiki 落地,即時協作只能靠加掛服務補上 —— 模型愈接近操作序列,即時協作愈自然;愈接近整份 blob,愈需要外掛即時層。

---

## 1. 定義問題(Problem definition)

### 1.1 目的
把兩個線上協作編輯器標竿放在同一組設計面向下解析:同樣是「多人寫文件」,為什麼消費級的 Google Docs 與企業級的 Confluence 在**儲存、協作、檢索**上做了截然不同的取捨。

### 1.2 要回答的具體問題
- **Q1 儲存**:文件存進哪種資料庫?文件模型(整份 blob / 字元序列 / block-tree / 操作日誌)?快取?有無 object storage 分離二進位?全文檢索(FTS)引擎與索引?
- **Q2 協作**:演算法(OT / CRDT / lock / save-based)?即時傳輸層(WebSocket / long-poll / SSE)?前端多人呈現(presence 游標 / 即時字元)?衝突怎麼解?
- **Q3 其他考題**:版本歷史 / diff / 回溯;離線編輯;擴展性;文件模型如何反過來決定協作演算法。

### 1.3 名詞先講清楚(縮寫首現全名 + 用途)
- **OT(Operational Transformation,操作轉換)**:一種多人協作演算法。每個使用者的編輯被表示成「操作」(如「在第 10 個字元插入 T」),當兩人同時改、操作到達順序不同,系統把後到的操作「轉換」成基於已套用操作的等價操作,讓所有端最終收斂到同一份文件。**通常需要一台伺服器決定操作的權威順序**。
- **CRDT(Conflict-free Replicated Data Type,無衝突複製資料型別)**:另一種協作演算法。把文件設計成「不管操作以什麼順序到達、合併結果都一樣」的資料結構,因此**理論上不需要中央伺服器仲裁**;代價是每個字元通常要背負額外中繼資料。
- **FTS(Full-Text Search,全文檢索)**:對文件正文建索引以支援關鍵字/語意搜尋。
- **telepointer / presence / awareness**:即時顯示「其他人現在游標/選取範圍在哪、誰在線上」的機制。

---

## 2. 定義比較 rubric(先立後量)

| # | Rubric 面向 | 判準 |
|---|-------------|------|
| R1 | **資料庫種類與文件模型** | 整份 blob、字元序列、block-tree 還是操作日誌?模型是否支撐所選協作演算法? |
| R2 | **協作演算法** | OT / CRDT / lock?字元級還是草稿級?伺服器權威還是去中心? |
| R3 | **即時傳輸層** | WebSocket / long-poll / SSE?有無專用即時服務/程序? |
| R4 | **前端多人呈現與衝突處理** | presence 游標?即時字元?衝突怎麼解? |
| R5 | **版本歷史 / diff / 回溯** | 全量快照還是增量?能否比對、回溯?回溯語意? |
| R6 | **快取** | 有無快取層?快取什麼? |
| R7 | **Object storage / 附件** | 附件走檔案系統 / DB / 物件儲存(S3)?文字與二進位是否分離? |
| R8 | **全文檢索 FTS** | 引擎(Lucene / Elasticsearch / OpenSearch…)?索引怎麼管理? |

貫穿主軸:**R2(協作演算法)與 R1(文件模型)互相決定** —— 模型愈接近「操作序列」愈能撐字元級即時協作,愈接近「整份 blob」就愈需要外掛即時服務。

---

## 3. 兩系統取證(公開一手來源;逐字引文見附錄)

### 3.1 Google Docs

- **R2 協作演算法 —— OT,文件=變更日誌(定論)**:Google 工程一手來源明確 —— 2010 年官方部落格說明新版編輯器把文件存成「一串按時序的變更」(如 `{InsertText 'T' @10}`),版本是「把變更歷史往前播放」算出來的,合併演算法就是 operational transformation〔G1〕;Google 專利描述伺服器接收 mutations 並以 OT 技術解衝突〔G2〕;2013 年 Drive Realtime API 官方文章直說「based on operational transformation (OT)」〔G3〕。**反證搜尋**(刻意找「Google Docs 其實用 CRDT」)**回空** —— 業界指出改用 CRDT 的是 *Figma* 而非 Google。→ OT 定論成立。
- **R3 傳輸 + R4 presence**:presence 是一等公民 —— Realtime API 追蹤誰連線、join/leave/change 都有事件〔G3〕;衝突由 OT transform 自動合併,使用者無感。**傳輸線材:UNKNOWN**(歷史上用 BrowserChannel/long-poll,現代觀察傾向 WebSocket,但無 Google 一手來源坐實)→ 不臆測。
- **R1/R5 儲存/版本**:模型層即「播放變更歷史」= mutation log〔G1〕;Drive API 佐證 Docs 累積龐大連續 revision 歷史、purgeable revision 保留約 30 天〔G4〕。**實體後端(Spanner/Bigtable/Colossus):UNKNOWN** —— 只有二手系統設計部落格聲稱,不採信為 fact。
- **R6/R7/R8**:Google 未公開 Docs 內部快取拓撲、儲存後端與 FTS 內部管線 → **UNKNOWN**(closure recipe 見 §4)。**離線**:官方支援(Chrome/Edge + 擴充)〔G5〕,reconcile 走 OT(基於變更帶 base revision,推論)。

### 3.2 Confluence(Server / Data Center)

- **R2 協作演算法 —— Synchrony 即時服務;演算法官方未明說**:Atlassian 官方描述 Synchrony 是即時同步任意資料模型的服務、對 WYSIWYG 編輯器有特化支援(含 telepointers 遠端選取),且「content data 存在 Synchrony 服務、由它作為頁面內容的 source of truth」;使用者共編一份 **shared draft**、即時看到彼此變更〔C1、C2〕。**反證搜尋**(「Synchrony 是 OT 還是 CRDT」):官方文件**沒有明說演算法** → 標 **UNKNOWN**;「伺服器權威 + buffered-then-merge」的形態偏向**集中式 OT 風格**,但這是 inference,非 Atlassian 一手 fact。
- **R3 傳輸**:WebSocket 為主(以 JWT + contentId 開 session),連不上時**退回 XHR**;Synchrony 是**獨立程序/獨立 JVM**,預設 port 8091〔C1、C2、C3〕。
- **R1/R5 儲存/版本**:頁面內容是 **XHTML 標記(技術上是 XML,含 macro 自訂元素)**存進關聯式 DB 的 `BODYCONTENT` 表,版本資訊在 `CONTENT` 表;支援 PostgreSQL / MySQL / Oracle / SQL Server〔C4、C5〕。每次修改建新版本、可 diff、可回溯,「回溯 = 複製舊版成新的當前版」〔C6〕。
- **R7 附件**:預設存 home 目錄的 `attachments` 檔案系統;Data Center 8.1+ 可改存 **Amazon S3**(僅附件資料);DB/WebDAV 儲存是 legacy 不再支援〔C7〕。**文字/二進位分離。**
- **R8 FTS**:預設**內嵌 Lucene**(索引在本機 home);Data Center 9.0+ 可選 **OpenSearch**(外部叢集、藍綠重建)〔C8〕。
- **R6 快取**:Data Center 有 Hazelcast 類叢集快取,但本次未取一手引文 → 標為未取證(non-load-bearing)。

---

## 4. 依證據重新確認 rubric

1. **R2 與 R1 的耦合被證據強化,升為主軸** —— Docs 的「op log 模型」直接讓字元級 OT 成立;Confluence 的「blob 模型」使即時協作只能以外掛服務(Synchrony)實現,且停在草稿/區塊級。因果清楚,保留為結論骨架。
2. **R6 快取對兩者皆不可 close** —— 內部未公開一手,誠實標 unknown,不硬填。
3. 其餘 R3/R4/R5/R7/R8 量測有效。

**未能關閉的 unknown + closure recipe:**
- **Google Docs 傳輸線材 / 儲存後端 / FTS 內部**:Google 未公開。Closure recipe:唯一可靠關閉方式是 Google 官方工程一手發文或原始碼公開;在其未公開前,任何「WebSocket / Spanner / Bigtable」宣稱只能標 inference/secondary,不可升為 fact。
- **Confluence Synchrony 的 OT/CRDT**:Atlassian 官方未明說。Closure recipe:反編譯 Synchrony JAR / 取得 Atlassian 工程一手聲明 / 官方架構白皮書;在此之前維持 UNKNOWN,僅以「伺服器權威 + buffered-merge 形態」作 OT-style 的 inference。

---

## 5. 依 rubric 建比較表(核心產物)

> 圖例:**Fact** = 官方一手引文可稽(附錄〔G#〕〔C#〕);*Inference* = 有據推論;**UNKNOWN** = 官方未公開。

| Rubric 面向 | **Google Docs** | **Confluence**(Server/Data Center) |
|---|---|---|
| **R1 資料庫 + 文件模型** | 文件 = **一串時序 mutation(操作日誌)**,如 `{InsertText 'T' @10}`(Fact〔G1〕)。實體後端 **UNKNOWN**。 | **XHTML 標記(技術上 XML)** 存**關聯式 DB** `BODYCONTENT` 表(Fact〔C4、C5〕);支援 PG/MySQL/Oracle/SQL Server。文件模型 = 整份標記 blob。 |
| **R2 協作演算法** | **OT(操作轉換)**,伺服器權威排序、以 base revision 合併(Fact〔G1、G2、G3〕)。字元級。反 CRDT 搜尋回空。 | **Synchrony 即時同步服務**,伺服器為「source of truth」、buffered-then-merge(Fact〔C1、C2〕)。演算法 **OT/CRDT 官方未明說(UNKNOWN)**;形態偏 *集中式 OT-style*(Inference)。草稿/區塊級。 |
| **R3 即時傳輸層** | **UNKNOWN**(歷史 BrowserChannel/long-poll,現代疑似 WebSocket,Google 未一手證實)。 | **WebSocket 為主、XHR 退回**;Synchrony 為**獨立 JVM 程序**、預設 **port 8091**(Fact〔C1、C2、C3〕)。 |
| **R4 前端多人呈現 + 衝突** | **字元級即時**,他人游標/選取即時可見;presence 追蹤 join/leave/change(Fact〔G3〕)。衝突由 OT transform 自動合併,使用者無感。 | **即時共編共享草稿**,**telepointers(遠端選取)**顯示他人游標(Fact〔C1、C2〕)。衝突由 Synchrony 即時同步 + buffered merge 處理。 |
| **R5 版本歷史 / diff / 回溯** | 連續 revision 歷史(「play forward the history」);purgeable revision ~30 天(Fact〔G1、G4〕)。 | **每次修改建新版本**,可 diff 比對、可回溯;回溯 = 複製舊版成新當前版(Fact〔C6〕)。 |
| **R6 快取** | **UNKNOWN**(內部未公開)。 | Data Center 有叢集快取(Hazelcast 類),本次未取一手引文(未取證)。 |
| **R7 Object storage / 附件** | **UNKNOWN**(Drive 底層 blob 儲存未由 Docs 文件公開;實務上 Drive 檔案與文件分離)。 | 附件**預設檔案系統**(home/`attachments`);**S3 物件儲存**(Data Center 8.1+,僅附件資料);DB/WebDAV 為 legacy 不支援(Fact〔C7〕)。**文字/二進位分離。** |
| **R8 全文檢索 FTS** | **UNKNOWN**(Drive/Docs FTS 內部管線未公開一手)。 | **Lucene 內嵌**(預設,Server);**OpenSearch**(Data Center 9.0+ 選項,外部叢集、藍綠重建)(Fact〔C8〕)。 |

---

## 6. 架構取捨定位 + wrapup

### 6.1 各站在什麼位置、為什麼

**Google Docs —— 「協作即時性」這一軸的工程極致,代價是把文件重新定義成操作日誌。**
它的核心洞察(2010 那篇一手文〔G1〕)是:**別把文件當 blob 比對版本,把文件當成「一串可重放的操作」**。一旦文件本體就是 op log,字元級 OT 合併、無感衝突解決、即時游標就都自然成立。代價:需要一台伺服器做操作權威排序(Jupiter/client-server OT),而且實體儲存/傳輸/搜尋內部 Google 從不公開 —— 這是一個**協作能力拉滿、內部細節封閉**的消費級標竿。它最強(R2/R4),但對外可審計性最低(R3/R6/R7/R8 多為 UNKNOWN)。

**Confluence —— 「企業 wiki + 即時協作」的折衷,把即時層(Synchrony)和落地層(XHTML+關聯式 DB)拆開。**
它沒有把文件改成 op log,而是保留「一份 XHTML 標記存進 `BODYCONTENT` 關聯式表」的傳統 wiki 落地方式,另外掛一個**獨立程序 Synchrony**(WebSocket、獨立 JVM、port 8091)去做即時同步與 telepointer。這讓它同時擁有「企業級可部署性(選 PG/Oracle/SQL Server、Lucene/OpenSearch 搜尋、S3 附件、完整版本 diff)」與「即時共編」。取捨:即時協作是**加掛的服務**而非文件模型的本質,所以 Synchrony 是可獨立開關/會離線 buffer 的元件;演算法本身 Atlassian 甚至不公開(OT/CRDT 官方 UNKNOWN)。它是**部署彈性與企業功能最完整**的一個。

### 6.2 貫穿主軸的驗證:文件模型決定協作演算法
- **op log 模型(Google Docs)→ 字元級 OT** 自然成立,衝突解決內建於模型。
- **整份標記 blob(Confluence)→ 需外掛即時服務(Synchrony)** 才有即時協作,且停在草稿/區塊級。
兩者恰好示範「模型愈像操作序列、協作愈即時」:**即時協作不是加功能,是選文件模型**。

### 6.3 讀者可帶走的一句話
設計協作編輯器時先問**「文件模型」**:要**字元級即時協作**就得像 Google Docs 把文件做成操作日誌(工程最重、與傳統儲存最不相容);要**企業 wiki + 即時但落地簡單**就學 Confluence 把即時層做成獨立服務、與 XHTML/關聯式儲存層解耦(即時能力停在草稿級,換來部署彈性與功能完整度)。

---

## 附錄 A — 一手來源逐字引文(擷取 2026-07-16)

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

## 附錄 B — 誠實聲明(honest negatives)
- Google Docs 的**傳輸線材、實體儲存後端、FTS 內部**,以及 Confluence Synchrony 的 **OT/CRDT 演算法**,官方未公開一手 → 一律標 UNKNOWN + closure recipe(§4),未以二手部落格冒充 fact。
