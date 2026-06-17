# plane-minio — MinIO (hạ tầng, bật/tắt theo env)

Lưu trữ file đính kèm (S3-compatible). Bật khi `STORAGE_MODE=internal`. Dùng S3/MinIO ngoài
thì đặt `STORAGE_MODE=external`, `USE_MINIO=0` và khai `AWS_*` trong `plane.env`.

Thông số (`MINIO_ROOT_USER/MINIO_ROOT_PASSWORD`) đọc từ `plane.env`. Volume kèm theo: `uploads`.

```yaml
  plane-minio:
    image: minio/minio:latest
    pull_policy: if_not_present
    restart: unless-stopped
    command: server /export --console-address ":9090"
    env_file:
      - plane.env
    volumes:
      - uploads:/export
```
