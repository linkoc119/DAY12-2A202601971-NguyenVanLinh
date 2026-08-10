# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng giữ chỗ bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Linh  Mã học viên: 2A202601971

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là lúc deploy lên cloud nhưng quên tạo biến `AGENT_API_KEY`.
> Nếu có mặc định `"changeme"`, service vẫn báo healthy và người ngoài có thể
> đoán khóa này để gọi `/ask`, làm phát sinh chi phí. Với trường bắt buộc, tiến
> trình dừng ngay bằng `ValidationError`; lỗi xuất hiện trong build/deploy log
> trước khi service nhận traffic nên tôi sửa được cấu hình trước khi công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật tôi lấy từ container là:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T02:55:29.528836+00:00", "user_id": "rate-check", "tokens_in": 392, "tokens_out": 43, "cost_usd": 8.46e-05}
> ```
>
> Từ log có cấu trúc này, tôi có thể (1) lọc/nhóm theo `user_id` rồi cộng
> `cost_usd` để tìm người dùng tiêu nhiều nhất và (2) đếm sự kiện theo khoảng
> thời gian hoặc `level` để tạo biểu đồ, cảnh báo. Chuỗi `đã trả lời xong` không
> chứa user, timestamp, token hay chi phí nên không làm được hai việc đó.

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
| 1 stage (bản đầu) | 1.69 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build thật hai image bằng Docker Desktop. Bản một stage dùng
> `python:3.11` đầy đủ nên mang theo nhiều gói hệ điều hành, công cụ và dữ liệu
> của base image; dependency cũng được cài thẳng vào chính image chạy. Bản
> multi-stage dùng `python:3.11-slim` và chỉ chép kết quả cài Python từ builder
> sang runtime. Vì vậy các thành phần phục vụ build và phần dư của base image
> không đi vào runtime, giảm khoảng 1.42 GB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi chỉ thêm một comment trong `app/main.py` rồi build lại, Docker báo
> `CACHED` cho `COPY requirements.txt`, `RUN pip install` và
> `COPY --from=builder`; chỉ `COPY app ./app`, các layer phía sau và bước export
> phải chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi source sẽ
> làm layer copy đổi, kéo theo việc vô hiệu hóa cache và cài lại toàn bộ thư
> viện dù `requirements.txt` không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗ hổng Python cho phép thực thi lệnh trong container; tiến
> trình đang là root nên kẻ tấn công có toàn quyền với filesystem, process và
> capability được cấp trong container; nếu runtime/kernel có lỗ hổng hoặc host
> mount/capability bị cấu hình quá rộng, quyền đó có thể bị dùng để tác động tới
> host. `USER appuser` cắt chuỗi ngay sau bước chiếm quyền thực thi: mã độc chỉ
> có UID 10001 với quyền hạn chế trong container. Nó không thay thế việc vá lỗi
> hay cấu hình sandbox nhưng giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa là 20 request: gửi 10 request trong giây cuối của phút thứ nhất
> (ví dụ 10:00:59), bộ đếm reset ở 10:01:00, rồi gửi thêm 10 request trong giây
> đầu của phút mới (10:01:00–10:01:01). Fixed window coi chúng thuộc hai phút
> khác nhau dù thực tế cả 20 request nằm trong khoảng hai giây; sliding window
> 60 giây vẫn nhìn thấy nhóm cũ nên chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
> tổng tiền đã tiêu trong cả tháng. Ví dụ user mới gửi một request rất dài khi
> tài khoản đã tiêu 9.999 USD trên ngân sách 10 USD: rate limit còn quota nhưng
> cost guard phải chặn. Ngược lại, user còn nguyên ngân sách nhưng gửi request
> thứ 11 trong vòng 60 giây: cost guard vẫn cho phép về tiền, còn rate limiter
> trả 429 để chặn burst traffic.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp kiểm tra Redis và cả ba container cùng trả
> 503 → orchestrator hiểu nhầm rằng ba process đã chết → cả ba bị restart gần
> như cùng lúc → trong lúc Redis vẫn lỗi, container mới tiếp tục fail probe và
> có thể rơi vào vòng lặp restart → hệ thống không còn instance nào phục vụ kể
> cả endpoint không cần Redis. Tách `/health` và `/ready` giúp process vẫn sống;
> load balancer chỉ tạm rút instance khỏi traffic và đưa lại khi Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, mỗi request thành công thêm hai message nên `history_length` trước
> các lượt hỏi tăng đều 0, 2, 4, 6...; các `ConversationStore` khác nhau cùng
> đọc một key nên vẫn thấy dữ liệu chung. Nếu dùng dict Python, mỗi container có
> một bản sao riêng. Khi load balancer phân phối request qua ba container, số
> quan sát được có thể nhảy kiểu 0, 0, 2, 0, 2... tùy container nhận request,
> thay vì tăng đơn điệu; restart container còn làm lịch sử của nó về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu trên Railway build và khởi tạo container thành công nhưng báo
> `Deployment failed during network process` và `Healthcheck failure`. Tôi mở
> luồng deploy, thấy lỗi nằm ở bước `Network > Healthcheck` chứ không phải build,
> rồi đối chiếu start command. `railway.toml` đã override Docker `CMD` bằng lệnh
> có `--port $PORT`; với image Docker, Railway chạy override dạng exec nên biến
> không được shell mở rộng. Tôi xóa `startCommand` để Railway dùng lại `CMD`
> trong Dockerfile vốn đã bọc `sh -c` và bind `0.0.0.0`. Sau khi commit, push và
> redeploy, cả `/health` và `/ready` trên Public URL đều trả 200.
