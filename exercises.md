# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đình Anh Đức  Mã học viên: 2A202602856

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu quên cấu hình `AGENT_API_KEY`, việc fail fast làm ứng dụng dừng ngay và deployment báo lỗi. Nhờ đó phát hiện cấu hình thiếu trước khi service nhận traffic. Nếu dùng khóa mặc định `"changeme"`, ứng dụng vẫn chạy và bất kỳ ai đoán được khóa này đều có thể gọi `/ask`, tiêu quota và ngân sách của service.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:03:54.758421+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```
Thứ nhất, có thể lọc log theo `event`, `level` hoặc `user_id`, ví dụ tìm toàn bộ request của `sv-test`. Thứ hai, có thể tổng hợp `tokens_in`, `tokens_out` và `cost_usd` để tạo dashboard hoặc cảnh báo chi phí. Dòng `print("đã trả lời xong")` không chứa các trường có cấu trúc để thực hiện hai việc này.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 272 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage dùng `python:3.11` đầy đủ và giữ toàn bộ hệ điều hành nền, dependency, cache cài đặt và các thành phần dùng trong quá trình build. Bản multi-stage dùng `python:3.11-slim`; stage runtime chỉ nhận dependency đã cài từ stage builder và source code, nên không mang toàn bộ filesystem trung gian sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py` rồi build lại, các layer base image, `COPY requirements.txt` và `RUN pip install` được lấy từ cache vì `requirements.txt` không đổi. Layer `COPY . .` và các layer đứng sau nó phải chạy lại vì nội dung source đã thay đổi. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần thay đổi một file source cũng làm cache của layer copy mất hiệu lực, khiến Docker phải cài lại toàn bộ dependency dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro có thể bắt đầu từ một lỗ hổng trong API cho phép kẻ tấn công thực thi lệnh trong container. Nếu container chạy bằng root và còn được cấp quyền nguy hiểm, mount thư mục host hoặc có lỗ hổng container escape, kẻ tấn công có thể dùng quyền root trong container để tác động đến host. Lệnh `USER appuser` làm mã ứng dụng và lệnh bị khai thác chỉ chạy với quyền của user thường, giảm quyền đọc, ghi và cài đặt trong container. Biện pháp này không tự loại bỏ mọi khả năng container escape, nhưng cắt bớt đặc quyền mà kẻ tấn công có ngay sau khi khai thác ứng dụng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây. Họ gửi 10 request ở cuối phút cũ, ví dụ lúc `10:00:59`, rồi gửi tiếp 10 request ngay đầu phút mới, ví dụ lúc `10:01:00`. Bộ đếm theo phút đồng hồ xem đây là hai cửa sổ khác nhau nên cả hai nhóm đều hợp lệ. Sliding window 60 giây sẽ nhìn thấy đủ 20 request trong 60 giây gần nhất và chặn nhóm vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong một khoảng thời gian, còn cost guard giới hạn tổng chi phí tích lũy của mỗi user trong tháng. Ví dụ user chỉ gửi một request trong phút nhưng đã tiêu gần hết ngân sách tháng; rate limit vẫn cho qua nhưng cost guard trả 402. Ngược lại, user còn nhiều ngân sách nhưng gửi request thứ 11 trong 60 giây với giới hạn 10 request/phút; cost guard vẫn cho phép về mặt chi phí nhưng rate limiter trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` và `/ready` rồi cho endpoint đó kiểm tra Redis, khi Redis mất kết nối thì cả ba container cùng trả 503. Load balancer loại cả ba instance khỏi danh sách nhận traffic nên service không còn backend khỏe để phục vụ. Nếu endpoint này đồng thời được dùng làm liveness probe, orchestrator còn cho rằng cả ba process bị hỏng và restart chúng. Các container mới vẫn không kết nối được Redis, tiếp tục trả 503 và có thể tạo thành vòng lặp restart. Tách hai endpoint giúp `/health` vẫn trả 200 vì process còn sống, còn `/ready` trả 503 để tạm ngừng nhận traffic cho đến khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi lưu history trong Redis, các container dùng chung một nguồn dữ liệu nên `history_length` tăng đều theo từng request, thường là `0, 2, 4, 6...` vì mỗi lượt lưu một message `user` và một message `assistant`. Nếu dùng dict Python, mỗi container có lịch sử riêng. Khi load balancer phân phối request qua ba container, tôi có thể thấy kết quả kiểu `0, 0, 0, 2, 2, 2...` hoặc thay đổi không đều tùy request rơi vào container nào. Khi container restart, lịch sử trong dict của container đó cũng mất hoàn toàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy lên Railway, `/health` trả 200 nhưng `/ready` trả `500 Internal Server Error`. Tôi xác định ứng dụng và Uvicorn vẫn chạy vì `/health` không phụ thuộc Redis, nên lỗi nằm ở cấu hình được dùng khi tạo Redis client. Trong Railway Variables, `AGENT_API_KEY` đang tự tham chiếu bằng `${{AGENT_API_KEY}}`, còn `REDIS_URL` lại tham chiếu nhầm `${{day12-redis.DATABASE_URL}}`. Tôi sửa `AGENT_API_KEY` thành giá trị khóa thật trong Railway secret variables và sửa Redis thành `${{day12-redis.REDIS_URL}}`, sau đó apply thay đổi và redeploy. Sau khi sửa, `/ready` kết nối được Redis và trả `{"status":"ready","redis":true}`.
