# 📖 PHẦN 9: TRIỂN KHAI KEYCLOAK TRONG MÔI TRƯỜNG PRODUCTION (HTTPS, CLUSTER, BACKUP, SCALING)

---

### 🎯 **Mục tiêu học**

Sau phần này, bạn sẽ:

1. Biết cách triển khai Keycloak với **Docker Compose + PostgreSQL + Nginx (HTTPS)**.
2. Biết cấu hình **reverse proxy** và **load balancing**.
3. Hiểu cơ chế **cluster / high-availability** của Keycloak.
4. Biết cách **backup & restore realm** và cơ sở dữ liệu.
5. Biết các **tùy chỉnh bảo mật** & **hardening best practices**.

---

## 🧩 1. Kiến trúc triển khai production

```text
                        ┌──────────────────────────┐
                        │         Internet         │
                        └──────────────┬───────────┘
                                       │  HTTPS (443)
                          ┌────────────┴────────────┐
                          │      Nginx Reverse Proxy│
                          │    (SSL termination)    │
                          └───────┬────────┬────────┘
                                  │        │
                 ┌────────────────┘        └─────────────────┐
                 │                                           │
         ┌───────────────┐                         ┌───────────────┐
         │ Keycloak Node1│                         │ Keycloak Node2│
         │ :8080          │     Shared PostgreSQL   │ :8080          │
         └───────────────┘──────────┬───────────────┘
                                    │
                             ┌────────────┐
                             │ PostgreSQL │
                             └────────────┘
```

---

## ⚙️ 2. Cấu hình Docker Compose (HA ready)

Tạo file `docker-compose.prod.yml`:

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: strongpassword
    volumes:
      - keycloak_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  keycloak1:
    image: quay.io/keycloak/keycloak:24.0
    command: start --hostname-strict=false
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: strongpassword
      KC_PROXY: edge
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    depends_on:
      - postgres
    ports:
      - "8081:8080"

  keycloak2:
    image: quay.io/keycloak/keycloak:24.0
    command: start --hostname-strict=false
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: strongpassword
      KC_PROXY: edge
    depends_on:
      - postgres
    ports:
      - "8082:8080"

  nginx:
    image: nginx:1.25
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/ssl/certs
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - keycloak1
      - keycloak2

volumes:
  keycloak_data:
```

---

## 🌐 3. File cấu hình `nginx.conf` (reverse proxy + HTTPS)

```nginx
events {}

http {
  upstream keycloak_cluster {
    server keycloak1:8080;
    server keycloak2:8080;
  }

  server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/certs/privkey.pem;

    location / {
      proxy_pass http://keycloak_cluster;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto https;
    }
  }
}
```

> ✅ Nginx chịu trách nhiệm SSL termination
> ✅ Keycloak chỉ phục vụ HTTP nội bộ

---

## 🔒 4. Bật HTTPS trong Keycloak

Nếu muốn Keycloak tự phục vụ HTTPS (không qua proxy), thêm:

```bash
--https-certificate-file=/opt/keycloak/conf/server.crt \
--https-certificate-key-file=/opt/keycloak/conf/server.key
```

Hoặc set biến môi trường:

```yaml
KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/conf/server.crt
KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/conf/server.key
```

> ⚠️ Production nên dùng Nginx / Traefik / HAProxy làm reverse proxy thay vì bật HTTPS trực tiếp trong Keycloak container.

---

## 🧠 5. High Availability & Clustering

Từ Keycloak 17+ (quarkus-based):

* Không cần Infinispan UDP multicast nữa.
* Dựa vào **shared database (PostgreSQL)** để lưu session.
* Session clustering hoạt động tự động nếu các node dùng chung DB + `--hostname-strict=false`.

**Check:**

```bash
docker exec -it keycloak1 /opt/keycloak/bin/kc.sh show-config
```

Bạn sẽ thấy cluster info khi các node cùng realm và DB.

---

## 🧩 6. Backup & Restore Realm

### 🔹 Backup Realm (JSON export)

```bash
docker exec -it keycloak1 /opt/keycloak/bin/kc.sh export \
  --dir /opt/keycloak/data/export \
  --realm demo
```

→ File được tạo tại `/opt/keycloak/data/export/demo-realm.json`

### 🔹 Restore Realm

```bash
docker exec -it keycloak1 /opt/keycloak/bin/kc.sh import \
  --dir /opt/keycloak/data/export
