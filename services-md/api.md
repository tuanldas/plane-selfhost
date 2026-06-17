# api — Backend Django REST API

Lõi nghiệp vụ của Plane. Mọi cấu hình kết nối (DB, Redis, RabbitMQ, MinIO) đọc từ `plane.env`. Luôn được bật.

```yaml
  api:
    <<: *app
    image: makeplane/plane-backend:${APP_RELEASE:-stable}
    command: ./bin/docker-entrypoint-api.sh
```
