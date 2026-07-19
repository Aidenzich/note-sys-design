# 能不能直接拿 FTS 或 S3 當「可編輯文件」的主存放?

> 來源:2026-07-18 調查(S3 conditional writes 部分擷取於 2026-07-18)。
> 本文是[三系統比較](./README.md)的延伸追問:S3 物件與 Lucene segment 都是 immutable 基底,而「可編輯」的本質是「能便宜地改一小塊」。

---

答案的關鍵在一個共通點:**S3 物件與 Lucene 索引段(segment)都是 immutable(不可就地修改)的基底** —— 而「可編輯」的本質是「能便宜地改一小塊」。

### 1. S3:可以當 save-based 主存,但即時協作要自己鋪 log

- **S3 物件不可部分更新**:每次編輯 = 整份重新 PUT。所以它天然只支撐 **save-based**(§1.3 例①的整份 blob 模式),不可能直接做字元級即時協作。
- **但 save-based 所需的兩塊拼圖,S3 現在原生都有**(Fact[1](#ref-1)[2](#ref-2)):
  - **版本歷史**:S3 物件版本化(object versioning)免費給你「每次存檔一版」。
  - **樂觀鎖**:2024-11 起 S3 支援 conditional write —— PUT 帶 `If-Match: <ETag>`,若物件已被別人改過(ETag 不符)回 **412 Precondition Failed**。這就是 compare-and-swap(CAS),與 save-based 編輯器「baseVersion 不符回 409」是同一個模式,只是狀態碼不同。
  - → 一個「wiki 頁 = 一個 S3 物件」的 save-based 編輯器,**現在可以不需要資料庫**就做出版本史+撞版偵測(*Inference:可行性推論,主流產品尚未見此架構*)。
- **要在 S3 上做到「可編輯」的進階招:在 immutable 物件上鋪一層日誌** —— 這正是 lakehouse 表格式(Delta Lake / Apache Iceberg)的做法:資料檔 immutable,「編輯」寫進 transaction log,讀取時播放 log 得到現況。**結構上就是 Google Docs 的 op-log 招式搬到 S3 上**(*Inference:類比*)。
- 實務定位:S3 適合當**附件/二進位層**(Confluence DC 8.1+ 正是如此[3](#ref-3))或 save-based 文件的 blob 層;**熱編輯路徑**(頻繁小改)放 S3 會付出整份重寫 + 請求延遲的代價。

### 2. FTS:不要當主存,當 derived index

- **Lucene 的索引段 immutable**:「更新一份文件」實際是把整份文件重新分析、寫進新 segment,舊的標記刪除、之後合併清理。對「頻繁小改」的編輯工作負載,這是**最大化的寫入放大**。
- 再加上:近即時(refresh 間隔)的可見性延遲、無交易語意、mapping 變更要全量重建索引 —— 把**可編輯的正本**放在 Elasticsearch/OpenSearch 是公認的反模式;它們的設計定位是**衍生索引(derived index)**。
- 業界慣例正是本文的兩個案例:**正本在 DB,FTS 是索引** —— Confluence 正本在 `BODYCONTENT`(關聯式),Lucene/OpenSearch 只是索引[4](#ref-4)[5](#ref-5)。
- 例外情境(非編輯器):log/事件這類**一寫不改**的資料,以 ES 當唯一存放是可行的 —— 因為那正好避開了「編輯」。

### 3. 一句話收束
**S3 = 可以當 save-based 的文件 blob 主存(版本化 + If-Match 樂觀鎖都原生),但字元級即時協作得自己在上面鋪 op log;FTS = 永遠當衍生索引,不當正本。** 兩者的 immutable 基底把你推回本文主軸:要嘛整份重寫(save-based),要嘛鋪一層操作日誌(Docs 的招)。

---


---

## References

- <a id="ref-1"></a>**[1]**AWS What's New,2024-11「Amazon S3 adds new functionality for conditional writes」**(https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-s3-functionality-conditional-writes/):S3 可在寫入前檢查物件是否未被修改 —— PutObject / CompleteMultipartUpload 帶 ETag 條件。
- <a id="ref-2"></a>**[2]**AWS S3 官方文件「How to prevent object overwrites with conditional writes」**(https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html):`If-Match` 帶 ETag,不符則寫入失敗回 **412 Precondition Failed** —— 即 compare-and-swap 式的樂觀並發控制。
- <a id="ref-3"></a>**[3]**Attachment storage**(https://confluence.atlassian.com/doc/attachment-storage-configuration-166876.html + Configuring S3 doc):「By default, Confluence stores attachments in the `attachments` directory within the … Confluence home folder.」;「Starting from Confluence 8.1, you can also **store your attachment data on Amazon S3** object storage.」;「S3 object storage is **for attachment data only.**」;DB/WebDAV 儲存「no longer supported」。
- <a id="ref-4"></a>**[4]**Confluence storage format**(https://confluence.atlassian.com/doc/confluence-storage-format-790796544.html):「the **XHTML-based format** that Confluence uses to store the content of pages …」「Technically, it's **XML**, since the storage format doesn't fully comply with the XHTML definition. For example, Confluence includes custom elements for macros」。
- <a id="ref-5"></a>**[5]**OpenSearch for Confluence Data Center**(https://confluence.atlassian.com/enterprise/opensearch-for-confluence-data-center-1653834676.html):「By default, Confluence uses **Lucene**.」「You can use **OpenSearch** starting from Confluence 9.0.」Lucene 索引在本機 home `index` 目錄;OpenSearch 由外部叢集管理、藍綠重建。
