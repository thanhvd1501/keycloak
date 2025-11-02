# 📖 PHẦN 8: TÍCH HỢP VỚI FRONTEND (NEXTJS/REACT) & API GATEWAY

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Tích hợp **Keycloak** với **Frontend SPA (React/NextJS)** qua SDK `keycloak-js`.
2. Hiểu luồng **Authorization Code Flow + PKCE** dành cho frontend an toàn.
3. Hiểu cách frontend – backend chia sẻ token và bảo mật API call.
4. Tích hợp Keycloak với **Spring Cloud Gateway** để xác thực trung gian.
5. Hiểu cơ chế **JWT propagation** giữa các microservices.

---

## 🧩 1. Kiến trúc tổng thể hệ thống Full-stack

```text
┌──────────────────────────────────────────────────────────┐
│                       Keycloak Server                    │
│  Realm: demo                                             │
│  ├── Clients: react-client, gateway-client, service-A     │
│  ├── Users / Roles / Policies                             │
└──────────────────────────────────────────────────────────┘
               ▲                ▲                  ▲
               │                │                  │
         (OIDC Auth)       (JWT Verify)        (JWT Propagation)
               │                │                  │
   ┌───────────────┐     ┌─────────────┐     ┌─────────────┐
   │ React / NextJS │──▶ │ API Gateway │──▶ │ Microservice │
   └───────────────┘     └─────────────┘     └─────────────┘
```

---

## ⚙️ 2. Tích hợp Keycloak với React (SPA)

### Cài SDK:

```bash
npm install keycloak-js
```

### `keycloak.js` setup file:

```javascript
import Keycloak from 'keycloak-js';

const keycloak = new Keycloak({
  url: 'http://localhost:8080',
  realm: 'demo',
  clientId: 'react-client'
});

export default keycloak;
```

### Trong `App.jsx`:

```javascript
import React, { useEffect, useState } from 'react';
import keycloak from './keycloak';

function App() {
  const [authenticated, setAuthenticated] = useState(false);

  useEffect(() => {
    keycloak.init({ onLoad: 'login-required', pkceMethod: 'S256' }).then(auth => {
      setAuthenticated(auth);
      console.log('Access Token:', keycloak.token);
    });
  }, []);

  if (!authenticated) return <div>Loading...</div>;

  return (
    <div>
      <h1>Welcome, {keycloak.tokenParsed.preferred_username}</h1>
      <button onClick={() => keycloak.logout()}>Logout</button>
    </div>
  );
}

export default App;
```

### 🔑 Keycloak sẽ tự động:

* Redirect user đến trang login.
* Sau khi login, trả về **Authorization Code**, đổi lấy token.
* Lưu token (access, refresh, id) trong session storage.

---

## 🧠 3. Authorization Code Flow + PKCE (chuẩn SPA)

```text
[1] React → /auth
[2] Keycloak → login page
[3] User nhập username/password
[4] Keycloak → redirect về React: /callback?code=XYZ
[5] React → /token (Keycloak): gửi code + PKCE verifier
[6] Keycloak → trả Access Token + Refresh Token
[7] React → gọi API Backend: Authorization: Bearer <access_token>
```

💡 PKCE (Proof Key for Code Exchange) = cơ chế bảo mật thay thế client_secret
→ đảm bảo token không bị đánh cắp qua redirect URI.

---

## 💻 4. Gọi API từ React đến Spring Boot

```javascript
fetch('http://localhost:8081/api/user/hello', {
  headers: {
    'Authorization': 'Bearer ' + keycloak.token
  }
})
.then(res => res.text())
.then(console.log);
```

Backend (Spring Boot) sẽ xác thực token giống như ở Phần 3.

---

## 🧱 5. Cấu hình Client trong Keycloak cho Frontend

