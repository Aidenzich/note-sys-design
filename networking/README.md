# 網路協定基礎 (Networking Fundamentals)

本章節涵蓋系統設計中常考的網路協定知識，包括 TCP/HTTP、DNS、和 WebSocket。

## 1. TCP/IP 核心概念

### 1.1 三次握手 (Three-Way Handshake)

```
    Client                    Server
       │                         │
       │────── SYN (seq=x) ─────→│  1. 客戶端發起連線
       │                         │
       │←── SYN-ACK (seq=y,      │  2. 伺服器確認並發起連接
       │    ack=x+1) ────────────│
       │                         │
       │────── ACK (ack=y+1) ───→│  3. 客戶端確認
       │                         │
       │    ════ 連線建立 ════    │
```

**為什麼是三次？**
- 確認雙方的發送和接收能力都正常
- 同步雙方的初始序列號 (ISN)
- 防止過期連線請求被誤接受

### 1.2 四次揮手 (Four-Way Handshake)

```
    Client                    Server
       │                         │
       │────── FIN ─────────────→│  1. 客戶端請求關閉
       │                         │
       │←────── ACK ─────────────│  2. 伺服器確認
       │                         │
       │        (伺服器可能還有資料要傳)
       │                         │
       │←────── FIN ─────────────│  3. 伺服器完成，請求關閉
       │                         │
       │────── ACK ─────────────→│  4. 客戶端確認
       │                         │
       │   ═══ TCP TIME_WAIT ═══ │  (等待 2MSL)
```

**TIME_WAIT 狀態：**
- 持續 2 × MSL (Maximum Segment Lifetime，通常 60 秒)
- 確保最後的 ACK 能送達
- 防止下一個連線收到舊封包

### 1.3 連線池 (Connection Pooling)

**問題：** 每次請求都建立新 TCP 連線開銷大

**解法：** 維護預建立的連線池，重複使用

```
┌────────────────────────────────────┐
│            Connection Pool          │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐       │
│  │Conn│ │Conn│ │Conn│ │Conn│ ...   │
│  │ #1 │ │ #2 │ │ #3 │ │ #4 │       │
│  └────┘ └────┘ └────┘ └────┘       │
└────────────────────────────────────┘
         │        │
    Request A  Request B
```

**關鍵參數：**
| 參數 | 說明 | 典型值 |
|:---|:---|:---|
| `min_pool_size` | 最小連線數 | 5-10 |
| `max_pool_size` | 最大連線數 | 20-100 |
| `connection_timeout` | 取得連線的等待時間 | 5s |
| `idle_timeout` | 閒置連線回收時間 | 60s |
| `max_lifetime` | 連線最大生命週期 | 30min |

---

## 2. HTTP 協定演進

### 2.1 版本比較

| 特性 | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|:---|:---|:---|:---|:---|
| **持久連線** | ❌ 每次請求新連線 | ✅ Keep-Alive | ✅ | ✅ |
| **Head-of-Line Blocking** | N/A | ⚠️ 有 | ✅ Stream 解決 | ✅ QUIC 解決 |
| **Header 壓縮** | ❌ | ❌ | ✅ HPACK | ✅ QPACK |
| **多工 (Multiplexing)** | ❌ | ❌ | ✅ | ✅ |
| **Server Push** | ❌ | ❌ | ✅ | ✅ |
| **傳輸層** | TCP | TCP | TCP | **UDP (QUIC)** |

### 2.2 HTTP/1.1 Head-of-Line Blocking

```
HTTP/1.1 Pipelining (理論上可以):
Request 1 ────→
Request 2 ────→  (不等 Response 1)
Request 3 ────→

但是 Response 必須依序返回:
←──── Response 1 (慢)
←──── Response 2 (等待中...)
←──── Response 3 (等待中...)

如果 Response 1 卡住，2 和 3 都被阻塞！
```

### 2.3 HTTP/2 Multiplexing

```
┌─────────────────────────────────────┐
│          Single TCP Connection       │
│  ┌─────────────────────────────────┐ │
│  │ Stream 1: Request A             │ │
│  │ Stream 2: Request B             │ │
│  │ Stream 3: Request C             │ │
│  └─────────────────────────────────┘ │
│                                       │
│  Frames interleaved:                  │
│  [A1][B1][A2][C1][B2][A3]...         │
└─────────────────────────────────────┘
```

**特點：**
- 單一連線承載多個 Stream
- Frame 可以交錯傳輸
- 每個 Stream 獨立，不會互相阻塞

### 2.4 HTTP/3 (QUIC)

**為什麼需要 QUIC？**
- HTTP/2 的 Head-of-Line Blocking 在 TCP 層仍存在
- TCP 丟包會阻塞所有 Stream
- QUIC 在 UDP 上實現可靠傳輸，每個 Stream 獨立處理丟包

```
HTTP/2 over TCP:
  TCP Packet Loss → ALL Streams blocked

HTTP/3 over QUIC:
  Stream 1 Packet Loss → Only Stream 1 retransmits
  Stream 2, 3 → Unaffected
```

