# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục         | Nội dung                                                                   |
| ----------- | -------------------------------------------------------------------------- |
| Họ và tên   | Nguyễn Hoài Nam                                                            |
| Mã học viên | 2A202601399                                                                |
| Repo        | https://github.com/NguyenHoaiNam2k5/K3-Day12-2A202601399-NguyenHoaiNam.git |

## Service

| Mục         | Nội dung                                      |
| ----------- | --------------------------------------------- |
| Public URL  | https://agent-production-9633.up.railway.app/ |
| Platform    | Railway                                       |
| Ngày deploy | 10/08/2026                                    |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến                    | Đã set | Ghi chú                                   |
| ----------------------- | ------ | ----------------------------------------- |
| `PORT`                  | ✅     | platform tự gán                           |
| `AGENT_API_KEY`         | ✅     | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL`             | ✅     | Redis add-on của platform                 |
| `RATE_LIMIT_PER_MINUTE` | ✅     | 10                                        |
| `MONTHLY_BUDGET_USD`    | ✅     | 10.0                                      |
| `LOG_LEVEL`             | ✅     | INFO                                      |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl.exe -i "$URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl.exe -i "$URL/ready"

# 3. Không có API key — mong đợi 401
curl.exe -i -X POST "$URL/ask" -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test"  --data-raw '{\"question\":\"Hello\"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl.exe -i -X POST "$URL/ask" -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test"  --data-raw '{\"question\":\"Deploy\"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
1..15 | ForEach-Object { $code = curl.exe -s -o NUL -w "%{http_code}" -X POST "$URL/ask" -H "Content-Type: application/json" -H "X-API-Key:              $AGENT_API_KEY" -H "X-User-Id: sv-test" --data-raw '{\"question\":\"test\"}'; Write-Host -NoNewline "$code " }; Write-Host
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:12:54 GMT
Server: railway-hikari
x-railway-request-id: HQLhcdGcTV6btOYMwUFZXw
Content-Length: 57
x-hikari-trace: sin1.nzn2
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:15:44 GMT
Server: railway-hikari
x-railway-request-id: mlUXEeqPQyKKxeNaY53eZw
Content-Length: 31
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}

# 3. Không có API key
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:18:26 GMT
Server: railway-hikari
x-railway-request-id: ihrQLTpCTqyGMUjcWUN5dQ
Content-Length: 39
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing API key"}

# 4. Có API key
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:43:54 GMT
Server: railway-hikari
x-railway-request-id: lWwDn0D1QwGuKmKI2prcFg
Content-Length: 331
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"answer":"Câu hỏi hay. Deploy thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 20 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":20,"cost_usd":9.51e-05,"tokens":{"in":458,"out":44}}

# 5. Rate limit
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt
- `screenshots/ready.png` — kết quả gọi `/ready` từ trình duyệt
- `screenshots/KoAPIKey.png` — kết quả lệnh #3
- `screenshots/CoAPIKey.png` — kết quả lệnh #4
- `screenshots/RateLimit.png` — kết quả lệnh #5
