# plane-db — PostgreSQL (hạ tầng, bật/tắt theo env)

Database chính. Bật khi `DB_MODE=internal`. Nếu dùng Postgres ngoài (`DB_MODE=external`)
thì bỏ service này và đặt `DATABASE_URL` trong `plane.env` trỏ tới DB ngoài.

Thông số (`POSTGRES_USER/PASSWORD/DB`) đọc từ `plane.env`. Volume kèm theo: `pgdata`.

```yaml
  plane-db:
    image: postgres:15.7-alpine
    pull_policy: if_not_present
    restart: unless-stopped
    command: postgres -c 'max_connections=1000'
    env_file:
      - plane.env
    environment:
      PGDATA: /var/lib/postgresql/data/pdata
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 10
```
