---
feature: cash-drawer
type: srs-screen
updated: 2026-08-12
status: draft
authors: [BA Team]
---

# Đặc tả màn hình: Cash Drawer (Kết nối két tiền & Tự động mở khi thanh toán tiền mặt)

## Screen: Cấu hình kết nối két (Store Manager)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Thiết bị / kênh kết nối | Dropdown | Có | [Trống] | - Chọn thiết bị/kênh mà két đang gắn (ví dụ máy in hóa đơn tại quầy).<br/>- Danh sách thiết bị lấy từ các thiết bị đã nhận diện trên terminal.<br/>*Lỗi:* "Vui lòng chọn thiết bị kết nối két". |
| Trạng thái kích hoạt | Toggle | Có | [Tắt] | - Bật để kích hoạt kết nối; chỉ một kết nối được bật trên mỗi terminal.<br/>- Nếu đã có kết nối khác đang bật → cảnh báo *"Đã có két đang kích hoạt, bật cái mới sẽ tắt cái cũ?"*, yêu cầu xác nhận trước khi chuyển.<br/>*Lỗi:* "Chỉ được kích hoạt một kết nối két trên mỗi máy". |
| Nút Thử mở két | Button | Có | [N/A] | - Chỉ cho phép click sau khi đã chọn thiết bị kết nối.<br/>- Gửi lệnh mở thử để kiểm tra kết nối; hiển thị kết quả ngay (mở thành công / thất bại).<br/>*Lỗi:* "Không mở được két, vui lòng kiểm tra lại thiết bị và kết nối". |
| Nút Lưu cấu hình | Button | Có | [N/A] | - Chỉ cho phép click sau khi đã chọn thiết bị kết nối.<br/>- Chặn click liên tục (disable ngay sau lượt click đầu tiên cho đến khi có phản hồi).<br/>*Lỗi:* "Lưu cấu hình thất bại, vui lòng thử lại". |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Cài đặt cửa hàng | Nhấn "Cấu hình két tiền" | Cấu hình kết nối két | Vào từ mục cài đặt thiết bị |
| Cấu hình kết nối két | Nhấn "Thử mở két" (thành công) | Cấu hình kết nối két | Hiện thông báo "Đã mở thử thành công" |
| Cấu hình kết nối két | Nhấn "Thử mở két" (thất bại) | Cấu hình kết nối két | Hiện lỗi + gợi ý kiểm tra thiết bị |
| Cấu hình kết nối két | Nhấn "Lưu" | Cài đặt cửa hàng | Quay lại sau khi lưu thành công |
| Cấu hình kết nối két | Nhấn "Quay lại" | Cài đặt cửa hàng | Không lưu thay đổi chưa commit |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- **Trạng thái thiết bị:** Hiển thị nhãn trạng thái kết nối (Đang kết nối / Mất kết nối / Chưa cấu hình) bên cạnh thiết bị đã chọn để quản lý dễ nhận biết.
- **Cảnh báo chuyển kích hoạt:** Khi bật kết nối mới trong khi cái cũ đang bật, dùng dialog xác nhận thay vì toggle trực tiếp, tránh tắt nhầm.
- **Phân quyền:** Chỉ vai trò Quản lý cửa hàng thấy màn hình này; thu ngân không vào được.

---

