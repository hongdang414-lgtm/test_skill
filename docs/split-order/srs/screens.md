---
type: srs-screens
feature: split-order
updated: 2026-07-21
---

## Screen: order-detail (Chi tiết đơn hàng — Điểm vào tách đơn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên màn hình | Tên trường / Thành phần | Control Type | Bắt buộc | Mặc định | Rules | Lỗi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Chi tiết đơn hàng | Nút **"Tách đơn"** | Button (Primary) | — | Hiển thị/Ẩn theo quyền | - **Ràng buộc hiển thị:** Chỉ hiển thị khi actor có quyền `SPLIT_ORDER`.<br/>- **Ràng buộc kích hoạt:** Chỉ `enabled` khi đơn hàng ở trạng thái `ACTIVE` và tổng SL sản phẩm ≥ 2.<br/>- Khi bị disabled: hiển thị tooltip giải thích lý do (thiếu quyền / đơn không đủ điều kiện / đang có payment pending). | - Thiếu quyền: Ẩn nút hoàn toàn.<br/>- Đơn không đủ SL: Tooltip _"Đơn hàng cần ít nhất 2 sản phẩm để tách."_<br/>- Đang pending payment: Tooltip _"Không thể tách khi có giao dịch đang xử lý."_<br/>- Đơn đã thanh toán/hủy: Tooltip _"Chỉ tách được đơn đang hoạt động."_ |
| Chi tiết đơn hàng | Thông tin đơn hàng (readonly) | Info Panel | — | Data từ API | - Hiển thị: Mã đơn, Bàn/Khu vực, Thời gian tạo, Tổng tiền hiện tại, Trạng thái.<br/>- Dữ liệu chỉ đọc, không cho phép chỉnh sửa tại đây. | — |
| Chi tiết đơn hàng | Danh sách sản phẩm (readonly) | List | — | Data từ API | - Mỗi dòng: Tên SP, SL, Đơn giá, Thành tiền, Giảm giá item.<br/>- Combo hiển thị tên combo + icon phân biệt, không expand thành phần. | — |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Chi tiết đơn hàng | Nhấn **"Tách đơn"** (enabled) | Màn hình Chọn sản phẩm tách | Mở dưới dạng Bottom Sheet (mobile) hoặc Modal (desktop) |
| Chi tiết đơn hàng | Nhấn **"Tách đơn"** (disabled) | Giữ nguyên + hiện Tooltip | Không chuyển màn |

### 3. Lưu ý UI/UX & Edge Logic

- **Trạng thái loading:** Khi kiểm tra eligibility, hiển thị skeleton loader trên nút "Tách đơn" (không disable button trống mà không có indicator).
- **Badge phân biệt combo:** Các sản phẩm combo hiển thị badge/icon riêng biệt (ví dụ: 🎁 hoặc label "COMBO") để thu ngân dễ nhận biết trước khi vào màn chọn tách.

---

