# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> [Điền câu trả lời]` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Hoàng Xuân Quân  Mã học viên: 2A202601868

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu không có cơ chế fail-fast, ứng dụng vẫn khởi động thành công ngay cả khi trên môi trường Cloud (như Railway) tôi quên thiết lập biến môi trường API_TOKEN. Khi đó, API sẽ "mở cửa" cho bất kỳ ai biết token "changeme" này truy cập, gọi model LLM miễn phí làm tiêu tốn tài nguyên thật. Việc bắt lỗi và crash ngay (fail-fast) giúp hệ thống báo động ngay lập tức, ngăn chặn hoàn toàn rủi ro bị lạm dụng do quên set secret.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log: `{"timestamp": "2026-08-10T15:40:00.000Z", "level": "INFO", "message": "chat request completed", "client_id": "sv-test", "cost_usd": 0.001}`
> Hai việc làm được:
> 1. Đưa thẳng log vào các hệ thống quản lý (như ELK, Datadog) để vẽ biểu đồ và thống kê một cách dễ dàng nhờ cấu trúc JSON.
> 2. Tìm kiếm và lọc log theo dữ liệu (ví dụ: filter mọi log theo một `client_id` cụ thể) dễ dàng mà không cần dùng regex phân tích văn bản thuần túy.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> - 1 stage (bản đầu): ~1000 MB (tuỳ theo base image)
> - Multi-stage: ~150 MB
> Giải thích: Phần chênh lệch là toàn bộ các công cụ dùng để build source code (gcc, python-dev), file cache của pip khi tải package. Dùng multi-stage, ta chỉ copy phần ứng dụng cuối cùng sang image trống, bỏ lại toàn bộ rác và công cụ build ở stage 1.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa một ký tự trong `app/main.py`: layer `COPY app/ app/` bị vô hiệu hóa bộ nhớ đệm (cache) và chạy lại. Các layer trước đó (như tải pip) vẫn sử dụng lại từ cache.
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Mọi thay đổi dù là nhỏ nhất trong file Python cũng làm hỏng cache của lệnh COPY, kéo theo việc phải chạy lại lệnh `pip install` rất mất thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro: Lỗ hổng RCE cho phép kẻ tấn công chèn mã thực thi. Nếu chạy bằng quyền root, kẻ tấn công chiếm quyền cao nhất trong container -> lợi dụng network hoặc volume để leo thang ra máy chủ host.
> Lệnh `USER appuser` cắt đứt chuỗi ở chỗ: Process bị giới hạn bởi quyền user thấp. Dù bị RCE, hacker cũng không thể đọc/ghi file hệ thống hay thao tác với container engine.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> - Cần header `WWW-Authenticate: Bearer` theo tiêu chuẩn RFC 6750 để gợi ý cho client biết phương thức xác thực nào đang được yêu cầu.
> - Trả cùng một lỗi vì bảo mật: Chống lại việc kẻ tấn công thăm dò. Gom chung một lỗi giúp che giấu thông tin nội bộ hệ thống, khiến hacker không biết được chính xác là do sai cấu trúc hay do hệ thống có token đó nhưng bị sai.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> - Chỉ gửi được đúng 10 request trước khi bị 429 vì sức chứa của xô bucket tối đa là 10.
> - Nếu bỏ `min(capacity, ...)`, client sẽ được tích 10 x 10 = 100 request. Việc này phá hỏng nguyên lý Rate Limit vì client lợi dụng thời gian ngủ để tạo "burst" request khổng lồ làm sập server.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> - $30/tháng: Thiệt hại tối đa ngay lúc đó là $30. Dịch vụ ngừng cho khách hàng đến tận tháng sau mới hồi phục.
> - $1/ngày: Thiệt hại tối đa trong ngày đó chỉ là $1. Ngày hôm sau dịch vụ tự động hồi phục. Cách này vừa giới hạn thiệt hại lớn, vừa duy trì trải nghiệm liên tục hơn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối.
> 2. K8s gọi endpoint /health, bị trả về 503 thay vì 200.
> 3. K8s kết luận các container đang chết nên lập tức tiêu diệt (restart) toàn bộ.
> 4. Downtime toàn cụm, trong khi lẽ ra chỉ cần ngừng nhận request mới và chờ Redis khôi phục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi gặp phải: `/readyz` trả 500 Internal Server Error (crash app) sau khi deploy lên Railway.
> - Tìm nguyên nhân: Nhờ /healthz vẫn trả 200, tôi biết app còn sống. Lỗi 500 chứ không phải 503 chứng tỏ biến `REDIS_URL` bị sai định dạng dẫn đến hàm khởi tạo kết nối quăng Exception.
> - Sửa: Lên tab Variables của Railway, đổi lại `REDIS_URL` cho đúng dạng `redis://...` từ tham chiếu biến Reference của Add-on Redis.
