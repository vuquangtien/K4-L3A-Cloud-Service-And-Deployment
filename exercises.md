# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Quang Tiến  Mã học viên: 2A202602872

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên cloud, nếu quên tạo `AGENT_API_KEY` mà app vẫn nhận mặc định
> `changeme`, endpoint `/ask` có thể chạy công khai với một khóa ai cũng đoán
> được. Không có default làm app dừng ngay khi start, nên tôi thấy lỗi cấu hình
> trong log deploy trước khi service nhận request hoặc phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log tôi quan sát được là: `{"event":"ask_completed","level":"info","timestamp":"2026-09-28T10:28:06.201375+00:00","user_id":"reflection-test","tokens_in":6,"tokens_out":44,"cost_usd":2.73e-05}`. Tôi có thể lọc/tổng hợp chi phí theo `user_id`, và tạo cảnh báo hoặc thống kê theo `level`, số token hay `cost_usd`. Với một chuỗi `print("đã trả lời xong")`, máy không có các trường để lọc hay cộng dồn như vậy.

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
| 1 stage (bản đầu) | Chưa đo xong: image `python:3.11` đầy đủ đang tải chậm ở máy tôi |
| Multi-stage | 247 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản multi-stage đo được là 247 MB. Bản one-stage dùng base Python đầy đủ và
> giữ luôn toàn bộ build environment cùng dependency trong image cuối; tôi
> không ghi một số đo ước lượng vì lần build image một-stage chưa tải xong base
> image. Phần chênh lệch chủ yếu là base image đầy đủ và các công cụ/layer chỉ
> cần khi build, còn bản multi-stage chỉ copy kết quả cài dependency sang
> runtime `python:3.11-slim`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ đổi `app/main.py`, layer `COPY requirements.txt` và `RUN pip install`
> vẫn được cache; build log thực tế hiển thị hai layer này là `CACHED`. Layer
> `COPY app` và các layer sau nó phải chạy lại. Nếu đặt `COPY . .` trước `RUN
> pip install`, thay đổi một ký tự code sẽ làm cache bị invalid từ `COPY . .`,
> khiến pip phải cài lại dependency dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu có lỗ hổng cho phép thực thi lệnh trong Python app, tiến trình trong
> container có thể chạy lệnh với quyền root. Kẻ tấn công sau đó có thể khai
> thác thêm cấu hình sai, mount nhạy cảm hoặc lỗi container runtime để tăng
> mức ảnh hưởng lên host. `USER appuser` khiến tiến trình app không có quyền
> root ngay từ đầu, nên lệnh thực thi được trong app bị giới hạn bởi quyền của
> user thường thay vì mặc định có toàn quyền trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request trong 2 giây: gửi 10 request ở 10:00:59 và thêm 10
> request ở 10:01:01. Bộ đếm theo phút đồng hồ reset tại giây 00 nên cho cả hai
> nhóm qua. Sliding window của tôi đếm bất cứ 60 giây liên tiếp nào, nên nhóm
> thứ hai bị chặn cho tới khi request cũ rời cửa sổ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất request, còn cost guard theo dõi tổng tiền đã
> dùng trong tháng của từng user. Ví dụ, user gửi ít request nhưng mỗi request
> có prompt rất dài: chưa vượt 10 request/phút nên rate limit cho qua, nhưng
> khi đã gần hết `MONTHLY_BUDGET_USD` thì cost guard chặn. Ngược lại, user gửi
> nhiều câu ngắn gần như không tốn tiền trong vài giây sẽ chạm rate limit trước
> khi chi tiêu tháng đủ lớn để cost guard chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng ping Redis, khi Redis mất kết nối thì cả ba container trả
> health fail. Orchestrator lần lượt coi từng container là chết và restart cả
> ba; trong lúc đó Redis vẫn lỗi nên container mới cũng lại fail, làm sự cố
> nhỏ thành gián đoạn toàn bộ service. Tách endpoint giúp `/health` chỉ kiểm
> tra process, còn `/ready` trả 503 để load balancer ngừng gửi request mới vào
> instance chưa dùng được Redis.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Compose hiện map cố định `8000:8000`, nên scale trực tiếp `agent=3` trên một
> host sẽ vướng trùng host port; để scale cần bỏ port mapping hoặc đặt load
> balancer phía trước. Logic history đã được kiểm tra bằng hai instance
> `ConversationStore` cùng Redis và instance thứ hai đọc được dữ liệu instance
> đầu ghi. Vì vậy các request cùng `X-User-Id` sẽ tăng history theo 0, 2, 4...
> dù đi vào instance khác. Nếu dùng dict Python riêng trong từng container,
> history_length sẽ tùy instance nhận request: có lúc 0, có lúc 2, thay vì tăng
> nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Tôi thử Railway trước nhưng dashboard hiển thị `Trial expired`, nên không
> thể tạo service mới bằng free trial. Tôi xác định nguyên nhân từ banner trên
> dashboard và chuyển sang Render. Trên Render, Blueprint từ `render.yaml` tạo
> `day12-redis` và `day12-agent`; sau khi deploy, `/health` và `/ready` đều
> trả 200, còn `/ask` không có key trả 401.
