# plane-mq — RabbitMQ (hạ tầng, bật/tắt theo env)

Hàng đợi phân phối job nền. Bật khi `MQ_MODE=internal`. Dùng RabbitMQ ngoài thì đặt
`MQ_MODE=external` và khai `AMQP_URL` trong `plane.env`.

Thông số (`RABBITMQ_DEFAULT_USER/PASS/VHOST`) đọc từ `plane.env`. Volume kèm theo: `rabbitmq_data`.

```yaml
  plane-mq:
    image: rabbitmq:3.13.6-management-alpine
    pull_policy: if_not_present
    restart: unless-stopped
    env_file:
      - plane.env
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 15s
      timeout: 10s
      retries: 10
```
