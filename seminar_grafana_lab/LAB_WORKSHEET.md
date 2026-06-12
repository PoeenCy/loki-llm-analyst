# 📝 BÀI TẬP THỰC HÀNH GRAFANA & LOKI (WORKSHEET)

**Họ và tên Học viên:** ..........................................................................
**Nhóm/Mã số:** .................................................................................

*Hướng dẫn: Sử dụng Grafana (Menu Explore -> Data source Loki) để truy vấn bộ dữ liệu Log thực tế. Tự tìm ra các manh mối dựa vào các gợi ý và điền đáp án vào các ô trống. Hãy khoanh tròn vào đáp án Trắc nghiệm mà bạn cho là phân tích chính xác nhất.*

---

## 🕵️ BÀI LAB 1 (DevOps): TRUY TÌM NGUYÊN NHÂN GÂY SẬP WEB

**Tình huống:** Khách hàng phàn nàn ứng dụng đột nhiên bị lỗi trắng trang. Hệ thống ghi nhận có 2 nhóm lỗi 500 khác nhau.

**💡 Gợi ý phá án (Cấp độ 1 - Chi tiết):**
- Mã lỗi HTTP đại diện cho "Internal Server Error" (Lỗi máy chủ nội bộ) là số 500.
- Cú pháp cơ bản của LogQL để lọc là `{job="apache", status="500"}`. Hãy chạy lệnh để tìm 3 dòng log lỗi.

### Phần 1: Thu thập Chỉ số (Điền vào chỗ trống)
1. Có tổng cộng `[ Số lượng (ví dụ: 5) ]` sự kiện lỗi sập server xuất hiện trong toàn bộ log.
2. **Sự cố 1 (Lỗi file văn bản - Xuất hiện 2 lần):** Bắt nguồn từ IP `[ IPv4 (ví dụ: 1.1.1.1) ]` cố gắng đọc file có đường dẫn là `[ Đường dẫn file (ví dụ: /index.html) ]`. Danh tính (User-Agent) của kẻ gọi đến là: `[ User-Agent (ví dụ: Googlebot/2.1) ]`
3. **Sự cố 2 (Lỗi giao thức - Chỉ xuất hiện 1 lần):** Bắt nguồn từ IP `[ IPv4 (ví dụ: 1.1.1.1) ]` gọi đến thư mục `[ Đường dẫn thư mục (ví dụ: /folder/) ]` bằng một phương thức (HTTP Method) rất lạ là: `[ HTTP Method (ví dụ: GET) ]`

### Phần 2: Tư duy phân tích (Trắc nghiệm)
**Câu hỏi 1 (Tìm hiểu Bản chất):** Phân tích sự cố số 1, tại sao một công cụ tìm kiếm (như Googlebot) truy cập vào một file văn bản (`.php.txt`) lại có thể làm sập máy chủ (Lỗi 500)?
- [ ] **A.** Máy chủ bị cấu hình sai, cố gắng biên dịch nội dung của một file văn bản thô dưới dạng mã lệnh PHP.
- [ ] **B.** Hệ thống đang bị tấn công từ chối dịch vụ phân tán (DDoS) thông qua các lỗ hổng chưa vá của máy chủ Apache.
- [ ] **C.** Googlebot đang bị lợi dụng để cố tình tiêm các đoạn mã độc hại vào máy chủ thông qua các truy vấn siêu lớn.
- [ ] **D.** Máy chủ đã cạn kiệt hoàn toàn dung lượng bộ nhớ RAM nên không thể cấp phát tài nguyên để đọc nội dung file tĩnh.

**Câu hỏi 2 (Giải quyết Vấn đề):** Phân tích sự cố số 2, với phương thức `OPTIONS` từ User-Agent `Microsoft Office Protocol Discovery`. Giải pháp xử lý triệt để sự cố này là gì?
- [ ] **A.** Thiết lập cấu hình Tường lửa (Firewall) hoặc hệ thống WAF để chặn vĩnh viễn địa chỉ IP đã gửi truy vấn này.
- [ ] **B.** Yêu cầu đội Lập trình bổ sung mã xử lý hoặc cấu hình lại Web Server để chấp nhận phương thức truy cập OPTIONS.
- [ ] **C.** Tiến hành khởi động lại (Restart) toàn bộ cụm máy chủ Web nhằm xóa sạch bộ nhớ đệm (Cache) đang bị lỗi.
- [ ] **D.** Khẩn trương cài đặt phần mềm diệt virus trên máy chủ để cách ly các file độc hại do Microsoft Office tạo ra.

