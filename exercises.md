# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Mai Tiến Dũng
>
> Mã học viên: 2A202601838

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy lên cloud, nếu quên khai báo `API_TOKEN`, ứng dụng sẽ dừng ngay lúc khởi động và log báo thiếu biến bắt buộc. Nhờ vậy em phát hiện lỗi cấu hình trước khi nhận traffic, thay vì app chạy với token mặc định rồi cho người lạ gọi API miễn phí. Giá trị `changeme` còn nguy hiểm vì rất dễ bị đoán và dùng chung giữa các môi trường.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ log: `{"event":"chat_completed","client_id":"anonymous","prompt_tokens":12,"completion_tokens":18,"usd_cost":0.0002}`. Log JSON có cấu trúc nên máy có thể lọc theo `event`, đếm request lỗi/thành công và tính tổng chi phí bằng hệ thống log hoặc truy vấn dữ liệu. 
Nó cũng giữ được các trường cố định để cảnh báo và dashboard; một câu `print` tự do không có schema ổn định nên khó tìm kiếm, phân tích và theo dõi tự động.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | khoảng 1.8 GB |
| Multi-stage | khoảng 250–300 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Bản multi-stage chỉ mang virtualenv đã cài dependency và source vào runtime, không mang compiler, cache pip hay các file build nên nhỏ hơn nhiều. Con số trên là số đo theo `docker images` sau khi build; kích thước có thể chênh lệch đôi chút tùy cache và nền tảng Docker.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Docker copy `requirements.txt` và cài dependency trước, sau đó mới `COPY` source. Vì vậy khi em sửa một ký tự trong `app/main.py`, các layer cài Python và dependency vẫn được lấy từ cache, chỉ các layer copy source và phía sau phải chạy lại. Nếu đặt `COPY . .` trước `pip install`, mọi thay đổi trong source đều làm mất cache của bước cài dependency, khiến build lâu hơn dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu code có lỗ hổng cho phép thực thi lệnh, tiến trình chạy bằng root có thể bị lợi dụng để đọc/sửa toàn bộ file trong container và khai thác thêm quyền của container. Nếu container được cấp quyền hoặc mount nhạy cảm, kẻ tấn công có thể tiếp cận tài nguyên của máy host. Dockerfile tạo user `app` không có đặc quyền rồi dùng `USER app`, nên từ thời điểm đó tiến trình bị giới hạn; lỗ hổng vẫn nguy hiểm nhưng giảm đáng kể khả năng leo thang thành root.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

`WWW-Authenticate: Bearer` là yêu cầu của chuẩn HTTP cho phản hồi `401`, cho client biết tài nguyên cần xác thực bằng scheme Bearer. Em dùng cùng thông báo `invalid or missing bearer token` cho thiếu header, sai scheme và sai token để không tiết lộ token có tồn tại hay không, tránh hỗ trợ kẻ tấn công dò thông tin. Client hợp lệ vẫn biết phải gửi Bearer token nhờ header chuẩn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Sau 10 phút, bucket vẫn bị giới hạn bởi `capacity`, nên chỉ có 10 token. Client gửi thành công 10 request liên tiếp; request thứ 11 nhận `429`. Nếu bỏ `min(capacity, ...)`, lượng nạp thêm là `10 phút × 10 token/phút = 100`, cộng 10 token ban đầu thành 110, nên client có thể xả 110 request tức thời và phá vỡ giới hạn burst thiết kế.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Với hạn mức tháng $30, sự cố bắt đầu lúc 2h có thể tiêu tối đa gần $30 trước khi hết tháng; ngân sách chỉ hồi khi sang tháng mới. Với hạn mức ngày $1, thiệt hại tối đa trong ngày UTC đó là $1 và ngân sách tự đặt lại ở 00:00 UTC hôm sau. Vì vậy giới hạn theo ngày thu hẹp thiệt hại tối đa khoảng 30 lần và tự phục hồi mà không cần can thiệp thủ công.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp hai endpoint và `/healthz` cũng ping Redis, khi Redis mất kết nối cả ba container sẽ báo unhealthy sau khoảng 30 giây. Orchestrator/load balancer lần lượt loại cả ba instance khỏi traffic hoặc restart chúng; các instance mới vẫn không kết nối được Redis nên cụm có thể rơi vào vòng restart và mất toàn bộ dịch vụ. Tách `/healthz` (chỉ kiểm tra process) khỏi `/readyz` (kiểm tra Redis) giúp container vẫn sống để quan sát, còn load balancer chỉ ngừng gửi request tới instance chưa sẵn sàng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi mở URL gốc trên Render, em nhận `{"detail":"Not Found"}`. Em kiểm tra log và route FastAPI rồi xác định đây không phải lỗi server: ứng dụng chỉ khai báo `/healthz`, `/readyz` và `/chat`, không có route `/`. Em dùng `/healthz` để kiểm tra service và cấu hình health check của Render vào endpoint đó; sau khi deploy, `/healthz` trả 200 và `/readyz` trả 200 khi Redis sẵn sàng.
