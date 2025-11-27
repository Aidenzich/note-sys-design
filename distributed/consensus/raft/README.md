# Raft 共識演算法學習筆記
![alt text](imgs/image.png)


### 1. 核心概念 (Core Concept)
Raft 的設計初衷是為了**可理解性 (Understandability)**。在 Raft (2013) 出現之前，Paxos 是標準，但難以理解且難以正確實作。
* **目標**：在分散式系統中，讓多個節點對「日誌序列 (Log Replication)」達成一致。
* **本質**：Raft 將共識問題分解為三個獨立的子問題：
    1.  **Leader Election (領袖選舉)**
    2.  **Log Replication (日誌複製)**
    3.  **Safety (安全性/一致性)**

### 2. 節點的三種狀態 (Node States)
在任何時間點，Raft 叢集中的每個節點只會處於以下三種狀態之一：

* **Follower (跟隨者)**：
    * 被動狀態，只回應來自 Leader 或 Candidate 的請求。
    * 如果超時沒收到 Leader 的心跳 (Heartbeat)，就會變成 Candidate。
* **Candidate (候選人)**：
    * 發起選舉的狀態。
    * 會向其他節點請求投票 ($RequestVote$)。
    * 如果獲得多數票 ($> N/2$)，就會晉升為 Leader。
* **Leader (領導者)**：
    * **霸道總裁**：負責處理**所有**客戶端的讀寫請求（強一致性下）。
    * 單向發送指令給 Follower，不接受 Follower 的數據更新。
    * 週期性發送 Heartbeat 維持統治地位。

### 3. 關鍵機制一：領袖選舉 (Leader Election)
Raft 使用**心跳機制 (Heartbeat)** 來觸發選舉。

1.  **觸發 (Trigger)**：Follower 有一個隨機的 `Election Timeout`（例如 150ms ~ 300ms）。如果在時間內沒收到 Leader 的 Heartbeat，它認為 Leader 掛了。
2.  **開始選舉**：
    * 增加自己的任期編號 (**Term**，類似邏輯時鐘)。
    * 狀態轉為 **Candidate**。
    * 投自己一票，並並行發送 `RequestVote RPC` 給其他人。
3.  **結果判定**：
    * **贏 (Win)**：獲得叢集**多數 (Majority / Quorum)** 節點的投票 $\rightarrow$ 成為 Leader，發送 Heartbeat。
    * **輸 (Lose)**：收到別人的 Heartbeat（且對方的 Term $\ge$ 自己的 Term）$\rightarrow$ 變回 Follower。
    * **平手 (Split Vote)**：如果多個 Candidate 同時發起選舉，票數被瓜分，無人過半。$\rightarrow$ 等待隨機時間後重新選舉。

> **設計亮點**：**隨機超時 (Randomized Timeout)** 是 Raft 的設計亮點，它極大降低了「無限平手」的機率，讓選舉能快速收斂。

### 4. 關鍵機制二：日誌複製 (Log Replication)
一旦選出 Leader，系統就進入正常運作模式。

1.  **Client Request**：客戶端發送指令（如 `SET x=5`）給 Leader。
2.  **Append Locally**：Leader 將指令寫入自己的 Log（此時狀態為 Uncommitted）。
3.  **Replicate**：Leader 透過 `AppendEntries RPC` 將 Log 條目複製給所有 Followers。
4.  **Commit**：
    * 當 Leader 收到**多數節點** ($N/2 + 1$) 的成功回覆。
    * Leader 將該條目設為 **Committed** 並應用到狀態機 (State Machine)。
    * Leader 返回結果給 Client。
5.  **Notify Followers**：Leader 在下一次 Heartbeat 中通知 Followers 該條目已 Commit，Followers 隨之應用到自己的狀態機。

### 5. 安全性與限制 (Safety & Constraints)
為了保證不會發生數據遺失或不一致，Raft 有嚴格的限制：

* **Log Matching Property**：
    * 如果兩個 Log 有相同的 Index 和 Term，則該條目存儲的指令一定相同。
    * 且它們**之前的所有條目**也都完全相同。
* **Leader Completeness (領袖完整性)**：
    * 被選為 Leader 的節點，必須包含**所有已經 Committed** 的 Log 條目。
    * **投票限制**：如果你（Candidate）的 Log 比我（Follower）舊，我就不投給你。這保證了新 Leader 繼承所有歷史遺產。
* **Term Check**：
    * 任何時候，只要發現對方的 Term 比自己大，自己立刻降級為 Follower。

### 6. Raft vs. Multi-Paxos 比較表
| 特性 | Raft | Multi-Paxos |
| :--- | :--- | :--- |
| **設計目標** | 易於理解、易於實作 | 效能極致、理論完備 |
| **日誌結構** | **強連續性**：不允許 Log 有空洞 (Holes) | **允許空洞**：允許亂序確認 (Out-of-order commit) |
| **Leader 權限** | **強 Leader**：Leader 必須擁有最新 Log 才能當選 | **協商型**：Leader 當選後可以學習缺少的 Log |
| **工業界應用** | Etcd, Consul, TiKV, CockroachDB | Chubby, Spanner, DynamoDB (AWS) |

**小結**
Raft 透過強制要求「強勢的 Leader」和「連續的 Log」，犧牲了極小部分的併發寫入靈活性，換取了系統極大的可理解性和實作穩定性。

