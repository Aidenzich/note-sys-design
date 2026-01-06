# Encryption (加密)

加密是保護資料機密性 (Confidentiality) 和完整性 (Integrity) 的最後一道防線。

## 1. 資料狀態分類

| 狀態 | 英文 | 風險 | 防護手段 |
|:---|:---|:---|:---|
| **靜態資料** | Data at Rest | 硬碟失竊、資料庫洩露 | 硬碟加密、TDE、KMS Envelope Encryption |
| **傳輸中資料** | Data in Transit | 中間人攻擊 (MITM)、監聽 | TLS 1.3、mTLS、VPN |
| **使用中資料** | Data in Use | 記憶體 Dump、Process 注入 | Enclave (SGX)、Homomorphic Encryption (實驗性) |

---

## 2. 靜態資料加密 (Encryption at Rest)

### 2.1 對稱加密 (Symmetric Encryption)

使用同一把金鑰進行加解密，速度快。

- **演算法**: **AES-256-GCM** (Galois/Counter Mode) 是目前業界標準，同時提供加密和完整性驗證。
- **金鑰管理**: 金鑰本身必須被嚴密保護 (Key Management Service, KMS)。

### 2.2 Envelope Encryption (信封加密)

解決「如何保護金鑰」的問題，特別是當資料量很大時。

```
1. 產生 Data Key (DK) (明文 + 密文)
   KMS GenerateDataKey() → {Plain_DK, Encrypted_DK}

2. 用 Plain_DK 加密大檔案
   AES(File, Plain_DK) → Encrypted_File

3. 銷毀記憶體中的 Plain_DK
   
4. 儲存
   Storage = {Encrypted_File + Encrypted_DK}
```

**解密流程：**
1. 讀取 `Encrypted_DK`
2. 呼叫 KMS Decrypt(`Encrypted_DK`) → 取得 `Plain_DK`
3. 用 `Plain_DK` 解密檔案

**優點：**
- KMS Master Key 永遠不離開 HSM (硬體安全模組)
- 大量資料無需傳輸到 KMS，效能高

### 2.3 Transparent Data Encryption (TDE)

資料庫層級的透明加密 (如 MySQL, SQL Server)。
- App 無需修改程式碼
- DB Engine 在寫入 Disk 前加密 Page，讀取時解密
- 防止硬碟被拔走但無法防止 SQL Injection

---

## 3. 傳輸中資料加密 (Encryption in Transit)

### 3.1 TLS 1.3 (Transport Layer Security)

目前最安全的傳輸協定。

**改進 (相較於 TLS 1.2)：**
- **0-RTT Resumption**: 更快的重連速度
- **移除不安全演算法**: 廢棄 RSA Key Exchange, RC4, CBC 等
- **強制 Perfect Forward Secrecy (PFS)**: 即使 Private Key 被竊，也無法解密過去的流量

### 3.2 憑證 (Certificates)

- **CA (Certificate Authority)**: 信任鏈的根基 (如 Let's Encrypt, DigiCert)
- **Self-signed**: 自簽憑證，用於內部測試，瀏覽器會報錯
- **mTLS (Mutual TLS)**: 雙向驗證，Server 驗 Client，Client 也驗 Server (用於微服務間)

---

## 4. 密碼雜湊 (Hashing)

**絕對不要加密密碼！要雜湊 (Hash)。**

### 4.1 演算法選擇

| 演算法 | 狀態 | 建議 | 原因 |
|:---|:---|:---|:---|
| **MD5** | 💀 已死 | 禁止使用 | 碰撞攻擊容易 |
| **SHA-1** | 💀 已死 | 禁止使用 | Google 已攻破 |
| **SHA-256** | ⚠️ 不推薦存密碼 | 用於簽章 | 計算太快，容易被 GPU 暴力破解 |
| **Bcrypt** | ✅ 推薦 | 通用 | 可調整 Cost Factor，抗 GPU |
| **Argon2** | ⭐ 最佳 | 強烈推薦 | 抗 GPU 和 ASIC，贏得密碼雜湊競賽 |
| **PBKDF2** | 🆗 可用 | 合規需要 | NIST 核准，但不如 Argon2 |

### 4.2 Salt (鹽)

防止 Rainbow Table 攻擊。
- 每個用戶必須有**獨一無二**的隨機 Salt
- `Hash = Argon2(Password + Salt)`

---

## 5. 常見加密誤區

1. **自己發明加密演算法**：千萬別做！使用已經驗證的標準庫 (Libsodium, Tink)。
2. **Hardcode 金鑰**：金鑰存代碼庫是嚴重漏洞，應使用 Environment Variables 或 Secret Manager。
3. **ECB 模式**：AES 使用 ECB 模式會保留相同的模式 (如企鵝圖)，不安全。應使用 GCM。
4. **忽略隨機數生成器 (RNG)**：加密需要 `Cryptographically Secure PRNG` (/dev/urandom)，不能用 `rand()`。

---

## 6. Reference
- [Google Tink Library](https://github.com/google/tink)
- [NIST Cryptographic Standards](https://csrc.nist.gov/projects/cryptographic-standards-and-guidelines)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
