# worker — Celery worker (tác vụ nền)

Xử lý job nền: gửi thông báo, import, webhook... Luôn bật.

```yaml
  worker:
    <<: *app
    image: makeplane/plane-backend:${APP_RELEASE:-stable}
    command: ./bin/docker-entrypoint-worker.sh
    depends_on:
      - api
```
