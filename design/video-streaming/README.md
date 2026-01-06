# Video Streaming 系統設計 (YouTube-like)

Video Streaming 系統涉及大規模儲存、轉碼管線、CDN 分發和自適應碼率技術。

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **上傳影片** | 支援多種格式，最大 10GB |
| **串流播放** | 支援 Web、Mobile、TV |
| **自適應碼率** | 根據網路調整畫質 |
| **搜尋與推薦** | 影片搜尋、個人化推薦 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **DAU** | 1 Billion |
| **影片數量** | 1 Billion+ |
| **每日上傳** | 500,000 影片 |
| **延遲 (首幀)** | < 2 秒 |
| **可用性** | 99.99% |

---

## 2. 高層架構

```
┌────────────────────────────────────────────────────────────────────┐
│                          Upload Flow                               │
│  User → API Gateway → Upload Service → Transcoding Pipeline        │
│                                              ↓                     │
│                                     Object Storage (S3)            │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│                          Playback Flow                             │
│  User → CDN (Edge) → Origin Shield → Object Storage (S3)           │
│         ↑                                                          │
│         └── Video Manifest (HLS/DASH)                              │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. 上傳與轉碼管線

### 3.1 上傳流程

```
1. Client 請求上傳 URL
2. API 返回 Pre-signed URL (直傳 S3)
3. Client 直接上傳到 S3 (繞過 API Server)
4. S3 觸發 Event → Transcoding Pipeline
```

```
┌──────┐    ┌───────────┐    ┌─────────────────┐
│Client│───→│ API Server│───→│ Generate Signed │
│      │←───│           │←───│ URL             │
└──┬───┘    └───────────┘    └─────────────────┘
   │
   │ Direct Upload
   ▼
┌──────────────────────────────────────┐
│            S3 (Raw Videos)           │
└──────────────────┬───────────────────┘
                   │ S3 Event
                   ▼
┌──────────────────────────────────────┐
│     Transcoding Queue (SQS/Kafka)    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│     Transcoding Workers (FFmpeg)     │
└──────────────────────────────────────┘
```

### 3.2 轉碼 (Transcoding)

**為什麼需要轉碼?**
- 原始影片格式多樣 (MOV, AVI, MKV...)
- 需要多種解析度適應不同設備
- 需要分段以支援自適應串流

**輸出格式：**
| 解析度 | Bitrate | 用途 |
|:---|:---|:---|
| 240p | 300 kbps | 極差網路 |
| 480p | 1 Mbps | 手機 3G |
| 720p | 3 Mbps | 桌面/4G |
| 1080p | 6 Mbps | 高清 |
| 4K | 20 Mbps | 高端設備 |

### 3.3 Parallel Transcoding

```
                    ┌─────────────────┐
                    │  Raw Video      │
                    │  (10 GB, 2 hrs) │
                    └────────┬────────┘
                             │ Split into chunks
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
    ┌────────────┐    ┌────────────┐    ┌────────────┐
    │ Chunk 1    │    │ Chunk 2    │    │ Chunk N    │
    │ (10 min)   │    │ (10 min)   │    │ (10 min)   │
    └──────┬─────┘    └──────┬─────┘    └──────┬─────┘
           │                 │                 │
   Parallel Transcode  Parallel Transcode  Parallel Transcode
           │                 │                 │
           └─────────────────┴─────────────────┘
                             │ Merge
                             ▼
                    ┌─────────────────┐
                    │ Final Output    │
                    └─────────────────┘
```

---

## 4. 串流協議

### 4.1 HLS vs DASH

| 特性 | HLS (Apple) | DASH (MPEG) |
|:---|:---|:---|
| **格式** | `.m3u8` + `.ts` | `.mpd` + `.m4s` |
| **Segment 長度** | 通常 6-10 秒 | 2-6 秒 |
| **Apple 支援** | ✅ Native | ⚠️ 需要 Player |
| **DRM** | FairPlay | Widevine, PlayReady |
| **延遲** | 較高 (20-30s) | 可低至 2-5s |

### 4.2 HLS Manifest 範例

```m3u8
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=300000,RESOLUTION=426x240
240p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480
480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=6000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
```

### 4.3 自適應碼率 (ABR - Adaptive Bitrate)

```
┌──────────────────────────────────────────────────────────────┐
│                   Player ABR Logic                          │
│                                                              │
│  1. 測量下載速度 (Bandwidth Estimation)                      │
│  2. 計算 Buffer 長度                                         │
│  3. 選擇合適的 Bitrate:                                      │
│     - Buffer 低 + 網速慢 → 降級                              │
│     - Buffer 高 + 網速快 → 升級                              │
│  4. 下載對應 Segment                                         │
└──────────────────────────────────────────────────────────────┘

