# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hoài Nam Mã học viên: 2A202601399

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc tôi deploy lên Railway nhưng quên khai báo
> `AGENT_API_KEY`. Nếu ứng dụng dừng ngay khi đọc cấu hình, deployment sẽ báo lỗi
> trước khi nhận traffic và tôi biết ngay biến nào còn thiếu. Nếu dùng khóa mặc
> định `"changeme"`, service vẫn chạy công khai; người khác có thể đoán khóa này,
> gọi `/ask` và làm phát sinh chi phí dưới tài khoản của tôi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi lấy được khi gọi `/ask` là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:32:09.269343+00:00", "user_id": "anonymous", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`.
> Từ log JSON này, tôi có thể lọc tất cả sự kiện `ask_completed` theo
> `user_id` để điều tra hoạt động của một người dùng; tôi cũng có thể cộng
> `tokens_in`, `tokens_out` và `cost_usd` để làm dashboard hoặc cảnh báo chi
> phí. Dòng `print("đã trả lời xong")` không có các trường có cấu trúc để máy
> lọc và tổng hợp như vậy.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản               | Dung lượng |
| ----------------- | ---------- |
| 1 stage (bản đầu) | 1690 MB    |
| Multi-stage       | 270 MB     |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker images`: bản single-stage dùng `python:3.11` là 1,69 GB,
> còn bản multi-stage dùng runtime `python:3.11-slim` là 270 MB, giảm khoảng
> 1,42 GB. Phần chênh lệch chủ yếu là hệ điều hành đầy đủ, công cụ phát triển,
> compiler và các dữ liệu chỉ cần trong quá trình cài dependency. Runtime chỉ
> nhận kết quả đã cài từ builder nên không phải mang toàn bộ môi trường build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa `app/main.py`, stage builder vẫn dùng lại các layer base,
> `WORKDIR`, `COPY requirements.txt` và `pip install` vì file requirements
> không đổi. Ở runtime, base image, `WORKDIR` và layer copy dependency từ
> builder cũng được dùng lại; từ `COPY app/ app/` trở đi phải cập nhật lại.
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một thay đổi trong source cũng
> làm mất cache của layer copy và buộc `pip install` chạy lại dù dependency
> không đổi, khiến mỗi lần build chậm và tốn mạng hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ ứng dụng Python có lỗ hổng cho phép thực thi lệnh từ xa. Kẻ tấn công
> trước tiên chạy lệnh trong container; nếu process là root, họ có toàn quyền
> sửa file và tiến trình trong container. Khi container còn được cấp capability
> nguy hiểm, mount Docker socket hoặc host path, họ có thể lợi dụng cấu hình đó
> hay một lỗ hổng container runtime để tác động tới host với quyền cao.
> `USER appuser` cắt chuỗi ở bước đầu bằng cách khiến mã bị chiếm quyền chỉ chạy với
> UID thường, giảm quyền và phạm vi thiệt hại. Nó không thay thế việc bỏ các
> mount/capability nguy hiểm nhưng tạo thêm một lớp phòng vệ quan trọng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây: gửi đủ 10
> request vào cuối một phút, chẳng hạn 10:00:59, rồi gửi thêm 10 request ngay
> sau khi bộ đếm reset ở 10:01:00. Cả hai phút đồng hồ đều không vượt mức
> 10/phút, nhưng thực tế 20 request dồn vào khoảng một đến hai giây. Sliding
> window nhìn lại đúng 60 giây nên không có khe hở ở ranh giới phút này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit khống chế số request trong một khoảng thời gian, còn cost guard
> khống chế tổng số tiền đã tiêu trong tháng. Một user gửi chỉ hai request nhưng
> mỗi prompt rất dài, hoặc đã gần hết ngân sách tháng, vẫn có thể qua rate limit
> nhưng bị cost guard chặn. Ngược lại, một user gửi 11 câu hỏi rất ngắn trong
> một phút có thể còn tiêu rất ít tiền và qua cost guard, nhưng request thứ 11
> phải bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp liveness và readiness rồi cho endpoint đó kiểm tra Redis, khi Redis
> mất kết nối thì cả ba container cùng trả health check lỗi. Load balancer rút
> cả ba khỏi danh sách nhận traffic, còn orchestrator tưởng process bị hỏng nên
> lần lượt restart chúng. Các container khởi động lại nhưng Redis vẫn mất kết
> nối, health check tiếp tục lỗi và cụm rơi vào vòng lặp restart; request đang
> xử lý cũng có thể bị gián đoạn. Tách endpoint giúp `/health` vẫn báo process
> sống để không restart vô ích, còn `/ready` trả 503 để tạm ngừng đưa traffic
> tới instance cho đến khi Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi ba agent dùng chung Redis, tôi thấy `history_length` tăng đều theo cùng
> một user: `0, 2, 4, 6, 8, ...`, vì mỗi request ghi thêm hai message và
> container nào nhận request cũng đọc cùng một key Redis. Nếu lưu bằng một
> `dict` Python, mỗi container có lịch sử riêng. Với nginx phân phối lần lượt,
> tôi có thể thấy dạng `0, 0, 0, 2, 2, 2, 4, ...` hoặc số liệu nhảy không đều
> tùy container nhận request; restart container còn làm lịch sử của riêng nó
> trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi cloud đáng nhớ của tôi là
> `/bin/sh: 1: exec: docker-entrypoint.sh: not found`. Build Dockerfile Python
> đã thành công nhưng container không thể khởi
> động. Tôi đọc deployment log và chạy `railway status --json`, từ đó thấy CLI
> đang liên kết với service tên `Redis`; Railway đã đưa image Python vào service
> Redis nhưng vẫn giữ start command `docker-entrypoint.sh redis-server` và
> volume `/data`. Tôi khôi phục deployment Redis, tạo một Empty Service riêng
> tên `agent`, liên kết CLI đúng service rồi deploy lại. Sau đó tôi đặt
> `AGENT_API_KEY`, tham chiếu `REDIS_URL` từ service Redis và để Railway tự cấp
> `PORT`; `/health` và `/ready` mới hoạt động đúng.
