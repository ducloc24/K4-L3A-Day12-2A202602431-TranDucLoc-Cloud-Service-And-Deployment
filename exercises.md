# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> Bài làm cá nhân, dựa trên quá trình thực hiện lab.
>
> Họ và tên: Trần Đức Lộc  Mã học viên: 2A202602431

---

### Câu 1 — Fail fast (CP1)

Nếu deploy lên cloud mà quên cấu hình `AGENT_API_KEY`, `Settings` báo thiếu trường bắt buộc ngay lúc khởi động nên deployment thất bại và mình phát hiện lỗi trong log trước khi service nhận request. Nếu đặt mặc định là `changeme`, app vẫn chạy; mình có thể tưởng đã bảo vệ API trong khi khóa yếu hoặc công khai đó vẫn được dùng để gọi `/ask`.

---

### Câu 2 — Log cho máy đọc (CP1)

Ví dụ một dòng JSON từ structured logger:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:49:55.254174+00:00", "user_id": "sv01", "tokens_in": 4, "tokens_out": 36, "cost_usd": 2.22e-05}
```

Từ log này, mình lọc và cộng chi phí theo `user_id` để biết user nào tiêu nhiều nhất; đồng thời lọc `level=error` hoặc đếm sự kiện theo thời gian để tính số lỗi/tỷ lệ lỗi. Một dòng `print("đã trả lời xong")` không có các trường cấu trúc đó để lọc và thống kê tự động.

---

### Câu 3 — Kích thước image (CP2)

Mình build lại Dockerfile one-stage ban đầu và Dockerfile multi-stage hiện tại trên cùng Docker Engine. Kết quả `docker image ls`:

| Bản | Dung lượng đo được |
|---|---:|
| 1 stage | 1.73 GB |
| Multi-stage | 271 MB |

Bản multi-stage nhỏ hơn khoảng 1.46 GB. Bản one-stage dùng `python:3.11` đầy đủ và giữ mọi thứ trong cùng image; bản multi-stage dùng `python:3.11-slim`, rồi chỉ chuyển các dependency đã cài sang runtime. Vì thế runtime không mang theo phần dư của base image đầy đủ và các thành phần chỉ cần trong giai đoạn build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Dockerfile hiện copy `requirements.txt` và cài thư viện trước khi copy source. Khi chỉ sửa `app/main.py`, layer cài dependency vẫn được dùng lại từ cache; các layer copy source nằm sau điểm thay đổi phải chạy lại. Nếu đặt `COPY . .` trước `pip install`, mỗi thay đổi trong source làm layer copy đổi, khiến bước cài thư viện phía sau mất cache và chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Nếu có lỗ hổng cho phép chạy lệnh trong app, kẻ tấn công có quyền của user đang chạy container. Chạy bằng root cho phép họ sửa nhiều file và truy cập rộng hơn vào tài nguyên được mount; nếu tiếp tục khai thác lỗ hổng container/runtime thì rủi ro ảnh hưởng host cũng tăng. `USER appuser` chuyển tiến trình sang user thường trước khi chạy app, giới hạn quyền trong container và giảm mức độ thiệt hại có thể gây ra.

---

### Câu 6 — Cửa sổ trượt (CP3)

Tối đa 20 request trong 2 giây: gửi 10 request ngay trước thời điểm phút đổi (ví dụ `10:00:59`), rồi thêm 10 request ngay sau khi bộ đếm theo phút reset (ví dụ `10:01:00`). Mỗi phút riêng lẻ vẫn có 10 request, nhưng trong một khoảng 2 giây người dùng đã gửi 20 request. Sliding window 60 giây tránh lỗ hổng này vì cả 20 request vẫn nằm trong cùng cửa sổ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Rate limit giới hạn số request trong một khoảng thời gian; cost guard giới hạn tổng chi phí trong tháng. Rate limit có thể cho request qua nhưng cost guard chặn nếu user còn ít ngân sách mà request dự kiến tốn nhiều tiền. Ngược lại, cost guard vẫn cho phép vì user còn ngân sách, nhưng rate limit chặn request thứ 11 trong cửa sổ 60 giây khi giới hạn là 10/phút.

---

### Câu 8 — `/health` khác `/ready` (CP4)

Nếu `/health` cũng ping Redis, khi Redis mất kết nối thì cả ba container đều báo unhealthy. Orchestrator có thể lần lượt loại bỏ hoặc restart cả ba, dù process của agent vẫn chạy; trong lúc đó cụm không còn instance phục vụ, và restart liên tục làm sự cố nặng hơn. Tách probe giúp `/health` vẫn báo process còn sống, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic vào các instance không kết nối được Redis.

---

### Câu 9 — Stateless (CP4)

Với Redis dùng chung, mỗi request đọc lịch sử nhất quán bất kể được xử lý bởi instance nào; mỗi lần hỏi mới thì lịch sử trước request tăng thêm hai message (user và assistant). Nếu lưu trong dict của từng process, mỗi container có bản lịch sử riêng: request vào instance mới có thể thấy `history_length` bằng 0 hoặc thấp hơn, nên con số thay đổi tùy request được chuyển đến container nào.

---

### Câu 10 — Deploy thật (CP5)

Khi deploy Railway, log báo `Invalid value for '--port': '$PORT' is not a valid integer.` Mình xem Deploy Logs và thấy lệnh start truyền nguyên chuỗi `$PORT` cho Uvicorn thay vì giá trị port Railway cấp. Nguyên nhân là Start Command chạy trực tiếp nên biến không được shell mở rộng. Mình bỏ `startCommand` đó khỏi `railway.toml` và để Dockerfile chạy `sh -c` với `${PORT:-8000}`. Sau đó deploy service thành công; ở bản Render, mình kiểm tra `/health` và `/ready` đều trả HTTP 200, Redis báo `true`.

