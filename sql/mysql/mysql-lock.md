> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL 的 Lock (Row Lock, Gap Lock & Next-Key Lock)
隔離等級 `Read Committed` & `Repeatable Read` 主要是解決 Transaction 併發時，Write Transaction 影響 Read Transaction 的問題，然而除了 Write 影響 Read，還有多個 Write Transaction 併發的情境要解決，例如 `Write Skew` & `Phantom Read`。

## 上鎖語法
在 MySQL 中，一般的 `SELECT` 是非阻塞的快照讀（Snapshot Read），若要實現「讀取並鎖定」以防止其他事務修改，需使用以下語法：
*   <span style="color: orange">**`FOR UPDATE` (排他鎖/寫鎖)**</span>：
    *   告訴 MySQL：「我現在要讀這幾行，而且我**準備要更新**它們，其他人不準動」。
    *   其他事務無法對這些 Row 加任何鎖（包括讀鎖與寫鎖）。
*   <span style="color: orange">**`LOCK IN SHARE MODE` (共享鎖/讀鎖)**</span>：
    *   告訴 MySQL：「我要讀這幾行，大家可以一起讀，但**誰都別想改**」。
    *   其他事務可以加讀鎖，但無法獲取寫鎖。

## 核心概念：三種行級鎖類型

在深入探討問題前，必須先理解 InnoDB 中三種基礎的行級鎖方案：


```
索引數據：    10          20          30
位置標記：    |           |           |
              ●-----------●-----------●

┌─────────────────────────────────────────────────────┐
│  Record Lock: 只鎖記錄本身                            │
│               不鎖間隙                               │
│                      [●]                            │
│                      20                             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Gap Lock: 只鎖間隙                                  │
│            不鎖端點記錄                               │
│              (---------)                            │
│              10        20                           │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Next-Key Lock: 鎖間隙 + 右端點記錄 (左開右閉)          │
│                 InnoDB 默認鎖定方式                   │
│              (---------●]                           │
│              10        20                           │
└─────────────────────────────────────────────────────┘
```
*   `●` 代表索引記錄（實心圓點）
*   `[●]` 代表只鎖定該記錄
*   `(---)` 代表鎖定間隙，但不含兩端記錄（開區間）
*   `(---●]` 代表鎖定間隙加上右端記錄（左開右閉區間）


1.  **Record Lock (記錄鎖)**：
    *   **對象**：針對索引記錄本身上鎖。
    *   **作用**：鎖住特定的 Row，防止其他事務進行 `UPDATE` 或 `DELETE`。
    *   **範例**：`SELECT * FROM t WHERE id = 1 FOR UPDATE;` 會在 `id=1` 的索引記錄上加鎖（REC_NOT_GAP）。

2.  **Gap Lock (間隙鎖)**：
    *   **對象**：鎖住索引記錄之間的「間隙」，但不包含記錄本身。
    *   **作用**：防止其他事務在該間隙中 `INSERT` 資料，主要用於解決 Phantom Read。
    *   **範例**：若有 id 1, 5，則 `BETWEEN 1 AND 5` 的間隙鎖會防止插入 id=2, 3, 4。

3.  **Next-Key Lock (臨鍵鎖)**：
    *   **對象**：**Record Lock + Gap Lock** 的組合。
    *   **作用**：鎖住記錄本身及其前面的間隙（<span style="color: orange">**左開右閉區間**</span>）。
    *   **特性**：InnoDB 在 `Repeatable Read` 隔離級別下的<span style="color: orange">**默認鎖定單位**</span>。
    *   **範例**：若索引包含值 10, 20, 30，查詢 `WHERE id = 20 FOR UPDATE` 會產生 `(10, 20]` 的 Next-Key Lock，鎖住 10~20 的間隙以及 20 本身。



## 什麼是 Write Skew & Phantom Read 問題
### Write Skew
> [!NOTE]
> Write Skew 是多個 Transaction 同時讀取相同資料，用當下資料狀態判斷邏輯後更新，結果更新內容出現異常。
> 它是一種典型的 **Race Condition (競態條件)**，屬於 **Check-Then-Act (先檢查後執行)** 的類型。如果不加鎖，並發的事務會基於「各自看到的舊版本快照」進行判斷，導致錯誤的業務邏輯執行（如上述庫存變負數）。

