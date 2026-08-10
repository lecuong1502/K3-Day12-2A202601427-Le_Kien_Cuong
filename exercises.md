# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
>
> Họ và tên: Lê Kiên Cường  Mã học viên: 2A202601427

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Nếu agent_api_key có giá trị mặc định như "changeme", app vẫn khởi động bình thường ngay cả khi tôi quên set biến AGENT_API_KEY trên Railway. Lúc đó service sẽ chạy công khai với một khóa ai cũng biết trước — bất kỳ ai cũng gọi được /ask và tiêu tiền ngân sách của tôi mà tôi không hề hay biết, cho tới khi nhìn hóa đơn. Với thiết kế không có mặc định, nếu tôi quên set biến, Settings() sẽ raise ValidationError ngay lúc container khởi động — deploy fail ngay lập tức, log hiện rõ lỗi, và tôi biết ngay để sửa trước khi bất kỳ ai gọi được vào service.*

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Một dòng log thật tôi thu được khi gọi /ask:
{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T...", "user_id": "sv01", "tokens_in": ..., "tokens_out": ..., "cost_usd": 0.0001}
Hai việc làm được mà print() không làm được: (1) lọc theo trường bằng công cụ log aggregator, ví dụ tìm tất cả sự kiện có cost_usd > 0.01 để phát hiện user tiêu nhiều; (2) tính tổng/thống kê tự động, ví dụ đếm số sự kiện event=ask_completed trong 5 phút để tính tỷ lệ request theo thời gian — với print() dạng câu tiếng Việt tự do, máy không parse được cấu trúc để làm việc này.*

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
| 1 stage (bản đầu) | 1.19 GB |
| Multi-stage | 183 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Chênh lệch gần 1 GB (giảm hơn 6.5 lần) đến từ hai nguồn chính: (1) base image
`python:3.11` đầy đủ (~900MB, chứa build tool, compiler, header files để biên
dịch package) so với `python:3.11-slim` (~150MB, chỉ có runtime tối thiểu);
(2) ở bản 1 stage, toàn bộ `pip cache`, các package build-dependency tạm thời,
và mọi thứ COPY vào bằng `COPY . .` (kể cả file không cần cho runtime như
`.git`, test, docs) đều nằm lại trong layer cuối cùng của image. Với
multi-stage, chỉ có kết quả cài đặt Python packages (`/usr/local`) và code
thực sự cần chạy (`app/`, `utils/`) được copy sang stage runtime — stage
builder chứa toàn bộ phần "rác" build-time bị vứt bỏ hoàn toàn, không để lại
dấu vết trong image cuối.*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Với Dockerfile hiện tại (COPY requirements.txt → pip install → COPY app/utils), khi tôi sửa 1 dòng trong app/main.py, Docker chỉ chạy lại từ layer COPY app ./app trở đi — layer cài requirements.txt vẫn dùng cache, build nhanh. Nếu đặt COPY . . lên trước pip install, bất kỳ thay đổi nào trong code (kể cả 1 dấu phẩy) cũng làm hỏng cache ngay từ layer COPY, khiến pip install phải chạy lại toàn bộ — build chậm hơn nhiều dù chỉ sửa code, không đụng dependency.*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Nếu code Python có lỗ hổng và container chạy bằng root, kẻ tấn công khai thác được lỗ hổng đó sẽ có quyền root bên trong container. Nếu container có cấu hình mount không an toàn hoặc kernel có lỗ hổng escape, quyền root trong container có thể leo thang thành quyền root trên chính máy host. Lệnh USER appuser cắt đứt chuỗi này ngay từ bước đầu: kể cả khi lỗ hổng bị khai thác, tiến trình chạy với quyền user thường (uid 10001), không có quyền ghi vào hệ thống hay thực hiện các thao tác cần root — giới hạn thiệt hại tối đa trong phạm vi user thường.*

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Với đếm theo phút đồng hồ, hạn mức 10/phút, người dùng có thể gửi 10 request vào cuối phút này (ví dụ 10:00:59) và 10 request nữa ngay đầu phút sau (10:01:00), tổng cộng 20 request trong vòng chưa tới 2 giây — vẫn "đúng luật" vì mỗi khung giờ tính riêng. Sliding window không có kẽ hở này vì nó luôn nhìn vào đúng 60 giây gần nhất tính từ thời điểm hiện tại, không phụ thuộc ranh giới phút cố định.*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Rate limit giới hạn số lượng request trong một khung thời gian, cost guard giới hạn số tiền tiêu trong tháng. Tình huống rate limit cho qua nhưng cost guard chặn: user gửi đúng 5 request/phút (dưới hạn mức 10), nhưng mỗi request hỏi một câu rất dài, tốn 50.000 token — chỉ vài request là vượt ngân sách tháng dù tần suất thấp. Tình huống ngược lại: user gửi 15 request/phút với câu hỏi cực ngắn, mỗi request tốn rất ít tiền — cost guard vẫn cho qua (chưa vượt ngân sách) nhưng rate limit chặn ở request thứ 11 vì tần suất quá cao, có thể là dấu hiệu bot đang spam.*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Nếu gộp làm một và cho /health kiểm tra Redis: khi Redis mất kết nối 30 giây, cả 3 container đều gọi Redis thất bại trong /health → cả 3 trả 503 → orchestrator hiểu nhầm là "process chết" và restart cả 3 container cùng lúc (đúng ra chỉ nên rút traffic, không nên restart vì process vẫn sống bình thường, chỉ là dependency tạm thời lỗi). Khi Redis phục hồi sau 30 giây, cả 3 container đang trong quá trình restart, không container nào sẵn sàng phục vụ ngay — biến một sự cố nhỏ (Redis chập chờn) thành downtime toàn hệ thống.*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Khi chạy --scale agent=3 và gọi /ask nhiều lần với cùng X-User-Id, history_length trong response tăng dần đều (1, 2, 3, 4, 5...) dù mỗi request có thể rơi vào container khác nhau — vì lịch sử được lưu chung trong Redis. Nếu lưu trong dict Python cục bộ của từng container, history_length sẽ không tăng đều mà nhảy lung tung — ví dụ request 1 vào container A (history=1), request 2 vào container B (history=1, vì B không biết gì về lịch sử ở A), request 3 lại về A (history=2)... agent trông như bị "mất trí nhớ" ngẫu nhiên.*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Lỗi tôi gặp: Error: Invalid value for '--port': '$PORT' is not a valid integer. khi deploy lên Railway. Nguyên nhân: startCommand trong railway.toml gọi uvicorn trực tiếp (exec form), không đi qua shell, nên biến $PORT không được giãn thành giá trị số thật mà bị truyền nguyên văn dạng chuỗi '$PORT'. Tôi tìm ra nguyên nhân bằng cách đọc kỹ log lỗi (railway logs) và nhận ra thông báo lỗi chỉ đích danh chuỗi $PORT chưa được xử lý. Cách sửa: bọc lệnh trong sh -c '...' để container tự chạy qua shell, giãn biến môi trường trước khi gọi uvicorn: startCommand = "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port $PORT'".*
