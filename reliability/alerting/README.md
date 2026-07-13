# Synthetic Monitoring 告警設計：從單次失敗到 M-of-N 與 SLO Burn Rate

Synthetic Monitoring（合成監控）會定期模擬使用者請求，從外部驗證 DNS、TLS、Gateway、反向代理與應用程式是否可用。它很適合發現「內部 metrics 看起來正常，但使用者其實連不上」的問題；然而，若將單次 probe failure 直接轉成通知，也很容易產生告警噪音。

本文從一個實際 staging 事件出發，整理業界常見的告警判定方式，以及一套可逐步演進到 Production SLO 的設計。

---

## 1. 問題：一分鐘的抖動，為什麼看起來故障了五分鐘？

### 1.1 觀察到的時序

系統每分鐘探測一次 endpoint，request timeout 為 8 秒。三次不同 endpoint 的事件都呈現相同模式：

| 時間 | Probe 結果 | 告警狀態 |
|:---|:---|:---|
| T | 約 8.2 秒 timeout | FIRING |
| T + 1m | HTTP 200 | 仍為 FIRING |
| T + 5m | HTTP 200 | RESOLVED |

這不是單純的 notification delivery latency，而是兩個機制疊加：

1. Alert rule 每分鐘評估一次，查詢最近 5 分鐘的資料。
2. 條件採 `at_least_once`：視窗內只要有一個失敗樣本就維持 FIRING。
3. Alertmanager 的 `group_wait` 與 `group_interval` 也可能進一步對齊通知時間。

因此，即使 endpoint 下一分鐘已恢復，舊的失敗樣本仍會在 rolling window 中停留約 5 分鐘。

### 1.2 Probe timeout 不等於 Service Process 掛掉

Public probe 測到的是完整路徑：

```text
Probe Host
  → DNS
  → TCP / TLS
  → Public Gateway / Nginx
  → SPA Dev Server 或 Reverse Proxy
  → Application Service
```

一次 timeout 只能證明「這條路徑未在 deadline 內完成」，不能直接證明最後面的 application process 不健康。

例如，某次 health endpoint 最後仍由 nginx 記錄為 HTTP 200，但 upstream 花了約 13.7 秒；8 秒 timeout 的 probe 已經先放棄。這代表 public proxy path 太慢，卻不足以證明直接連線到 service process 也會失敗。

---

## 2. 為什麼 Single-Failure Alert 容易吵？

設 probe 結果為：

```text
up = 1  → 成功
up = 0  → timeout、連線錯誤或非預期狀態碼
```

若規則是：

```text
min(up[5m]) < 1
```

它的語意是「最近五分鐘只要有一次失敗就告警」。這種 high-sensitivity 規則有幾個問題：

- 短暫 DNS、TCP、TLS 或 scheduler 抖動會直接通知。
- 一次失敗、下一次成功仍要等舊樣本離開視窗才恢復。
- 單一 probe host 的網路問題會被誤認為目標服務問題。
- Public route health 與 internal process health 混在同一個 service label。
- 當 probes 依序執行時，一個慢 endpoint 會延後後續 probes。

Single-failure 並非完全無用；它適合作為 event、dashboard annotation 或高敏感度訊號，但通常不應直接成為需要人處理的 Slack/Pager 告警。

---

## 3. 業界常見的四種判定方式

### 3.1 Consecutive Failures（連續失敗）

```text
連續失敗 N 次才 FIRING
連續成功 M 次才 RESOLVED
```

Kubernetes probe 的 `failureThreshold` / `successThreshold`，以及 Prometheus alert rule 的 `for`，都是這個思路。

| 優點 | 缺點 |
|:---|:---|
| 容易理解與實作 | 中間一次成功會重置計數 |
| 可消除單次抖動 | 對間歇性失敗不敏感 |
| Detection latency 可預估 | 可能漏掉「失敗、成功、失敗」型 degradation |

適合 process liveness 或明確的持續性故障。

### 3.2 M-of-N / Rolling Failure Rate

```text
最近 N 次樣本中，至少 M 次失敗
```

也可以寫成：

```text
failure_rate = failed_samples / valid_samples
FIRING when failure_rate >= X%
```

這正是 synthetic monitoring 常見的做法。它比純 consecutive failure 更能抓到間歇性錯誤：

```text
失敗 → 成功 → 失敗
```

即使沒有連續失敗，區間可用率仍明顯下降。

