---
type: srs-screens
feature: tpaygate-connect-bank
updated: 2026-07-30
---

## Screen: bank-link-listing (Danh sách liên kết ngân hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Banner cảnh báo chưa cấu hình | Alert Banner (Warning) | — | Ẩn | - Hiển thị khi thiếu ≥1 thông số T-PayGate (`clientId`, `tenantId`, `source`, `clientSecret`) — BR-01.<br/>- Nội dung: _"Chưa cấu hình thông số T-PayGate. Liên hệ quản trị viên để hoàn tất cài đặt."_<br/>- Ẩn toàn bộ danh sách và nút thêm khi banner này hiển thị. |
| Danh sách liên kết ngân hàng | Table / Card List (readonly) | — | Data từ `GET /config-bank/list` | - Mỗi item hiển thị: Logo ngân hàng (`UrlLogo`), Tên ngân hàng (join `BankCode` với `/bank` — BR-02), Tên chủ TK (`AccountName`), Số tài khoản (`AccountNo`), Số VA (`VaNumber`), Trạng thái (badge `IsConnected`), Ngày tạo (`DateCreated`).<br/>- Sort mặc định: `DateCreated` mới nhất.<br/>- `VaNumber` nullable: hiển thị `—` nếu null.<br/>- `UrlLogo` nullable: hiển thị icon ngân hàng mặc định nếu null. |
| Badge Trạng thái | Badge (readonly) | — | Tự động | - `IsConnected = true`: badge xanh lá — "Đã kết nối".<br/>- `IsConnected = false`: badge xám — "Đã ngắt kết nối". |
| Nút **"Thêm liên kết ngân hàng"** | Button (Primary) | — | Enabled | - Ẩn / disable khi chưa cấu hình thông số T-PayGate (BR-01).<br/>- Nhấn → mở màn chọn ngân hàng và luồng kết nối. |
| Menu hành động (⋮) mỗi dòng | Dropdown Menu | — | — | - **"Xem chi tiết"**: luôn hiển thị.<br/>- **"Ngắt kết nối"**: chỉ hiển thị khi `IsConnected = true` (BR-14).<br/>- **"Xóa khỏi danh sách"** (nếu `IsConnected = false`): xóa record khỏi UI (giữ trong DB để đối soát).<br/>*Lỗi ngắt kết nối:* _"Không thể ngắt kết nối. Vui lòng thử lại."_ |
| Panel rỗng (Empty State) | Empty State | — | Hiện khi DS rỗng | - Nội dung: _"Chưa có liên kết ngân hàng nào. Nhấn 'Thêm liên kết ngân hàng' để bắt đầu."_ |
| Chỉ thị tải (Loading) | Skeleton Cards | — | Hiện khi đang tải | - Hiển thị 3 skeleton card khi đang gọi `GET /config-bank/list`. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Danh sách liên kết | Nhấn **"Thêm liên kết ngân hàng"** | Màn chọn ngân hàng | Mở màn hình hoặc modal chọn ngân hàng |
| Danh sách liên kết | Nhấn **"Ngắt kết nối"** | `[[docs/tpaygate-disconnect-bank/srs/screens|Disconnect Bank Dialog]]` | Kích hoạt luồng Ngắt kết nối (xem đặc tả riêng) |
| Danh sách liên kết | Nhấn **"Xem chi tiết"** | Chi tiết liên kết ngân hàng | Readonly — hiển thị đầy đủ thông tin |
| Danh sách liên kết | Nhận `postMessage("tabClosed")` | Làm mới danh sách (tại chỗ) | Gọi lại `GET /config-bank/list`, không chuyển trang |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Tên ngân hàng không có trong `/config-bank/list`:** Phải join `BankCode` với danh sách từ `GET /bank` để lấy `Name`. Nên cache danh sách ngân hàng (không gọi lại mỗi khi render). `UrlLogo` dùng field từ `/bank` (key `Logo`) khi kết nối mới, dùng `UrlLogo` từ `/config-bank/list` cho record đã lưu — hai tên field khác nhau cho cùng dữ liệu.
- **Không có pagination từ T-PayGate:** API `/config-bank/list` trả toàn bộ danh sách. Nếu nhiều merchant → phân trang phía đối tác (client-side pagination hoặc virtual scroll).
- **Trạng thái "Đã ngắt kết nối":** Sau khi disconnect, giữ lại record trong danh sách với badge xám để đối soát, không xóa cứng khỏi UI. Cho phép xóa thủ công khỏi UI (không xóa DB).

