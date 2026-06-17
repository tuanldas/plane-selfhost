# Hướng dẫn chi tiết: Plane self-host sau Traefik external

Tài liệu này giải thích cách template hoạt động, cách dựng một instance Plane mới trên server, cách cấu hình phía Traefik, và cách xử lý sự cố. Mô hình áp dụng: **mỗi server chạy đúng một instance Plane**, TLS do **Traefik có sẵn** đảm nhiệm.

---

## 1. Kiến trúc tổng quan

Plane (Community Edition) chạy 13 container. Toàn bộ lưu lượng đi vào qua **một** service tên `proxy` — đây là một **Caddy** đóng vai trò reverse proxy nội bộ, định tuyến tới `web`, `api`, `space`, `admin`, `live` (WebSocket).

Các service chính:

| Service | Vai trò |
|---------|---------|
| `proxy` | Caddy — cổng vào duy nhất, lắng nghe cổng 80 trong container |
| `web` | Giao diện Next.js |
| `admin` | Trang quản trị instance |
| `space` | Trang public/deploy board |
| `live` | WebSocket cho cộng tác thời gian thực |
| `api` | Backend Django (REST API) |
| `worker` / `beat-worker` | Celery xử lý tác vụ nền / định kỳ |
| `migrator` | Chạy migrate DB lúc khởi động rồi thoát |
| `plane-db` | PostgreSQL |
| `plane-redis` | Valkey (cache + broker) |
| `plane-mq` | RabbitMQ |
| `plane-minio` | Lưu trữ file (S3-compatible) |

Khi đặt sau Traefik, ta **không** để Plane tự xử lý HTTPS. Thay vào đó:

```
Người dùng → (443, TLS) → Traefik → (network chung, HTTP cổng 80) → proxy (Caddy) → các service Plane
```

Traefik terminate TLS và chuyển tiếp HTTP thuần tới container `proxy`. Vì vậy Caddy nội bộ được cấu hình `SITE_ADDRESS=:80` (chỉ phục vụ HTTP, không tự xin chứng chỉ).

---

## 1b. Hai chế độ (toggle `USE_TRAEFIK`)

Template chọn chế độ qua biến `USE_TRAEFIK` trong `config.env`. Có thể chạy `./plane init` để được hỏi-đáp và sinh `config.env` tự động.

**`USE_TRAEFIK=true` — sau Traefik external (TLS do Traefik):**

```
Người dùng → (443, TLS) → Traefik → (network chung, HTTP 80) → proxy (Caddy) → service Plane
```

- `./plane build` dùng fragment `proxy.traefik.md` (gắn labels, nối network external, không publish cổng host) và thêm khối `networks:` external.
- `SITE_ADDRESS=:80`, `APP_PROTOCOL=https`, `WEB_URL=https://<DOMAIN>`, `MINIO_ENDPOINT_SSL=1` (ép URL upload dùng `https`, tránh lỗi mixed content). Khi `TRAEFIK_TLS=false` (vd `.localhost`) thì dùng `http` và `MINIO_ENDPOINT_SSL=0`.
- Cần: `TRAEFIK_NETWORK`, `CERT_RESOLVER`, `TRAEFIK_ENTRYPOINT` khớp với Traefik của bạn.

**`USE_TRAEFIK=false` — chạy Plane mặc định, không thêm gì:**

```
Người dùng → (80/443) → proxy (Caddy của Plane) → service Plane
```

- `./plane build` dùng fragment `proxy.standalone.md` (publish `LISTEN_HTTP_PORT`/`LISTEN_HTTPS_PORT` ra host), không gắn Traefik.
- bootstrap chỉ đặt `WEB_URL`/`CORS_ALLOWED_ORIGINS=http://<DOMAIN>` và sinh `SECRET_KEY`; mọi thứ còn lại giữ nguyên mặc định trong `plane.env`.
- Truy cập bằng `http://<DOMAIN>` (hoặc IP server). Muốn HTTPS tự động, đổi cổng, hay tinh chỉnh khác: sửa trực tiếp `plane.env` (`WEB_URL`, `SITE_ADDRESS`, `APP_PROTOCOL`, `LISTEN_HTTP_PORT`...) rồi `./plane build && docker compose up -d`.

Khi nào dùng cái nào: server đã có Traefik làm cổng vào chung cho nhiều app → bật Traefik (`true`). Server chỉ chạy mỗi Plane / muốn để Plane lo như mặc định → `false`.

## 2. Template làm gì (gộp module, không tải)