## Screen: split-item-selector (Màn hình Chọn sản phẩm tách)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên màn hình | Tên trường / Thành phần | Control Type | Bắt buộc | Mặc định | Rules | Lỗi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Chọn SP tách | **Header** — Mã đơn gốc | Label (readonly) | — | Mã đơn gốc | Hiển thị mã đơn gốc để thu ngân tham chiếu, không chỉnh sửa. | — |
| Chọn SP tách | **Danh sách sản phẩm đơn gốc** | Scrollable List | — | Tất cả SP từ đơn gốc | - Mỗi dòng gồm: Checkbox chọn, Tên SP, Đơn giá, SL hiện có, Ô nhập SL tách.<br/>- Sản phẩm combo: Checkbox chọn cả combo, không hiển thị checkbox cho từng thành phần. Label "COMBO" phân biệt. | — |
| Chọn SP tách | **Checkbox chọn sản phẩm** | Checkbox | Không (chọn tối thiểu 1) | Unchecked | - Khi tick: Kích hoạt ô nhập SL tách cho dòng đó, mặc định SL tách = 1.<br/>- Khi bỏ tick: Xóa giá trị SL tách, không tính vào đơn mới.<br/>- **Ràng buộc:** Không được tick tất cả sản phẩm nếu sau đó đơn gốc sẽ rỗng (BR-03). | - Chọn toàn bộ: Disable nút Xác nhận + cảnh báo _"Đơn gốc phải giữ lại ít nhất 1 sản phẩm."_ |
| Chọn SP tách | **Ô nhập SL tách** | Number Input (Stepper) | Có (khi SP được chọn) | 1 | - **Kiểm tra rỗng:** Không cho phép để trống khi SP được chọn.<br/>- **Giá trị biên:** `min=1`, `max=SL_hiện_có_của_SP_đó`.<br/>- **Định dạng:** Chỉ nhận số nguyên dương, không nhận số thập phân.<br/>- **Ràng buộc phụ thuộc:** Chỉ enabled khi checkbox dòng đó được tick. | - Để trống: _"Vui lòng nhập số lượng."_<br/>- Vượt quá SL gốc: _"Số lượng tối đa là {SL_gốc}."_<br/>- Nhập 0 hoặc âm: _"Số lượng phải ít nhất là 1."_<br/>- Nhập số thập phân: Tự động làm tròn xuống hoặc báo _"Chỉ nhập số nguyên."_ |
| Chọn SP tách | **Tóm tắt lựa chọn** | Summary Panel (sticky bottom) | — | Tự động cập nhật | - Hiển thị realtime: "Đã chọn {N} sản phẩm / {M} loại để tách".<br/>- Cập nhật mỗi khi thay đổi checkbox hoặc SL tách. | — |
| Chọn SP tách | **Nút "Xem Preview"** | Button (Primary) | — | Enabled khi hợp lệ | - Enabled khi: Đã chọn ≥ 1 sản phẩm, tất cả SL hợp lệ, đơn gốc còn ≥ 1 SP.<br/>- Disabled khi vi phạm bất kỳ điều kiện trên. | - Disabled + tooltip khi chọn toàn bộ hoặc chưa chọn gì. |
| Chọn SP tách | **Nút "Huỷ"** | Button (Secondary) | — | Enabled | - Khi nhấn: Hiển thị dialog xác nhận _"Bạn có chắc muốn hủy? Mọi lựa chọn sẽ bị xóa."_<br/>- Xác nhận hủy: Đóng Bottom Sheet/Modal, đơn gốc không thay đổi. | — |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Chọn SP tách | Nhấn **"Xem Preview"** (enabled) | Preview tài chính | Chuyển sang bước preview |
| Chọn SP tách | Nhấn **"Huỷ"** → Xác nhận | Chi tiết đơn hàng | Đóng modal/bottom sheet |
| Chọn SP tách | Swipe down / Nhấn ngoài vùng (mobile) | Dialog xác nhận hủy | Ngăn thoát nhầm |

### 3. Lưu ý UI/UX & Edge Logic

- **Phân biệt combo:** Dòng sản phẩm combo được highlight bằng màu nền nhạt hoặc border riêng. Tooltip icon giải thích "Combo sẽ được tách nguyên bộ".
- **Trạng thái disabled ô SL:** Khi checkbox chưa được tick, ô nhập SL hiển thị dạng mờ (opacity 50%), không nhận input.
- **Validation realtime:** Kiểm tra ngay khi người dùng thay đổi giá trị, không đợi đến khi nhấn nút.
- **Accessibility:** Mỗi ô nhập SL có `aria-label` rõ ràng, ví dụ: "Số lượng tách cho Phở Bò".

---

