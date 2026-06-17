# migrator — Chạy migrate DB rồi thoát

Chạy một lần lúc khởi động để tạo/cập nhật schema, sau đó tự thoát (không restart vô hạn). Luôn bật.

```yaml
  migrator:
    <<: *app
    image: makeplane/plane-backend:${APP_RELEASE:-stable}
    command: ./bin/docker-entrypoint-migrator.sh
    restart: "no"
```
