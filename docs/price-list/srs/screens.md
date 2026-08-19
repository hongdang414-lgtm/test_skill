---
type: srs-screens
feature: price-list
updated: 2026-07-23
---

## Screen: price-list-listing (Danh sách bảng giá)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Ô tìm kiếm | Textbox (Search) | Không | Trống | - Tìm theo tên hoặc mã bảng giá.<br/>- Debounce 300ms. Placeholder: "Tìm theo tên hoặc mã bảng giá..." |
| Bộ lọc Trạng thái | Dropdown (Filter) | Không | "Tất cả" | - Tùy chọn: "Tất cả", "Chưa diễn ra", "Đang diễn ra", "Đã kết thúc".<br/>- Cập nhật bảng ngay khi thay đổi. |
| Bảng danh sách bảng giá | Table (readonly) | — | Data từ API | - Cột: Mã, Tên, Trạng thái (badge màu), Thời gian áp dụng, Nhóm KH, Số SP/Combo/Option, Ngày tạo.<br/>- Sort mặc định: Ngày tạo mới nhất.<br/>- Click vào row → mở Chi tiết bảng giá.<br/>- Phân trang 20 mục/trang. |
| Badge Trạng thái | Badge (readonly) | — | Tự động | - "Chưa diễn ra": badge xám.<br/>- "Đang diễn ra": badge xanh.<br/>- "Đã kết thúc": badge đen viền. |
| Nút **"Thêm bảng giá"** | Button (Primary) | — | Enabled | - Chỉ hiển thị khi có quyền `MANAGE_PRICE_LIST` (BR-01).<br/>- Nhấn → mở form tạo mới. |
| Menu hành động (⋮) | Dropdown Menu | — | — | - Mỗi row có menu: "Xem chi tiết", "Sửa" (ẩn nếu Đã kết thúc — BR-17), "Sao chép" (BR-14), "Xóa" (ẩn nếu Đang diễn ra — BR-06).<br/>*Lỗi:* "Không thể xóa bảng giá đang diễn ra." |
| Panel rỗng (Empty State) | Empty State | — | Hiện khi DS rỗng | - Nội dung: "Chưa có bảng giá nào. Nhấn Thêm bảng giá để bắt đầu." |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| DS bảng giá | Nhấn **"Thêm bảng giá"** | Form tạo bảng giá | Mở trang mới |
| DS bảng giá | Click row / "Xem chi tiết" | Chi tiết bảng giá | Readonly nếu Đã kết thúc |
| DS bảng giá | Nhấn "Sửa" | Form sửa bảng giá | Cùng form tạo, prefill data |
| DS bảng giá | Nhấn "Sao chép" | Form tạo bảng giá | Prefill data từ bản gốc, tên thêm " (Bản sao)" |
| DS bảng giá | Nhấn "Xóa" | Dialog xác nhận xóa | — |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Loading:** Skeleton table 5 rows khi tải lần đầu.
- **Badge trạng thái:** Trạng thái tự động cập nhật khi reload trang (BR-15).
- **Responsive:** Bảng cuộn ngang trên tablet, ẩn cột Ngày tạo trên mobile.

---