例如扣庫存的 API 這樣實作：

```sql
BEGIN
  SELECT id, quantity FROM products WHERE id = ?;

  if quantity > 0 
    UPDATE products SET quantity = quantity - 1 WHERE id = ?
COMMIT
```

由於 `SELECT` 預設是<span style="color: orange">**快照讀 (Snapshot Read)**</span>，當多個 Transaction 同時執行時，它們可能都會讀取到相同的舊資料（例如 `quantity = 1`）。

這導致多個事務都通過了 `if quantity > 0` 的判斷，隨後各自執行 `UPDATE`，最終導致庫存變成負數（Lost Update）。要解決該問題，必須將讀取操作改為<span style="color: orange">**當前讀 (Current Read)**</span>，即加上 `FOR UPDATE` 語法：

```sql
BEGIN
  SELECT id, quantity FROM products WHERE id = ? FOR UPDATE; -- 加上了 FOR UPDATE

  if quantity > 0 
    UPDATE products SET quantity = quantity - 1 WHERE id = ?
COMMIT
```

此時 MySQL 會對讀取到的 products 紀錄上 row lock，先取得 lock 的 Transaction 會執行 update 成功，後面的 Transaction 會拿到更新後到 quantity 判斷 <= 0 則不 update。

除了加 `FOR UPDATE` 語法外也可寫成

```sql
UPDATE products SET quantity = quantity - 1 WHERE id = ? AND quantity > 0
```

`UPDATE` 語法同樣會上 row lock，瞬間多個 `UPDATE` 執行時也不會有問題。


### Phantom Read
> [!NOTE]
> Phantom Read 則是相同 Transaction 內，對範圍資料執行兩次 SQL 出現的結果不一樣。

例如我們想統計並處理 id 1~10 的訂單：

```sql
BEGIN
  -- 第一次查詢：原本只有 id=1, 3, 5 三筆資料
  SELECT * FROM orders WHERE id BETWEEN 1 AND 10 FOR UPDATE;
  
  -- 業務邏輯處理...
  
  -- 如果此時沒有 Gap Lock，另一個 Transaction 插入了 id=2
  -- INSERT INTO orders (id) VALUES (2);
  
  -- 再次查詢或更新：發現多了一筆 id=2 的資料（幻影）
  UPDATE orders SET status = 'DONE' WHERE id BETWEEN 1 AND 10;
COMMIT
```

原本預期只處理 3 筆，結果卻處理了 4 筆，這就是 Phantom Read。

> [!CAUTION]
> 如果只用一般的 `Record Lock` (鎖住 id=1, 3, 5) 能解決嗎？
> 答案是不能，因為 `INSERT id=2` 並沒有碰到任何現存的鎖，所以會成功插入，導致下一次查詢或更新範圍時出現「幻影」。


#### 那麼什麼樣的鎖可以解決 `Phantom Read` 問題？

MySQL 使用 <span style="color: orange">**Gap Lock (間隙鎖)**</span> 來解決 Phantom Read。它的核心思想是：**不僅鎖住存在的記錄，還要鎖住記錄之間的「空隙」，防止別人插入新資料**。

具體來說（以上述 orders 表為例）：

1.  **鎖定範圍**：
    當執行 `SELECT * FROM orders WHERE id BETWEEN 1 AND 10 FOR UPDATE` 時：
    *   除了對存在的 `id=1, 3, 5` 上 **Row Lock**。
    *   還會對 `(1, 3)`, `(3, 5)`, `(5, 10]` 這些區間上 **Gap Lock**。
    *   結果：任何想要 `INSERT` 到這些範圍的動作（例如 `id=2` 或 `id=4`）都會被阻擋，從而消滅了幻讀。

