# proxy (standalone) — Caddy, publish cổng ra host

Cổng vào của Plane khi KHÔNG dùng Traefik (`USE_TRAEFIK=false`). Caddy publish cổng
`LISTEN_HTTP_PORT`/`LISTEN_HTTPS_PORT` ra host. Cấu hình (SITE_ADDRESS...) đọc từ `plane.env`.

build.sh chọn fragment này khi `USE_TRAEFIK=false`.

```yaml
  proxy:
    <<: *app
    image: makeplane/plane-proxy:${APP_RELEASE:-stable}
    ports:
      - ${LISTEN_HTTP_PORT:-80}:80
      - ${LISTEN_HTTPS_PORT:-443}:443
    depends_on:
      - web
      - api
      - space
```
