# 利用 MySQL Partition 以優化大表查詢

當資料表到千萬筆資料即便有 Index 查詢速度仍可能不理想，就像 O(LogN) 的演算法，當 N 很大時程式仍會執行很慢，若要程式執行變快，其中一個方式是把 N 拆成多個小的子集，然後並行處理，沿著這個思路，是否也可把一個大表拆成多個小表呢？

常見的分表策略是把一張 `orders` Table 依照時間拆成 N 張小表 `202505_orders` ＆ `202504_orders` ，如果查詢 2025/5 月的訂單只要去 `202505_orders` Table 查詢，這時候 N 就變小了，速度就快多了，缺點是程式端需要修改邏輯，尤其需要跨表搜尋一整年訂單時邏輯會變得複雜。

## 那麼有沒有不需修改程式端邏輯就能有分表效果的功能呢？

**Partition 是 MySQL 的拆表技術，可將一個 Table 分成多個 idb 檔案，這些 idb 稱為 Partition，每個 idb 檔案有獨立的 Clustered & Secondary B+Tree**，這些 idb 檔案共用同一個 metadata 資料且被視為同一張表，程式端不需要修改 SQL 邏輯，因為 MySQL 會依照查詢條件判斷去哪個 idb 檔案中搜尋。

## 使用 Partition 時，拆分資料的依據是什麼?

首先要指定 Partition Key，可選單個或多個欄位，不同 Partition Key Value 會被分到不同 idb 檔。

但 Partition Key 的選擇有些限制：

Partition Key 必須是所有 Unique Key 的子集

例如：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	uuid VARCHAR(100),  
	created_at DATETIME,  
	PRIMARY KEY id,  
	UNIQUE KEY idx_uuid uuid  
);
```

若要用 `created_at` 作為 orders Table Partition 的 Key 是不行的，因為 PK & Unique Key `idx_uuid` 都沒有包含到 `created_at` ，因此需要改成：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	uuid VARCHAR(100),  
	created_at DATETIME,  
	PRIMARY KEY (id, created_at),  
	UNIQUE KEY idx_uuid (uuid, created_at)  
);
```

原因在於跨 Partition 檢查唯一值效能太差，假設 PK 不包含 `created_at` ，用 created_at 切 Partition 時，為了檢查 PK 的唯一性，INSERT 就需要跨 Partition 檢查，而跨 Partition 會多查詢 I/O 效能較差。

如果所有 Unique Index 都有 created_at 欄位就只要在單一 Partition 檢查唯一性，因為跨 Partition 的 created_at 值肯定不一樣。

**Partition Key 不能為 BLOB 或 TEXT**

因為上述限制，Partition Table 的 PK 都需要有 Partition Key，如果 Partition 為 BLOBL 或 TEXT，PK 就會變得很大，所有 Secondary Index Tree 的 Leaf Node 也會變大，表的總體資料量會變多不好管理，這也是使用 Partition 的缺點之一，PK 會變大，如果 Index 太多，總體資料量會變大很多。

**Partition Table 不能有 Foreign Key**

除了 Partition Key 限制以外，Table 本身也有限制，不能有 Foreign Key 跟不能當作其他 Table 的 Foreign Key，原因也跟跨 Partition 查詢有關。

Foreign Key 有 Referential Integrity Check，不論對 Parent 或 Child Table 資料操作都需要檢查關連表的對應資料，當 Foreign Key 不是 Partition Key 時，檢查 Referential Integrity Check 就會跨 Partition 查詢，效能很差。

## 指定完 Partition Key 後，我如何拆分資料？

MySQL 提供的四種 Partitioning Types (Range, List, Hash & Key)：

**Range Partitioning  範圍分區**: 依照資料不同連續範圍分區

例如依照  `orders` Table 的 `created_at` 欄位，每一年分一個 partition：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,
	PRIMARY KEY (id, created_at)
)  
PARTITION BY RANGE ( YEAR(created_at) ) (  
PARTITION p0 VALUES LESS THAN (2024),  
PARTITION p1 VALUES LESS THAN (2025),  
PARTITION p2 VALUES LESS THAN MAXVALUE,  
);
```

List Partitioning  清單分區 : 定義不同 Value 組別分區

例如需要用 orders Table 裡面的 zip_code 依照不同地區分類：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,
	PRIMARY KEY (id, zip_code)
)  
PARTITION BY LIST ( zip_code ) (  
PARTITION pNorth VALUES IN (1,2,3),  
PARTITION pSouth VALUES IN (4,5,6),  
PARTITION pEast VALUES IN (7,8,9),  
PARTITION pCentral VALUES IN (10,11,12)  
);
```

List 會限制 `INSERT` 行為，如果有 `zip_code` 無法被分區的情況會噴錯。

**Hash Partitioning  哈希分區**：依照 partition 數量透過 hash value 自動分區。

Range 或 List 會要手動擴充或調整 Partition，例如加新的一年到 Range Partition 或者新增新的 `zip_code` 到 List Partition，如果是單純要資料拆成固定 N 個子集，例如將 orders 依照 user_id 隨機分成 5 個 partition，可用 Hash：

```sql!
CREATE TABLE orders (  
   id INT,   
   user_id INT,   
   zip_code INT,  
   created_at DATETIME,
   PRIMARY KEY (id, user_id)
)  
PARTITION BY HASH ( user_id )  
PARTITION 5;
```

Hash Partitioning 用取餘數的方式 ( e.g `id % 5` ) 將資料分配到不同的 partition，若 Partition 欄位非數字型別，會用 Hash 演算法 (e.g CRC32) 將資料轉成數字後取餘數分配。

Key Partitioning  金鑰分區 ：更高效的 Hash Partitioning

