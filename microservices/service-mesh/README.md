# Service Mesh (服務網格)

Service Mesh 是一個專注於處理服務間通訊 (Service-to-Service Communication) 的基礎設施層，通常以輕量級網路代理 (Sidecar Proxy) 的形式與應用程式部署在一起。

## 1. 核心概念

### 1.1 Sidecar Pattern

將網路通訊功能從應用程式中剝離，放入一個獨立的 Process/Container (Proxy)。

```
Pod / Host
┌───────────────────────┐
│  Application Code     │
│  (Business Logic)     │
│          ↕ (localhost)│
│  ┌─────────────────┐  │      Network       ┌─────────────────┐
│  │ Sidecar Proxy   │◀-┼-------------------▶│ Sidecar Proxy   │
│  │ (Envoy/Linkerd) │  │                    │ (Other Service) │
│  └─────────────────┘  │                    └─────────────────┘
└───────────────────────┘
```

- **Data Plane (數據平面)**: 實際處理流量的代理 (如 Envoy)。
- **Control Plane (控制平面)**: 管理和配置代理的中心組件 (如 Istiod)。

### 1.2 為什麼需要 Service Mesh?

在微服務架構中，開發者需要重複實現以下功能：
- Retry / Circuit Breaking
- Tracing / Metrics
- TLS 加密
- Canary Deployment

Service Mesh 將這些功能**下沉到基礎設施層**，對應用程式**透明** (Zero Code Change)。

---

## 2. 核心功能

### 2.1 流量管理 (Traffic Management)

- **智慧路由**: 根據 Header/Path 路由 (`User-Agent: iOS` → v2)
- **Canary Release (金絲雀發佈)**: 98% 流量到 v1，2% 到 v2
- **Traffic Mirroring (流量鏡像)**: 複製一份即時流量到 v2 進行測試 (不影響正式回覆)
- **Fault Injection (故障注入)**: 模擬延遲或錯誤，測試系統韌性

### 2.2 安全性 (Security)

- **mTLS (Mutual TLS)**: 自動為服務間通訊加密並驗證身份
  - 傳統：每個 App 自己管憑證 💀
  - Mesh：Sidecar 自動 Rotate 憑證，App 還是用 HTTP ✅
- **AuthZ Policies**: 定義誰可以呼叫誰 (Service A can call Service B, but not C)

### 2.3 可觀測性 (Observability)

- 自動生成 **Service Graph** (Kiali)
- 統一收集 Golden Signals (Latency, Traffic, Errors, Saturation)
- 分散式追蹤 Span 自動生成 (需 App 傳遞 Trace Context Header)

---

## 3. Istio 架構 (標準範例)

```
       Control Plane (Istiod)
     ┌───────────────────────┐
     │ Discovery (Pilot)     │ ← 服務發現 & 配置派發
     │ Security (Citadel)    │ ← 憑證管理
     │ Validation (Galley)   │
     └───────────┬───────────┘
                 │ xDS Protocol (Config Updates)
                 ▼
       Data Plane (Envoy Proxies)
   App A ↔ [Proxy] ═══════ [Proxy] ↔ App B
```

---

## 4. 效能與代價

引入 Service Mesh 不是沒有代價的：

1. **延遲 (Latency)**: 每個請求增加兩次跳轉 (Outbound Proxy + Inbound Proxy)。通常增加 2-5ms。
2. **資源消耗**: 每個 Pod 多一個 Sidecar Container，消耗 CPU/RAM。
3. **複雜度**: 維運 Control Plane 的難度高 (Though managed services like AWS App Mesh help)。

**何時使用？**
- ❌ 服務數量少 (< 20)
- ✅ 服務數量多，多語言 (Polyglot)，需要統一治理
- ✅ 對安全性 (Zero Trust) 有嚴格要求

---

## 5. 無 Sidecar 模式 (Cilium / eBPF)

新一代 Service Mesh (如 Cilium Service Mesh) 利用 Linux Kernel 的 **eBPF** 技術，直接在 Kernel 層處理網路封包，無需注入 Sidecar Container。

- **優點**: 效能更好，無 Sidecar 注入困擾
- **缺點**: L7 處理能力目前不如 Envoy 成熟

---

## 6. Reference
- [Istio Documentation](https://istio.io/latest/docs/)
- [The Service Mesh Pattern](https://martinfowler.com/articles/microservices-externalized.html)
- [Cilium: Service Mesh without Sidecars](https://cilium.io/blog/2021/12/01/cilium-service-mesh-beta/)
