## 📗 PHẦN 2: CÀI ĐẶT & CẤU HÌNH KEYCLOAK (Docker + Realm + Client)

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Cài đặt và chạy Keycloak 23.x qua **Docker Compose** (với PostgreSQL).
2. Hiểu cấu trúc **realm, client, user, role, group** trong Admin Console.
3. Tạo **realm mới**, **client cho Spring Boot**, **user và role**.
4. Cấu hình bảo mật client (confidential / public) và redirect URIs.
5. Gọi thử **token endpoint** để xác thực và lấy JWT bằng Postman.

---

## 🧩 1. Cấu trúc triển khai Keycloak (Kiến trúc tổng thể)

```text
+-----------------------------+
|        Keycloak Server      |
|-----------------------------|
| Realm: demo                 |
|   ├── Users (alice, bob)    |
|   ├── Roles (USER, ADMIN)   |
|   ├── Clients:              |
|   │    ├── spring-client    |
|   │    ├── react-frontend   |
|   │    └── postman-client   |
|-----------------------------|
| Database: PostgreSQL        |
+-----------------------------+

       ▲            ▲
       |            |
       |            |
Frontend (React)    Spring Boot API
    (OIDC)             (JWT verify)
```

---

## 🐳 2. Cài đặt Keycloak qua Docker Compose

Tạo file `docker-compose.yml`:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
    volumes:
      - keycloak_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    command:
      - start-dev
      - --import-realm
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
      KC_DB_SCHEMA: public
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_PROXY: edge
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  keycloak_data:
```

> 💡 Chạy lệnh:
>
> ```bash
> docker-compose up -d
> ```
>
> Truy cập: [http://localhost:8080](http://localhost:8080)
> → Đăng nhập bằng: `admin / admin`

---

## 🏗️ 3. Tạo Realm & Client & User trong Keycloak

### 🧭 Bước 1: Tạo **Realm**

1. Vào Admin Console → menu **Realms** (góc trái trên cùng).
2. Chọn **Add Realm** → nhập tên: `demo`
3. Lưu lại.

---

### ⚙️ Bước 2: Tạo **Client** cho ứng dụng Spring Boot

1. Trong realm `demo` → chọn **Clients → Create client**.

2. Nhập:

   * **Client ID**: `spring-client`
   * **Client type**: `OpenID Connect`
   * **Client authentication**: ✅ Bật (đây là “confidential client”)
   * Nhấn **Next**

3. Ở tab **Settings**:

   * **Root URL**: `http://localhost:8081` *(nếu Spring Boot chạy port 8081)*
   * **Valid redirect URIs**: `http://localhost:8081/*`
   * **Web origins**: `*` *(hoặc để rỗng nếu chỉ backend dùng)*
   * **Direct Access Grants Enabled**: ✅ (cho phép login bằng username/password qua API)
   * Nhấn **Save**.

4. Sang tab **Credentials** → copy **Client Secret**.
   (Dùng cho Spring Boot sau này).

---

### 👤 Bước 3: Tạo **User**

1. Vào **Users → Add User**

   * Username: `alice`
   * Email: `alice@example.com`
   * Enabled: ✅
   * Lưu lại.
2. Qua tab **Credentials** → Set Password → `alice123`

   * Disable “Temporary” → ✅ để user có thể login luôn.

---

### 🧱 Bước 4: Tạo **Role** & gán cho User

1. Vào **Roles → Add Role**

   * Role name: `USER`
   * Lưu lại.
2. Mở user `alice` → tab **Role Mappings**

   * Chọn `USER` → Add Selected.

---

## 🔑 4. Test lấy Access Token bằng Postman / cURL

### 📮 Endpoint chuẩn Keycloak:

```
POST http://localhost:8080/realms/demo/protocol/openid-connect/token
```

### Body (x-www-form-urlencoded):

| Key           | Value         |
| ------------- | ------------- |
| grant_type    | password      |
| client_id     | spring-client |
| client_secret | <SECRET>      |
| username      | alice         |
| password      | alice123      |

### Ví dụ cURL:

```bash
curl -X POST http://localhost:8080/realms/demo/protocol/openid-connect/token \
  -d "client_id=spring-client" \
  -d "client_secret=<SECRET>" \
  -d "grant_type=password" \
  -d "username=alice" \
  -d "password=alice123"
```

👉 Kết quả:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9....",
  "expires_in": 300,
  "refresh_token": "...",
  "id_token": "...",
  "scope": "openid email profile"
}
```

---

## 🧠 5. Giải thích ngắn gọn các loại token

| Token             | Dùng để                                 | Ai dùng                |
| ----------------- | --------------------------------------- | ---------------------- |
| **Access Token**  | Gọi API (Spring Boot sẽ verify JWT này) | Backend                |
| **Refresh Token** | Xin lại Access Token khi hết hạn        | Backend / Frontend     |
| **ID Token**      | Chứa thông tin người dùng               | Frontend hiển thị info |

---

## 💡 6. Kiểm tra token JWT

Copy access token và dán vào [jwt.io](https://jwt.io)
Bạn sẽ thấy các claim quan trọng:

```json
{
  "preferred_username": "alice",
  "realm_access": { "roles": ["USER"] },
  "aud": "account",
  "iss": "http://localhost:8080/realms/demo"
}
```

---

## ⚙️ Bài tập thực hành

1. Dựng thành công Keycloak với PostgreSQL qua Docker Compose.
2. Tạo Realm, Client, User, Role theo hướng dẫn.
3. Lấy Access Token bằng Postman.
4. Decode token tại jwt.io và quan sát các claim.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                       | Giải thích                                         |
| --------------------------------------------- | -------------------------------------------------- |
| Không bật “Client Authentication” cho backend | Spring Boot cần secret để xác thực client          |
| Không thêm `Direct Access Grants Enabled`     | Không thể dùng password grant để login             |
| Dùng sai realm hoặc client_id                 | Lỗi “invalid_client” hoặc “invalid_grant”          |
| Trùng port 8080 với Spring Boot               | Đổi 1 trong 2 (ví dụ Keycloak = 8080, Boot = 8081) |
| Không copy đúng client secret                 | Token request sẽ 401                               |

---

## ✅ Best Practices

* Sử dụng **PostgreSQL** thay vì H2 để lưu realm & user an toàn.
* Tạo **realm riêng cho từng môi trường** (dev/test/prod).
* Xuất cấu hình realm ra file JSON (dễ backup).
* Dùng **Keycloak Admin REST API** để tự động tạo user/client/role.
* Cấu hình HTTPS ngay từ đầu khi deploy production.
