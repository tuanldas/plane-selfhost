# web — Giao diện chính (Next.js)

UI người dùng tương tác hằng ngày. Luôn bật.

```yaml
  web:
    <<: *app
    image: makeplane/plane-frontend:${APP_RELEASE:-stable}
    depends_on:
      - api
```
