# CDN (Content Delivery Network)

CDN 是一個地理分散的伺服器網路，透過將內容快取到離用戶最近的節點，來加速內容傳輸並降低源站壓力。

## 1. 核心架構

```
                         ┌─────────────────┐
                         │   Origin Server │ (源站)
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
        ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
        │ PoP Asia  │       │ PoP Europe│       │ PoP US    │
        │ (Edge)    │       │ (Edge)    │       │ (Edge)    │
        └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
              │                   │                   │
       ┌──────┴──────┐     ┌──────┴──────┐     ┌──────┴──────┐
       │   Users     │     │   Users     │     │   Users     │
       │ (Asia)      │     │ (Europe)    │     │ (US)        │
       └─────────────┘     └─────────────┘     └─────────────┘
```

### 1.1 關鍵術語

| 術語 | 說明 |
|:---|:---|
| **PoP (Point of Presence)** | CDN 的邊緣節點機房位置 |
| **Edge Server** | 位於 PoP 的快取伺服器 |
| **Origin Server** | 原始內容的源站伺服器 |
| **Origin Shield** | 第二層快取，保護源站免受大量回源請求 |
| **Cache Hit/Miss** | 請求命中快取 / 需回源取得 |

---

## 2. 運作流程

### 2.1 首次請求 (Cache Miss)

```
1. User (台北) → DNS 查詢 cdn.example.com
2. DNS 返回最近的 PoP IP (東京)
3. User → Edge (東京): GET /image.jpg
4. Edge 檢查本地快取 → Miss
5. Edge → Origin (美國): GET /image.jpg
6. Origin 返回內容 + Cache-Control Header
7. Edge 快取內容
8. Edge → User 返回內容
```

### 2.2 後續請求 (Cache Hit)

```
1. User (大阪) → DNS 查詢 cdn.example.com
2. DNS 返回最近的 PoP IP (東京)
3. User → Edge (東京): GET /image.jpg
4. Edge 檢查本地快取 → Hit ✅
5. Edge → User 返回內容 ⚡ (不需回源)
```

---

## 3. 快取控制 (Cache Control)

### 3.1 HTTP Headers

| Header | 說明 | 範例 |
|:---|:---|:---|
| `Cache-Control` | 主要控制指令 | `max-age=3600, public` |
| `Expires` | 絕對過期時間 (已過時) | `Wed, 01 Jan 2025 00:00:00 GMT` |
| `ETag` | 內容指紋 | `"abc123"` |
| `Last-Modified` | 最後修改時間 | `Tue, 15 Nov 2022 12:45:26 GMT` |
| `Vary` | 根據哪些 Header 區分快取 | `Accept-Encoding` |

### 3.2 Cache-Control 指令

| 指令 | 說明 |
|:---|:---|
| `public` | 可被任何快取儲存 (CDN, Browser) |
| `private` | 只能被 Browser 快取，不給 CDN |
| `no-cache` | 每次需向源站驗證 (可快取但需確認) |
| `no-store` | 完全禁止快取 |
| `max-age=N` | 快取有效期 (秒) |
| `s-maxage=N` | 專給 CDN/Proxy 的 max-age |
| `stale-while-revalidate=N` | 過期後 N 秒內可先返回舊內容，背景更新 |

### 3.3 快取策略建議

| 資源類型 | 策略 | 原因 |
|:---|:---|:---|
| 靜態資源 (JS/CSS/Image) | `max-age=31536000` (1年) + 檔名帶 Hash | 永久快取，更新時改檔名 |
| HTML 頁面 | `no-cache` 或 `max-age=60` | 需要較即時更新 |
| API Response | `private, max-age=0` | 通常個人化，不應快取 |
| 公開 API | `public, max-age=60` | 短時間快取減少負載 |

---

## 4. 快取失效 (Cache Invalidation)

### 4.1 失效方式

| 方式 | 說明 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **TTL 過期** | 等待快取自然過期 | 簡單、無需操作 | 無法立即更新 |
| **Purge (清除)** | 主動刪除指定 URL 的快取 | 精確控制 | 需 API 呼叫 |
| **Soft Purge** | 標記為 stale，背景更新 | 不中斷服務 | 短暫返回舊內容 |
| **版本化 URL** | `file.js?v=2` 或 `file.abc123.js` | 最可靠 | 需修改引用處 |