---

## Screen: bank-selection-modal (Modal chọn ngân hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề modal | Label | — | "Chọn ngân hàng" | - Cố định. |
| Ô tìm kiếm ngân hàng | Textbox (Search) | Không | Trống | - Tìm theo tên hoặc tên viết tắt (`ShortName`).<br/>- Debounce 300ms.<br/>- Placeholder: _"Tìm ngân hàng..."_ |
| Danh sách ngân hàng | Grid / List (selectable) | Có | Load từ `GET /bank` | - Mỗi item: Logo ngân hàng (`Logo`), Tên ngân hàng (`Name`), Tên viết tắt (`ShortName`).<br/>- Chỉ hiển thị ngân hàng `IsActive = true` — BR-02.<br/>- Gọi API ẩn danh (không kèm Authorization) — BR-03.<br/>- Click vào item → chọn ngân hàng đó.<br/>*Lỗi tải:* _"Không thể tải danh sách ngân hàng. Vui lòng thử lại."_ (kèm nút Thử lại). |
| Nút **"Tiếp tục"** | Button (Primary) | — | Disabled | - Enabled khi đã chọn 1 ngân hàng.<br/>- Nhấn → tiếp tục sang luồng kết nối (Embedded UI hoặc form API trực tiếp). |
| Nút **"Đóng"** | Button (Secondary) | — | Enabled | - Đóng modal. Không thay đổi gì. |
| Chỉ thị tải | Skeleton Grid | — | Hiện khi đang tải | - Hiển thị 6 skeleton item khi đang gọi `GET /bank`. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Modal chọn ngân hàng | Nhấn **"Tiếp tục"** (Embedded UI) | Popup T-PayGate Embedded UI | Mở URL `/view/connect?bankCode={code}&token={AccessToken}` trong popup/tab mới |
| Modal chọn ngân hàng | Nhấn **"Tiếp tục"** (API trực tiếp) | Form nhập thông tin tài khoản | Truyền `bankCode` đã chọn vào form |
| Modal chọn ngân hàng | Nhấn **"Đóng"** | Danh sách liên kết | Đóng modal |
| Modal chọn ngân hàng | Nhấn ngoài modal | Danh sách liên kết | Giống "Đóng" |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Không gọi `/bank` kèm Bearer mà không ký:** Nếu có token trong bộ nhớ → gọi ẩn danh (không đính kèm Authorization header) để tránh 403 — BR-03.
- **Fallback khi không có logo:** Hiển thị icon ngân hàng generic với tên viết tắt (`ShortName`).

---

