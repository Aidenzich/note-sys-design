# Auto Complete

## Requirements

### Functional Requirements (商業需求)
- Fast Response
- 


### Non-Functional Requirements (系統需求)
- Scalable
- Highly available



### 流量估算：

DAU -> 搜尋 -> 有多少 request.


> ~24,000 query per second (QPS) = 10,000,000 users * 10 queries / day * 20 char / 24 hours / 3600 sec



## High-Level Design
1. Data gathering Service, 取得 User 吃queries 然後即時的整合這些 queries. 儘管 Real-time processing 對於 large data set 來說不太現實，但是個好的開始點
2. Query Service: 根據 given 的 search query or prefix, 返回 5 個 most frequently searches terms



## Data Gathering Service:
假設我們有一個 frequency table, 用來存 query string 的frequency

當使用者 enter queires twitch , twitter, twitter, and twillo sequentially 時，正個 freq table 會以下圖做 update


twitch : 1
twitter: 1
twitter: 2



