> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

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