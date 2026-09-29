# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Bùi Phương Duy |
| Mã học viên | 2A202602684 |
| Repo | https://github.com/DuyPhuong8804/K4-L3B-DAY12-BuiPhuongDuy-2A202602684-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-n1ri.onrender.com |
| Platform | Render (Blueprint từ `render.yaml`) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự gán |
| `AGENT_API_KEY` | ✅ | nhập lúc tạo Blueprint (`sync: false` trong render.yaml), không nằm trong repo |
| `REDIS_URL` | ✅ | tự nối từ service `day12-redis` qua `fromService` trong render.yaml |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

```
# 1. Liveness
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/1.1 200 OK
{"status":"ready","redis":true}

# 3. Khong co API key
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

# 4. Co API key
HTTP/1.1 200 OK
{"answer":"Với Deploy la gi, cách làm phổ biến trong production là đặt một lớp gateway phía trước để lo authentication, rate limiting và bảo vệ chi phí. (Mình đang nhớ 2 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":2,"cost_usd":3.315e-05,"tokens":{"in":41,"out":45}}

# 5. Rate limit - goi 15 lan (RATE_LIMIT_PER_MINUTE=10, cua so truot 60s)
200 200 200 200 200 200 200 200 429 429 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Phương Án Dự Phòng

Không áp dụng — đã deploy thành công lên Render, không cần dùng `LOCAL_FALLBACK`.
