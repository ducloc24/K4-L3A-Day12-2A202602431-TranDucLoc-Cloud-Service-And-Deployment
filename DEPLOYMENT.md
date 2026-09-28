# Thông Tin Deploy — Checkpoint 5

> Chỉ ghi tên biến môi trường; tuyệt đối không đưa giá trị API key hoặc Redis URL vào tài liệu.

## Thông tin học viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Trần Đức Lộc |
| Mã học viên | 2A202602431 |
| Repository | https://github.com/ducloc24/K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://day12-agent-xf56.onrender.com |
| Platform | Render |
| Ngày kiểm tra deployment | 2026-09-28 |

## Biến môi trường đã cấu hình trên Render

| Biến | Đã set | Ghi chú |
|---|---|---|
| `PORT` | Có | Render cấp cho web service |
| `AGENT_API_KEY` | Có | Đặt trong dashboard Render; không ghi giá trị vào repo |
| `REDIS_URL` | Có | Redis service của Render |
| `RATE_LIMIT_PER_MINUTE` | Có | `10` |
| `MONTHLY_BUDGET_USD` | Có | `10.0` |
| `LOG_LEVEL` | Có | `INFO` |

## Lệnh Kiểm Tra
Thay <URL> bằng Public URL ở trên:

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

## Kết Quả Chạy Thật
Dán output của các lệnh trên vào đây:

1. (.venv) PS C:\Users\PC\Documents\K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment> curl.exe -i https://day12-agent-xf56.onrender.com/health
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:34:23 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 217d96c8-8d95-4f6c
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a42218686dad5e3b-HAN
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. (.venv) PS C:\Users\PC\Documents\K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment> curl.exe -i https://day12-agent-xf56.onrender.com/ready 
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:34:36 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 23fcd32e-ca4f-4645
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a42218b9dce25e36-HAN
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}

3. (.venv) PS C:\Users\PC\Documents\K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment> curl.exe -i -X POST https://day12-agent-xf56.onrender.com/ask `  -H "Content-Type: application/json" `  -d '{\"question\":\"Hello\"}'
HTTP/1.1 401 Unauthorized
Date: Mon, 28 Sep 2026 10:37:48 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 2654e581-0682-4bb3
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a4221d67fbad5e35-HAN
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}

4. (.venv) PS C:\Users\PC\Documents\K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment> curl.exe -i -X POST https://day12-agent-xf56.onrender.com/ask `  -H "Content-Type: application/json" `  -H "X-API-Key: $env:AGENT_API_KEY" `  -H "X-User-Id: sv-test" `  --data-binary "@request.json"
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 10:41:59 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: 97570aeb-66e8-430d
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a422238b7abf5e36-HAN
alt-svc: h3=":443"; ma=86400

{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái,và giới hạn tài nguyên.","user_id":"sv-test","history_length":0,"cost_usd":2.265e-05,"tokens":{"in":3,"out":37}}

5. (.venv) PS C:\Users\PC\Documents\K4-L3A-Day12-2A202602431-TranDucLoc-Cloud-Service-And-Deployment> 1..15 | ForEach-Object {
>>   curl.exe -s -o NUL -w "%{http_code} " -X POST https://day12-agent-xf56.onrender.com/ask `
>>     -H "Content-Type: application/json" `
>>     -H "X-API-Key: $env:AGENT_API_KEY" `
>>     -H "X-User-Id: sv-test" `
>>     --data-binary "@request.json"
>> }
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429 


## Ảnh chụp màn hình
Ảnh dashboard và kết quả health check có thể lưu trong `screenshots/`.
