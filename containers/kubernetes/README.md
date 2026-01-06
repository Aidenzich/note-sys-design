# Kubernetes (K8s) 核心架構

Kubernetes 是 Google 開源的容器編排平台，用於自動化部署、擴展和管理容器化應用。

## 1. 核心架構

K8s 遵循 Master-Worker 架構。

### 1.1 Control Plane (Master Node)

負責管理 Cluster 的全局狀態。

- **API Server (kube-apiserver)**: 唯一入口，處理 REST 請求，驗證並寫入 etcd。
- **etcd**: 分散式 Key-Value Store，儲存 Cluster 的所有配置與狀態 (Source of Truth)。
- **Scheduler (kube-scheduler)**: 決定新 Pod 應該跑在哪個 Node 上 (考慮資源、親和性)。
- **Controller Manager**: 維護 Cluster 狀態 (如 ReplicaSet Controller 確保 Pod 數量正確)。
- **Cloud Controller Manager**: 與雲端供應商 API 介接 (如建立 AWS ELB)。

### 1.2 Worker Node (Data Plane)

實際執行應用程式的地方。

- **Kubelet**: Node 上的 Agent，負責與 API Server 通訊，並管理 Docker/containerd 容器。
- **Kube-proxy**: 維護網路規則 (IPVS/Iptables)，實現 Service 的負載均衡。
- **Container Runtime**: 負責運行容器 (containerd, RIO-O, Docker Engine)。

---

## 2. 核心物件 (Objects)

### 2.1 Pod

- K8s 的最小部署單位。
- 一個 Pod 包含一個或多個容器 (如 App + Sidecar)。
- **共享**: Network Namespace (IP, Port) 和 Storage Volume。
- **生命週期**: Ephemeral (短暫的)，死掉重啟後 IP 會變。

### 2.2 Deployment

- 管理無狀態應用 (Stateless Apps)。
- 定義期望狀態 (如 `replicas: 3`)。
- 支援 Rolling Update (滾動更新) 和 Rollback。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
```

### 2.3 Service

- 提供穩定的網路入口 (VIP) 來存取一組 Pod。
- **ClusterIP**: 僅 Cluster 內部可存取 (預設)。
- **NodePort**: 開放 Node 上的特定 Port 供外部存取。
- **LoadBalancer**: 請求雲端廠商提供 LB (如 AWS ELB)。

### 2.4 Ingress

- L7 Load Balancer (HTTP/HTTPS)。
- 管理外部存取，提供 SSL Termination、域名路由。
- 需配合 Ingress Controller (如 Nginx, AWS ALB Controller)。

### 2.5 ConfigMap & Secret

- **ConfigMap**: 儲存非機密配置 (如環境變數、config 檔)。
- **Secret**: 儲存機密資訊 (Base64 編碼，如密碼、憑證)。

### 2.6 StatefulSet

- 管理有狀態應用 (Database, Kafka)。
- 保證穩定的網路標識 (`web-0`, `web-1`) 和有序部署。

---

## 3. 網路模型 (Networking)

K8s 網路的黃金法則：
1. 所有 Pod 可以不經 NAT 互相通訊。
2. 所有 Node 可以不經 NAT 與所有 Pod 通訊。
3. Pod 看到的自己 IP 與別人看它的 IP 一樣。

**CNI (Container Network Interface)**:
K8s 不實作網路，而是定義介面，由 Plugin 實作 (如 Flannel, Calico, Cilium)。

---

## 4. 自動擴展 (Autoscaling)

### 4.1 HPA (Horizontal Pod Autoscaler)

根據 CPU/Memory 使用率自動調整 **副本數量** (Replicas)。

```yaml
spec:
  scaleTargetRef:
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### 4.2 VPA (Vertical Pod Autoscaler)

自動調整 Pod 的 **CPU/Memory Request/Limit** (需重啟 Pod)。

### 4.3 Cluster Autoscaler

當 Node 資源不足以排程新 Pod 時，自動新增 Node (雲端環境)。

---

## 5. 常見面試題

**Q: Pod 和 Container 的區別？**
A: Pod 是 K8s 的邏輯單位，可以包含多個 Container。同 Pod 內的 Container 共享 IP 和 Volume。

**Q: Deployment 和 StatefulSet 的區別？**
A: Deployment 認為 Pod 是可替換的 (無狀態)；StatefulSet 保證 Pod 的順序性和持久化標識 (有狀態)。

**Q: Kube-proxy 的作用？**
A: 監聽 API Server，更新 Node 上的 Iptables/IPVS 規則，將 Service VIP 的流量轉發到後端 Pod。

---

## 6. Reference
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
