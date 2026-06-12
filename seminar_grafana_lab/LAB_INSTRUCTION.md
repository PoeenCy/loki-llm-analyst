# HƯỚNG DẪN THỰC HÀNH GRAFANA & LOKI (SEMINAR LAB)

Chào mừng các bạn đến với phần thực hành hệ thống Giám sát tập trung bằng Grafana và Loki. Trong bài Lab này, chúng ta sẽ đóng vai trò là những Kỹ sư Vận hành (DevOps) và Kỹ sư Bảo mật (SOC) để điều tra các sự cố từ một máy chủ web đang bị tấn công.

Dữ liệu bạn đang phân tích là **dữ liệu thật 100%** từ máy chủ Apache.

> **⚠️ LƯU Ý QUAN TRỌNG VỀ TRUY VẤN (CẢNH BÁO TIMEOUT):**
> Hệ thống máy chủ Lab đã được thiết lập giới hạn thời gian chạy câu lệnh (Timeout) tối đa là **10 giây** để đảm bảo công bằng tài nguyên cho tất cả sinh viên. 
> Nếu câu lệnh của bạn báo lỗi `Timeout` hoặc `context deadline exceeded`, điều đó có nghĩa là lệnh của bạn **chưa được tối ưu** (Ví dụ: Bạn ép hệ thống đếm lại khối lượng `[24h]` mỗi 2 giây thay vì dùng biến linh hoạt `$__interval`). Hãy tìm cách viết gọn và tối ưu lại nhé!

---

## 📊 THÔNG TIN HỆ THỐNG MẪU
- **Phân bổ thời gian:** Từ ngày 05/06/2026 đến 08/06/2026
- **Thống kê sơ bộ:**
| Dữ liệu | Số lượng |
| :--- | :--- |
| Ngày 05/06/2026 | 1.632 sự kiện |
| Ngày 06/06/2026 | 2.893 sự kiện |
| Ngày 07/06/2026 | 2.896 sự kiện |
| Ngày 08/06/2026 | 2.579 sự kiện |
| **HTTP Status 200 (Thành công)** | **9.126 sự kiện** |
| **HTTP Status 404 (Không tìm thấy)** | **213 sự kiện** |
| **HTTP Status 500 (Lỗi hệ thống)** | **3 sự kiện** |

---

## 🧠 PHẦN 1: BÀI HƯỚNG DẪN LÀM QUEN (CÔNG THỨC PHÁ ÁN 3 BƯỚC)

Để sử dụng Grafana hiệu quả như một thám tử thực thụ, bạn không nên đọc log theo từng dòng một cách thủ công. Hãy áp dụng **Công Thức Phá Án 3 Bước** sau đây:

`[FILTER] Lọc -> [GROUP] Gom nhóm -> [VERIFY] Đối chiếu`

1. **[FILTER] Lọc:** Dùng LogQL để thu hẹp phạm vi tìm kiếm. (Ví dụ: Chỉ tìm các dòng có chữ "error" hoặc mã lỗi 404).
2. **[GROUP] Gom nhóm:** Dùng hàm tính toán để gom nhóm các lỗi giống nhau hoặc gom theo IP/đường dẫn. (Ví dụ: Xem IP nào gây ra nhiều lỗi nhất).
3. **[VERIFY] Đối chiếu:** Đọc lướt qua một vài dòng log chi tiết (Raw Logs) của nhóm vừa tìm được để xác nhận nguyên nhân thật sự.

### 🎯 Bài làm thực chiến Khởi động: Tìm IP "Khách hàng thân thiết"
**Yêu cầu:** Hãy dùng công thức trên để tìm xem trên toàn bộ hệ thống, Địa chỉ IP nào đang truy cập web của chúng ta nhiều nhất?

**Lệnh LogQL gợi ý (Chạy ở chế độ Code):**
```logql
topk(1, sum by (client_ip) (count_over_time({job="apache"} | regexp "^(?P<client_ip>\\S+)" [$__range])))
```
*(Giải thích: Gom nhóm đếm số lượng log theo IP, lấy top 1 cao nhất).*

