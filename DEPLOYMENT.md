# Thông tin deploy — Checkpoint 5

## Học viên và repository

| Mục | Giá trị |
|---|---|
| Họ tên | Trần Nguyễn Tiến Đức |
| Mã học viên | 2A202602871 |
| Repository | https://github.com/TNTD-dev/K4-L3A-DAY12-TranNguyenTienDuc-2A202602871-CloudServicesAndDeployment |

## Service công khai

| Mục | Giá trị |
|---|---|
| Public URL | https://day12-agent-3wj1.onrender.com |
| Platform | Render, Web Service Docker Free và Key Value Free |
| Ngày deploy | 2026-09-28 |
| Region | Singapore |
| Service ID | `srv-dat1s3l9fdbs73fj9c1g` |

## Cấu hình trên Render

| Biến | Nguồn giá trị |
|---|---|
| `PORT` | Render cấp cho Web Service; Dockerfile đọc biến này |
| `AGENT_API_KEY` | Secret đặt trong cấu hình Web Service, không lưu trong repo |
| `REDIS_URL` | Địa chỉ nội bộ của Render Key Value `day12-redis` |
| `RATE_LIMIT_PER_MINUTE` | Cấu hình Web Service: 10 |
| `MONTHLY_BUDGET_USD` | Cấu hình Web Service: 10.0 |
| `LOG_LEVEL` | Cấu hình Web Service: INFO |

Không có giá trị secret nào được ghi trong tài liệu này. Redis Key Value Free không lưu dữ liệu xuống đĩa; lịch sử và bộ đếm có thể mất khi Redis khởi động lại.

## Kết quả kiểm tra thực tế

Các lệnh dưới đây được gọi vào URL công khai ngày 2026-09-28. Khóa hợp lệ chỉ nằm trong `.env` cục bộ và cấu hình Render.

```text
GET /health → HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

GET /ready → HTTP 200
{"status":"ready","redis":true}

POST /ask không có X-API-Key → HTTP 401
{"detail":"invalid or missing API key"}

POST /ask có X-API-Key hợp lệ và X-User-Id: cloud-check → HTTP 200
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"cloud-check","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

15 lần POST /ask liên tiếp với X-User-Id: rate-proof-20260928:
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh minh chứng

- `screenshots/render-deployment.png`: dashboard Render với Web Service đang Live.
- `screenshots/health-response.png`: kết quả gọi `/health` trên URL công khai.
