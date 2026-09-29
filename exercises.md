# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền nội dung dưới từng câu hỏi bằng lời của chính bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lưu Quang Khải  Mã học viên: 2A202602599

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tạo service `agent` trên Railway, nếu tôi quên đặt `AGENT_API_KEY`, app dừng ngay lúc khởi động và health check không thể báo thành công. Tôi sẽ kiểm tra biến môi trường trước khi đưa service ra Internet. Nếu code tự dùng khóa `changeme`, deployment vẫn xanh nhưng bất kỳ ai đoán được khóa đó đều gọi được `/ask`, gây tốn quota và chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log tôi lấy sau khi gọi `/ask` vào container local: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:31:10.170906+00:00", "user_id": "lab-reflection-check", "tokens_in": 6, "tokens_out": 38, "cost_usd": 2.37e-05}`. Từ các trường JSON, tôi có thể lọc các request của một `user_id` để điều tra sự cố và cộng `cost_usd` theo khoảng thời gian để theo dõi chi phí. Dòng `print("đã trả lời xong")` không có dữ liệu có cấu trúc cho hai việc đó.

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
| 1 stage (Dockerfile ban đầu trong Git) | 1.73 GB (1,727,879,358 byte) |
| Multi-stage | 271 MB (271,013,916 byte) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile một stage từ commit CP0 với cùng mã nguồn hiện tại và đo bằng `docker image inspect`: bản cũ 1,727,879,358 byte, bản nhiều stage 271,013,916 byte, giảm khoảng 1.46 GB. Bản đầu dùng `python:3.11` đầy đủ và cài thư viện ngay trong image chạy; bản mới dùng `python:3.11-slim`, chỉ copy thư viện từ stage builder sang stage runtime. Chênh lệch lớn chủ yếu do base image đầy đủ so với bản slim; builder stage cũng không nằm trong image cuối. Các chức năng của app vẫn được giữ nguyên.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, `COPY requirements.txt` và `RUN pip install ...` đứng trước `COPY app ./app`, nên khi chỉ sửa `app/main.py`, Docker dùng lại layer cài thư viện; layer copy `app` và các bước phía sau phải được xét lại. Nếu `COPY . .` đứng trước `RUN pip install`, chỉ một thay đổi trong code cũng làm mất cache của layer copy, khiến bước cài thư viện chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗi thực thi lệnh tùy ý trong API có thể cho kẻ tấn công chạy lệnh bên trong container. Nếu container chạy root, kẻ đó có quyền root trong container; khi host còn có cấu hình nguy hiểm như mount Docker socket hoặc có lỗ hổng container escape, quyền này có thể dẫn tới quyền cao trên host. `USER appuser` làm tiến trình ứng dụng chạy với UID thường 10001, nên mã bị chiếm quyền cũng bắt đầu với quyền thấp hơn. Chỉ riêng việc chạy root trong container chưa tự động đồng nghĩa root trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request vào cuối một phút, chẳng hạn 10:00:59, rồi gửi 10 request ngay đầu phút sau, chẳng hạn 10:01:00. Bộ đếm theo phút lịch reset ở giây 00 nên cả hai nhóm đều hợp lệ dù dồn vào khoảng hai giây. Cửa sổ trượt 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau đến và chặn từ request thứ 11.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lần gọi trong 60 giây; cost guard giới hạn tổng tiền của từng user trong tháng. Ví dụ user mới gọi 1 lần trong phút này nhưng đã tiêu hết ngân sách tháng: rate limit cho qua, cost guard trả 402. Ngược lại, user gửi request thứ 11 trong cùng 60 giây khi mới tiêu rất ít tiền: ngân sách còn nhưng rate limit trả 429. Trong `/ask`, cả hai được kiểm tra trước khi gọi mock LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Khi Redis mất kết nối, endpoint gộp sẽ trả lỗi dù cả ba tiến trình Python vẫn sống. Nếu orchestrator dùng endpoint đó làm liveness probe, nó coi cả ba container là hỏng và lần lượt restart; request đang xử lý có thể bị ngắt. Redis vẫn mất 30 giây nên các container mới lại trượt probe, gây thêm vòng restart và mất traffic. Với code hiện tại, `/health` chỉ kiểm tra tiến trình, còn `/ready` trả 503 khi Redis không ping được; bộ định tuyến có thể ngừng gửi request vào instance chưa sẵn sàng mà không restart chúng chỉ vì Redis lỗi tạm thời.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi thử ba container dùng chung Redis và cùng `X-User-Id`, lần lượt qua cổng local 8000, 8001, 8002. Cả ba request đều HTTP 200 và `history_length` lần lượt là 0, 2, 4 vì mỗi lượt thêm một tin nhắn user và một tin nhắn assistant vào Redis. Nếu dùng dict Python riêng trong mỗi process, mỗi container chỉ thấy lịch sử của chính nó; ba lượt đầu đi vào ba container khác nhau sẽ đều thấy 0. Compose hiện map cố định `8000:8000`, nên tôi chạy thêm hai container trên cùng Docker network ở cổng 8001 và 8002 để kiểm chứng việc chia sẻ state.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Trong lần deploy Railway, tôi đặt `startCommand` trong `railway.toml` với `$PORT`; lệnh này được truyền theo cách khiến `$PORT` không được shell mở rộng. Uvicorn nhận chuỗi đó thay vì một số cổng hợp lệ, nên tiến trình không khởi động và health check không đạt. Tôi xem trạng thái/log deployment, đối chiếu lệnh khởi chạy, rồi bỏ `startCommand` để Railway dùng `CMD` của Dockerfile: `sh -c "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"`. Deployment sau đó `SUCCESS`; URL công khai trả `/health` và `/ready` đều 200.