---

## 3. DNS (Domain Name System)

### 3.1 解析流程

```
                        ┌───────────────────┐
                        │  Root DNS (.com)  │
                        └─────────┬─────────┘
                                  │
User ──→ Browser Cache ──→ OS Cache ──→ Local DNS ──→ TLD DNS (.com)
                                                          │
    ←── IP Address ←── Local DNS (Cache) ←── Authoritative DNS (example.com)
```

### 3.2 記錄類型

| 類型 | 用途 | 範例 |
|:---|:---|:---|
| **A** | 域名 → IPv4 | `example.com → 93.184.216.34` |
| **AAAA** | 域名 → IPv6 | `example.com → 2606:2800:...` |
| **CNAME** | 別名 → 正式名稱 | `www.example.com → example.com` |
| **MX** | 郵件伺服器 | `example.com → mail.example.com` |
| **TXT** | 任意文字 (驗證用) | SPF、DKIM、Domain Verification |
| **NS** | 指定 Name Server | `example.com → ns1.cloudflare.com` |
| **SRV** | 服務發現 | `_ldap._tcp.example.com` |

### 3.3 TTL (Time To Live)

| 場景 | 建議 TTL | 原因 |
|:---|:---|:---|
| 靜態網站 | 1 天 (86400s) | 減少 DNS 查詢 |
| 動態服務 | 5-15 分鐘 | 平衡快取與更新速度 |
| 計畫遷移前 | 降至 5 分鐘 | 確保快速切換 |
| Failover | 60 秒 | 快速故障轉移 |

### 3.4 GeoDNS / Anycast

**GeoDNS (Latency-based Routing):**
```
User (Tokyo) → DNS Query → GeoDNS
                              │
                              ├→ Tokyo Server IP (closest)
                              ├→ Singapore Server IP
                              └→ US Server IP
```

**Anycast:**
- 多個伺服器共享同一 IP
- BGP 路由自動導向最近的 PoP
- 常用於 CDN 和 DNS 服務本身

---

## 4. WebSocket

### 4.1 與 HTTP 的差異

| 特性 | HTTP | WebSocket |
|:---|:---|:---|
| **連線模式** | 請求-回應 | 全雙工持久連線 |
| **由誰發起** | 僅 Client | 雙向 |
| **開銷** | 每次請求帶完整 Header | 僅握手時有 Header |
| **適用場景** | REST API、靜態資源 | 即時通訊、遊戲、股票 |

### 4.2 握手過程

```http
# Client Request (Upgrade)
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

# Server Response
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### 4.3 擴展挑戰

| 挑戰 | 解法 |
|:---|:---|
| **Load Balancer** | 使用 Sticky Session 或 L4 LB |
| **橫向擴展** | Redis Pub/Sub / Kafka 做跨 Server 訊息轉發 |
| **心跳保活** | Ping/Pong Frame (每 30-60 秒) |
| **重連處理** | Client 端實作 Exponential Backoff |

### 4.4 替代方案

| 技術 | 適用場景 |
|:---|:---|
| **Server-Sent Events (SSE)** | 僅需 Server → Client 單向推送 (如通知) |
| **Long Polling** | 不支援 WebSocket 的環境 |
| **gRPC Streaming** | 微服務間的雙向串流 |

---

## 5. gRPC

### 5.1 核心特性

| 特性 | 說明 |
|:---|:---|
| **Protocol Buffers** | 二進位序列化，比 JSON 小 3-10 倍 |
| **HTTP/2** | 多工、Header 壓縮 |
| **強型別 Schema** | `.proto` 定義，自動生成程式碼 |
| **雙向串流** | Unary, Server/Client/Bidirectional Streaming |

### 5.2 通訊模式

```protobuf
service Greeter {
  // Unary (一來一回)
  rpc SayHello (HelloRequest) returns (HelloReply);
  
  // Server Streaming
  rpc ListFeatures (Rectangle) returns (stream Feature);
  
  // Client Streaming
  rpc RecordRoute (stream Point) returns (RouteSummary);
  
  // Bidirectional Streaming
  rpc RouteChat (stream RouteNote) returns (stream RouteNote);
}
```

### 5.3 gRPC vs REST

| 面向 | gRPC | REST |
|:---|:---|:---|
| 效能 | ⚡ 高 | 普通 |
| 瀏覽器支援 | ⚠️ 需 gRPC-Web | ✅ 原生 |
| 可讀性 | ❌ 二進位 | ✅ JSON 可讀 |
| Schema 強制 | ✅ 必須 | ❌ 可選 |
| 適用場景 | 內部微服務 | 公開 API |

---

## 6. Reference
- [HTTP/2 Explained](https://http2-explained.haxx.se/)
- [HTTP/3 Explained](https://http3-explained.haxx.se/)
- [High Performance Browser Networking](https://hpbn.co/)
- [gRPC Documentation](https://grpc.io/docs/)
