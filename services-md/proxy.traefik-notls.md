# proxy (traefik, KHÔNG TLS) — route HTTP qua Traefik

Dùng khi `USE_TRAEFIK=true` và `TRAEFIK_TLS=false` — hợp cho domain `.localhost` / test,
hoặc khi TLS được xử lý ở tầng khác. Route qua entrypoint HTTP của Traefik, không xin cert.

`./plane build` chọn fragment này khi `TRAEFIK_TLS=false`. Đặt `TRAEFIK_ENTRYPOINT=web`
(tên entrypoint HTTP trong Traefik của bạn).

```yaml
  proxy:
    <<: *app
    image: makeplane/plane-proxy:${APP_RELEASE:-stable}
    networks:
      - default
      - traefik-net
    depends_on:
      - web
      - api
      - space
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=${TRAEFIK_NETWORK}"
      - "traefik.http.routers.${PLANE_INSTANCE}.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.${PLANE_INSTANCE}.entrypoints=${TRAEFIK_ENTRYPOINT}"
      - "traefik.http.routers.${PLANE_INSTANCE}.service=${PLANE_INSTANCE}"
      - "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.server.port=80"
      - "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.passhostheader=true"
```
