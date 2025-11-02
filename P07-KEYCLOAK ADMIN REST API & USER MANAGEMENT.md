# 📓 PHẦN 7: KEYCLOAK ADMIN REST API & USER MANAGEMENT

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Hiểu cơ chế hoạt động của **Keycloak Admin REST API**.
2. Lấy được **Admin Access Token** để gọi các API quản trị.
3. Gọi được các API:

   * Tạo, tìm kiếm, xoá **user**
   * Đặt lại **password**
   * Gán **role / group**
   * Lấy danh sách **clients, realms**
4. Sử dụng **Postman hoặc cURL** để test.
5. Biết cách tích hợp với **Spring Boot** để tự động hoá quản trị user / client.

---

## 🧠 1. Tổng quan Keycloak Admin REST API

Keycloak cung cấp một REST API cho phép bạn thao tác toàn bộ hệ thống, tương đương như Admin Console.

| Thành phần            | Endpoint chính                                  | Mô tả                          |
| --------------------- | ----------------------------------------------- | ------------------------------ |
| **Realms**            | `/admin/realms`                                 | Tạo / xoá / config realm       |
| **Users**             | `/admin/realms/{realm}/users`                   | CRUD user                      |
| **Roles**             | `/admin/realms/{realm}/roles`                   | Quản lý role                   |
| **Clients**           | `/admin/realms/{realm}/clients`                 | CRUD client                    |
| **Groups**            | `/admin/realms/{realm}/groups`                  | Quản lý nhóm                   |
| **Sessions / Tokens** | `/realms/{realm}/protocol/openid-connect/token` | Lấy token đăng nhập hoặc admin |

---

## 🔐 2. Lấy **Admin Access Token**

Để gọi API quản trị, bạn cần token với quyền `realm-management / manage-users`.

### Bước 1️⃣ – Gọi endpoint token

```bash
curl -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" \
  -d "grant_type=password"
```

✅ Kết quả:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 300,
  "token_type": "Bearer"
}
```

---

## 👤 3. API: Tạo User

```bash
curl -X POST http://localhost:8080/admin/realms/demo/users \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
        "username": "newuser",
        "email": "newuser@example.com",
        "enabled": true,
        "firstName": "New",
        "lastName": "User"
      }'
```

→ Trả về `201 Created` nếu thành công.

---

## 🔑 4. Đặt mật khẩu cho user

Sau khi tạo user, bạn cần set password qua endpoint credentials.

### Bước 1️⃣ – Lấy user ID:

```bash
curl -X GET http://localhost:8080/admin/realms/demo/users?username=newuser \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

→ Kết quả:

```json
[
  {
    "id": "0f1bfe63-cc7a-4e12-a8ad-f3e56a84f50a",
    "username": "newuser"
  }
]
```

### Bước 2️⃣ – Set password:

```bash
curl -X PUT http://localhost:8080/admin/realms/demo/users/0f1bfe63-cc7a-4e12-a8ad-f3e56a84f50a/reset-password \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
        "type": "password",
        "value": "newpassword123",
        "temporary": false
      }'
```

✅ User có thể login ngay.

---

## 🎭 5. Gán Role cho User

### Bước 1️⃣ – Lấy danh sách role realm:

```bash
curl -X GET http://localhost:8080/admin/realms/demo/roles \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

→ Ví dụ:

```json
[
  {"id": "a123b456", "name": "USER"},
  {"id": "b789c012", "name": "ADMIN"}
]
```

### Bước 2️⃣ – Gán role cho user:

```bash
curl -X POST http://localhost:8080/admin/realms/demo/users/0f1bfe63-cc7a-4e12-a8ad-f3e56a84f50a/role-mappings/realm \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '[{"id":"a123b456", "name":"USER"}]'
```

✅ Giờ user `newuser` có quyền `ROLE_USER`.

---

## 🧾 6. Lấy danh sách tất cả users

```bash
curl -X GET http://localhost:8080/admin/realms/demo/users \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

Bạn có thể thêm query param:

```
?first=0&max=20&search=alice
```

---

## 🧨 7. Xoá user

```bash
curl -X DELETE http://localhost:8080/admin/realms/demo/users/<USER_ID> \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

---

## 💻 8. Demo dùng Spring Boot gọi Keycloak Admin API

Ví dụ tạo user tự động qua REST template.

```java
@Service
public class KeycloakAdminService {

    private static final String BASE_URL = "http://localhost:8080/admin/realms/demo";
    private final RestTemplate restTemplate = new RestTemplate();

    public void createUser(String token, String username, String email) {
        String url = BASE_URL + "/users";
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> body = Map.of(
            "username", username,
            "email", email,
            "enabled", true
        );

        HttpEntity<Map<String, Object>> request = new HttpEntity<>(body, headers);
        restTemplate.exchange(url, HttpMethod.POST, request, Void.class);
    }
}
```

---

## 🔍 9. Sơ đồ flow: Admin API Workflow

```text
[1] Admin client → Request token (/token)
[2] Keycloak → Return admin access token
[3] Spring Boot / Postman → Call Admin API (users, roles, etc.)
[4] Keycloak Admin Endpoint → Execute operation (create user, assign role, etc.)
[5] Response → HTTP 201 / 200 / 204
```

---

## ⚙️ 10. Bài tập thực hành

1. Lấy Admin token qua Postman.
2. Tạo user mới `john`.
3. Set password và role `USER`.
4. Login bằng `john` và gọi `/api/user/hello`.
5. Xoá user và kiểm tra lại (token cũ sẽ invalid).

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                                  | Giải thích                         |
| -------------------------------------------------------- | ---------------------------------- |
| Dùng `spring-client` token thay vì admin token           | Không có quyền gọi `/admin/*`      |
| Không bật `Service Accounts Enabled` cho admin-cli       | Token không lấy được               |
| Không tìm đúng user ID (UUID)                            | API không thực thi                 |
| Dùng password grant với admin account trong realm “demo” | Chỉ `master` realm có quyền admin  |
| Không refresh admin token                                | Token hết hạn sau 5 phút → lỗi 401 |

---

## ✅ Best Practices

* Tạo **một client riêng** cho automation (vd: `system-admin-client`) thay vì dùng `admin-cli`.
* Dùng **client_credentials flow** cho automation an toàn hơn.
* Ẩn admin credentials trong secret manager (Vault, AWS Secrets...).
* Giới hạn quyền của admin client chỉ ở mức cần thiết (`manage-users`, `view-realm`).
* Ghi log chi tiết mỗi thao tác tạo/sửa/xoá user.
