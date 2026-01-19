> 📌 此文件來自 https://ithelp.ithome.com.tw/users/20177857/ironman 的 IT 邦鐵人賽教程，僅針對個人學習用途進行筆記與修改。

# MySQL 的 Schema 管理

Transaction ACID 中的 Consistency 要求資料更新前後要符合資料規則，確保資料正確，而資料規則主要是透過 Schema 來制定的，例如 Data Type & Constraint 等，實務上 Schema 很難建立了就不修改，修改 Schema 的 SQL 雖然很簡單，但其中卻暗藏陷阱，一不小心就會 Lock Table 導致查詢寫入全部卡住。

## MySQL Schema 儲存在哪裡？

| 版本 | 儲存位置 | 特性 |
| :--- | :--- | :--- |
| **8.0 之前** | `.frm` 檔案 | 單純檔案儲存 |
| **8.0 之後** | System Table Space 的 `.idb` | 將 Schema 資料視為紀錄，**能使用 Transaction 去更新** |

> [!NOTE]
> 不論 `.frm` 還是 system table space，Schema 資料與資料皆是**分開儲存**。
> **好處**：節省空間，不用每一筆資料都重複儲存 Schema 定義。

### 既然是分開儲存，那麼更新 Schema 時，就不會去更新到資料對嗎？

答案是<span style="color: orange">不一定</span>，例如:
```sql
ALTER TABLE users ADD COLUMN count SMALLINT NOT NULL AFTER name
```

新增欄位時加了 AFTER 就需要去更新資料，而更新勢必要上鎖，就會導致 `Table Lock`，如果 Table 資料多，鎖的時間就會很久。

原因在於 Page 儲存資料的格式是 Schema 定義的，例如

```sql
CREATE TABLE users (  
	id int auto_increment,  
	name char(100),  
	gender int,  
	primary key (id)  
); 
```

資料格式是由 Schema 定義的，以 `users` Table 為例：

**修改前 (Original Schema):**
```
[id (4 bytes)] | [name (100 bytes)] | [gender (4 bytes)]
```

MySQL 讀取一筆 Row 預期長度為 `4 + 100 + 4 = 108 bytes`。

**修改後 (After ADD COLUMN ... AFTER name):**
```
[id (4 bytes)] | [name (100 bytes)] | [count (2 bytes)] | [gender (4 bytes)]
```
Schema 變更後，預期長度變為 `4 + 100 + 2 + 4 = 110 bytes`。

> [!CAUTION]
> **問題點：**
> 如果 `.idb` 檔案中的舊資料沒有重寫更新，MySQL 依據**新 Schema** 去讀取**舊資料**時，就會發生錯亂 (例如：讀完 `name` 後，錯誤地將 `gender` 的前 2 bytes 當作 `count` 讀取)，導致解析失敗。

## 那麼只要 ADD COLUMN 都會變動格式，都會 Lock Table 一段時間嗎？

<span style="color: orange">不一定！</span> MySQL 修改 Schema 的演算法有三個，`Copy` & `Instant` & `InPlace`。

| 演算法 | 機制說明 | 鎖定 (Lock) 特性 | 適用場景 & 備註 |
| :--- | :--- | :--- | :--- |
| **Copy** | 建立新的 `.idb` 檔，將舊資料依照新格式複製過去，最後刪除舊檔。 | **需對資料上鎖 (Table Lock)**<br>搬移過程中鎖定，避免資料不一致，耗時較長。 | **會破壞資料格式的變更**<br>如：修改 Data Type、`ADD COLUMN ... AFTER`。 |
| **Instant** | 僅修改 Schema Metadata。若讀取舊資料時發現長度不足 (格式變更)，系統會自動補上預設值，無需物理重寫資料。 | **Table Lock 時間極短**<br>執行非常快。 | **不破壞資料格式**<br>如：`ALTER COLUMN NAME`、`ADD COMMENT`<br>**特定新增欄位**<br>如：`ADD COLUMN` (在 Schema 尾端且可為 NULL 或有 Default Value)。 |
| **InPlace** | 不建立新檔，在原 `.idb` 擴展空間。<br>1. **Background Thread**: 背景同步資料 (如建立 Index)。<br>2. **Online Alter Log**: 紀錄並重放搬移期間的異動。 | **不需長時間 Lock**<br>執行過程允許讀寫，僅在最後更新 Schema 時短暫上鎖。 | **需重建結構但不強制重寫格式**<br>如：`ADD INDEX` (需為每筆資料建立新的 Index Tree)。 |

其實理論上 `ADD COLUMN` 若影響資料格式也可參考 InPlace 算法實作，可惜 MySQL 會用 COPY 算法，但如果你會調整資料格式又不想 Lock Table 那麼久，可以用 **online-schema-change** 工具，其概念是 InPlace + Copy 整合，他的流程是：

1. 建立新 Schema 的 Table (e.g new_users)
2. 在舊的 Table (e.g users) 建立一個 `INSERT` & `UPDATE` & `DELETE` 三種 trigger，透過 trigger 同步舊 Table 的異動到新 table
3. 使用 Primary Key 批次搬移舊資料到新資料
4. 資料搬完後，會獲取新舊兩個 Table 的 Table Lock 然後 rename 新 Table 名稱跟舊 Table 名稱 (e.g users -> old_users; new_users -> users)
5. 移除舊 Table (e.g old_users)