2.  **Next-Key Lock**：
    在 Repeatable Read (RR) 級別下，InnoDB 預設使用的其實是 **Next-Key Lock**，也就是 **Row Lock + Gap Lock** 的組合。
    *   它鎖定的是一個「左開右閉」的區間。
    *   例如：鎖定 `(1, 3]` 代表鎖住 1~3 的間隙以及 3 本身。

3.  **無窮大鎖定 (Supremum)**：
    如果查詢條件是 `WHERE id > 10 FOR UPDATE`，為了防止別人在 10 之後插入任何新資料（例如 `id=100`），MySQL 會鎖住 `(10, +∞)` 的範圍。這在實現上是通過鎖定一個特殊的 `supremum pseudo-record` 來達成的。

## 為什麼需要意向鎖？（效能考量）

問題場景：T1 想對整個表加 X 鎖（Table lock），但表裡很多 row 已被其他 transaction 持有 row lock。如果沒 intention lock，T1 得逐行檢查「這 row 有沒有被鎖」，這是 O(n) 災難。

意向鎖解決方案：
- T2 在某 row 加 X 鎖前，先在表上加 IX（表級）。
- T1 想加表 X 鎖時，只查表級：有 IS/IX → 直接等，不用查每行。

MySQL / InnoDB 的「鎖模式相容矩陣」（<span style="color: orange">**超重要，面試必背**</span>）：
四個縮寫代表：
- **IS（Intention Shared）意向共享鎖**  
  - 表級鎖。  
  - 意思是「這個交易**打算**在某些 row 上加共享鎖 S」。  
  - 典型來源：`SELECT ... LOCK IN SHARE MODE` 會在表上拿 IS，再對符合條件的 row 拿 S 鎖。
- **IX（Intention Exclusive）意向排他鎖**  
  - 表級鎖。  
  - 意思是「這個交易**打算**在某些 row 上加排他鎖 X」。  
  - 典型來源：`SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` 等 DML，會先在表上拿 IX，再對具體 row 拿 X 鎖。
- **S（Shared Lock）共享鎖**  
  - 可以是 row 級，也可以是表級。  
  - 拿到 S 鎖的交易可以讀，不可以改；多個 S 可以同時存在。  
  - 例如 `SELECT ... LOCK IN SHARE MODE` 在符合條件的 row 上會加 S 鎖。
- **X（Exclusive Lock）排他鎖**  
  - 可以是 row 級，也可以是表級。  
  - 拿到 X 鎖的交易可以讀寫該 row / 表，且不允許其他任何交易再對同一資源拿 S 或 X。  
  - 例如 `UPDATE ...`、`DELETE ...`、`LOCK TABLES ... WRITE` 會對目標加 X 鎖。