**💡 Đáp án thực tế:** IP `66.249.73.135` đứng top 1 với **482 lượt truy cập**. Sau khi verify (đối chiếu), đây chính là con bọ tìm kiếm của Google (Googlebot) đang quét để index website của chúng ta!

---

## 💡 PHẦN 2: 3 BÀI HƯỚNG DẪN TƯ DUY XỬ LÝ SỰ CỐ (MINDSET)

Trước khi bắt tay vào thực hành hệ thống thật, hãy trang bị 3 tư duy sau đây để tránh hoảng loạn khi hệ thống báo đỏ:

### 1️⃣ Tư duy của Lập trình viên (Dev) - "Nút bấm tàng hình"
- **Hiện tượng:** Bạn thấy biểu đồ lỗi 404 (Not Found) tăng đột biến ở một đường dẫn file ảnh hoặc CSS.
- **Tư duy đúng:** Đây có thể chỉ là do đội Lập trình (Dev) **quên upload** một file ảnh hoặc file CSS lên máy chủ. Giải pháp là báo cho Dev kiểm tra lại file.

**🎯 Bài tập thực chiến Tư duy Dev:**
- **Yêu cầu:** Hãy tìm ra file nào (đường dẫn path) bị lỗi 404 nhiều nhất hệ thống?
- **LogQL gợi ý:**
  ```logql
  topk(3, sum by (path) (count_over_time({job="apache", status="404"} | regexp ".*\"[A-Z]+ (?P<path>\\S+) " [$__range])))
  ```
- **💡 Đáp án:** Đứng thứ nhì trong danh sách lỗi 404 chính là file ảnh `/presentations/logstash-puppetconf-2012/images/office-space-printer-beat-down-gif.gif` (bị lỗi 32 lần). Rõ ràng đội Dev đã làm thiếu mất file ảnh này trên server!

### 2️⃣ Tư duy của Kỹ sư Mạng (NetOps) - "Giờ cao điểm kẹt xe"
- **Hiện tượng:** Cảnh báo băng thông mạng (Network Traffic) tăng vọt tạo thành một đỉnh nhọn (Spike) khổng lồ trên biểu đồ. Web bị chậm.
- **Tư duy đúng:** Hãy khoan chặn! Có thể hôm nay đội Marketing đang chạy Ads. Khách hàng đổ xô vào xem sản phẩm cùng một lúc gây kẹt xe. Cần dùng LogQL để tính toán lượng Bytes/Count.

**🎯 Bài tập thực chiến Tư duy NetOps:**
- **Yêu cầu:** Tính tổng dung lượng mạng (Bytes) đã truyền tải và tìm ra "đỉnh Spike" lớn nhất trong ngày 05/06/2026.
- **LogQL gợi ý:**
  ```logql
  sum(sum_over_time({job="apache"} | regexp ".* (?P<bytes>\\d+) \".*\" \".*\"" | unwrap bytes [1h]))
  ```
- **💡 Đáp án:** Khung giờ **04:15 sáng ngày 07/06** có một đỉnh tải dữ liệu khổng lồ lên tới **206 Mil** (khoảng 196 MB). Sau khi Verify, đó là các lượt tải hoàn toàn bình thường (Status 200), không phải tấn công DDoS! Cần cân nhắc mua thêm băng thông.

### 3️⃣ Tư duy của Kỹ sư Bảo mật (SecOps) - "Kẻ gõ cửa mù quáng"
- **Hiện tượng:** Bộ phận SOC phát hiện hệ thống phát sinh lỗi `404 Not Found` liên tục từ một vài địa chỉ IP. Nghi ngờ có tin tặc đang chạy công cụ tự động để rà quét tìm lỗ hổng hoặc các thư mục quản trị ẩn trên máy chủ của chúng ta.
- **Tư duy đúng:** Nhận diện ngay đây là một công cụ tự động (Bot) đang tiến hành rà quét mù quáng (Scan) hoặc dò mật khẩu. Cần cấu hình Tường lửa (Firewall) chặn ngay lập tức IP này.

