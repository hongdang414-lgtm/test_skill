---
type: srs-screens
feature: qr-payment
updated: 2026-08-03
---

## Screen: payment-method-selection (Màn hình chọn phương thức thanh toán)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tổng tiền đơn hàng | Label (highlight) | — | Dynamic | - Hiển thị `totalAmount` đã tính thuế, phí dịch vụ, giảm giá. Định dạng tiền tệ VND (VD: `250.000 đ`). |
| Nút **"Chuyển khoản QR"** | Button (Primary) | — | Enabled / Disabled | - Enabled khi có `configId` hoặc `ConfigBankId IsConnected=true` — BR-01.<br/>- Disabled khi không có cấu hình ngân hàng hợp lệ, kèm tooltip: _"Chưa cấu hình tài khoản ngân hàng. Liên hệ Quản lý."_ — BR-01.<br/>- Khi nhấn: gọi Backend để tạo bill QR. Disable nút ngay sau khi nhấn, hiển thị spinner để chống double-submit. |
| Nút **"Tiền mặt"** | Button (Secondary) | — | Enabled | - Chuyển sang màn hình thanh toán tiền mặt. |
| Nút **"Thẻ / Quẹt thẻ"** | Button (Secondary) | — | Enabled / Disabled | - Enabled tùy theo cấu hình thiết bị đầu cuối thẻ. |
| Thông báo lỗi cấu hình | Alert Box (Error) | — | Ẩn | - Hiển thị khi gọi API tạo bill thất bại với lỗi cấu hình (BR-01, BR-04).<br/>- Nội dung: _"Không thể tạo mã QR. Vui lòng kiểm tra cấu hình ngân hàng hoặc liên hệ Quản lý."_ |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Chọn phương thức | Nhấn **"Chuyển khoản QR"** (tạo bill thành công) | Màn hình QR (qr-payment-display) | Chuyển sau khi nhận `BillCode` + QR từ T-PayGate. |
| Chọn phương thức | Nhấn **"Chuyển khoản QR"** (lỗi cấu hình/chữ ký) | Giữ nguyên + hiển thị Alert lỗi | Thu ngân có thể thử lại hoặc chọn phương thức khác. |
| Chọn phương thức | Nhấn **"Tiền mặt"** | Màn hình thanh toán tiền mặt | — |
| Chọn phương thức | Nhấn **"Quay lại"** | Màn hình chi tiết đơn hàng | Không thay đổi trạng thái đơn. |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Ngăn chặn tạo bill kép (Anti Double-Submit):** Nút "Chuyển khoản QR" bị disable ngay sau khi nhấn và hiển thị spinner. Chỉ mở lại nút khi API trả lỗi xác định (403 cấu hình, 403 chữ ký).
- **Kiểm tra cấu hình ngân hàng sớm:** Thực hiện kiểm tra `configId`/`ConfigBankId` tại thời điểm load màn hình (không phải khi nhấn nút) để disable nút và hiển thị tooltip ngay, tránh trường hợp thu ngân nhấn rồi mới thấy lỗi.
- **Phân biệt lỗi cấu hình và lỗi kết nối:** Lỗi 403 chữ ký/hết hạn → thông báo "Lỗi kết nối hệ thống thanh toán, thử lại sau". Lỗi 403 `configBankId does not exist or is disconnected` → thông báo "Tài khoản ngân hàng đã ngắt kết nối, liên hệ Quản lý".

---

