# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lâm Quang Anh Quân  Mã học viên: 2A202602467

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy mà quên đặt `AGENT_API_KEY`, `Settings` báo lỗi ngay lúc khởi động, nên tôi biết cấu hình chưa đủ trước khi service nhận request. Nếu dùng khóa mặc định `changeme`, service vẫn chạy và bất kỳ ai biết giá trị đó đều có thể gọi `/ask`.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Cần chạy `/ask` và dán **dòng log thật** vào đây sau khi tự cấu hình môi trường. Với `event`, `timestamp`, `user_id` và `cost_usd`, tôi có thể lọc các lần gọi của một user trong khoảng thời gian cụ thể và cộng chi phí của user đó. Dòng `print("đã trả lời xong")` thiếu các trường để lọc và tính tổng.

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
| 1 stage (bản đầu) | Cần đo bằng `docker images agent:single` |
| Multi-stage | Cần đo bằng `docker images agent:multi` |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Cần tự build cả hai image để ghi số đo thật. Bản một stage giữ base image đầy đủ, thư viện và mọi file được `COPY . .`; bản multi-stage chỉ đưa thư viện đã cài và `app/`, `utils/` vào runtime `python:3.11-slim`. Chênh lệch cụ thể phụ thuộc image và dependency tại lúc build, nên tôi chưa ghi số MB khi chưa đo.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer `COPY requirements.txt` và cài thư viện ở builder vẫn dùng cache; layer `COPY app ./app` ở runtime cùng các layer sau nó phải tạo lại. Nếu `COPY . .` đứng trước `RUN pip install`, sửa code làm layer copy đổi, kéo theo cài lại thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Lỗ hổng cho phép chạy lệnh có thể giúp kẻ tấn công chiếm quyền của process trong container. Nếu process chạy root, tác động trong container lớn hơn; cấu hình host nguy hiểm như mount socket Docker hoặc lỗi thoát container có thể dẫn tới quyền cao trên host. `USER appuser` hạ quyền process trong container, giảm tác động của bước đầu, nhưng không tự loại bỏ mọi đường leo quyền lên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 10 request ở giây 59 của phút trước và 10 request ở giây 00 của phút sau: tổng cộng 20 request trong khoảng 2 giây nhưng mỗi phút đồng hồ vẫn chỉ có 10. Cửa sổ trượt nhìn lại 60 giây nên chặn lượt thứ 11 trong khoảng đó.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm số request trong 60 giây; cost guard cộng chi phí theo user và tháng. User mới gửi một request nhưng chi phí dự kiến vượt ngân sách sẽ qua rate limit rồi bị cost guard chặn. User còn ngân sách nhưng đã gửi đủ 10 request trong 60 giây sẽ bị rate limit chặn dù cost guard vẫn cho phép.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp trả lỗi trên cả ba container → orchestrator có thể đánh dấu cả ba unhealthy và restart đồng thời → trong lúc restart không còn instance nhận traffic. Khi tách probe, `/health` vẫn báo process sống; `/ready` trả 503 để load balancer ngừng gửi request, rồi trả 200 khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Cần tự chạy 3 instance để ghi `history_length` quan sát được. Theo luồng code, cùng một Redis sẽ cho chuỗi `0, 2, 4, 6, 8` với năm request liên tiếp, vì mỗi lượt thêm hai message. Nếu dùng dict Python riêng, request chuyển sang instance khác sẽ thấy lịch sử rỗng hoặc ngắn hơn; các con số tăng không đều và phụ thuộc instance nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Cần tự deploy rồi ghi **một lỗi thực tế**, thông báo lỗi, cách tìm nguyên nhân trong build/runtime log và cách sửa. Tôi chưa có một lần deploy thực để có thể ghi lỗi quan sát được mà không bịa bằng chứng.
