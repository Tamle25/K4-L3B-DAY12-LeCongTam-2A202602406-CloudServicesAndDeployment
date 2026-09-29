# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay dòng placeholder mẫu bằng câu trả lời của bạn.
>
> Họ và tên: Lê Công Tâm  Mã học viên: 2A202602406

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi triển khai dịch vụ lên cloud (như Railway hoặc Render), nếu người lập trình quên khai báo biến môi trường `AGENT_API_KEY` trong dashboard cấu hình. Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và báo trạng thái "healthy", dẫn đến một lỗ hổng bảo mật nghiêm trọng: bất kỳ ai trên internet đều có thể dùng key `"changeme"` để gọi `/ask`, tiêu tốn hạn mức và tài nguyên mô hình của dịch vụ mà chủ sở hữu không hay biết. Ngược lại, nhờ không có giá trị mặc định, Pydantic sẽ ném ngoại lệ `ValidationError` và crash ngay lúc khởi động (Fail Fast). Điều này buộc developer phải cấu hình đúng API key bí mật trước khi service có thể đi vào hoạt động và nhận request.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:49:15.123456+00:00", "user_id": "sv01", "cost_usd": 0.0001}
```

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn và phân tích tập trung dễ dàng**: Các hệ thống gom log đám mây (như Datadog, Grafana Loki, CloudWatch) có thể tự động parse cấu trúc các trường JSON. Từ đó có thể lọc chính xác request theo từng `user_id`, tính tổng chi phí (`cost_usd`) bằng các truy vấn aggregation mà không cần viết regex bóc tách chuỗi phức tạp và dễ gãy.
2. **Thiết lập cảnh báo tự động chuẩn xác (Automated Alerting)**: Có thể dễ dàng đặt rule giám sát tự động kích hoạt cảnh báo khi xuất hiện `"level": "error"` hoặc khi `"cost_usd"` của một request tăng bất thường, giúp hệ thống phát hiện sự cố nhanh chóng.

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
| 1 stage (bản đầu: python:3.11 full) | ~1020 MB |
| Multi-stage (python:3.11-slim) | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
1. **Trình biên dịch và công cụ phát triển thừa**: Base image đầy đủ `python:3.11` tích hợp toàn bộ bộ công cụ build C/C++ (`gcc`, `g++`, `make`), các file header thư viện (`python3-dev`, `libc-dev`) và các gói tiện ích mở rộng của hệ điều hành. Trong khi đó, stage runtime của bản multi-stage dùng `python:3.11-slim` chỉ chứa môi trường runtime tối thiểu.
2. **Bộ nhớ đệm và file tạm (build artifacts / package cache)**: Nhờ dùng `--prefix=/install` ở builder và chỉ copy thư mục cài đặt sang runtime sạch, image cuối không bị lưu giữ cache tải gói của pip (`--no-cache-dir`) hay các file tạm trong quá trình biên dịch.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

1. **Các layer được dùng lại từ cache (CACHED)**:
   - Toàn bộ stage `builder` (`COPY requirements.txt .`, `RUN pip install ...`) được cache 100% vì file requirements không đổi.
   - Trong stage `runtime`: layer base image, `COPY --from=builder /install /usr/local`, và `RUN useradd ...` đều được dùng lại từ cache.
2. **Các layer phải chạy lại**:
   - Chỉ layer `COPY app ./app` (và các bước sau nó như export image) phải chạy lại. Do đó thời gian build lại chỉ mất dưới 1 giây.
3. **Nếu đặt `COPY . .` lên trước `RUN pip install`**:
   - Mỗi lần sửa dù chỉ 1 ký tự trong source code, checksum của thư mục thay đổi làm mất hiệu lực toàn bộ layer cache từ thời điểm `COPY` trở đi (cache busting).
   - Docker sẽ bị ép phải chạy lại lệnh `RUN pip install` và tải/cài đặt lại tất cả thư viện mạng từ đầu mỗi lần build, gây lãng phí băng thông và tốn thêm vài phút cho mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

1. **Chuỗi sự kiện khi chạy bằng root**:
   - Kẻ tấn công khai thác lỗ hổng thực thi mã từ xa (RCE) trong code Python (ví dụ thư viện deserialization, command injection qua input không được lọc).
   - Vì container chạy bằng `root` mặc định, tiến trình Python chạy với UID 0. Kẻ tấn công giành được một interactive shell với đặc quyền `root` (UID 0) bên trong container.
   - Từ quyền root trong container, kẻ tấn công khai thác các lỗ hổng container breakout (lỗ hổng Linux kernel, lỗ hổng runtime của container, hoặc các volume/socket gắn từ host như `/var/run/docker.sock`). Do UID 0 trong container mặc định map với UID 0 của kernel host, kẻ tấn công chiếm quyền kiểm soát hoàn toàn máy chủ host (root host).
2. **Lệnh `USER` cắt đứt chuỗi ở đâu**:
   - Lệnh `USER appuser` (UID 10001) tước bỏ quyền root ngay trong container. Nếu code Python bị RCE, kẻ tấn công chỉ có quyền của user thông thường `appuser`: không thể đọc file hệ thống quan trọng, không truy cập được socket quản trị, và bị chặn hoàn toàn khả năng khai thác các kỹ thuật container breakout yêu cầu quyền root.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa:** **20 request** trong 2 giây.
- **Giải thích cách đạt được:**
  - Nếu đếm theo phút đồng hồ cố định (Fixed Window Counter), bộ đếm tự động reset về 0 ở đầu mỗi phút (giây 00).
  - Người dùng có thể gửi 10 request vào giây cuối cùng của phút thứ nhất (`10:00:59`). Khi đó bộ đếm của phút 10 ghi nhận 10/10 request (vẫn hợp lệ, không bị chặn).
  - Ngay sang giây tiếp theo (`10:01:00`), bộ đếm bước sang phút mới và reset về 0. Người dùng lập tức gửi tiếp 10 request trong giây này.
  - Kết quả: Trong 2 giây liên tiếp từ `10:00:59` đến `10:01:00`, người dùng gửi được $10 + 10 = 20$ request (gấp đôi hạn mức quy định) mà hệ thống fixed window không hề phát hiện. Thuật toán Sliding Window của chúng ta khắc phục triệt để lỗi này bằng cách luôn tính chính xác số request trong cửa sổ 60 giây trượt lùi từ thời điểm gửi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

1. **Khác biệt cốt lõi:**
   - **Rate Limiter:** Kiểm soát **tốc độ / tần suất** request trong một cửa sổ thời gian ngắn (ví dụ: 10 request/phút) để bảo vệ hạ tầng máy chủ khỏi bị quá tải hoặc tấn công từ chối dịch vụ (DoS/Spam).
   - **Cost Guard:** Kiểm soát **ngân sách tài chính thực tế (USD)** trong chu kỳ dài (theo user/tháng) dựa trên lượng token LLM tiêu thụ, giúp bảo vệ ví tiền và tài nguyên API của dự án.
2. **Tình huống Rate limit cho qua nhưng Cost guard chặn (Mã 402):**
   - User chỉ gửi đúng 1 request trong 10 phút (tần suất rất thấp, rate limit cho qua). Nhưng request này kèm một đoạn prompt cực lớn (50k token) ước tính tốn $1.50, trong khi hạn mức còn lại của user trong tháng chỉ còn $0.20 $\rightarrow$ Cost Guard phát hiện vượt ngân sách và lập tức chặn trước khi gọi model.
3. **Tình huống Cost guard cho qua nhưng Rate limit chặn (Mã 429):**
   - User mới tạo tài khoản và còn nguyên $10 ngân sách tháng. User dùng script gửi liên tục 15 request câu hỏi ngắn "Xin chào" chỉ trong vòng 3 giây. Tổng chi phí của 15 câu chỉ khoảng $0.0001 (rất nhỏ so với ngân sách $10), nhưng do gọi quá nhanh vượt 10 request/phút nên từ request thứ 11 đã bị Rate Limiter chặn lại với mã 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. **Redis gặp sự cố / mất kết nối mạng trong 30 giây.**
2. **Endpoint gộp (liveness probe) thất bại đồng loạt:** Cả 3 container khi được orchestrator kiểm tra liveness đều không ping được Redis nên trả về mã lỗi 503 hoặc timeout.
3. **Orchestrator phán đoán sai và khởi động lại toàn bộ cụm:** Vì liveness probe báo hỏng, orchestrator (Kubernetes/Docker/Cloud) lầm tưởng rằng process của container bị deadlock/treo và lập tức gửi tín hiệu tiêu diệt (kill) rồi restart cả 3 container.
4. **Vòng xoáy khởi động lại liên hoàn (Cascading Failure / CrashLoop):** Sau khi restart, các container khởi động lại nhưng Redis vẫn chưa phục hồi trong khoảng thời gian 30s đó, liveness tiếp tục đỏ $\rightarrow$ container tiếp tục bị restart liên tục.
5. **Downtime trầm trọng không đáng có:** Toàn bộ dịch vụ bị sập hoàn toàn. Khi Redis hoạt động trở lại, các container vẫn đang trong tiến trình khởi động lại chậm chạp.
*Kết luận:* Tách riêng `/health` (chỉ kiểm tra process sống) và `/ready` (kiểm tra dependency) giúp load balancer chỉ tạm thời ngừng đẩy traffic vào container khi Redis lỗi, mà không tiêu diệt và restart vô ích các container đang hoạt động bình thường.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

1. **Khi lưu trên Redis (Stateless - thiết kế chuẩn):**
   - Dù mỗi request của cùng một user được load balancer phân phối ngẫu nhiên tới container A, B hay C, cả 3 container đều đọc và ghi lịch sử chung vào Redis. Do đó `history_length` tăng đều đặn và liên tục: 0 $\rightarrow$ 2 $\rightarrow$ 4 $\rightarrow$ 6 $\rightarrow$ 8... Ngữ cảnh hội thoại được bảo toàn 100%.
2. **Nếu lưu trong dict Python (Stateful - lưu trong RAM process):**
   - Mỗi container sở hữu một bộ nhớ RAM hoàn toàn cô lập. Khi gọi `/ask` nhiều lần:
     - Lần 1 rơi vào Container 1: `history_length` = 0 (lưu câu hỏi 1 vào RAM của Container 1).
     - Lần 2 rơi vào Container 2: Container 2 không hề có dữ liệu của câu hỏi 1 $\rightarrow$ `history_length` lại quay về 0 (agent bị "mất trí nhớ").
     - Lần 3 rơi vào Container 3: `history_length` tiếp tục bằng 0.
     - Lần 4 quay vòng lại Container 1: `history_length` bất ngờ nhảy lên 2.
   - Kết quả: `history_length` nhảy hỗn loạn tùy thuộc vào thuật toán cân bằng tải (Round-robin), cuộc trò chuyện bị đứt quãng và dịch vụ hoàn toàn không thể scale ngang (scale out).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:** 
  `Deploy failed: Health check timeout / Service failed to respond on port` và `ConnectionRefusedError: [Errno 111] Connect call failed ('127.0.0.1', 6379)` khi kiểm tra readiness.
- **Cách tìm ra nguyên nhân:** 
  - Đọc Runtime Logs và Deploy Logs trên dashboard của nền tảng Cloud (Render/Railway). 
  - Phát hiện 2 nguyên nhân: (1) Nền tảng cloud tự cấp một cổng ngẫu nhiên qua biến môi trường `$PORT` (ví dụ cổng `10000`), nhưng nếu app bind cố định cổng `8000` thì load balancer ngoài không chuyển tiếp traffic tới được; (2) Service agent cố kết nối tới `redis://localhost:6379/0` (trong môi trường container cloud, `localhost` trỏ vào chính container agent chứ không phải container Redis).
- **Cách sửa chữa:**
  1. Trong `Dockerfile`, dùng shell expansion để đọc cổng động: `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.
  2. Trên dashboard của nền tảng Cloud, tạo một Redis instance (hoặc add-on), lấy connection string thật và khai báo vào biến môi trường `REDIS_URL` (ví dụ `redis://default:...@roundhouse.proxy.rlwy.net:PORT/0` hoặc `redis://redis:6379/0` trong mạng nội bộ). Đồng thời cấu hình đầy đủ `AGENT_API_KEY`.
