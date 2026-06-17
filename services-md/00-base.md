# 00 — Base (header + anchor dùng chung)

Phần khung đầu file compose: định nghĩa anchor `&app` (env_file + restart) để các service ứng dụng tái sử dụng, tránh lặp. build.sh luôn đưa phần này lên đầu.

```yaml
# File này được build.sh sinh tự động từ services-md/*.md — KHÔNG sửa tay.
x-app: &app
  env_file:
    - plane.env
  pull_policy: if_not_present
  restart: unless-stopped
```
