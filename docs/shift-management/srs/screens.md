---
feature: shift-management
type: srs-screen
updated: 2026-09-08
status: draft
authors: [BA Team]
---

# Đặc tả màn hình: Shift Management (Bật/Tắt Quản lý ca)

## Screen: Cài đặt > Quản lý ca

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Công tắc "Quản lý ca" | Toggle | — | Theo cấu hình cửa hàng | - Hiển thị trạng thái hiện tại cho **mọi vai trò** (BR-004).<br/>- Chỉ kích hoạt với tài khoản có quyền `MANAGE_SHIFT_SETTING` (BR-003); vai trò khác: toggle vô hiệu kèm nhãn *"Chỉ quản lý được thay đổi"*.<br/>- Không có cấu hình phụ nào khác trên màn này (BR-002). |
| Dialog xác nhận bật | Dialog | — | — | - Hiện khi kéo toggle sang Bật: *"Từ thời điểm này, nhân viên phải mở ca trước khi thực hiện giao dịch có dòng tiền."*<br/>- Nút **Xác nhận** / **Bỏ qua**. Bỏ qua: hoàn về Tắt, không đổi gì.<br/>- Xác nhận: lưu và hiệu lực ngay (BR-007). |
| Dialog chặn tắt | Dialog + bảng | — | — | - Hiện khi tắt mà còn ca mở (BR-009): *"Không thể tắt Quản lý ca khi vẫn còn {N} ca đang mở. Vui lòng đóng tất cả ca trước khi tiếp tục."*<br/>- Danh sách ca đang mở: người mở / thời điểm mở / thiết bị; mỗi dòng có nút **"Đóng ca"**.<br/>- Khi danh sách rỗng (đã đóng hết): thao tác tắt thực hiện bình thường. |
| Thông báo cấu hình vừa đổi | Toast + làm mới | — | — | - Khi hai quản lý đổi gần như đồng thời (BR-017): người còn lại nhận *"Cấu hình vừa được {NGƯỜI} cập nhật"* và giá trị trên màn làm mới. |
| Thông báo kết quả | Toast | — | — | - *"Đã bật Quản lý ca — hiệu lực ngay"* / *"Đã tắt Quản lý ca"*.<br/>- Kèm dòng thời điểm hiệu lực từ audit mốc: *"Hiệu lực: 08/09/2026 14:02"* (BR-018). |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Cài đặt > Quản lý ca | Kéo toggle Bật + Xác nhận | (trên chỗ) trạng thái Bật + toast | Hiệu lực ngay (BR-007) |
| Cài đặt > Quản lý ca | Kéo toggle Bật + Bỏ qua | (trên chỗ) giữ trạng thái Tắt | Không đổi cấu hình |
| Cài đặt > Quản lý ca | Kéo toggle Tắt khi còn ca mở | Dialog chặn tắt | BR-009 |
| Dialog chặn tắt | Nhấn "Đóng ca" trên một dòng | Màn đóng ca của ca đó | Quản lý đóng ca hộ — bắt buộc lý do (BR-009); quy trình thuộc use case Đóng ca |
| Dialog chặn tắt | Đóng hết ca rồi tắt lại | (trên chỗ) trạng thái Tắt + toast | Mốc tắt ghi audit (BR-018) |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Không có nút "tắt kèm tự đóng ca"** ở giai đoạn 1 — tắt luôn phải qua dialog chặn nếu còn ca (Mục 5.3 spec).
- **Vô quyền:** toggle readonly, không hiện dialog xác nhận khi thử thao tác (BR-003).
- **Sự kiện realtime:** nếu cấu hình đổi từ thiết bị khác trong lúc mở màn, giá trị được làm mới ngay tại chỗ (không đá người dùng khỏi dialog đang mở — BR-017).

---