![Screenshot 2026-01-06 at 18.25.57](https://hackmd.io/_uploads/r1wGQP9VWg.png)

- IS/IX 之間相容（允許多個 transaction 同時宣告意向）
- 但表 X 鎖會被任何 IS/IX 阻塞
- IS、IX 彼此大多相容，因為只是「意向」，允許多個交易同時打算鎖不同行。  
- 真正會互相衝突的是 S / X 這兩種「實體讀寫鎖」，特別是 X 幾乎跟所有其他模式都不相容。

## 除了 Row Lock, Gap Lock & Next-Key Lock 還有其他種 Lock 嗎？

Lock 除了 FOR UPDATE 語法的 Write Lock 以外還有 `LOCK IN SHARE MODE` 的 Read Lock，Read Lock 互相不會卡住，但 Read & Write Lock 互相會卡住。

此外還有 Table Lock 會直接鎖住整張表，例如執行 `LOCK TABLE orders WRITE` or `LOCK TABLE orders READ` 會對整張表上寫鎖 or 讀鎖。

而在執行 Table Lock 前，要先確認 Table 中是否有 row lock，但總不能 full table scan 一筆筆檢查，因此有了 `Intention Lock`，該鎖只會跟 Table Lock 互卡，執行 `row lock` 會先對 Table 上 `Intention Lock`，`Intention Lock` 不會卡住 `row lock` 跟其他 `Intention Lock`，只卡 Table Lock，因此執行 Table Lock 時如果有其他 row lock，會被 Table 的 `Intention Lock` 卡住，不用 full table scan 檢查。

## 資料什麼時候會被上鎖?

基本上只有 `SELECT` 加上 `FOR UPDATE & LOCK IN SHARE MODE` 和 `DELETE or UPDATE` 等更新語法時會對資料上鎖，但有時上鎖範圍與方式會讓你意想不到。

假如 tasks Table 中有 10 筆待處理任務，執行 S`ELECT count(1) FROM tasks WHERE status = '待處理' FOR UPDATE` 時會只鎖那 10 筆嗎？

答案是不一定，如果 `status` 有 Index 那確實只會鎖 10 筆，但如果<span style="color: orange">**沒有 `status index` 就會 `Lock Table` 了**</span>，因為 MySQL 架構是 SQL Layer 跟 Storage Engine 分開，沒命中 index 時，要在 Storage Engine Full Table Scan 後交由 SQL Layer 過濾資料，而上鎖必須在 Storage Engine 完成。

Gap Lock 主要發生在範圍查詢 `WHERE id > 10 FOR UPDATE` or 查詢非 unique 索引 `WHERE status = 1 FOR UPDATE` ，但如果查詢條件沒有命中資料時 `WHERE id = 10 FOR UPDATE` ， id 是 `unique primary key`，但 `id = 10` 資料不存在，也有 `Gap Lock` 並會鎖住 `id=10` 前一筆跟後一筆資料的間隙，如果前一筆是 5 後一筆是 15 會鎖 5~15 範圍導致 `INSERT INTO t (id) VALUES (11)` 卡住。

此外，`UPDATE & DELETE` 不存在資料，甚至 `INSERT ON DUPLICATE KEY UPDATE` 到不存在資料直接 `INSERT` 時也都會上 Gap Lock。

最後是 Isolation Level 如果設定成 Serializable，雖能解決上述所有一致性問題 (Dirty Read, Non-repeatable Read, Write Skew & Phantom Read)，但會自動把所有 `SELECT` 加上 `LOCK IN SHARE MODE` 的讀鎖。

## MySQL 如何避免 DeadLock

使用 Lock 最大的風險就是 DeadLock，最常見案例是「互相持有並等待」：

```sql
-- transaction A:
BEGIN
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
...  
COMMIT;  
  
-- transaction B:  
BEGIN
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
...  
COMMIT;
```

當 A & B Transaction 同時執行，A 持有 `users id=1` 資源並等待獲取 `users id=2` 資源，而 B 持有 `users id=2` 資源並等待 `users id=1` ，雙方互相等待對方釋放資源，造成 DeadLock。

該 DeadLock 解法很簡單，將 Lock 順序改成一致就可以了：

```sql
-- transaction A:  
BEGIN
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
...  
COMMIT;  
  
-- transaction B:  
BEGIN
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
...  
COMMIT;
```

又或者可改成 `SELECT * FROM users WHERE id IN (1, 2) FOR UPDATE;` 

但當：

`SELECT * FROM users WHERE id IN (1, 2) FOT UPDATE;` 以及
`SELECT * FROM users WHERE id IN (2, 1) FOT UPDATE;` 同時執行會 DeadLock 嗎？ `IN` 裡面的參數順序不同會導致 Lock 順序不一致嗎？

## 我們如何知道執行中 SQL 是怎麼上鎖的

`performance_schema` DB 中 `data_locks` table 可用來看當前所有 Lock 詳細狀況，該 Table Schema 為：

![Screenshot 2026-01-05 at 22.26.47](https://hackmd.io/_uploads/SkAb5HYEWl.png)

例如：

```sql
-- 建立一個 users table 有唯一主鍵 id 以及非唯一 index name  
create table users (  
	id int auto_increment,  
	name varchar(100),  
	gender int,  
	key (name),  
	primary key (id)  
);  
  
-- 建立三筆資料  
insert into users (id, name, gender) values  
(1, 'apple', 0),(2, 'banana',1),(3, 'apple',1);  
  
-- 對 id = 1 唯一索引資料上鎖  
begin;  
select * from users where id = 1 for update;  
....  
```

在 transaction 不 commit 情況下執行 `select * from  performance_schema.data_locks;` 可看到：

![Screenshot 2026-01-05 at 22.28.15](https://hackmd.io/_uploads/BJIFcHt4Ze.png)
![Screenshot 2026-01-05 at 22.28.23](https://hackmd.io/_uploads/ryRtqBYEZg.png)

分別有兩個 Lock：
 1. **Intention Lock**: 第一個是 Table Lock 且 Lock Mode IX 代表 Intention Exclusive Lock。
 2. **Row Lock**: 第二個是使用 PRIMARY Index 的 Record Lock 其 Lock Mode 為 **X,REC_NOT_GAP**，X 代表 Exclusive Lock 而 REC_NOT_GAP 代表不是 Gap Lock 只鎖單一 Record 不影響 `INSERT` 行為，最後 LOCK_DATA 代表被鎖住的 Index 資料，也就是 `id=1` 的資料。

接下來：

```sql
-- 對 name = 'apple' 非唯一索引資料上鎖  
BEGIN;  
select * from users where name = 'apple' for update;  
...
```

執行 `select * from performance_schema.data_locks;` 可看到：

![Screenshot 2026-01-05 at 22.42.59](https://hackmd.io/_uploads/r1dkCBtNWg.png)

出現四個 Lock：

1. **Intention Lock** : TABLE 的 Intention Lock 。
2. **name Index Lock** : 使用 name Index 對兩筆 name=apple 資料上 3. **Exclusive Record Lock**，但因為是非唯一索引，會上 Gap Lock 所以沒有 **REC_NOT_GAP**
3. **PRIMARKY Lock** : 同時也需要對 PRIMARY Index id=1 & id=3 上 Exclusive Record Lock，但 PRIMARY 是唯一索引沒有 Gap Lock 所以有 **REC_NOT_GAP**
4. **Gap Lock** : 最後會在 `name` Index 上一個 Exclusive Gap Lock，由於 name 排序是 apple => banana，所以 Gap Lock 會鎖住這段 [apple, banana] (包含 apple，不包含 banana) 範圍的 `insert` 行為

隨後執行，`INSERT INTO users (name) VALUES ('apple'), ('apple2')` 在 [apple, banana] 範圍 Insert 資料，卡住後會發現 Lock 多了兩個：

![Screenshot 2026-01-05 at 23.06.08](https://hackmd.io/_uploads/BJCEmUFVbe.png)

1. **Intention Lock**: 多一個 Intention Lock，Intention Lock 跟 Intention Lock 不會互斥，所以是 GRANTED 狀態
2. **WAITING Lock**: 多一個 Exclusive, Gap, Insert Intention Lock 狀態是 WAITING，代表 Insert 操作被其他 Gap Lock 卡住了。

另外如果將 `select * from users where name = "apple" for update;`換成 `SELECT * FROM users WHERE name = "banana" FOR UPDATE`，由於排序上 banana 是最後一筆資料，Gap Lock 會變這樣：

![Screenshot 2026-01-05 at 23.11.21](https://hackmd.io/_uploads/Hks_EIY4-l.png)

supremum pseudo-record 代表向上無限衍生且不存在的紀錄 ，對該紀錄上 Exclusive Lock 會導致 INSERT 資料若排序在 banana 後面都會卡住 (e.g INSERT INTO users (name) VALUES ('dog'), ('cat') )。

總結幾個常見的 Lock Mode

- X : Exclusive Lock 也就是 Write Lock
- S : Shared Lock 也就是 Read Lock
- I : Table Intention Lock
- REC_NOT_GAP : 只鎖單筆資料，不鎖範圍
- GAP : 不鎖單筆資料，鎖一個範圍不能 Insert 新資料
- INSERT_INTENTION : 代表 Insert 正在等其他 Gap Lock 釋放
- X 搭配 supremum pseudo-record : 對無限延伸的範圍上 Gap Lock

## MySQL Lock 的執行順序

當執行

```sql
BEGIN;  
SELECT * FROM users WHERE id IN (1, 2, 3) FOR UPDATE;  
...  
```

上圖，可看到 id (1,2,3) 都有上鎖成功，然後執行

```sql
BEGIN;  
SELECT * FROM users WHERE id IN (3, 2, 1) FOR UPDATE;  
...
```

理論上，如果上鎖順序是相反，會拿到 `id=3` 的鎖並卡住，但實際上如下圖所示，實際執行是拿到 `id=1` 的鎖卡住，由此可見上鎖順序跟 SQL 寫法無關。

![Screenshot 2026-01-06 at 18.32.05](https://hackmd.io/_uploads/BJKoNwqNZx.png)

## 上鎖順序跟什麼有關?

跟 <span style="color: orange">**Index Tree Scan 順序**</span>有關，id 的 Index 順序是 1->2->3 ，上鎖順序同樣會是 1->2->3，但如果第二個 SQL 加上 `ORDER BY` 上鎖順序就會相反了：

```sql
BEGIN
SELECT * FROM users WHERE id (1, 2, 3) ORDER BY id DESC FOR UPDTE;
```

如上圖，會拿到 id=3 的鎖並卡住。

除了 `ORDER BY` 會影響上鎖順序外，建立 Index 時使用逆序設定也會影響：


### 將所有 index 改成 unique，並將 nick_name index 順序設定成 
```sql
DESC  
create table users (  
	id int auto_increment,  
	name varchar(100),  
	gender int,  
	nick_name varchar(100),  
	unique key (name),  
	unique key (nick_name DESC),  
	primary key (id)  
);

insert into users (id, name, nick_name, gender) values
 (1, 'apple', 'apple', 0),(2, 'banana', 'banana', 1);
 
begin;  
select * from users where name IN ('apple', 'banana') for update;  
....  
  
begin;  
select * from users where nick_name IN ('apple', 'banana') for update;  
....
```

此時查詢 Lock 狀態會發現 (上圖)，由於 `nick_name` Index 是 `Desc`，所以會從 id=2 的資料開始上鎖，跟 `name` Index 順序相反，導致上面 SQL 看起來順序一致，但同時執行時卻造成了 DeadLock。

## 除了順序情境外，還有什麼會造成 DeadLock 的情境嗎？

在併發寫入情境中，更新順序若相反也會造成 DeadLock，但其實寫入更常見的 DeadLock 是 Gap Lock，例如：

```sql
-- transaction A  
begin;  
UPDATE users SET gender=0 WHERE id = 1;  
if users not found:  
	INSERT INTO users (gender, id) VALUES (0 , 1);  
  
-- transaction B  
begin;  
UPDATE users SET gender=1 WHERE id = 2;  
if users not found:  
	INSERT INTO users (gender, id) VALUES (0 , 2);
```

當兩個 Transaction 都 UPDATE 不存在資料時，都會上 Gap Lock 且因為 users 表沒資料，範圍就是 (無限小, 無限大)，此時 A & B Transaction 在執行 INSERT 時就會被對方的 Gap Lock 卡住變成 DeadLock。

另外就是加上面 SQL 精簡成 INSERT INTO users (id, gender) VALUES (1, 0) ON DUPLICATE KEY UPDATE gender=0; ，如果 UPDATE 不存在的值會上 Gap Lock 並 INSERT ，此時若多個 Transaction 執行：

```sql
-- transaction A  
INSERT INTO users (id, gender) VALUES (1, 0) ON DUPLICATE KEY UPDATE gender=0;  
  
-- transaction B  
INSERT INTO users (id, gender) VALUES (2, 0) ON DUPLICATE KEY UPDATE gender=1;  
  
-- transaction C  
INSERT INTO users (id, gender) VALUES (3, 0) ON DUPLICATE KEY UPDATE gender=0;
```

1. `transaction A` 獲取 Gap Lock 並進入 `insert`
2. `transaction B & C` 同時獲取 Gap Lock ，並等待 t`ransaction A insert`  行為結束，才能 `insert`
3. `transaction A commit` 釋放 Gap Lock ，但由於 `transaction B & C` 都獲取了 Gap Lock，當他們兩繼續執行 `insert` 行為時，會被彼此的 Gap Lock 卡住，導致 DeadLock

**要解決上述 DeadLock 問題很簡單，就是先執行 INSERT ，出現 ON DUPLICATE KEY ERROR 在執行 UPDATE ，若想避免頻繁出現 ON DUPLICATE KEY ERROR，可用 in memory cache 紀錄一個 flag。**

## Foreign Key 造成的 DeadLock 案例：

`customers` 跟 `orders` 是一對多關係，`orders` 表中有 `customer_id` 欄位關聯到 `customers` 表的 `id`。

執行 `INSERT INTO orders (customer_id) VALUES (123);` 時，為確保 customers.id=123 這筆資料存在， MySQL 會對 customers id=123 這筆資料上讀鎖，因此如果執行：

```sql
-- transaction A  
BEGIN:
INSERT INTO orders (id, customer_id) VALUES (1, 123);
UPDATE customers SET count=count+1 WHERE id = 123;

-- transaction B
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (2, 123);
UPDATE customers SET count=count+1 WHERE id = 123;
```

Transaction A & B 同時執行，A & B 同時 INSERT orders 成功並對 customers 上讀鎖，隨後 A & B 執行 `UPDATE customers` 時就會被彼此的讀鎖卡住，造成 DeadLock。

## 發生 DeadLock 時，MySQL 會怎麼處理？

MySQL 有 `DeadLock Detection` 的功能，會主動偵測 DeadLock 並 Rollback 其中一個 Transaction，讓另一個 Transaction 順利執行，可透過設定 <span style="color: orange">`innodb_lock_wait_timeout`</span> 參數調整 Transaction 卡住的時間，如果 Transaction 卡住時間超過該參數就會被強制 Rollback。

MySQL 還會紀錄最後一次 DeadLock 的詳細資訊，可以透過執行 `SHOW ENGINE INNODB STATUS` 指令獲取 InnoDB 的詳細資訊，其中 `LATEST DETECTED DEADLOCK Section` 為會紀錄最後兩個造成 DeadLock 的 Transaction 資訊：

```sql
------------------------  
LATEST DETECTED DEADLOCK  
------------------------  
2020-12-26 00:05:14 0x7f9657cf9700  
*** (1) TRANSACTION:  
TRANSACTION 14048, ACTIVE 1 sec starting index read  
mysql tables in use 1, locked 1  
LOCK WAIT 11 lock struct(s), heap size 1136, 6 row lock(s), undo log entries 2  
MySQL thread id 54, OS thread handle 140283242518272, query id 45840 172.22.0.1 api-server updating  
update `products` set `sold` = 32 where `id` = '919'  
  
*** (1) HOLDS THE LOCK(S):  
RECORD LOCKS space id 3 page no 8 n bits 336 index PRIMARY of table `online-transaction`.`products` trx id 14048 lock mode S locks rec but not gap  
Record lock, heap no 259 PHYSICAL RECORD: n_fields 7; compact format; info bits 0  
0: len 4; hex 00000397; asc ;;  
1: len 6; hex 0000000036d7; asc 6 ;;  
2: len 7; hex 010000013f1e26; asc ? &;;  
3: len 21; hex 50726163746963616c204672657368204d6f757365; asc Practical Fresh Mouse;;  
4: len 4; hex 800000b1; asc ;;  
5: len 4; hex 800000fe; asc ;;  
6: len 4; hex 80000020; asc ;;  
  
  
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:  
RECORD LOCKS space id 3 page no 8 n bits 336 index PRIMARY of table `online-transaction`.`products` trx id 14048 lock_mode X locks rec but not gap waiting  
Record lock, heap no 259 PHYSICAL RECORD: n_fields 7; compact format; info bits 0  
0: len 4; hex 00000397; asc ;;  
1: len 6; hex 0000000036d7; asc 6 ;;  
2: len 7; hex 010000013f1e26; asc ? &;;  
3: len 21; hex 50726163746963616c204672657368204d6f757365; asc Practical Fresh Mouse;;  
4: len 4; hex 800000b1; asc ;;  
5: len 4; hex 800000fe; asc ;;  
6: len 4; hex 80000020; asc ;;  
  
  
*** (2) TRANSACTION:  
TRANSACTION 14052, ACTIVE 1 sec starting index read  
mysql tables in use 1, locked 1  
LOCK WAIT 11 lock struct(s), heap size 1136, 6 row lock(s), undo log entries 2  
MySQL thread id 57, OS thread handle 140283970258688, query id 45841 172.22.0.1 api-server updating  
update `products` set `sold` = 34 where `id` = '919'  
  
*** (2) HOLDS THE LOCK(S):  
RECORD LOCKS space id 3 page no 8 n bits 336 index PRIMARY of table `online-transaction`.`products` trx id 14052 lock mode S locks rec but not gap  
Record lock, heap no 259 PHYSICAL RECORD: n_fields 7; compact format; info bits 0  
0: len 4; hex 00000397; asc ;;  
1: len 6; hex 0000000036d7; asc 6 ;;  
2: len 7; hex 010000013f1e26; asc ? &;;  
3: len 21; hex 50726163746963616c204672657368204d6f757365; asc Practical Fresh Mouse;;  
4: len 4; hex 800000b1; asc ;;  
5: len 4; hex 800000fe; asc ;;  
6: len 4; hex 80000020; asc ;;  
  
  
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:  
RECORD LOCKS space id 3 page no 8 n bits 336 index PRIMARY of table `online-transaction`.`products` trx id 14052 lock_mode X locks rec but not gap waiting  
Record lock, heap no 259 PHYSICAL RECORD: n_fields 7; compact format; info bits 0  
0: len 4; hex 00000397; asc ;;  
1: len 6; hex 0000000036d7; asc 6 ;;  
2: len 7; hex 010000013f1e26; asc ? &;;  
3: len 21; hex 50726163746963616c204672657368204d6f757365; asc Practical Fresh Mouse;;  
4: len 4; hex 800000b1; asc ;;  
5: len 4; hex 800000fe; asc ;;  
6: len 4; hex 80000020; asc ;;  
  
*** WE ROLL BACK TRANSACTION (2)
```

以上面的資訊為例，我們來試著分析 DeadLock 的解法：

1. 首先 SQL 執行語法為：(1) Transaction 為 update `products` set `sold` = 32 where `id` = 919 (2) Transaction 為 update `products` set `sold` = 34 where `id` = 919
2. 分析 Transaction 1 Lock 狀況：
    - (1) HOLDS THE LOCK(S) 顯示為 lock mode S locks rec but not gap 代表 Transaction 1 有拿到該資料的讀鎖權限
    - (1) WAITING FOR THIS LOCK TO BE GRANTED 顯示 lock_mode X locks rec but not gap waiting 代表 Transaction 1 在等待寫鎖權限，且不是因為 Gap Lock 被卡住。
3. 分析 Transaction 2 Lock 狀況：
    - (2) HOLDS THE LOCK(S) 和 (2) WAITING FOR THIS LOCK TO BE GRANTED 的資訊顯示跟 Transaction 1 狀況一樣。

透過上面資訊可以推導，這兩個 Transaction 都拿到了讀鎖，並往下執行 `UPDATE` 獲取寫鎖時互相卡住了，所以完整的 Transaction 應該長這樣：

```sql
-- Transaction 1  
begin;  
SELECT id, sold FROM products WHERE id = 919 LOCK IN SHARE MODE;  
...  
UPDATE products SET sold = 32 WHERE id = 919  
commit;  
  
-- Transaction 2  
begin;  
SELECT id, sold FROM products WHERE id = 919 LOCK IN SHARE MODE;  
...  
UPDATE products SET sold = 34 WHERE id = 919  
commit;
```

邏輯看起來是同時賣出相同產品，該情況有兩種解法：

1. 將 `LOCK IN SHARE MODE` 改成 `FOR UPDATE` 確保先拿讀鎖，誰先拿到先執行 `UPDATE`
2. 移除 `SELECT` 語法，使用 `UPDATE products SET sold = sold + ? WHERE id = 919 AND sold >= ?` 效能較好