**🎯 Bài tập thực chiến Tư duy SecOps (Truy tìm IP càn quét):**
- **Yêu cầu:** Viết câu lệnh LogQL để tìm ra địa chỉ IP nào phát sinh lỗi `404` nhiều nhất trên hệ thống và số lượng lỗi `404` mà IP này đã tạo ra.
- **LogQL gợi ý:**
  ```logql
  sum by (client_ip) (count_over_time({job="apache", status="404"} | regexp "^(?P<client_ip>\\S+)" [$__range]))
  ```
- **💡 Đáp án (Dữ liệu thực tế và Danh sách IOCs):**
  - **Malicious IP (Kẻ tấn công):** `208.91.156.11`
  - **Count (Tần suất):** 60 lần gây ra lỗi 404.
  - **Target URL (Mục tiêu):** `/files/logstash/logstash-1.3.2-monolithic.jar` (Kẻ này đang tìm kiếm một thư viện Java cũ có thể chứa lỗ hổng).
  - **User-Agent:** `Chef Client/10.18.2` (Sử dụng công cụ tự động hóa chứ không phải người dùng thật).
  - **Hành động của SOC:** Chặn ngay lập tức IP `208.91.156.11` trên Tường lửa (WAF).

---

## 🔬 PHẦN 3: CÁC BÀI LAB THỰC CHIẾN MỨC ĐỘ NÂNG CAO

## 🕵️ BÀI LAB 1 (DevOps - Quản trị Ứng dụng): Truy tìm Nguyên nhân Gây Sập Web (Lỗi 500)

### 🚨 Hiện tượng
Sếp gọi điện khẩn cấp báo rằng có một số khách hàng phàn nàn ứng dụng bị lỗi `500 Internal Server Error` (lỗi xuất phát từ code của máy chủ). Màn hình của họ hiện trắng xóa.

### ❓ Yêu cầu của Lab
Là một kỹ sư DevOps, bạn phải tìm ra nhanh chóng:
1. Có tổng cộng bao nhiêu sự kiện lỗi 500 đã xảy ra?
2. Các đường dẫn (URL) nào đang bị sập (lỗi code) và công cụ/trình duyệt nào gây ra lỗi đó?

### 🖥️ Câu lệnh LogQL gợi ý (Chạy trên Grafana Explore)
```logql
# Tìm kiếm tất cả các log có mã lỗi 500
{job="apache", status="500"}
```

### 💡 LỜI GIẢI THỰC TẾ (Đáp án thật từ file log và Phân tích IOCs):
*   **Tổng số lượng lỗi 500:** **3 lần**
*   **Chi tiết Báo cáo Lỗi (Incident Report):**
    *   **Lỗi số 1:**
        - **URL bị lỗi (Target):** `/misc/Title.php.txt` (Xảy ra 2 lần vào lúc 03:05 và 15:05 ngày 06/06).
        - **IP Gây lỗi (Client IP):** `66.249.73.135`
        - **User-Agent:** `Googlebot/2.1`
        - **Phân tích của DevOps:** Máy chủ cấu hình sai (Misconfiguration) khiến nó cố gắng chạy code PHP bên trong một file `.txt` khi Googlebot quét qua, dẫn đến Crash server.
    *   **Lỗi số 2:**
        - **URL bị lỗi (Target):** `/projects/xdotool/`
        - **HTTP Method:** `OPTIONS` (Xảy ra vào lúc 14:05 ngày 08/06).
        - **IP Gây lỗi (Client IP):** `64.131.102.243`
        - **User-Agent:** `Microsoft Office Protocol Discovery`
        - **Phân tích của DevOps:** Ứng dụng MS Office của khách hàng cố gắng kiểm tra giao thức WebDAV bằng lệnh `OPTIONS`, nhưng web server của chúng ta không hỗ trợ Method này nên trả về lỗi 500. Cần báo cho team Code bổ sung xử lý Method OPTIONS.

---

## 🔬 BÀI LAB 2 (NetOps - Băng Thông Tặc): Truy tìm Kẻ cắp băng thông ẩn danh

### 🚨 Hiện tượng
Đường truyền Internet của doanh nghiệp bị nghẽn nghiêm trọng. Đội ngũ giám sát (SOC) nhận thấy có dấu hiệu ai đó đang liên tục tải lén dữ liệu trong khung giờ từ **08:00 sáng đến 10:00 sáng ngày 07/06/2026**.