---

## 🔬 BÀI LAB 2 (NetOps): TRUY TÌM KẺ CẮP BĂNG THÔNG ẨN DANH

**Tình huống:** Đường truyền Internet của doanh nghiệp bị nghẽn nghiêm trọng. Đội ngũ giám sát phát hiện có ai đó đang liên tục tải lén dữ liệu trong khung giờ từ **08:00 sáng đến 10:00 sáng ngày 07/06/2026**.

**💡 Gợi ý phá án:**
- **Thu hẹp phạm vi:** Hãy cài đặt mốc thời gian (Time Range) trong Grafana đúng với khung giờ xảy ra sự cố (Từ `2026-06-07 08:00:00` đến `2026-06-07 10:00:00`).
- **Nghi binh (Nhiễu):** Bạn có thể thử đếm tổng số lượng request (hàm `count_over_time`) hoặc tìm những IP bị lỗi 404 nhiều nhất xem chúng có phải thủ phạm không. Biết đâu thủ phạm lại là kẻ nói nhiều nhất? 
- **Phân tích:** Đừng chỉ nhìn vào bề nổi của một cá thể (ví dụ 1 request to nhất), hãy nhìn vào tổng thể hành vi của chúng trong cả một khoảng thời gian dài. Một giọt nước không làm tràn ly, nhưng một cơn mưa rào thì có thể. Hãy dùng phép cộng dồn để tìm ra sự thật!

### Phần 1: Phân tích Dữ liệu (Điền vào chỗ trống)
1. Nguyên nhân thực sự gây nghẽn băng thông là do IP nào?: `[ IPv4 (ví dụ: 1.1.1.1) ]`
2. Kẻ cắp đã dùng phần mềm (User-Agent) nào để liên tục cào dữ liệu qua mặt hệ thống?: `[ User-Agent (ví dụ: Internet Download Manager/6.38) ]`

### Phần 2: Tư duy phân tích (Trắc nghiệm)
**Câu hỏi 3 (Bản chất Sự cố):** Tại sao file có dung lượng lớn nhất trong 1 request lại KHÔNG phải là nguyên nhân chính gây cạn kiệt băng thông trong trường hợp này?
- [ ] **A.** Vì request lớn nhất bị hệ thống tự động ngắt kết nối trước khi tải xong nên không ảnh hưởng.
- [ ] **B.** Sử dụng công cụ cắt nhỏ file (Mã 206 - Partial Content) giúp kẻ tấn công tải song song đa luồng liên tục với lượng dữ liệu tích tiểu thành đại, lách qua các bộ lọc cảnh báo chỉ bắt các "request đơn lẻ có dung lượng bự".
- [ ] **C.** Kẻ tấn công cố tình tải file bự bị rớt mạng giữa chừng nên hệ thống tự động chuyển sang tải các trang lỗi 404.
- [ ] **D.** Do tường lửa (Firewall) của doanh nghiệp đã chặn hoàn toàn các tiến trình sao lưu nội bộ.

**Câu hỏi 4 (Giải quyết Vấn đề):** Giải pháp kiến trúc nào TỐI ƯU NHẤT để hệ thống không lặp lại tình trạng tắc nghẽn này trong tương lai?
- [ ] **A.** Thực hiện lệnh xóa hoặc ẩn ngay lập tức tệp tin bị lỗi đó trên máy chủ gốc để ngăn chặn triệt để mọi hành vi tải xuống.
- [ ] **B.** Cấu hình quy tắc Tường lửa (Firewall) khóa vĩnh viễn IP đó, đồng thời chặn hoàn toàn các truy vấn từ công cụ ngầm Wget.
- [ ] **C.** Áp dụng ngay chính sách giới hạn tốc độ mạng (Bandwidth Throttling) đối với tất cả các máy chủ Web của toàn bộ công ty.
- [ ] **D.** Tối ưu phân tải bằng cách đẩy các tệp tĩnh siêu nặng lên Hệ thống Phân phối Nội dung (CDN) hoặc bộ lưu trữ đám mây S3.

