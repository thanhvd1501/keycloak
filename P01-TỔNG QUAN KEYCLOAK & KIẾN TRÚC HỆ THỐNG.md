### 📘 PHẦN 1: TỔNG QUAN KEYCLOAK & KIẾN TRÚC HỆ THỐNG

---

#### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Hiểu **Keycloak là gì**, tại sao nó được dùng trong xác thực & phân quyền hiện đại.
2. Nắm rõ **kiến trúc bên trong Keycloak** (realm, client, user, role, token).
3. Phân biệt **SSO, OAuth2, OIDC** và vai trò của Keycloak trong hệ sinh thái bảo mật.
4. Hiểu **luồng xác thực (authentication flow)** giữa Spring Boot ↔ Keycloak ↔ Frontend.
5. Có hình dung thực tế về **mô hình triển khai** và **cách các thành phần tương tác**.

---

### 📗 1. Keycloak là gì?

**Keycloak** là một **Identity and Access Management (IAM)** open-source do Red Hat phát triển, giúp bạn **xác thực (authentication)** và **phân quyền (authorization)** cho ứng dụng mà **không cần tự viết hệ thống login phức tạp**.

💡 Nói ngắn gọn:

> “Keycloak là trung tâm quản lý danh tính (Identity Provider) cho toàn bộ hệ thống, giúp bạn đăng nhập 1 lần (SSO), quản lý user, role, và token an toàn.”

---

### 📙 2. Vì sao cần Keycloak?

| Vấn đề                                                        | Giải pháp Keycloak                   |
| ------------------------------------------------------------- | ------------------------------------ |
| Phải tự viết form login, reset password, verify email, OAuth2 | Keycloak cung cấp sẵn UI và API      |
| Nhiều app cần đăng nhập 1 lần (SSO)                           | Keycloak quản lý chung qua **realm** |
| Cần quản lý user, role, group, quyền truy cập                 | Có **RBAC & Policy Engine**          |
| Muốn login bằng Google, Facebook, GitHub                      | Hỗ trợ **Identity Brokering**        |
| Muốn quản lý token, refresh, revoke                           | Có sẵn **OIDC/OAuth2 Token Service** |

---

### 📒 3. Kiến trúc tổng quan của Keycloak

Một Keycloak server có nhiều **realm** – mỗi realm là **một không gian bảo mật độc lập**, có **users, clients, roles** riêng biệt.

#### 🔹 Thành phần chính:

