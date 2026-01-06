# Authentication & Authorization (認證與授權)

系統安全的核心支柱，區分「你是誰」和「你能做什麼」。

## 1. 概念區分

| 概念 | 英文 | 定義 | 範例 |
|:---|:---|:---|:---|
| **認證** | Authentication (AuthN) | 驗證用戶身份 | 登入、指紋辨識、OTP |
| **授權** | Authorization (AuthZ) | 驗證是否有權限執行操作 | 管理員刪除貼文、VIP 觀看影片 |

---

## 2. 認證機制

### 2.1 Session-based (傳統)

1. Client POST 帳密
2. Server 驗證成功，建立 Session，回傳 `Set-Cookie: session_id=xyz` (HttpOnly)
3. Client 後續請求自動帶上 Cookie
4. Server 查詢 Session Store (Redis) 驗證

- ✅ 易於註銷 (Revoke)
- ❌ 需維護 Server 狀態 (Stateful)，CORS 問題

### 2.2 Token-based (JWT)

**JSON Web Token (JWT)** 是無狀態的認證憑證。

結構：`Header.Payload.Signature`

```json
Header: {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "123", "role": "admin", "exp": 1609459200}
Signature: HMACSHA256(base64(Header) + "." + base64(Payload), secret)
```

- ✅ 無狀態 (Stateless)，適合微服務
- ✅ 易於跨域 (Header: `Authorization: Bearer <token>`)
- ❌ 難以註銷 (需配合 Blacklist 或短效期 + Refresh Token)

**Refresh Token 模式：**
- **Access Token**: 短效期 (如 15 min)，存記憶體
- **Refresh Token**: 長效期 (如 7 days)，存 HttpOnly Cookie (安全)
- 當 Access Token 過期，用 Refresh Token 換新的

### 2.3 SSO (Single Sign-On)

用戶登入一次，即可存取代多個系統。
- 常見協議：SAML (企業用), CAS, OIDC

---

## 3. OAuth 2.0 & OIDC

### 3.1 OAuth 2.0 (授權框架)

讓用戶授權第三方應用存取其資源，而不洩露密碼。

**角色：**
- **Resource Owner**: 用戶
- **Client**: 第三方應用
- **Authorization Server**: 發 Token 的 (如 Google Auth)
- **Resource Server**: 存資料的 (如 Google Photos)

**授權碼模式 (Authorization Code Grant) - 最常用：**

```
1. Client ──Redirect──→ Auth Server (User 登入並同意)
2. Auth Server ──Redirect with Code──→ Client (Backend)
3. Client (Backend) ──POST Code──→ Auth Server
4. Auth Server ──Return Access Token──→ Client
```
- Code 在後端交換，Token 不暴露給瀏覽器，安全性高。

### 3.2 OpenID Connect (OIDC) (認證層)

OAuth 2.0 只做授權，OIDC 在其之上增加了 **ID Token** (JWT 格式)，用於標準化的身份認證。

- **OAuth 2.0**: "我有權限讀取相片" (Access Token)
- **OIDC**: "我是 John Doe，這是我的 Email" (ID Token)

---

## 4. 授權模型

### 4.1 RBAC (Role-Based Access Control)

基於「角色」分配權限。

```
Role: Admin
Permissions: [create_user, delete_user, view_report]

User: Alice -> Admin
Result: Alice has [create_user, ...]
```
- ✅ 簡單、直觀
- ❌ 難以處理細細微性條件 (如：只能刪除"自己"的貼文)

### 4.2 ABAC (Attribute-Based Access Control)

基於「屬性」動態計算權限。

**Policy:**
```
Allow Update IF:
  Subject.Role == 'Editor' AND
  Resource.Owner == Subject.ID AND
  Environment.Time < 18:00
```
- ✅ 極其靈活
- ❌ 規則複雜，效能較差

---

## 5. 安全性最佳實踐

| 措施 | 說明 |
|:---|:---|
| **HTTPS** | 強制全站 HTTPS，防止中間人攻擊 |
| **HttpOnly Cookie** | 防止 XSS 竊取 Session/Token |
| **CSRF Token** | 防止跨站請求偽造 (Session 模式必需) |
| **Password Hashing** | 決不明文存密碼，使用 **Bcrypt** 或 **Argon2** 加鹽 |
| **MFA** | 多因素認證 (SMS, TOTP) |
| **Principle of Least Privilege** | 最小權限原則 |

---

## 6. Reference
- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [Auth0 - Authentication vs Authorization](https://auth0.com/docs/get-started/identity-fundamentals/authentication-and-authorization)
