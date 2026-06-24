# Wide Column vs Elasticsearch Doc Values

Wide Column Store 與 Elasticsearch Doc Values 都可以用「Key-Value 排列方向」來理解，但兩者解決的問題不同：

- **Wide Column**：以 Row / Partition 為中心，優化高吞吐寫入與依主鍵讀取。
- **Doc Values**：以 Field / Column 為中心，優化排序、聚合與 script 讀取欄位值。

---

## 1. 核心對照

| 特性維度 | Wide Column Store | Elasticsearch Doc Values |
| --- | --- | --- |
| 代表系統 | Cassandra、ScyllaDB、HBase、Bigtable | Elasticsearch / Lucene |
| 核心 Wrap-up | 以 `PartitionKey / RowKey` 為中心，將同一個 Row 或 Partition 內的資料組織在一起 | 以 `Field / Column` 為中心，將同一個欄位在不同 DocID 上的值組織在一起 |
| 切分方向 | 橫著切：Row-oriented / partition-oriented | 直著切：Column-oriented |
| 抽象 Key-Value 映射 | `(RowKey + ColumnKey) -> Value` | `(FieldName + SegmentDocID) -> Value` |
| 建立時機 | 寫入進 Commit Log / MemTable，之後 flush 成 SSTable | document index time 建立，隨 Lucene Segment 落盤 |
| 最擅長場景 | 高併發寫入、依 Partition Key 查一段資料 | sorting、aggregation、script access |
| 稀疏資料 | 沒寫入的 column 不需要存成 NULL value | 缺值會透過 Lucene encoding / metadata 表示，成本通常低，但不是邏輯上完全免費 |
| 壓縮特性 | 同一 Row 內可能有不同欄位與型別，壓縮效果取決於資料模型 | 同一欄位通常型別一致，適合 columnar encoding 與壓縮 |

---

## 2. 用 Key-Value 方向理解

假設有三份文件或三筆 Row：

```text
Doc 1 / Row A: age = 25, city = "Taipei"
Doc 2 / Row B: age = 30
Doc 3 / Row C: city = "Tokyo"
```

### Wide Column: Row 為中心

Wide Column 可以抽象成：

```text
"RowA:age"  -> 25
"RowA:city" -> "Taipei"
"RowB:age"  -> 30
"RowC:city" -> "Tokyo"
```

它的重點是：如果要查 `RowA`，同一個 Row / Partition 內相關資料可以一起被定位與掃描。

### Doc Values: Column 為中心

Doc Values 可以抽象成：

```text
"age:Doc1"  -> 25
"age:Doc2"  -> 30
"city:Doc1" -> "Taipei"
"city:Doc3" -> "Tokyo"
```

它的重點是：如果要對 `age` 做 aggregation，例如平均、百分位數、排序，就可以只讀 `age` 這個欄位的 columnar values，而不必把整份 `_source` 或其他欄位載入。

> 實際 Lucene Doc Values 是 per-segment 的 docID-to-value 結構。`DocID` 指的是 Lucene Segment 內部的 document id，不是 Elasticsearch `_id`。

---

## 3. 為什麼 Doc Values 適合聚合

倒排索引適合搜尋：

```text
term -> [docIDs]
```

例如：

```text
"error" -> [Doc1, Doc7, Doc9]
```

但排序與聚合需要的是反方向：

```text
docID -> field value
```

例如：

```text
Doc1 -> age = 25
Doc2 -> age = 30
Doc3 -> age = missing
```

如果沒有 Doc Values，Elasticsearch 早期需要把倒排索引反轉成 fielddata 放進 JVM Heap，容易造成 OOM。Doc Values 則在索引建立時就把欄位值以 on-disk columnar format 建好，查詢時主要依賴 OS filesystem cache，而不是把大量 fielddata 堆在 JVM Heap。

---

## 4. 不要混淆的地方

### Wide Column 不是分析型 Column Store

Wide Column 名字裡有 Column，但它不是 Snowflake、ClickHouse、Parquet 那種分析型 columnar database。它的核心仍然是根據 Partition Key / Clustering Key 讀寫一段資料。

適合：

- 查某個 conversation 的最新訊息。
- 查某個 device 某天的時間序列資料。
- 寫入大量 event / log / activity record。

不適合：

- 對所有使用者的 `age` 算平均。
- 任意欄位 ad-hoc aggregation。
- 大量跨 partition scan。

### Doc Values 不是主要搜尋索引

Doc Values 是為排序、聚合、script 與部分欄位存取設計的 columnar data structure。全文搜尋仍主要依賴倒排索引。

適合：

- `ORDER BY timestamp`
- `terms aggregation by status`
- `avg(price)`
- script 讀取欄位值

不適合取代：

- full-text search
- BM25 scoring
- analyzer/tokenizer 建出的 inverted index

---

## 5. 記憶口訣

- **Wide Column**：以 Row / Partition 為中心。先想「我要怎麼根據 Key 找到一段資料」。
- **Doc Values**：以 Field / Column 為中心。先想「我要怎麼快速掃某個欄位做排序與聚合」。

更短的版本：

```text
Wide Column: RowKey + ColumnKey -> Value
Doc Values : FieldName + DocID   -> Value
```

但要記得：Wide Column 的 `RowKey` 是資料建模與分片核心；Doc Values 的 `DocID` 是 Lucene Segment 內部 ID，主要服務排序、聚合與 scripts。

---

## 6. 參考

- [Elasticsearch doc_values](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/doc-values)
- [Lucene sparse versus dense document values](https://www.elastic.co/blog/sparse-versus-dense-document-values-with-apache-lucene)
