---
type: srs-screens
feature: merge-order
updated: 2026-07-23
---

## Screen: order-detail-merge (Chi tiết đơn hàng — Điểm vào ghép đơn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Nút **"Ghép đơn"** | Button (Primary) | — | Hiển thị/Ẩn theo quyền | - Chỉ hiển thị khi actor có quyền `MERGE_ORDER` (BR-03). Thiếu quyền → ẩn nút hoàn toàn.<br/>- Chỉ `enabled` khi đơn ở trạng thái `ACTIVE`, không có payment pending (BR-01, BR-05), chưa ở trạng thái `MERGED` (BR-07).<br/>- Khi disabled: hiển thị tooltip giải thích lý do.<br/>*Lỗi:* "Chỉ ghép được đơn đang hoạt động."<br/>*Lỗi:* "Không thể ghép khi có giao dịch đang xử lý."<br/>*Lỗi:* "Đơn đã được ghép trước đó." |
| Thông tin đơn hàng (Mã đơn, Bàn, Thời gian, Trạng thái) | Info Panel (readonly) | — | Data từ API | - Hiển thị thông tin tóm tắt đơn gốc. Dữ liệu chỉ đọc, không cho phép chỉnh sửa.<br/>- Nếu đơn ở trạng thái `MERGED`: hiển thị badge `ĐÃ GHÉP` bên cạnh trạng thái. |
| Danh sách sản phẩm | List (readonly) | — | Data từ API | - Mỗi dòng gồm: Tên SP, SL, Đơn giá, Thành tiền, Giảm giá item (nếu có).<br/>- Sản phẩm combo hiển thị badge "COMBO", không expand thành phần. |
| Tổng tiền (Tạm tính, Giảm giá, Thuế, Phí DV, Cần TT) | Label (readonly) | — | Tính từ API | - Tất cả giá trị tiền định dạng `{N}.000đ` (dấu chấm phân cách nghìn).<br/>- Dữ liệu tính từ Backend, Frontend chỉ hiển thị. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Chi tiết đơn hàng | Nhấn **"Ghép đơn"** (enabled) | Danh sách đơn có thể ghép | Mở Bottom Sheet (mobile) hoặc Modal (desktop) |
| Chi tiết đơn hàng | Nhấn **"Ghép đơn"** (disabled) | Giữ nguyên + hiện Tooltip | Không chuyển màn |
| Chi tiết đơn hàng | Nhấn **"Thanh toán"** | Màn hình thanh toán | Luồng thanh toán bình thường |
| Chi tiết đơn hàng | Nhấn **← Danh sách** | Danh sách đơn hàng | Quay lại trang danh sách |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Trạng thái loading:** Khi kiểm tra eligibility, hiển thị skeleton loader trên nút "Ghép đơn" (không để button trống mà không có indicator).
- **Badge MERGED:** Đơn đã bị ghép hiển thị badge `ĐÃ GHÉP` và ẩn hoàn toàn nút "Ghép đơn".
- **Badge COMBO:** Sản phẩm combo hiển thị badge riêng biệt để cashier nhận biết trước khi ghép.

---

