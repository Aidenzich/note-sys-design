# News Feed 系統設計

News Feed (或 Timeline) 是社交媒體的核心功能，涉及複雜的資料模型、Fan-out 策略、和排序演算法。

## 1. 需求分析

### 1.1 功能需求

| 功能 | 說明 |
|:---|:---|
| **發布貼文** | 用戶可發布文字、圖片、影片 |
| **瀏覽動態** | 看到關注對象的貼文 (按時間/演算法排序) |
| **互動** | 按讚、留言、分享 |
| **即時更新** | 新貼文即時顯示 |

### 1.2 非功能需求

| 需求 | 目標 |
|:---|:---|
| **DAU** | 100 Million |
| **每用戶 Followee** | 平均 200 人 |
| **QPS (讀)** | 100M × 10 次/天 / 86400 ≈ 12,000 QPS |
| **延遲** | < 200ms |

---

## 2. 核心問題：Push vs Pull

| 策略 | 機制 (Description) | 優點 (Pros) | 缺點 (Cons) |
| :--- | :--- | :--- | :--- |
| **Pull Model**<br>(讀取時聚合 / Fan-out on Read) | **核心概念**：除非用戶主動讀取，否則系統不執行任何聚合操作。<br><br>**流程**：<br>1. User 請求 Feed。<br>2. 系統查詢所有 Followees (關注對象) 的最新貼文 (`SELECT ... WHERE created_at > last_seen`)。<br>3. 在記憶體中進行合併排序 (Merge & Sort)。<br>4. 回傳結果。 | ✅ **發布貼文即時 (O(1))**：寫入成本極低，只需存入 DB。<br>✅ **不浪費儲存**：無需為每個粉絲維護一份 Feed List。 | ❌ **讀取延遲高**：若關注 200+ 人，需執行大量 DB 查詢與聚合計算。<br>❌ **DB 壓力大**：高流量用戶頻繁刷新會導致 DB 負載過重。 |
| **Push Model**<br>(Fan-out on Write) | **核心概念**：將 Feed 預先計算好 (Pre-computed)，讀取時直接取用。<br><br>**流程**：<br>1. User A 發布貼文。<br>2. 系統找出所有 Followers。<br>3. 將 Post ID 寫入每個 Follower 的 Feed Cache (`feed:{follower_id}`)。<br>4. User B 讀取時，直接回傳 Cache 內容。 | ✅ **讀取極快 (O(1))**：不需要即時聚合，不僅快且保護 DB。<br>✅ **適合多讀少寫**：符合大多數社交媒體的使用模式。 | ❌ **寫入放大 (Write Amplification)**：名人發文 (e.g. 100萬粉絲) 會觸發百萬次寫入，導致「發送延遲」。<br>❌ **儲存成本高**：同一份資料被複製多份。 |

### 2.3 Hybrid Model (業界主流)

```
普通用戶 (< 10K Followers): Push Model
名人/大 V (> 10K Followers): Pull Model (讀取時即時聚合)

User Feed = Cached Posts (Push) + Celebrity Posts (Pull, real-time)
```

```
┌─────────────────────────────────────────────────────────────┐
│                     Feed Generation                        │
│                                                            │
│  ┌──────────────────┐    ┌──────────────────┐             │
│  │  Feed Cache      │ +  │ Celebrity Posts  │             │
│  │  (Pre-generated) │    │ (Real-time Query)│             │
│  └────────┬─────────┘    └────────┬─────────┘             │
│           │                       │                        │
│           └───────────┬───────────┘                        │
│                       ▼                                    │
│              Merge & Rank                                  │
│                       ↓                                    │
│              Return Feed                                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 高層架構

```
                                 ┌─────────────┐
                                 │   Web/App   │
                                 └──────┬──────┘
                                        │
                                 ┌──────▼──────┐
                                 │ API Gateway │
                                 └──────┬──────┘
                                        │
              ┌─────────────────────────┼─────────────────────────┐
              │                         │                         │
       ┌──────▼──────┐          ┌───────▼──────┐          ┌──────▼──────┐
       │ Post Service│          │ Feed Service │          │ User Service│
       │  (Write)    │          │  (Read)      │          │             │
       └──────┬──────┘          └───────┬──────┘          └─────────────┘
              │                         │
              │                         └─────────────┐
              ▼                                       ▼
       ┌────────────┐                          ┌────────────┐
       │   Kafka    │                          │ Feed Cache │
       │            │                          │  (Redis)   │
       └─────┬──────┘                          └────────────┘
             │
     ┌───────┴───────┐
     ▼               ▼
┌─────────┐   ┌─────────┐
│ Fan-out │   │ Post DB │
│ Workers │   │(Cassandra)│
└────┬────┘   └─────────┘
     │
     ▼
┌────────────┐
│ Feed Cache │
│  (Redis)   │
└────────────┘
```

---

## 4. 資料模型

### 4.1 Post Table (Cassandra)

```sql
CREATE TABLE posts (
    user_id BIGINT,
    post_id BIGINT,  -- Snowflake ID (時間排序)
    content TEXT,
    media_urls LIST<TEXT>,
    created_at TIMESTAMP,
    like_count BIGINT,
    comment_count BIGINT,
    
    PRIMARY KEY ((user_id), post_id)
) WITH CLUSTERING ORDER BY (post_id DESC);
```

### 4.2 Feed Cache (Redis)

```
Key: feed:{user_id}
Value: Sorted Set (Score = Post Timestamp/Snowflake ID)

