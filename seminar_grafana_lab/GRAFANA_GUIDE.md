# 📗 HƯỚNG DẪN SỬ DỤNG GRAFANA CHO NGƯỜI MỚI BẮT ĐẦU (TỪ SỐ 0 ĐẾN THỰC CHIẾN)

Chào mừng bạn đến với Grafana! Nếu bạn chưa bao giờ dùng công cụ này, đừng lo lắng. Hãy coi Grafana như một **Bảng điều khiển camera an ninh** của toàn bộ hệ thống máy chủ. Thay vì xem hình ảnh, bạn sẽ xem các biểu đồ dữ liệu và các dòng trạng thái (Log).

Bài hướng dẫn này sẽ không nói lý thuyết suông. Chúng ta sẽ "cầm tay chỉ việc" cách sử dụng giao diện và cách xử lý khi có sự cố xảy ra.

---

## 🚀 BƯỚC 1: ĐĂNG NHẬP VÀ LÀM QUEN GIAO DIỆN CHÍNH

1. Mở trình duyệt web và truy cập vào: `https://34.126.87.102.nip.io/`

2. Chào mừng bạn đến trang chủ Grafana! Đừng để các nút bấm làm bạn rối mắt, hãy chỉ tập trung vào menu bên trái (biểu tượng 3 gạch ngang). Chọn **Explore** (Biểu tượng cái la bàn).
   👉 *Explore chính là "phòng thẩm vấn", nơi bạn sẽ truy vấn và moi móc mọi thông tin từ hệ thống.*

---

## 🔎 BƯỚC 2: CÁCH ĐỌC LOG CƠ BẢN NHẤT

Khi vào trang **Explore**, bạn sẽ thấy một giao diện gồm 2 phần chính:
- **Khung chọn nguồn dữ liệu (Góc trái trên cùng):** Bấm vào chữ `loki` (Đây chính là tên kho lưu trữ Log của chúng ta).
- **Khung nhập lệnh (Ở giữa):** Có 2 chế độ là `Builder` (kéo thả xếp gạch) và `Code` (gõ lệnh).

Chúng ta sẽ luôn dùng **Code** để kiểm soát hoàn toàn truy vấn. Nhấn chọn nút **Code** ở góc bên phải của ô nhập.

### 🕵️ Tình huống 1 (DevOps): Máy chủ đột nhiên bị sập!
Sếp gọi điện báo web không vào được, khách hàng phàn nàn bị trắng trang. Bạn cần biết ngay máy chủ đang báo lỗi gì.

**Hành động:**
1. Trong ô nhập lệnh Code, hãy gõ câu chú sau:
   `{job="apache"} |= " 500 "`
   *(Giải thích: Lấy tất cả log của web server Apache, SAU ĐÓ lọc ra những dòng có chứa chữ "500" - mã lỗi sập server).*
2. Bấm nút **Run query** màu xanh dương (hoặc `Shift + Enter`).
3. Kéo chuột xuống dưới phần Logs, bạn sẽ thấy các dòng log báo lỗi sập server. Hãy bấm vào dấu `>` để mở rộng dòng log ra và đọc các thông số IOCs (Indicators of Compromise/Failure) như: IP, URL bị sập, Method để mang đi báo cáo cho team Dev sửa code!

---

## 📈 BƯỚC 3: PHÂN TÍCH CHUYÊN SÂU - VẼ BIỂU ĐỒ TÌM IP CÀN QUÉT (404)

### 🕵️ Tình huống 2 (SecOps): Có kẻ đang càn quét hệ thống!
Hệ thống báo động có quá nhiều lỗi 404 (Không tìm thấy trang). Rất có thể một Hacker đang dùng công cụ dò quét lỗ hổng trên web của chúng ta. Sếp yêu cầu: **"Tìm ngay IP nào đang quét nhiều nhất và chặn nó lại!"**

**❓ TẠI SAO CHÚNG TA PHẢI DÙNG CHẾ ĐỘ CODE THAY VÌ BUILDER Ở BÀI NÀY?**
Nếu dùng chế độ Builder (Xếp gạch), bạn sẽ gặp phải 2 giới hạn cực kỳ lớn của Grafana:
1. Bạn phải vào `Operations -> Formats -> Regexp` và gõ Regex thủ công rất phức tạp mới bóc tách được IP.
2. (Quan trọng nhất) Chế độ Builder **KHÔNG CHO PHÉP** nhập biến `$__range` vào ô thời gian, mà ép bạn phải dùng `$__auto`. Điều này khiến kết quả đếm bị xé lẻ theo từng giờ (tạo ra hiện tượng đỉnh ảo 14 lần như bạn thấy trên biểu đồ) thay vì cộng dồn tổng số (60 lần) trong toàn bộ thời gian.
Đó là lý do các kỹ sư chuyên nghiệp luôn ưu tiên dùng chế độ Code để kiểm soát hoàn toàn kết quả!

