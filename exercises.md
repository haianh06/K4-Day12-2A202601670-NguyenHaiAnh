# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử lúc deploy lên server thật mà mình quên set biến môi trường, app nó báo lỗi rồi crash luôn thì mình biết ngay mà sửa. Chứ nếu để mặc định là `"changeme"`, app vẫn chạy bình thường nhưng ai biết được token mặc định đó thì có thể dùng chùa API của mình luôn, rất rủi ro về bảo mật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> `{"event": "chat_completed", "client_id": "sv-test", "prompt_tokens": 12, "completion_tokens": 18, "usd_cost": 0.0005, "level": "info", "timestamp": "2026-08-10T10:00:00Z"}`
> Hai cái này print bình thường khó làm:
> 1. Đưa log vào mấy tool monitor (kiểu ELK, DataDog) thì mình có thể filter hoặc search theo từng field được luôn (vd search log của riêng thằng client_id="sv-test" chẳng hạn).
> 2. Mình lấy luôn trường `usd_cost` hoặc `completion_tokens` để tính tiền hoặc vẽ biểu đồ mà không cần dùng regex tách chuỗi ra cực như log text bình thường.

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
| 1 stage (bản đầu) | ~1.1 GB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch nhiều như vậy là do bản 1 stage nó ôm luôn cả mấy cái tool để build (như gcc, make), cache của pip, apt, với cả đống mã nguồn thừa lúc tải thư viện về. Sang multi-stage thì mình chỉ lấy đúng cái đống thư viện đã cài đặt xong xuôi ném sang một cái base image mới trắng tinh và nhẹ hơn (như slim), nên kết quả cuối cùng nhẹ đi rất nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Lúc sửa `main.py` thì các layer chạy trước lệnh `COPY app/ app/` (hoặc `COPY . .`) vẫn lấy lại từ cache bình thường, kể cả bước `RUN pip install`. Chỉ từ layer `COPY` cái file bị sửa trở đi mới phải chạy lại thôi.
> Nếu mình để `COPY . .` lên trước `RUN pip install` thì chỉ cần sửa 1 xíu trong file code, layer `COPY` cũng sẽ bị vô hiệu cache. Kéo theo lệnh cài thư viện pip install đằng sau phải chạy lại từ đầu, đợi rất mất thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ đoạn code Python của mình bị lỗi bảo mật cho phép hacker tiêm mã độc vào chẳng hạn, mã độc đó sẽ được chạy với quyền root trong container. Xong từ quyền root này, hacker sẽ tìm cách khai thác lỗ hổng để "vượt rào" ra ngoài (container escape), rồi từ đó thao túng luôn cả hệ điều hành của máy host.
> Mình xài lệnh `USER` ép app chạy bằng user thường, để lỡ hacker có chui vào được thì cũng chỉ có quyền thấp, không đủ sức để phá phách gì sâu hơn.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> - Gắn `WWW-Authenticate: Bearer` đơn giản là để chuẩn hóa theo HTTP, nó nhắc cho client biết là API đang xài kiểu xác thực Bearer token.
> - Báo lỗi chung chung 401 để chống lại trò "thăm dò" (enumeration attack). Nếu báo rõ kiểu "token sai rồi", hacker sẽ biết cấu trúc token có vẻ hợp lệ và cố tình brute-force. Giấu đi thì hacker sẽ khó đoán hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> - Gửi được 10 request thôi vì sức chứa của bucket (capacity) cao nhất chỉ là 10.
> - Nếu bỏ `min(capacity, ...)` đi thì token sẽ được cộng dồn mãi luôn. Sau 10 phút, bucket sẽ tích được 10 * 10 = 100 token. Lúc đó client có thể spam 1 lúc cả 100 request làm server quá tải.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> - Nếu set $30/tháng: Lỡ bị lỗi spam gọi từ 2h sáng thì mình có thể mất tiêu 30$ ngay trong hôm đó. Chưa kể client đó sẽ bị khóa không cho xài tiếp đến tận tháng sau.
> - Nếu set $1/ngày: Đen lắm thì hôm đó mình mất 1$ là nó chặn lại. Rồi cứ qua 0h ngày hôm sau là hệ thống tự reset lại cho client dùng tiếp bình thường, an toàn và đỡ thiệt hại hơn hẳn.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối.
> 2. End-point kiểm tra sức khỏe của cả 3 container đều báo lỗi vì không ping được tới Redis.
> 3. Hệ thống quản lý thấy check health thất bại, tưởng container của mình lỗi.
> 4. Thế là nó tự động kill rồi khởi động lại cả 3 container, gây downtime toàn bộ api.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Mình dính ngay cái lỗi app build xong nhưng chạy phát là crash luôn. Mở log trên Railway ra soi thì thấy báo lỗi: `ValidationError ... api_token Field required`.
> Nguyên nhân: Mình quên chưa set biến môi trường trên cloud.
> Sửa nhanh: Vào tab Variables của app trên dashboard, tạo thêm biến `API_TOKEN` rồi gán cho nó cái chuỗi bí mật, đợi nó tự redeploy là chạy ngon.