ZADD feed:123 1609459200 "post:456"
ZADD feed:123 1609459100 "post:457"

ZREVRANGE feed:123 0 20  -- 取最新 20 條
```

### 4.3 Social Graph (關注關係)

```sql
-- 選項 1: Traditional RDBMS
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);

-- 選項 2: Graph Database (Neo4j)
(User:A)-[:FOLLOWS]->(User:B)
```

---

## 5. Fan-out 流程

### 5.1 發布貼文

```python
def publish_post(user_id, content):
    # 1. 寫入 Post DB
    post_id = snowflake.generate()
    post_db.insert(user_id, post_id, content)
    
    # 2. 發送到 Kafka
    kafka.send("new_posts", {
        "user_id": user_id,
        "post_id": post_id,
        "follower_count": user.follower_count
    })
```

### 5.2 Fan-out Worker

```python
def fanout_worker(message):
    user_id = message.user_id
    post_id = message.post_id
    
    # 判斷是否為大 V
    if message.follower_count > 10000:
        # 大 V: 不 Fan-out，僅標記為 celebrity post
        cache.sadd("celebrity_posts", post_id)
        return
    
    # 普通用戶: Fan-out 到所有 Followers
    followers = social_graph.get_followers(user_id)
    
    for batch in chunks(followers, 1000):
        for follower_id in batch:
            redis.zadd(f"feed:{follower_id}", {post_id: timestamp})
            redis.zremrangebyrank(f"feed:{follower_id}", 0, -1001)  # 保留最新 1000 條
```

---

## 6. Feed 讀取

```python
def get_feed(user_id, cursor=None, limit=20):
    # 1. 從 Cache 取得預生成 Feed
    cached_posts = redis.zrevrange(f"feed:{user_id}", 0, limit)
    
    # 2. 取得追蹤的大 V 最新貼文
    celebrity_followees = get_celebrity_followees(user_id)
    celebrity_posts = []
    for celebrity_id in celebrity_followees:
        posts = post_db.query(user_id=celebrity_id, limit=5)
        celebrity_posts.extend(posts)
    
    # 3. 合併並排序
    all_posts = merge_and_rank(cached_posts, celebrity_posts)
    
    # 4. 分頁 (Cursor-based)
    return paginate(all_posts, cursor, limit)
```

---

## 7. 排序演算法 (Ranking)

### 7.1 時間順序 (Chronological)

```
Score = Post Timestamp
```
- ✅ 公平、透明
- ❌ 垃圾/不相關內容充斥

### 7.2 參與度排序 (Engagement-based)

```
Score = (Likes × 1) + (Comments × 3) + (Shares × 5) + (TimDecay)
TimDecay = 1 / (1 + hours_since_post^1.5)
```

### 7.3 機器學習排序 (ML Ranking)

```
Features:
- 作者與讀者的互動歷史
- 貼文類型 (影片/圖片/文字)
- 用戶偏好 (停留時間、點擊率)
- 社交關係強度

Model: LightGBM / Deep Neural Network
Output: 預測用戶參與機率
```

---

## 8. 即時更新

### 8.1 Polling

```javascript
// Client 每 30 秒拉取一次
setInterval(() => {
    fetch('/api/feed/new?since=' + lastPostId)
}, 30000)
```

### 8.2 Long Polling

```
Client → Server: Do you have new posts?
Server → (等待，直到有新貼文或 timeout)
Server → Client: New posts!
Client → Immediately reconnect
```

### 8.3 WebSocket / SSE

```
Client ←→ Server: Persistent Connection
Server pushes new posts in real-time
```

---

## 9. 快取策略

### 9.1 多層快取

```
L1: CDN (Static assets: Images, Videos)
L2: Local Cache (Application memory)
L3: Redis (Feed cache, Hot user profiles)
L4: Database
```

### 9.2 快取失效

```python
def on_unfollow(user_id, unfollowed_id):
    # 從 Feed Cache 移除該用戶的貼文
    posts_to_remove = get_posts_by_user(unfollowed_id)
    for post_id in posts_to_remove:
        redis.zrem(f"feed:{user_id}", post_id)
```

---

## 10. 擴展性

### 10.1 分片策略

| 資料 | 分片 Key | 原因 |
|:---|:---|:---|
| Posts | user_id | 同用戶貼文儲存在一起 |
| Feed Cache | user_id | 每個用戶的 Feed 獨立 |
| Social Graph | follower_id | 查詢「我追蹤誰」高效 |

### 10.2 地理分佈

```
US Users → US Data Center
EU Users → EU Data Center

跨區域複製:
- Celebrity Posts: 全球複製
- Normal User Feed: 僅本區域
```

---

## 11. Trade-offs 總結

| 決策 | 選項 A | 選項 B | 推薦 |
|:---|:---|:---|:---|
| Fan-out 策略 | Push Only | Hybrid (Push + Pull) | **Hybrid** |
| 排序 | Chronological | ML Ranking | 依產品需求 |
| 即時更新 | Polling | WebSocket | WebSocket (若支援) |
| Feed 儲存 | 全部 | 最近 N 條 | **最近 N 條** |

---

## 12. Reference
- [System Design Interview - Alex Xu, Chapter 11](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)
- [Scaling Instagram Infrastructure](https://instagram-engineering.com/instagram-engineering-blog-bab6f3b8e4)
- [Twitter's Recommendation Algorithm](https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm)
