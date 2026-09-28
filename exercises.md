# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thu Hằng  Mã học viên: 2A202602463

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy service lên môi trường production nhưng đội ngũ vận hành quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard.
- Nếu để giá trị mặc định là `"changeme"`: Service vẫn khởi động bình thường và vượt qua health check liveness. Tuy nhiên, bất kỳ ai cũng có thể dùng key mặc định `"changeme"` để gọi API làm rò rỉ dữ liệu hoặc lạm dụng tài nguyên LLM, đồng thời các client thật gửi API key riêng sẽ bị lỗi xác thực 401 mà hệ thống giám sát không phát hiện ngay lúc triển khai.
- Khi "chết sớm" (fail fast): App sẽ ném `ValidationError` ngay lúc nạp cấu hình và container crash ngay lập tức. Hệ thống điều phối (như Kubernetes/Render) sẽ báo lỗi deployment thất bại, đội ngũ phát triển phát hiện ngay nguyên nhân thiếu cấu hình qua log khởi động để bổ sung trước khi có người dùng truy cập.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"timestamp": "2026-09-28T17:41:45.120Z", "level": "INFO", "message": "Request processed", "user_id": "sv-test", "path": "/ask", "status_code": 200, "duration_ms": 142.5, "cost_usd": 0.002}
```

Hai việc làm được với log JSON mà `print()` thông thường không làm được:
1. **Truy vấn và lọc có cấu trúc (Structured Querying):** Các hệ thống thu thập log tập trung (Datadog, ELK, CloudWatch) có thể tự động parse các trường (field-level parsing), cho phép tìm kiếm nhanh các request có `status_code >= 400` hoặc lọc toàn bộ log của riêng một `user_id` cụ thể mà không cần viết regex phức tạp.
2. **Tổng hợp số liệu và cảnh báo tự động (Aggregation & Alerting):** Có thể chạy các phép tính thống kê trực tiếp trên các trường dữ liệu số như tính độ trễ trung bình (`avg(duration_ms)`), tính tổng chi phí LLM (`sum(cost_usd)` theo giờ/ngày), hoặc thiết lập ngưỡng cảnh báo (alert) tự động khi tỉ lệ lỗi vượt quá 5%.

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
| 1 stage (bản đầu) | 1050 MB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~865 MB) bao gồm:
- Trình biên dịch C/C++ (`gcc`, `g++`, `build-essential`) và các công cụ phát triển cần cho việc biên dịch package Python nhưng không cần ở runtime.
- File header và thư viện dev (`python3-dev`, `libc6-dev`).
- Cache tạm thời của trình quản lý gói hệ điều hành (`apt lists cache`) và cache tải về của Python (`pip cache`).
- Các file tài liệu hướng dẫn (man pages, doc) đi kèm các package hệ thống được cài ở stage build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại (đặt `COPY requirements.txt .` và `RUN pip install` trước `COPY app ./app`):
  - Các layer phía trước (`FROM`, `useradd`, `COPY requirements.txt .`, `RUN pip install`) đều được dùng lại từ cache (CACHED) vì file `requirements.txt` không thay đổi.
  - Chỉ các layer từ `COPY app ./app` trở đi mới bị huỷ cache và phải chạy lại, giúp thời gian build chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa một ký tự trong mã nguồn, layer `COPY . .` thay đổi checksum làm Docker huỷ toàn bộ cache từ layer đó trở đi.
  - Kết quả là lệnh `RUN pip install` sẽ bị chạy lại từ đầu, tải và cài đặt lại toàn bộ thư viện mỗi lần sửa code, khiến thời gian build kéo dài từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công phát hiện và khai thác một lỗ hổng trong code Python (ví dụ: Remote Code Execution qua insecure deserialization hoặc command injection).
2. Lệnh độc hại được thực thi với quyền của process hiện tại trong container. Nếu không chỉ định `USER`, process chạy với quyền `root` (UID 0).
3. Do sở hữu quyền `root`, kẻ tấn công có thể chỉnh sửa bất kỳ file nào trong container, can thiệp vào tài nguyên hệ thống, và nếu có volume mount hoặc lỗ hổng container escape (như kernel exploit, misconfigured capabilities, Docker socket), kẻ tấn công thoát khỏi container và chiếm quyền root trực tiếp trên máy chủ host.
4. Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ bước 2: Process Python chỉ chạy với quyền user thường (`UID 10001`). Khi đó, kẻ tấn công bị giới hạn nghiêm ngặt, không thể ghi đè file hệ thống, không cài thêm phần mềm độc hại và không đủ đặc quyền để thực hiện các bước leo thang đặc quyền ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: **20 request** trong 2 giây liên tiếp.
- Giải thích:
  - Với cơ chế đếm theo phút đồng hồ (fixed window), bộ đếm được reset về 0 vào mỗi đầu phút (giây 00).
  - Kẻ tấn công gửi 10 request vào giây `59` của phút thứ nhất (hệ thống cho phép vì vẫn trong quota 10 của phút đó).
  - Ngay ở giây tiếp theo (giây `00` của phút thứ hai), bộ đếm bị reset về 0, kẻ tấn công lập tức gửi thêm 10 request nữa.
  - Tổng cộng trong khoảng thời gian chỉ 2 giây (từ giây 59 đến giây 00), hệ thống phải hứng chịu tới 20 request (gấp đôi hạn mức cho phép). Sliding window khắc phục điều này bằng cách luôn tính chính xác tổng số request trong cửa sổ 60 giây trôi qua tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác biệt:**
  - `Rate limiter`: Giới hạn **tần suất (tốc độ)** gọi API trong khoảng thời gian ngắn (ví dụ: 10 request / 60 giây), mục đích bảo vệ hạ tầng máy chủ khỏi bị quá tải, nghẽn mạng và tấn công từ chối dịch vụ (DoS).
  - `Cost guard`: Giới hạn **tổng chi phí tài chính tích lũy** trong chu kỳ dài (ví dụ: $10.0 / tháng), mục đích bảo vệ ngân sách chi trả cho tài nguyên LLM/dịch vụ ngoài.
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:**
  - User gọi request đầu tiên trong ngày (tốc độ gọi rất chậm, không vi phạm 10 req/phút), nhưng trong tháng user này đã tiêu lũy kế đạt $10.01 (vượt mức $10.0). Rate limiter cho qua, nhưng Cost guard chặn lại và trả về mã `402 Payment Required`.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:**
  - User mới tạo tài khoản, ngân sách tháng còn nguyên $10.0 (chưa tiêu đồng nào), nhưng dùng bot gửi liên tiếp 15 request chỉ trong vòng 3 giây. Tổng chi phí của 15 request này chỉ khoảng $0.03 (chưa vượt ngân sách tháng), nhưng do vượt quá 10 req/phút nên Rate limiter lập tức chặn lại và trả về mã `429 Too Many Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. **Redis gặp sự cố hoặc gián đoạn mạng trong 30 giây.**
