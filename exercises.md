# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Phùng Gia Bảo  Mã học viên: 2A202602386

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên Render nhưng quên tạo `AGENT_API_KEY`. Nếu có
> khóa mặc định `"changeme"`, service vẫn báo deploy thành công và người ngoài
> có thể đoán khóa để gọi `/ask`, làm tiêu quota. Với trường bắt buộc, Pydantic
> dừng app ngay khi khởi động; lỗi xuất hiện trong deploy log trước khi service
> nhận traffic nên tôi có thể bổ sung secret rồi deploy lại.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T04:23:45.421267+00:00","user_id":"sv-test","tokens_in":1,"tokens_out":35,"cost_usd":2.115e-05}`.
> Từ log này tôi có thể lọc hoặc đếm request theo `event` và `user_id`; đồng
> thời có thể cộng `cost_usd`, token theo thời gian để làm dashboard/cảnh báo.
> Chuỗi `print("đã trả lời xong")` không có các trường ổn định để máy truy vấn.

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
| 1 stage (bản đầu) | 1.7 GB (~1700 MB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi đo cùng cột `DISK USAGE` của `docker images`: bản 1-stage là 1.7 GB,
> còn bản multi-stage là 271 MB, dưới yêu cầu 500 MB. Bản đầu dùng base
> `python:3.11` đầy đủ nên chứa nhiều thư viện hệ điều hành và công cụ không cần
> cho runtime. Bản multi-stage dùng `python:3.11-slim`; stage cuối chỉ nhận
> dependency từ `/install` cùng `app/` và `utils/`, không mang cache pip, Git,
> test, screenshot hay công cụ chỉ cần lúc build. Docker Desktop còn hiển thị
> `CONTENT SIZE` là kích thước nội dung nén/chia sẻ layer; để so sánh hai image
> tôi dùng nhất quán `DISK USAGE` là 1.7 GB và 271 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt` và
> `RUN pip install` vẫn dùng cache vì requirements không đổi. Docker chỉ chạy
> lại layer `COPY app ./app` và các layer phía sau nó. Nếu `COPY . .` nằm trước
> `RUN pip install`, mọi thay đổi source làm layer COPY đổi, khiến pip cài lại
> toàn bộ dependency dù `requirements.txt` không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công có thể chạy
> lệnh trong container. Khi process chạy root, lệnh đó có UID 0 trong container;
> kết hợp cấu hình nguy hiểm như mount socket/volume hoặc lỗ hổng runtime, họ có
> thể sửa dữ liệu hay leo thang sang host. `USER agent` hạ quyền process trước
> khi Uvicorn chạy, nên bước “thực thi lệnh trong app” chỉ nhận quyền của user
> thường và giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong khoảng 2 giây: gửi 10 request ngay trước lúc phút
> hiện tại kết thúc, ví dụ `10:00:59`, rồi gửi tiếp 10 request ngay sau khi bộ
> đếm reset ở `10:01:00`. Mỗi phút đồng hồ vẫn chỉ ghi nhận 10 request, nhưng
> thực tế 20 request tập trung sát nhau. Sliding window 60 giây vẫn nhìn thấy
> cả hai nhóm nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây, còn cost guard giới hạn tổng
> tiền của từng user trong tháng UTC. Một request rất dài và đắt vẫn có thể nằm
> trong hạn mức 10 request/phút nhưng bị cost guard chặn vì ngân sách tháng đã
> gần hết. Ngược lại, user còn nguyên ngân sách nhưng gửi request thứ 11 trong
> một phút sẽ bị rate limiter chặn dù tổng chi phí vẫn rất thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint liveness cũng ping Redis, khi Redis mất kết nối thì cả ba
> container cùng trả 503. Orchestrator hiểu nhầm cả ba process bị hỏng và lần
> lượt restart chúng. Các container mới vẫn không ping được Redis nên tiếp tục
> bị đánh dấu lỗi và tạo vòng lặp restart, trong khi restart app không sửa được
> Redis. Tách `/health` giúp process vẫn được coi là sống; `/ready` trả 503 để
> load balancer tạm ngừng gửi traffic cho tới khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, dù request được chuyển qua ba container,
> `history_length` vẫn tăng nhất quán theo từng cặp message: 0, 2, 4, 6... cho
> cùng `X-User-Id`. Nếu dùng dict Python, mỗi container có lịch sử riêng nên
> con số có thể nhảy hoặc lùi, ví dụ 0, 0, 2, 0, 2 tùy request rơi vào instance
> nào; restart một instance còn làm phần lịch sử của instance đó mất hẳn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp khi kiểm tra bản deploy là PowerShell báo
> `ParserError: The '<' operator is reserved for future use`. Tôi đã dán nguyên
> lệnh Bash có placeholder `<URL>` và dấu `\` xuống dòng vào PowerShell. Tôi
> nhận ra nguyên nhân từ vị trí parser chỉ vào ký tự `<`, sau đó thay placeholder
> bằng URL Render thật và dùng `Invoke-RestMethod` với cú pháp PowerShell. Kết
> quả sau khi sửa là `/health` và `/ready` trả 200, `/ask` không key trả 401,
> còn `/ask` với key hợp lệ trả 200.
