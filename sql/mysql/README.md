# MySQL 架構

MySQL 架構分為
- SQL layer
- Storage Engine

## SQL Layer 負責：

- Client 連線管理和請求解析
- SQL 語法解析和制定執行計劃
- 實作 SQL function (SUM, AVG etc)
- 管理 Stored Procedures

## Storage Engine 負責：

- 資料儲存，查詢與寫入
- 提供 ACID & Index 功能

![Screenshot 2025-10-15 at 15.04.35](https://hackmd.io/_uploads/rkNpPThTee.png)

Application 透過 API (e.g JDBC) 與 SQL Layer 交互，而 SQL Layer 透過內部 API 操作 Storage Engine 中的資料，分層的好處是在不修改外部介面的情況下，能替換不同功能的 Storage Engine，MySQL 目前主流的 Storage Engine 為 InnoDB。

# InnoDB 架構

![Screenshot 2025-10-15 at 15.08.58](https://hackmd.io/_uploads/Byhdu63pxg.png)
(圖片來源：https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966)

依照 InnoDB 架構圖，資料分別存在記憶體跟硬碟中，放在硬碟是確保資料不遺失，而為了加速查詢會把查詢過的資料放到記憶體。

可以看到在 on-disk structure 中有許多 tablespces 的儲存元件，實際資料就會儲存在此。

## Tablespace 的用途

為何需要有 table spaces，而非單純將資料儲存於檔案中。
其實資料確實是存在檔案中的，最單純的 `insert` 指令發生時，`write` syscall 會把內容寫進檔案，但反覆的執行 `write` syscall 會造成硬碟空間碎片化，因為，OS 執行 `write` 時，如果發現檔案空間不夠就會隨機分配磁碟空間。

這樣會造成資料散落在多個不連續的磁碟區塊，當要讀取多筆資料時，會有較多的 disk io 使得讀取速度變慢。

為了解決這個問題，InnoDB 會執行 `fallocate` syscall 向 OS 申請一塊連續的磁碟空間，稱為 Extent (default 1MB)。

為了有效利用這塊連續空間，系統需要紀錄檔案中有哪些區塊是可用的，而可用的區塊紀錄就是 Tablespace 負責的事情，其中包含
1. 紀錄哪些區塊可用
2. 哪些區塊有資料
3. 當資料被刪除後，重新標記可用區塊

此外 InnoDB 實際寫入或更新時，大部分情況會用 pwrite syscall :

- write : 往檔案最後面寫入資料，順序寫入效能好，但併發時會競爭鎖
- pwrite : 指定檔案寫入位置，修改特定檔案內容，不同檔案位置修改不需競爭鎖
有了 Tablesapce 管理 Extent，就能在寫入時指定不同硬碟區塊執行 pwrite，併發時效能更好。

### 不同 Tablespace 用途

- System Tablespace : InnoDB 儲存各種優化或 ACID 功能會用到的資料，以及 Table Schema 的內容。

- **File-Per-Table Tablespace**：`innodb_file_per_table]]= ON` (預設為 ON)時使用該 Tablespace 儲存 **Table ＆ Index** 資料的空間，每個 Table 有獨立檔案，該檔案為 `.idb` 檔。

- General Tablespace：用戶自定義的 Tablespace，CREATE TABLE 時可指定儲存的 Tablespace，為 Shared 模式，多個 Table 會存在同個檔案中。

- Temporary Tablespace：存放執行查詢中的臨時資料，例如 subquery, group by, distinct 等，當資料量太大無法在 in-memory 完成時就必須將部分資料放到硬碟，該 Tablespace 資料會在重啟時刪除。

### 為何預設要用 File-Per-Table TableSpace？

當 innodb file per tablespace 為 off 時，所有 table 資料會存到 system tablespace 並用一個檔案儲存所有 table 資料，也就是 Shared 模式，此外 MySQL 5.7 之前也只有 Shared 模式。

但 Shared 模式有以下缺點

1. **會造成資料碎片化**
所有表存在同個 Tablespace 相當於一個連續空間的 extent 會有多張表的資料，導致一張表的資料更容易跨多個 extent，造成資料散落在不同硬碟區塊。

2. **備份還原資料時無法指定表，只能備份還原整個 DB 較耗時**
目前備份還原工具都是針對檔案備份還原，Shared 模式下無法針對特定幾張表備份還原，因為單個檔案中包含很多表資料，要備份部分表，需要解析檔案內容，如果操作失誤會弄壞整個資料庫。

3. **刪除表後硬碟空間無法還給OS**
此外 Shared 模式下無法把多餘的硬體空間還給 OS，因為 file system 是用連續的 bytes 讀寫檔案，如果中間某段資料被刪除了，需要把後面所有資料往前移動，才能把被刪除的空間還給 OS，而移動資料會花大量時間，InnoDB 不因為刪除而移動資料，因此即便你刪除了某張表，Shared 模式下也無法釋放這張表佔用的空間。

因此 5.7 之後提供 file per table 模式解決上述問題，但該模式也有對應的缺點，當檔案變多，需要建立更多 file socket object ，而每個 file socket object 會在 kernel 初始化 buffer cache 用於資料讀寫的緩衝，因此 file socket 太多帶來記憶體壓力。

為了解決 file socket 太多的缺點，8.0 後將原本每個 table 會有各自的 frm file 儲存 metadata (e.g schema) 以及 idb file 儲存資料，改成把所有 table metadata 統一放到 system tablespace 只留下 idb file。

此外也可以建立 general tablespace 將多個小 table 統一放到同一個檔案中。

# 資料格式

Tablespace 除了管理 extent 外，也負責將抽象化結構映射到硬碟區塊，因此 extent 中的資料不能隨意儲存，要有特定資料結構優化插入查詢的效能，以及特定格式將 bytes 內容解析成帶有欄位意義的資料。

MySQL 設計目的是要能符合頻繁查詢和寫入的情境，因此需要查詢 & 寫入時間複雜度都不能太差的結構。

首先，要從大量資料中查詢單筆資料，最快非 Binary Search 莫屬，但用 Bianry Search 資料需要有序，若用有序陣列，每次插入需要移動陣列元素，並不高效，而 Binary Search Tree (BST) 插入時只要找到位置後修改指標內容，複雜度為 O(LogN)。

但如果有序插入到 BST，會把 Binary Tree 變成 Linked List，插入搜尋會變 O(N)，因此要用平衡 BST (e.g AVL Tree) 確保不論如何插入資料，左右子樹高最多差 1 ，雖然插入會多一個旋轉的操作，但插入和查詢複雜度都為 O(LogN)，是查詢跟寫入都不會太差的結構。

![Screenshot 2025-12-10 at 18.42.08](https://hackmd.io/_uploads/HJpvRpLzbl.png)

(圖片來源：https://ithelp.ithome.com.tw/m/articles/10353866)

## MySQL 使用 B+Tree 而非 AVL Tree

AVL Tree 雖儲存在硬碟中，但 Traversal 時要把節點讀到記憶體解析後才能往下個節點找，因此節點越多讀取 I/O 次數越多，優化方式是一次讀取多個節點 (i.e Sub Tree)，但前提要用連續硬碟空間儲存多個節點。

而 B+Tree 正是將多個有序資料壓縮成一個節點且葉節點彼此左右相連的平衡 BST ，B+Tree 的節點稱為 Page 預設為 16 KB 的連續硬碟空間，Treversal 時一次 I/O 能讀到多筆資料，有效降低 I/O 次數。

### B+Tree 節點 (Page) 裡的結構

Page 裡不是以 AVL Tree 的方式做存儲，原因是 AVL Tree 有多個指標和要記錄樹高，佔用較多空間，Page 結構目的是塞越多資料越好，盡可能降低 Traversal 時的 I/O 次數，因此結構更像是有序陣列。

![Screenshot 2025-12-10 at 18.45.13](https://hackmd.io/_uploads/r1eXkALzZg.png)

(圖片來源：https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)

**Page 結構**

![Screenshot 2025-12-10 at 19.00.23](https://hackmd.io/_uploads/rybhG08MZe.png)

- header：紀錄 Page Metadata 例如 ID，指向 Child & Sibling Node 的指摽，Page Type (Leaf or Non-Leaf) ，最後更新的 LSN 以及 Free Space 大小等。
- infimum (無窮小紀錄) & supremum (無窮大紀錄)：幫助在進行 Binary Search 時不用處理超出陣列邊界的情況，碰到無窮小或無窮大代表資料不在 Page 中。
- data：實際儲存資料的地方，是以 row-based 的方式儲存，例如 |column_a|column_b|column_c|，欄位順序為 Table Schema 宣告順序。
- free space：可放新資料的空間。
- tailer : 用來效驗資料更新是否完整寫入，例如 header 的 LSN 跟 tailer LSN 不一樣代表資料沒完整更新。

**而 Page Directory 是讀寫優化的關鍵結構**

新增資料時，會直接放進 Free Space，不會維持有序陣列，也就是說 `|data1|data2|` 不是有序的，無法 Binary Search，雖然插入變快了，但查詢怎麼辦？

資料在硬碟空間裡雖不是物理有序的，但可用指標 (file offset) 指向下筆紀錄，建立有序的 linked list，不過每次插入都要 scan 整個 linked list O(N) 找到有序的位置，插入效能又變差了，因此需要 page directory！

**Page directory**

page directory 為一個稀疏索引的有序陣列，該陣列不儲存所有資料，陣列 item 又稱為 slot 儲存某段範圍的最小值的指標 (file offset)，例如 1->2->3->4->6 的有序 linked list，可建立一個 slot 為 2 的 page directory [1,3,6]，當插入 (5) 到 free space 後，可透過 page directory [1,3,6] 陣列進行 Binary Search 找到 3 ，並透過指摽往後找，直到遇到第一個比 (5) 大的資料後插入，變成
1->2->3->4->5->6，此時維持有序的 linked list 時間複雜度就變成 O(LogN+K)，K 為 slot 範圍長度。

page directory 為稀疏的有序陣列大小可控，slot 範圍會根據插入情況動態調整，且不一定每次插入都要搬移資料，插入不會因此變太慢。

從 Page 中查資料時，也可用 page directory 縮小查詢範圍，找出 linked list 搜尋起點在往後查詢，時間複雜度為 O(LogN+K)，整體概念有點像只有兩層的 Skip List。

![Screenshot 2025-12-10 at 19.17.11](https://hackmd.io/_uploads/H1ziLCLzbe.png)


### B+Tree 中間點分裂的缺陷

B+Tree 在插入時需要處理節點 (Page) 塞滿的情況，一旦塞滿需要將節點資料分裂出去，傳統做法是從中間點分裂，將左半部跟右半部資料分成兩個子節點，中間值放到父節點：

![Screenshot 2025-12-10 at 19.17.54](https://hackmd.io/_uploads/B1j6UCUMbe.png)

(圖來源：http://mysql.taobao.org/monthly/2021/06/05/)

然而該分裂法在順序插入時，會導致多數節點都處於半滿的情況，可用 https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html 模擬順序插入。

因此 MySQL 採用插入點分裂，需要分裂時檢查新資料是否為節點中的最大或最小值，如果是最大值嘗試將新資料放到右邊子節點，如果右邊子節點不存在就建立一個新節點存放新值，反之亦然，該方法能確保順序插入時，每個節點幾乎全滿。

![Screenshot 2025-12-10 at 19.18.33](https://hackmd.io/_uploads/rJWxP0UzZx.png)

(圖來源：http://mysql.taobao.org/monthly/2021/06/05/)

# MySQL 如何高效地查詢資料

有了 B+Tree 查詢單筆資料只要 O(LogN)，但除了單筆查詢還有
- 範圍查詢: `id between 1 AND 100`
- 多條件查詢: `user_id = ? AND status = ?`

## 查詢範圍資料

B+Tree 是 B-Tree 的衍生版，傳統 B-Tree 在分裂的時候，會把分裂點留在 parent node，其餘資料放到 child node，例如 [1,2,3] page 分裂後會變成， 1<-2->3，parent node 為 2，左 child node 為 1 右 child node 為 3：

![Screenshot 2025-12-11 at 17.43.27](https://hackmd.io/_uploads/HkJVfG_MWl.png)

此時，雖單筆查詢很快，但範圍查詢，例如：`id > 10` 需要從 root 往左 sub tree 掃描，然後再回到 root 從右 sub tree 掃描，在 tree 結構來回穿梭效率差。

而 B+Tree 在分裂時，會把分裂點放在 parent node & child node，例如 [1,2,3] page 會分裂成 1<-2->[2,3]，parent node 為 2，左 child node 為 1 右 child node 為 [2,3]：

![Screenshot 2025-12-11 at 17.45.57](https://hackmd.io/_uploads/ByebazzuMbg.png)


雖然資料多儲存一份，但可以保證 leaf node 包含所有資料，此時將 leaf node 左右關聯建立 double linked list，範圍查詢 `id > 10` 就只要先找到 `id = 10` 的 leaf node 然後往右找出其他資料，線性掃描不用來回穿梭，效率比 B-Tree 快多了。

然而實際資料不會只有 1, 2 or 3，而是一筆多欄位紀錄，如果每次分裂都複製一份完整記錄到 parent & child node B+Tree 容量會變很大，因此 B+Tree 分裂時只複製排序 key 到 parent node，並把完整資料移到 child node 上，最終 B+Tree 結構中 Non-Leaf Node 只儲存排序 key，Leaft Node 儲存完整資料。

![Screenshot 2025-12-11 at 17.46.59](https://hackmd.io/_uploads/Sybb7MuMZg.png)

### 建立多個 index 時 B+ 樹的行為

建立 Table 時，B+Tree 會用 primary key 作為排序 key 以建立 B+ tree，並在 Leaf node 儲存完整資料。但查詢光靠 primary key 不夠，還會用到其他欄位 e.g user_id = ?，此時建立 user_id 的 index，MySQL 會建立額外的 B+Tree 並用 user_id 作為排序 key，但 Leaf Node 不會儲存完整資料，而是儲存 primary key。

因此在 MySQL 中有兩種類型 B+Tree：

1. Clustered Index Tree：用 primary key 為排序 key，leaf node 儲存完整資料。
2. Secondary Index Tree：用其他欄位為排序 key，leaf node 儲存排序 key 跟 primary key。

該設計好處是建立非 primary key 的 index 不會佔用太多空間，且更新非排序欄位時，只要更新 Clustered Index Tree 就好，缺點是用非 primary key 的 index 查詢完整資料時，要先從 Secondary Index Tree 中找到 primary key，再去 Clustered Index Tree 找完整資料，會有兩次 O(LogN)。

## 多條件查詢

當查詢 `user_id = ? AND status = ?` 時，直觀做法是分別從 `user_id` 的 Index Tree 和 `status` 的 Index Tree 取兩批資料後取交集，但這個做法 I/O 次數太多，且 `status` 可能很多筆資料，取交集效率會很差。

**更好地做法是建立 Composite Index 用多個欄位組成排序 Key，例如 (user_id, status) 作為排序 Key，先排序 user_id，相同 user_id 的資料再排序 status。**

使用 Composite Index 後，`user_id = 1 AND status = 2` 查詢過程會變成

- 從 `(user_id, status)` 的 Index Tree 中先依照 `user_id` 進行 binary search 找到第一筆 `user_id = 1` 的 Node
- 此時該 Node 的 `status` 會為最小值且 sub tree 包含 user_id = 1 且 status 是有序的節點
- 再用 status 進行 binary search 從 sub tree 中找到 `status = 2` 的資料，I/O 次數少也不用取交集，效率更好。

### Composite Index 的排序方式

Composite Index 會依照 index 宣告的欄位順序決定誰先排序，(user_id, status) vs (status, user_id) 前者先排序 user_id 後再排序 status，後者相反，此外誰先排序對於查詢性能是有很大影響的。

例如查詢 `user_id = 1 AND status BETWEEN 1 AND 3` 時，`(user_id, status)` 的性能會比 `(status, user_id)` 好很多，假設用 (status, user_id) Index Tree，status 先排序，相同 status 的資料在用 user_id 排序，依照上面查詢要先找出 status BETWEEN 1 AND 3 的資料，但這個 sub tree 中的資料 user_id 已經不是有序的：

| status | user_id |
| ------ | ------- |
| 1      | 1       |
| 1      | 2       |
| 2      | 1       |
| 2      | 2       |
| 3      | 1       |
| 3      | 2       |

因此無法再用 user_id 進行 binary search，反觀 `(user_id, status)` 的 Index Tree 是先找出 `user_id = 1` 的 sub tree 且該 sub tree status 是有序的，因此可用 binary search 在 sub tree 中在找出 `status between 1 AND 3` 的資料。

| user_id | status |
| ------- | ------ |
| 1       | 1      |
| 1       | 1      |
| 1       | 1      |
| 1       | 2      |
| 1       | 2      |
| 1       | 3      |

也就是說，當查詢是 `a = ? AND b = ? AND c >= ?` 時，**Composite Index 的宣告順序要把 c 放在最後面**，因為**一旦經過範圍查詢，後面欄位已不在有序無法用 binary search 過濾資料**，此外如果 composite index 是 (a, b, c) 是無法 cover 到 b AND = ? AND c >= ? 的查詢條件，因為要先用 a 欄位過濾資料， b 欄位才是有序的，而反過來 (a, b, c) 可以 cover a = ? 的條件，因此 index (a) 跟 index (a, b, c) 是有同樣效果的。

## MySQL Optimizer

執行 SQL query 時，MySQL 會將語法解析成 Abstract Syntax Tree 後交由 Optimizer 制定 Query Plan 並決定要用哪個 Index，而 Optimizer 決策因素有：

- Index 的篩選能力
- 總 I/O 次數
- CPU 消耗

### Index 篩選能力
MySQL 會統計每個 Index 的 cardinality (基數)，也就是該欄位中有多少種 Value，例如 orders 表中有 (status) & (user_id) Index，status 只有處理中，失敗，完成 三種狀態，cardinality 就是 3

而 user_id 則可能有 1~10w 種，在相同查詢條件下，cardinality 越高篩選能力越好，所以說
`SELECT * FROM orders WHERE user_id = 1 AND status = '完成'` 的情況下，選用 (user_id) Index 效果更佳。

但只用 cardinality 會有誤判，因為真實資料會傾斜，例如 處理中 只是中間狀態，資料通常很少，因此 MySQL 會加上 histogram 統計出不同 Value 的佔比，假設在 user_id = 1 是一個大戶的情況下執行 SELECT * FROM orders WHERE user_id = 1 AND status = 處理中 ，選用 (status) Index 篩選能力反而更好。

### 總 I/O 次數

越少 I/O 查詢效能越快，而 I/O 次數會受到 Index Tree 高度和是否要回 Clustered Index 影響。
**Index Tree 越高，Traversal 時就要載入越多節點 (Page) 到記憶體**。
假設 orders Table 有 (user_id) & (user_id, status) 兩個 index，並執行下面查詢

`SELECT * FROM orders WHERE user_id = 123 AND status = 'PENDING'`

依照篩選力 (user_id, status) 能篩選掉更多資料，但因為 (user_id, status) 一筆 Index Key 的 size 較大，等於一個 Page 能放的 Index Key 筆數變少，樹就會變高，因此在 I/O 次數上可能 (user_id) 更好。

不過更常見的情況是 orders Table `(user_id)` & `(created_time, user_id)` 並執行下面查詢

`SELECT user_id, created_time FROM orders WHERE user_id = 123 AND created_time > NOW()`

雖然 (user_id) 能篩選掉更多資料，但因為 SELECT 需要 created_time 的欄位，使用 (user_id) Index 還需要回 Clustered Index 查到完整資料，但 (created_time, user_id) Index Tree 中就有 user_id & created_time 的欄位資料，不用回 Clustered Index，在 I/O 次數上 (created_time, user_id) 會更好。

### CPU 消耗

從 Index Tree 過濾完資料後，若資料還不精準，需要額外在記憶體中過濾，也就是說篩選力越好，對 CPU 的消耗就越小，但除了過濾資料 `Order By` 也可能消耗不少的 CPU。

例如 orders Table 有 `(created_time)` Index 並執行

`SELECT * FROM orders WHERE created_time > ? ORDER BY id`

如果不用 `created_time` Index 先過濾資料，就必須把所有資料放進記憶體中一筆筆過濾，CPU 消耗不小，但如果 orders 表不大且 created_time > ? 過濾不多資料時，由於 `ORDER BY id` 用 `created_time` index 過濾後，還要在記憶體中用 id 額外排序，此時 MySQL 會覺得用 primary key Index Tree CPU 消耗可能比較小，因為用 primary key Index Tree 把資料放到記憶體時，資料已是依照 id 排序，一筆筆比對只要 O(N)，額外排序還要 O(nLogN)，因此可能選用 primary key。


**Optimizer 會透過 Index 統計數據依照上面三個因素計算出一個 cost 分數，最後選用 cost 最低的 Index Tree。**

## MySQL 的效能檢測 EXPLAIN

EXPLAIN 功能，可以在不實際執行 Query 的情況下，查詢 Optimizer 制定的 Query Plan 細節，Explain 出來的內容有：

- prossible_keys ＆ key ：Index 內容
    - prossible_keys ：query 中可以用的 inedx
    - key：Optimizer 選中的 index

- ref： 與 index key 比較的資料類別
    - const：使用常數比較 (e.g id = 5)
    - column name：使用其他表的 column 比較 (e.g o.customer_id = c.customer_id)
    - func：比較的 index key 有經過 function 處理 (e.g UPPER(name) = 'JOHN')
    - NULL：沒有等於比較 (e.g id > 5 或全表掃描)

- type ：Index 的查詢方式
**效能排序：system > const > eq_ref > ref > range > index > ALL**
    - system：Table 裡只有一筆 row 或是空的
    - const：只查詢一筆
    - eq_ref：查詢條件來自其他表的唯一值，例如 JOIN
    - ref：= 查詢條件非唯一值
    - range：範圍查詢 (e.g BETWEEN, >, <, IN …)
    - index : FULL Index Scan，掃描整個 Secondary Index Tree
    - ALL : FULL Table Scan，掃描整個 Clustered Index Tree

- extra : 除了 Index Tree 搜尋外的其他操作
    - using index：只用到 Secondary Index Tree，不需回 Clustered Index Tree 拿完整資料
    - using filesort：無法使用 index 排序，需要花 CPU 在記憶體中排序
    - using where : 表示 index tree 無法過濾完資料，還要在記憶體中過濾
    - using temporary：是否有用臨時表，例如 Union 或 Sub Query，會額外消耗記憶體或硬碟空間
    - using MMR : 使用 Multi-Range Read (MMR) 優化，通常用在 Range 查詢。
    - using intersect/union/sort_union：使用 index merge 優化 ，通常用在不同欄位的 OR 條件
    - using index condition : 使用 composited index 的 index condition pushdown (ICP) 優化

### MMR (Multi-Range Read) 優化：
用 secondary index range 查詢且需要回 clustered index 找完整資料時，由於 secondary index 排序跟 primary key 排序不一樣，若依照 secondary index 排序回 clustered index 找資料會造成隨機 I/O，而 MMR 優化是將 secondary index 找到的資料 primary key 暫存到記憶體，排序後批次去 clustered index 查詢，將隨機 I/O 變成順序 I/O。

### Index Merge 優化：
使用 OR 時，例如 `SELECT * FROM t WHERe a = 1 OR b = 2`，會分別查詢 Index Tree 並把結果 Merge 起來並移除重複 primary key 資料，再用 Merge 的結果去 Clustered Index 找完整資料，降低去 Clustered Index 查找數量。

### ICP (index condition pushdown) 優化：
使用 index `(a, b, c)` 執行 `SELECT * FROM t WHERE a = ? AND b > ? AND c = ?` 時

由於 b 是範圍查詢 c 欄位無法用 binary search，此時因為 `SELECT *` 要回 clustered index 找完整資料，就會觸發 ICP，ICP 會在 index (a, b, c) tree 中一筆筆比對 c = ? 的資料，藉此過濾掉更多資料，減少要回 clustered index 查找的數量，降低 I/O 次數。

**rows & filtered** : Optimizer 根據 Index Tree 統計資料估數的查詢數量

- rows：預估會 scan 多少筆數
- filtered：預估 scan 出來的 row 有多少比例完全匹配查詢條件，filtered 率越高越好

**key_len**：預估使用到的 Index Key 長度，可用來判斷 Composite Index 是否所有欄位都用到

## EXPLAIN ANALYZE 觀測 query 實際執行的 CPU 和時間

EXPLAIN 主要是看 Optimizer 透過統計資料制定的 Query Plan，如果想看到實際 Query 的執行時間和 cost 分數，可以執行 **EXPLAIN ANALYZE**。

EXPLAIN ANALYZE 會實際執行 Query 並輸出:

![Screenshot 2025-12-11 at 18.57.22](https://hackmd.io/_uploads/HksumQOGbe.png)

- 執行計劃節點，也就是 Query 的步驟，以上面為例可以看到有四個步驟：
    - users u Table 的 Full Table Scan
    - Filter (u.country = 'US') 資料
    - orders 表使用 Index 查詢 user_id=u.id
    - 將兩批資料使用 Nested loop inner join 關聯起來
- Optimizer 預估資訊 (cost=1000.25 rows=5000)，cost 是成本分數，cost 越高可能會消耗越多 CPU ，rows 是預計 scan 筆數
- 實際執行資訊 (actual time=0.020..10.532 rows=100000 loops=1)
    - actual time 時間單位為 ms，讀到第一筆花 0.02ms，讀完最後一筆花 10ms
    - rows 實際總共 scan 的筆數
    - loops 這個節點被執行了幾次

### Optimizer 的預估與 EXPLAIN ANALYZE 的執行結果具有誤差的情況

由於 Optimizer 使用的統計數據是抽樣調查，難免會有誤差，此時可透過
ANALYZE TABLE your_table; 重新抽樣，若發現光靠 cardinality 會不準，可透過

`ANALYZE TABLE your_table UPDATE HISTOGRAM ON column_name WITH 100 BUCKETS;`

建立 histogram 統計數據。

如果還是不準，最後可試 `ALTER TABLE your_table STATS_SAMPLE_PAGES=100;` 調整抽樣數量，不過要注意 MySQL 執行抽樣統計是會消耗 CPU 的，尤其建立 Histogram 時是需要逐筆計算的，可能造成 Full Table Scan。

## 優化查詢效能 (Query Optimization)

了解 Optimizer 原理跟 EXPLAIN 語法能幫助定位效能問題，以下為幾個 queries 的優化案例

### 善用 Union ALL 來優化查詢效能

**Query 1 - 避免觸發 Index Merge**
- 優化前
```sql
SELECT * FROM t WHERE (user_id = 1 ) OR (receive_id = 1)`
```
- 優化後
```sql
SELECT * FROM t WHERE user_id = 1 UNION ALL SELECT * FROM t WHERE receive_id = 1
```

優化前查詢會用 (user_id) & (receive_id) Index 並觸發 Index Merge 消除重複資料，但其實兩個 OR 查詢並沒有太多交集資料，如果觸發 Index Merge 反而會多花 CPU 執行 Merge，因此可改用 UNION ALL 避免觸發 Index Merge。

**Query 2 - 避免 Range 查詢**
- 優化前
```sql!
SELECT * FROM orders WHERE user_id = 1 AND status IN (1,3,5) AND created_at > ?

SELECT * FROM t WHERE (user_id, receive_id) IN ((1, 2), (3, 4), (5, 6))
```

優化後
```sql!
SELECT * FROM orders WHERE user_id = 1 AND status = 2 AND created_at > ? UNION ALL SELECT * FROM orders WHERE user_id = 1 AND status = 3 AND created_at > ? ...

SELECT * FROM t WHERE user_id = 1 AND receive_id = 2 UNION ALL SELECT * FROM t WHERE user_id = 3 AND receive_id = 4 ...
```

優化前的 IN 條件，理論上要多個 = 比較，但發現變成 range 查詢，原因是 Optimizer 發現 IN 條件有連續性，用 Range 查詢可能比多個 = 重複查詢快，但實務上 range 查詢可能會 scan 到很多不相關的資料，因此可改用 UNION ALL 避免變成 range 查詢。

### 避免過大的 OFFSET

- 優化前
```sql!
SELECT * FROM t WHERE user_id = 1 LIMIT 100 OFFSET 9999999999
```

- 優化後
```sql!
SELECT * FROM t JOIN (SELECT id FROM t WHERE user_id = 1 LIMIT 100 OFFSET 9999999999) sub_query ON o.id = sub_query.id WHERE user_id = 1
```

優化前，因為有 SELECT * MySQL OFFSET 邏輯會變成從 secondary index 找到資料後，要回 clustered index 拿到完整資料，才跳過該筆資料，因此 OFFSET 量大就造成回 clustered index 量大，改成 SELECT id 就不需要回 clustered index 就能跳過資料。

### 強制 ORDER BY `primary_key` LIMIT N 使用特定 index 做查詢

- 優化前
```sql!
SELECT * FROM t WHERE uid = 123 ORDER BY id ASC LIMIT 100
```

- 優化後
```sql!
SELECT * FROM t force index(idx_uid) WHERE uid = 123 ORDER BY id ASC LIMIT 100
```

優化前，即便 uid 有 index，MySQL 仍選擇用 primary key 來查詢，原因在於優化器發現要先排序後取前 100 筆紀錄，如果直接用 primary key 就不用排序，可以直接取前 100 筆，應該會比較快，但如果表很大且 uid 基數很大時，使用 uid index 效能會更好

### 妥善編排 Query 的執行順序

- 優化前
```sql!
SELECT COUNT(1), status FROM t WHERE uid = 1 GROUP BY status LIMIT 1000
```

- 優化後
```sql!
SELECT COUNT(1), status FROM ( SELECT status FROM t WHERE uid = 1 LIMIT 1000) GROUP BY status
```

由於 LIMIT 執行順序在 GROUP BY 後，優化前寫法會 GROUP BY 所有資料，但其實需求只要 GROUP BY 前 1000 筆就好，執行順序[參考](https://medium.com/@cindy20303705/sql%E6%9F%A5%E8%A9%A2%E5%9F%B7%E8%A1%8C%E9%A0%86%E5%BA%8F-4d62584d372d)

### Composite Index 比 Single Column Index 好用

設計 Table Schema Index 時建議先從 Composite Index 著手，例如用 (user_id, status) 取代 (user_id)，雖然 Composite Index 資料較多，但其實 Index Key 只多一個欄位可說是毫無差別，然而多一個欄位不僅能 cover 更多情境的查詢，還可觸發 ICP 優化降低回 Clustered Index 的 I/O 次數

### 避免使用 sql function 在查詢欄位上

```sql!
SELECT * FROM users WHERE id+1 > 100;
```
or
```sql!
SELECT * FROM dtb_user_main WHERE UPPER(name) = 'vic';
```

由於要先修改值才能比對，因此 MySQL 無法使用 binary search 因為修改過後不一定是有序的，最終會變成 full table scan。

# MySQL Buffer Pool

資料除了儲存在硬碟中，MySQL 還會將讀取過資料放在記憶體，用來提升查詢效能，但記憶體空間有限，無法把所有資料都放入，因此淘汰資料的策略就很重要了。

MySQL 不是用 LFU (Least Frequently Used) 策略，因為 LFU 無法適應熱點資料變化，例如客服在某天頻繁查詢特定用戶的資料，但後續不再使用，該資料就會一直放在快取中，且許多業務情境 (e.g 電商 & 交易所 & 搶票) 新建立資料高機率會馬上被查詢，而 LFU 不適用於這種時間局部性，因為當 LFU 滿了，剛建立資料容易馬上被淘汰。

## Buffer Pool 的 LRU (Least Recently Used)

雖然 LRU 能彌補上述 LFU 缺點，但也會產生新問題，例如執行 SELECT * FROM users; 全表掃描，依照 LRU 最新載入資料要放進快取並逐出最後讀取資料，因此會把大量資料放進快取並逐出其他資料，然而全表掃描操作通常為排程任務，有許多非熱門資料，若因此排擠掉熱門資料那就得不償失了。

因此 MySQL 用兩個 LRU 結構，分別為 Old & New Sublist，新讀取資料會放到 Old Sublist 若反覆被讀取才放到 New Sublist，在 New Sublist 的資料為長期活躍資料且不受全表掃描影響，通常 Old Sublist 佔總快取的 38% 而 New Sublist 為 62%。

![Screenshot 2025-12-18 at 20.51.29](https://hackmd.io/_uploads/Byi3Od-mWx.png)

**資料從 Old Sublist 搬移到 New Sublist 的時機為：**

當資料位於 Old Sublist 後 3/4 段落中，且過 n sec 後再次被讀到才會移到 New Sublist，n sec 可透過 innodb_old_blocks_time 設定，預設 1 sec。

等 n 秒設計是因為 MySQL 快取單位為 Page，一個 Page 有多筆連續資料，如果是 Full Table Scan，相同 Page 短時間會被讀取多次，避免 Full Table Scan 搬大量 Page 到 New Sublist，需要等 n 秒設計，而不搬前 1/4 段落是因為這些資料還不會被踢除 ，搬這些資料到 New Sublist 效益比較低。

### 在 Buffer Pool 中找資料

由於快取單位為 Page，因此快取中 hash map 結構的 key 為 `space_id, page_no`：

- space_id : 對應不同 table space
- page_no：對應 table space 不同 page
從 Buffer Pool 中查資料，要先找到 b+tree 的 root page，在 traversal 到 leaf node 才能找到完整資料，因此查資料不只放一個 page 到快取，還有 traversal 路徑的所有 page 都會進去，Index 越多節點佔用空間會越多，且 SELECT * 時，會把 Secondary ＆ Clustered Index 都放進快取。

### 快取單位使用 Page 的原因

不能用 (table_name, column_name, column_value) 為 key 嗎？
如果將快取單位改成紀錄，且 hash map 結構的 key 改為 
`table_name, column_name, column_value`

不僅查詢不用 traversal 整棵樹，也不用把多餘的 page 放進快取中，但缺點就是順序查詢 WHERE create_time > ? 時，要不斷讀取 hash map，且讀取 page 後要將 page 中紀錄一筆筆放到 hash map 會消耗較多 CPU。

另外只放 page 中有命中的紀錄到快取，雖然比較高效跟省空間，但資料查詢除了時間局部性還有空間局部性，例如執行完 id = 1 後，id=2 高機率未來會被用到，因此放整個 page 能提高快取命中率並降低 I/O 次數，此外 MySQL 還提供兩個 read-ahead 功能優化空間局部性。

**Linear read-ahead：**

如果 extent 中 page 依照硬碟連續空間位置順序被讀取，MySQL 會把物理位置的下一個 extent 透過異步 process 載入到記憶體中，可透過 `innodb_read_ahead_threshold` 參數設定多少個 page 被連續讀取才觸發。

注意這裡的物理順序不代表 b+tree 的順序，是硬碟連續空間的意思

**Random read-ahead：**

如果一個 extent 內有許多 page 被隨機讀取，MySQL 會把整個 extent 透過異步 process 載入到記憶體中，可透過 `innodb_random_read_ahead` 開關開啟，官方說如果連續隨機讀取到 13 個 page 才會觸發。

### 快取查詢的時間複雜度
大部分情況需要 traversal B+Tree 所以會是 O(LogN)，但 MySQL 提供了 Adaptive Hash Table (AHT) 記憶體結構，可透過 `innodb_adaptive_hash_index` 參數開啟，其 key 類似 (table_name, column_name, column_value) 而 value 是指向 Buffer Pool 中儲存資料的 leaf page。

AHT 大小是動態調整的，會根據查詢情況決定新增或刪除 key，雖然 AHT 能讓快取查詢變成 O(1)，但缺點是高併發下 AHT 效能會很差，因為讀取 AHT 會觸發動態更新，因此多個讀也要競爭相同鎖，若要提高 AHT 併發效能可調整 `innodb_adaptive_hash_index_parts` 會建立多個 AHT partition。

而從 Buffer Pool 讀取就是單純讀取，所以可用讀寫鎖，併發讀取效能較好，此外也可調整 `innodb_buffer_pool_instances` 參數建立多個 Buffer Pool partition。

## 優化 MYSQL 的快取命中率

雖然快取能加速查詢，但如果命中率太低幫助就不大，由於 `INSERT` 也會將資料放在快取，查詢剛建立的資料一定會命中快取，因此理想的快取命中率要在 98% 左右，如果掉到 95% 以下效能就會有明顯下降

執行
```sql!
show engine innodb status
```

![Screenshot 2025-12-22 at 19.32.14](https://hackmd.io/_uploads/Bk47no8mWl.png)

可看到 InnoDB 運作狀況，其中 BUFFER POOL AND MEMORY 段落有 Buffer Pool 相關數據，該段落裡的 Buffer pool hit rate 1000 / 1000 代表一千次查詢中有幾次有命中快取。

當 hit rate 下降時，reads/s (每秒平均有多少查詢需要 I/O) 通常會上升，此時可先檢查 new sublist LRU 中有無資料：

- Free Buffer : 可用的空間，單位為 page
- Database pages : LRU 全部頁數（不含 free）
- Old database pages : old sublist 頁數
- new sublist ≈ Database pages − Old database pages

如果 Free Buffer 很大且 new sublists page 不多，代表 old sublist 不斷刷新快取，page 來不及進到 new sublist 就被淘汰了，**此時資料庫大概率在執行全表掃描或大範圍的查詢**。

但如果 new sublist page 很多， hit rates 卻下降且 reads/s 升高，可在檢查下面數據：

- young-making rate : 1000 次查詢中有幾次在 old sublist 命中，且把 Page 移到 new sublist
- not young-making rate : 1000 次查詢中有幾次在 old sublist 命中，但沒把 Page 移到 new sublist

如果 `young-making rate` 低 `not young-making rate` 高，此時查詢只走進 `old sublist`，表示 `new sublist` 有資料但命中率不高，代表查詢模式有變，熱資料需要被更新，但是 `old sublist` 卻不斷刷新快取，導致新熱點 page 進不到 new sublist。

要解決上述問題，需修復全表掃描，或者調整 `innodb_old_blocks_pct` 參數，把 old sublist 調大，降低 page 被淘汰的機率，提高 page 被移到 new sublist 機率。

### 當 hit rate 下降， young-making rate 卻依然很高時

代表有大量資料進入 new sublist，但空間不夠資料一直淘汰掉，此時在升級記憶體空間前，可檢查 Buffer Pool 都塞了什麼資料。

`information_schema` DB 中的 `innodb_buffer_page_lru` Table 紀錄了快取資料，執行 
```sql!
select Table_name, Index_name, Data_size from information_schema.innodb_buffer_page_lru where table_name like "{db_name}.%"
```
可看有哪些 Index 被放進快取中，例如：

![Screenshot 2025-12-22 at 19.38.32](https://hackmd.io/_uploads/r13q6s87Wg.png)

上圖可發現，有三個 Index Tree 在快取，由於 insert 資料時需要更新所有 index 因此會載入多 index tree 的 page 到記憶體中。

然而 `idx_uid_price` 跟 `idx_uid` 有同等的效果

```sql!
SELECT * FROM orders WHERE user_id = ? 
```

會命中 `idx_uid_price` 或者 `idx_uid` Index，因此可刪除 idx_uid 減省空間。

此外還可發現 primary Index Tree 的 data size 比其他 index tree 大 ，因此 SELECT * 不僅查詢較慢還會載入較多資料到記憶體，而 index 太多時，也會在 insert 時塞入更多 index tree 到 Buffer Pool 中。

因此可先從優化查詢跟減少 index 數量著手，記憶體還是不夠的話，我們還可看 Buffer Pool 中的 Page Type，執行 

```sql!
select DISTINCT (PAGE_TYPE) from information_schema.innodb_buffer_page_lru
```

看是否有其他系統資料佔用空間。

![Screenshot 2025-12-22 at 19.41.06](https://hackmd.io/_uploads/ryoE0sI7Wx.png)

查看後，會發現除了 index tree 外還有其他系統資料，其中可能會佔用大量空間的是 Undo Log，Undo Log 是實作 Isolation & Atomicity 功能的結構 (下一章節會說明)，若 Transaction 執行太久不 Commit，會造成 Undo Log 資料變大，可在 show engine innodb status 的 `Transaction` 段落中查 History list length，該值代表還沒被清除的 Undo Log 數量，若該值很大可透過下面 SQL 查詢執行最久的 Transaction thread ID，然後透過 SHOW PROCESSLIST 找到執行的 Process 並直接 Kill 掉。

## Buffer Pool 數據的理想狀態

- Free Pages 接近 0 且 Page Types 大多為 Index Page：代表空間有效被利用
- young-making rate & not young-making rate 長時間接近 0:
代表大部分查詢都有命中 new sublist，查詢模式穩定
- hit rate 接近 100% 且 reads/s 不多: 代表快取命中率高，I/O 次數不多，查詢普遍都很快

# MySQL Atomicity & Isolation (Undo Log & MVCC) 的實現方式

Transaction 的 ACID 是寫入資料的重要功能，而 Atomicity & Isolation 則是工程師最常用且最容易影響讀寫效能的兩大特性。

## MySQL Atomicity

直覺思考，可以在資料庫 (記憶體 or 硬碟) 中為每一個 Transaction 建立 private workspace 暫存 Transaction 的異動內容，Commit 時將內容同步到 public workspace 讓其他 Transaction 也看得到，如果 Rollback 就把 private workspace 整個刪除。

然而該方法有兩個致命缺陷：

- private workspace 如果只儲存異動內容，Transaction 重複查詢更新結果，就要不斷從 public workspace 擷取 base 紀錄後加上異動內容，耗效能，如果直接儲存更新後的完整記錄又會導致多個 transaction 開啟多個 private workspace 時佔用太多空間。

- 另外是 Commit 時要同步多筆資料到 public workspace，較耗時，可能造成 Client 送出 Commit 後，在等待中斷線，導致 DB 收到 Commit 執行完回成功，但 Client 沒收到的不一致狀況，Commit 執行越久不一致風險越大。

因此 MySQL 採用相反的設計，Transaction 中所有更新都會同步到 public workspace，也就是 innodb 中的記憶體和硬碟儲存空間，並另外紀錄 Rollback 內容到 Undo Log 空間中，每個紀錄會有一個隱藏的 Rollback Pointer 欄位去指向 Undo Log 內容，Rollback 時將 Undo Log 內容更新到 public workspace，Commit 時去標記 Undo Log 內容為可刪除，隨後由異步 process 刪除 Undo Log。

該方法讓 Commit 變快，Undo Log 只儲存異動內容省空間，且 Transaction 可直接重複查詢更新結果，看似完美了，但還是有兩個問題要解決：

多個 Transaction 同時更新同筆資料，如果 A Transaction 要 Rollback，但 B Transaction Commit 了，A Transaction Rollback 內容會蓋掉 B Transaction 的 Commit，結果就是把 B Transaction 也 Rollback 了，因此多個 Transaction 更新相同紀錄時，必須對該紀錄上鎖，直到 Commit or Rollback 才將鎖釋放。

因為異動都會先同步到 public workspace，其他 Transaction 也會去 public workspace 讀資料，如果被讀到的資料後來被 Rollback 了，相當於其他 Transaction 讀到了髒資料，而這就是 Isolation 要解決的 Dirty Read 問題， Transaction 不能讀到未 Commit 的資料。

## Dirty Read 的解決方案

簡單的方式就是查詢資料時也上鎖，如果別的 Transaction 正在更新資料，就要等 Commit or Rollback 後才能讀到資料，但該方法在併發時效能太差了，有沒有無鎖的機制？

Multiversion Concurrency Control (MVCC) 是 MySQL 實現 Isolation 機制的方法，可在無鎖情況下避免Transaction 讀到未 Commit 資料，其靈感類似 Git 版本控制，當你更新檔案 A 時，其他人可讀取檔案 A 的舊版本，不會看到你沒 commit 的修改。

實作 MVCC 必須儲存不同版本的資料，此時 Undo Log 就派上用場，若當前資料是未 Commit 狀態，透過 Undo Log 裡的 Rollback 內容，就能把資料還原成前一個版本，因為更新時會對資料上鎖，因此同時一筆資料只會有一筆未 Commit 紀錄，因此他的前一版一定是已 Commit 的資料。

但 Isolation 除了要解決 Dirty Read 還要解決 Non-repeatable Read，也就是同一個 Transaction 內讀相同紀錄兩次結果要相同，然而如果該紀錄在兩次查詢之間有被更新並 Commit，就會造成第二次查詢結果與第一次不同。

## Non-repeatable Read 的解決方案

解決 Non-repeatable Read 方法是在 Transaction Begin 時建立資料的 Snapshot，但要把整個資料庫複製一份不現實，可行方式是 Begin 時紀錄資料庫版本號，讀資料發現版本超前時，就透過 Undo Log Rollback 到 Snapshot 版本號，因此 Undo Log 不能只紀錄前一版本的 Rollback，還要維護每筆資料的版本鏈。

Undo Log 中每個 Rollback 節點會有 Pointer 指向前一版本的 Rollback 節點 (如圖)。

![Screenshot 2026-01-05 at 20.08.22](https://hackmd.io/_uploads/r1T5t7tEWg.png)

(圖片來源：https://www.alibabacloud.com/blog/an-in-depth-analysis-of-undo-logs-in-innodb_598966)

此外，每次 Begin Transaction 時 MySQL 會給一個 global 遞增 ID (trx_id)，當 Transaction 更新資料時，會把 trx_id 同步寫到 B+Tree & Undo Log 節點中，因此 trx_id 就是資料庫的版本號。

### 如何決定要 Rollback 到哪個版本？

由於 Transaction 會同時 Begin 多個，無法單純 Rollback Begin 之前的 trx_id，需要透過演算法來決定 Transaction 能讀到哪個版本。

MySQL 在 Transaction Begin 時會建立 Page View 結構：

- m_creator_trx_id：當前 transaction 的 trx_id。
- m_ids：Begin transaction 的當下，所有未 Commit 的 transactions trx_id 集合。
- m_up_limit_id：m_ids 集合中最小的 Trx ID
- m_low_limit_id：下一個 Transaction Begin 時會拿到的 trx_id，也就是未來的版本。並透過下面算法決定哪個版本可讀：id 代表資料當前版本，判斷可否讀規則如下：
    1. 如果 id == m_creator_trx_id 代表是自己修改的紀錄，return true
    2. 如果 id < m_up_limit_id 代表在 Begin 前就 Commit 了，return true
    3. 如果 id ≥ m_low_limit_id 代表在 Begin 後才出現的，return false
    4. 如果 m_up_limit_id ≤ id < m_low_limit_id ，且 id 在 m_ids 裡面，代表在 Begin 當下還沒 commit ，return false
    5. 如果 m_up_limit_id ≤ id < m_low_limit_id，且 id 不在 m_ids 裡面，代表在 Begin 當下已經 commit ，return true
若 return false 就要透過 rollback pointer 往前一版本 rollback，重複檢查直到 return true。

## Transaction 執行太久會怎麼樣？

Transaction 的執行時間對效能是有顯著影響的，若 Transaction 執行很久沒有 Commit/Rollback 會有兩個問題：

1. Begin 之後的 Undo Log 版本持續累積，查詢資料需要 Rollback 很多層
2. 為了要 Rollback 到 Begin 時的版本，Begin 版本後的 Rollback 節點都不能刪除，導致 Undo Log 資料太大，佔用過多記憶體會影響資料快取命中率降低查詢效能。

不過上述問題是在 Isolation Level Repeatable Read (不能 Non-repeatable Read ) 時發生，如果是 Read Committed (不能 Dirty Read) MySQL 是每次查詢都建立 Page View，因此會查詢到後來 Commit 的資料。

# MySQL 的 Lock (Row Lock, Gap Lock & Next-Key Lock)

Isolation Level Read Committed & Repeatable Read Level 主要是解決 Transaction 併發時，Write Transaction 影響 Read Transaction 的問題，然而除了 Write 影響 Read，還有多個 Write Transaction 併發的情境要解決，例如 Write Skew & Phantom Read。

## 什麼是 Write Skew & Phantom Read 問題

Write Skew 是多個 Transaction 同時讀取相同資料，用當下資料狀態判斷邏輯後更新，結果更新內容出現異常，例如扣庫存的 API 這樣實作：

```sql
BEGIN
SELECT id, quantity FROM products WHERE id = ?;

if quantity > 0 
  UPDATE products SET quantity = quantity - 1 WHERE id = ?
COMMIT
```

瞬間執行多次會導致產品的庫存變成負數，要解決該問題可加上 `FOR UPDATE` 語法：

```sql
BEGIN
SELECT id, quantity FROM products WHERE id = ? FOR UPDATE;

if quantity > 0 
  UPDATE products SET quantity = quantity - 1 WHERE id = ?
COMMIT
```

此時 MySQL 會對讀取到的 products 紀錄上 row lock，先取得 lock 的 Transaction 會執行 update 成功，後面的 Transaction 會拿到更新後到 quantity 判斷 <= 0 則不 update。

除了加 `FOR UPDATE` 語法外也可寫成

```sql!
UPDATE products SET quantity = quantity - 1 WHERE id = ? AND quantity > 0
```

`UPDATE` 語法同樣會上 row lock，瞬間多個 `UPDATE` 執行時也不會有問題。

Phantom Read 則是相同 Transaction 內，對範圍資料執行兩次 SQL 出現的結果不一樣，例如實作一個任務排程：

```sql!
BEGIN
SELECT count(1) FROM tasks WHERE status = '待處理'

if count <= 10
   UPDATE tasks SET status = '處理中' WHERE status = '待處理'
COMMIT
```

原本邏輯是最多一次處理 10 筆，但如果有其他 Transaction 在 `SELECT count(1)` 之後 `UPDATE` 之前執行 `INSERT INTO tasks (status) VALUES ('待處理')` 使得待處理任務數量超過 10，那麼 `UPDATE` 出來的結果就會超過 10 筆，不符合預期。

如果在 `SELECT count(1)` 上加上 `FOR UPDATE` 的 `row lock` 能解決嗎？

答案是不能，因為 `row lock` 只會鎖住查詢出的 10 筆資料，`INSERT` 並沒有對那 10 筆資料做任何讀寫，所以 `row lock` 不會卡住 `INSERT` 無法解決 Phantom Read 問題。

## 那麼什麼樣的鎖可以解決 Phantom Read 問題？

MySQL 使用 `Gap Lock` 阻擋 Transaction 對特定範圍 `INSERT` 資料，而會阻擋哪些範圍，是透過 Index 排序跟查詢條件有關，例如 `SELECT id FROM orders WHERE id BETWEEN 1 AND 10 FOR UPDATE` **除了有 row lock 外，還有 `id 1 ~ 10` 範圍的 Gap Lock**，用來阻擋 `INSERT orders (id) VALUES (5)` 這類 SQL。

Gap Lock 準確來是說對資料的間隙上鎖，例如 `id 1 ~ 10`，有些查詢是 `WHERE id >= 10 FOR UPDATE` 就會對 **10 ~ 無限大 範圍上 Gap Lock，另外 Row Lock + Gap Lock 的組合又稱為 Next-Key Lock**，例如 `orders Table 有 id(1, 3, 5)` 三筆資料，執行 `SELECT id FROM orders WHERE id BETWEEN 1 AND 10 FOR UPDATE` 就會對 (1, 3, 5) 上 Row Lock 跟 id 1 ~ 10 上 Gap Lock。

## 為什麼需要意向鎖？（效能考量）

問題場景：T1 想對整個表加 X 鎖（Table lock），但表裡很多 row 已被其他 transaction 持有 row lock。如果沒 intention lock，T1 得逐行檢查「這 row 有沒有被鎖」，這是 O(n) 災難。

意向鎖解決方案：
- T2 在某 row 加 X 鎖前，先在表上加 IX（表級）。
- T1 想加表 X 鎖時，只查表級：有 IS/IX → 直接等，不用查每行。

MySQL / InnoDB 的「鎖模式相容矩陣」（超重要，面試必背）：
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

# 除了 Row Lock, Gap Lock & Next-Key Lock 還有其他種 Lock 嗎？

Lock 除了 FOR UPDATE 語法的 Write Lock 以外還有 `LOCK IN SHARE MODE` 的 Read Lock，Read Lock 互相不會卡住，但 Read & Write Lock 互相會卡住。

此外還有 Table Lock 會直接鎖住整張表，例如執行 `LOCK TABLE orders WRITE` or `LOCK TABLE orders READ` 會對整張表上寫鎖 or 讀鎖。

而在執行 Table Lock 前，要先確認 Table 中是否有 row lock，但總不能 full table scan 一筆筆檢查，因此有了 `Intention Lock`，該鎖只會跟 Table Lock 互卡，執行 `row lock` 會先對 Table 上 `Intention Lock`，`Intention Lock` 不會卡住 `row lock` 跟其他 `Intention Lock`，只卡 Table Lock，因此執行 Table Lock 時如果有其他 row lock，會被 Table 的 `Intention Lock` 卡住，不用 full table scan 檢查。

## 資料什麼時候會被上鎖?

基本上只有 `SELECT` 加上 `FOR UPDATE & LOCK IN SHARE MODE` 和 `DELETE or UPDATE` 等更新語法時會對資料上鎖，但有時上鎖範圍與方式會讓你意想不到。

假如 tasks Table 中有 10 筆待處理任務，執行 S`ELECT count(1) FROM tasks WHERE status = '待處理' FOR UPDATE` 時會只鎖那 10 筆嗎？

答案是不一定，如果 `status` 有 Index 那確實只會鎖 10 筆，但如果沒有 `status index` 就會 `Lock Table` 了，因為 MySQL 架構是 SQL Layer 跟 Storage Engine 分開，沒命中 index 時，要在 Storage Engine Full Table Scan 後交由 SQL Layer 過濾資料，而上鎖必須在 Storage Engine 完成，若等 SQL Layer 過濾完資料，再告訴 Storage Engine 要上鎖哪些資料，會造成部分命中條件的資料沒上鎖到，因為其他 Transaction 可在 SQL Layer 過濾資料時 Insert or Update，因此要在 Storage Engine Full Table Scan 時上鎖，就變成 Table Lock 了。

Gap Lock 主要發生在範圍查詢 `WHERE id > 10 FOR UPDATE` or 查詢非 unique 索引 `WHERE status = 1 FOR UPDATE` ，但如果查詢條件沒有命中資料時 `WHERE id = 10 FOR UPDATE` ， id 是 `unique primary key`，但 `id = 10` 資料不存在，也有 `Gap Lock` 並會鎖住 `id=10` 前一筆跟後一筆資料的間隙，如果前一筆是 5 後一筆是 15 會鎖 5~15 範圍導致 `INSERT INTO t (id) VALUES (11)` 卡住。

此外，`UPDATE & DELETE` 不存在資料，甚至 `INSERT ON DUPLICATE KEY UPDATE` 到不存在資料直接 `INSERT` 時也都會上 Gap Lock。

最後是 Isolation Level 如果設定成 Serializable，雖能解決上述所有一致性問題 (Dirty Read, Non-repeatable Read, Write Skew & Phantom Read)，但會自動把所有 `SELECT` 加上 `LOCK IN SHARE MODE` 的讀鎖。

# MySQL 如何避免 DeadLock

使用 Lock 最大的風險就是 DeadLock，最常見案例是「互相持有並等待」：

```sql!
## transaction A:
BEGIN
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
...  
COMMIT;  
  
## transaction B:  
BEGIN
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
...  
COMMIT;
```

當 A & B Transaction 同時執行，A 持有 `users id=1` 資源並等待獲取 `users id=2` 資源，而 B 持有 `users id=2` 資源並等待 `users id=1` ，雙方互相等待對方釋放資源，造成 DeadLock。

該 DeadLock 解法很簡單，將 Lock 順序改成一致就可以了：

```sql!
## transaction A:  
BEGIN
SELECT * FROM users WHERE id = 1 FOR UPDATE;  
SELECT * FROM users WHERE id = 2 FOR UPDATE;  
...  
COMMIT;  
  
## transaction B:  
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

```sql!
### 建立一個 users table 有唯一主鍵 id 以及非唯一 index name  
create table users (  
	id int auto_increment,  
	name varchar(100),  
	gender int,  
	key (name),  
	primary key (id)  
);  
  
### 建立三筆資料  
insert into users (id, name, gender) values  
(1, 'apple', 0),(2, 'banana',1),(3, 'apple',1);  
  
### 對 id = 1 唯一索引資料上鎖  
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

```sql!
### 對 name = 'apple' 非唯一索引資料上鎖  
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

# MySQL Lock 的執行順序

當執行

```sql!
BEGIN;  
SELECT * FROM users WHERE id IN (1, 2, 3) FOR UPDATE;  
...  
```

上圖，可看到 id (1,2,3) 都有上鎖成功，然後執行

```sql!
BEGIN;  
SELECT * FROM users WHERE id IN (3, 2, 1) FOR UPDATE;  
...
```

理論上，如果上鎖順序是相反，會拿到 `id=3` 的鎖並卡住，但實際上如下圖所示，實際執行是拿到 `id=1` 的鎖卡住，由此可見上鎖順序跟 SQL 寫法無關。

![Screenshot 2026-01-06 at 18.32.05](https://hackmd.io/_uploads/BJKoNwqNZx.png)

## 上鎖順序跟什麼有關?

跟 Index Tree Scan 順序有關，id 的 Index 順序是 1->2->3 ，上鎖順序同樣會是 1->2->3，但如果第二個 SQL 加上 `ORDER BY` 上鎖順序就會相反了：

```sql!
BEGIN
SELECT * FROM users WHERE id (1, 2, 3) ORDER BY id DESC FOR UPDTE;
```

如上圖，會拿到 id=3 的鎖並卡住。

除了 `ORDER BY` 會影響上鎖順序外，建立 Index 時使用逆序設定也會影響：

```sql!
### 將所有 index 改成 unique，並將 nick_name index 順序設定成 DESC  
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

```sql!
### transaction A  
begin;  
UPDATE users SET gender=0 WHERE id = 1;  
if users not found:  
	INSERT INTO users (gender, id) VALUES (0 , 1);  
  
### transaction B  
begin;  
UPDATE users SET gender=1 WHERE id = 2;  
if users not found:  
	INSERT INTO users (gender, id) VALUES (0 , 2);
```

當兩個 Transaction 都 UPDATE 不存在資料時，都會上 Gap Lock 且因為 users 表沒資料，範圍就是 (無限小, 無限大)，此時 A & B Transaction 在執行 INSERT 時就會被對方的 Gap Lock 卡住變成 DeadLock。

另外就是加上面 SQL 精簡成 INSERT INTO users (id, gender) VALUES (1, 0) ON DUPLICATE KEY UPDATE gender=0; ，如果 UPDATE 不存在的值會上 Gap Lock 並 INSERT ，此時若多個 Transaction 執行：

```sql!
### transaction A  
INSERT INTO users (id, gender) VALUES (1, 0) ON DUPLICATE KEY UPDATE gender=0;  
  
### transaction B  
INSERT INTO users (id, gender) VALUES (2, 0) ON DUPLICATE KEY UPDATE gender=1;  
  
### transaction C  
INSERT INTO users (id, gender) VALUES (3, 0) ON DUPLICATE KEY UPDATE gender=0;
```

1. `transaction A` 獲取 Gap Lock 並進入 `insert`
2. `transaction B & C` 同時獲取 Gap Lock ，並等待 t`ransaction A insert`  行為結束，才能 `insert`
3. `transaction A commit` 釋放 Gap Lock ，但由於 `transaction B & C` 都獲取了 Gap Lock，當他們兩繼續執行 `insert` 行為時，會被彼此的 Gap Lock 卡住，導致 DeadLock

**要解決上述 DeadLock 問題很簡單，就是先執行 INSERT ，出現 ON DUPLICATE KEY ERROR 在執行 UPDATE ，若想避免頻繁出現 ON DUPLICATE KEY ERROR，可用 in memory cache 紀錄一個 flag。**

## Foreign Key 造成的 DeadLock 案例：

`customers` 跟 `orders` 是一對多關係，`orders` 表中有 `customer_id` 欄位關聯到 `customers` 表的 `id`。

執行 `INSERT INTO orders (customer_id) VALUES (123);` 時，為確保 customers.id=123 這筆資料存在， MySQL 會對 customers id=123 這筆資料上讀鎖，因此如果執行：

```sql!
### transaction A  
BEGIN:
INSERT INTO orders (id, customer_id) VALUES (1, 123);
UPDATE customers SET count=count+1 WHERE id = 123;

### transaction B
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (2, 123);
UPDATE customers SET count=count+1 WHERE id = 123;
```

Transaction A & B 同時執行，A & B 同時 INSERT orders 成功並對 customers 上讀鎖，隨後 A & B 執行 `UPDATE customers` 時就會被彼此的讀鎖卡住，造成 DeadLock。

## 發生 DeadLock 時，MySQL 會怎麼處理？

MySQL 有 `DeadLock Detection` 的功能，會主動偵測 DeadLock 並 Rollback 其中一個 Transaction，讓另一個 Transaction 順利執行，可透過設定 `innodb_lock_wait_timeout` 參數調整 Transaction 卡住的時間，如果 Transaction 卡住時間超過該參數就會被強制 Rollback。

MySQL 還會紀錄最後一次 DeadLock 的詳細資訊，可以透過執行 `SHOW ENGINE INNODB STATUS` 指令獲取 InnoDB 的詳細資訊，其中 `LATEST DETECTED DEADLOCK Section` 為會紀錄最後兩個造成 DeadLock 的 Transaction 資訊：

```sql!
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

```sql!
## Transaction 1  
begin;  
SELECT id, sold FROM products WHERE id = 919 LOCK IN SHARE MODE;  
...  
UPDATE products SET sold = 32 WHERE id = 919  
commit;  
  
## Transaction 2  
begin;  
SELECT id, sold FROM products WHERE id = 919 LOCK IN SHARE MODE;  
...  
UPDATE products SET sold = 34 WHERE id = 919  
commit;
```

邏輯看起來是同時賣出相同產品，該情況有兩種解法：

1. 將 `LOCK IN SHARE MODE` 改成 `FOR UPDATE` 確保先拿讀鎖，誰先拿到先執行 `UPDATE`
2. 移除 `SELECT` 語法，使用 `UPDATE products SET sold = sold + ? WHERE id = 919 AND sold >= ?` 效能較好

# MySQL 的 write durability (WAL & Redo Log)

Transaction ACID 中 Durability 要求資料不可遺失，因此一旦 Transaction Commit 後，資料就要寫進硬碟，不能只放在記憶體中。

但 MySQL `INSERT` or `UPDATE` 都要找到 B+Tree 中的 Page 更新，不同資料位於不同 Page，頻繁更新時等於是隨機 I/O，對硬碟效能影響很大，例如：

- 傳統硬碟的磁頭要前後移動，移動距離大。
- SSD 更新資料不能直接覆蓋，要先擦除既有資料才能更新。

如果是順序 I/O 寫入資料，傳統硬碟磁頭只要往前，而 SSD 只要不斷往空白區塊寫入不用擦除，效能都比隨機 I/O 好。

但由於資料往往會散落在不同區塊，更新時也會需要覆蓋既有的區塊，因此，善用順序 I/O 以提升效能就變成可考量點。

## 利用順序 I/O 提升寫入效能

MySQL 使用 Write Ahead Log (WAL) 結構優化寫入效能，WAL 是 append-only 的檔案，只能往檔案後面空白空間寫資料，不能覆蓋前面的內容，而 Redo Log 是 MySQL 實現 Durability 的 WAL。

當資料更新時，MySQL 會先把 Page 載入 Buffer Pool 並在記憶體中修改 Page 內容，隨後把修改的內容寫入 Redo Log 中，此時寫入就完成了，隨後會有異步 process (aka flush process) 定期把被修改的 Page (aka dirty page) 寫入硬碟 (Table Space) 中的 B+Tree 結構。

Redo Log 不會放整個 Page 資料，而是儲存 Page 的修改內容 (e.g space_id, page_no, file offset, change payload)，當 DB crash 後重啟，可重放 Redo Log 內容，將修改資料同步到 B+Tree 結構。

![Screenshot 2026-01-06 at 18.57.20](https://hackmd.io/_uploads/SygO9DqNWl.png)

Redo Log 只儲存修改內容，雖然可節省空間，但在 recovery 時帶來新的挑戰。

由於 flush process 寫入單位為整個 Page，如果寫入一半時 OS crash 或者硬碟出問題，會導致 Page 處於半更新狀態 (aka torn page)，header LSN (更新版本) 會與 tailer LSN 不一致，此時直接重放 Redo Log 可能無法正確恢復 torn page，因為 page 內容與 Redo Log 紀錄的狀態不一致（例如，file offset 100 的數據預期是 id=2，但實際可能是 id=3 或損壞）。

**因此 MySQL 用了另一個順序寫入結構 Doublewrite Buffer 解決這個問題，flush process 會先把 dirty page 寫入 Doublewrite Buffer，整個 page 成功寫入 Doublewrite Buffer 後才會更新到 B+Tree 結構中的 Page。**

當 os crash db 重啟時，如果檢查到 B+Tree 中 page 為 torn page，可透過 Doublewrite Buffer 內容直接覆蓋上去，如果 Doublewrite Buffer 裡的 page 為 torn page 就直接棄用，這樣 B+Tree 就不會有 torn page ，重放 redo log 不會有問題。

當 os crash db 重啟時，如果檢查到 B+Tree 中 page 為 torn page，可透過 Doublewrite Buffer 內容直接覆蓋上去，如果 Doublewrite Buffer 裡的 page 為 torn page 就直接棄用，這樣 B+Tree 就不會有 torn page ，重放 redo log 不會有問題。

## 如果發現大量寫入影響效能怎麼辦？
此時可調整 `innodb_flush_log_at_trx_commit` 參數：

- `innodb_flush_log_at_trx_commit=1` 1 是預設參數，寫入後就馬上把 Buffer Pool 中修改資料寫進 Redo Log File，並呼叫 fsync 確保 OS 把異動從 OS Cache 寫入 Disk。
- `innodb_flush_log_at_trx_commit=0` 0 是效能最好的參數，寫入時異動資料只保留在 Buffer Pool 並透過 Background Log Writer Thread 每秒更新到 Redo Log File，且更新時不呼叫 fsync 而是 write syscall 將資料寫進 OS File Cache 讓 OS 自行決定何時將 OS Cache 寫入 Disk。
- `innodb_flush_log_at_trx_commit=2` 2 是介於中間的參數，寫入時異動資料保留在 Buffer Pool， Background Log Writer Thread 每秒更新到 Redo Log File，且更新時執行 fsync 確保 OS 把異動寫入 Disk。
雖提升效能，但犧牲 Durability。

預設 `innodb_flush_log_at_trx_commit=1` 最保險，但瞬間大量寫入造成大量 I/O 可能導致 Redo Log 寫入延遲，crash 時資料還是會丟失。

因此 MySQL 實作 Group Commit 優化，將瞬間多個寫入合併在一起，只執行一次 I/O ＆ fsync syscall 將多筆異動寫入 Redo Log File，實作方法是透過共享鎖 & LSN (更新版本)：

1. 瞬間同時有三個寫入 (TxA, TxB, TxC) ，此時三個異動都會寫進 Buffer Pool
2. TxA , TxB & TxC 會競爭同一把鎖，假如 TxA 搶到了，TxA 就稱為 Leader Transaction 會負責把 Buffer Pool 所有資料寫入 Redo Log 並執行 fsync，並更新最後寫入的 LSN
3. TxB & TxC 拿到鎖後會檢查自身 LSN 是否小於最後寫入 LSN，小於就跳過不執行 I/O

## 建立太多 Index 是不是也會影響寫入效能？

假設 orders Table 有 (uid) 的 Index，在執行 `UPDATE orders SET uid=1, status=1 WHERE id = 2` 時，MySQL 會透過 Clustered Index 找到 Leaf Page 放進 Buffer Pool 更新，同時還要更新 (uid) 的 Secondary Index ，因此也會把 Secondary Index Leaf Page 放進 Buffer Pool 更新，且除了 Leaf Page 查詢經過的 Page 也會被放進快取。

因此建立太多 Index 不僅會提高寫進 Redo Log 的資料量，還會載入更多資料到記憶體擠壓到空間。

**MySQL 為避免單純寫入造成大量 Secondary Index Page 載入到記憶體，使用 Change Buffer 結構來做到延遲更新**，同樣是 `UPDATE orders SET uid=1, status=1 WHERE id = 2`，有了 Change Buffer 後，MySQL 只要載入 Clustered Index 的 Leaf Page 和查詢路徑 Page 到 Buffer Pool 更新，同時 Secondary Index 的更新只需要載入查詢路徑 Page 同時定位到 page_no 後，將更改內容和 page_no 寫入 Change Buffer 中，後續若有查詢需要這個 page_no (Leaf Page)，才會真的去讀取，並與 Change Buffer 中異動內容 merge，同時 flush process 也會定期把 Change Buffer 內容更新回 B+Tree Page 中。

**有了 Change Buffer ，寫入時能減少載入 Secondary Index Leaf Page 的 I/O 跟記憶體資料，當然缺點就是讀取 Secondary Index 時，多了一個從 Change Buffer 中 Merge 的過程，且不適用於 Unique Index 因為必須載入 Leaf Page 檢查是否有衝突。**

總結，建立太多 Index 不只影響寫入效能還會影響查詢效能，雖然有 Change Buffer 優化，但只用於 non-unique index，效果可能不明顯。

# MySQL 的 Flush Process (快取資料寫入硬碟)

MySQL 寫入流程是：

1. 將資料從硬碟載入 Buffer Pool (記憶體)
2. 在記憶體中更新資料，並寫入異動內容 Log Buffer (Redo Log 記憶體緩衝區) 在寫到 Redo Log File
3. 同時把異動後的 Page (Dirty Page) 記憶體指標放入 Flush List (FIFO Queue)
4. Flush Process 定期從 Flush List 拿出第一個 Dirty Page 並寫入 5. Doublewrite Buffer
5. Doublewrite Buffer 寫入成功後，在寫入 B+Tree 結構的 Page
6. Flush Process 寫入 Dirty Page 標記 Checkpoint LSN，讓 Redo Log 可清除資料

## 那麼什麼是 Log sequence number (LSN)？用途為何？

recover 資料時需要知道從哪一個起點開始重放，因此有 Log Sequence Number (LSN) 用來記錄 Redo Log 寫入狀況，Redo Log 每次新增都會增加 LSN 的數字，是 global 遞增數字，例如某個 `INSERT` 的 LSN 為 100，代表到這次 INSERT 為止，總共寫了 100 bytes 資料到 Redo Log 中。

每個 Page 中會記錄 Oldest Modification LSN 以及 Newest Modification LSN：

Oldest Modification LSN — 該 Page 第一次變 Dirty Page 時的 LSN，Flush List 的順序會從最小的 Oldest Modification LSN 開始刷新到 Disk。

Newest Modification LSN — 該 Page 最後一次被更新的 LSN，可以用來追蹤該 Page 是否短時間內被更新多次，是否需要先被刷新到 Disk。

當 Flush Process 將最小的 Oldest LSN 的 Dirty Page 寫入硬碟時，Checkpoint LSN 就會更新，例如：

- Dirty Page 1: oldest_modification = 900
- Dirty Page 2: oldest_modification = 901，寫入 Dirty Page 1 後，Checkpoint LSN 可往前推進到 901。

而 Checkpoint LSN 可用來當作 recover 資料起點，以及 Redo Log 會清除 LSN 在 Checkpoint 之前的資料。

除了 Checkpoint LSN，InnoDB 也會記錄不同重要時機的 LSN，這些 LSN 可用於確認同步進度並觀測目前寫入的效能，可透過 `show engine innodb status ` 查看：

![Screenshot 2026-01-06 at 20.12.48](https://hackmd.io/_uploads/HJmS2_5V-e.png)

- Log sequence number — 當前最新的 LSN
- Log buffer assigned up to — log buffer 中最新的 LSN 包含未 commit 的 transaction
- Log buffer completed up to — log buffer 中 最新 commit 的 transaction 的LSN
- Log written up to — log buffer 寫入 redo log disk 的 LSN
- Log flushed up to — redo log file 中已經執行 fsync 將資料從 OS cache 更新到硬碟的 LSN
- Added dirty pages up to — flush list 中最晚被更新的 LSN，對應 dirty page 的 newest modification LSN
- Pages flushed up to — flush list 最早被更新的 LSN，對應 dirty page 的 oldest modification LSN
- Last checkpoint at — 最後一筆更新進 B+Tree Disk 的 LSN，Redo Log 會從該 LSN 後進行資料恢復，且 Checkpoint 以前的 Redo Log 內容是可以刪除的，避免 Redo Log 內容太多

當 **Pages flushed up to - Added dirty pages up to** 數值很大時，代表有很多 Dirty Page 等待被 Flush，那麼 Flush Process 觸發的時機為何？如何避免太多 Dirty Page 阻塞？

## Flush Process 觸發的時機

主要兩個觸發點：

- Sharp Checkpoint：將所有 Dirty Page 寫回 Disk，當數據庫正常關閉時觸發。
- Fuzzy Checkpoint：由 Background Cleaner Thread 每秒執行，依照當前 Buffer Pool 狀況動態決定寫回 Disk 的 Dirty Page 數量。

Sharp Checkpoint 可透過 `innodb_fast_shutdown` 參數開啟，但 Fuzzy Checkpoint 的參數就比較複雜了，其有兩種 Flush 模式：

- BUF_FLUSH_LRU — 從 LRU 尾端找出可以 Flush 的 Dirty Pages
為了避免 LRU 驅逐不常用的 Page 時，需要花費額外的時間先 Flush Dirty Page 到硬碟，Cleaner Thread 每秒會依照 innodb_lru_scan_depth 參數從尾端找 N 個 Pages 檢查是否有需要 Flush 的 Dirty Page，BUF_FLUSH_LRU 並不會更新 checkpoint LSN。

- BUF_FLUSH_LIST — 從 Flush List 中最小的 oldest modification lsn 開始 flush，寫入硬碟後會更新 checkpoint LSN

Cleaner Thread 觸發 BUF_FLUSH_LIST 時，不會把所有 Flush List 的 Dirty Page 更新到硬碟，而是透過演算法動態計算出合適的 Dirty Pages 數量，該 Flush 又稱為 Adaptive Flush，其算法為：

`n_pages` 代表幾個 dirty pages 要刷新，他是由三個數字加總取平均值：

1. `(innodb_io_capacity * (ut_max(pct_for_dirty, pct_for_lsn)) / 100` 代表要使用多少全力刷新 dirty pages
innodb_io_capacity 參數代表每秒可有多少 I/O 用於 Flush Dirty Pages，該參數會乘上一個比例決定最終要用多少 I/O，比例由下面兩個數字取大的計算。
`pct_for_dirty` 代表目前 LRU 髒頁的比例 `(modified_pages / total_pages_in_buffer_pool) × 100`，如果小於 `innodb_max_dirty_pages_pct_lwn` 會等於 0，超過 `innodb_max_dirty_pages_pct` 會設定成 100，使用全力 Flush。

2. avg_page_rate 代表平均每秒產生多少 Dirty Pages，計算方式：
`avg_page_rate = ((sum_dirty_pages / time_elasped) + avg_page_rate) / 2`

3. pages_for_lsn Redo LSN 增長了多少，由於 LSN 是 bytes 單位，會將 bytes 單位換算成 Page 數量，計算方式：
total_lsn = current_lsn - checkpoint_lsn;  
avg_lsn_per_page = 2048.0;  
pages_for_lsn = total_lsn / avg_lsn_per_page;

最後三個變數取平均後並確保 page 數量不能超過 `innodb_io_capacity_max` 設定，計算出 n_pages，代表這次要從 flush list 拿出多少個 dirty page 寫進硬碟。

如果發生 Checkpoint LSN 過於落後 Latest LSN，可以透過調大 `innodb_io_capacity` 參數或者調低 `innodb_max_dirty_pages_pct` 參數來增加每次 Flush Dirty Pages 的數量。

如果發現是 Buffer Pool 中 Dirty Pages 比例很高，可以調大 `innodb_lru_scan_depth` 參數，增加 LRU 掃描 dirty page 的範圍，避免淘汰 Page 還要等 Dirty Page 寫入會影響查詢效能。

另外可透過 `show engine innodb status` 的 BUFFER POOL AND MEMORY 查看 Modified db pages數量，該數量代表 Buffer Pool 中有多少 Dirty Pages。

# MySQL 的 Schema 管理

Transaction ACID 中的 Consistency 要求資料更新前後要符合資料規則，確保資料正確，而資料規則主要是透過 Schema 來制定的，例如 Data Type & Constraint 等，實務上 Schema 很難建立了就不修改，修改 Schema 的 SQL 雖然很簡單，但其中卻暗藏陷阱，一不小心就會 Lock Table 導致查詢寫入全部卡住。

## MySQL Schema 儲存在哪裡？

8.0 前儲存在 `.frm` 檔案，**8.0 後則儲存到 system table space 的 `.idb` 並將 Schema 資料視為紀錄，能使用 Transaction 去更新**，但不論 `.firm` 還是 system table space ，Schema 資料與資料是分開儲存，好處就是省空間，不用每資料都另外儲存 Schema 資料。

## 既然是分開儲存，那麼更新 Schema 時，就不會去更新到資料對嗎？

答案是不一定，例如 `ALTER TABLE users ADD COLUMN count SMALLINT NOT NULL AFTER name` ，新增欄位時加了 AFTER 就需要去更新資料，而更新勢必要上鎖，就會導致 Table Lock，如果 Table 資料多，鎖的時間就會很久。

原因在於 Page 儲存資料的格式是 Schema 定義的，例如

```sql!
create table users (  
	id int auto_increment,  
	name char(100),  
	gender int,  
	primary key (id)  
); 
```

資料格式為，`[4bytes | 100 bytes | 4 bytes]`，前 4 bytes 代表 id，中間 100 bytes 是 name ，最後 4 bytes 是 gender，如果執行 `ALTER TABLE users ADD COLUMN count SMALLINT NOT NULL AFTER name` 格式會變成 `[4bytes | 100 bytes | 2 bytes | 4 bytes]`，若 `idb` 檔案中資料沒更新，讀資料時依照新的 Schema 解析就會出錯。

## 那麼只要 ADD COLUMN 都會變動格式，都會 Lock Table 一段時間嗎？

不一定！MySQL 修改 Schema 的演算法有三個，Copy & Instant & InPlace 。

- **Copy** - 建立新的 idb 檔，把舊的資料依照新的格式搬移過去，然後再刪除舊的 idb 檔，且搬移過程中要對資料上鎖，避免舊 idb 檔案被更新沒同步到新 idb 檔案。

- **Instant** - 只修改 Schema 資料，Table Lock 時間很短。
如果 ADD COLUMN 在 Schema 尾端且可為 NULL 或有 Default Value，MySQL 就會用 Instant 算法，執行會非常快，但資料格式有變，解析不會有問題嗎？
執行 ALTER TABLE users ADD COLUMN count SMALLINT NOT NULL DEFAULT 1 ，資料格式會變成 [4bytes | 100 bytes | 4 bytes | 2 bytes ]，解析舊格式時發現資料長度不夠，可以自動補預設值，因此不更新資料也不會解析錯誤。
因此像是 ALTER COLUMN NAME or ADD COMMENT 等不會破壞資料格式的 Migration 都會用 Instant，而修改 Data Type 跟 ADD COLUMN AFTER 等會破壞資料格式的 Migration 會用 Copy。除了 Column 調整，ADD INDEX 也是常見的 Migration，建立新的 Index 雖不會影響資料格式，但需要為每筆資料建立新的 Index Tree，因此會此用 InPace 。

- **InPlace** - 不會建立新的 `.idb` 檔案，而是在原有的 `.idb` 檔案中擴展新空間塞新資料，塞完新資料後會拿 Table Lock 並更新 Schema 內容，上鎖時間不多，InPlace 透過兩個機制完成：
    - **Background Thread** : 在背景執行將原有資料同步到新空間，例如將原有資料一筆筆建立新的 Index 。
    - **Online Alter Log**：由於 InPlace 執行不會 Lock，因此資料會不斷被更新，會導致 Background Thread 同步完某筆資料後，該筆資料又被更新，這時需要一個機制追蹤這些後來被更新的內容，所以 MySQL 用額外的 Log 結構去儲存 Background Thread 執行過程中的所有修改內容，並在 Background Thread 執行完後重放 Log 的內容建立新的資料。

其實理論上 `ADD COLUMN` 若影響資料格式也可參考 InPlace 算法實作，可惜 MySQL 會用 COPY 算法，但如果你會調整資料格式又不想 Lock Table 那麼久，可以用 **online-schema-change** 工具，其概念是 InPlace + Copy 整合，他的流程是：

1. 建立新 Schema 的 Table (e.g new_users)
2. 在舊的 Table (e.g users) 建立一個 `INSERT` & `UPDATE` & `DELETE` 三種 trigger，透過 trigger 同步舊 Table 的異動到新 table
3. 使用 Primary Key 批次搬移舊資料到新資料
4. 資料搬完後，會獲取新舊兩個 Table 的 Table Lock 然後 rename 新 Table 名稱跟舊 Table 名稱 (e.g users -> old_users; new_users -> users)
5. 移除舊 Table (e.g old_users)

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

# MySQL 如何應付大量查詢流量？(Binlog, Slave DB)

隨著系統業務量增加，即便將查詢優化到極致，系統仍會負荷不了瞬間大量查詢，此時只剩垂直與水平擴充兩個選項，垂直擴充相對簡單，提升硬體 CPU 和記憶體，但缺點是升級時需要停機，因此水平擴充較為常見，MySQL 常見的 Cluster 架構就是 Master-Slave 架構，一台 Master 處理寫入請求並同步給多台 Slave，而多台 Slave 負責查詢請求。

## 為何只需要一台 Master？寫入請求不需要分流嗎？

多 Master 的 Cluster 架構較複雜且容易出錯，例如：

- 多台 Master 同時更新相同資料時，誰的結果是最新的
- 採用 Data Sharding 會造成跨 Server Transaction 實作 ACID 困難
且寫入比讀取花更少 CPU 和記憶體，通常查詢頻率又比寫入高，因此需要分流通常是查詢請求，因此單 Master 多 Slave 架構能降低複雜性，並分散查詢請求提升負載上限。

那麼 Master 如何向 Slave 同步資料？

當 Master 收到更新請求 (e.g `INSERT`, `UPDATE`, `DELETE` ) 時，需要即時將修改內容同步給 Slave，同步方式有兩種，Push & Pull：

**Master Push 方式** -  Master 主動推送更新資料給 Slave

- 優點
資料更新後能立即推送，資料同步延遲低

- 缺點
Master 需要紀錄不同 Slave 接收進度並透過不同 Thread 推送不同進度資料，Master 端實作複雜

**Slave Pull 方式** - Slave 向 Master 拉取更新資料

- 優點
    - 由 Slave 管理接受進度，Master 邏輯單純，只負責將變更資料寫進 Queue 中， Slave 依照各自進度讀取
    - 多台 Slave 不會佔用 Master 太多額外資源，因為 Master 不用管理 Slave 進度 
- 缺點
    - 透過 Polling 方式讀取資料，Polling 間隙時間會導致資料同步延遲 

在單 Master 架構中，Master 是唯一寫入點，其穩定性與效能很重要。為了降低 Master 的複雜度與負載風險，MySQL 採用 Slave 主動拉取資料 的設計，雖然資料同步會有些許延遲，但優點是：

- Master 專注於寫入，不需追蹤每個 Slave 的進度或管理推送邏輯
- Slave 可根據自身能力調整拉取頻率與批次大小，在資料量大時一次拉取更多資料，反而提高處理效率

而在拉取模式下，Master 要將**變更資料先寫入一個暫存區**，讓不同的 Slave 依照其進度拉取資料，而這個暫存區就是 Binlog！

## 為何需要新的儲存區？不能用 Redo Log 嗎？

既然所有寫入都會先寫到 Redo Log 中，Slave 不能直接去 Redo Log 拉資料同步嗎？然而，直接用 Redo Log 同步資料有兩個缺點：

- Redo Log 屬於 InnoDB 是 Storage Engine 結構，如果換一個 Engine 需要修改 Slave 同步邏輯，缺乏系統彈性。
- 資料同步至 B+Tree 後就會從 Redo Log 清除，不會等到所有 Slave 成功同步在清除，不符合同步所需的持久性保障。
因此 MySQL 需要在 SQL Layer 層使用 Binlog 結構用來當作 Slave 同步的暫存區。

![Screenshot 2026-01-06 at 20.49.04](https://hackmd.io/_uploads/S1BoNK5E-x.png)

而 Binlog 的儲存格式有分：

STATEMENT：直接儲存 SQL 指令內容 (e.g INSERT )
- 優點：佔用硬碟空間小
- 缺點：非確定性函數 (e.g NOW(), RAND() )會導致 Master Slave 資料不一致

ROW：直接儲存修改後的完整資料內容
- 優點：Master Slave 數據精準一致，不用解析 SQL 指令同步效率高
- 缺點：佔用大量空間

MIXED：結合 Statement & Row 依照 MySQL 自行判斷何時要用哪個
- 優點：平衡資料大小以及空間消耗
- 缺點：不可預測性，無法確保 MySQL 行為跟你預期的一樣

## 寫入要分別同步到 Redo Log ＆ Binlog 兩個結構，會不會有同步 Redo Log 成功到 Binlog 失敗的可能？

如果 Redo Log 成功 Binlog 失敗會造成 Master 和 Slave 資料不一致，因此 MySQL 透過 2-Phased Commit 解決 Redo Log & Binlog 同步議題：

![Screenshot 2026-01-06 at 20.50.41](https://hackmd.io/_uploads/ryXbSt54Zx.png)

其核心精神是；

1. 先執行完所有 Application 層的邏輯，確保沒有任何錯誤
2. 最後在一起執行 fsync system call 這個最不可能出錯的指令

而 2-Phased Commit 也有 Group Commit 技術，但實作方式 SQL Layer 層聚合多筆 Transaction Commit 後的內容，透過 2-Phased Commit 流程，用一個 Prepare 送出多筆變更，最後執行一個 fsync 寫入多筆內容。

可透過下面參數控制 SQL Layer 聚合方式：

- binlog_group_commit_sync_delay ：等待多久 (單位為微秒) 後執行 Prepare & Fsync
- binlog_group_commit_sync_no_delay_count ：累積多少個 Transaction Commit 後執行 Prepare & Fsync

## Master 寫資料進 Binlog 後，Slave 如何處理這些資料？

首先 Slave 要知道從 **Binlog 哪個起點開始同步**，且避免故障重啟要重頭同步，Slave 要記住上次同步的進度。

傳統是 Binlog File Location & Position 方式 (執行 SHOW MASTER STATUS;)：

![Screenshot 2026-01-06 at 20.51.52](https://hackmd.io/_uploads/HyFrSF5VWl.png)

File： Binlog 檔案名稱，後面數字為遞增數字，越大代表資料越新
Position：代表 Binlog 寫入進度，是 bytes 單位，也是下一個寫入的起始位置

該方式好理解，但缺點是不同 DB 同步相同 Binlog 內容，但顯示出的 File Location & Position 卻不一定相同：

原因是不同 Server 有不同配置，例如不同 Binlog 檔名，是否要壓縮，Format 等等，就算配置都一樣， Binlog 紀錄的 Header 會依照不同 Server 有所差異，就會導致大小不依而產生不同的 Position。

![Screenshot 2026-01-06 at 20.52.23](https://hackmd.io/_uploads/BJoOrF9VZg.png)

![Screenshot 2026-01-06 at 20.52.28](https://hackmd.io/_uploads/ByzKHYcNZe.png)

而不同 File Location & Position 會帶來 Fail Over 的問題，假設 Master 掛了短時間無法恢復，必須把 Slave 轉成 Master，但轉換後卻發現新 Master 的 File Location & Position 的內容跟舊 Master 不一致，導致其他 Slave 沒法用目前進度來繼續同步新資料，因此需要人工介入，重新設定所有 Slave 的 Binlog 起始點，無法做到自動化 Fail Over。

為了解決該問題，MySQL 使用 Gtid 的出現，跨 DB 的唯一 Transaction ID，其格式為 _source_id_:_transaction_id ：

- source_id：該 Transaction 來源的 MySQL Server ID，跨 Server 唯一
- transaction_id：來源 MySQL Server 所生產的遞增 ID

![Screenshot 2026-01-06 at 20.53.36](https://hackmd.io/_uploads/B18nHFcV-g.png)

如圖，相同的 Binlog 內容，雖然 File Location & Position 不同，但 Executed_Gtid_set 相同，Gtid Set 是用來表達多個 Gtid 的格式 ( `source_id:begin_transaction_id-last_transaction_id` )，例如：

- Retrieved Gtid Set：從 Master 拉下來的 GTID 集合
- Executed Gtid Set：實際寫入到 Redo Log 的 GTID 集合
- Purged Gtid Set : 已同步過的 GTID 集合。

## Slave 是怎麼同步資料進硬碟的？

Slave 從 Master Binlog 拉取資料後不會馬上寫入 Redo Log，而是先同步到 Relay Log 中，原因在於：

- Relay Log 作為中繼檔案可避免每次讀取後需馬上解析並更新所帶來的延遲與風險，單純同步 Binlog 資料到 Relay Log，寫入成本低、邏輯單純，能提高拉取效能並保證系統異常後可回溯。
- 有 Relay Log 後可將純寫入 & SQL 執行邏輯解耦，實現雙執行緒模型，Replica I/O Thread 負責拉取資料並寫入到 Relay Log，Replica SQL Thread 負責讀取 Relay Log 並寫入到 InnoDB，好處是錯誤隔離且能針對彼此情境做單獨優化。

說到優化，Replica I/O & SQL Thread 作為同步兩大核心，其效能會影響 Slave 同步速度，首先：

Replica I/O Thread 負責連上 Master 後不斷拉取資料，為確保 Relay Log 順序跟 Binlog 順序一致因此無法並行處理，不過 I/O Thread 只負責同步資料，其效能瓶頸在於網路吞吐量，需要能內拉取大量資料並一次執行 `fsync` ，因此可透過 `binlog_transaction_compression` 參數壓縮 binlog 內容提高吞吐。

Replica SQL Thread 需解析 Binlog 並透過 Storage Engine 寫入資料，其邏輯較複雜且耗時，需透過並行處理提高同步速度，因此 MySQL 使用了 **Multi-Threaded Replica** (a.k.a MTS) 技術。

要並行執行 SQL 首先要確保 SQL 之間沒有依賴關係，例如：

```sql!
transaction A  
UPDATE users SET status = 2 WHERE user_id = 1;  

transaction B  
UPDATE users SET status = 1 WHERE status = 2;
```

上面案例執行順序不同，產生結果就會不相同，因此有依賴關係。
最簡單的判斷邏輯就是不同 DB 的 Transaction 彼此絕對沒有依賴關係：

```sql!
transaction A   
UPDATE db_a.users SET status = 2 WHERE user_id = 1;  

transaction B  
UPDATE db_b.users SET status = 1 WHERE status = 2;
```

這也是 MySQL 5.6 引進的 **Per-Database Replication**，但實用性太低，大部分情況都是單一 DB 就需要有好的同步效能，因此 MySQL 5.7 開發了 **Logical_clock Replication**：

**其概念是 Master 並行執行了哪些 Transaction，Slave 就也可以並行處理，而 Master 實際並行執行的 Transaction 就是 Group Commit 中的Transaction**，因此 MySQL 在 Binlog 內容中加入了 Group Commit 的邊際值，而 **Logical_clock Replication** 會透過分析 Group Commit 邊際值來判斷哪些 Transaction 是可以並行執行的：

![Screenshot 2026-01-06 at 20.58.36](https://hackmd.io/_uploads/BJ008K54Zx.png)

上面 Binlog 內容的 last_committed 代表 transaction 執行時前一個完成的 transaction 序號，**因此相同時 last_committed 值的 transaction 代表是同時執行的，Logical_clock Replication 會解析該內容找出可並行處理的 transaction。**

# MySQL 如何架設高可用的 Master-Slave 架構？(ProxySQL & Orchestrator)

架設高可用的 MySQL Cluster 不只要讓多台 Slave 去接收 Master 的 Binlog 同步資料，還要做到當 Master Crush 時，Slave 能自動接替成為 Master 同時 Client 送出的寫入請求也能自動轉換。

那麼要如何建立多個 Slave 去監聽 Master 的 Binlog？

## 首先架設 Master & Slave 需要設定這些參數：

- server-id : 該 server 的唯一識別號，主要用於 cluster 彼此識別身份，例如 slave 在拉取 binlog 時也會記錄該 binlog 是從哪個 server-id 來的，需要額外設定而不用 ip 或者 hostname 的原因在於如果 server 更換機器 ip & hostname 是可能改變的。
- log-bin：開啟 binlog 。
- log-bin-basename：指令 binlog 檔名的前綴，預設是 binlog，也可以透過指定 /foldera/folderb/mybinlog 來指定 binlog 儲存路徑，可將 binlog 儲存在不同硬碟避免影響查詢寫入的硬碟效能。
- binlog-format : 設定 binlog 格式，例如 ROW。
- gtid-mode：使用 gtid 作為 binlog 進度追蹤。
- enforce_gtid_consistency：確保所有寫入 Master 的指令都是 gtid-safe 也就是重複執行一定會產生一樣的值，例如 UPDATE … LIMIT 不指定 ORDER BY 就不是 gtid-safe 。
- log_replica_updates：當 server 為 slave 時 replay master 資料後會寫入其 binlog，當 slave 升級成 master 時，可以讓其他 slave 繼續接收 binlog 資料。
- binlog_expire_logs_seconds：binlog 檔案多久要過期，避免硬碟塞爆。
- bind-address：設定 server 傾聽的 ip 位置，建議為 0.0.0.0 才能讓 slave 透過 public ip 連進來。

而 **Slave** 的設定跟 Master 一樣，只是多了 read-only 確保他不能處理寫入請求。

```yaml!
services:  
   mysql1:  
    container_name: mysql1  
    image: mysql/mysql-server:8.0  
    platform: linux/amd64  
    networks:  
        - mysql_cluster  
    environment:  
        MYSQL_ROOT_PASSWORD: secret  
        MYSQL_ROOT_HOST: "%"  
        MYSQL_DATABASE: local_test  
        MYSQL_USER: test  
        MYSQL_PASSWORD: test  
    volumes:  
        - ./data/mysql1:/var/lib/mysql  
    ports:  
        - 127.0.0.1:3306:3306  
    command:  
    [  
        "--server-id=1",  
        "--log-bin=mysql1-bin",  
        "--binlog-format=ROW",  
        "--gtid-mode=ON",  
        "--enforce-gtid-consistency=ON",  
        "--log-replica-updates=ON",  
        "--binlog-expire-logs-seconds=604800",  
    ]  
  
   mysql2:  
    container_name: mysql2  
    image: mysql/mysql-server:8.0  
    platform: linux/amd64  
    networks:  
        - mysql_cluster  
    environment:  
        MYSQL_ROOT_PASSWORD: secret  
        MYSQL_ROOT_HOST: "%"  
        MYSQL_DATABASE: local_test  
        MYSQL_USER: test  
        MYSQL_PASSWORD: test  
    volumes:  
        - ./data/mysql2:/var/lib/mysql  
    ports:  
        - 127.0.0.1:3307:3307  
    command:  
    [  
        "--read_only=ON",  
        "--port=3307",  
        "--server-id=2",  
        "--log-bin=mysql2-bin",  
        "--binlog-format=ROW",  
        "--gtid-mode=ON",  
        "--enforce-gtid-consistency=ON",  
        "--log-replica-updates=ON",  
        "--binlog-expire-logs-seconds=604800",  
    ]  
  
networks:  
    mysql_cluster:  
    driver: bridge
```

執行 `docker compose -f ./mysql-cluster.yaml up -d` 啟動 server 後，需要進入 slave db 設定 Master DB 連線資訊：

```
CHANGE REPLICATION SOURCE TO  
SOURCE_HOST = 'mysql1',  
SOURCE_PORT = 3306,  
SOURCE_USER = 'root',  
SOURCE_PASSWORD = 'secret',  
SOURCE_AUTO_POSITION = 1;  
  
START REPLICA;  
```

`SOURCE_AUTO_POSITION=1` 代表使用 `gtid` 同，可執行 `SHOW SLAVE STATUS;` 檢查 Slave 同步狀況：

確認 Slave IO & SQL Thread 執行中，就沒問題了！

從頭建立 Cluster 簡單，但在既有的 Cluster 加入新 Slave 就有新的挑戰了。

當 Master Binlog 過期遺失後，新加入的 Slave 無法透過 Binlog 同步到完整資料時該怎麼辦？

**第一個方法是邏輯備份**，透過 `mysqldump` 將 Master DB 所有資料轉換成 SQL 指令輸出到特定檔案：

```shell!
mysqldump -h 127.0.0.1 -u root --password=secret  
--single-transaction \ <= 使用 snapshot transaction 讀資料  
--quick \ <= 使用批次處理避免一次載入大量資料到記憶體  
--skip-lock-tables \ <= 明確指定不要 Lock Table  
--set-gtid-purged=ON \ <= 產生 GTID_PURGED 指令  
local_test > backup.sql
```

執行後，會在 `backup.sql` 裡面看到 Master DB 將 Schema 以及資料轉成 `CREATE TABLE` 和 `INSERT` 的指令。

`SQL_LOG_BIN=0` 是避免執行下面指令時把資料寫入 binlog，由於是用 `mysqldump` 還原資料而不是 binlog replay，所以將 `backup.sql` 內容同步到 binlog 會造成 Cluster 內有相同 binlog 內容但不同 gtid 的情況。

設定 `GTID_PURGED` 為 `mysqldump` 拉資料當下 Master 已完成的 GTID Set，讓 Slave 在同步完 `backup.sql` 內容可以直接啟動 Replica 同步後續 binlog 資料，避開重複資料。

邏輯備份較耗時且資料量大時產生的 `backup.sql` 內容會特別大且複雜，備份過程也會消耗 master db 的 CPU。

**第二個方法是物理備份**，直接複製 MySQL `datadir` 底下資料，但備份方式不是單純執行 `cp` 指令就好，因為在複製的過程中，資料仍不斷在更新，單純複製會發生資料不一致的問題，例如 複製完前半段包含 id=100 的資料，隨後複製後半段 id=200 資料，此時同時更新了 `id = 100` & `id = 200` 資料會導致 `id=100` 為舊資料 `id=200` 為新資料。

為了解決這個問題需要使用 `Percona XtraBackup` 工具，其備份資料的也包含 redo log 內容，在備份完後，透過指令去 Replay redo log 內容就能讓資料更新到最新狀態。

```shell!
# 使用 percona-xtrabackup 從 master 複製檔案到 backup folder 中  
docker run --rm \  
--network compose_mysql_cluster \  
--user=root \  
-v ./data/mysql4:/backup \  
--volumes-from mysql1 \  
percona/percona-xtrabackup:8.0 \  
xtrabackup --backup \  
--target-dir=/backup \  
--host=mysql \  
--user=root \  
--password=secret --no-lock  

# replay redo log 更新資料到最新狀態  
docker run --rm \  
--network compose_mysql_cluster \  
--user=root \  
-v ./data/mysql4:/backup \  
--volumes-from mysql1 \  
percona/percona-xtrabackup:8.0 \  
xtrabackup --prepare \  
--target-dir=/backup \  
--host=mysql \  
--user=root \  
--password=secret --no-lock
```

物理備份限制是版本跟設定要一致，避免出現對資料格式不兼容的情況，但備份速度比邏輯備份快上很多。

## 架設完 Cluster 後，要如何做到自動 Auto Fail Over？

[orchestrator](https://github.com/openark/orchestrator) 是一個 MySQL Cluster 管理工具，提供 GUI 管理 MySQL Server 的網路關係，並提供 Server 狀態追蹤以及 Auto Fail Over 的功能。

![Screenshot 2026-01-06 at 21.06.31](https://hackmd.io/_uploads/HJFndK94bg.png)

啟動 orchestrator 前要設定好配置：

基礎配置 - 連上 MySQL Server 的通用帳號密碼，以及 Orchestrator 用的 DB 配置，Orchestrator 會用額外 DB 來儲存 Cluster 資訊

```json
"MySQLTopologyUser": "root", => 連上 mysql server 通用帳號  
"MySQLTopologyPassword": "secret", => 連上 mysql server 通用密碼  
"DefaultInstancePort": 3306, => 連上 mysql server 預設 port  
"MySQLOrchestratorHost": "orchestrator_db", => orchestrator 儲存 cluster 資訊的 db host  
"MySQLOrchestratorDatabase": "orchestrator", => orchestrator 儲存 cluster 資訊的 db name  
"MySQLOrchestratorUser": "root", => orchestrator 儲存 cluster 資訊的 db 帳號  
"MySQLOrchestratorPassword": "secret", => orchestrator 儲存 cluster 資訊的 db 密碼  
"MySQLOrchestratorPort": 3309, => orchestrator 儲存 cluster 資訊的 db port  
"ListenAddress": ":3000", => orchestrator admin gui port number  
"InstancePollSeconds": 5,  
"UnseenInstanceForgetHours": 1,  
"HTTPAuthUser": "admin", => admin 帳號  
"HTTPAuthPassword": "secret", => admin 密碼
```

服務發現配置 - Orchestrator 會透過 Master DB 主動發現 Slave，可在 GUI 畫面上設定一個 Master DB 連線 host & port ，並用兩種方式發現 Slave：

- `DiscoverByShowSlaveHosts` 參數為 True - Orchestrator 會執行 show slave hosts 指令找 slave
- `DiscoverByShowSlaveHosts` 參數為 False - Orchestrator 會執行 `select substring_index(host, ‘:’, 1) as slave_hostname from information_schema.processlist where command IN (‘Binlog Dump’, ‘Binlog Dump GTID’)` Query 找 slave

當獲得 Master & Slave Host 後，透過 [HostnameResolveMethod](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-resolve.md) ＆ [MySQLHostnameResolveMethod](https://github.com/openark/orchestrator/blob/master/docs/configuration-discovery-resolve.md) 參數將 Host 解析成 IP：

- HostnameResolveMethod：解析 master or slave hostname 的方式，例如在 docker 環境你會收到 mysql1 之類的，若 orchestrator 不在 docker 環境中你就需要將 mysql1 解析成 ip，此時可以將 HostnameResolveMethod 設定為 ip，也可設定成 none 不解析。

- MySQLHostnameResolveMethod：連上 Slave 後 Orchestrator 會執行 `SHOW SLAVE STATUS` 指令儲存 Slave 與 Master 關連，但如果 Orchestrator 是用 IP 連上 Master，但 Slave 是用 container (e.g mysql1) 連上 Master，資料比對會有問題，因此要透過該參數設定解析 MySQL IP 轉成 mysql1，例如設定參數為 report_host 會執行 `@@**global**.report_host` 獲取 MySQL 環境變數中 report_host 設定。

Auto Failover 配置 - Orchestrator 會定期檢查 Master 狀態，當有連線問題，且其他 Slave 也與他失聯後，就會啟動 Auto Failover：

- **ApplyMySQLPromotionAfterMasterFailover**：是否啟動 Failover，啟動後會透過 reset slave all & set read_only=0 指令將 slave 換成 master
- **PreventCrossDataCenterMasterFailover**：受否要避免相同 DataCenter 的 Slave 被提拔成 Master
- **PreventCrossRegionMasterFailover**：受否要避免相同 Region 的 Slave 被提拔成 Master
- **FailMasterPromotionIfSQLThreadNotUpToDate**：如果當 Slave Relay Log 都還沒 Replay 完是否要讓 Fail Over 失敗
- **FailMasterPromotionOnLagMinutes** : 當 Slave binlog lag 太久就要讓 Fail Over 失敗

另外也可以設置 Hook 來通知 Orchestrator 正在執行 Fail Over：

```json!
"PreFailoverProcesses": [  
"echo 'Will recover from {failureType} on {failureCluster}' >> /tmp/recovery.log"  
],  
"PostFailoverProcesses": [  
"echo '(for all types) Recovered from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Successor: {successorHost}:{successorPort}' >> /tmp/recovery.log"  
],
```

![Screenshot 2026-01-06 at 21.11.06](https://hackmd.io/_uploads/HkN0Yt5EZe.png)

![Screenshot 2026-01-06 at 21.11.13](https://hackmd.io/_uploads/SJOAFFcNbg.png)

## 最後，當 Master 替換後，Client 要如何在不替換連線的情況下將寫入請求送到新 Master？

可在 Cluster 前架設一個 [Proxy SQL](https://proxysql.com/)，透過 Proxy 自動分流，當 Master 替換成不同 Server，Proxy 也能自動偵測改變分流路線，程式端完全不需要修改配置。

Proxy SQL 除了分流 SQL 到不同 SQL Server 之外，還提供了：

- 統一管理連線池，避免太多 Server 各自建立大量連線，衝爆 MySQL Server 連線上限。
- 提供高度客製化的查詢快取，針對特定 Query 語法設定快取以及 TTL 時間。
- Query Rewrite 功能，例如把 SELECT * 改成 SELECT id, name。

Proxy SQL 本身自帶 SQLite 資料庫，會將 Cluster 連線資訊以及分流規則紀錄在裡面，此外也有很多系統參數可以微調行為，可以參考 https://proxysql.com/documentation/global-variables/。

以下提供基礎配置：

```json!
# 配置 sqlite 儲存路徑  
datadir="/var/lib/proxysql"  
  
# 配置 proxy sql admin 帳號密碼，以及模擬 mysql 介面的入口點  
# 使用者透過登入 admin 帳號來調整 proxy sql 設定  
admin_variables =  
{  
    admin_credentials="admin:admin"  
    mysql_ifaces="0.0.0.0:6032"  
}  
  
# 設定 mysql server 監控用的帳號密碼  
mysql_variables =  
{  
    monitor_username="root"  
    monitor_password="secret"  
}  
  
# 設定叢集資訊，hostgroup 1 為 master 2 為 slave，設定完後  
# proxysql 會定時監控 hostgroup 內的 mysql server read_only 參數  
# 並調整到正確的 host group id  
mysql_replication_hostgroups =  
(  
    {  
        writer_hostgroup=1  
        reader_hostgroup=2  
        comment="cluster1"  
    }  
)  
  
# 設定 mysql server 連線資訊，hostgroup 可以都設定成 1  
# 等 proxysql 透過 read_only 參數自行調整  
mysql_servers =  
(  
    {  
        address="mysql1"  
        port=3306  
        hostgroup=1  
        max_connections=200  
    },  
    {  
        address="mysql2"  
        port=3307  
        hostgroup=1  
        max_connections=200  
    },  
    {  
        address="mysql3"  
        port=3308  
        hostgroup=1  
        max_connections=200  
    }  
)  
  
# proxy sql 連上 mysql_servers 的帳號  
# application server 同時也會用該組帳號連上 proxysql  
# 再由 proxysql forward sql 到後面的 mysql server  
mysql_users =  
(  
    {  
        username = "root"  
        password = "secret"  
        default_hostgroup = 1  
        max_connections=1000  
        default_schema="information_schema"  
        active = 1  
    }  
)  
  
# 定義 query 分流規則，match_pattern 可以用 regex 去寫  
# 不用擔心每次分流都要通過一次 regex match 影響效能  
# proxy sql 會 cache 起來  
mysql_query_rules =  
(  
    {  
        rule_id=1  
        active=1  
        username="root"  
        match_pattern="^SELECT .* FOR UPDATE$"  
        destination_hostgroup=1  
        apply=1  
    },  
    {  
        rule_id=2  
        active=1  
        username="root"  
        match_pattern="^SELECT"  
        destination_hostgroup=2  
        apply=1  
    },  
    {  
        rule_id=3  
        active=1  
        username="root"  
        match_pattern="^INSERT|^UPDATE|^DELETE"  
        destination_hostgroup=1  
        apply=1  
    }  
)
```

詳細設定可參考：https://ithelp.ithome.com.tw/articles/10381791