## Screen: qr-payment-display (Màn hình hiển thị mã QR thanh toán)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề màn hình | Label | — | "Quét mã QR để thanh toán" | - Cố định. |
| Mã QR thanh toán | Image / QR Canvas | ✅ | Dynamic | - Ưu tiên render từ `QrDataURL` nếu có giá trị (xử lý cả có/không có prefix `data:`). Fallback render từ chuỗi EMV `QRBase64` bằng thư viện phía client nếu `QrDataURL` rỗng — BR-05.<br/>- Nếu cả hai rỗng: không hiển thị khung QR trống, hiển thị thông báo lỗi và nút "Tạo lại" — BR-05.<br/>- Kích thước: tối thiểu 200×200 px. |
| Số tiền cần thanh toán | Label (Large, highlight) | — | Dynamic | - Hiển thị `totalAmount` định dạng VND (VD: `250.000 đ`). Phông chữ lớn, màu nổi bật. |
| Tên ngân hàng | Label | — | Dynamic | - Lấy từ `InfoPayment.Bank` trong response tạo bill — BR-06. |
| Số tài khoản VA | Label (Monospace, copyable) | — | Dynamic | - Lấy từ `InfoPayment.AccountNumber` — BR-06.<br/>- Hiển thị nút "Sao chép" cạnh số tài khoản để khách có thể chuyển khoản thủ công. |
| Tên chủ tài khoản | Label | — | Dynamic | - Lấy từ `InfoPayment.AccountName` — BR-06. |
| Nội dung chuyển khoản | Label (Monospace, highlight, copyable) | — | Dynamic | - Lấy từ `InfoPayment.Description` = `BillCode` — BR-06.<br/>- Hiển thị nút "Sao chép". Nhấn mạnh tầm quan trọng: _"Vui lòng nhập đúng nội dung chuyển khoản để hệ thống tự động xác nhận."_ |
| Đếm ngược thời gian | Countdown Timer | — | `MM:SS` | - Đếm ngược từ thời gian timeout do POS cấu hình (khuyến nghị: 5 phút = 300 giây) kể từ thời điểm tạo bill — BR-07.<br/>- Khi còn ≤ 60 giây: chuyển màu đỏ/cảnh báo.<br/>- Khi hết giờ: ẩn QR và hiển thị trạng thái QR hết hạn. |
| Trạng thái thanh toán | Badge / Status Indicator | — | "Đang chờ thanh toán..." | - Cập nhật theo thời gian thực qua WebSocket/SSE (BR-13).<br/>- Các trạng thái: `Đang chờ...` (xám), `Thanh toán một phần` (vàng), `Đã thanh toán` (xanh lá). |
| Nút **"Hủy / Chọn phương thức khác"** | Button (Secondary, Danger) | — | Enabled | - Cho phép thu ngân hủy QR và quay lại màn hình chọn phương thức — BR-15.<br/>- Hiển thị Dialog xác nhận trước khi hủy để tránh nhấn nhầm. |
| Nút **"Xác nhận thanh toán thủ công"** | Button (Secondary) | — | Ẩn / Hiển thị khi cần | - Chỉ hiển thị theo quyền của thu ngân hoặc khi hệ thống webhook không về trong thời gian nhất định.<br/>- Cho phép thu ngân xác nhận thanh toán thủ công sau khi đã kiểm tra thực tế. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn hình QR | Nhận webhook thành công (Amount ≥ total) | Màn hình thanh toán thành công (qr-payment-success) | Tự động chuyển qua WebSocket/SSE — BR-13. |
| Màn hình QR | Nhận webhook (Amount < total) | Giữ nguyên, cập nhật badge "Thanh toán một phần" | Hiển thị số tiền đã nhận và số tiền còn thiếu — BR-10. |
| Màn hình QR | Timer hết hạn | Màn hình QR hết hạn (qr-payment-expired) | Ẩn QR, hiện nút tạo lại — BR-07. |
| Màn hình QR | Nhấn **"Hủy"** (xác nhận) | Màn hình chọn phương thức thanh toán | BillCode đánh dấu hủy nội bộ — BR-15. |
| Màn hình QR | Nhấn **"Xác nhận thủ công"** | Màn hình thanh toán thành công | Thu ngân phải nhập ghi chú xác nhận thủ công. |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Realtime + Polling dự phòng:** Sử dụng WebSocket/SSE để nhận cập nhật trạng thái tức thì. Đồng thời polling `GET /order/get-billCode?billCode={billCode}` mỗi ~5 giây làm dự phòng khi WebSocket bị mất kết nối — BR-13.
- **Kích thước QR đủ lớn cho màn hình cảm ứng:** QR phải đủ to để thiết bị của khách quét được từ khoảng cách 30–50 cm. Tránh co ép QR vào góc nhỏ của màn hình.
- **Thông tin chuyển khoản thủ công luôn hiển thị rõ:** Ngay cả khi QR hiển thị đầy đủ, vẫn luôn hiển thị số VA, tên tài khoản và nội dung chuyển khoản để khách có thể nhập thủ công nếu app ngân hàng không quét được — BR-06.
- **Tự động focus màn hình:** Khi màn hình QR đang hiển thị, không cho phép thu ngân thao tác trên các vùng khác của POS để tránh chuyển màn hình vô tình trong khi khách đang quét.

---

## Screen: qr-payment-success (Màn hình thanh toán QR thành công)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Icon thành công | Icon / Animation | — | ✅ (checkmark xanh) | - Hiển thị animation checkmark nổi bật khi màn hình load. |
| Thông báo trạng thái | Label (Large) | — | "Thanh toán thành công!" | - Cố định. Phông chữ lớn, màu xanh lá. |
| Số tiền đã thanh toán | Label | — | Dynamic | - Hiển thị `amountPaid` từ webhook, định dạng VND. |
| Thời gian thanh toán | Label | — | Dynamic | - Hiển thị `PaymentTime` từ webhook, định dạng `HH:mm DD/MM/YYYY`. |
| Mã BillCode | Label (Monospace) | — | Dynamic | - Hiển thị `BillCode` để đối soát nếu cần. |
| Ngân hàng thanh toán | Label | — | Dynamic | - Hiển thị tên ngân hàng từ `InfoPayment.Bank`. |
| Nút **"In hóa đơn"** | Button (Primary) | — | Enabled | - Kích hoạt luồng in hóa đơn (receipt). |
| Nút **"Đơn hàng mới"** | Button (Secondary) | — | Enabled | - Đóng đơn hàng hiện tại, quay về màn hình danh sách đơn hoặc tạo đơn mới. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Thanh toán thành công | Nhấn **"In hóa đơn"** | Giữ nguyên + gửi lệnh in | In receipt và giữ màn hình. |
| Thanh toán thành công | Nhấn **"Đơn hàng mới"** | Màn hình tạo đơn hàng mới / Danh sách bàn | Đóng đơn hàng hiện tại. |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Transition animation:** Màn hình QR chuyển sang màn hình thành công cần có animation mượt mà (fade-in hoặc slide) để thu ngân và khách hàng nhận biết rõ ràng giao dịch đã hoàn tất.
- **Âm thanh xác nhận (tuỳ chọn):** Phát âm thanh thông báo thanh toán thành công nếu thiết bị POS hỗ trợ và cài đặt cho phép.