| Mục                   | Giá trị                          |
| --------------------- | -------------------------------- |
| Client ID             | react-client                     |
| Client Type           | Public                           |
| Standard Flow Enabled | ✅                                |
| Direct Access Grants  | ❌                                |
| Implicit Flow         | ❌                                |
| PKCE Required         | ✅                                |
| Valid Redirect URIs   | `http://localhost:3000/*`        |
| Web Origins           | `*` hoặc `http://localhost:3000` |

---

## 🌐 6. Tích hợp API Gateway (Spring Cloud Gateway)

### Mục tiêu:

Gateway sẽ xác thực JWT từ Keycloak một lần duy nhất, rồi forward request cho các microservices phía sau.

---

### `pom.xml`

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

---

### `application.yml`

```yaml
server:
  port: 8085

spring:
  cloud:
    gateway:
      routes:
        - id: service-a
          uri: http://localhost:8082
          predicates:
            - Path=/api/service-a/**
          filters:
            - StripPrefix=1
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/demo
```

Gateway sẽ:

1. Kiểm tra JWT từ Header Authorization.
2. Nếu hợp lệ → forward request đến service A.
3. Nếu không → trả 401 Unauthorized.

---

## 🔄 7. JWT Propagation giữa microservices

Các microservices cần nhận JWT từ Gateway để kiểm tra role / claim.

Ví dụ:

```java
@GetMapping("/serviceA/data")
public String serviceA(Authentication auth) {
    var username = auth.getName();
    var roles = auth.getAuthorities();
    return "Service A data for " + username + " with roles " + roles;
}
```

Spring Security tự động decode JWT và chuyển thành `Authentication`.

---

## 🔁 8. Token Exchange (khi một service gọi service khác)

Khi **Service A** cần gọi **Service B** nhân danh người dùng, bạn có thể dùng **Keycloak Token Exchange API**.

### Gọi API:

```bash
POST /realms/demo/protocol/openid-connect/token
  grant_type=urn:ietf:params:oauth:grant-type:token-exchange
  subject_token=<user_access_token>
  requested_token_type=urn:ietf:params:oauth:token-type:access_token
  audience=service-b
```

Keycloak trả về token mới dành riêng cho service-b.

💡 Dùng khi:

* Có nhiều microservice tách biệt bảo mật.
* Cần "chuyển quyền" giữa service.

---

## 🧠 9. Sơ đồ tổng thể flow

```text
[User]
   ↓ (OIDC Login)
[Frontend SPA]
   ↓ (Bearer Token)
[API Gateway]
   ↓ (JWT Verification)
[Microservice A] ──► (Token Exchange) ──► [Keycloak] ──► [Microservice B]
```

---

## ⚙️ 10. Bài tập thực hành

1. Tạo client `react-client` trong Keycloak (Public + PKCE).
2. Chạy React demo ở port 3000.
3. Login → xem token trên console.
4. Gọi API `/api/user/hello` qua Bearer Token.
5. Tạo Gateway ở port 8085 → test xác thực token.
6. Triển khai thêm Service A, Service B → thử **token propagation**.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                                         | Giải thích                                 |
| ----------------------------------------------- | ------------------------------------------ |
| Không bật PKCE                                  | Bị lỗi 400 “PKCE required”                 |
| Đặt `client_secret` cho public client           | Không được – public client không có secret |
| Quên thêm Redirect URI                          | Keycloak chặn login                        |
| Gateway không kiểm tra issuer-uri               | Có thể bị bypass token giả                 |
| Microservice không forward Authorization header | Token bị mất khi qua tầng gọi API          |

---

## ✅ Best Practices

* SPA → dùng **Authorization Code Flow + PKCE**.
* Backend → xác thực JWT qua **Resource Server**.
* Gateway → xác thực một lần duy nhất (Central Auth).
* Sử dụng **short access token**, **long refresh token**.
* Dùng **token exchange** khi cần context đa dịch vụ.
* Tách Realm theo môi trường (dev/test/prod).
