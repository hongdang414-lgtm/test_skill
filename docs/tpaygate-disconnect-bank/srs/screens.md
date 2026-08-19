---
type: srs-screens
feature: tpaygate-disconnect-bank
updated: 2026-07-31
---

## Screen: disconnect-bank-dialog (Dialog xác nhận ngắt kết nối ngân hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề Dialog | Label | — | "Xác nhận ngắt kết nối" | - Cố định. |
| Nội dung xác nhận | Label (multiline) | — | Dynamic | - Nội dung động: _"Bạn có chắc chắn muốn ngắt kết nối tài khoản **{AccountNo}** — **{AccountName}** tại **{BankName}**? Sau khi ngắt, bạn sẽ không thể tạo hóa đơn thanh toán mới với tài khoản này cho đến khi tạo kết nối lại."_ — BR-02. |
| Cảnh báo bổ sung | Alert Box (Warning) | — | Hiển thị | - Nội dung cảnh báo: _"Các hóa đơn đã tạo trước đó với tài khoản này vẫn có thể nhận thanh toán bình thường. Chỉ việc tạo hóa đơn thanh toán mới bị chặn."_ — BR-05. |
| Nút **"Ngắt kết nối"** | Button (Primary, Danger) | — | Enabled | - Yêu cầu người dùng có quyền Quản lý cửa hàng / Chủ cửa hàng — BR-08.<br/>- Khi nhấn: lập tức disable nút và hiển thị icon loading (spinner) để chống double-submit.<br/>- Gọi `POST /api/v1/public-api/config-bank/disconnect` với body `{ "configBankId": "..." }` và header ký HMAC-SHA256 chuẩn POST — BR-03.<br/>- Nếu API lỗi HTTP 403 `#TPayGate: configBankId does not exist or is disconnected.` → tự động cập nhật DB sang `IsConnected = false` và báo thành công đồng bộ — BR-07.<br/>- Nếu timeout / mất mạng → không tự động retry, gọi `GET /config-bank/list` để xác nhận trạng thái thực tế — BR-06.<br/>*Lỗi chung:* _"Ngắt kết nối thất bại. Vui lòng thử lại sau."_ |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Đóng Dialog, quay lại Danh sách liên kết ngân hàng. Không thay đổi bất kỳ trạng thái hay dữ liệu nào. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog ngắt kết nối | Nhấn **"Ngắt kết nối"** (thành công) | Danh sách liên kết (cập nhật trạng thái) | Hiển thị Toast: _"Đã ngắt kết nối tài khoản {AccountNo}."_ Dòng liên kết tương ứng chuyển sang trạng thái "Đã ngắt kết nối" — BR-04. |
| Dialog ngắt kết nối | Nhấn **"Ngắt kết nối"** (lỗi chữ ký / API) | Giữ nguyên Dialog + Thông báo lỗi | Hiển thị lỗi inline hoặc alert trong Dialog để người dùng thử lại hoặc hủy. |
| Dialog ngắt kết nối | Nhấn **"Hủy"** | Danh sách liên kết | Đóng Dialog. |
| Dialog ngắt kết nối | Nhấn vùng ngoài (backdrop) | Danh sách liên kết | Tương đương hành động nút "Hủy". |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Ngăn chặn gửi yêu cầu kép (Anti Double-Submit):** Ngay sau khi người dùng nhấn nút "Ngắt kết nối", hệ thống lập tức khóa nút (disabled) và hiển thị biểu tượng tải. Chỉ mở lại nút nếu T-PayGate trả về lỗi xác định (sai chữ ký, lỗi hệ thống ngân hàng).
- **Nhấn ngoài vùng Dialog (Backdrop Click):** Thao tác click ngoài vùng Dialog được xử lý an toàn như thao tác "Hủy", giúp người dùng dễ dàng thoát Dialog mà không vô tình ngắt kết nối.
- **Phân biệt màu sắc trực quan:** Nút "Ngắt kết nối" sử dụng màu đỏ (Danger / Alert) nhằm nhấn mạnh đây là thao tác quan trọng có ảnh hưởng đến luồng thu tiền của cửa hàng.

---

## Screen: bank-link-item-disconnect-state (Trạng thái dòng liên kết sau khi ngắt kết nối trên UI Danh sách)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Badge Trạng thái | Badge (readonly) | — | "Đã ngắt kết nối" | - Hiển thị badge màu xám (Gray/Muted): **"Đã ngắt kết nối"** khi `IsConnected = false` — BR-01. |
| Nút hành động **"Ngắt kết nối"** | Menu Item | — | Ẩn / Disabled | - Khi dòng ở trạng thái `IsConnected = false`, hành động "Ngắt kết nối" bị ẩn hoàn toàn trong menu thao tác — BR-01. |
| Nút hành động **"Xóa khỏi danh sách"** | Menu Item (Danger) | — | Hiển thị | - Chỉ hiển thị đối với các liên kết có `IsConnected = false`.<br/>- Cho phép người dùng ẩn record khỏi giao diện hiển thị để làm sạch danh sách.<br/>- **Lưu ý quan trọng:** Hành động này chỉ đánh dấu ẩn trên UI/cờ hiển thị phía client; **tuyệt đối không xóa bản ghi (`DELETE`) trong cơ sở dữ liệu** để đảm bảo khả năng đối soát hóa đơn — BR-04. |
| Thông tin chú giải (Tooltip / Hint) | Tooltip | — | — | - Khi người dùng rẽ chuột (hover) vào badge hoặc dòng đã ngắt kết nối, hiển thị chú giải: _"Tài khoản này đã ngắt kết nối. Không thể tạo hóa đơn mới. Để sử dụng lại, vui lòng chọn 'Thêm liên kết ngân hàng'."_ — BR-05, BR-10. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Danh sách liên kết | Nhấn **"Xóa khỏi danh sách"** trên dòng đã ngắt | Danh sách liên kết (đã làm sạch) | Record bị ẩn khỏi UI, giữ trong DB với `IsConnected = false`. |
| Danh sách liên kết | Nhấn **"Xem chi tiết"** trên dòng đã ngắt | Chi tiết liên kết (Readonly) | Hiển thị thông tin tài khoản và ngày ngắt kết nối. |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Không có hành động "Kết nối lại" trên dòng đã ngắt:** Do T-PayGate không hỗ trợ tái kích hoạt một `ConfigBankId` đã bị disconnect, hệ thống không hiển thị nút "Kết nối lại" trên dòng này. Người dùng muốn kết nối lại tài khoản phải đi qua luồng "Thêm liên kết ngân hàng" mới — BR-10.
- **Bộ lọc danh sách theo trạng thái:** Trong màn hình Danh sách liên kết ngân hàng, hỗ trợ bộ lọc nhanh "Đang hoạt động" (`IsConnected = true`) và "Đã ngắt kết nối" (`IsConnected = false`) giúp Quản lý dễ dàng đối soát khi số lượng tài khoản nhiều.
