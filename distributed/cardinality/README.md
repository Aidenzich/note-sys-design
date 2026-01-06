# Cardinality Issues Cheat Sheet for Senior Engineers (L5+)

在高併發與大規模分散式系統中，**基數（Cardinality）** 往往是性能退化的隱形殺手。這份清單整理了從資料庫到觀測性系統（Observability）常見的基數挑戰。


## 1. 資料庫與索引優化 (Databases & Indexing)
理解 B-Tree 與基數的關係是基礎，關鍵在於如何處理邊界案例。

* **索引選擇性 (Index Selectivity):**
    * **公式:** $Selectivity = \frac{Distinct Values}{Total Records}$
    * **問題:** 低基數欄位（如：`status`, `gender`）建立 B-Tree 索引通常無效，優化器可能改採全表掃描。
        - **B-Tree Node & Page Fragmentation:** 當欄位基數低（Low Cardinality）時，一個索引值會對應到大量的 RowIDs。如果這些資料在物理磁碟上的分佈是分散的，讀取這些 RowIDs 就會導致嚴重的 **Random I/O**。
        - **The Tipping Point (轉折點):**
            - **Index Scan:** 讀取 Index Page (Sequential) -> 獲取大量 RowIDs -> **大量 Random I/O** 訪問 Data Pages。
            - **Full Table Scan:** 直接進行 **Sequential I/O** (Multi-block Read)，一次讀取多個連續的 Data Pages 到 Buffer Pool。
            - **結論:** 當預估回傳行數超過總行數的 5%~20%（視資料庫而定）時，Random I/O 的尋址開銷會遠大於 Sequential I/O 的吞吐開銷，因此優化器主動放棄索引。
    * **對策:** 使用 **Partial Index**（針對特定狀態建索引）或考慮 **Bitmap Index**（若資料庫支援）。
* **複合索引順序 (Composite Index Ordering):**
    * **原則:** 通常將高基數欄位放在最左側，以利於過濾掉大部分資料。
    * **例外:** 若低基數欄位常被用於等值查詢（Equality check），而高基數用於範圍查詢（Range query），則順序需反轉。

## 2. 監控與觀測性系統 (Observability - Prometheus/M3/VictoriaMetrics)

這是目前最常發生「基數爆炸」的領域。

* **標籤爆炸 (Label/Tag Cardinality Explosion):**
    * **問題:** 在 Prometheus 指標中加入 `user_id`, `order_id` 或 `email` 作為 Label。
    * **後果:** 時序資料庫 (TSDB) 記憶體崩潰，索引查詢變慢。
    * **對策:** <span style="color: red;">嚴格禁止將 UUID 放入 Label。</span>
        * 使用 <span style="color: orange;">**Recording Rules**</span> 預先聚合。
        * 實施 <span style="color: orange;">**Relabeling**</span> 在採集端丟棄高基數標籤。
* **分位數計算 (Quantiles & Histograms):**
    * **問題:** `le` 標籤過多會導致維度急劇增加。
    * **對策:** 評估是否需要細粒度的分位數，或者改用 **Native Histograms** (Prometheus 2.40+)。

## 3. 分散式系統與大數據 (Distributed Systems & Big Data)

* **資料傾斜 (Data Skew):**
    * **問題:** 在 Join 或 Group By 時，Partition Key 的基數分佈不均（如：少數熱點 User ID 佔據 80% 資料）。
    * **對策:** * **Salting:** 給 Key 加上隨機後綴，強制打散分佈。
        * **Broadcast Join:** 如果一方資料量小，直接廣播避免 Shuffle。
* **唯一計數估算 (Distinct Count Estimation):**
    * **場景:** 需要計算 DAU 或大量唯一 ID 時，精確計算成本太高。
    * **對策:** 使用 **HyperLogLog (HLL)**。Redis, Presto, BigQuery 均內建此演算法，誤差通常在 1% 左右，但記憶體佔用極低。

---

## 4. 系統架構設計防禦

* **API 頻率限制 (Rate Limiting):**
    * **問題:** 以 `ip` 作為 Key 進行 Counter 計數，若遇到 DDoS，Redis 中的 Key 基數會瞬間爆炸。
    * **對策:** 設置 Key 的 TTL，或使用 **Sliding Window Log** 時配合基數上限熔斷。
* **快取穿透 (Cache Penetration):**
    * **問題:** 大量請求不存在的高基數 ID。
    * **對策:** 使用 **Bloom Filter** 攔截不存在的 Key。

---

## 5. 快速檢測工具清單

| 工具/指令 | 用途 |
| :--- | :--- |
| `ANALYZE TABLE` (SQL) | 更新統計資訊，查看 `cardinality` 欄位 |
| `topk` (PromQL) | 找出維度最高的指標名稱 |
| `redis-cli --bigkeys` | 找出佔用記憶體最多的 Key 或集合 |
| `hyperloglog` | 在不存儲原始值的情況下估算基數 |

---
> **L5 思考點:** 遇到效能問題時，第一時間檢查該維度的「基數」是否符合當初設計系統時的預期。如果基數會隨著用戶增長而無限線性增長，該設計即具備潛在風險。