Mọi định nghĩa nằm trong `services-md/`. Mỗi file `.md` mô tả một service / cấu hình và chứa đúng một khối ```` ```yaml ```` là phần compose của nó.

Chỉ có một CLI duy nhất: `./plane`. Các lệnh:

- `./plane init` — hỏi-đáp tạo `config.env`.
- `./plane build` — đọc `config.env`, gộp fragment → `docker-compose.yml` (chuẩn Compose v2, có `name:`, biến render sẵn).
- `./plane up` — tự `init` nếu thiếu cấu hình → tạo/vá `plane.env` (WEB_URL/CORS theo DOMAIN, `MINIO_ENDPOINT_SSL` theo HTTPS/HTTP, sinh SECRET_KEY...) → `build` → bảo đảm network Traefik → `docker compose up -d`.
- `./plane backup` — sao lưu Postgres.

Khi `build`, file compose được gộp như sau:

1. Luôn gồm 8 service ứng dụng + backend: `web, space, admin, live, api, worker, beat-worker, migrator`.
2. Thêm service hạ tầng theo toggle: `DB_MODE/REDIS_MODE/MQ_MODE/STORAGE_MODE = internal` → thêm `plane-db / plane-redis / plane-mq / plane-minio` (kèm volume tương ứng). Đặt `external` → bỏ service đó, bạn khai connection trong `plane.env`.
3. Chọn biến thể `proxy`: `USE_TRAEFIK=true` → `proxy.traefik.md` (labels + nối network external, không publish cổng) và thêm khối `networks:` external; `false` → `proxy.standalone.md` (publish `LISTEN_HTTP_PORT`/`LISTEN_HTTPS_PORT` ra host).

Sau đó quản lý bằng lệnh `docker compose` bình thường ngay trong thư mục này: `docker compose ps`, `docker compose logs -f`, `docker compose down`, `docker compose up -d`...

### Thêm service mới hoặc cấu hình thêm

Tạo file `services-md/<tên>.md` với khối ```` ```yaml ```` chứa định nghĩa service (thụt 2 space, dùng được `<<: *app`), rồi khai báo điều kiện bật nó trong hàm `cmd_build` của script `plane` (thêm vào danh sách `CORE`/`INFRA` hoặc một toggle env mới). Chạy lại `./plane build` để sinh compose mới.

> Ghi chú: image tag mặc định là `${APP_RELEASE:-stable}`. Vì bộ này tự chứa (không tải file gốc), hãy đối chiếu image tag / biến với phiên bản Plane bạn muốn trước khi chạy production.

---

## 3. Các bước dựng trên server mới

### 3.1. Chuẩn bị phía Traefik (làm một lần)

Bạn đã có Traefik chạy sẵn. Cần biết 3 thông tin và bảo đảm chúng khớp với `config.env`:

- **Tên network external** Traefik dùng. Kiểm tra:
  ```bash
  docker network ls
  ```
  Nếu chưa có, ví dụ tạo:
  ```bash
  docker network create traefik-public
  ```
  và bảo đảm container Traefik được nối vào network này.

- **Tên certResolver** trong cấu hình Traefik (ví dụ `letsencrypt`). Trích đoạn `traefik.yml` mẫu:
  ```yaml
  entryPoints:
    web:
      address: ":80"
      http:
        redirections:
          entryPoint: { to: websecure, scheme: https, permanent: true }
    websecure:
      address: ":443"

  certificatesResolvers:
    letsencrypt:
      acme:
        email: ban@example.com
        storage: /letsencrypt/acme.json
        httpChallenge:
          entryPoint: web

  providers:
    docker:
      exposedByDefault: false
  ```

- **Tên entrypoint HTTPS** (thường `websecure`).

### 3.2. Trỏ DNS

Tạo bản ghi **A** cho `DOMAIN` trỏ về IP công khai của server. Let's Encrypt (HTTP-01) cần domain phân giải đúng thì mới cấp được chứng chỉ.

### 3.3. Dựng Plane

```bash
git clone <repo-này> plane && cd plane
cp config.env.example config.env
nano config.env
```

Điền `config.env`, ví dụ:

```ini
DOMAIN=plane.congty.vn
PLANE_INSTANCE=plane
USE_TRAEFIK=true
TRAEFIK_NETWORK=traefik-public
CERT_RESOLVER=letsencrypt
TRAEFIK_ENTRYPOINT=websecure
DB_MODE=internal
REDIS_MODE=internal
MQ_MODE=internal
STORAGE_MODE=internal
```

Chạy:

```bash
chmod +x plane
./plane up
```

Lần đầu sẽ kéo image và chạy migrate (vài phút). Theo dõi:

```bash
docker compose logs -f migrator
```

Khi `migrator` thoát thành công, kiểm tra:

```bash
docker compose ps
```

Mở `https://plane.congty.vn`. Lần đầu vào `https://plane.congty.vn/god-mode` để tạo tài khoản quản trị, sau đó tạo workspace và project.

---

## 4. Cấu hình labels Traefik (giải thích)

Các labels gắn trên service `proxy` (giá trị được `./plane build` render sẵn từ `config.env`):

```yaml
- "traefik.enable=true"
- "traefik.docker.network=${TRAEFIK_NETWORK}"
- "traefik.http.routers.${PLANE_INSTANCE}.rule=Host(`${DOMAIN}`)"
- "traefik.http.routers.${PLANE_INSTANCE}.entrypoints=${TRAEFIK_ENTRYPOINT}"
- "traefik.http.routers.${PLANE_INSTANCE}.tls=true"
- "traefik.http.routers.${PLANE_INSTANCE}.tls.certresolver=${CERT_RESOLVER}"
- "traefik.http.routers.${PLANE_INSTANCE}.service=${PLANE_INSTANCE}"
- "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.server.port=80"
- "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.passhostheader=true"
```