| Thành phần                  | Vai trò                                                              |
| --------------------------- | -------------------------------------------------------------------- |
| **Realm**                   | Không gian quản lý riêng (như tenant). Ví dụ: `demo-realm`.          |
| **User**                    | Người dùng cuối (đăng nhập, có vai trò).                             |
| **Group**                   | Nhóm người dùng (gán quyền tập thể).                                 |
| **Client**                  | Ứng dụng kết nối với Keycloak (ví dụ: Spring Boot API, React App).   |
| **Role**                    | Vai trò (quyền hạn), có thể thuộc realm hoặc client.                 |
| **Token**                   | JWT chứa thông tin user, roles, scope, expiration.                   |
| **Identity Provider (IdP)** | Dịch vụ xác thực bên ngoài (Google, Azure AD…).                      |
| **Keycloak Admin Console**  | Giao diện quản trị ([http://localhost:8080](http://localhost:8080)). |
| **Keycloak REST API**       | Giao tiếp tự động hóa với Keycloak (tạo user, client, role...).      |

---

### 🧠 4. Luồng xác thực OIDC / OAuth2 trong Keycloak

Hãy tưởng tượng hệ thống gồm:

* **Frontend** (React/Vue)
* **Backend API** (Spring Boot)
* **Keycloak Server**

---

#### 🧩 4.1 Luồng **Authorization Code Flow** (chuẩn OIDC):

```text
[1] User → Frontend: Nhấn “Login”
[2] Frontend → Keycloak: Redirect đến /realms/demo/protocol/openid-connect/auth
[3] Keycloak → User: Hiển thị trang đăng nhập
[4] User → Keycloak: Nhập username/password → xác thực thành công
[5] Keycloak → Frontend: Redirect về URL callback kèm “authorization code”
[6] Frontend → Backend (Spring Boot): Gửi code để đổi lấy token
[7] Backend → Keycloak: Gọi /token endpoint để lấy Access Token + Refresh Token + ID Token
[8] Backend → Lưu Access Token, sử dụng Bearer Token khi gọi API
[9] Spring Boot → Xác thực token qua public key của Keycloak (JWT verification)
```

---

#### 🔑 Token types:

| Token             | Mục đích                                 | Thời hạn |
| ----------------- | ---------------------------------------- | -------- |
| **Access Token**  | Dùng để truy cập API                     | ~5 phút  |
| **Refresh Token** | Dùng để lấy token mới mà không login lại | ~30 phút |
| **ID Token**      | Chứa thông tin người dùng (name, email)  | ~5 phút  |

---

#### 📊 Sơ đồ text-based mô tả flow:

```text
+-----------+          +------------+          +--------------+
|  Browser  |          |  Keycloak  |          |  Spring Boot |
+-----------+          +------------+          +--------------+
      |                       |                         |
      |--- (1) Login -------->|                         |
      |                       |                         |
      |<-- (2) Redirect ------|                         |
      |--- (3) Send Code ------------------------------->|
      |                       |                         |
      |                       |<-- (4) Exchange Code -->|
      |                       |--- (5) Return Token ---->|
      |                       |                         |
      |<-- (6) Access API (Bearer Token) -------------->|
      |                       |                         |
```

---

### 📘 5. Keycloak trong hệ thống Spring Boot

Trong ứng dụng Spring Boot 3.x, ta cấu hình **Resource Server** để xác thực JWT phát hành bởi Keycloak.

Ví dụ `application.yml`:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/demo
```

Spring Security sẽ tự động:

* Gọi endpoint `/.well-known/openid-configuration` của Keycloak để lấy public key.
* Kiểm tra chữ ký JWT.
* Ánh xạ `roles` trong token thành `GrantedAuthorities`.

---

### 💻 Ví dụ thực tế nhanh

#### Docker Compose chạy Keycloak:

```yaml
version: "3"
services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    command:
      - start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    ports:
      - "8080:8080"
```

> Chạy lệnh:
>
> ```bash
> docker-compose up -d
> ```
>
> Truy cập: [http://localhost:8080](http://localhost:8080)
> Đăng nhập admin / admin → Tạo **realm**, **client**, **user** đầu tiên.

---

### ⚙️ Bài tập thực hành

1. Chạy Keycloak bằng Docker Compose.
2. Truy cập Admin Console, tạo:

   * Realm: `demo`
   * Client: `spring-client` (confidential)
   * User: `alice`
   * Role: `USER`
3. Dùng Postman:

   * Gọi `/token` endpoint để lấy Access Token.
   * Decode JWT tại [jwt.io](https://jwt.io).

---

### ⚠️ Sai lầm phổ biến

| Sai lầm                                   | Giải thích                                         |
| ----------------------------------------- | -------------------------------------------------- |
| Dùng port 8080 trùng với Spring Boot      | Đổi port Keycloak sang 8081                        |
| Không phân biệt realm role và client role | Realm role = toàn cục; Client role = theo ứng dụng |
| Không cấu hình `issuer-uri` đúng          | Gây lỗi “Invalid Issuer” khi xác thực JWT          |
| Không lưu refresh token                   | Người dùng bị đăng xuất sớm                        |
| Nhầm giữa Access Token và ID Token        | API chỉ nên dùng **Access Token**                  |

---

### ✅ Best Practices

* Luôn tách riêng **realm** cho từng môi trường (dev/test/prod).
* Sử dụng **HTTPS** cho mọi giao tiếp (dù là local).
* Giữ **token lifetime** ngắn, kết hợp refresh token.
* Không hardcode client secret trong code – dùng `.env` hoặc secret manager.
* Sử dụng **Keycloak Admin REST API** để tự động hoá tạo user/client.