## Screen: merge-target-selector (Danh sách đơn có thể ghép)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Header — Mã đơn gốc | Label (readonly) | — | Mã đơn gốc | - Hiển thị "Ghép từ đơn: {MÃ_ĐƠN_GỐC}" để thu ngân tham chiếu. Không chỉnh sửa. |
| Ô tìm kiếm mã đơn | Textbox (Search) | Không | Trống | - Tìm nhanh theo mã đơn hàng.<br/>- Chấp nhận ký tự alphanumeric + dấu gạch ngang.<br/>- Tìm kiếm tức thì (debounce 300ms). Placeholder: "Tìm theo mã đơn hàng..." |
| Bộ lọc Bàn/Khu vực | Dropdown (Filter) | Không | "Tất cả bàn" | - Lọc danh sách đơn theo bàn/khu vực.<br/>- Tùy chọn: "Tất cả bàn", "Cùng bàn", hoặc chọn bàn cụ thể.<br/>- Cập nhật danh sách ngay khi thay đổi giá trị bộ lọc. |
| Danh sách đơn đích (Order Cards) | Selectable List (Radio) | Có (chọn 1) | Chưa chọn | - Mỗi card hiển thị: Mã đơn, Bàn, Số SP, Tổng tiền, Thời gian tạo.<br/>- Chỉ hiển thị đơn `ACTIVE`, loại đơn gốc (BR-04), loại đơn `MERGED` (BR-07).<br/>- Tap/click để chọn (single-select, radio behavior).<br/>- Đơn khác bàn với đơn gốc: hiển thị tag "Khác bàn".<br/>- Sort mặc định: cùng bàn trước → thời gian mới nhất.<br/>*Lỗi:* "Vui lòng chọn một đơn hàng đích." |
| Panel rỗng (Empty State) | Empty State | — | Hiện khi DS rỗng | - Hiển thị khi không có đơn đích nào thỏa điều kiện.<br/>- Nội dung: Icon rỗng + "Không tìm thấy đơn hàng nào đủ điều kiện để ghép." |
| Nút **"Tiếp tục"** | Button (Primary) | — | Disabled | - Chỉ `enabled` khi đã chọn 1 đơn đích.<br/>- Nhấn → gọi API POST /orders/{source}/merge/preview, chuyển sang màn Preview.<br/>*Lỗi:* "Vui lòng chọn đơn hàng đích trước khi tiếp tục." |
| Nút **"Huỷ"** | Button (Secondary) | — | Enabled | - Đóng Bottom Sheet/Modal. Đơn gốc không thay đổi. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| DS đơn ghép | Nhấn **"Tiếp tục"** (enabled) | Preview tài chính | Gọi API preview trước khi chuyển |
| DS đơn ghép | Nhấn **"Huỷ"** | Chi tiết đơn hàng gốc | Đóng modal, đơn gốc không đổi |
| DS đơn ghép | Swipe down / Nhấn ngoài vùng | Chi tiết đơn hàng gốc | Mobile only — có dialog xác nhận |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Trạng thái loading:** Khi tải danh sách, hiển thị 3 skeleton card placeholder.
- **Tag "Khác bàn":** Đơn thuộc bàn khác có tag màu xám "Khác bàn" để cashier nhận biết nhanh.
- **Sort ưu tiên:** Đơn cùng bàn hiển thị trên cùng, đơn khác bàn xuống dưới.
- **Pull-to-refresh:** Hỗ trợ kéo để làm mới danh sách (mobile).

---

## Screen: merge-preview (Preview tài chính đơn đích sau ghép)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Panel **"Đơn gốc (sẽ ghép)"** | Info Panel (readonly) | — | Data từ đơn gốc | - Hiển thị: Mã đơn gốc, Bàn, DS sản phẩm, Tạm tính.<br/>- Trạng thái chuyển đổi: `ACTIVE → MERGED` (hiển thị mũi tên).<br/>- Dữ liệu chỉ đọc. |
| Panel **"Đơn đích (sau ghép)"** | Info Panel (readonly) | — | Tính toán từ API | - Hiển thị: Mã đơn đích, Bàn, DS sản phẩm (cũ + mới gộp), Tạm tính, Giảm giá SP (BR-09), Giảm giá đơn (BR-11), Thuế, Phí DV, Tổng cần thanh toán (BR-12).<br/>- SP mới từ đơn gốc được highlight (background nhạt + badge "MỚI GHÉP").<br/>- SP trùng được gộp SL: hiển thị SL tổng (BR-13).<br/>*Lỗi:* "Không thể tính toán, vui lòng thử lại." |
| Cảnh báo giảm giá bị hủy | Alert (Warning) | — | Hiển thị có điều kiện | - Chỉ hiện khi đơn gốc có giảm giá order-level (BR-10).<br/>- Nội dung: "Giảm giá đơn hàng [TÊN_GG] ({GIÁ TRỊ}) của đơn gốc sẽ bị hủy sau khi ghép." |
| Cảnh báo khác bàn | Alert (Warning) | — | Hiển thị có điều kiện | - Chỉ hiện khi đơn gốc và đơn đích thuộc bàn khác nhau (BR-18).<br/>- Nội dung: "Đơn gốc (Bàn {X}) và đơn đích (Bàn {Y}) thuộc 2 bàn khác nhau. Tất cả sản phẩm sẽ chuyển về Bàn {Y}." |
| Nút **"Xác nhận Ghép đơn"** | Button (Primary, Danger) | — | Enabled | - Dùng style Danger vì thao tác không thể undo.<br/>- Nhấn → hiển thị Dialog xác nhận lần cuối. |
| Nút **"Quay lại chọn đơn"** | Button (Secondary) | — | Enabled | - Quay lại DS đơn có thể ghép, giữ nguyên lựa chọn trước đó. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Preview ghép | Nhấn **"Xác nhận Ghép đơn"** | Dialog xác nhận cuối | Hỏi lần cuối trước khi commit |
| Preview ghép | Nhấn **"Quay lại chọn đơn"** | DS đơn có thể ghép | Giữ nguyên lựa chọn |
| Preview ghép | Dialog xác nhận → **"Đồng ý"** | Loading → Kết quả | Thực thi ghép đơn |
| Preview ghép | Dialog xác nhận → **"Hủy"** | Preview ghép | Đóng dialog, giữ nguyên |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Layout 2 panel:** Desktop/Tablet hiển thị đơn gốc và đơn đích side-by-side. Mobile: dọc, đơn đích trên + đơn gốc (collapsed accordion) dưới.
- **Highlight SP mới:** Sản phẩm chuyển từ đơn gốc highlight nền nhạt + badge "MỚI GHÉP".
- **Loading preview:** Spinner trên các ô tổng tiền khi chờ API tính toán.
- **Định dạng tiền:** `{N}.000 đ` (dấu chấm phân cách nghìn).

