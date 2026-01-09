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