## Screen: price-list-form (Tạo / Chỉnh sửa bảng giá)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tên bảng giá | Textbox | Có | Trống | - Không được để trống (BR-02).<br/>*Lỗi:* "Tên bảng giá không được để trống."<br/>- Tối đa 100 ký tự Unicode.<br/>*Lỗi:* "Tên tối đa 100 ký tự."<br/>- Unique toàn hệ thống (check khi blur + khi submit).<br/>*Lỗi:* "Tên bảng giá đã tồn tại." |
| Mã bảng giá | Textbox + Toggle | Có | Auto-gen ON | - Toggle "Tự động sinh mã": ON → readonly, hiển thị preview `BG-{YYYYMMDD}-{SEQ}`.<br/>- OFF → cho phép nhập thủ công, format `^[A-Za-z0-9_-]{3,20}$` (BR-03).<br/>*Lỗi:* "Mã chỉ chấp nhận chữ cái, số, gạch ngang, gạch dưới (3–20 ký tự)."<br/>*Lỗi:* "Mã bảng giá đã tồn tại." |
| Mô tả | Textarea | Không | Trống | - Tối đa 500 ký tự (BR-18). Hiển thị bộ đếm `{N}/500`.<br/>*Lỗi:* "Mô tả tối đa 500 ký tự."<br/>- Placeholder: "Nhập mô tả ngắn về bảng giá..." |
| Thời gian bắt đầu | DateTimePicker | Có | Trống | - Bắt buộc chọn (BR-04).<br/>*Lỗi:* "Vui lòng chọn thời gian bắt đầu."<br/>- Phải ≥ thời điểm hiện tại.<br/>*Lỗi:* "Thời gian bắt đầu không được ở trong quá khứ."<br/>- Readonly khi bảng giá "Đang diễn ra" (BR-13). |
| Thời gian kết thúc | DateTimePicker | Có | Trống | - Bắt buộc chọn (BR-04).<br/>*Lỗi:* "Vui lòng chọn thời gian kết thúc."<br/>- Phải sau Thời gian bắt đầu.<br/>*Lỗi:* "Thời gian kết thúc phải sau thời gian bắt đầu."<br/>- Readonly khi "Đang diễn ra" (BR-13). |
| Bật khung giờ (Happy Hour) | Toggle Switch | Không | OFF | - ON → hiển thị thêm 3 trường: Giờ bắt đầu, Giờ kết thúc, Ngày áp dụng.<br/>- OFF → bảng giá áp dụng cả ngày trong khoảng thời gian.<br/>- Readonly khi "Đang diễn ra" (BR-13). |
| Giờ bắt đầu (Happy Hour) | TimePicker | Có (khi toggle ON) | Trống | - Chỉ hiện khi toggle Happy Hour = ON.<br/>- Format HH:mm (24h) (BR-19).<br/>*Lỗi:* "Vui lòng chọn giờ bắt đầu." |
| Giờ kết thúc (Happy Hour) | TimePicker | Có (khi toggle ON) | Trống | - Phải sau Giờ bắt đầu (BR-19).<br/>*Lỗi:* "Giờ kết thúc phải sau giờ bắt đầu." |
| Ngày áp dụng trong tuần | Checkbox Group | Có (khi toggle ON) | Tất cả checked | - 7 checkbox: T2, T3, T4, T5, T6, T7, CN (BR-19).<br/>- Ít nhất 1 ngày được chọn.<br/>*Lỗi:* "Vui lòng chọn ít nhất 1 ngày trong tuần." |
| Nhóm khách hàng | Multi-select Dropdown | Không | "Tất cả khách hàng" | - Chọn từ DS nhóm KH đã thiết lập.<br/>- Mặc định "Tất cả khách hàng" (áp dụng cho mọi khách).<br/>- Readonly khi "Đang diễn ra" (BR-13). |
| **Tab Sản phẩm** | Tab Panel | — | Active tab | - Hiển thị bảng SP đã thêm vào bảng giá: STT, Tên SP, SKU, Giá gốc, Phương thức, Giá trị điều chỉnh, Giá mới, Xóa.<br/>- Nút "Thêm sản phẩm" mở modal. |
| **Tab Combo** | Tab Panel | — | — | - Tương tự tab SP nhưng cho Combo: STT, Tên combo, Thành phần (collapsed), Giá gốc, Phương thức, Giá trị điều chỉnh, Giá mới, Xóa. |
| **Tab Tùy chọn** | Tab Panel | — | — | - Tương tự tab SP nhưng cho Option/Add-on: STT, Tên option, Nhóm option, Giá gốc, Phương thức, Giá trị điều chỉnh, Giá mới, Xóa. |
| Bảng SP/Combo/Option đã thêm | Table (editable) | — | Rỗng | - Mỗi dòng: Tên, Giá gốc (readonly), Dropdown phương thức (Nhập trực tiếp / Chiết khấu % / Giảm giá trị), Input giá trị điều chỉnh, Giá mới (tự tính — readonly), Nút xóa (x).<br/>- Giá mới cập nhật realtime khi thay đổi phương thức/giá trị (BR-11).<br/>- Giá mới < 0 → hiển thị 0 + cảnh báo (BR-08).<br/>*Lỗi:* "Giá không được nhỏ hơn 0."<br/>- Chiết khấu > 100% → lỗi inline.<br/>*Lỗi:* "Chiết khấu không được vượt quá 100%." |
| Nút **"Lưu"** | Button (Primary) | — | Enabled | - Validate toàn bộ form trước khi submit.<br/>- Check overlap thời gian (BR-05).<br/>- Check ≥ 1 mục (BR-09).<br/>- Disable + spinner sau click (chống double-click).<br/>*Lỗi:* "Vui lòng thêm ít nhất 1 sản phẩm, combo hoặc tùy chọn."<br/>*Lỗi:* "Thời gian trùng với bảng giá [{TÊN}]." |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Nếu form có thay đổi: hiển thị dialog "Bạn có chắc muốn hủy? Dữ liệu chưa lưu sẽ mất."<br/>- Quay về DS bảng giá. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Form tạo/sửa | Nhấn **"Thêm sản phẩm"** (tab SP) | Modal thêm SP | — |
| Form tạo/sửa | Nhấn **"Thêm combo"** (tab Combo) | Modal thêm Combo | — |
| Form tạo/sửa | Nhấn **"Thêm tùy chọn"** (tab Option) | Modal thêm Option | — |
| Form tạo/sửa | Nhấn **"Lưu"** (thành công) | DS bảng giá + Toast | Toast: "Tạo bảng giá thành công." |
| Form tạo/sửa | Nhấn **"Lưu"** (lỗi) | Giữ nguyên form + lỗi inline | — |
| Form tạo/sửa | Nhấn **"Hủy"** | DS bảng giá | Có dialog xác nhận nếu dirty |
| Form tạo/sửa | Nhấn **← Quay lại** | DS bảng giá | Giống Hủy |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Layout 2 cột (Desktop):** Cột trái: thông tin cơ bản (Tên, Mã, Mô tả, Thời gian, Nhóm KH). Cột phải: Tabs SP/Combo/Option.
- **Realtime preview giá:** Khi thay đổi phương thức hoặc giá trị điều chỉnh → cột "Giá mới" cập nhật ngay (không cần nhấn nút).
- **Dirty form detection:** Nếu user thay đổi bất kỳ trường nào và nhấn Hủy/Back → dialog xác nhận.
- **Readonly mode:** Khi bảng giá "Đang diễn ra" (BR-13): các trường cơ bản readonly, chỉ cho sửa bảng giá SP.

