# proxy (traefik) — Caddy sau Traefik external

Cổng vào khi dùng Traefik (`USE_TRAEFIK=true`). KHÔNG publish cổng host; thay vào đó nối
vào network external của Traefik và gắn labels. Traefik lo TLS (Let's Encrypt).

build.sh chọn fragment này khi `USE_TRAEFIK=true`, và tự thêm khối `networks:` external ở cuối.

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
      - "traefik.http.routers.${PLANE_INSTANCE}.tls=true"
      - "traefik.http.routers.${PLANE_INSTANCE}.tls.certresolver=${CERT_RESOLVER}"
      - "traefik.http.routers.${PLANE_INSTANCE}.service=${PLANE_INSTANCE}"
      - "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.server.port=80"
      - "traefik.http.services.${PLANE_INSTANCE}.loadbalancer.passhostheader=true"
```
