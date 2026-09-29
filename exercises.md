# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder (in nghiêng, bắt đầu bằng `>`) bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Phương Duy  Mã học viên: 2A202602684

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi mình deploy service lên Railway lần đầu, mình đã set nhầm biến môi
> trường vào service Redis thay vì service agent (do CLI đang link sai
> service). Nếu `agent_api_key` có giá trị mặc định kiểu `"changeme"`, service
> vẫn khởi động bình thường và trả lời request — nghĩa là ai cũng gọi được
> `/ask` bằng key `"changeme"` mà mình không hề biết, vì service vẫn "chạy
> tốt" trên bề mặt. Vì `agent_api_key` không có default, `Settings()` sẽ ném
> `ValidationError` ngay khi container khởi động nếu thiếu biến này — container
> crash và log lỗi rõ ràng, nên mình biết ngay lúc deploy là có gì đó sai,
> thay vì phát hiện ra sau khi đã có người lạ gọi API miễn phí bằng key mặc định.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:59:22.985268+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}
> ```
> Hai việc làm được mà `print()` không làm được:
> 1. Lọc/tổng hợp theo field: vì log là JSON có key rõ ràng, mình có thể hỏi
>    "tổng `cost_usd` của `user_id=sv01` hôm nay là bao nhiêu?" bằng cách parse
>    từng dòng và cộng dồn field `cost_usd`. Với `print("đã trả lời xong")`,
>    không có cách nào tách được user hay chi phí ra khỏi chuỗi text tự do.
> 2. Cảnh báo tự động theo điều kiện: một hệ thống giám sát (Datadog, Railway
>    log filter...) có thể đọc field `level` và chỉ báo động khi
>    `level == "error"`, hoặc tính tỷ lệ lỗi theo `event`. Log dạng câu văn
>    không có cấu trúc để máy phân biệt "đây là log lỗi" hay "đây là log
>    bình thường" nếu không tự viết thêm parser riêng cho từng định dạng câu.

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
| 1 stage (bản đầu) | 1730 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.46GB đến từ hai nguồn chính. Thứ nhất, base image: bản 1-stage
> dùng `python:3.11` (bản đầy đủ, có sẵn nhiều công cụ build, compiler, thư
> viện hệ thống không cần cho lúc chạy) trong khi bản multi-stage dùng
> `python:3.11-slim` cho cả hai stage. Thứ hai và quan trọng hơn: ở bản
> 1-stage, `pip install` chạy ngay trong image cuối cùng nên toàn bộ cache
> của pip, các gói build-time (nếu có gói nào cần biên dịch) đều nằm lại
> trong image. Ở bản multi-stage, `pip install --prefix=/install` chạy trong
> stage `builder` riêng, sau đó chỉ có `COPY --from=builder /install
> /usr/local` mang đúng phần thư viện đã cài sang stage `runtime` — mọi thứ
> khác của stage `builder` (cache pip, layer trung gian) bị vứt bỏ hoàn toàn,
> không có mặt trong image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng comment vào cuối `app/main.py` rồi build lại. Kết quả:
> `COPY requirements.txt .` và `RUN pip install --prefix=/install ...` (toàn
> bộ stage `builder`) đều báo `CACHED` — không cài lại thư viện. Chỉ có
> `COPY app ./app` trở đi (COPY utils, RUN useradd) phải chạy lại, vì Docker
> cache theo nội dung file: layer nào COPY một file đã đổi thì layer đó và
> mọi layer sau nó trong cùng stage bị hủy cache, còn layer trước đó (dựa
> trên `requirements.txt` không đổi) vẫn giữ nguyên.
>
> Nếu đặt `COPY . .` lên trước `RUN pip install`, mọi thay đổi dù chỉ 1 ký tự
> trong bất kỳ file source nào cũng làm layer `COPY . .` mất cache, kéo theo
> `RUN pip install` ngay sau đó cũng phải chạy lại từ đầu — tức là sửa một
> dòng code cũng phải tải và cài lại toàn bộ thư viện trong `requirements.txt`,
> làm build chậm đi rất nhiều lần so với việc chỉ COPY code thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) code Python có một lỗ hổng cho phép thực thi lệnh tùy ý
> (ví dụ deserialize dữ liệu không kiểm soát, hoặc một dependency có lỗ hổng
> RCE) → (2) kẻ tấn công khai thác lỗ hổng đó, chạy được shell bên trong
> container → (3) nếu process đang chạy bằng root, shell đó cũng là root
> *bên trong* container → (4) root trong container có thể ghi vào những vùng
> filesystem nhạy cảm hoặc lợi dụng lỗ hổng container runtime (container
> escape) để leo thang ra ngoài, và nếu escape được thì đó là quyền root
> *trên host*, không chỉ trong container nữa.
>
> Lệnh `USER appuser` cắt đứt chuỗi này ở bước (3): dù kẻ tấn công khai thác
> được lỗ hổng ở bước (2), shell họ có được chỉ mang quyền của `appuser` — một
> user thường không có quyền ghi vào hệ thống, không cài được package, và các
> kỹ thuật container-escape phổ biến dựa vào quyền root bên trong container sẽ
> không hoạt động. Thiệt hại bị giới hạn lại ở bên trong container, không lan
> ra tới host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách đạt được: gửi 10 request vào lúc
> 10:00:59 (giây cuối của phút 10:00, vẫn tính vào "hạn mức của phút 10:00" —
> đúng luật vì bộ đếm phút 10:00 chưa đầy) rồi gửi tiếp 10 request vào lúc
> 10:01:00–10:01:01 (bộ đếm vừa reset về 0 cho phút 10:01, nên 10 request này
> cũng "đúng luật" của phút mới). Cả 20 request xảy ra trong khoảng 2 giây
> thực tế nhưng được tính là 2 cửa sổ riêng biệt (phút 10:00 và phút 10:01)
> nên không request nào bị chặn. Sliding window bằng ZSET của mình không có
> kẽ hở này vì nó luôn đếm đúng 60 giây *gần nhất tính từ thời điểm hiện tại*,
> không phụ thuộc vào ranh giới phút đồng hồ cố định.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn *số lượng* request trong một khoảng thời gian, còn cost
> guard giới hạn *tổng số tiền* đã tiêu trong một tháng — hai đại lượng độc
> lập nhau.
>
> Tình huống rate limit cho qua nhưng cost guard phải chặn: user chỉ gửi 2
> request/phút (rất thấp so với hạn mức 10/phút) nhưng mỗi câu hỏi rất dài,
> mỗi request tốn 50.000 token. Rate limit thấy "2 request/phút, còn quota"
> nên cho qua, nhưng nếu tổng chi phí trong tháng đã vượt `MONTHLY_BUDGET_USD`
> thì cost guard vẫn phải trả về 402, bất kể tốc độ gọi có chậm đến đâu.
>
> Tình huống ngược lại — cost guard cho qua nhưng rate limit phải chặn: một
> user còn dư ngân sách rất nhiều (ví dụ mới tiêu 0.01/10 USD) nhưng viết
> script gửi 100 request liên tiếp trong 1 giây, mỗi request rất rẻ. Cost
> guard thấy tổng tiền vẫn thấp nên không chặn, nhưng rate limit phải chặn từ
> request thứ 11 trở đi vì đã vượt `RATE_LIMIT_PER_MINUTE` — bảo vệ hệ thống
> khỏi bị spam/DoS dù chi phí mỗi request không đáng kể.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. Endpoint gộp (đóng vai trò cả liveness lẫn readiness)
>    gọi `store.ping()`, nhận `False`, trả về 503 cho cả 3 container cùng lúc
>    (vì cả 3 đều dùng chung Redis).
> 2. Orchestrator (Docker/K8s/Railway) đang dùng endpoint này làm **liveness
>    probe**, thấy 503 liên tục thì hiểu là "process đã chết, cần restart" —
>    chứ không hiểu là "process sống nhưng phụ thuộc ngoài đang lỗi".
> 3. Cả 3 container đều bị restart gần như đồng thời, vì cả 3 cùng chia sẻ
>    một điều kiện lỗi (Redis chết) tại cùng một thời điểm.
> 4. Trong lúc cả 3 container đang restart, không còn container nào phục vụ
>    request — toàn bộ service down hoàn toàn, dù bản thân code Python không
>    hề có lỗi.
> 5. Sau 30 giây, Redis sống lại. Nhưng nếu quá trình restart + khởi động lại
>    của 3 container mất nhiều hơn 30 giây, thì service tiếp tục down thêm
>    một khoảng sau khi Redis đã ổn — một sự cố hạ tầng nhỏ (Redis giật 30s)
>    bị khuếch đại thành downtime toàn bộ hệ thống, đúng thứ mà việc tách
>    `/health` (không kiểm tra Redis) và `/ready` (có kiểm tra) được thiết kế
>    để tránh: nếu tách riêng, `/ready` báo 503 chỉ khiến load balancer tạm
>    ngừng gửi traffic vào, còn `/health` vẫn 200 nên không container nào bị
>    restart oan.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy 3 container agent riêng biệt (cùng trỏ vào 1 Redis) và gọi `/ask`
> luân phiên round-robin qua từng container với cùng `X-User-Id=sv-scale-test`:
> container 1 → `history_length: 0`, container 2 → `2`, container 3 → `4`,
> container 1 → `6`, container 2 → `8`. Số liệu tăng đều đặn dù request nhảy
> lung tung giữa 3 container khác nhau — chứng minh cả 3 cùng đọc/ghi chung
> một nguồn dữ liệu (Redis List `history:sv-scale-test`).
>
> Nếu lịch sử được lưu trong một dict Python trong RAM của từng process: mỗi
> container sẽ có dict riêng, không container nào biết dict của container
> khác. Kết quả sẽ là `history_length` luôn ở mức rất thấp và "giật cục" theo
> việc request rơi vào container nào — ví dụ container 1 thấy `0, 2` (2 lần nó
> được gọi), container 2 thấy `0, 2` (độc lập, không biết 2 lượt trước ở
> container 1), container 3 cũng thấy `0` lần đầu — agent trông như liên tục
> "quên" các lượt trò chuyện trước đó mỗi khi load balancer đổi sang container
> khác.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần đầu deploy lên Railway, `railway up` báo "Deploy complete" nhưng service
> "Deploy crashed" ngay sau đó. Chạy `railway logs` thấy lỗi lặp lại liên tục:
> `/bin/sh: 1: exec: docker-entrypoint.sh: not found`. Ban đầu mình tưởng
> Dockerfile của mình thiếu gì đó, nhưng `docker-entrypoint.sh` không hề xuất
> hiện trong Dockerfile của mình — đó là entrypoint mặc định của image
> `redis:8.2`.
>
> Nguyên nhân tìm ra bằng cách chạy `railway status`: thấy "Linked service"
> chỉ có "Redis", và ID của service này trùng khớp với ID trong dòng
> "Deployment" lúc `railway up` chạy. Hóa ra sau khi mình chạy
> `railway add --database redis`, CLI tự chuyển "service đang liên kết" sang
> Redis. Khi mình chạy `railway up` ngay sau đó, nó build image agent của
> mình rồi deploy đè lên chính service Redis — container mới (chạy Python)
> không có file `docker-entrypoint.sh` mà start command cũ (dành cho Redis)
> đang cố gọi.
>
> Cách sửa: tạo một service riêng cho app (`railway add --service
> day12-agent`), link CLI đúng vào service đó (`railway service link
> day12-agent`), set lại toàn bộ biến môi trường cho service này, rồi
> `railway up --service day12-agent` để build/deploy đúng chỗ. Sau đó gặp
> thêm một lỗi khác (`/ready` báo Redis timeout do private networking giữa 2
> service mới tạo chưa thiết lập xong) nên cuối cùng mình chuyển hẳn sang
> Render — deploy bằng Blueprint (`render.yaml`) tự nối `REDIS_URL` giữa web
> service và Redis service, không phải tự tay cấu hình như Railway, và chạy
> ổn định ngay từ lần đầu.