**Bước 1: Chuyển sang chế độ gõ Code**
- Đảm bảo bạn đang chọn nút **Code** ở góc phải ô nhập lệnh.

**Bước 2: Gõ lệnh truy vết**
- Dán nguyên xi câu lệnh sau vào ô (Đừng lo, trông nó dài nhưng rất logic):
  ```logql
  sum by (client_ip) (count_over_time({job="apache", status="404"} | regexp "^(?P<client_ip>\\S+)" [$__range]))
  ```

**Bước 3: Chạy và đọc kết quả trên biểu đồ**
- Bấm **Run query** (hoặc nhấn `Shift + Enter`).
- **LƯU Ý QUAN TRỌNG VỀ THỜI GIAN:** Hãy đảm bảo bộ lọc thời gian (Time Picker) ở góc trên bên phải của bạn đang bao trùm trọn vẹn cả 4 ngày có log (từ `2026-06-05` đến `2026-06-08`).
- Kéo xuống dưới phần biểu đồ (Graph). Bạn sẽ thấy đường màu tím vọt lên cao nhất chạm mốc **60** trên trục dọc. Nhìn xuống chú thích (Legend) bên dưới, đó chính là IP `208.91.156.11`.

**✅ Kết luận hành động:** Bạn lập tức thấy IP `208.91.156.11` đứng Top 1 với 60 lần gây ra lỗi 404. Bạn lấy IP này và chặn nó trên Firewall.

---

### 🕵️ Tình huống 3: Điều tra chuyên sâu (Drill-down) - Kẻ tấn công đang làm gì?

Sau khi biết được IP `208.91.156.11` là thủ phạm gây ra bão lỗi 404, bạn (với tư cách là kỹ sư SOC) không chỉ chặn nó, mà còn phải lấy được các Chỉ số xâm nhập (IOCs) như Target URL, User-Agent để lập biên bản.

**Bước 1: Lọc riêng IP của thủ phạm**
- Trở lại ô nhập lệnh **Code** trong Explore. Xóa lệnh cũ và nhập lệnh mới này:
  ```logql
  {job="apache"} |= "208.91.156.11"
  ```

**Bước 2: Xem log thô (Raw Logs)**
- Bấm **Run query**.
- Kéo xuống mục **Logs** bên dưới biểu đồ để đọc từng dòng chi tiết.

**Bước 3: Phân tích hành vi & Thu thập IOCs**
Khi đọc một dòng log, hãy truy xuất các IOCs sau:
1. **Target URL (`GET /files/logstash/logstash-1.3.2-monolithic.jar`):** Kẻ này liên tục lùng sục để tải một file Java cũ có thể chứa lỗ hổng.
2. **Lý do lỗi (404):** Hệ thống không có file này, nên văng lỗi 404.
3. **User-Agent (`Chef Client`):** Kẻ này đang dùng một công cụ tự động hóa (Bot/Script) chứ không phải người dùng thật.

**✅ Kết luận cuối cùng:** Đây là một cuộc dò quét mù quáng (Blind Crawl) bằng công cụ tự động. Chặn IP này là hoàn toàn chính xác!
---

## 💡 3 THÓI QUEN CỦA CHUYÊN GIA GRAFANA

1. **Luôn kiểm tra Time Picker đầu tiên:** 90% trường hợp "anh ơi sao hệ thống trống trơn không có log" là do quên chỉnh thời gian ở góc trên bên phải về đúng ngày giờ xảy ra sự cố (chọn `Last 7 days` nếu đang làm lab).
2. **Dùng tính năng Auto-Refresh khi theo dõi sự cố:** Ở cạnh cái đồng hồ góc trên bên phải, có một biểu tượng mũi tên vòng tròn. Hãy chọn `5s` hoặc `10s` (Tự động tải lại sau mỗi 5 giây). Khi bạn đã chặn IP, bạn ngồi nhìn màn hình tự giật lại mỗi 5 giây, nếu không thấy IP đó xuất hiện nữa -> Bạn đã sửa lỗi thành công!
3. **Thêm `|~ "từ khóa"` để tìm kiếm siêu tốc:** Trong mục Code, nếu bạn chỉ muốn tìm nhanh một từ khóa trong log mà lười tạo điều kiện phức tạp, hãy dùng dấu `|~`. Ví dụ: `{job="apache"} |~ "login"` (Sẽ tìm toàn bộ log có dính chữ login).

Bây giờ, bạn đã có đủ tư duy và công cụ để tham gia buổi Seminar sắp tới như một Kỹ sư hệ thống thực thụ. Hãy mạnh dạn bấm và khám phá Grafana nhé!
