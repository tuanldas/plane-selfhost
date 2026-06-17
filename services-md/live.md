# live — WebSocket cộng tác thời gian thực

Đồng bộ thay đổi real-time giữa người dùng (editor, board). Luôn bật.

```yaml
  live:
    <<: *app
    image: makeplane/plane-live:${APP_RELEASE:-stable}
    depends_on:
      - api
      - web
```
