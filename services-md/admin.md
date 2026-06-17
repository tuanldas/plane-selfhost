# admin — Trang quản trị instance (god-mode)

Quản trị cấp instance: cấu hình xác thực, email, AI... truy cập tại `/god-mode`. Luôn bật.

```yaml
  admin:
    <<: *app
    image: makeplane/plane-admin:${APP_RELEASE:-stable}
    depends_on:
      - api
      - web
```
