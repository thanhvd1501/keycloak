# 📚 PHẦN 6: AUTHORIZATION NÂNG CAO & RESOURCE PERMISSIONS

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Hiểu **Authorization Services** trong Keycloak hoạt động ra sao.
2. Biết tạo **Resource**, **Scope**, **Policy**, **Permission** trong Admin Console.
3. Biết cách **bảo vệ tài nguyên cụ thể (resource-based)** thay vì chỉ role-based.
4. Biết **Keycloak Policy Enforcer** hoạt động trong Spring Boot.
5. Viết ví dụ **API chỉ cho phép chủ sở hữu (owner)** truy cập tài nguyên của họ.

---

## 🧩 1. Authorization Services là gì?

> Là hệ thống đánh giá chính sách (policy evaluation engine) được Keycloak tích hợp sẵn, cho phép bạn:
>
> * Xác định **tài nguyên (Resource)** — ví dụ: `/orders/123`
> * Xác định **scope (hành động)** — `view`, `edit`, `delete`
> * Định nghĩa **policy** — “Chỉ cho phép chủ sở hữu hoặc role ADMIN”
> * Gộp lại thành **permission** — gắn resource + policy để ra quyết định cuối cùng

---

## ⚙️ 2. Kiến trúc tổng thể

```text
     ┌──────────────────────────┐
     │      Keycloak Server     │
     │──────────────────────────│
     │ Realm: demo              │
     │   ├─ Resources (orders)  │
     │   ├─ Scopes (view, edit) │
     │   ├─ Policies (Owner, Admin) │
     │   └─ Permissions (order_access)│
     └──────────────────────────┘
                  ▲
                  │
                  │
           Enforcer Library
                  │
                  ▼
     ┌──────────────────────────┐
     │   Spring Boot Resource   │
     │    Server (API)          │
     │──────────────────────────│
     │ SecurityConfig + adapter │
     │  verifies token & checks │
     │  permissions with Keycloak│
     └──────────────────────────┘
```

---

## 🧠 3. Bước 1 – Kích hoạt Authorization cho Client

1. Trong **Clients → spring-client → Settings**, bật:

   * ✅ **Authorization Enabled**
   * ✅ **Service Accounts Enabled**
   * Nhấn **Save**.

2. Sang tab **Authorization** (mới xuất hiện): bạn sẽ thấy các menu:

   * **Resources**
   * **Scopes**
   * **Policies**
   * **Permissions**

---

## 🧩 4. Bước 2 – Tạo Resource (tài nguyên cần bảo vệ)

**Ví dụ:** bạn muốn bảo vệ các đơn hàng (`/orders/{id}`)

1. Vào **Authorization → Resources → Create**

   * Name: `Order Resource`
   * URI: `/api/orders/*`
   * Type: `order`
   * Add Scopes: `view`, `edit`, `delete`
   * Save.

---

## 🔧 5. Bước 3 – Tạo Policy (chính sách truy cập)

### 🔹 a. Policy theo Role

1. Vào **Policies → Create Policy → Role-based Policy**

   * Name: `Admin Policy`
   * Realm role: `ADMIN`
   * Save.

### 🔹 b. Policy theo người sở hữu (Owner-based Policy)

1. Vào **Policies → Create Policy → User-based Policy**

   * Name: `Owner Policy`
   * Users: chọn `alice` (hoặc token claim owner).
   * Save.

(Ở thực tế, ta sẽ truyền **owner claim** trong token và dùng policy script.)

---

## 🎛 6. Bước 4 – Tạo Permission (kết hợp resource + policy)

1. Vào **Permissions → Create Permission → Resource-based**

   * Name: `Order Access`
   * Resources: `Order Resource`
   * Apply Policies: `Admin Policy`, `Owner Policy`
   * Logic: `OR`
   * Save.

→ Nghĩa là:

> “Chỉ cho phép ADMIN hoặc chủ sở hữu truy cập resource `/api/orders/*`”.

---

## 💻 7. Bước 5 – Cấu hình Enforcer trong Spring Boot

Spring Boot cần gửi token đến Keycloak để hỏi:

> “User này có quyền truy cập resource X không?”

### `application.yml`

```yaml
keycloak:
  auth-server-url: http://localhost:8080
  realm: demo
  resource: spring-client
  credentials:
    secret: <SECRET>
  policy-enforcer-config:
    enforcement-mode: ENFORCING
```

> Khi bật **policy-enforcer**, Spring Boot sẽ gửi request sang Keycloak Authorization Endpoint để kiểm tra quyền trước khi cho phép truy cập API.

---

## 🧱 8. Demo Controller Resource-based

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public String viewOrder(@PathVariable String id, Authentication auth) {
        return "Order #" + id + " viewed by " + auth.getName();
    }

    @PostMapping("/{id}")
    public String editOrder(@PathVariable String id, Authentication auth) {
        return "Order #" + id + " edited by " + auth.getName();
    }
}
```

Giờ khi bạn gọi `/api/orders/123`:

* Nếu user `alice` là owner → ✅ OK
* Nếu user `bob` là `ADMIN` → ✅ OK
* Nếu user khác → ❌ 403 Forbidden

---

## 🔍 9. Sequence Flow minh họa

```text
[1] User → API: GET /api/orders/123 (Bearer Token)
[2] API → Policy Enforcer:
      "Check permission for /api/orders/123, scope=view"
[3] Keycloak Authorization Engine:
      - Load resource "Order Resource"
      - Evaluate policies:
          a. Is user ADMIN? (false)
          b. Is user owner? (true)
[4] Keycloak → API: Permit
[5] API → Return 200 OK
```

---

## 🧠 10. Khi nào dùng Resource-based Authorization?

| Use Case                                                       | Loại Authorization nên dùng     |
| -------------------------------------------------------------- | ------------------------------- |
| Phân quyền cố định (Admin/User)                                | **RBAC**                        |
| Quyền linh hoạt theo dữ liệu (owner, status, group, region...) | **ABAC / Resource-based**       |
| Microservices (service-to-service)                             | **Client credentials + scopes** |

---

## ⚙️ Bài tập thực hành

1. Bật Authorization cho client `spring-client`.
2. Tạo Resource `/api/orders/*` với scopes `view`, `edit`.
3. Tạo hai Policy: `Admin Policy`, `Owner Policy`.
4. Tạo Permission `Order Access`.
5. Test API với:

   * Token của `alice` (owner).
   * Token của `bob` (ADMIN).
   * Token của `charlie` (không role).

Quan sát kết quả: chỉ alice hoặc bob được phép.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                           | Giải thích                         |
| ------------------------------------------------- | ---------------------------------- |
| Bật policy enforcer nhưng không có permission nào | Mọi request bị 403                 |
| URI của resource không khớp thực tế               | Keycloak không match được endpoint |
| Chưa bật `Authorization Enabled`                  | Tab Authorization không xuất hiện  |
| Không truyền đúng scope khi gọi API               | Keycloak không tìm thấy permission |
| Dùng enforcement-mode = PERMISSIVE khi muốn chặn  | Nó sẽ “bỏ qua” kiểm tra policy     |

---

## ✅ Best Practices

* **RBAC** cho quyền tĩnh, **ABAC** cho quyền động.
* Luôn đặt tên rõ ràng cho resource và policy (`order:view`, `order:edit`).
* Dùng **Script Policy** khi logic phức tạp (ví dụ chỉ cho phép owner trong 24h).
* Cấu hình **Decision Strategy = AFFIRMATIVE** (chỉ cần 1 policy pass).
* Dùng **UMA (User-Managed Access)** nếu bạn cần user tự chia sẻ tài nguyên.