```

💡 Dùng để deploy nhanh các realm qua môi trường dev → staging → prod.

---

## 💾 7. Backup Database (PostgreSQL)

```bash
docker exec -it postgres pg_dump -U keycloak keycloak > backup.sql
```

Restore:

```bash
cat backup.sql | docker exec -i postgres psql -U keycloak keycloak
```

---

## ⚙️ 8. Performance tuning checklist

| Thành phần        | Tối ưu đề xuất                                                    |
| ----------------- | ----------------------------------------------------------------- |
| **Database**      | Tăng `shared_buffers`, bật `pg_stat_statements`                   |
| **Cache**         | Sử dụng Infinispan + memory caching mặc định                      |
| **Tokens**        | Giảm Access Token Lifespan (~5–10 phút)                           |
| **Sessions**      | Bật “Offline Session Timeout” hợp lý                              |
| **Reverse Proxy** | Dùng HTTP/2 + gzip compression                                    |
| **Logs**          | Bật JSON structured logs để dễ phân tích                          |
| **Cluster**       | Healthcheck qua `/realms/master/.well-known/openid-configuration` |

---

## 🛡️ 9. Hardening & Security Best Practices

| Biện pháp                                  | Mô tả                                                 |
| ------------------------------------------ | ----------------------------------------------------- |
| ✅ **Dùng HTTPS toàn hệ thống**             | Không bao giờ để HTTP public                          |
| ✅ **Ẩn admin endpoint**                    | Dùng `KC_HTTP_RELATIVE_PATH=/auth`                    |
| ✅ **Giới hạn đăng nhập**                   | Bật “Brute Force Detection”                           |
| ✅ **Bảo vệ token**                         | Dùng short-lived access token, refresh token rotation |
| ✅ **Audit logs**                           | Dùng event listener ghi log vào ELK / Grafana Loki    |
| ✅ **Bảo vệ realm master**                  | Không để user thường truy cập                         |
| ✅ **Tách client cho mỗi app**              | Giảm rủi ro secret leak                               |
| ✅ **Bật CORS và CSP headers**              | Bảo vệ frontend khỏi XSS                              |
| ✅ **Quản lý secret qua Vault/K8s Secrets** | Không hardcode secret                                 |

---

## 🧰 10. Deployment options

| Môi trường             | Giải pháp                                             |
| ---------------------- | ----------------------------------------------------- |
| 🐳 **Docker Compose**  | Dev / staging (1-2 node)                              |
| ☸️ **Kubernetes**      | Production (autoscaling, healthcheck, rolling update) |
| 🧱 **VM / Bare-metal** | On-premise, cần HAProxy / Nginx load balancing        |
| ☁️ **Cloud service**   | Keycloak Operator hoặc RH-SSO trên OpenShift          |

---

## 🧩 11. Sơ đồ triển khai Production Architecture (Text-based)

```text
┌──────────────────────────┐
│        User (Browser)    │
└──────────────┬───────────┘
               │ HTTPS
               ▼
┌──────────────────────────┐
│       Nginx Proxy        │  ← SSL termination, load balancing
└──────────────┬───────────┘
        ┌──────┴──────────┐
        ▼                 ▼
┌──────────────┐   ┌──────────────┐
│ Keycloak #1  │   │ Keycloak #2  │  ← Shared PostgreSQL DB
└──────┬───────┘   └──────┬───────┘
       │ Shared realm data │
       ▼                   ▼
┌──────────────────────────┐
│      PostgreSQL Server   │
└──────────────────────────┘
```

---

## ⚙️ Bài tập thực hành

1. Dựng hệ thống Keycloak HA 2 node + PostgreSQL + Nginx bằng Docker Compose.
2. Tạo chứng chỉ tự ký (self-signed) bằng OpenSSL.
3. Cấu hình HTTPS với Nginx.
4. Test truy cập qua `https://localhost` → Admin Console.
5. Export realm → restore sang máy khác.

---

## ⚠️ Sai lầm phổ biến

| Sai lầm                          | Giải thích                           |
| -------------------------------- | ------------------------------------ |
| Dùng H2 DB trong production      | Không hỗ trợ HA hoặc backup          |
| Không có reverse proxy           | Gây lỗi redirect, CORS, HTTPS        |
| Token lifetime quá dài           | Tăng rủi ro leak token               |
| Không bật brute force protection | Dễ bị tấn công password spraying     |
| Không backup realm               | Mất toàn bộ config nếu container mất |

---

## ✅ Best Practices Summary

* Dùng **PostgreSQL hoặc MySQL** làm database backend.
* Luôn deploy **ít nhất 2 Keycloak nodes** trong production.
* Reverse proxy qua **Nginx / Traefik** (HTTPS termination).
* Backup realm định kỳ (cron job + S3 / GCS).
* Dùng **Vault** hoặc **Kubernetes Secret** cho client secret.
* Theo dõi metrics qua **Prometheus / Grafana**.
* Dùng **Keycloak Operator** cho Kubernetes scaling.