### ❓ Yêu cầu của Lab
1. **Lọc Thời Gian:** Cài đặt mốc thời gian trong Grafana từ `2026-06-07 08:00:00` đến `2026-06-07 10:00:00`.
2. Kiểm tra xem file có dung lượng tải xuống lớn nhất có phải là thủ phạm gây nghẽn mạng hay không.
3. Sử dụng hàm tính tổng dung lượng để tìm ra kẻ ngốn băng thông thực sự và công cụ hắn sử dụng.

### 🖥️ Câu lệnh LogQL gợi ý (Chạy trên Grafana Explore)
```logql
# BƯỚC 1: Tìm ra con số dung lượng lớn nhất đã từng được tải trong 1 request
max_over_time({job="apache"} | regexp ".* (?P<bytes>\\d+) \".*\" \".*\"" | unwrap bytes [$__range])

# BƯỚC 2: Tìm kẻ ngốn TỔNG CỘNG nhiều băng thông nhất (Chạy ở chế độ Instant)
topk(10, sum by (client_ip) (sum_over_time({job="apache"} | regexp "^(?P<client_ip>\\S+).* (?P<bytes>\\d+) \".*\" \".*\"" | unwrap bytes [$__range])))
```

### 💡 LỜI GIẢI THỰC TẾ (Đáp án thật từ file log và Danh sách IOCs):
*   **Nguyên nhân thực sự gây nghẽn băng thông (Thủ phạm):** `203.113.88.99` (Ngốn tổng cộng ~200MB qua hàng loạt request nhỏ).
*   **Công cụ tải (User-Agent):** `Internet Download Manager/6.38`
*   **Nhận định của Kỹ sư NetOps:** Kẻ tấn công rất tinh vi. Hắn không tải 1 file khổng lồ để tránh hệ thống giám sát cảnh báo (bỏ qua cú lừa IP nội bộ `10.0.50.100` tải file backup 99MB). Hắn dùng phần mềm IDM cắt nhỏ file thành hàng chục mảnh và tải song song đa luồng (HTTP 206 Partial Content). Nhờ dùng hàm tính tổng `sum_over_time` kết hợp gom nhóm `sum by (client_ip)`, Kỹ sư đã vạch trần được thủ đoạn "tích tiểu thành đại" này.

---

## 🕵️ BÀI LAB 3 (Security - Mức Độ Nguy Kịch): Kẻ Thù Ẩn Mình (Stealthy Attack)

### 🚨 Hiện tượng
Hệ thống IDS báo động đỏ: Có dấu hiệu một tin tặc đã xâm nhập vào hệ thống. Ban đầu, hắn lướt web như một người dùng bình thường để không đánh động ai. Nhưng sau đó, hắn đã lợi dụng một điểm yếu trong tính năng của website để tiếp cận và lấy đi những dữ liệu không được phép.

### ❓ Yêu cầu của Lab
1. Viết câu lệnh LogQL tìm kiếm tất cả các lượt truy cập **thành công (Status = 200)** và có chứa dấu hiệu của kỹ thuật Directory Traversal (lùi thư mục `../`).
2. Truy xuất toàn bộ các chỉ số xâm nhập (IOCs - Indicators of Compromise) để mở cuộc điều tra diện rộng.

### 🖥️ Câu lệnh LogQL gợi ý (Chạy trên Grafana Explore)
```logql
# BƯỚC 1: Tìm tất cả các vụ nạy két sắt thành công (Dùng Regex bắt URL Encoding)
{job="apache", status="200"} |~ "(?i)(\\.\\./|%2E%2E%2F|passwd)"
# => Kết quả ra 3 IP: Kỹ sư (10.0.50.88), Backup (10.0.50.15), và Lính gác (10.0.50.10).
# => Phân tích: Áp dụng Gợi ý "Ngựa thành Troy", loại trừ Kỹ sư và Backup. Tập trung vào Lính gác nội bộ (10.0.50.10).

# BƯỚC 2: Truy vết Lính gác (10.0.50.10) — xem nó vừa chạm vào file gì trước khi nổ LFI?
{job="apache"} |= "10.0.50.10"
# => Kết quả: Nó vừa quét file "avatar_5921.png" ngay trước khi nạy két sắt.

# BƯỚC 3: Truy vết kẻ chủ mưu - Ai đã đưa file "avatar_5921.png" vào hệ thống?
{job="apache"} |= "avatar_5921.png"
# => Bắt được IP gốc của Hacker: 185.15.20.77
```