## Screen: split-preview (Màn hình Preview tài chính trước khi tách)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên màn hình | Tên trường / Thành phần | Control Type | Bắt buộc | Mặc định | Rules | Lỗi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Preview tách đơn | **Panel "Đơn gốc còn lại"** | Info Panel (readonly) | — | Tính toán từ lựa chọn | - Hiển thị: Danh sách SP còn lại, Tạm tính, Giảm giá SP, Giảm giá đơn (đã phân bổ theo BR-07), Thuế, Phí DV, **Tổng cần thanh toán**.<br/>- Dữ liệu chỉ đọc. | Nếu tính toán lỗi: Hiển thị _"Không thể tính toán, vui lòng thử lại."_ |
| Preview tách đơn | **Panel "Đơn mới"** | Info Panel (readonly) | — | Tính toán từ lựa chọn | - Hiển thị: Danh sách SP tách sang, Tạm tính, Giảm giá SP (phân bổ theo BR-08), Giảm giá đơn (phân bổ theo BR-07), Thuế, Phí DV, **Tổng cần thanh toán**.<br/>- Ghi chú nhỏ: "Mã đơn mới sẽ được tạo sau khi xác nhận." | Nếu tổng không cân bằng: Hiển thị cảnh báo đỏ _"Có sai lệch tài chính, vui lòng liên hệ kỹ thuật."_ |
| Preview tách đơn | **Tổng kiểm tra cân bằng** | Validation Summary | — | Tự động | - Hiển thị: "Tổng đơn gốc + Đơn mới = {X} VNĐ (khớp với đơn gốc ban đầu {Y} VNĐ)".<br/>- Nếu sai lệch ≤ 1 VNĐ: Hiển thị ✅ và ghi chú "Chênh lệch làm tròn {Z} VNĐ đã được phân bổ vào đơn gốc". | Sai lệch > 1 VNĐ: Ẩn nút Xác nhận, hiển thị cảnh báo lỗi hệ thống. |
| Preview tách đơn | **Cảnh báo coupon không phân bổ** | Alert (Warning) | — | Hiển thị khi có coupon cố định | - **Ràng buộc phụ thuộc:** Chỉ hiển thị khi đơn gốc có coupon cố định không phân bổ theo tỷ lệ được (BR-08).<br/>- Nội dung: _"Voucher [TÊN_VOUCHER] sẽ giữ lại ở đơn gốc. Đơn mới không áp dụng voucher này."_ | — |
| Preview tách đơn | **Nút "Xác nhận Tách đơn"** | Button (Primary, Danger style) | — | Enabled (khi tài chính hợp lệ) | - Enabled khi tổng tiền cân bằng (BR-10).<br/>- Nhấn → Hiển thị dialog xác nhận lần cuối trước khi thực thi. | Disabled khi tài chính không cân bằng. |
| Preview tách đơn | **Nút "Quay lại chỉnh sửa"** | Button (Secondary) | — | Enabled | Quay lại màn hình Chọn SP tách, giữ nguyên lựa chọn trước đó. | — |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Preview tách đơn | Nhấn **"Xác nhận Tách đơn"** | Dialog xác nhận cuối | Hỏi lần cuối trước khi commit |
| Preview tách đơn | Nhấn **"Quay lại chỉnh sửa"** | Chọn sản phẩm tách | Giữ nguyên lựa chọn |
| Preview tách đơn | Dialog xác nhận → **"Đồng ý"** | Màn hình kết quả / Loading | Thực thi tách đơn |
| Preview tách đơn | Dialog xác nhận → **"Hủy"** | Preview tách đơn | Đóng dialog, giữ nguyên |

### 3. Lưu ý UI/UX & Edge Logic

