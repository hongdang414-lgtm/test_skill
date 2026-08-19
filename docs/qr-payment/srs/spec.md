---
feature: qr-payment
type: srs-spec
version: 1.0.0
updated: 2026-08-03
status: draft
authors: [BA Team]
changelog:
  - 2026-08-03 | /srs-spec   | [spec] initialized qr-payment specification, 15 business rules
  - 2026-08-03 | /activity   | [flows] added activity, sequence, and lifecycle diagrams for qr-payment
  - 2026-08-03 | /screen     | [screens] added specification for qr-payment screens
---

# Đặc tả tính năng: Thanh toán đơn hàng qua mã QR ngân hàng (QR Payment)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép thu ngân tạo mã QR VietQR gắn với đơn hàng để khách hàng quét và thanh toán qua ứng dụng ngân hàng. Hệ thống POS gọi **T-PayGate Public API** (`POST /api/v1/public-api/order/bill`) để sinh `BillCode` + chuỗi EMV QR (`QRBase64`), hiển thị QR trên màn hình thanh toán, sau đó lắng nghe webhook từ T-PayGate để xác nhận trạng thái thanh toán theo thời gian thực. Phạm vi: màn hình chọn phương thức thanh toán, màn hình hiển thị mã QR, và xử lý webhook xác nhận thanh toán trong hệ thống POS đối tác. |
| **2. Actors (Tác nhân)** | **Chính:** Thu ngân (Cashier) — thao tác tạo QR và xác nhận thanh toán trên POS.<br/>**Phụ trợ phía khách:** Khách hàng — dùng app ngân hàng quét QR và thực hiện chuyển khoản.<br/>**Hỗ trợ hệ thống:** T-PayGate API (tạo bill, sinh QR, trả webhook), Ngân hàng đối tác (nhận lệnh chuyển khoản, ghi có tài khoản VA, gửi xác nhận về T-PayGate), T-PayGate Admin (cấu hình `configId`/`ConfigBankId`). |
| **3. Pre-conditions** | 1. Hệ thống POS đã hoàn tất quy trình onboarding T-PayGate: có `clientId`, `tenantId`, `source`, `clientSecret` hợp lệ.<br/>2. Tồn tại ít nhất một `configId` (cấp sẵn) hoặc một `ConfigBankId` với `IsConnected = true` được cấu hình trong POS.<br/>3. Đơn hàng (Order) đang ở trạng thái chờ thanh toán, có `orderId` xác định và `totalAmount > 0`.<br/>4. Thu ngân đã chọn phương thức thanh toán **"Chuyển khoản QR"** tại màn hình Thanh toán.<br/>5. `Access Token` OAuth còn hiệu lực (hoặc hệ thống tự refresh từ cache 55 phút).<br/>6. Đồng hồ máy chủ POS đồng bộ NTP (lệch < 5 phút so với T-PayGate). |
| **4. Expected Results** | **Happy Path — Thanh toán thành công:**<br/>1. Thu ngân chọn "Chuyển khoản QR" → hệ thống gọi `POST /order/bill` với `refTransactionId` duy nhất, `amount` và `description`.<br/>2. T-PayGate trả về `BillCode` + `QRBase64` (chuỗi EMV) + `QrDataURL` + `InfoPayment`.<br/>3. Màn hình QR hiển thị mã QR, số tiền, tên tài khoản, số VA và nội dung chuyển khoản (`BillCode`).<br/>4. Khách dùng app ngân hàng quét QR và xác nhận thanh toán.<br/>5. T-PayGate nhận xác nhận từ ngân hàng, gọi webhook `POST` về URL đã đăng ký của POS với `RefTransactionId`, `BillCode`, `Amount`, `VirtualAccount`, `ActualAccount`, `PaymentTime`.<br/>6. POS verify chữ ký webhook, đối chiếu số tiền, cập nhật trạng thái đơn hàng sang **"Đã thanh toán"**, phát sự kiện realtime cập nhật UI.<br/>7. Thu ngân thấy màn hình tự động chuyển sang trạng thái **"Thanh toán thành công"**, có thể in hóa đơn.<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Tạo bill lỗi chữ ký/cấu hình]:** T-PayGate trả HTTP 403 `#TPayGate: Invalid signature.` hoặc `Request expired.` → Hiển thị lỗi, thu ngân thử lại hoặc chuyển phương thức thanh toán khác. Không retry tự động.<br/>- **[B — Tạo bill lỗi x-config-id không hợp lệ]:** T-PayGate trả HTTP 403 `configBankId does not exist or is disconnected.` → Thông báo rõ lỗi cấu hình, yêu cầu quản lý kiểm tra liên kết ngân hàng.<br/>- **[C — Tạo bill timeout/mạng lỗi]:** Không tự động retry. Hệ thống gọi `GET /order/get-refTransactionId` để kiểm tra trạng thái trước khi quyết định xử lý tiếp (tránh tạo bill trùng).<br/>- **[D — Khách thanh toán sai số tiền (thiếu)]:** Webhook về với `Amount` < `totalAmount` → State `Thanh toán một phần` → Hiển thị trạng thái chờ, yêu cầu khách bổ sung hoặc thu ngân xử lý thủ công.<br/>- **[E — Khách chưa quét QR, thu ngân hủy]:** Thu ngân nhấn "Hủy thanh toán QR" → Đơn hàng quay về trạng thái chờ chọn phương thức. BillCode vẫn tồn tại trên T-PayGate (không có API hủy bill); POS chỉ hủy nội bộ.<br/>- **[F — QR hết thời gian hiển thị]:** Sau thời gian timeout do POS tự quản lý (VD: 5 phút không có giao dịch), QR bị ẩn và hiển thị nút "Tạo lại QR". POS gọi lại API tạo bill mới với `refTransactionId` mới.<br/>- **[G — Webhook chữ ký sai hoặc replay]:** POS từ chối xử lý, trả HTTP 200 với `MessageError` có nội dung lỗi để T-PayGate retry. Không cập nhật trạng thái đơn hàng. |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **Phải có cấu hình ngân hàng hợp lệ trước khi tạo QR** | Trước khi gọi `POST /order/bill`, hệ thống POS kiểm tra tồn tại `configId` (cấp sẵn từ T-PayGate) hoặc `ConfigBankId` đang `IsConnected = true`. Ưu tiên `configId` nếu đã cấu hình; nếu không, lấy `ConfigBankId` đầu tiên có `IsConnected = true`. | Nếu không tìm thấy cấu hình hợp lệ → Disable phương thức "Chuyển khoản QR", hiển thị thông báo: _"Chưa cấu hình tài khoản ngân hàng nhận tiền. Vui lòng liên hệ Quản lý để thiết lập."_ |
| **BR-02** | **refTransactionId phải duy nhất trong phạm vi POS** | Mỗi lần tạo bill mới, hệ thống phải sinh `refTransactionId` duy nhất (VD: `{storeCode}-{orderId}-{timestamp-ms}` hoặc UUID v4). T-PayGate không dedupe theo `refTransactionId` — mỗi lần gọi đều sinh một `BillCode` mới. | Nếu phát hiện `refTransactionId` đã tồn tại trong DB nội bộ trước khi gọi API → Từ chối tạo bill, log cảnh báo trùng lặp và gọi `GET /order/get-refTransactionId` để kiểm tra trạng thái. |
| **BR-03** | **Tuyệt đối không retry POST /order/bill khi timeout** | Nếu request tạo bill gặp timeout hoặc lỗi mạng, hệ thống không được tự động gọi lại `POST /order/bill`. Mỗi lần gọi lại đều sinh một bill mới với QR riêng — khách có thể thanh toán nhiều lần cho cùng một đơn. | Khi timeout: gọi `GET /order/get-refTransactionId?refTransactionId={ref}` để kiểm tra xem bill đã được tạo chưa. Nếu đã tạo → dùng `BillCode` từ kết quả vấn tin. Nếu chưa → có thể tạo lại với `refTransactionId` mới. |
| **BR-04** | **Request tạo bill phải ký HMAC-SHA256 dạng POST** | `POST /api/v1/public-api/order/bill` yêu cầu header `x-config-id`, `x-api-time` (Unix seconds), `x-signature` (HMAC-SHA256 hex chữ thường). Công thức ký POST: `{clientId}_{tenantId}_{source}_{timestamp}_{rawBody}` (serialize body một lần, ký trên chuỗi đó, gửi chính chuỗi đó). | Thiếu hoặc sai chữ ký → T-PayGate trả HTTP 403 `#TPayGate: Invalid signature.`<br/>`x-api-time` lệch > 5 phút → HTTP 403 `#TPayGate: Request expired.`<br/>Hiển thị lỗi kỹ thuật, ghi log cảnh báo bảo mật. |
| **BR-05** | **Render QR theo thứ tự ưu tiên fallback** | Sau khi nhận response tạo bill thành công: (1) Nếu `QrDataURL` có giá trị → hiển thị trực tiếp (xử lý cả có/không có prefix `data:`). (2) Nếu `QrDataURL` rỗng → fallback render QR từ chuỗi EMV `QRBase64`. (3) Nếu cả hai rỗng → coi như thất bại, không hiển thị khung QR trống, thông báo lỗi. | Không hiển thị màn hình QR với khung trống. Nếu không có QR → báo lỗi và cho phép thử lại. |
| **BR-06** | **Nội dung chuyển khoản phải là BillCode** | Số VA (`InfoPayment.AccountNumber`) và nội dung chuyển khoản (`InfoPayment.Description` = `BillCode`) phải được hiển thị rõ ràng trên màn hình QR để khách có thể chuyển khoản thủ công nếu không quét QR được. Nội dung chuyển khoản phải khớp với `BillCode` — đây là khóa để T-PayGate định danh giao dịch và gọi webhook về đúng đơn hàng. | Nếu thiếu `InfoPayment` trong response → hệ thống chỉ hiển thị QR mà không hiển thị thông tin chuyển khoản thủ công; ghi log cảnh báo. |
| **BR-07** | **POS tự quản lý thời hạn hiển thị QR** | T-PayGate không trả về `ExpiredAt` trong response tạo bill. POS phải tự đặt timeout hiển thị QR (khuyến nghị: 5 phút kể từ thời điểm tạo bill). Khi hết timeout → QR bị ẩn, hiển thị nút "Tạo lại QR mới". Tạo lại phải sinh `refTransactionId` mới. | Nếu khách thanh toán sau khi QR đã hết timeout hiển thị trên POS: webhook vẫn có thể về bình thường (T-PayGate không tự hủy bill). POS phải xử lý webhook theo `RefTransactionId` và đối chiếu với DB nội bộ. |
| **BR-08** | **Verify chữ ký webhook trước khi xử lý** | Mọi webhook nhận từ T-PayGate phải được verify chữ ký HMAC-SHA256 trước khi xử lý nghiệp vụ. Công thức ký webhook (chiều về): `{tenantRoot.ClientId}_{tenant.TenantId}_{tenantRoot.Name}_{timestamp}_{rawBody}` — đoạn thứ 3 là `Name` của tenant gốc, **không phải** `source`. Đọc raw body trước khi parse JSON. | Chữ ký sai → trả HTTP 200 với `MessageError` có nội dung lỗi để T-PayGate retry. Ghi log chi tiết để phân tích. Không cập nhật trạng thái đơn hàng. |
| **BR-09** | **Chống replay webhook — cửa sổ ±5 phút** | Khi nhận webhook, kiểm tra `|now − x-api-time| ≤ 300 giây`. Webhook với `x-api-time` lệch quá 5 phút bị từ chối là replay attack. | Từ chối xử lý; trả HTTP 200 với `MessageError: "Request expired"` để không làm T-PayGate retry không cần thiết. Ghi log cảnh báo bảo mật. |
| **BR-10** | **Đối chiếu số tiền webhook trước khi xác nhận thanh toán** | Sau khi verify chữ ký và chống replay, so sánh `Amount` trong webhook với `totalAmount` của đơn hàng tương ứng: (1) `Amount ≥ totalAmount` → xác nhận đơn hàng "Đã thanh toán". (2) `Amount < totalAmount` → cập nhật trạng thái "Thanh toán một phần", chờ thanh toán bổ sung. | Không bao giờ tự động đánh dấu "Đã thanh toán" khi số tiền chưa đủ. Ghi log cảnh báo khi phát hiện thanh toán thiếu. |
| **BR-11** | **Idempotency webhook — chống xử lý kép** | Tra cứu đơn hàng theo `RefTransactionId` và `BillCode`. Nếu đơn hàng đã ở trạng thái "Đã thanh toán" → bỏ qua (retry/trùng), trả HTTP 200 với `MessageError` rỗng. Nếu không tìm thấy giao dịch → trả HTTP 200 với `MessageError: "Transaction not found"`. | Không double-process: ghi nhận một lần duy nhất. Sử dụng cơ chế lock hoặc atomic update để xử lý concurrency khi webhook đến song song. |
| **BR-12** | **Phản hồi webhook đúng format** | Sau khi xử lý webhook thành công, trả HTTP 200 với body `{ "MessageError": null }` hoặc chuỗi rỗng. Không trả `{ "MessageError": true }` (sai sẽ bị T-PayGate coi là lỗi và retry). Chỉ đặt `MessageError` có nội dung khi thật sự muốn T-PayGate gửi lại. | Nếu POS trả sai format → T-PayGate retry vô hạn, gây tải hệ thống và log nhiễu. Bắt buộc kiểm tra format response webhook trong test UAT. |
| **BR-13** | **Cập nhật UI thời gian thực sau khi nhận webhook** | Sau khi commit DB cập nhật trạng thái đơn hàng, hệ thống phát sự kiện realtime (WebSocket/SSE) về client POS. Đồng thời duy trì polling định kỳ (~5 giây) làm dự phòng trường hợp WebSocket mất kết nối. | Nếu WebSocket bị mất kết nối, polling dự phòng đảm bảo màn hình QR tự động chuyển sang "Thanh toán thành công" trong tối đa ~5 giây sau khi webhook được xử lý. |
| **BR-14** | **Lưu trữ BillCode và thông tin bill vào DB nội bộ** | Sau khi tạo bill thành công, POS lưu vào DB: `refTransactionId`, `billCode`, `amount`, `configBankId` (hoặc `configId`), `createdAt`, `status = "Pending"`. Đây là cơ sở để đối soát và xử lý webhook. | Nếu lưu DB thất bại sau khi bill đã tạo trên T-PayGate → ghi log khẩn cấp kèm `BillCode`, cần đối soát thủ công. |
| **BR-15** | **Cho phép thu ngân hủy QR và chuyển phương thức thanh toán** | Thu ngân có thể nhấn "Hủy" hoặc "Chọn phương thức khác" để thoát khỏi màn hình QR bất kỳ lúc nào trước khi webhook thanh toán về. Hành động này chỉ hủy nội bộ (đơn hàng quay về trạng thái chờ); `BillCode` trên T-PayGate vẫn tồn tại. Nếu sau đó khách thanh toán QR cũ → webhook về, POS đối chiếu theo `RefTransactionId` để xử lý hoặc từ chối. | Sau khi thu ngân hủy QR và chọn phương thức khác (VD: tiền mặt) → đánh dấu `BillCode` cũ là "Đã hủy nội bộ" trong DB POS. Nếu webhook cũ về với bill đã hủy nội bộ → từ chối xử lý, trả 200 ack thành công, log cảnh báo. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Các sơ đồ đầy đủ tại `docs/qr-payment/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/qr-payment/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** T-PayGate có rate limit cho `POST /order/bill` không (số lần gọi tối đa trong khoảng thời gian nhất định)? Nếu có, ngưỡng cụ thể là bao nhiêu để POS thiết kế circuit breaker phù hợp?
- [ ] **OQ-02:** Khi `QrDataURL` rỗng (service render QR của T-PayGate lỗi), POS fallback render từ chuỗi EMV `QRBase64` — thư viện render QR phía client nào được khuyến nghị? Cần xác nhận thư viện tương thích với chuẩn EMV VietQR.
- [ ] **OQ-03:** Chính sách retry webhook thực tế của T-PayGate là bao nhiêu lần, khoảng cách giữa các lần retry, và thời gian tối đa? Cần thông tin này để POS thiết kế cơ chế đối soát bổ sung khi webhook không về (gọi `GET /order/get-billCode` định kỳ).
- [ ] **OQ-04:** T-PayGate có hỗ trợ cơ chế hủy bill đã tạo không (VD: khi đơn hàng bị hủy trên POS)? Tài liệu v2.1 không đề cập API hủy bill trong public-api. Cần xác nhận để thiết kế luồng hủy đơn có bill QR đang chờ.
- [ ] **OQ-05:** Khi thanh toán một phần (`Thanh toán một phần`), T-PayGate có tự động gửi webhook bổ sung khi nhận thêm tiền cho cùng `BillCode` không? Hay mỗi giao dịch ghi có riêng lẻ sẽ là một webhook riêng?

---

## 6. References

- `docs/tpaygate/tai-lieu-tich-hop-cho-doi-tac-thu-3.md` — Tài liệu tích hợp T-PayGate v2.1 (nguồn chính)
- `docs/tpaygate/cap-phat-thong-so-cho-doi-tac-thu-3.md` — Quy trình cấp phát thông số onboarding
- `[[docs/tpaygate/srs/spec|T-PayGate Connect Bank SRS]]` — Đặc tả tính năng Thêm liên kết ngân hàng đối tác
- `[[docs/tpaygate-disconnect-bank/srs/spec|T-PayGate Disconnect Bank SRS]]` — Đặc tả tính năng Ngắt kết nối ngân hàng
- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- `@../screen/SKILL.md` — Skill Con phụ trách đặc tả màn hình
- T-PayGate Public API — `POST /api/v1/public-api/order/bill`
- T-PayGate Public API — `GET /api/v1/public-api/order/get-refTransactionId`
- T-PayGate Public API — `GET /api/v1/public-api/order/get-billCode`
- VietQR Standard — EMV QRCPS Merchant Presented Mode (MPM)