Hash Partitioning 只能用單欄位作為 Partition Key，且 Hash 算法較簡單容易碰撞，Key Partitioning 可用多個欄位搭配更複雜的 Hash 演算法，能更平均地分配資料，但也花更多效能。

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,
	PRIMARY KEY (id, user_id)  
)  
PARTITION BY KEY ( user_id, id )  
PARTITION 5;

```

## 可以用多個 Partition Types 組合嗎？

用 Sub Partition 功能可組合多欄位跟不同 Partition Type，例如：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,
	PRIMARY KEY (id, user_id),
	UNIQUE KEY (id, created_at)
) PARTITION BY RANGE ( YEAR(created_at) )  
SUBPARTITION BY HASH(user_id)  
(  
	PARTITION p2023 VALUES LESS THAN (2024)  
		SUBPARTITIONS 3,  
	PARTITION p2024 VALUES LESS THAN (2025)  
		SUBPARTITIONS 3  
);
```

先依照 created_at 每年建立 Partition，每年的 Partition 底下再依照 user_id 建立三個 Hash Partition，總共會建立 6 個。

## 如果要調整 Partition 怎麼辦？

除了使用的 CREATE TABLE 建立 Partition 外，其他操作主要為 Drop & Modify ，但不是每個指令都能用於所有 Partition Types :

- List & Range Partition Management :
- Drop Partition：移除 Partition 資料，有兩種 Algorithm - Inplace & Copy：
    - Inplace 是將 Partition 資料直接移除
    - Copy 則是會將資料嘗試搬移到其他 Partition，如果無法搬遷則移除
- Add Partition：新增 Partition，Range 只能往尾端範圍新增，而 List 只能新增新的 Group。
- REORG Partition：調整既有 Partition 內容，例如往 Range 中間段新增，或者調整 List 既有 Group 內容。

**Hash & Key Partition Management :**
Hash & Key 的操作相對簡單，透過 COALESCE 減少數量，透過 ADD 增加數量。

但增加或減少 Partition 會觸發資料重新分配，例如取餘數 `x % N` 的 N 變動導致所有資料要重新計算並分配到新的 Partition，資料量大會非常耗時。

Linear Hash or Linear Key Partitioning 可以減少 Partition 數量變動時需要搬移的資料量：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,
	PRIMARY KEY (id, user_id)
)  
PARTITION BY LINEAR HASH ( user_id )  
PARTITION 5;
```

Linear 是用 bit operation 來分配 Partition，假設 Partition 數量為 4，可以用二進制 00, 01, 10, 11 分別代表不同的 Partition 編號，然後用 AND operation `partition_key & 11 (n-1)` 找出被分配的 Partition，當 Partition 數量變成 8 會多出一個 bit` partition_key & 111`，此時最多只有一半的資料要分配到新 Partition，例如 partition_key 是 `001`，`001 & 11` 跟 `001 & 111` 結果都是 `001` 不用重分配，只有多出的 bit 為 1 (e.g `101`) 時需要重分配。

Linear 缺點就是 Partition 數量要為 2 的次方數，且 bit operation 會比取餘數花更多效能。

此外上述調整 Partition 內容都會涉及 Table Lock，可用 pt-online-schema-change 工具避免 Lock 太久，但如果是 Drop Partition，MySQL 支援一種更快更安全的方式，就是 Partition Exchange：

當需要 Drop 太舊的 Range 或者 List Partition，可先建立一張空 Table，該表不用 Partition 但 Schema 內容要跟 Partition Table 一模一樣，隨後執行 Exchange 指令將 Partition 與 Table 資料交換：

```sql!
ALTER TABLE orders EXCHANGE PARTITION p_2000 WITH TABLE 2000_orders;
```

**Exchange** 指令只更改 Table Metadata 不搬移 Table 資料，因此 Table Lock 時間非常短暫，執行完後 `orders` 表的 p_2000 會變空，而 `2000_orders` 表會有原本 p_2000 的資料，此時 Drop 空的 p_2000 Partition 會非常快，而 Drop `2000_orders` 表也不會影響正在運行的 `orders` 表。

## 那麼使用 Partition 有什麼缺點嗎？

首先是 **跨 Partition 查詢效能很差**，資料分散到不同 idb 中的 B+Tree ，除了要建立更多 File Socket 外還會增加 I/O 次數，如果資料量不大切 Partition 效能反而會更差。

如果切完 Partition 但需要 Full Table Scan 建議使用 explicit partition selection 加程式端並行查詢：

```sql!
thread 1  
SELECT * FROM orders PARTITION (p1);  
thread 2  
SELECT * FROM orders PARTITION (p2);  
thread 3  
SELECT * FROM orders PARTITION (p3);
```

另外 Secondary Index Tree 也會被切成 Partition，因此即便 Table 有獨立的 Non Unique Index，單獨使用 Index 查詢效能不會提升太多：

```sql!
CREATE TABLE orders (  
	id INT,  
	user_id INT,  
	zip_code INT,  
	created_at DATETIME,  
	PRIMARY KEY (id, user_id),  
	KEY idx_created_at (created_at)  
)  
PARTITION BY HASH ( user_id )  
PARTITION 5;
```

即便 `orders` Table 有 `idx_created_at` Index，執行 `SELECT * FROM orders WHERE created_at = ?` 因為跨 Partition 查詢效能會變差，要使用 explicit partition selection

```sql!
thread 1  
SELECT * FROM orders PARTITION (p1) WHERE created_at = ?;  
thread 2  
SELECT * FROM orders PARTITION (p2) WHERE created_at = ?;  
thread 3  
SELECT * FROM orders PARTITION (p3) WHERE created_at = ?;
```

或查詢條件加上 partition column `SELECT * FROM orders WHERE user_id = ? AND created_at = ?` 。

