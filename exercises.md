# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Mạnh Hùng  Mã học viên: 2A202602708

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định là `"changeme"`, khi deploy lên cloud mà quên cấu hình biến môi trường, ứng dụng vẫn khởi động thành công. Kẻ tấn công có thể dễ dàng đoán ra API Key này để dùng chùa API, gây thất thoát chi phí lớn (cost guard bị qua mặt) hoặc lộ dữ liệu. Việc "fail fast" giúp phát hiện cấu hình sai ngay từ lúc app khởi động trên cloud, chặn đứng rủi ro trước khi mở ra internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

`{"level": "INFO", "message": "Request processed", "user_id": "sv-test", "cost_usd": 0.05, "duration_ms": 120, "timestamp": "2026-09-29T10:00:00Z"}`
1. Có thể dùng các hệ thống quản lý log (như ELK, Datadog) để parse, query và vẽ biểu đồ dashboard theo dõi chi phí (cost_usd) hoặc thời gian xử lý (duration_ms) của hệ thống.
2. Dễ dàng filter, tìm kiếm các request bất thường hoặc tính toán tổng chi phí của user `sv-test` bằng các công cụ phân tích mà không cần viết regex để bóc tách chuỗi phức tạp như với hàm print thông thường.

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
| 1 stage (bản đầu) | ~ 1 GB |
| Multi-stage | ~ 150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch dung lượng chủ yếu là do bản 1 stage chứa toàn bộ các công cụ build (build-essential, trình biên dịch C), pip cache, và các file mã nguồn không cần thiết. Bản multi-stage chỉ copy lại môi trường ảo (virtualenv) đã cài đặt xong các thư viện cần thiết, bỏ lại toàn bộ rác và công cụ build ở stage builder nên nhỏ hơn rất nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa `app/main.py`, các layer cài đặt thư viện (`COPY requirements.txt` và `RUN pip install`) vẫn được dùng lại (cached) vì nội dung file requirements không đổi. Chỉ có layer `COPY . .` và các layer sau nó phải chạy lại.
Nếu đặt `COPY . .` lên trước `RUN pip install`, thì mọi thay đổi nhỏ trong code (như sửa 1 ký tự) đều làm vô hiệu hóa cache của lệnh `COPY`. Kết quả là lệnh `RUN pip install` đứng sau sẽ bị mất cache và phải tải/cài đặt lại toàn bộ thư viện từ đầu, làm quá trình build rất chậm.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu có lỗ hổng thực thi mã từ xa (RCE) trong code Python, kẻ tấn công sẽ thực thi được lệnh shell bên trong container với tư cách là root. Từ đó, kẻ tấn công có thể lợi dụng các cấu hình lỗi của container (chạy chế độ privileged, share volume nhạy cảm) để escape ra ngoài và giành quyền root trên máy host.
Lệnh `USER appuser` chuyển quyền chạy tiến trình sang một user không có đặc quyền. Khi đó, kẻ tấn công dù có hack được vào container thì cũng chỉ có quyền thấp của `appuser`, không thể thay đổi hệ thống hay escape ra máy host, cắt đứt chuỗi tấn công leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Họ có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích: Người dùng có thể gửi 10 request ở giây 59 của phút hiện tại. Ở giây 00 của phút tiếp theo, bộ đếm bị reset về 0 nên họ có thể gửi ngay tiếp 10 request nữa. Kết quả là trong vỏn vẹn 2 giây (giây 59 và giây 00), họ đã gửi 20 request, vượt gấp đôi giới hạn mong muốn. Sliding window sẽ đếm trượt chính xác 60 giây nên khắc phục được lỗi này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit chặn theo tần suất request trong thời gian ngắn (chống spam). Cost guard chặn theo tổng chi phí tích lũy trong kỳ (chống vượt ngân sách).
- **Rate limit cho qua, Cost guard chặn:** Một user hỏi rải rác 1 câu mỗi phút (không bị Rate limit), nhưng hỏi cả tháng trời và đã tiêu hết quỹ ngân sách 10$. Request tiếp theo của họ sẽ bị Cost guard chặn.
- **Cost guard cho qua, Rate limit chặn:** User mới tinh (chưa tốn đồng nào) dùng bot bắn 50 request trong 1 giây. Cost guard chưa vượt ngưỡng, nhưng Rate limit sẽ chặn ngay từ request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Redis mất kết nối.
2. Endpoint `/health` của cả 3 container đều gọi Redis thất bại và trả về HTTP 500 (hoặc timeout).
3. Hệ thống điều phối (như Kubernetes/Railway) thấy liveness probe bị lỗi, kết luận rằng cả 3 container đều đã bị treo.
4. Hệ thống kill và restart cả 3 container liên tục (crash loop) dẫn đến sập toàn bộ dịch vụ. Việc tách `/ready` giúp container tạm ngắt traffic để đợi Redis hồi phục, còn `/health` vẫn trả về 200 để tránh bị hệ thống tự động kill oan.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu lịch sử trong dict Python ở RAM, state của hệ thống sẽ bị phân mảnh. Mỗi container chỉ biết những request mà bộ cân bằng tải (load balancer) đẩy vào nó.
Do đó, khi gọi `/ask` nhiều lần, `history_length` sẽ không tăng đều đặn (1, 2, 3...) mà sẽ nhảy lộn xộn (ví dụ: 1, 1, 1, 2, 2, 2...) tùy thuộc vào việc mỗi request rơi ngẫu nhiên trúng container nào. Nhờ dùng Redis lưu state tập trung, cả 3 container mới đọc được cùng 1 state nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** Lỗi HTTP 500 Internal Server Error khi test endpoint `/ask` trên cloud.
- **Tìm nguyên nhân:** Tôi vào bảng điều khiển (Dashboard) của Railway, mở tab Logs của Agent service ra xem thì thấy lỗi kết nối không được vì `REDIS_URL` bị trống hoặc sai.
- **Cách sửa:** Tôi khởi tạo Redis add-on, copy chuỗi connection string của nó và paste vào phần Variables của service Agent trên Railway dưới tên biến `REDIS_URL`, sau đó deploy lại là thành công.
