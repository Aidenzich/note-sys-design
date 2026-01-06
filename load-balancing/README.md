# Load Balancing (負載均衡)

Load Balancer 是分散式系統中分配流量到多個後端伺服器的關鍵組件，負責提升系統的可擴展性、可用性與效能。

## 1. 層級分類 (L4 vs L7)

| 特性 | L4 Load Balancer | L7 Load Balancer |
|:---|:---|:---|
| **OSI 層級** | Transport Layer (TCP/UDP) | Application Layer (HTTP/HTTPS) |
| **決策依據** | IP + Port | URL Path, Headers, Cookies, Body |
| **Session 黏著** | 基於 IP (不精確) | 基於 Cookie/Header (精確) |
| **SSL Termination** | ❌ 通常 Pass-through | ✅ 在 LB 解密 |
| **效能** | ⚡ 極高 (不解析內容) | 較低 (需解析 HTTP) |
| **典型產品** | AWS NLB, HAProxy (L4 Mode), LVS | AWS ALB, Nginx, Envoy |

```
L4 決策流程:
Client [IP:Port] ──→ [LB] ──→ Backend (純轉發，不看內容)

L7 決策流程:
Client [GET /api/v2/users] ──→ [LB 解析 HTTP] ──→ Route to API Service
Client [GET /static/logo.png] ──→ [LB 解析 HTTP] ──→ Route to CDN/Static Service
```

---

## 2. 負載均衡演算法 (Algorithms)

### 2.1 靜態演算法

| 演算法 | 原理 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Round Robin** | 依序輪流分配 | 簡單、公平 | 不考慮伺服器負載差異 |
| **Weighted Round Robin** | 按權重比例分配 | 可處理異構伺服器 | 權重需手動設定 |
| **IP Hash** | `hash(client_ip) % N` | 同 Client 固定到同 Server | 伺服器變動時分佈失衡 |

### 2.2 動態演算法

| 演算法 | 原理 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Least Connections** | 分給當前連線數最少的 | 自動適應負載 | 需維護連線計數 |
| **Weighted Least Conn** | 考慮權重的 Least Conn | 適合異構伺服器 | 計算略複雜 |
| **Least Response Time** | 分給回應時間最短的 | 最佳化延遲 | 需持續量測延遲 |
| **Random** | 隨機選擇 | 簡單、無狀態 | 可能短期不均勻 |

### 2.3 進階演算法 - Consistent Hashing

**問題：** 傳統 Hash (`hash(key) % N`) 在節點增減時會導致大規模重分佈。

**解法：** Consistent Hashing 將 Key 和 Node 都映射到一個環上：

```
        Hash Ring (0 ~ 2^32 - 1)
           ┌────────────────┐
          /                  \
         A ──────── Key1 ─── B
         │                   │
         │       Key2        │
         │         ↓         │
         D ─────────────── C
          \                  /
           └────────────────┘

Key1 → 順時針找到 B
Key2 → 順時針找到 C
```

**Virtual Nodes (Vnodes)：**
- 每個實體節點對應多個虛擬節點
- 解決節點數少時的不均勻問題

**應用場景：** Cassandra, Memcached Client, Nginx upstream_hash

---

## 3. 健康檢查 (Health Check)

### 3.1 檢查類型

| 類型 | 原理 | 適用場景 |
|:---|:---|:---|
| **TCP Check** | 嘗試建立 TCP 連線 | 基本存活檢測 |
| **HTTP Check** | 發送 HTTP 請求，檢查 Status Code | Web 服務 |
| **Script Check** | 執行自訂腳本 | 複雜業務邏輯 |

### 3.2 常見配置 (Nginx 範例)

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=2;
    server 10.0.0.3:8080 backup;   # 備用伺服器
    
    # 健康檢查 (需 nginx-plus 或第三方模組)
    health_check interval=5s fails=3 passes=2;
}
```

### 3.3 熔斷邏輯

```
                    ┌──────────────┐
                    │   Healthy    │
                    └──────┬───────┘
                           │ 連續失敗 N 次
                           ▼
                    ┌──────────────┐
         定時重試 → │   Unhealthy  │
                    └──────┬───────┘
                           │ 連續成功 M 次
                           ▼
                    ┌──────────────┐
                    │   Healthy    │
                    └──────────────┘
```

---

## 4. Session 親和性 (Session Affinity / Sticky Session)

**問題：** 無狀態 LB 可能讓同一用戶的請求分散到不同 Server，導致 Session 遺失。

**解法：**

| 方法 | 原理 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Source IP Hash** | 根據 Client IP Hash | 無需額外狀態 | NAT 後的用戶會集中 |
| **Cookie Injection** | LB 注入識別 Cookie | 精確控制 | 需 L7 LB |
| **Session Replication** | Server 間同步 Session | 透明於 LB | 增加 Server 負擔 |
| **Centralized Session** | Session 存 Redis | 完全無狀態 | 增加 Redis 依賴 |

> **現代最佳實踐：** 使用 **JWT + Redis Session Store**，使後端完全無狀態。

---

## 5. SSL/TLS Termination

| 模式 | 流程 | 優點 | 缺點 |
|:---|:---|:---|:---|
| **Termination at LB** | Client ↔️ LB (HTTPS) ↔️ Backend (HTTP) | LB 可做 L7 路由、減少 Backend 負擔 | LB 到 Backend 明文傳輸 |
| **SSL Passthrough** | Client ↔️ LB (TCP) ↔️ Backend (HTTPS) | 端到端加密 | LB 無法做 L7 路由 |
| **Re-encryption** | Client ↔️ LB (HTTPS) ↔️ Backend (HTTPS) | 端到端加密 + L7 路由 | LB 與 Backend 雙重開銷 |

---

## 6. 高可用架構

### 6.1 Active-Passive (主備)

```
        ┌────────────────────────┐
        │      Virtual IP        │
        └───────────┬────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
   ┌────▼────┐             ┌────▼────┐
   │  LB 1   │  Heartbeat  │  LB 2   │
   │ (Active)│◄───────────►│(Standby)│
   └────┬────┘             └─────────┘
        │
   ┌────▼────────────────────────┐
   │        Backend Servers      │
   └─────────────────────────────┘
```

**機制：** VRRP (Virtual Router Redundancy Protocol) / Keepalived

### 6.2 Active-Active (雙活)

```
                 ┌─────────────┐
                 │  DNS / BGP  │ ← Anycast
                 └──────┬──────┘
          ┌─────────────┴─────────────┐
          │                           │
     ┌────▼────┐                 ┌────▼────┐
     │  LB 1   │                 │  LB 2   │
     │(Active) │                 │(Active) │
     └────┬────┘                 └────┬────┘
          │                           │
     ┌────▼───────────────────────────▼────┐
     │          Backend Servers            │
     └─────────────────────────────────────┘
```

**優點：** 充分利用兩台 LB 的資源
**挑戰：** 狀態同步 (如連線表)

---

## 7. 雲端服務對應

| 功能 | AWS | GCP | Azure |
|:---|:---|:---|:---|
| L4 LB | NLB | Network LB | Load Balancer |
| L7 LB | ALB | Application LB | Application Gateway |
| Global LB | Global Accelerator | Cloud Load Balancing | Front Door |
| DNS LB | Route 53 | Cloud DNS | Traffic Manager |

---

## 8. Reference
- [Nginx Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/)
- [HAProxy Configuration](https://www.haproxy.com/documentation/)
- [Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/)
