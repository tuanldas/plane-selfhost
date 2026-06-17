# Plane self-host + Traefik (template)

Khuôn Docker để dựng nhanh một instance [Plane](https://plane.so) (công cụ quản lý dự án, thay thế Jira/Linear) trên **server mới**. Mô hình: **1 server = 1 Plane**. Có toggle `USE_TRAEFIK` để chọn theo từng server:

- `USE_TRAEFIK=true` — chạy **sau Traefik external** có sẵn (Traefik lo TLS, gắn labels).
- `USE_TRAEFIK=false` — **standalone**: Caddy bên trong Plane tự phục vụ (tự xin Let's Encrypt cho domain, hoặc HTTP thuần cho test/LAN). Không cần Traefik.

Không tải gì từ mạng. Mỗi service / cấu hình là một **fragment `.md` nhỏ** trong `services-md/` (bên trong có khối YAML). Một CLI duy nhất `./plane` đọc `config.env` rồi **gộp** các phần cần thiết thành một `docker-compose.yml` chuẩn Compose v2 (có `name:`, biến đã render sẵn) — sau đó bạn dùng `docker compose up/down/logs` **trần**, không cần cờ `-f`/`-p`.

## Cần có sẵn

- Server Linux, Docker Engine 24+ và Docker Compose v2.
- RAM tối thiểu 4GB (khuyến nghị 8GB — Plane chạy tới 13 service).
- DNS: bản ghi A của domain trỏ về IP server.
- (Chỉ khi `USE_TRAEFIK=true`) Traefik đang chạy với một **network external** và một **certResolver** (Let's Encrypt).

## Dùng nhanh

```bash
git clone <repo-này> plane && cd plane
chmod +x plane
./plane up        # hỏi-đáp cấu hình (lần đầu) → gộp compose → khởi động
```

Theo dõi tới khi `migrator` chạy xong rồi mở `https://<DOMAIN>`:

```bash
docker compose logs -f migrator
```

Lần đầu vào `https://<DOMAIN>/god-mode` để tạo tài khoản quản trị instance.

## CLI `plane`

```bash
./plane up          # dựng / cập nhật (tự hỏi cấu hình nếu thiếu)
./plane init        # chỉ tạo lại config.env (hỏi-đáp)
./plane build       # render lại docker-compose.yml sau khi sửa services-md/ hoặc config.env
./plane backup      # sao lưu Postgres ra ./backups
sudo ./plane swap 4G  # (server Linux) tạo swapfile 4G — hữu ích khi RAM thấp
```

Mọi thao tác khác dùng `docker compose` **trần** ngay trong thư mục (nhờ `name:` trong file):

```bash
docker compose ps
docker compose logs -f
docker compose down
docker compose up -d
docker compose restart
```

## File trong template

| File | Vai trò |
|------|---------|
| `plane` | CLI duy nhất: `init` / `up` / `build` / `backup`. |
| `services-md/*.md` | Mỗi service / cấu hình một fragment nhỏ (có khối YAML). Nguồn để gộp compose. |
| `config.env(.example)` | Cấu hình: `USE_TRAEFIK`, network/cert, toggle hạ tầng (`DB_MODE`...). |
| `plane.env(.example)` | Biến runtime của Plane (DB/Redis/MQ/MinIO, SECRET_KEY...). |
| `docker-compose.yml` | File tự sinh bởi `./plane build` (đừng sửa tay). |
| `docs/HUONG-DAN.md` | Hướng dẫn chi tiết + giải thích cấu hình + xử lý sự cố. |

Nâng cấp Plane: đặt tag vào `APP_RELEASE` trong `plane.env`, rồi `./plane build && docker compose up -d`.

Chi tiết từng bước và cấu hình Traefik: xem [`docs/HUONG-DAN.md`](docs/HUONG-DAN.md).
