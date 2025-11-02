# 📙 PHẦN 3: KẾT NỐI SPRING BOOT VỚI KEYCLOAK (OIDC Integration)

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Kết nối ứng dụng Spring Boot 3.x với Keycloak qua **OpenID Connect / OAuth2**.
2. Hiểu rõ cách **Spring Security 6 xác thực JWT** từ Keycloak.
3. Viết REST API có bảo vệ token (Bearer Token).
4. Thực hành login & test API bằng Postman.
5. Hiểu **cách ánh xạ role trong JWT → GrantedAuthority** trong Spring.

---

## 🧩 1. Tổng quan kiến trúc luồng kết nối

Hệ thống lúc này gồm:

```text
+---------------------+
|    Keycloak Server  |
| (issuer: http://localhost:8080/realms/demo)
|---------------------|
|  ✓ Auth (login)     |
|  ✓ Token Service    |
|  ✓ User/Role Mgmt   |
+---------------------+
           ↑
           |  (JWT Access Token)
           ↓
+----------------------+
|  Spring Boot API     |
|----------------------|
| SecurityConfig.java  |
| application.yml       |
| JwtAuthConverter.java |
+----------------------+
           ↑
           | (Bearer Token)
           ↓
+----------------------+
|  Postman / Frontend  |
|  (Access Token gửi qua Header)
+----------------------+
```

---

## 🧱 2. Cấu trúc project Spring Boot

Tạo project Spring Boot 3.x (Java 17) qua Spring Initializr với các dependencies:

* `spring-boot-starter-web`
* `spring-boot-starter-security`
* `spring-boot-starter-oauth2-resource-server`
* `spring-boot-starter-oauth2-client` *(nếu có frontend login)*
* `spring-boot-starter-validation`

**Cấu trúc thư mục:**

```
src
 └── main
     ├── java/com/example/keycloakdemo
     │    ├── config/
     │    │    ├── SecurityConfig.java
     │    │    └── JwtAuthConverter.java
     │    └── controller/
     │         └── DemoController.java
     └── resources/
          └── application.yml
```

---

## ⚙️ 3. Cấu hình `application.yml`

```yaml
server:
  port: 8081

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/demo
```

> 🔍 `issuer-uri` sẽ được Spring Security dùng để:
>
> * Lấy public key (JWK Set) từ Keycloak.
> * Kiểm tra chữ ký JWT.
> * Xác minh các claim (`exp`, `iss`, `aud`).

---

## 🔒 4. Cấu hình Security (Spring Security 6)

Tạo file `SecurityConfig.java`:

```java
package com.example.keycloakdemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // Tắt CSRF để test API
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasRole("USER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt()); // bật xác thực JWT
        return http.build();
    }
}
```

---

## 🧠 5. Giải thích cơ chế xác thực JWT trong Spring Boot

1. Khi Postman gửi `Authorization: Bearer <access_token>`
2. Spring Security kiểm tra token có hợp lệ không:

   * Xác minh chữ ký (RSA256) bằng public key từ Keycloak.
   * Kiểm tra `issuer` và `audience`.
3. Nếu hợp lệ → Token được chuyển thành **Authentication** trong SecurityContext.
4. Các `roles` trong JWT sẽ được map thành `GrantedAuthority` → quyết định quyền truy cập.

---

## ⚙️ 6. Mapping Role (realm roles → authorities)

Keycloak thường lưu role trong token như sau:

```json
"realm_access": {
  "roles": ["USER", "ADMIN"]
}
```

→ ta cần viết converter để ánh xạ `roles` này thành authorities của Spring Security.

Tạo file `JwtAuthConverter.java`:

```java
package com.example.keycloakdemo.config;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;

@Component
public class JwtAuthConverter implements Converter<Jwt, JwtAuthenticationToken> {

    @Override
    public JwtAuthenticationToken convert(Jwt jwt) {
        Collection<GrantedAuthority> authorities = extractAuthorities(jwt);
        return new JwtAuthenticationToken(jwt, authorities, getPrincipalName(jwt));
    }

    private Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
        var realmAccess = (Object) jwt.getClaim("realm_access");
        if (realmAccess == null) return Collections.emptyList();

        List<String> roles = ((Map<String, List<String>>) realmAccess).get("roles");
        if (roles == null) return Collections.emptyList();

        return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
    }

    private String getPrincipalName(Jwt jwt) {
        return jwt.getClaimAsString("preferred_username");
    }
}
```

Và sửa `SecurityConfig.java` để dùng converter này:

```java
.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt.jwtAuthenticationConverter(new JwtAuthConverter()))
);
```

---

## 💻 7. Viết Controller test quyền truy cập

Tạo file `DemoController.java`:

```java
package com.example.keycloakdemo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {

    @GetMapping("/api/public/hello")
    public String publicHello() {
        return "Public API: no token required.";
    }

    @GetMapping("/api/user/hello")
    public String userHello() {
        return "Hello USER! You have valid Keycloak token.";
    }

    @GetMapping("/api/admin/hello")
    public String adminHello() {
        return "Hello ADMIN! You have admin privileges.";
    }
}
```

---

## 🔍 8. Test bằng Postman

1. Lấy **Access Token** từ Keycloak như đã làm ở Phần 2.
2. Thêm Header vào request:

```
Authorization: Bearer <access_token>
```

3. Test các endpoint:

| Endpoint                | Yêu cầu role | Kết quả                               |
| ----------------------- | ------------ | ------------------------------------- |
| `GET /api/public/hello` | none         | ✅ 200 OK                              |
| `GET /api/user/hello`   | USER         | ✅ 200 OK                              |
| `GET /api/admin/hello`  | ADMIN        | ❌ 403 Forbidden nếu token chỉ có USER |

---

## 🧠 9. Sơ đồ text-based mô tả luồng xác thực

```text
[1] Postman gửi request:
    GET /api/user/hello
    Authorization: Bearer eyJhbGciOi...

[2] Spring Security:
    - Parse JWT
    - Verify Signature (RSA)
    - Check issuer = http://localhost:8080/realms/demo
    - Extract roles from realm_access.roles

[3] Nếu hợp lệ → GrantedAuthority("ROLE_USER")

[4] Controller /api/user/hello được phép truy cập
```

---

## ⚙️ Bài tập thực hành

1. Tạo project Spring Boot 3.x.
2. Cấu hình `application.yml`, `SecurityConfig.java`, `JwtAuthConverter.java`.
3. Viết 3 API: public / user / admin.
4. Lấy token từ Keycloak và test bằng Postman.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                      | Giải thích                             |
| -------------------------------------------- | -------------------------------------- |
| Không bật “Client Authentication” ở Keycloak | Token không thể lấy được               |
| Thiếu `issuer-uri` trong `application.yml`   | Spring không biết xác minh token ở đâu |
| JWT không có `realm_access.roles`            | Spring không map được quyền            |
| Dùng sai prefix role (`ROLE_`)               | Spring không nhận diện được            |
| Token hết hạn (exp)                          | Luôn lấy token mới trước khi test      |

---

## ✅ Best Practices

* Dùng **custom JwtAuthConverter** để linh hoạt ánh xạ role.
* Nếu nhiều client (microservice) → nên dùng **client roles** thay vì realm role.
* Thêm **actuator/health** làm endpoint public.
* Dùng `spring.security.oauth2.resourceserver.jwt.audiences` để giới hạn audience.
* Luôn log claim quan trọng khi debug.
