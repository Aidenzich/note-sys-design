### 什麼是 Recording Rules？

**Recording Rules** 允許你將頻繁需要計算、或是運算量龐大（耗時長）的查詢表達式，預先計算並儲存為一組新的「時間序列（Time Series）」。

### 為什麼它是處理「UUID/高基數標籤」的對策？

當你錯誤地將 UUID 放入標籤（Label）時，會產生數以萬計的微小序列，直接查詢這些原始數據會導致資料庫記憶體崩潰或查詢逾時。Recording Rules 在這裡扮演「預先聚合」的角色：

1.  **維度降級（Aggregation）：** 你可以設定一個規則，在後台定時將帶有 UUID 的原始指標進行聚合（例如使用 `sum` 或 `avg`），並在 `by` 子句中**排除掉 UUID 標籤**，只保留如 `service_name` 或 `endpoint` 等低基數標籤。
    
2.  **產生新指標：** 計算後的結果會被存成一個新的指標名稱（例如 `job:http_requests:rate5m`）。這個新指標不再包含 UUID 標籤。

3.  **加速查詢：** 之後你在 Grafana 面板或告警規則中，直接查詢這個「預先聚合過」的新指標，而非去掃描那堆帶有 UUID 的原始數據。

### 實際範例

假設原始指標 `http_request_duration_seconds_count` 包含了 `uuid` 標籤：

* **原始狀態（高基數）：**
    ```
    http_request_duration_seconds_count{service="orders", uuid="abc-123", endpoint="/pay"}
    http_request_duration_seconds_count{service="orders", uuid="def-456", endpoint="/pay"}
    ```
    （這裡可能有幾百萬行）

* **Recording Rule 設定：**
    ```yaml
    groups:
      - name: custom_rules 
        rules:
          - record: service:http_requests:rate5m 
            # (取速率) Prometheus 先抓取過去 5 分鐘內 http_request_duration_seconds_count 這個 Counter 的增長率，計算出「每秒平均請求數」。
            expr: sum by (service, endpoint) (rate(http_request_duration_seconds_count[5m]))
    ```

* **結果：**
    Prometheus 會每隔一段時間跑一次這個 `sum`，產生一個**不含 uuid** 的新指標：
    ```
    service:http_requests:rate5m{service="orders", endpoint="/pay"}
    ```
    
    這樣你在看 Dashboards 時，讀取速度會提升數十倍，且不會觸發 TSDB 的基數爆炸保護。

在架構設計中，Recording Rules 是**空間換取時間**與**計算下推（Push-down）**的典型應用。它無法阻止原始數據寫入磁碟，但它能確保監控系統在呈現與告警層面依然保持可用。