## Screen: Yêu cầu mở ca (overlay trên màn bán hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Thông điệp yêu cầu | Overlay toàn màn | — | — | - Hiện khi vào màn bán hàng với cấu hình Bật mà người dùng chưa có ca (BR-005): *"Chưa có ca đang mở. Mở ca để tiếp tục bán hàng."*<br/>- Mọi control bán hàng phía sau bị vô hiệu — không có đường bán hàng không ca. |
| Nút "Mở ca" | Button (primary) | — | — | - Đi tới form mở ca (chi tiết thuộc use case Mở ca).<br/>- Sau khi mở thành công: overlay đóng, vào màn bán hàng bình thường; đơn đang tạo dở (nếu có) giữ nguyên (BR-013). |
| Nút "Để sau" | Button (secondary) | — | — | - Rời màn bán hàng về màn chính — **không phải** bỏ qua yêu cầu ca; quay lại màn bán vẫn gặp overlay. |
| Xử lý cấu hình đổi trong lúc overlay | (hệ thống) | — | — | - Nếu cấu hình đổi sang Tắt (realtime): overlay tự đóng, bán hàng tự do (BR-015).<br/>- Nếu vẫn Bật: overlay giữ nguyên. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn bán hàng (Bật, chưa có ca) | Vào màn | Overlay yêu cầu mở ca | BR-005 — hệ thống không tự mở |
| Overlay | Nhấn "Mở ca" | Form mở ca | Use case Mở ca |
| Form mở ca | Mở ca thành công | Màn bán hàng | Đơn dở giữ nguyên (BR-013) |
| Overlay | Nhấn "Để sau" | Màn chính | Không bán được cho đến khi mở ca |
| Thanh toán đơn dở | Bị chặn khi chưa có ca | Overlay yêu cầu mở ca | Giữ giỏ hàng, mở ca xong thanh toán tiếp (BR-013) |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Một cửa duy nhất:** overlay không thể đóng bằng nút X hay click nền — chỉ "Mở ca" hoặc rời màn.
- **Không mất dữ liệu:** giỏ hàng/đơn nháp luôn giữ nguyên xuyên suốt overlay (BR-013).
- Copy thông điệp ngắn gọn, tránh trách móc: hướng dẫn hành động tiếp theo thay vì nêu lỗi.

---

## Screen: Banner trạng thái ca (màn bán hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Nội dung banner | Thanh trạng thái | — | Theo ca người dùng | - Khi Bật và có ca: *"Ca của {NGƯỜI} — mở lúc {GIỜ}"*.<br/>- Khi Bật và chưa có ca: *"Chưa mở ca"* — nhấn dẫn tới mở ca (kèm overlay Mở ca).<br/>- Khi Tắt: banner ẩn hoàn toàn (BR-010). |
| Xem chi tiết ca | Nhấn banner | — | — | - Mở drawer chi tiết ca hiện tại (doanh thu tạm tính, giao dịch của ca — chi tiết thuộc use case Lịch sử ca). |
| Nút "Đóng ca" (trong banner) | Button | — | — | - Chỉ hiện với ca của **chính người dùng**; ca người khác không có nút này trên banner (đóng ca hộ thực hiện từ màn chức năng ca — BR-009). |
| Cập nhật trạng thái | (hệ thống) | — | — | - Cập nhật theo thời gian thực khi cấu hình/ca đổi (BR-015); nếu trễ thì đối chiếu lại ở giao dịch kế tiếp. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn bán hàng | Nhấn banner | Drawer chi tiết ca hiện tại | Chỉ khi Bật |
| Banner (ca của mình) | Nhấn "Đóng ca" | Màn đóng ca | Use case Đóng ca |
| Banner ("Chưa mở ca") | Nhấn banner | Overlay yêu cầu mở ca | BR-005 |

### 3. Các điểm lưu ý trên trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- Banner mỏng, không che khu vực bán hàng; màu trạng thái chỉ là hỗ trợ, luôn kèm chữ rõ nghĩa.
- Thời gian hiển thị theo giờ địa phương cửa hàng, một mốc thống nhất từ máy chủ.

---

## Screen: Màn chức năng ca khi cấu hình Tắt (khóa + tooltip)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Mục Mở ca / Đóng ca / Đối soát tiền | Menu item — khóa | — | — | - Khi Tắt: hiển thị ở trạng thái khóa kèm tooltip *"Quản lý ca đang tắt. Liên hệ quản lý để bật"* (BR-010, OQ-03).<br/>- Khác màn bán hàng (ẩn điểm nhanh), tại đây giữ hiển thị khóa để quản lý tìm được chỗ bật lại. |
| Mục Lịch sử ca | Menu item — hoạt động | — | — | - Luôn truy cập được kể cả khi Tắt: xem chỉ đọc dữ liệu ca đã phát sinh (BR-011). |
| Nhãn trạng thái trang | Nhãn | — | Theo cấu hình | - *"Quản lý ca: Đang TẮT"* hiển thị cho mọi nhân viên (BR-004); quản lý nhấn vào được dẫn tới Cài đặt > Quản lý ca. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Menu chức năng ca | Nhấn "Lịch sử ca" | Danh sách lịch sử ca | Hoạt động cả khi Tắt (BR-011) |
| Menu chức năng ca | Thử nhấn mục khóa | Tooltip giải thích | Không điều hướng đi đâu; quản lý được gợi ý tới Cài đặt |
| Nhãn trạng thái | Nhấn (quản lý) | Cài đặt > Quản lý ca | Để bật lại |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Khóa chứ không ẩn** tại màn này (kết quả OQ-03): nhân viên hiểu tính năng tồn tại, giảm câu hỏi "tính năng biến đâu rồi".
- Lịch sử ca hiển thị nhãn "ngoài ca" cho các giao dịch phát sinh ngoài giai đoạn quản lý ca, giúp đối chiếu (BR-008).
