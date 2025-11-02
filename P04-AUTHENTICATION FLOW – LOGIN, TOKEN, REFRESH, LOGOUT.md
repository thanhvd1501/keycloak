# 📔 PHẦN 4: AUTHENTICATION FLOW – LOGIN, TOKEN, REFRESH, LOGOUT

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Hiểu **cách Keycloak triển khai OAuth2 / OIDC** trong thực tế.
2. Phân biệt rõ **Authorization Code Flow (với PKCE)**, **Password Flow**, **Client Credentials Flow**.
3. Thực hành **login, refresh token, logout** bằng Postman hoặc cURL.
4. Nắm được **cấu trúc và thời hạn của Access Token / Refresh Token / ID Token**.
5. Hiểu rõ **luồng dữ liệu giữa Frontend – Keycloak – Spring Boot**.

---

## 🧭 1. Tổng quan các OAuth2 Flow Keycloak hỗ trợ

| Flow                          | Dành cho                                   | Mô tả ngắn                                            |
| ----------------------------- | ------------------------------------------ | ----------------------------------------------------- |
| **Authorization Code (PKCE)** | SPA / Web App (React, NextJS)              | Bảo mật cao nhất – user login qua trình duyệt         |
| **Resource Owner Password**   | CLI / Test / Legacy                        | Gửi username & password trực tiếp – không khuyến nghị |
| **Client Credentials**        | Service ↔ Service (microservices)          | Không có user – dùng cho máy chủ giao tiếp            |
| **Refresh Token**             | Giữ phiên đăng nhập mà không cần login lại | Dùng refresh token để xin access token mới            |
| **Logout**                    | Mọi loại client                            | Xoá session Keycloak + revoke refresh token           |

---

## 🔍 2. Authorization Code Flow (chuẩn OIDC – khuyến nghị)

Đây là **flow chính** khi bạn có **Frontend (React, Vue, NextJS)** và **Backend API (Spring Boot)**.

### 🔑 Cấu trúc cơ bản:

```text
[Frontend] → [Keycloak] → [Frontend] → [Backend]
```

---

### 🧠 Sơ đồ minh họa text-based chi tiết

```text
1️⃣ User click “Login” ở Frontend
     ↓
2️⃣ Frontend redirect đến Keycloak:
     GET /realms/demo/protocol/openid-connect/auth
     ?client_id=react-client
     &redirect_uri=http://localhost:3000/callback
     &response_type=code
     &scope=openid profile email
     &code_challenge=<PKCE_CODE>
     &code_challenge_method=S256
     ↓
3️⃣ Keycloak hiển thị trang đăng nhập
     User nhập username/password
     ↓
4️⃣ Keycloak xác thực → redirect lại Frontend kèm “authorization_code”
     → http://localhost:3000/callback?code=xyz
     ↓
5️⃣ Frontend gọi Backend:
     POST /token
     {
       "code": "xyz",
       "code_verifier": "<PKCE_VERIFIER>"
     }
     ↓
6️⃣ Backend gọi Keycloak:
     POST /realms/demo/protocol/openid-connect/token
     → nhận Access Token + Refresh Token + ID Token
     ↓
7️⃣ Backend trả về Access Token cho Frontend (hoặc giữ trong cookie secure)
     ↓
8️⃣ Mọi request API tiếp theo:
     Authorization: Bearer <access_token>
```

---

## 💻 3. Flow Login & Token Exchange qua Postman

### 1️⃣ Lấy Authorization Code

Gửi request trong browser hoặc Postman (method GET):

```
http://localhost:8080/realms/demo/protocol/openid-connect/auth?
client_id=spring-client&
response_type=code&
redirect_uri=http://localhost:8081/callback&
scope=openid
```

→ Sau khi login, bạn sẽ thấy redirect URL:

```
http://localhost:8081/callback?code=abc123
```

---

### 2️⃣ Đổi code lấy Access Token

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/token \
  -d "grant_type=authorization_code" \
  -d "client_id=spring-client" \
  -d "client_secret=<SECRET>" \
  -d "code=abc123" \
  -d "redirect_uri=http://localhost:8081/callback"