Network: Slow → Fast → Slow
Quality:  240p → 720p → 480p → 1080p → 720p
          (Seamless switching)
```

---

## 5. 儲存架構

### 5.1 儲存層級

| 層級 | 儲存類型 | 用途 | 成本 |
|:---|:---|:---|:---|
| **Heat** | SSD | 熱門影片 (7 天內) | 高 |
| **Warm** | HDD / S3 Standard | 一般影片 | 中 |
| **Cold** | S3 Glacier | 老舊/長尾影片 | 低 |

### 5.2 成本優化

**熱度追蹤：**
```python
def access_video(video_id):
    redis.zincrby("video_popularity", 1, video_id)
    
# 定期任務
def tiering_job():
    hot_videos = redis.zrevrange("video_popularity", 0, 10000)
    for video in all_videos:
        if video.id in hot_videos:
            move_to_ssd(video)
        elif video.age > 90_days and video.views < 100:
            move_to_glacier(video)
```

---

## 6. CDN 架構

### 6.1 多層 CDN

```
              ┌─────────────────────────────────────┐
              │          Global Load Balancer       │
              │          (Anycast / GeoDNS)         │
              └──────────────────┬──────────────────┘
                                 │
       ┌─────────────────────────┼─────────────────────────┐
       ▼                         ▼                         ▼
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│  Edge PoP   │          │  Edge PoP   │          │  Edge PoP   │
│  (Tokyo)    │          │  (London)   │          │  (NYC)      │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       └────────────────────────┴────────────────────────┘
                                │
                         ┌──────▼──────┐
                         │Origin Shield│
                         │  (Regional) │
                         └──────┬──────┘
                                │
                         ┌──────▼──────┐
                         │   Origin    │
                         │   (S3)      │
                         └─────────────┘
```

### 6.2 快取策略

| 內容類型 | TTL | 原因 |
|:---|:---|:---|
| `.m3u8` / `.mpd` | 1-5 秒 | 直播需即時更新 |
| `.ts` / `.m4s` (VOD) | 1 年 | 內容不變 |
| Thumbnail | 1 天 | 可能更新 |

---

## 7. 資料庫設計

### 7.1 Video Metadata

```sql
CREATE TABLE videos (
    video_id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(500),
    description TEXT,
    status ENUM('processing', 'ready', 'failed', 'deleted'),
    duration_seconds INT,
    view_count BIGINT DEFAULT 0,
    like_count BIGINT DEFAULT 0,
    created_at TIMESTAMP,
    
    -- 儲存位置
    manifest_url VARCHAR(2048),
    thumbnail_url VARCHAR(2048),
    
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);
```

### 7.2 View Count 更新

**問題：** 每次觀看都 UPDATE 會造成 DB 壓力巨大

**解法：** Write-Behind + Batch Update

```python
# 即時寫入 Redis
def increment_view(video_id):
    redis.hincrby("video_views", video_id, 1)

# 定時聚合寫入 DB (每分鐘)
def flush_views_to_db():
    views = redis.hgetall("video_views")
    for video_id, count in views.items():
        db.execute("UPDATE videos SET view_count = view_count + ? WHERE id = ?", count, video_id)
    redis.delete("video_views")
```

---

## 8. 低延遲直播

### 8.1 傳統 HLS vs Low-Latency HLS (LL-HLS)

| 特性 | 傳統 HLS | LL-HLS |
|:---|:---|:---|
| Segment 長度 | 6-10 秒 | 1-2 秒 |
| 延遲 | 20-30 秒 | 2-5 秒 |
| 機制 | 完整 Segment | Partial Segments |

### 8.2 WebRTC (超低延遲)

```
延遲 < 1 秒，適合視訊通話、互動直播
不適合大規模廣播 (架構複雜)
```

---

## 9. Reference
- [Netflix Tech Blog - Video Encoding](https://netflixtechblog.com/)
- [YouTube Engineering](https://blog.youtube/)
- [HLS Specification (Apple)](https://developer.apple.com/streaming/)
- [MPEG-DASH Standard](https://dashif.org/)
