# Cloud File Storage 系統設計 (Dropbox-like)

Cloud File Storage 涉及檔案同步、衝突解決、高效傳輸和跨裝置一致性。

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **上傳/下載** | 支援任意檔案類型 |
| **同步** | 多裝置自動同步 |
| **檔案歷史** | 版本控制、還原 |
| **分享** | 連結分享、協作編輯 |
| **離線存取** | 本地快取 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **可靠性** | 99.999% 不丟檔 |
| **同步延遲** | < 10 秒 |
| **支援檔案大小** | 最大 50 GB |
| **儲存效率** | 跨用戶去重 |

---

## 2. 高層架構

```
┌────────────────────────────────────────────────────────────────┐
│                        Client (Desktop/Mobile)                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Local Folder → Watcher → Chunker → Upload/Download Queue │  │
│  └──────────────────────────────────────────────────────────┘  │
└───────────────────────────┬────────────────────────────────────┘
                            │ HTTPS / WebSocket
                ┌───────────┴───────────┐
                ▼                       ▼
        ┌─────────────┐         ┌─────────────┐
        │ API Gateway │         │ Sync Service│
        └──────┬──────┘         │ (WebSocket) │
               │                └──────┬──────┘
               │                       │
        ┌──────▼──────┐         ┌──────▼──────┐
        │ Metadata DB │         │ Notification│
        │ (MySQL)     │         │ Service     │
        └─────────────┘         └─────────────┘
               │
        ┌──────▼──────┐
        │Block Storage│
        │ (S3)        │
        └─────────────┘
```

---

## 3. 核心設計：Chunking

### 3.1 為什麼要 Chunking?

| 問題 | 解法 |
|:---|:---|
| 大檔案上傳失敗需重來 | 分塊後只需重傳失敗的塊 |
| 小修改需重傳整個檔案 | 只傳修改的塊 |
| 難以去重 | 塊級別去重 |

### 3.2 Chunking 策略

**固定大小 (Fixed-Size Chunking)：**
```
檔案 = [Block 1: 4MB][Block 2: 4MB][Block 3: 4MB]...
```
- 簡單
- 插入操作導致後續塊全變

**內容定義切分 (Content-Defined Chunking - CDC)：**
```
使用滾動雜湊 (Rolling Hash) 找到切分點
當 hash(window) % D == 0 時切分

結果: 即使插入資料，大部分塊不變
```
- Dropbox 和 Google Drive 都使用 CDC
- 典型塊大小: 512KB - 4MB

### 3.3 去重 (Deduplication)

```
┌─────────────────────────────────────────────────────────────┐
│                     File: report.pdf                        │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐               │
│  │Block A │ │Block B │ │Block C │ │Block D │               │
│  │Hash: a1│ │Hash: b2│ │Hash: c3│ │Hash: d4│               │
│  └────────┘ └────────┘ └────────┘ └────────┘               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │ Check if hash exists in DB    │
              │ a1: NEW, b2: EXISTS, c3: NEW  │
              └───────────────────────────────┘
                              │
                              ▼
              Upload only a1, c3, d4 (Skip b2)
```

**去重級別：**
| 級別 | 說明 | 節省空間 |
|:---|:---|:---|
| File-level | 整個檔案相同才去重 | 低 |
| Block-level | 塊級別去重 | 高 |
| Cross-user | 不同用戶間去重 | 最高 (隱私考量) |

---

## 4. 資料模型

### 4.1 Metadata Database

```sql
-- 用戶表
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    storage_quota BIGINT,
    storage_used BIGINT
);

-- 檔案/資料夾表 (樹狀結構)
CREATE TABLE file_metadata (
    file_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    parent_id BIGINT,  -- NULL = root
    name VARCHAR(255),
    is_folder BOOLEAN,
    size BIGINT,
    version BIGINT,
    checksum VARCHAR(64),
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    deleted_at TIMESTAMP,  -- Soft delete
    
    INDEX idx_parent (user_id, parent_id),
    UNIQUE INDEX idx_path (user_id, parent_id, name)
);

-- 塊表
CREATE TABLE blocks (
    block_hash VARCHAR(64) PRIMARY KEY,
    size INT,
    reference_count INT,  -- 去重計數
    storage_path VARCHAR(512)
);

-- 檔案-塊關聯 (一個檔案由多個塊組成)
CREATE TABLE file_blocks (
    file_id BIGINT,
    block_index INT,
    block_hash VARCHAR(64),
    PRIMARY KEY (file_id, block_index)
);
```

### 4.2 版本控制