- **Layout 2 cột:** Trên tablet/desktop, hiển thị đơn gốc và đơn mới song song (side-by-side). Trên mobile, xếp dọc với accordion expand/collapse.
- **Highlight thay đổi:** Các dòng giảm giá đã phân bổ lại nên được highlight màu vàng nhạt để thu ngân dễ nhận thấy sự thay đổi so với đơn gốc.
- **Loading state khi tính toán preview:** Hiển thị spinner trên các ô tổng tiền trong khi chờ API tính toán, tránh hiển thị số 0 sai lệch.
- **Số tiền định dạng:** Tất cả giá trị tiền tệ hiển thị theo định dạng `{N}.000 đ` (ví dụ: `150.000 đ`), sử dụng dấu chấm làm phân cách hàng nghìn.

---

## Screen: split-result (Màn hình Kết quả tách đơn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên màn hình | Tên trường / Thành phần | Control Type | Bắt buộc | Mặc định | Rules | Lỗi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Kết quả tách đơn | **Banner thành công** | Alert (Success) | — | Hiển thị khi thành công | Nội dung: _"Tách đơn thành công! Đơn hàng [MÃ_GỐC] đã được tách thành 2 đơn."_ | — |
| Kết quả tách đơn | **Card Đơn gốc** | Order Card | — | Data từ API response | - Hiển thị: Mã đơn gốc, danh sách SP còn lại, tổng tiền cập nhật.<br/>- Badge trạng thái: `ACTIVE`.<br/>- Nút hành động: **"Thanh toán"**, **"Xem chi tiết"**. | — |
| Kết quả tách đơn | **Card Đơn mới** | Order Card | — | Data từ API response | - Hiển thị: Mã đơn mới (được sinh tự động), danh sách SP tách sang, tổng tiền.<br/>- Badge trạng thái: `ACTIVE` + Badge phụ: `Tách từ [MÃ_GỐC]`.<br/>- Nút hành động: **"Thanh toán"**, **"Xem chi tiết"**. | — |
| Kết quả tách đơn | **Nút "Xong"** | Button (Primary) | — | Enabled | Đóng màn hình kết quả, quay về danh sách đơn hàng hoặc màn hình chính POS. | — |
| Kết quả tách đơn | **Banner lỗi** | Alert (Error) | — | Hiển thị khi thất bại | - Hiển thị khi tách thất bại (lỗi hệ thống, tài chính không cân bằng).<br/>- Nội dung: _"Tách đơn thất bại. Đơn hàng gốc không thay đổi. Vui lòng thử lại hoặc liên hệ hỗ trợ."_<br/>- Kèm nút: **"Thử lại"**, **"Đóng"**. | — |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Kết quả tách đơn | Nhấn **"Thanh toán"** (Đơn gốc) | Màn hình thanh toán đơn gốc | Thanh toán độc lập |
| Kết quả tách đơn | Nhấn **"Thanh toán"** (Đơn mới) | Màn hình thanh toán đơn mới | Thanh toán độc lập |
| Kết quả tách đơn | Nhấn **"Xem chi tiết"** | Chi tiết đơn hàng tương ứng | |
| Kết quả tách đơn | Nhấn **"Xong"** | Màn hình chính POS / DS đơn hàng | |
| Kết quả tách đơn (lỗi) | Nhấn **"Thử lại"** | Preview tách đơn | Giữ nguyên lựa chọn cũ |
| Kết quả tách đơn (lỗi) | Nhấn **"Đóng"** | Chi tiết đơn hàng gốc | Đơn gốc không thay đổi |

### 3. Lưu ý UI/UX & Edge Logic

- **Auto-redirect:** Sau khi hiển thị banner thành công 5 giây mà người dùng không tương tác, có thể auto-redirect về màn hình chính POS (cần confirm với product owner — ghi OQ-06).
- **In bill:** Ngay tại màn hình kết quả, nếu máy in kết nối, hiển thị nút **"In bill"** cho từng đơn riêng biệt.
- **Trạng thái offline:** Nếu kết nối mạng bị mất trong quá trình xử lý, hiển thị cảnh báo _"Mất kết nối. Vui lòng kiểm tra lại trạng thái đơn hàng."_ và không hiển thị kết quả thành công/thất bại chưa xác định.
