# space — Trang public / deploy board

Phục vụ các project/board được publish ra ngoài. Luôn bật.

```yaml
  space:
    <<: *app
    image: makeplane/plane-space:${APP_RELEASE:-stable}
    depends_on:
      - api
      - web
```