```sql
CREATE TABLE file_versions (
    version_id BIGINT PRIMARY KEY,
    file_id BIGINT,
    version_number INT,
    size BIGINT,
    checksum VARCHAR(64),
    created_at TIMESTAMP,
    device_id VARCHAR(128)  -- 哪個裝置修改的
);
```

---

## 5. 同步機制

### 5.1 同步流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Side                              │
│                                                             │
│  1. File Watcher 偵測本地變更                               │
│  2. 計算新的塊雜湊                                          │
│  3. 與 Server 對比，找出差異塊                              │
│  4. 上傳差異塊                                              │
│  5. 更新 Metadata                                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Server Side                              │
│                                                             │
│  1. 接收新 Metadata + Blocks                                │
│  2. 儲存 Blocks 到 Object Storage                           │
│  3. 更新 Metadata DB                                        │
│  4. 通知其他裝置 (Push via WebSocket)                        │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 即時通知

```
Client A ──→ Upload change ──→ Server
                                  │
                                  ├──→ Notify Client B (WebSocket)
                                  └──→ Notify Client C (WebSocket)

Client B: Download changed blocks
Client C: Download changed blocks
```

---

## 6. 衝突解決

### 6.1 衝突發生場景

```
Time    Client A                Server              Client B
  │                               │
  1     File v1                  File v1             File v1
  │                               │
  2     Edit → File v1a            │                Edit → File v1b
  │                               │
  3     Upload v1a ─────────────→ │
  │                             File v1a
  4                               │ ←──────────── Upload v1b (Conflict!)
```

### 6.2 解決策略

| 策略 | 說明 | 實現者 |
|:---|:---|:---|
| **Last-Write-Wins** | 後覆蓋先，可能丟失 | 簡單實現 |
| **Keep Both** | 建立 `file (conflicted copy).ext` | Dropbox |
| **Merge** | 自動合併 (適用特定格式) | Google Docs |
| **Manual** | 通知用戶選擇 | 安全但麻煩 |

**Dropbox 方法：**
```
- 原檔案: report.doc
- 衝突後: report.doc, report (John's conflicted copy 2024-01-15).doc
```

---

## 7. 儲存優化

### 7.1 Delta Sync (差異同步)

```
File v1: [Block A][Block B][Block C][Block D]
File v2: [Block A][Block B'][Block C][Block D][Block E]
                      ↑                         ↑
                   Changed                     New

只需上傳 Block B' 和 Block E
```

### 7.2 壓縮

```
Before Upload:
  Raw Block → Compression (gzip/zstd) → Encrypted Block → Upload

Before Download:
  Download → Decrypt → Decompress → Raw Block
```

### 7.3 智慧同步 (Smart Sync)

```
問題: 用戶有 1TB 雲端資料，本地磁碟只有 256GB

解法: 僅同步 Metadata，內容按需下載 (Placeholder)

macOS/Windows: 使用雲端圖示標記檔案狀態
  ☁️ Cloud-only (Placeholder)
  ✓ Downloaded
  ⏳ Syncing
```

---

## 8. 安全性

### 8.1 加密

| 階段 | 加密方式 |
|:---|:---|
| **傳輸中** | TLS 1.3 |
| **儲存時** | AES-256 (Server-side) |
| **端到端** | Client-side encryption (可選) |

### 8.2 分享安全

```sql
CREATE TABLE share_links (
    link_id VARCHAR(32) PRIMARY KEY,
    file_id BIGINT,
    created_by BIGINT,
    password_hash VARCHAR(128),  -- 可選密碼
    expires_at TIMESTAMP,
    download_limit INT,
    download_count INT DEFAULT 0,
    permissions ENUM('view', 'download', 'edit')
);
```

---

## 9. 擴展性

### 9.1 Metadata Sharding

```
Shard Key: user_id

User 1-1M → MySQL Shard 1
User 1M-2M → MySQL Shard 2
...
```

### 9.2 Block Storage

```
Object Storage (S3/GCS) 天然無限擴展
Key: block_hash (SHA-256)
Path: s3://bucket/{prefix}/{block_hash}
```

---

## 10. 監控指標

| 指標 | 說明 |
|:---|:---|
| **Sync Latency** | 檔案變更到同步完成的時間 |
| **Upload/Download Speed** | 傳輸速度 |
| **Conflict Rate** | 衝突發生頻率 |
| **Dedup Ratio** | 去重節省的比例 |
| **Storage Efficiency** | 有效使用率 |

---

## 11. Reference
- [Dropbox Tech Blog - Magic Pocket](https://dropbox.tech/infrastructure/magic-pocket-two-years-later)
- [Google Drive Architecture](https://cloud.google.com/files)
- [Building a Distributed File Sync System](https://www.researchgate.net/publication/332173041)