## Screen: bank-connect-form (Form nhập thông tin kết nối ngân hàng — Luồng API trực tiếp)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Ngân hàng đã chọn | Label (readonly) | — | Từ bước chọn ngân hàng | - Hiển thị logo + tên ngân hàng đã chọn. Không chỉnh sửa tại form này. Có nút "Đổi ngân hàng" quay lại modal chọn. |
| Tên cửa hàng / Merchant Name | Textbox | Có | Trống | - Tên cửa hàng hoặc doanh nghiệp (`merchantName`) — BR-06.<br/>- Tối đa 100 ký tự.<br/>*Lỗi:* _"Tên cửa hàng không được để trống."_<br/>*Lỗi:* _"Tên cửa hàng tối đa 100 ký tự."_ |
| Tên chủ tài khoản | Textbox | Có | Trống | - Tên chủ tài khoản ngân hàng (`accountName`) — BR-06.<br/>- Tối đa 100 ký tự. Nên nhập in hoa không dấu theo định dạng ngân hàng.<br/>*Lỗi:* _"Tên chủ tài khoản không được để trống."_ |
| Số tài khoản | Textbox | Có | Trống | - Số tài khoản nhận tiền (`accountNo`) — BR-06.<br/>- Validate `^[0-9]{6,20}$` (phụ thuộc ngân hàng).<br/>*Lỗi:* _"Số tài khoản không được để trống."_<br/>*Lỗi:* _"Số tài khoản không hợp lệ."_ |
| CMND/CCCD | Textbox | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn theo từng ngân hàng (`identity`) — BR-06.<br/>- Validate `^[0-9]{9,12}$`.<br/>*Lỗi:* _"CMND/CCCD không hợp lệ."_ |
| Số điện thoại | Textbox | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn (`phone`) — BR-06.<br/>- Validate `^(0[3-9][0-9]{8})$` (SĐT Việt Nam).<br/>*Lỗi:* _"Số điện thoại không hợp lệ."_ |
| Email | Textbox | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn (`email`) — BR-06.<br/>*Lỗi:* _"Email không đúng định dạng."_ |
| Mã merchant (Prefix) | Textbox | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn, mã merchant do ngân hàng cấp (`prefix`). |
| Mã định danh (Client ID ngân hàng) | Textbox | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn, mã do ngân hàng cấp cho merchant (`clientId` trong body connect) — BR-07, BR-18.<br/>- **Label phải ghi rõ:** _"Mã do ngân hàng cấp (không phải Client ID T-PayGate)"_.<br/>- Tooltip giải thích: _"Đây là mã định danh merchant do ngân hàng cấp, khác với Client ID T-PayGate dùng cho xác thực."_ |
| Encrypt Key | Textbox (Password masked) | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn (`encryptKey`). Input kiểu password để ẩn giá trị. |
| Secret Key | Textbox (Password masked) | Không (phụ thuộc ngân hàng) | Trống | - Tùy chọn (`secretKey`). Input kiểu password để ẩn giá trị. |
| Nút **"Kết nối ngân hàng"** | Button (Primary) | — | Enabled | - Validate toàn bộ form phía client trước khi submit.<br/>- Disable + spinner sau click (chống double-submit — BR-05).<br/>- Gọi `POST /config-bank/connect` với đầy đủ HMAC-SHA256 (BR-04).<br/>*Lỗi API:* hiển thị `Error.Message` trong toast hoặc alert trên form. |
| Nút **"Hủy"** | Button (Secondary) | — | Enabled | - Quay lại màn hình danh sách liên kết ngân hàng. Hiển thị dialog xác nhận nếu form có thay đổi. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Form kết nối | Nhấn **"Kết nối ngân hàng"** (IsOTPConfirmation=true) | Màn nhập OTP | Truyền `ConfigBankId` sang màn OTP |
| Form kết nối | Nhấn **"Kết nối ngân hàng"** (IsConnected=true ngay) | Danh sách liên kết + Toast | Toast: _"Kết nối ngân hàng thành công."_ Gọi `/config-bank/list` để lấy VaNumber (BR-09) |
| Form kết nối | Nhấn **"Đổi ngân hàng"** | Modal chọn ngân hàng | Quay lại bước chọn |
| Form kết nối | Nhấn **"Hủy"** | Danh sách liên kết | Dialog xác nhận nếu form dirty |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Trường tùy chọn theo ngân hàng (OQ-05):** Hiện tại hiển thị tất cả trường tùy chọn, dùng collapsible section "Thông tin nâng cao" để không làm form quá dài. Chờ xác nhận T-PayGate về API trả cấu hình field theo ngân hàng.
- **Disable sau khi submit:** Sau khi nhấn "Kết nối ngân hàng", disable form và hiển thị spinner. Không re-enable trừ khi nhận được lỗi từ API (BR-05).
- **Timeout xử lý:** Nếu `/connect` timeout (đề xuất 30s), không tự động retry. Hướng dẫn người dùng kiểm tra lại danh sách trước khi thử lại (BR-05).

---