### 3.3 Multi-Probe Quorum（多探測點共識）

從多個獨立位置執行同一個 public check：

```text
Taipei probe  ─┐
Tokyo probe   ─┼→ 2 of 3 locations failed → alert
Singapore     ─┘
```

它能區分：

- 單一 probe host 或 ISP 故障。
- 區域性連線問題。
- 目標服務的全球性故障。

成熟的 public synthetic monitoring 通常不依賴單一探測點。

### 3.4 SLO Burn Rate

對 Production paging，單看 health endpoint 通常不夠。更成熟的做法是先定義：

```text
SLI = good requests / total eligible requests
SLO = 例如 30 天內 99.9% requests 成功
Error Budget = 1 - SLO
```

Burn rate 表示錯誤預算被消耗得多快：

```text
burn_rate = current_error_rate / sustainable_error_rate
```

再搭配 fast-burn 與 slow-burn 兩種視窗：

| 類型 | 目的 | 視窗特性 |
|:---|:---|:---|
| Fast burn | 快速發現大規模 user impact | 短視窗、高門檻 |
| Slow burn | 發現持續的小幅 degradation | 長視窗、低門檻 |

Synthetic probe 仍然有價值，但主要作為 black-box SLI 與診斷訊號，而不是 Production paging 的唯一依據。

---

## 4. 建議的 Staging 起始方案

若目前每分鐘只有一個樣本，可以先採取以下狀態機：

### 4.1 FIRING 條件

```text
Window: 最近 5 分鐘
Minimum valid samples: >= 4
Failed samples: >= 2
Failure rate: >= 40%
Latest sample: failed
```

`latest sample must be failed` 很重要。否則 endpoint 已恢復時，舊失敗仍可能讓區間失敗率超標，繼續製造過期告警。

### 4.2 RESOLVED 條件

```text
最近連續 2 次 probe 成功
```

FIRING 與 RESOLVED 使用不同門檻稱為 hysteresis。它能避免狀態在臨界值附近反覆切換，也不需要機械式等待整個 failure window 過期。

### 4.3 No Data 必須獨立處理

以下狀況不能被當成 endpoint success，也不應與 endpoint failure 混為一談：

- Probe scheduler 沒有執行。
- OTLP export 失敗。
- Metrics backend 無法 ingest。
- Alert query 本身失敗。

應建立獨立規則：

```text
ProbeTelemetryMissing:
  expected samples == 0 for 2-3 evaluation periods
```

### 4.4 邏輯示例

```python
def should_fire(samples):
    valid = deduplicate(samples)
    if len(valid) < 4:
        return False

    recent = valid[-5:]
    failures = sum(sample.up == 0 for sample in recent)
    failure_rate = failures / len(recent)

    return (
        recent[-1].up == 0
        and failures >= 2
        and failure_rate >= 0.40
    )


def should_resolve(samples):
    valid = deduplicate(samples)
    return len(valid) >= 2 and all(sample.up == 1 for sample in valid[-2:])
```

這組數值是合理起點，不是通用真理。門檻應根據 endpoint 重要性、probe frequency、正常 latency 分布與團隊可接受的 detection latency 調整。

---

## 5. Failure Rate 的資料陷阱

### 5.1 計算前必須先去重

Metrics backend 常將 sample table 與 time-series/resource attributes table 分開儲存。若直接 join，而 attributes table 對同一 fingerprint 存在多列，單一 probe sample 可能被展開多次。

```text
一個真實失敗 sample
  × 多筆相同 fingerprint attributes
  = query 中的多筆失敗列
```

`min(up)` 不受重複列影響，但以下計算會失真：

```sql
countIf(up = 0) / count()
```

應先以穩定的 sample identity 去重，例如：

```text
(fingerprint, sample_timestamp)
```

再計算：

```text
failed samples
valid samples
failure rate
latest sample state
```

### 5.2 必須保留失敗階段

只有 `up=0` 不足以診斷問題。建議同時保留：

| Field | 用途 |
|:---|:---|
| `observed` | timeout、HTTP 503、DNS error 等 |
| `duration_ms` | 判斷是否碰到 deadline |
| `error_phase` | DNS、connect、TLS、TTFB、body |
| `probe_location` | 判斷單一地點或全球問題 |
| `target` | Endpoint identity |
| `expected` | 預期狀態與 latency objective |

Duration 應是獨立 metric value，而不是高基數 label。

---

## 6. 將「可用路徑」與「Service Health」拆開

