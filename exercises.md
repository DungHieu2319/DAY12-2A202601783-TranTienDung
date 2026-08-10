# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder (bắt đầu bằng `>` in nghiêng) bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Tiến Dũng  Mã học viên: 2A202601783

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy CP5 lên Railway, tôi phải set 5-6 biến môi trường trong dashboard
> (`AGENT_API_KEY`, `REDIS_URL`, `RATE_LIMIT_PER_MINUTE`...). Giả sử tôi quên
> set `AGENT_API_KEY` — nếu field này có mặc định kiểu `"changeme"`, app vẫn
> khởi động bình thường, `/health` vẫn trả 200, deploy vẫn "thành công". Tôi sẽ
> không phát hiện ra gì bất thường cho tới khi có ai đó gọi `/ask` bằng đúng
> chuỗi `"changeme"` (một giá trị ai cũng đoán được vì nó nằm trong chính mã
> nguồn công khai trên GitHub) và xài miễn phí LLM call trả bằng tiền của tôi.
> Vì `agent_api_key: str` không có default, Pydantic ném `ValidationError` ngay
> lúc `Settings()` được khởi tạo — app crash, Railway healthcheck fail, deploy
> đứng ở trạng thái "Failed" rõ ràng. Tôi buộc phải vào set đúng biến trước khi
> service chạy được, thay vì phát hiện ra lỗ hổng bảo mật vài tuần sau khi xem
> hoá đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Gọi thật vào bản deploy trên Railway, log ghi lại (qua `log_event` trong
> `app/main.py`) trông như sau:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:17:05.123456+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc/truy vấn theo trường** — vì đây là JSON có cấu trúc, tôi có thể hỏi
>    thẳng "tổng `cost_usd` của `user_id=sv-test` trong tháng 8 là bao nhiêu"
>    bằng một câu lệnh `jq` hoặc query trên Datadog/CloudWatch. Với
>    `print("đã trả lời xong")`, thông tin `user_id`, `cost_usd` không tồn tại
>    dưới dạng dữ liệu — chỉ là chữ cho người đọc, máy không tách được.
> 2. **Cảnh báo tự động** — vì `level` và `cost_usd` là field riêng, tôi có thể
>    đặt alert kiểu "báo tôi nếu tổng `cost_usd` một `user_id` vượt X trong 1
>    giờ" hoặc "báo nếu `level=error` xuất hiện quá N lần/phút". Chuỗi tự do
>    thì công cụ giám sát không có gì để so sánh ngưỡng.

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
| 1 stage (bản đầu, `python:3.11` đầy đủ) | 1730 MB |
| Multi-stage (`python:3.11-slim`, builder + runtime) | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ≈ 1.46GB, đến từ hai chỗ:
>
> 1. **Base image.** `python:3.11` (bản đầy đủ) mang theo toàn bộ toolchain
>    build (gcc, make, header files, các thư viện dev...) và nhiều gói OS
>    không cần thiết lúc chạy app — chỉ cần thiết lúc *build* một số Python
>    package có phần mở rộng C. `python:3.11-slim` bỏ hết phần này, chỉ giữ
>    runtime Python tối thiểu.
> 2. **Multi-stage tách build khỏi runtime.** Ở bản multi-stage, stage
>    `builder` cài `pip install` (có thể cần compiler cho một số wheel) nhưng
>    stage đó bị **bỏ lại hoàn toàn** sau khi build xong — stage runtime chỉ
>    `COPY --from=builder` đúng thư mục `~/.local` chứa package đã cài, không
>    mang theo pip cache, compiler tạm, hay source code trung gian nào.
>
> Nói cách khác: bản 1-stage mang cả "xưởng sản xuất" đi theo thành phẩm, bản
> multi-stage chỉ mang đúng thành phẩm.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi: stage `builder` làm `COPY requirements.txt .` rồi
> `RUN pip install --user` **trước**, sau đó mới sang stage runtime `COPY . .`
> để lấy toàn bộ source code. Docker build theo layer, mỗi `COPY`/`RUN` là một
> layer, và Docker cache lại layer nếu input của nó (file được copy, hoặc lệnh
> chạy) không đổi so với lần build trước.
>
> Khi tôi sửa 1 ký tự trong `app/main.py` rồi build lại:
> - Layer `COPY requirements.txt .` và `RUN pip install --user ...` ở stage
>   `builder` **được dùng lại từ cache** — vì `requirements.txt` không đổi,
>   Docker không cần cài lại toàn bộ dependency (việc này tốn nhiều giây tới
>   cả phút).
> - Layer `COPY . .` ở stage runtime **phải chạy lại** — vì nội dung thư mục
>   đã đổi (đúng 1 file), Docker phát hiện checksum khác nên invalidate layer
>   này và mọi layer sau nó.
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install` (gộp về 1 stage như bản
> gốc lab đưa): mỗi lần sửa dù chỉ 1 ký tự bất kỳ trong code, layer `COPY . .`
> bị invalidate → kéo theo layer `RUN pip install` ngay sau nó **cũng bị chạy
> lại từ đầu**, dù `requirements.txt` không hề đổi. Build sẽ chậm hơn nhiều
> lần trong vòng lặp code → build → test lặp đi lặp lại lúc phát triển.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python của tôi có lỗ hổng — ví dụ một dependency có
> lỗi remote code execution, hoặc tôi lỡ để endpoint nào đó thực thi lệnh
> không kiểm soát input; (2) kẻ tấn công khai thác lỗ hổng đó, chạy được lệnh
> tuỳ ý **bên trong container**; (3) nếu process app đang chạy bằng **root**
> trong container, lệnh tuỳ ý đó cũng chạy với quyền root — kẻ tấn công đọc
> ghi được mọi file trong container, cài thêm phần mềm độc hại; (4) nếu thêm
> một lỗ hổng container-escape (kernel bug, cấu hình Docker socket bị mount
> nhầm, misconfiguration...), quyền root *trong container* dễ leo thang thành
> quyền root *trên máy host* hơn nhiều so với quyền của một user thường —
> nhiều kỹ thuật escape container giả định sẵn tiến trình đang chạy root.
>
> Lệnh `USER appuser` (tôi thêm trong `Dockerfile`, chuyển sang user thường
> uid 1000 trước khi `CMD` chạy) cắt đứt chuỗi này ở bước (3): dù kẻ tấn công
> khai thác được lỗ hổng ở bước (1)-(2), lệnh họ chạy được cũng chỉ có quyền
> của `appuser` — không tự ý cài phần mềm hệ thống, không đổi được file thuộc
> sở hữu root, và bề mặt tấn công để leo thang lên host bị thu hẹp đáng kể so
> với việc họ có sẵn quyền root ngay từ đầu.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đúng 10 request vào
> giây `:59` của phút hiện tại (dùng hết hạn mức 10/phút của "phút này" —
> hợp lệ, bộ đếm chưa vượt), rồi gửi tiếp 10 request vào giây `:01` của phút
> kế tiếp (bộ đếm vừa reset về 0 lúc `:00`, nên 10 request này cũng hợp lệ
> theo đúng luật 10/phút). Tổng cộng 20 request lọt qua trong cửa sổ 2 giây
> thực tế (`:59` → `:01`), dù cả hai lần đều "đúng luật" theo kiểu đếm-theo-
> phút-đồng-hồ. Đây chính xác là lỗ hổng mà cách đếm sliding window (ZSET
> trong `app/rate_limiter.py`, luôn nhìn lại đúng 60 giây gần nhất tính từ
> thời điểm request tới, không neo theo mốc giây `:00`) không mắc phải.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác biệt cốt lõi: **rate limit đếm số lượng request** trong một cửa sổ
> thời gian (bảo vệ khỏi spam/DoS), còn **cost guard đếm số tiền** đã tiêu
> trong một tháng (bảo vệ ví tiền). Một request rẻ và một request đắt bị rate
> limit đối xử như nhau, nhưng cost guard thì không.
>
> - **Rate limit cho qua, cost guard chặn**: user gửi rất ít request trong
>   phút này (ví dụ 1 request, còn nguyên hạn mức 10/phút) nên `limiter.check`
>   pass — nhưng user đó đã tích luỹ chi tiêu gần chạm/vượt
>   `MONTHLY_BUDGET_USD` từ những ngày trước (ví dụ hỏi những câu rất dài,
>   history dài kéo theo nhiều token). `guard.check` chặn với `402`.
> - **Cost guard cho qua, rate limit chặn**: user hỏi những câu rất ngắn, rẻ
>   tiền (chi phí mỗi request gần như 0, `guard.check` luôn pass vì ngân sách
>   còn dư rất nhiều) nhưng gửi liên tục, dồn dập, vượt quá 10 request/phút —
>   `limiter.check` chặn với `429` dù tiền chưa tốn bao nhiêu, đơn giản vì
>   tần suất gọi quá nhanh (có thể là bug ở client, hoặc cố tình spam).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
>
> 1. Redis mất kết nối. Cả 3 container `agent` cùng gọi vào Redis khi xử lý
>    endpoint gộp và đều nhận lỗi/timeout.
> 2. Vì endpoint gộp vừa đóng vai `/health` (liveness) vừa kiểm tra Redis, cả
>    3 container trả về `503` cho probe liveness — không chỉ readiness.
> 3. Orchestrator (Docker/K8s/Railway) hiểu "liveness fail" là **process đã
>    chết/kẹt, cần khởi động lại nó**, chứ không hiểu là "phụ thuộc bên ngoài
>    đang lỗi". Nó bắt đầu **restart cả 3 container** gần như đồng thời.
> 4. Trong lúc cả 3 container đang restart, **không còn container nào phục vụ
>    được traffic** — toàn bộ service down hoàn toàn, dù bản thân code của
>    `agent` hoàn toàn khoẻ mạnh và chỉ có Redis (một dependency ngoài) bị
>    lỗi tạm thời.
> 5. Container mới khởi động lại, nhưng Redis vẫn chưa hồi trong 30 giây đó,
>    nên `/health` (gộp) tiếp tục trả `503` → orchestrator lại restart tiếp →
>    vòng lặp restart liên tục ("crash loop") cho tới khi Redis sống lại.
>
> Nếu tách riêng (đúng như CP1+CP4 tôi đã làm): `/health` không đụng Redis nên
> luôn 200 (process vẫn sống, không bị restart oan), chỉ `/ready` báo `503` →
> load balancer ngừng đẩy traffic **mới** vào container đó nhưng container vẫn
> chạy, sẵn sàng nhận traffic lại ngay khi `/ready` xanh trở lại — không có
> vòng lặp restart, không mất 30 giây downtime hoàn toàn.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với `ConversationStore` lưu trong Redis (bản tôi đã làm ở CP4): mỗi lần gọi
> `/ask` với cùng `X-User-Id`, dù load balancer đẩy request vào container nào
> trong 3 container (`agent=3`), `history_length` vẫn **tăng dần đều đặn**
> (0, 2, 4, 6...) — vì cả 3 container đọc/ghi chung một Redis, không phân
> biệt request tới từ instance nào.
>
> Nếu lịch sử được lưu trong một `dict` Python **trong từng process** thay vì
> Redis: mỗi container có bộ nhớ RAM riêng, không chia sẻ gì với nhau.
> `history_length` sẽ **nhảy lung tung, không tăng đều** — ví dụ request 1 và
> 3 rơi vào container A thấy `history_length` tăng bình thường, nhưng request
> 2 rơi vào container B lại thấy `history_length = 0` (vì container B chưa
> từng thấy user này) rồi bắt đầu đếm lại từ đầu ở riêng nó. Agent trông như
> "mất trí nhớ" ngẫu nhiên tuỳ round-robin của load balancer rơi vào container
> nào — đúng vấn đề mà state ngoài process (Redis) giải quyết.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu tiên lên Railway, deployment fail ở bước **Network →
> Healthcheck** sau đúng 30 giây (đúng `healthcheckTimeout` khai trong
> `railway.toml`), dù build và deploy container báo thành công. Domain public
> trả về `404 Application not found` từ chính edge của Railway — nghĩa là
> chưa có deployment nào sống để route vào.
>
> Cách tìm nguyên nhân: vào tab **Deploy Logs** trên dashboard Railway, đọc
> log stdout/stderr thật của container lúc khởi động. Log hiện rõ:
> ```
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
> lặp lại liên tục — app crash-loop ngay khi khởi động vì `uvicorn` nhận
> nguyên văn chuỗi `"$PORT"` làm giá trị cổng thay vì một số thật.
>
> Nguyên nhân gốc: `railway.toml` có `startCommand = "uvicorn app.main:app
> --host 0.0.0.0 --port $PORT"`. Railway thực thi `startCommand` **không
> thông qua shell**, nên `$PORT` không được shell expand thành giá trị số
> (Railway tự gán, ví dụ `8080`) mà bị truyền thẳng dưới dạng chuỗi ký tự
> `$PORT` vào `uvicorn` — và `uvicorn` không tự hiểu cú pháp biến môi trường
> kiểu shell.
>
> Cách sửa: bọc câu lệnh trong `sh -c "..."` để ép một shell thật sự expand
> biến trước khi `exec` chương trình:
> ```toml
> startCommand = "sh -c \"uvicorn app.main:app --host 0.0.0.0 --port $PORT\""
> ```
> Sau khi push commit sửa, deployment build lại, `$PORT` được expand đúng,
> `/health` trả `200` và healthcheck pass.