2. **Container kiểm tra liveness thất bại:** Endpoint liveness gọi `ping()` tới Redis và trả về mã lỗi hoặc timeout.
3. **Orchestrator (Docker/Kubernetes) hiểu nhầm container bị treo (deadlock):** Khi liveness probe thất bại liên tiếp quá số lần quy định, orchestrator kết luận process container bị hỏng và phát lệnh kill để restart cả 3 container.
4. **Vòng lặp CrashLoopBackOff và cascading failure:** Khi cả 3 container khởi động lại, chúng lại kiểm tra Redis ngay lúc khởi động, tiếp tục fail và lại bị kill tiếp. Việc toàn bộ cụm container bị restart đồng loạt làm rớt toàn bộ các request đang phục vụ dở dang, tạo tải dồn dập và có thể làm sập luôn Redis khi nó vừa hồi phục.
5. **Cách đúng:** Tách riêng `/health` (liveness: chỉ kiểm tra process Python còn sống) và `/ready` (readiness: kiểm tra Redis; nếu Redis chết thì chỉ tạm ngắt traffic khỏi container chứ không restart container).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi dùng Redis (Stateless): Dù load balancer điều phối các request lần lượt vào ngẫu nhiên container A, B hay C, cả 3 container đều đọc/ghi vào cùng một Redis database tập trung, do đó `history_length` tăng đều đặn và chính xác: `1 -> 2 -> 3 -> 4 -> 5`.
- Nếu lưu trong dict Python trong bộ nhớ RAM của container (Stateful):
  - Mỗi container có một vùng nhớ riêng rẽ. Khi gửi 5 request được load balancer phân phối round-robin qua 3 container (A, B, C):
    - Request 1 vào A: `history_length` = 1.
    - Request 2 vào B: B chưa từng thấy user này nên `history_length` = 1 (bị "mất trí nhớ").
    - Request 3 vào C: `history_length` = 1.
    - Request 4 vào A: `history_length` = 2.
  - Con số `history_length` sẽ nhảy thất thường (1, 1, 1, 2, 2,...) tùy thuộc vào request rơi vào container nào, khiến ngữ cảnh cuộc trò chuyện của user bị đứt quãng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Lỗi kết nối Redis khi deploy lên cloud Render khiến readiness probe không sẵn sàng (`/ready` trả về 503).
- **Thông báo lỗi & Dấu hiệu:** Gọi `/health` trả về 200 OK nhưng gọi `/ready` trả về `503 {"status": "not ready", "redis": false}`.
- **Cách tìm ra nguyên nhân:** Mở tab **Logs** trên Render dashboard của service `day12-agent`, phát hiện biến môi trường `REDIS_URL` chưa nhận được connection string từ service `day12-redis` do cấu hình Blueprint chưa link đúng thuộc tính.
- **Cách khắc phục:** Cấu hình đúng `fromService` với `name: day12-redis`, `type: redis`, `property: connectionString` trong file `render.yaml` để Render tự động truyền chuỗi kết nối nội bộ vào biến `REDIS_URL` của web service, sau đó deploy lại. Khi đó service kết nối Redis thành công và `/ready` trả về `200 {"status": "ready", "redis": true}`.
