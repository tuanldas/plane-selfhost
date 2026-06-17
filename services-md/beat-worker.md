# beat-worker — Celery beat (tác vụ định kỳ)

Bộ lập lịch cho các tác vụ chạy theo chu kỳ. Luôn bật.

```yaml
  beat-worker:
    <<: *app
    image: makeplane/plane-backend:${APP_RELEASE:-stable}
    command: ./bin/docker-entrypoint-beat.sh
    depends_on:
      - api
```