---

## Screen: qr-payment-expired (Màn hình QR hết thời gian)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Thông báo hết hạn | Label | — | "Mã QR đã hết thời gian" | - Cố định. Hiển thị khi timer về 0 — BR-07. |
| Hướng dẫn | Label | — | "Vui lòng tạo mã QR mới để tiếp tục thanh toán" | - Cố định. |
| Nút **"Tạo lại mã QR"** | Button (Primary) | — | Enabled | - Gọi lại Backend để tạo bill mới với `refTransactionId` mới — BR-02, BR-07.<br/>- Disable nút ngay khi nhấn, hiển thị spinner. |
| Nút **"Hủy / Chọn phương thức khác"** | Button (Secondary) | — | Enabled | - Quay lại màn hình chọn phương thức — BR-15. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| QR hết hạn | Nhấn **"Tạo lại mã QR"** (thành công) | Màn hình QR (qr-payment-display) | Hiển thị QR mới với BillCode mới — BR-07. |
| QR hết hạn | Nhấn **"Tạo lại mã QR"** (lỗi) | Giữ nguyên + hiển thị lỗi | Như quy tắc lỗi tạo bill — BR-03, BR-04. |
| QR hết hạn | Nhấn **"Hủy"** | Màn hình chọn phương thức | Đơn hàng quay về chờ phương thức — BR-15. |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Trường hợp khách thanh toán QR cũ sau khi hết timeout hiển thị:** Webhook vẫn có thể về sau khi màn hình đã chuyển sang "QR hết hạn". Backend POS phải xử lý đúng theo `RefTransactionId` trong DB nội bộ — BR-07, BR-11. UI cần lắng nghe WebSocket ngay cả khi đang ở màn hình "QR hết hạn" để chuyển sang "Thanh toán thành công" nếu webhook về.

---

## Screen: qr-payment-cancel-dialog (Dialog xác nhận hủy thanh toán QR)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Tiêu đề Dialog | Label | — | "Xác nhận hủy QR?" | - Cố định. |
| Nội dung xác nhận | Label (multiline) | — | Dynamic | - _"Khách hàng chưa hoàn tất thanh toán. Nếu hủy, mã QR này sẽ không còn hiệu lực trên màn hình POS. Bạn có chắc muốn hủy và chọn phương thức khác không?"_ — BR-15. |
| Cảnh báo | Alert Box (Warning) | — | Hiển thị | - _"Nếu khách đã chuyển khoản theo mã QR này, thanh toán vẫn sẽ được ghi nhận tự động."_ — BR-15. |
| Nút **"Hủy thanh toán QR"** | Button (Primary, Danger) | — | Enabled | - Xác nhận hủy, đánh dấu `BillCode` là "Đã hủy nội bộ", quay về màn hình chọn phương thức. |
| Nút **"Tiếp tục chờ"** | Button (Secondary) | — | Enabled | - Đóng Dialog, quay lại màn hình QR đang hiển thị. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog hủy QR | Nhấn **"Hủy thanh toán QR"** | Màn hình chọn phương thức | Đơn hàng về trạng thái chờ — BR-15. |
| Dialog hủy QR | Nhấn **"Tiếp tục chờ"** | Màn hình QR (tiếp tục countdown) | Không thay đổi gì. |
| Dialog hủy QR | Nhấn vùng ngoài (backdrop) | Màn hình QR (tiếp tục countdown) | Tương đương "Tiếp tục chờ". |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Màu nút phân biệt rõ ràng:** Nút "Hủy thanh toán QR" dùng màu đỏ (Danger), nút "Tiếp tục chờ" dùng màu neutral, để thu ngân không nhấn nhầm.
- **Ưu tiên nút "Tiếp tục chờ":** Mặc định focus vào nút "Tiếp tục chờ" khi Dialog mở — giảm rủi ro hủy nhầm trên màn hình cảm ứng.
- **Countdown timer tiếp tục chạy trong Dialog:** Dialog không dừng timer. Nếu timer hết trong khi Dialog đang mở → chuyển sang màn hình QR hết hạn và đóng Dialog.
