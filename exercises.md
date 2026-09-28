# Phiếu phản ánh — K4 Level 3A, Ngày 12

Họ và tên: Trần Nguyễn Tiến Đức
Mã học viên: 2A202602871

## Câu 1 — Fail fast (CP1)

Nếu quên đặt `AGENT_API_KEY` trên Render, `Settings` báo lỗi ngay khi service khởi động. Test CP1 đã kiểm tra việc thiếu biến làm khởi tạo cấu hình thất bại; khi deploy tôi đặt khóa trong cấu hình Web Service trước khi chạy. Nếu dùng khóa mặc định `changeme`, service vẫn chạy và bất kỳ ai biết khóa mẫu đều có thể gọi `/ask`; việc phát hiện muộn hơn vì `/health` vẫn báo bình thường.

## Câu 2 — Log cho máy đọc (CP1)

Một dòng log thu được sau khi gọi `/ask` trong Docker Compose:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:58:38.088096+00:00", "user_id": "local-check", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Từ các trường này, tôi có thể cộng `cost_usd` theo `user_id` để tìm người dùng tiêu nhiều nhất, và đếm event theo khoảng thời gian từ `timestamp` để theo dõi lưu lượng. Dòng `print("đã trả lời xong")` không cung cấp dữ liệu có cấu trúc để làm hai việc đó.

## Câu 3 — Kích thước image (CP2)

Tôi build Dockerfile một stage ban đầu thành `agent:single`, sau đó build Dockerfile multi-stage thành `agent:multi`. Lệnh `docker images` trên máy cho kết quả:

| Bản | Dung lượng đo được |
|---|---:|
| Một stage, `python:3.11` | 1,77 GB |
| Multi-stage, `python:3.11-slim` | 297 MB |

Bản một stage mang theo các thành phần của base image Python đầy đủ và nhiều file trong build context. Bản multi-stage chỉ giữ runtime slim, thư viện đã cài và mã `app`/`utils`; stage builder và các file phát triển không nằm trong image cuối. Chênh lệch đo được là khoảng 1,47 GB.

## Câu 4 — Thứ tự lệnh Dockerfile (CP2)

Tôi đổi một ký tự trong comment của `app/main.py`, build lại rồi khôi phục file. Log build báo các bước liên quan base image, `COPY requirements.txt`, `pip install` và copy dependency từ builder là `CACHED`; `COPY app`, `COPY utils` và bước tạo user ở runtime chạy lại. Nếu đặt `COPY . .` trước `RUN pip install`, thay đổi ở `main.py` sẽ làm mất cache từ bước copy đó và phải cài lại dependency.

## Câu 5 — Vì sao không chạy bằng root (CP2)

Một lỗi thực thi mã từ xa trong Python có thể cho kẻ tấn công chạy lệnh trong container. Nếu process là root và container còn bị cấu hình nguy hiểm như mount Docker socket hoặc thư mục nhạy cảm của host, kẻ tấn công có thể lợi dụng quyền đó để điều khiển host. `USER appuser` chuyển process sang UID 10001, nên lệnh khai thác ban đầu chỉ có quyền của user thường trong container. Điều này giảm quyền có thể dùng để leo thang, dù vẫn cần bảo vệ các mount và Docker daemon.

## Câu 6 — Cửa sổ trượt (CP3)

Với bộ đếm reset vào giây `00`, có thể gửi 10 request ở cuối phút cũ và 10 request ngay đầu phút mới: tổng cộng 20 request trong khoảng 2 giây mà mỗi phút lịch vẫn không quá 10. Sliding window 60 giây nhìn lại toàn bộ 60 giây gần nhất, nên nhóm thứ hai sẽ bị chặn.

## Câu 7 — Rate limit và cost guard (CP3)

Rate limit giới hạn tốc độ gọi, còn cost guard giới hạn tổng chi phí theo user trong tháng. Nếu user chỉ gọi một request trong phút nhưng đã tiêu hết ngân sách tháng, rate limit cho qua còn cost guard trả 402. Ngược lại, user còn nhiều ngân sách nhưng gọi request thứ 11 trong cùng cửa sổ 60 giây thì cost guard chưa chặn, rate limit trả 429.

## Câu 8 — `/health` khác `/ready` (CP4)

Nếu cả hai probe cùng kiểm tra Redis, khi Redis mất kết nối 30 giây, ba container đều trả 503 cho liveness. Orchestrator xem cả ba là lỗi và khởi động lại cùng lúc; các request đang xử lý bị cắt và hệ thống không có instance nào nhận traffic trong thời gian restart. Với hai endpoint tách riêng, `/health` vẫn trả 200 vì process còn sống, còn `/ready` trả 503 để load balancer tạm ngừng chuyển request đến các instance đó. Khi Redis phục hồi, readiness trở lại 200 mà không cần restart hàng loạt.

## Câu 9 — Stateless (CP4)

Tôi chạy ba container agent sau Nginx bằng `docker compose -f docker-compose.yml -f docker-compose.scale.yml up -d --scale agent=3`, rồi gọi `/ask` năm lần cùng `X-User-Id: scale-check`. `history_length` lần lượt là **0, 2, 4, 6, 8**. Log xác nhận request phân bổ qua cả ba container: agent-1 nhận 1, agent-2 nhận 2, agent-3 nhận 2. Nếu thay Redis bằng dict Python trong từng process, mỗi container sẽ chỉ thấy lịch sử của chính nó; chuỗi trên sẽ có lần lặp lại số cũ hoặc bắt đầu lại từ 0 khi request chuyển container.

## Câu 10 — Deploy thật (CP5)

Lỗi tôi gặp khi thử Railway là `Your workspace has been restricted. Please attach a payment method or contact support to resolve this.` khi chạy `railway init`. Tôi kiểm tra bằng Railway CLI và thấy lệnh chưa tạo được project, nên không thể tiếp tục cấp Redis hay URL trên workspace đó. Tôi chuyển sang Render, dùng CLI tạo Key Value Free và Web Service Docker Free, đặt `REDIS_URL` trỏ tới Key Value nội bộ. Sau khi deploy, `/health` và `/ready` đều trả 200 trên URL HTTPS công khai; `/ask` thiếu khóa trả 401 và có khóa trả 200.