---

## 🛡️ BÀI LAB 3 (SecOps): KẺ THÙ ẨN MÌNH (STEALTHY ATTACK)

**Tình huống:** Hệ thống IDS báo động đỏ: Có dấu hiệu một tin tặc đã xâm nhập vào hệ thống. Ban đầu, hắn lướt web như một người dùng bình thường để không đánh động ai. Nhưng sau đó, hắn đã lợi dụng một điểm yếu trong tính năng của website để tiếp cận và lấy đi những dữ liệu không được phép.

**💡 Thông báo Hệ thống (Alert):**
Hệ thống đang trong thời gian cập nhật cấu hình. Lợi dụng lúc này, một "con ngựa thành Troy" đã lách qua các chốt kiểm soát tĩnh. Kẻ gian không tự tay nổ súng mà mượn tay chính lính gác nội bộ để mở két sắt mật khẩu. Cảnh báo: Kẻ trực tiếp cầm súng chưa chắc đã là chủ mưu!

### Phần 1: Thu thập Chỉ số (Điền vào chỗ trống)
1. Địa chỉ IP của Hacker thực sự là: `[ IPv4 (ví dụ: 1.1.1.1) ]`
2. Đường dẫn (URL) chứa điểm yếu bị khai thác để đánh cắp dữ liệu là gì?: `[ Đường dẫn URL (ví dụ: /path/to/api) ]`
3. Thời điểm (Timestamp) chính xác mà vụ đánh cắp dữ liệu xảy ra: `[ Timestamp (ví dụ: 07/Jun/2026:10:18:05 +0000) ]`
4. Trình duyệt (User-Agent) mà Hacker sử dụng để ngụy trang giống người dùng bình thường là gì?: `[ User-Agent (ví dụ: Chrome/120.0.0.0) ]`

### Phần 2: Tư duy phân tích (Trắc nghiệm)
**Câu hỏi 5 (Tìm hiểu Bản chất):** Dựa vào dấu vết để lại, thủ đoạn tinh vi nhất (Second-Order Attack) mà tin tặc đã sử dụng để đánh lừa hệ thống giám sát là gì?
- [ ] **A.** Sử dụng các ký tự đặc biệt ngẫu nhiên để làm tràn bộ đệm của máy chủ, khiến hệ thống giám sát tự động bị sập trước khi lưu log.
- [ ] **B.** Lợi dụng lỗ hổng giao thức mạng để làm giả mạo địa chỉ IP nguồn, khiến hệ thống giám sát ghi nhận sai lệch vị trí của kẻ tấn công.
- [ ] **C.** Cấy mã độc vào một tệp tin hợp lệ và đợi một tiến trình nội bộ đáng tin cậy tự động thực thi, giúp che giấu hoàn toàn sự xâm nhập.
- [ ] **D.** Mã hóa toàn bộ gói tin bằng thuật toán mã hóa bất đối xứng, ngăn chặn hệ thống giám sát đọc được bất kỳ nội dung nào bên trong nó.

**Câu hỏi 6 (Giải quyết Vấn đề):** Ngay khi phát hiện ra hệ thống đã bị lộ lọt file chứa mật khẩu cốt lõi, bước xử lý sự cố (Incident Response) ĐẦU TIÊN VÀ QUAN TRỌNG NHẤT mà bạn phải làm là gì?
- [ ] **A.** Lập tức đăng tải các bài viết đính chính trên mạng xã hội nhằm mục đích trấn an dư luận và thông báo cho khách hàng về sự cố.
- [ ] **B.** Nhanh chóng tiến hành format (xóa sạch toàn bộ) ổ cứng của máy chủ bị nhiễm bệnh để ngăn chặn triệt để mã độc lây lan nội bộ.
- [ ] **C.** Tạm thời cô lập máy chủ khỏi mạng lưới Internet, khẩn trương vá lỗ hổng mã nguồn và tiến hành đổi toàn bộ mật khẩu hệ thống.
- [ ] **D.** Chủ động gửi email thương lượng trực tiếp với nhóm Hacker, yêu cầu họ tuyệt đối không được phát tán file dữ liệu ra bên ngoài.
