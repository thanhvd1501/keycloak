# 📕 PHẦN 5: ROLE-BASED ACCESS CONTROL (RBAC) – REALM ROLE, CLIENT ROLE & COMPOSITE ROLE

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Hiểu **cấu trúc phân quyền trong Keycloak** và cách chúng xuất hiện trong JWT.
2. Phân biệt ba loại role: **Realm Role**, **Client Role**, **Composite Role**.
3. Gán role cho **user, group, client** đúng cách.
4. Mapping role từ token vào Spring Security (`GrantedAuthority`).
5. Viết API Spring Boot giới hạn quyền truy cập theo role.

---

## 🧭 1. Kiến trúc Role-Based Access trong Keycloak

Keycloak chia quyền thành ba cấp:

| Loại Role          | Mức độ ảnh hưởng                    | Nơi xuất hiện                        | Ví dụ                               |
| ------------------ | ----------------------------------- | ------------------------------------ | ----------------------------------- |
| **Realm Role**     | Toàn bộ realm (toàn hệ thống)       | `realm_access.roles`                 | `ADMIN`, `USER`, `MANAGER`          |
| **Client Role**    | Riêng cho từng client               | `resource_access["client-id"].roles` | `READ_PRODUCTS`, `WRITE_PRODUCTS`   |
| **Composite Role** | Role tổng hợp (chứa nhiều role con) | Cả realm hoặc client                 | `SUPER_ADMIN` = `ADMIN` + `MANAGER` |

---

## 🧩 2. Minh họa sơ đồ phân quyền

```text
REALM: demo
│
├── Realm Roles:
│     ├── USER
│     ├── ADMIN
│     └── MANAGER
│
├── Client: spring-client
│     ├── Client Roles:
│     │     ├── READ_PRODUCTS
│     │     ├── WRITE_PRODUCTS
│     │     └── DELETE_PRODUCTS
│
└── Composite Role:
      SUPER_ADMIN = ADMIN + WRITE_PRODUCTS + DELETE_PRODUCTS
```

---

## ⚙️ 3. Tạo Roles trong Keycloak

### 🔹 Tạo Realm Role:

1. Vào **Roles → Add Role**.

   * Name: `USER`
   * Description: “Basic user role”
   * Save.
2. Lặp lại với `ADMIN`, `MANAGER`.

---

### 🔹 Tạo Client Role:

1. Chọn **Clients → spring-client → Roles → Add Role**.

   * Name: `READ_PRODUCTS`, `WRITE_PRODUCTS`.
   * Save.
2. Những role này **chỉ có hiệu lực với client `spring-client`**.

---

### 🔹 Tạo Composite Role:

1. Quay lại **Roles → Add Role**

   * Name: `SUPER_ADMIN`
   * Chọn tab “Composite Roles” → Add selected → chọn `ADMIN`, `WRITE_PRODUCTS`, `DELETE_PRODUCTS`.
2. Giờ `SUPER_ADMIN` sẽ bao gồm tất cả quyền con.

---

## 👥 4. Gán Role cho User

1. Vào **Users → alice → Role Mappings**.
2. Chọn:

   * Realm Roles: `USER`.
   * Client Roles → spring-client → chọn `READ_PRODUCTS`.
3. Gán thêm cho `bob`: `ADMIN`, `WRITE_PRODUCTS`.

---

## 🔍 5. Cách Role xuất hiện trong Token JWT

Ví dụ token `access_token` (decode trên jwt.io):

```json
{
  "preferred_username": "alice",
  "realm_access": {
    "roles": ["USER"]
  },
  "resource_access": {
    "spring-client": {
      "roles": ["READ_PRODUCTS"]
    }
  }
}
```

→ Nghĩa là:

* Alice có quyền USER (toàn hệ thống)
* Alice có quyền READ_PRODUCTS (trên client spring-client)

---

## 🧠 6. Cấu hình ánh xạ Role trong Spring Boot

Spring chỉ nhận diện `GrantedAuthority` có prefix `"ROLE_"`.
Ta sẽ mở rộng converter để map cả realm và client roles.

