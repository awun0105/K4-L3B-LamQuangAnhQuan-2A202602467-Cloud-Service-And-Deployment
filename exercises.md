# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lâm Quang Anh Quân  Mã học viên: 2A202602467

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Thiếu `AGENT_API_KEY` thì app dừng ngay lúc khởi động. Nếu mặc định là `changeme`, người lạ biết khóa đó có thể gọi `/ask` trước khi tôi phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log thật từ container: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T16:39:43.884257+00:00", "user_id": "sv-lab-observe-0929", "tokens_in": 2, "tokens_out": 34, "cost_usd": 2.07e-05}`. Có thể lọc theo user/thời gian và cộng chi phí; `print("đã trả lời xong")` không có dữ liệu để làm hai việc đó.

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
| 1 stage (image cũ `day12-agent:cp2-test`) | 1.24 GB |
| Multi-stage (`day12-agent:prod`) | 184 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image một stage đã có từ bản Dockerfile gốc; `docker history` xác nhận `FROM python:3.11`, `COPY . .`, và `pip install`. Image mới dùng `python:3.11-slim`, chỉ chép dependency và source cần chạy sang stage cuối; vì vậy nhỏ hơn khoảng 1.06 GB theo số hiển thị của Docker.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi đổi một ký tự trong bản sao tạm rồi build lại: `COPY requirements.txt` và `RUN pip install` báo `CACHED`, còn `COPY app` báo `DONE`. Đặt `COPY . .` trước `pip install` sẽ khiến sửa code kéo theo cài lại thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng chạy lệnh cho kẻ tấn công quyền của process trong container. Nếu process là root và host cấu hình nguy hiểm, họ có thêm đường leo quyền lên host. `USER appuser` hạ quyền process, giảm tác động ban đầu.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: 10 ở cuối phút trước, 10 ở đầu phút sau. Cửa sổ trượt 60 giây sẽ chặn lượt thứ 11.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm request trong 60 giây; cost guard tính tiền theo tháng. Một request rất đắt có thể bị cost guard chặn dù chưa hết lượt. Ngược lại, user còn tiền nhưng gọi quá 10 lần/phút sẽ bị rate limit chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → cả ba endpoint gộp báo lỗi → orchestrator có thể restart cả ba, làm cụm mất khả năng phục vụ. Khi tách probe, `/health` vẫn 200; `/ready` trả 503 rồi tự về 200 khi Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi gọi luân phiên instance 1 → 2 → 3 → 1 → 2; `history_length` thực tế là `0, 2, 4, 6, 8`. Nếu dùng dict riêng, mỗi instance chỉ thấy lượt của nó, nên số sẽ không tăng đều như vậy.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Chưa deploy cloud nên chưa có lỗi cloud thực tế để ghi. Cần bổ sung thông báo lỗi, cách tìm nguyên nhân trong log và cách sửa sau lần deploy thật.
