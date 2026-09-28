# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đức Danh  Mã học viên: 2A202602722

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy tôi quên tạo `AGENT_API_KEY` trên
> dashboard. Nếu code dùng mặc định `"changeme"`, service vẫn báo deploy thành
> công và người khác có thể đoán khóa để gọi `/ask`. Khi trường này bắt buộc,
> Pydantic báo `ValidationError` ngay lúc khởi động, nên bản cấu hình sai không
> thể nhận traffic và tôi biết phải bổ sung secret trước khi tiếp tục.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được là
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T16:25:54.350585+00:00","user_id":"sv-smoke","tokens_in":3,"tokens_out":41,"cost_usd":2.505e-05}`.
> Từ log này tôi có thể lọc hoặc đếm số request theo `event` và `user_id`; tôi
> cũng có thể cộng `cost_usd`, theo dõi token và đặt cảnh báo khi chi phí tăng.
> Chuỗi `print("đã trả lời xong")` không có các trường ổn định để máy thực hiện
> hai việc đó.

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
| 1 stage (bản đầu) | 435.6 MB |
| Multi-stage | 64.0 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo bằng `docker image inspect`: bản một stage là 435,563,513 bytes, còn
> bản multi-stage là 64,000,509 bytes, giảm khoảng 371.6 MB. Phần chênh lệch
> chủ yếu là base image Python đầy đủ và những thành phần chỉ cần trong quá
> trình build. Bản runtime dùng `python:3.11-slim` và chỉ nhận dependency đã
> cài cùng source cần chạy, nên không mang toàn bộ môi trường builder sang.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ đổi `app/main.py`, các layer lấy base image, `COPY requirements.txt`
> và `RUN pip install` vẫn dùng lại cache vì file dependency không đổi. Layer
> `COPY app ./app` và các layer runtime đứng sau nó phải tạo lại. Nếu đặt
> `COPY . .` trước `RUN pip install`, mọi thay đổi source sẽ làm mất cache từ
> `COPY` trở đi và pip phải cài lại toàn bộ thư viện dù `requirements.txt`
> không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python có lỗ hổng thực thi lệnh từ xa, kẻ tấn công trước hết có
> quyền của process trong container. Nếu process là root và container còn có
> mount hoặc cấu hình đặc quyền nguy hiểm, họ có nhiều khả năng sửa file hệ
> thống, truy cập tài nguyên nhạy cảm hoặc lợi dụng thêm một lỗ hổng thoát
> container để ảnh hưởng host. `USER appuser` chuyển process sang UID 10001,
> nên ngay sau bước thực thi lệnh, quyền của kẻ tấn công đã bị giới hạn thay vì
> mặc nhiên có root trong container. Biện pháp này giảm hậu quả chứ không thay
> thế việc vá lỗ hổng và cấu hình container an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây: gửi 10 request
> vào 10:00:59, rồi sau khi bộ đếm theo phút reset ở 10:01:00, gửi tiếp 10
> request vào 10:01:01. Cả hai nhóm đều không vượt 10 request trong từng phút
> đồng hồ, nhưng tổng tải tức thời vẫn là 20. Sliding window 60 giây sẽ nhìn
> thấy cả hai nhóm cùng lúc và chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng số tiền của từng user trong cả tháng. Một user gửi chậm, ví dụ mỗi phút
> một request rất dài, có thể luôn qua rate limit nhưng bị cost guard chặn khi
> hết ngân sách. Ngược lại, một user chưa tiêu đáng kể nhưng gửi 11 request rất
> ngắn liên tiếp sẽ còn ngân sách nhưng request thứ 11 bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container
> cùng trả 503 cho liveness. Orchestrator coi cả ba process bị hỏng và lần lượt
> hoặc đồng thời restart chúng. Trong lúc restart, load balancer không còn
> instance khỏe để nhận request. Redis có thể phục hồi sau 30 giây nhưng các
> container vẫn đang khởi động lại, làm sự cố phụ thuộc ngắn biến thành gián
> đoạn toàn dịch vụ. Tách `/ready` giúp load balancer tạm ngừng gửi traffic mà
> không buộc orchestrator restart process vẫn còn sống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi gọi hai lần với cùng `X-User-Id`, tôi quan sát `history_length` tăng từ 0
> lên 2 vì một lượt hỏi trước đó lưu hai message (user và assistant) trong
> Redis. Với ba instance dùng dict Python riêng, request đi vào instance chưa
> từng gặp user sẽ lại thấy 0; request quay lại đúng instance cũ mới thấy 2,
> 4... Vì load balancer phân phối request, con số sẽ tăng không đều hoặc nhảy
> lùi thay vì tăng nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi dùng phương án local fallback nên chưa tạo service cloud. Lỗi triển khai
> gần nhất khi smoke test là `curl: (7) Failed to connect to 127.0.0.1 port
> 8000`. Lúc đó `docker compose ps` cho thấy agent mới ở trạng thái `health:
> starting`, và lệnh gọi trong sandbox cũng không truy cập được cổng host. Tôi
> đọc `docker compose logs agent` để xác nhận Uvicorn đã bind `0.0.0.0:8000`,
> chờ healthcheck chuyển sang `healthy`, rồi gọi lại ngoài sandbox. Kết quả sau
> đó là `/health` 200 và `/ready` 200 với `redis=true`.
