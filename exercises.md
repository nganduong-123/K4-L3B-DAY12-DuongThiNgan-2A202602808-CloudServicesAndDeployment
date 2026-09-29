# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng mẫu in nghiêng bằng câu trả lời của chính bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Dương Thị Ngân  Mã học viên: 2A202602808

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên Render nhưng tôi quên tạo biến
> `AGENT_API_KEY`. Nếu code có khóa mặc định `"changeme"`, service vẫn báo
> khỏe và người ngoài có thể đoán khóa để gọi `/ask`, làm phát sinh chi phí.
> Với cấu hình hiện tại, Pydantic dừng ứng dụng ngay bằng `ValidationError`;
> tôi thấy lỗi trong deploy log và sửa cấu hình trước khi URL nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi nhận được khi gọi service local là:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T03:19:30.994871+00:00","user_id":"ngan-local-check","tokens_in":7,"tokens_out":39,"cost_usd":2.445e-05}`.
> Từ JSON này tôi có thể (1) lọc và cộng chi phí theo `user_id`, và (2) dựng
> cảnh báo theo số request/lỗi trong một khoảng thời gian dựa trên `timestamp`
> và `level`. Dòng `print("đã trả lời xong")` không chứa đủ trường để làm hai
> việc đó một cách tự động.

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
| 1 stage (bản đầu) | 1.7 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build thật bằng Docker Desktop ngày 29/09/2026. Bản một stage dùng
> `python:3.11` đầy đủ nên đạt 1.7 GB, còn bản multi-stage dùng
> `python:3.11-slim` và chỉ copy thư viện runtime nên còn 271 MB. Phần chênh
> chủ yếu là base Debian đầy đủ, công cụ build, cache và các thành phần chỉ cần
> trong lúc cài package chứ không cần khi chạy FastAPI.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Docker tái sử dụng các layer base image, `COPY requirements.txt` và
> `pip install` vì dependency không đổi. Từ layer `COPY app ./app` trở xuống
> phải tạo lại, gồm copy source và bước tạo/chown user phía sau. Nếu đặt
> `COPY . .` trước `pip install`, chỉ sửa một ký tự trong source cũng làm mất
> cache của layer cài dependency, khiến pip tải và cài lại toàn bộ package.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu API có lỗi cho phép thực thi lệnh, mã độc trước tiên chạy với UID của
> process trong container. Khi process là root, kẻ tấn công có quyền sửa mọi
> file trong container và có cơ hội khai thác mount hoặc lỗi container runtime
> để tác động host với quyền cao. Lệnh `USER agent` cắt chuỗi ngay tại bước
> đầu: payload chỉ có quyền của tài khoản không đặc quyền, nên phạm vi ghi file
> và thao tác hệ thống bị giới hạn mạnh.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong 2 giây: gửi 10 request ở cuối phút cũ,
> ví dụ 10:00:59, rồi thêm 10 request ngay đầu phút mới, 10:01:00. Bộ đếm theo
> phút đã reset dù 20 request nằm sát nhau. Sliding window nhìn lại đúng 60
> giây nên vẫn thấy 10 request cũ và chặn đợt tiếp theo.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất request trong cửa sổ 60 giây; cost guard giới
> hạn tổng tiền của từng user theo tháng. Một user gửi ít request nhưng mỗi
> request có prompt/lịch sử rất dài có thể qua rate limit nhưng bị cost guard
> chặn. Ngược lại, user còn gần như toàn bộ ngân sách nhưng gửi request thứ 11
> trong một phút sẽ bị rate limit chặn dù cost guard vẫn cho phép.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả lỗi. Orchestrator hiểu nhầm process
> chết và lần lượt restart cả ba container. Container mới vẫn kiểm tra Redis
> đang lỗi nên tiếp tục fail rồi bị restart, tạo restart loop và làm toàn bộ
> dịch vụ gián đoạn. Tách riêng giúp `/health` vẫn 200 vì process còn sống,
> còn `/ready` trả 503 để load balancer tạm rút instance khỏi traffic; khi
> Redis trở lại, instance sẵn sàng ngay mà không cần restart cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi thử local, request đầu trả `history_length=0`; request sau cùng user thấy
> lịch sử đã tăng thêm hai message (user và assistant). Hai instance
> `ConversationStore` trong test cũng đọc được cùng dữ liệu vì key nằm ở Redis.
> Nếu dùng dict Python, mỗi container có một dict riêng: request bị chuyển sang
> instance khác sẽ thấy lịch sử quay về 0 hoặc một số cũ, nên con số tăng giảm
> thất thường thay vì tăng nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lúc đầu tôi chọn Railway vì CLI trên máy đã đăng nhập, nhưng lệnh tạo project
> trả thông báo `Your trial has expired. Please select a plan to continue using
> Railway.`. Tôi xác nhận đây là giới hạn tài khoản chứ không phải lỗi code vì
> project chưa được tạo và CLI không link được thư mục. Tôi chuyển sang phương
> án Render được bài cho phép, deploy `render.yaml` bằng Blueprint, nhập secret
> trong dashboard và nối `REDIS_URL` tới Render Key Value. Sau khi deploy,
> `/health` và `/ready` đều trả 200, còn `/ask` không có key trả đúng 401.
