# Thông Tin Deploy — Checkpoint 5

> Service đã được triển khai thật trên Railway. File này chỉ ghi tên biến môi
> trường và nguồn cấu hình, không chứa giá trị API key.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Văn Linh |
| Mã học viên | 2A202601971 |
| Repo | https://github.com/linkoc119/DAY12-2A202601971-NguyenVanLinh |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-f0e8.up.railway.app |
| Domain phụ | https://day12-agent-production-fdc4.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Dùng

Chỉ liệt kê tên biến và nguồn giá trị; không công khai secret.

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | Đặt trong Variables của service agent, không nằm trong repo |
| `REDIS_URL` | ✅ | Reference Variable tới `day12-redis.REDIS_URL` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Kết Quả Chạy Thật

Các lệnh kiểm tra được gọi vào Public URL Railway sau khi deployment chuyển
sang trạng thái Online.

```text
/health 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
/ready 200 {"status":"ready","redis":true}
/ask without key 401 {"detail":"invalid or missing API key"}
/ask with key 200 {'user_id': 'cp5-final-check', 'history_length': 0, 'has_answer': True}
rate limit [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429, 429, 429, 429]
```

## Ảnh Chụp Màn Hình

`screenshots/health.png` ghi lại kết quả thật từ `/health` và `/ready` của Public
URL Railway. `screenshots/dashboard.png` ghi lại hai service `day12-agent` và
`day12-redis` đều Online, cùng deployment đang Active và Successful.

## Sự Cố Khi Deploy

Lần deploy đầu build thành công nhưng thất bại ở `Network > Healthcheck`. Nguyên
nhân là `railway.toml` override Docker `CMD` bằng start command chứa `$PORT`
nhưng không bọc shell, nên biến không được mở rộng. Bản vá đã xóa override để
Railway dùng `CMD ["sh", "-c", ... ${PORT:-8000}]` trong Dockerfile. Sau khi
redeploy, `/health` và `/ready` đều trả 200.