```

✅ Nhận về:

```json
{
  "access_token": "eyJhbGciOiJSUzI1...",
  "refresh_token": "eyJhbGciOiJIUzI1...",
  "id_token": "...",
  "token_type": "Bearer",
  "expires_in": 300,
  "refresh_expires_in": 1800
}
```

---

## 🔁 4. Refresh Token Flow (Làm mới Access Token)

> Khi Access Token (5 phút) hết hạn, bạn không cần login lại — chỉ cần refresh.

### Gửi request:

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/token \
  -d "grant_type=refresh_token" \
  -d "client_id=spring-client" \
  -d "client_secret=<SECRET>" \
  -d "refresh_token=<refresh_token>"
```

✅ Trả về Access Token mới:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "mới",
  "expires_in": 300
}
```

---

## 🚪 5. Logout Flow

Đăng xuất khỏi Keycloak và thu hồi token.

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/logout \
  -d "client_id=spring-client" \
  -d "client_secret=<SECRET>" \
  -d "refresh_token=<refresh_token>"
```

✅ Kết quả:

* Refresh token bị thu hồi.
* Access token hết hiệu lực.
* User bị đăng xuất khỏi session Keycloak.

---

## 🧱 6. Password Flow (Direct Access Grant) – chỉ dùng cho test/CLI

Dành cho các trường hợp **không có frontend** (Postman, CLI).

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=spring-client" \
  -d "client_secret=<SECRET>" \
  -d "username=alice" \
  -d "password=alice123"
```

✅ Nhận về Access Token + Refresh Token.

⚠️ Không nên dùng trong production vì nó truyền username/password trực tiếp.

---

## 🤝 7. Client Credentials Flow (Microservices-to-Microservices)

Dùng khi **service A gọi service B**, không có user cụ thể nào.

Cấu hình Keycloak:

* Tạo client `service-a`
* Bật `Client Authentication`
* Tắt `Direct Access Grant`

Gọi:

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=service-a" \
  -d "client_secret=<SECRET>"
```

✅ Trả về Access Token đại diện cho **client**, không có user claim.

---

## 🧠 8. Giải thích token và thời hạn

| Token             | Dùng cho                | Thời gian sống mặc định | Có thể refresh?      |
| ----------------- | ----------------------- | ----------------------- | -------------------- |
| **Access Token**  | Gọi API                 | 5 phút                  | ✅ bằng Refresh Token |
| **Refresh Token** | Xin Access Token mới    | 30 phút                 | ❌                    |
| **ID Token**      | Hiển thị thông tin user | 5 phút                  | ❌                    |

Bạn có thể chỉnh trong:

> **Realm Settings → Tokens → Access Token Lifespan**

---

## 🧩 9. Sơ đồ tóm tắt flow tổng thể

```text
   +----------+               +-----------+              +---------------+
   |  Browser |               | Keycloak  |              | Spring Boot API |
   +----------+               +-----------+              +---------------+
        |                          |                             |
        |-- (1) /auth ------------>|                             |
        |<-(2) login page ---------|                             |
        |--(3) credentials ------->|                             |
        |<-(4) redirect code ------|                             |
        |--(5) /token ------------>|                             |
        |<-(6) token --------------|                             |
        |--(7) /api/user ----------> Authorization: Bearer <token>|
        |<-(8) response -----------|                             |
        |--(9) /logout ----------->|                             |
        |<-(10) session ended -----|                             |
```

---

## ⚙️ Bài tập thực hành

1. Thực hiện Authorization Code Flow (code → token) qua Postman.
2. Lấy Access Token và gọi Spring Boot API `/api/user/hello`.
3. Khi token hết hạn, thực hiện Refresh Token Flow.
4. Đăng xuất bằng logout endpoint và thử lại (sẽ thấy token hết hiệu lực).

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                               | Giải thích                                |
| ------------------------------------- | ----------------------------------------- |
| Dùng “password grant” trong SPA       | Không an toàn – tiết lộ password          |
| Không lưu refresh token               | User bị đăng xuất sớm                     |
| Gọi sai redirect URI                  | Keycloak báo “Invalid redirect_uri”       |
| Dùng Access Token đã hết hạn          | Nhận lỗi 401 từ Spring Boot               |
| Không revoke refresh token khi logout | User vẫn có thể login lại bằng refresh cũ |

---

## ✅ Best Practices

* Với **SPA / Web App** → luôn dùng **Authorization Code Flow + PKCE**.
* Với **microservices** → dùng **Client Credentials Flow**.
* Luôn **xác minh token hết hạn (exp)** trong backend.
* Sử dụng **secure cookie (HttpOnly)** để lưu token ở frontend.
* Bật **SSL (https)** ngay cả trong môi trường staging.
* Dùng **short access token, long refresh token** để cân bằng bảo mật và UX.
