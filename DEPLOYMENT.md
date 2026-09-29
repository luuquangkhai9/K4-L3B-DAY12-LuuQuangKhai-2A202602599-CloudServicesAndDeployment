# Thông tin triển khai — Checkpoint 5

## Thông tin học viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Lưu Quang Khải |
| Mã học viên | 2A202602599 |
| Repo | https://github.com/luuquangkhai9/K4-L3B-DAY12-LuuQuangKhai-2A202602599-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://agent-production-f3bd.up.railway.app |
| Platform | Railway |
| Project | `day12-luuquangkhai-agent` |
| Service | `agent` |
| Ngày deploy | 2026-09-29 |
| Nguồn deploy | Railway CLI upload từ repository này, dùng `Dockerfile` |
| Health check trên Railway | `/health`, timeout 30 giây |

## Biến môi trường trên Railway

Chỉ ghi tên biến và nguồn cấp. Giá trị secret không nằm trong repository.

| Biến | Nguồn cấp |
|---|---|
| `PORT` | Railway cấp cho service |
| `AGENT_API_KEY` | Secret đặt trên Railway; bản dùng để kiểm tra được giữ trong `.env` local dưới tên `DEPLOY_API_KEY` |
| `REDIS_URL` | Biến tham chiếu `${{Redis.REDIS_URL}}` của Redis service trong cùng project |
| `RATE_LIMIT_PER_MINUTE` | Railway variable: 10 |
| `MONTHLY_BUDGET_USD` | Railway variable: 10.0 |
| `LOG_LEVEL` | Railway variable: INFO |

Redis service chỉ dùng mạng nội bộ Railway. `agent` có domain HTTPS công khai.

## Kết quả kiểm tra thực tế

Kiểm tra trực tiếp URL công khai; deployment cuối `9cc8a7d1-cfe1-4d24-8ad7-9b480e664ab6` báo `SUCCESS` với health check `/health`:

| Yêu cầu | Mã HTTP | Kết quả quan sát |
|---|---:|---|
| `GET /health` | 200 | `{"status":"ok","service":"day12-agent","version":"1.0.0"}` |
| `GET /ready` | 200 | `{"status":"ready","redis":true}` |
| `POST /ask` không có API key | 401 | Request bị từ chối |
| `POST /ask` có API key hợp lệ | 200 | Response có trường `answer` |
| 15 lần `POST /ask` với cùng user trong một phút | 10 × 200, 5 × 429 | Sliding-window rate limit hoạt động |

Kết quả 15 lần gọi: `200,200,200,200,200,200,200,200,200,200,429,429,429,429,429`.

## Ảnh minh chứng

- `screenshots/dashboard.png`: trang quản lý service `agent` trên Railway.
- `screenshots/health.png`: kết quả gọi `/health` từ trình duyệt.

Không ghi API key, Redis URL có mật khẩu hoặc Railway token trong tài liệu và ảnh.