### 💡 LỜI GIẢI THỰC TẾ (Đáp án thật từ file log và Danh sách IOCs):
*   **🚨 Trạng thái an ninh:** **ĐÃ BỊ XÂM NHẬP (COMPROMISED) MỨC ĐỘ CRITICAL**
*   **Danh sách Chỉ số Xâm nhập (IOCs):**
    *   **Attacker IP (IP Chủ mưu):** `185.15.20.77`
    *   **Internal IP (Kẻ bị lợi dụng/nổ súng):** `10.0.50.10` (Tiến trình quét file nội bộ).
    *   **Vulnerability Type (Loại lỗ hổng):** Second-Order Attack (Tấn công gián tiếp) + SSRF + Path Traversal.
    *   **Cơ chế tấn công (Kill Chain):**
        1. **Noise Upload:** Người dùng thường và Kỹ sư (`10.0.50.88`) cũng đang upload các file ảnh bình thường (`avatar_user_102.jpg`, `avatar_system_icon.png`) để tạo nhiễu.
        2. **Infection:** Hacker (`185.15.20.77`) ngấm ngầm Upload một file mã độc ngụy trang là một tệp ảnh ngẫu nhiên mang tên `avatar_5921.png`.
        3. **Execution:** Tiến trình lính gác nội bộ (`10.0.50.10`) tuần tự đọc và quét 3 file ảnh trên.
        4. **Exploitation:** Khi quét đến file `avatar_5921.png`, mã độc bị kích hoạt, ép lính gác (`10.0.50.10`) tự động truy cập URL `/projects/xdotool/view_file?path=%2E%2E%2F.../etc/passwd` để nạy két sắt. Vì chung lớp mạng nội bộ, hệ thống tin tưởng tuyệt đối (200 OK).
*   **⚠️ Ghi chú quan trọng (Mê cung 3 Tầng):** Bài Lab này thiết kế 3 IP cùng chung một lớp mạng nội bộ (`10.0.50.x`) cùng thực hiện đọc `/etc/passwd` (200 OK):
    - **Bẫy 1:** Kỹ sư hệ thống (`10.0.50.88`) cố tình gõ payload để kiểm tra lỗ hổng. (Học viên chọn IP này: Rớt vì kỹ sư tự tay gõ chứ không dùng Ngựa thành Troy).
    - **Bẫy 2:** Server Backup (`10.0.50.15`) chạy tác vụ đồng bộ cấu hình qua API. (Rớt vì đây chỉ là tác vụ tự động).
    - **Sự thật:** Tiến trình Image Scanner (`10.0.50.10`) bị nhiễm độc bởi file ảnh do Hacker (`185.15.20.77`) đưa vào. Học viên phải đối chiếu chính xác tên file bị quét ngay trước khi nổ LFI là `avatar_5921.png`, sau đó tìm ngược lại kẻ nào đã upload cái file mang tên đó thì mới thoát khỏi Mê cung!
*   **Nhận định của Đội SOC:** Đây là một cuộc tấn công APT có chủ đích và tinh vi. Kẻ tấn công (`185.15.20.77`) ban đầu truy cập trang chủ, tải file CSS để tạo log nhiễu, sau đó thử càn quét dò lỗi file `.env`, tiêm SQL Injection nhưng bị chặn (mã 404). Cuối cùng, hắn đã tìm ra điểm yếu ở tính năng `view_file` và lấy cắp mật khẩu Linux thành công. 
*   **Hành động khẩn cấp:** Cô lập (Isolate) máy chủ, vá ngay lỗ hổng Path Traversal trong mã nguồn của tính năng `view_file` và yêu cầu đổi toàn bộ mật khẩu hệ điều hành!

---
*Tài liệu được thiết kế riêng cho kỳ Seminar tháng 06/2026 bởi Lead DevOps & SOC Engineer.*
*Mọi thông số kỹ thuật dựa trên dữ liệu thật của hệ thống.*