## Screen: bank-otp-confirmation (Màn nhập OTP xác nhận kết nối)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề | Label | — | "Xác nhận kết nối ngân hàng" | - Cố định. |
| Mô tả hướng dẫn | Label (multiline) | — | Dynamic | - Nội dung: _"Mã OTP đã được gửi đến số điện thoại đăng ký dịch vụ của bạn. Vui lòng nhập mã OTP để hoàn tất kết nối."_ |
| Ô nhập OTP | OTP Input (6 ô riêng biệt hoặc 1 textbox) | Có | Trống | - Format: chỉ nhận ký tự số (`^[0-9]{4,8}$`) — BR-15.<br/>- Validate phía client trước khi gọi API.<br/>*Lỗi:* _"OTP không hợp lệ. Vui lòng kiểm tra lại."_<br/>*Lỗi API sai OTP:* _"OTP không chính xác. Vui lòng nhập lại."_<br/>*Lỗi API hết hạn OTP:* _"OTP đã hết hạn. Vui lòng yêu cầu gửi lại."_ |
| Đếm ngược thời gian OTP | Label (readonly) | — | — | - Hiển thị thời gian còn lại (nếu biết TTL từ ngân hàng — OQ-05). Khi hết → hiển thị nút "Gửi lại OTP". |
| Nút **"Gửi lại OTP"** | Link / Button (Text) | — | Ẩn | - Hiển thị sau khi OTP hết hạn hoặc sau N giây.<br/>- Nhấn → gọi lại `POST /config-bank/connect` với cùng thông tin để nhận OTP mới. Cần sinh `ConfigBankId` mới và cập nhật context màn hình. |
| Nút **"Xác nhận"** | Button (Primary) | — | Enabled | - Validate OTP phía client trước khi gọi API (BR-15).<br/>- Disable + spinner sau click.<br/>- Gọi `POST /config-bank/confirm` (ký HMAC-SHA256 — BR-04).<br/>*Lỗi chung:* _"Xác nhận thất bại. Vui lòng thử lại."_ |
| Nút **"Quay lại"** | Button (Secondary) | — | Enabled | - Quay lại form nhập thông tin tài khoản. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn OTP | Nhấn **"Xác nhận"** (thành công) | Danh sách liên kết + Toast | Toast: _"Kết nối ngân hàng thành công."_ Lưu `ConfigBankId` + `VaNumber` vào DB (BR-08) |
| Màn OTP | Nhấn **"Xác nhận"** (lỗi OTP) | Giữ màn OTP + thông báo lỗi inline | Cho phép nhập lại |
| Màn OTP | Nhấn **"Gửi lại OTP"** | Giữ màn OTP (reset countdown) | Gọi lại `/connect`, nhận `ConfigBankId` mới |
| Màn OTP | Nhấn **"Quay lại"** | Form nhập thông tin tài khoản | Pre-fill lại thông tin đã nhập |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **OTP input UX:** Dùng 6 ô input riêng biệt (mỗi ô 1 chữ số) — tự động focus sang ô tiếp theo khi nhập. Hỗ trợ paste cả chuỗi OTP vào. Trên mobile, mở keyboard số.
- **Disable khi submit:** Disable toàn bộ form + spinner trên nút "Xác nhận" sau khi nhấn. Re-enable khi nhận được lỗi từ API.
- **"Gửi lại OTP" tạo `ConfigBankId` mới:** Context màn hình OTP phải được cập nhật với `ConfigBankId` mới từ response `/connect` lần gọi lại. Không dùng `ConfigBankId` cũ khi gọi `/confirm` sau khi "Gửi lại OTP".

---

## Screen: disconnect-bank-dialog (Dialog xác nhận ngắt kết nối ngân hàng)

> **Lưu ý modular BA:** Đặc tả màn hình Dialog xác nhận ngắt kết nối và trạng thái hiển thị của liên kết đã bị hủy trong danh sách đã được tách riêng thành tính năng độc lập `tpaygate-disconnect-bank`.
> 
> Xem đầy đủ đặc tả chi tiết tại: `[[docs/tpaygate-disconnect-bank/srs/screens|T-PayGate Disconnect Bank Screens]]`.