---

## Screen: add-items-modal (Modal thêm Sản phẩm / Combo / Tùy chọn)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề modal | Label | — | Dynamic | - Nội dung thay đổi theo tab: "Thêm sản phẩm" / "Thêm combo" / "Thêm tùy chọn". |
| Ô tìm kiếm | Textbox (Search) | Không | Trống | - Tìm theo tên hoặc mã SP/Combo/Option.<br/>- Debounce 300ms. Placeholder: "Tìm theo tên hoặc mã..." |
| Bộ lọc Danh mục | Dropdown | Không | "Tất cả" | - Lọc theo danh mục sản phẩm (cho tab SP), nhóm combo (cho tab Combo), nhóm option (cho tab Option). |
| Danh sách mục chọn | Checkbox List | Có (≥1) | Chưa chọn | - Mỗi item: Checkbox, Tên, SKU/Mã, Giá gốc.<br/>- Multi-select (checkbox).<br/>- Mục đã có trong bảng giá: hiển thị badge "Đã thêm" + disabled (BR-07).<br/>- Phân trang hoặc infinite scroll (20 mục/lần). |
| Thanh đếm đã chọn | Label (readonly) | — | "Đã chọn: 0" | - Cập nhật realtime: "Đã chọn: {N} sản phẩm/combo/tùy chọn". |
| Nút **"Thêm vào bảng giá"** | Button (Primary) | — | Disabled | - Enabled khi ≥ 1 mục được chọn.<br/>*Lỗi:* "Vui lòng chọn ít nhất 1 mục."<br/>- Nhấn → thêm các mục đã chọn vào bảng giá (giá mới mặc định = giá gốc, phương thức = Nhập trực tiếp). |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Đóng modal. Không thay đổi gì. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Modal thêm mục | Nhấn **"Thêm vào bảng giá"** | Form tạo/sửa + cập nhật bảng | Đóng modal, thêm các dòng mới vào bảng |
| Modal thêm mục | Nhấn **"Hủy"** | Form tạo/sửa | Đóng modal, không thay đổi |
| Modal thêm mục | Nhấn ngoài modal (backdrop) | Form tạo/sửa | Giống "Hủy" |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Loading:** Skeleton list 5 items khi tải lần đầu.
- **Badge "Đã thêm":** Mục đã có trong bảng giá hiển thị badge + checkbox disabled để tránh trùng (BR-07).
- **Infinite scroll:** Tải thêm 20 mục khi cuộn đến cuối danh sách.

---

## Screen: delete-confirm-dialog (Dialog xác nhận xóa bảng giá)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề | Label | — | "Xác nhận xóa bảng giá" | - Cố định, không chỉnh sửa. |
| Nội dung xác nhận | Label (multiline) | — | Dynamic | - Nội dung: "Bạn có chắc chắn muốn xóa bảng giá [{TÊN}]? Hành động này không thể hoàn tác."<br/>- Nếu bảng giá có sản phẩm: thêm dòng "Bảng giá này chứa {N} sản phẩm/combo/tùy chọn." |
| Nút **"Xóa"** | Button (Primary, Danger) | — | Enabled | - Xóa bảng giá khỏi hệ thống.<br/>- Disable + spinner sau click (chống double-click).<br/>*Lỗi:* "Xóa bảng giá thất bại. Vui lòng thử lại." |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Đóng dialog. Không thay đổi gì. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog xóa | Nhấn **"Xóa"** (thành công) | DS bảng giá + Toast | Toast: "Đã xóa bảng giá thành công." |
| Dialog xóa | Nhấn **"Xóa"** (lỗi) | Giữ dialog + thông báo lỗi | — |
| Dialog xóa | Nhấn **"Hủy"** | DS bảng giá | Đóng dialog |
| Dialog xóa | Nhấn ngoài dialog (backdrop) | DS bảng giá | Giống "Hủy" |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Ngăn double-click:** Sau khi nhấn "Xóa", disable nút ngay + spinner.
- **Backdrop click:** Nhấn ngoài dialog = đóng dialog (giống "Hủy").