一個成熟的監控模型至少有四層：

| Layer | 量測內容 | 適合回答的問題 |
|:---|:---|:---|
| External synthetic | Public DNS/TLS/Gateway 完整路徑 | 使用者從外部能否存取？ |
| Internal synthetic | 內網 Gateway 或 service endpoint | 問題是否在外部入口？ |
| Process health | 直接 liveness/readiness | Process 是否活著、依賴是否就緒？ |
| Request SLI | 真實流量成功率與 latency | 使用者是否正在受到影響？ |

不要用「經過 SPA dev server / reverse proxy 的 public `/healthz`」代表 application process liveness。它應被命名為 public-route availability；process liveness 則應直接探測 service listener。

```text
Public route failed + Internal process healthy
  → DNS / TLS / Gateway / Proxy path 問題

Public route failed + Internal process failed
  → Service 或 Host 問題

Single probe location failed + Other locations healthy
  → Probe location / network path 問題
```

---

## 7. Alert Routing 與通知內容

Detection、incident identity 與 notification grouping 是不同問題，不應混在一起。

### 7.1 Incident Identity

Endpoint incident 至少應由以下 labels 識別：

```text
environment
service
probe_kind
endpoint
probe_location（多探測點時視需求納入）
```

若 Alertmanager 只使用：

```text
group_by = [alertname]
```

不同 service/endpoint 可能被粗略地併入同一通知群組。應依 incident identity 分組，同時避免把 timestamp、完整 error text 等高基數欄位放入 group key。

### 7.2 通知應包含

- FIRING / RESOLVED 狀態。
- Service、probe kind、endpoint。
- Failed / valid sample counts 與 failure rate。
- Latest observed result、duration 與 reason。
- Probe location。
- Alert rule 與 dashboard/runbook link。
- 首次失敗與最近失敗時間。

---

## 8. Production 演進路線

### Phase 1：降低噪音

- Single failure 只記錄，不通知。
- 使用 M-of-N + minimum sample count。
- 加入 recovery hysteresis。
- 將 no-data 拆成獨立規則。
- Alertmanager 改成 incident-level grouping。

### Phase 2：改善定位

- Public-path 與 direct process health 分離。
- Probes 改為並行或各自獨立排程。
- 增加第二個以上獨立 probe location。
- 記錄 DNS/connect/TLS/TTFB 等階段時間。

### Phase 3：導入 SLO Alerting

- 以真實 request success/latency 定義 SLI。
- 建立 28 或 30 天 SLO 與 error budget。
- 使用 multi-window fast-burn / slow-burn alerts。
- Synthetic monitoring 保留為 black-box validation 與 root-cause signal。

### Phase 4：用歷史資料校準

在正式取代舊規則前，先以 shadow mode 執行新規則並 replay 歷史資料，量測：

- 每週 alerts 數量。
- 單次抖動被抑制的比例。
- 真實 outage 的 detection latency。
- FIRING 後無需人工處理的比例。
- Alert acknowledgment 與實際 user impact 的關聯。

門檻應由資料校準，而不是只靠直覺決定。

---

## 9. 結論

「在區間中多次抽樣，失敗數量或比例超過門檻才判定失敗」是業界常見且成熟的方向。實務上通常稱為 M-of-N 或 rolling failure rate。

一套可靠的實作還必須補上：

1. Minimum sample count。
2. Latest sample state。
3. Recovery hysteresis。
4. No-data handling。
5. Sample de-duplication。
6. Multiple probe locations。
7. Public-path 與 process-health 分層。
8. Production SLO burn-rate alerting。

對低流量 staging，M-of-N 是務實的第一步；對需要 on-call paging 的 Production，最終目標應是以 user-impact SLI 與 error-budget burn rate 決定是否打擾人。

---

## 10. References

- [Grafana — Configure per-check alerts](https://grafana.com/docs/grafana-cloud/testing/synthetic-monitoring/configure-alerts/configure-per-check-alerts/)
- [Grafana — Handle connectivity errors in alerts](https://grafana.com/docs/grafana/latest/alerting/guides/connectivity-errors/)
- [Grafana — Synthetic Monitoring alerting](https://grafana.com/docs/grafana-cloud/synthetic-monitoring/synthetic-monitoring-alerting/)
- [Prometheus — Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Kubernetes — Configure liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Google Cloud — Alerting on your burn rate](https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring/alerting-on-budget-burn-rate)
