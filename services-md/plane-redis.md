# plane-redis — Valkey (hạ tầng, bật/tắt theo env)

Cache + message broker cho Celery. Bật khi `REDIS_MODE=internal`. Dùng Redis/Valkey ngoài
thì đặt `REDIS_MODE=external` và khai `REDIS_URL` trong `plane.env`.

Volume kèm theo: `redisdata`.

```yaml
  plane-redis:
    image: valkey/valkey:7.2.5-alpine
    pull_policy: if_not_present
    restart: unless-stopped
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10
```