### `JwtAuthConverter.java` (bản mở rộng)

```java
package com.example.keycloakdemo.config;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;

@Component
public class JwtAuthConverter implements Converter<Jwt, JwtAuthenticationToken> {

    @Override
    public JwtAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        authorities.addAll(extractRealmRoles(jwt));
        authorities.addAll(extractClientRoles(jwt));
        return new JwtAuthenticationToken(jwt, authorities, jwt.getClaimAsString("preferred_username"));
    }

    private Collection<GrantedAuthority> extractRealmRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess == null) return List.of();
        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
    }

    private Collection<GrantedAuthority> extractClientRoles(Jwt jwt) {
        Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
        if (resourceAccess == null) return List.of();

        List<String> roles = resourceAccess.values().stream()
                .flatMap(obj -> ((Map<String, List<String>>) obj).get("roles").stream())
                .toList();

        return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
    }
}
```

---

## 🔒 7. Cập nhật SecurityConfig

```java
http
  .authorizeHttpRequests(auth -> auth
      .requestMatchers("/api/public/**").permitAll()
      .requestMatchers("/api/admin/**").hasRole("ADMIN")
      .requestMatchers("/api/product/read").hasRole("READ_PRODUCTS")
      .requestMatchers("/api/product/write").hasRole("WRITE_PRODUCTS")
      .anyRequest().authenticated()
  )
  .oauth2ResourceServer(oauth2 -> oauth2
      .jwt(jwt -> jwt.jwtAuthenticationConverter(new JwtAuthConverter()))
  );
```

---

## 💻 8. Viết controller demo quyền hạn

```java
package com.example.keycloakdemo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductController {

    @GetMapping("/api/product/read")
    public String readProduct() {
        return "✅ READ_PRODUCTS allowed";
    }

    @GetMapping("/api/product/write")
    public String writeProduct() {
        return "✅ WRITE_PRODUCTS allowed";
    }

    @GetMapping("/api/admin/dashboard")
    public String adminPanel() {
        return "👑 ADMIN access granted";
    }
}
```

---

## 🧠 9. Minh hoạ flow kiểm tra quyền

```text
User (alice)
  ├── Realm roles: [USER]
  ├── Client roles: [READ_PRODUCTS]
  ↓
Request: GET /api/product/read
Header: Authorization: Bearer eyJhbGciOi...
↓
Spring Security:
  - parse JWT
  - authorities = ["ROLE_USER", "ROLE_READ_PRODUCTS"]
↓
Endpoint requires ROLE_READ_PRODUCTS → ✅ allowed
```

---

## ⚙️ 10. Bài tập thực hành

1. Tạo thêm user `bob` với roles: `ADMIN`, `WRITE_PRODUCTS`.
2. Dùng Postman:

   * Gọi `/api/product/read` bằng token của `alice` → ✅
   * Gọi `/api/product/write` bằng token của `alice` → ❌ 403
   * Gọi `/api/product/write` bằng token của `bob` → ✅
   * Gọi `/api/admin/dashboard` bằng token của `bob` → ✅
3. Thử composite role `SUPER_ADMIN` → gán cho `alice` và quan sát token.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                  | Giải thích                                |
| ---------------------------------------- | ----------------------------------------- |
| Quên prefix `ROLE_` khi map              | Spring không hiểu quyền                   |
| Chỉ map realm roles, bỏ qua client roles | Một nửa quyền bị mất                      |
| Gán role cho client sai                  | Role không xuất hiện trong token          |
| Token không cập nhật sau khi gán role    | Cần **logout/login lại** để token refresh |
| Không kiểm soát role duplication         | Spring có thể tạo authorities trùng       |

---

## ✅ Best Practices

* Dùng **Realm Role** cho quyền toàn cục (Admin, User).
* Dùng **Client Role** cho quyền chi tiết từng ứng dụng hoặc microservice.
* Dùng **Composite Role** để gom nhóm quyền phức tạp.
* Gán role qua **Group** nếu hệ thống nhiều người dùng.
* Audit & export role định kỳ bằng **Admin REST API**.