### 4.2 常見問題

**問題一：快取時間過長，緊急更新無法生效**
- 解法：使用版本化 URL + 長 TTL

**問題二：Purge 請求延遲傳播**
- 解法：使用 **Soft Purge** + `stale-while-revalidate`

---

## 5. Origin Shield (二層快取)

### 5.1 沒有 Origin Shield

```
     Edge (Tokyo) ────┐
     Edge (Seoul) ────┼──→ Origin (US) ← 大量回源
     Edge (HK)    ────┘
```

### 5.2 有 Origin Shield

```
     Edge (Tokyo) ────┐
     Edge (Seoul) ────┤──→ Shield (LA) ──→ Origin (US)
     Edge (HK)    ────┘         │
                                 ↓
                          僅一次回源
```

**優點：**
- 減少源站負載 (多個 Edge 的 Miss 合併為一次)
- 降低回源延遲 (Shield 通常離 Origin 較近)
- 提高 Cache Hit Rate

---

## 6. CDN 類型

| 類型 | 說明 | 代表服務 |
|:---|:---|:---|
| **Push CDN** | 手動上傳內容到 CDN 節點 | S3 + CloudFront |
| **Pull CDN** | 第一次請求時自動從源站拉取 | CloudFlare, Akamai |
| **Peer-to-Peer** | 用戶間互相傳輸 (直播) | Twitch (部分), Apple FairPlay |

### Push vs Pull

| 特性 | Push CDN | Pull CDN |
|:---|:---|:---|
| **設定** | 需手動上傳 | 設定 Origin 即可 |
| **首次請求** | 立即命中 | 需回源 (稍慢) |
| **適用場景** | 靜態資源量固定 | 動態網站、資源量大 |
| **成本** | 儲存 + 傳輸 | 主要是傳輸 |

---

## 7. 進階主題

### 7.1 動態內容加速 (Dynamic Site Acceleration)

傳統 CDN 主要快取靜態內容，但動態內容也可受益：

| 技術 | 說明 |
|:---|:---|
| **TCP 優化** | CDN 與 Origin 保持長連線，避免三次握手 |
| **路由優化** | 使用私有骨幹網路 (如 CloudFlare Argo) |
| **Edge Computing** | 在邊緣運行部分邏輯 (如 CloudFlare Workers) |
| **Prefetch** | 預測性預載入 |

### 7.2 Anycast

```
傳統 Unicast:
cdn.example.com → 固定 IP → 可能很遠

Anycast:
cdn.example.com → 同一 IP 多處宣告 → BGP 自動路由到最近
```

**優點：**
- 自動地理分流
- 天然 DDoS 緩解 (攻擊分散到多個 PoP)

### 7.3 Edge Computing

| 服務 | 說明 |
|:---|:---|
| **CloudFlare Workers** | 在 Edge 運行 JavaScript |
| **AWS Lambda@Edge** | 在 CloudFront Edge 運行 Lambda |
| **Fastly Compute@Edge** | 在 Edge 運行 WebAssembly |

**典型用途：**
- A/B Testing (請求改寫)
- 認證驗證 (JWT 驗證)
- 圖片轉換 (WebP 自動轉換)
- Bot Detection

---

## 8. 主流 CDN 比較

| 服務 | PoP 數量 | 特色 | 適用場景 |
|:---|:---|:---|:---|
| **CloudFlare** | 300+ | 免費方案強大、Workers 生態 | 中小型網站、API |
| **AWS CloudFront** | 400+ | 與 AWS 深度整合 | AWS 用戶 |
| **Akamai** | 4000+ | 企業級、最成熟 | 大型企業 |
| **Fastly** | 70+ | 即時 Purge、VCL 靈活 | 內容頻繁更新 |
| **Google Cloud CDN** | 100+ | 與 GCP 整合 | GCP 用戶 |

---

## 9. Reference
- [CloudFlare Learning - What is a CDN?](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)
- [AWS CloudFront Developer Guide](https://docs.aws.amazon.com/cloudfront/)
- [HTTP Caching - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