Giải thích nhanh: `traefik.docker.network` ép Traefik kết nối tới container qua đúng network chung; `loadbalancer.server.port=80` trỏ tới cổng Caddy nội bộ; `passhostheader=true` giữ Host header để link/redirect của Plane đúng domain. WebSocket (`live`) được Traefik chuyển tiếp tự động — không cần cấu hình thêm header `Upgrade/Connection`.

---

## 5. Vận hành

```bash
./plane up              # dựng / cập nhật
docker compose ps       # trạng thái
docker compose logs -f  # log tất cả service
docker compose restart  # khởi động lại
docker compose down     # dừng, GIỮ dữ liệu
./plane backup          # dump Postgres ra ./backups
```

### Sao lưu

Dữ liệu nằm ở Postgres (project/issue/user) và MinIO (file đính kèm).

```bash
./plane backup     # Postgres

# MinIO (volume) — đổi <project>_uploads cho đúng tên volume thực tế (docker volume ls)
docker run --rm -v plane_uploads:/data -v "$PWD":/backup alpine \
  tar czf /backup/backups/minio-$(date +%Y%m%d).tar.gz /data
```

### Nâng cấp Plane

Đặt tag mong muốn vào `APP_RELEASE` trong `plane.env` (vd `APP_RELEASE=v1.2.3` hoặc `stable`), rồi:

```bash
./plane build           # render lại image tag mới vào compose
docker compose pull     # kéo image mới
docker compose up -d    # áp dụng
```

`plane.env` và dữ liệu được giữ nguyên, nên cấu hình và secret không mất.

---

## 6. Xử lý sự cố

**`no configuration file provided: not found` khi chạy `docker compose ...`.** Chưa có `docker-compose.yml` trong thư mục (chưa build). Chạy `./plane build` (hoặc `./plane up`) để sinh file, rồi chạy lại lệnh `docker compose`.

**Sai/thiếu service sau khi build.** Kiểm tra toggle trong `config.env` (`DB_MODE`, `REDIS_MODE`...), chạy `./plane build` rồi xem `docker-compose.yml`. Mỗi service đến từ một fragment trong `services-md/`.

**Image tag không khớp phiên bản mong muốn.** Bộ này tự chứa (không tải file gốc của Plane), nên image dùng `APP_RELEASE` trong `plane.env`. Đặt đúng tag rồi `./plane build && docker compose up -d`.

**Truy cập domain báo 404 từ Traefik.** Traefik chưa thấy container. Kiểm tra: `proxy` đã nối đúng `TRAEFIK_NETWORK` chưa (`docker inspect`), Traefik có bật docker provider không, tên `CERT_RESOLVER`/`TRAEFIK_ENTRYPOINT` có khớp cấu hình Traefik không. Xem `docker compose config` để in compose đã merge.

**Không lấy được chứng chỉ HTTPS.** DNS chưa trỏ đúng IP, hoặc cổng 80 (HTTP-01) chưa mở tới Traefik. Xem log Traefik.

**WebSocket lỗi / không cập nhật thời gian thực.** Thường do `WEB_URL` không khớp domain thật. Bảo đảm `WEB_URL=https://<DOMAIN>` và `CORS_ALLOWED_ORIGINS` cùng giá trị, rồi `docker compose restart`.

**Tạo project / upload ảnh cover báo "Failed to upload" (hoặc upload file đính kèm lỗi).** Trang chạy HTTPS nhưng Plane sinh URL upload `http://` → browser chặn *Mixed Content*. Mở DevTools › Network sẽ thấy `POST http://<DOMAIN>/uploads` bị chặn. Nguyên nhân: TLS terminate ở tầng trên (Traefik/Cloudflare) nên `api` thấy request là HTTP, mà mặc định Plane lấy scheme của upload URL theo request. Khắc phục: đặt `MINIO_ENDPOINT_SSL=1` trong `plane.env` rồi `docker compose up -d` (ép presigned URL dùng `https`). CLI `./plane` tự đặt `=1` khi `USE_TRAEFIK=true` + `TRAEFIK_TLS=true`; nếu TLS do Cloudflare/upstream lo (vd standalone sau Cloudflare) thì đặt tay `=1`.

**`migrator` lặp lại hoặc lỗi.** Postgres chưa sẵn sàng. Xem `docker compose logs plane-db`, đợi DB lên rồi `docker compose restart migrator`.

**Hết RAM (container bị Killed/OOM).** Plane cần >= 4GB. Thêm swap hoặc nâng RAM; giảm `GUNICORN_WORKERS` về `1` trong `plane.env`.

---

## 7. Nguồn tham khảo

- Plane — Docker Compose (community): https://developers.plane.so/self-hosting/methods/docker-compose
- Plane — External reverse proxy (mẫu Traefik): https://developers.plane.so/self-hosting/govern/reverse-proxy
- Plane — Environment variables: https://developers.plane.so/self-hosting/govern/environment-variables