## Screen: Dialog mở két thủ công (Cashier)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Lý do mở két | Dropdown | Có | [Trống] | - Bắt buộc chọn một lý do: Đổi tiền lẻ / Nạp tiền vào két / Kiểm tra/đếm tiền / Khác.<br/>*Lỗi:* "Vui lòng chọn lý do mở két". |
| Ghi chú (lý do khác) | Textbox | Có (khi chọn Khác) | [Vô hiệu hóa] | - Chỉ được kích hoạt khi lý do = "Khác"; các lý do khác thì vô hiệu hóa và để trống.<br/>- Tối thiểu 3 ký tự, tối đa 200 ký tự.<br/>*Lỗi:* "Ghi chú phải có ít nhất 3 ký tự". |
| Nút Mở két | Button | Có | [N/A] | - Chỉ cho phép click sau khi đã chọn lý do hợp lệ (và nhập ghi chú nếu là "Khác").<br/>- Chặn click liên tục (disable ngay sau click đầu tiên đến khi có phản hồi).<br/>- Khi click: kiểm tra thu ngân đã đăng nhập & quyền, gửi lệnh mở, hiển thị kết quả.<br/>*Lỗi:* "Két không mở được, vui lòng thử lại". |
| Nút Hủy | Button | Có | [N/A] | - Đóng dialog, không gửi lệnh mở, không ghi lịch sử. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn thanh toán / Màn bán hàng | Nhấn "Mở két" | Dialog mở két thủ công | Mở dialog khi cần |
| Dialog mở két thủ công | Nhấn "Mở két" (thành công) | Đóng dialog (quay màn trước) | Hiện "Đã mở két", ghi lịch sử thủ công |
| Dialog mở két thủ công | Nhấn "Mở két" (thất bại) | Dialog mở két thủ công | Giữ dialog, hiện lỗi + cho thử lại |
| Dialog mở két thủ công | Nhấn "Hủy" | Đóng dialog (quay màn trước) | Không ghi lịch sử |
| Mở thủ công lỗi nhiều lần | Gợi ý | Cấu hình kết nối két | Nếu liên tục thất bại → gợi ý kiểm tra kết nối (chỉ Quản lý) |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- **Kiểm tra đăng nhập & quyền:** Nếu thu ngân chưa đăng nhập hoặc không có quyền, vô hiệu hóa nút "Mở két" và hiển thị lý do trực tiếp thay vì để người dùng nhập lý do rồi mới chặn.
- **Ghi lịch sử mọi kết quả:** Cả mở thành công lẫn thất bại đều ghi vào lịch sử mở két (loại: thủ công, lý do, thu ngân).
- **Tối ưu thao tác:** Dialog ưu tiên chọn lý do nhanh (dropdown) để thu ngân mở két trong vài giây, không cản trở việc bán hàng.

---

## Screen: Lịch sử mở két (Store Manager)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Lọc theo khoảng ngày | Date Range Picker | Không | [Hôm nay] | - Chọn ngày bắt đầu và kết thúc; kết thúc ≥ bắt đầu.<br/>*Lỗi:* "Ngày kết thúc phải sau hoặc bằng ngày bắt đầu". |
| Lọc theo thu ngân | Dropdown (đa chọn) | Không | [Tất cả] | - Lọc danh sách theo người mở; mặc định hiển thị tất cả thu ngân trong khoảng ngày. |
| Lọc theo loại mở | Dropdown (đa chọn) | Không | [Tất cả] | - Lựa chọn: Tự động / Thủ công; mặc định hiển thị cả hai. |
| Lọc theo kết quả | Dropdown (đa chọn) | Không | [Tất cả] | - Lựa chọn: Thành công / Thất bại / Đã mở sẵn; mặc định hiển thị tất cả. |
| Bảng danh sách mở két | Data Table | N/A | [N/A] | - Cột: Thời gian, Người mở, Loại mở, Lý do / Mã đơn hàng, Kết quả, Chi tiết lỗi.<br/>- Mặc định sắp xếp mới nhất trên cùng.<br/>- Phân trang khi danh sách dài; hỗ trợ cuộn dọc trong vùng bảng. |
| Nút Xuất báo cáo | Button | Không | [N/A] | - Xuất danh sách đang lọc ra báo cáo (PDF/Excel) — tùy theo OQ-04.<br/>*Lỗi:* "Xuất báo cáo thất bại, vui lòng thử lại". |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Cài đặt cửa hàng | Nhấn "Lịch sử mở két" | Lịch sử mở két | Vào từ mục theo dõi thiết bị |
| Lịch sử mở két | Thay đổi bộ lọc | Lịch sử mở két | Tải lại bảng theo bộ lọc |
| Lịch sử mở két | Nhấn 1 dòng (chi tiết) | Dialog chi tiết bản ghi | Xem đầy đủ thông tin một lần mở |
| Lịch sử mở két | Nhấn "Xuất báo cáo" | Tải file báo cáo | Xuất theo bộ lọc hiện tại |
| Lịch sử mở két | Nhấn "Quay lại" | Cài đặt cửa hàng | Quay lại màn trước |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- **Trạng thái tải dữ liệu:** Khi thay đổi bộ lọc, hiển thị skeleton/spinner cho bảng.
- **Dữ liệu theo dõi:** Mỗi dòng ghi rõ người mở và thời gian để dễ kiểm tra; dòng thất bại được đánh dấu trực quan (màu cảnh báo) để quản lý chú ý.
- **Chỉ xem, không xóa:** Thu ngân không sửa/xóa lịch sử; chỉ Quản lý có quyền xem và xuất. Log mở két không cho xóa (BR-10).