---

## Screen: merge-confirm-dialog (Dialog xác nhận ghép đơn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề | Label | — | "Xác nhận ghép đơn" | - Tiêu đề cố định, không chỉnh sửa. |
| Nội dung xác nhận | Label (multiline) | — | Dynamic | - Nội dung: "Ghép toàn bộ {N} sản phẩm từ đơn [MÃ_GỐC] vào đơn [MÃ_ĐÍCH]. Đơn gốc sẽ được đánh dấu Đã ghép và không thể sử dụng lại."<br/>- Nếu có cảnh báo khác bàn (BR-18): thêm dòng cảnh báo bàn.<br/>- Nếu có giảm giá bị hủy (BR-10): thêm dòng cảnh báo giảm giá. |
| Nút **"Đồng ý"** | Button (Primary, Danger) | — | Enabled | - Thực thi API POST /orders/{source}/merge/confirm.<br/>- Sau khi nhấn: disable nút ngay lập tức + hiển thị spinner (ngăn double-click).<br/>*Lỗi:* "Ghép đơn thất bại. Vui lòng thử lại." |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Đóng dialog, quay lại màn Preview. Không thay đổi gì. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog xác nhận | Nhấn **"Đồng ý"** | Loading → Kết quả | Thực thi ghép |
| Dialog xác nhận | Nhấn **"Hủy"** | Preview ghép | Đóng dialog |
| Dialog xác nhận | Nhấn ngoài dialog (backdrop) | Preview ghép | Giống "Hủy" |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Ngăn double-click:** Sau khi nhấn "Đồng ý", disable nút ngay + spinner. Không cho phép nhấn lần 2.
- **Backdrop click:** Nhấn ngoài dialog = đóng dialog (giống "Hủy").

---

## Screen: merge-result (Kết quả ghép đơn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Banner thành công | Alert (Success) | — | Hiển thị khi OK | - Nội dung: "Ghép đơn thành công! Đơn [MÃ_GỐC] đã được ghép vào đơn [MÃ_ĐÍCH]." |
| Card Đơn đích (sau ghép) | Order Card | — | Data từ API | - Hiển thị: Mã đơn đích, Bàn, DS sản phẩm đầy đủ, Tổng tiền cập nhật.<br/>- Badge trạng thái: `ACTIVE`.<br/>- Nút hành động: "Thanh toán", "Xem chi tiết". |
| Info Đơn gốc | Label (readonly) | — | Data từ API | - Hiển thị: Mã đơn gốc + Badge `ĐÃ GHÉP`.<br/>- Ghi chú: "Đơn gốc đã được đánh dấu Đã ghép." |
| Nút **"Xem chi tiết đơn đích"** | Button (Primary) | — | Enabled | - Mở chi tiết đơn đích (full screen).<br/>- Auto-redirect sau 3 giây nếu không tương tác (theo yêu cầu luồng: mở chi tiết đơn đích sau ghép). |
| Nút **"Về danh sách"** | Button (Secondary) | — | Enabled | - Quay về DS đơn hàng / màn hình chính POS. |
| Banner lỗi | Alert (Error) | — | Hiển thị khi thất bại | - Nội dung: "Ghép đơn thất bại. Cả 2 đơn hàng không thay đổi. Vui lòng thử lại hoặc liên hệ hỗ trợ."<br/>- Nút kèm theo: "Thử lại", "Đóng". |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Kết quả ghép (OK) | Nhấn **"Xem chi tiết đơn đích"** | Chi tiết đơn đích | Full screen, auto-redirect 3s |
| Kết quả ghép (OK) | Nhấn **"Thanh toán"** (trên card) | Màn thanh toán đơn đích | Thanh toán độc lập |
| Kết quả ghép (OK) | Nhấn **"Về danh sách"** | DS đơn hàng / POS chính | — |
| Kết quả ghép (lỗi) | Nhấn **"Thử lại"** | Preview ghép | Giữ nguyên lựa chọn cũ |
| Kết quả ghép (lỗi) | Nhấn **"Đóng"** | Chi tiết đơn gốc | Đơn gốc không thay đổi |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Auto-redirect:** Sau 3 giây thành công mà không tương tác → tự mở chi tiết đơn đích (theo luồng user: mở chi tiết đơn đích sau ghép).
- **Offline:** Nếu mất kết nối: cảnh báo "Mất kết nối. Vui lòng kiểm tra lại trạng thái đơn hàng." — không hiển thị kết quả chưa xác định.
