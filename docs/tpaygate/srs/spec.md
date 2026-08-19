---
feature: tpaygate-connect-bank
type: srs-spec
version: 1.0.0
updated: 2026-07-30
status: draft
authors: [BA Team]
changelog:
  - 2026-07-30 | /srs-spec   | [spec] initialized tpaygate-connect-bank specification, 18 business rules
  - 2026-07-30 | /activity   | [flows] added tpaygate-connect-bank-main-flow activity, connect-bank-lifecycle
  - 2026-07-30 | /sequence   | [flows] added tpaygate-connect-bank-sequence sequence, 4 actors
  - 2026-07-30 | /screen     | [screens] added specification for 4 screens
  - 2026-07-31 | /srs-spec   | [spec] refactored: removed disconnect rules and linked to separate tpaygate-disconnect-bank SRS
---

# Đặc tả tính năng: Thêm liên kết ngân hàng đối tác (T-PayGate Connect Bank)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép đối tác thứ 3 (POS, ERP, eCommerce…) **liên kết tài khoản ngân hàng của merchant với T-PayGate** để sinh mã VietQR và nhận thanh toán tự động. Mỗi cặp (merchant, ngân hàng) chỉ cần **kết nối một lần**; sau đó dùng `ConfigBankId` kết quả để tạo hóa đơn (bill) cho mọi giao dịch tiếp theo. Phạm vi: màn hình quản lý kết nối ngân hàng trong hệ thống đối tác (front-end đối tác), giao tiếp với **Public API của T-PayGate**. Hỗ trợ hai luồng kết nối: **Embedded UI** (T-PayGate cung cấp giao diện) và **API trực tiếp** (đối tác tự thiết kế UI). |
| **2. Actors (Tác nhân)** | **Chính:** Người dùng đối tác (nhân viên / quản lý merchant thực hiện kết nối ngân hàng).<br/>**Hỗ trợ:** T-PayGate API (xử lý nghiệp vụ kết nối, sinh ConfigBankId, VA Number), Ngân hàng đối tác (xác thực tài khoản, gửi OTP), Database phía đối tác (lưu ConfigBankId), T-PayGate Admin (cấp phát thông số onboarding: `clientId`, `tenantId`, `source`, `clientSecret`). |
| **3. Pre-conditions** | 1. T-PayGate đã cấp đủ 4 thông số onboarding: `clientId`, `tenantId`, `source`, `clientSecret`.<br/>2. Đối tác đã cấu hình các thông số đó vào hệ thống (không để trống).<br/>3. Đối tác đã lấy được `Access Token` hợp lệ từ endpoint OAuth (`POST /api/v1/oauth/token`).<br/>4. Đồng hồ hệ thống máy chủ đối tác đồng bộ NTP (lệch < 5 phút so với T-PayGate).<br/>5. Ngân hàng muốn kết nối có `IsConnectProvider = true` (kiểm tra qua `GET /api/v1/public-api/bank`). |
| **4. Expected Results** | **Happy Path — Luồng Embedded UI:**<br/>1. Người dùng mở màn hình Danh sách liên kết ngân hàng.<br/>2. Nhấn "Thêm liên kết ngân hàng" → chọn ngân hàng từ danh sách.<br/>3. Hệ thống mở trang T-PayGate Embedded UI (popup/tab) để nhập thông tin tài khoản và OTP.<br/>4. T-PayGate xử lý kết nối, gửi `postMessage("tabClosed")` khi hoàn tất.<br/>5. Hệ thống đối tác nhận tín hiệu, gọi lại `/config-bank/list` để lấy `ConfigBankId` mới.<br/>6. Màn hình cập nhật — hiển thị kết nối mới với trạng thái `IsConnected = true`.<br/><br/>**Happy Path — Luồng API trực tiếp (có OTP):**<br/>1. Người dùng nhập thông tin tài khoản vào form đối tác.<br/>2. Đối tác gọi `POST /config-bank/connect` (có ký HMAC-SHA256).<br/>3. T-PayGate trả `IsOTPConfirmation = true`, hiển thị màn nhập OTP.<br/>4. Người dùng nhập OTP, đối tác gọi `POST /config-bank/confirm`.<br/>5. T-PayGate trả `IsConnected = true` + `VaNumber` → lưu vào DB.<br/>6. Màn hình danh sách cập nhật.<br/><br/>**Alternate Branches:**<br/>- **[A — Kết nối không cần OTP]:** `IsConnected = true` ngay ở response `/connect`, nhưng không có `VaNumber`. Gọi thêm `/config-bank/list` để lấy `VaNumber`.<br/>- **[B — Ngân hàng không hỗ trợ]:** Ngân hàng không có trong danh sách hoặc `IsConnectProvider = false` → không hiển thị trong danh sách chọn.<br/>- **[C — Kết nối đã tồn tại]:** Đối tác gọi `/connect` lần 2 với cùng tài khoản → T-PayGate có thể sinh thêm `ConfigBankId` mới (không idempotent). Đối tác phải tự kiểm tra trùng lặp trước khi gọi API.<br/>- **[D — Hủy kết nối]:** Được đặc tả và xử lý độc lập tại tính năng Ngắt kết nối ngân hàng (`tpaygate-disconnect-bank`). Xem chi tiết tại `[[docs/tpaygate-disconnect-bank/srs/spec|T-PayGate Disconnect Bank SRS]]`. |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **Phải có thông số T-PayGate trước khi thực hiện** | Trước khi cho phép thêm liên kết ngân hàng, hệ thống phải kiểm tra 4 thông số bắt buộc: `clientId`, `tenantId`, `source`, `clientSecret`. Nếu bất kỳ thông số nào thiếu hoặc trống → chặn toàn bộ tính năng. | Ẩn/disable nút "Thêm liên kết". Hiển thị banner: _"Chưa cấu hình thông số T-PayGate. Liên hệ quản trị viên để hoàn tất cài đặt."_ |
| **BR-02** | **Chỉ hiển thị ngân hàng đang hỗ trợ kết nối** | Danh sách ngân hàng để chọn khi thêm liên kết phải lấy động từ `GET /api/v1/public-api/bank`. Chỉ hiển thị các ngân hàng đang hoạt động (`IsActive = true`). Không được hardcode danh sách ngân hàng trong ứng dụng. | Nếu API `/bank` lỗi → hiển thị thông báo: _"Không thể tải danh sách ngân hàng. Vui lòng thử lại."_ Không hiển thị ngân hàng nào. |
| **BR-03** | **Gọi `/bank` phải đúng phương thức xác thực** | Endpoint `GET /api/v1/public-api/bank` hoạt động ẩn danh (không cần token). Nếu gọi kèm `Authorization: Bearer` mà **không có** `x-signature` và `x-api-time` → T-PayGate trả **403**. Phải chọn một trong hai: gọi ẩn danh hoàn toàn, hoặc gọi có Bearer kèm đủ chữ ký. | Áp dụng một trong hai chiến lược nhất quán. Nếu nhận 403 → log lỗi `TPAYGATE_SIGNATURE`, không hiển thị danh sách ngân hàng. |
| **BR-04** | **Mọi request `/config-bank/*` phải ký HMAC-SHA256** | Tất cả endpoint kết nối ngân hàng (`/connect`, `/confirm`, `/list`) đều yêu cầu chữ ký HMAC-SHA256 gửi kèm header `x-signature` và `x-api-time`. Công thức ký khác nhau giữa POST và GET: POST có kèm `rawBody`, GET không có body. Giá trị `x-api-time` phải là Unix timestamp (số nguyên, tính bằng giây). | Thiếu hoặc sai chữ ký → T-PayGate trả **403** `#TPayGate: Invalid signature.`<br/>`x-api-time` lệch > 5 phút → **403** `#TPayGate: Request expired.`<br/>Đồng hồ phải đồng bộ NTP trước go-live. |
| **BR-05** | **Không retry POST `/config-bank/connect` khi timeout** | Endpoint `/config-bank/connect` không idempotent về phía T-PayGate (mỗi lần gọi có thể sinh một `ConfigBankId` mới). Nếu request timeout, đối tác phải **trước tiên gọi** `GET /config-bank/list` để kiểm tra xem kết nối đã được tạo chưa, trước khi quyết định tạo lại. | Nếu kết nối đã tồn tại với `IsConnected = true` và cùng `accountNo` → thông báo: _"Tài khoản này đã được kết nối. Không cần thêm lại."_ Không tạo bản ghi trùng. |
| **BR-06** | **Thông tin tài khoản ngân hàng bắt buộc** | Khi kết nối qua API trực tiếp, các trường sau bắt buộc: `bankCode`, `merchantName`, `accountName`, `accountNo`. Các trường `identity`, `phone`, `email`, `prefix`, `clientId` (mã ngân hàng cấp), `encryptKey`, `secretKey` là tùy chọn, phụ thuộc vào yêu cầu của từng ngân hàng cụ thể. | *Lỗi bắt buộc:* _"[Tên trường] không được để trống."_<br/>*Lỗi độ dài:* _"[Tên trường] không hợp lệ."_ |
| **BR-07** | **`clientId` trong body connect khác `clientId` OAuth** | Trường `clientId` trong request body của `/config-bank/connect` là **mã định danh do ngân hàng cấp cho merchant** (khác hoàn toàn với `clientId` dùng trong OAuth và ký HMAC). Phải tách biệt hai giá trị này trong cấu hình và code. | Nếu dùng nhầm `clientId` OAuth làm `clientId` ngân hàng → kết nối thất bại phía ngân hàng hoặc gây lỗi nghiệp vụ không rõ ràng. Ghi chú rõ trên form: _"Mã do ngân hàng cấp (khác với Client ID T-PayGate)"_. |
| **BR-08** | **`ConfigBankId` phải kiểm tra trùng trước khi lưu** | Sau khi gọi `/connect` hoặc `/confirm` thành công, trước khi INSERT record vào DB phía đối tác, phải kiểm tra `ConfigBankId` trả về đã tồn tại chưa. Nếu đã tồn tại (do retry, do mạng không ổn định) → UPDATE thay vì INSERT thêm. | Ghi đè record cũ (UPSERT). Không tạo bản ghi trùng `ConfigBankId` trong DB. |
| **BR-09** | **Lấy `VaNumber` khi kết nối không qua OTP** | Khi T-PayGate trả `IsConnected = true` ngay từ response `/connect` (không cần OTP), response đó **không chứa `VaNumber`**. Phải gọi thêm `GET /config-bank/list` và lọc theo `ConfigBankId` để lấy `VaNumber`. | Nếu không gọi lại `/list`, `VaNumber` sẽ hiển thị trống trong giao diện. Bắt buộc thực hiện bước này trong luồng code. |
| **BR-10** | **Tín hiệu `tabClosed` chỉ là lệnh refresh, không phải bằng chứng thành công** | Khi dùng Embedded UI, T-PayGate gửi `postMessage("tabClosed")` hoặc `postMessage({ type: "tabClosed" })`. Tín hiệu này chỉ có nghĩa "tab đã đóng" — **không xác nhận kết nối thành công**. Sau khi nhận tín hiệu, bắt buộc gọi lại `GET /config-bank/list` để xác nhận `IsConnected`. | Nếu không gọi lại `/list`, có thể hiển thị trạng thái thành công sai khi người dùng đóng tab giữa chừng. |
| **BR-11** | **Bắt buộc kiểm tra `event.origin` khi nhận postMessage** | Xử lý `window.addEventListener("message", ...)` phải kiểm tra `event.origin` khớp với origin của T-PayGate trước khi xử lý nội dung. Bỏ qua kiểm tra này cho phép bất kỳ trang web nào giả lập sự kiện "kết nối thành công". | Từ chối xử lý nếu `event.origin` không hợp lệ. Log cảnh báo bảo mật. |
| **BR-12** | **Không tự lấy Base URL từ input người dùng** | URL kết nối T-PayGate (Staging / Production) phải map cứng theo môi trường, không để người dùng nhập tự do. Staging: `https://t-paygate.tpos.dev`; Production: `https://t-paygate.tpos.app`. | Nếu để người dùng nhập URL tuỳ ý → nguy cơ trỏ nhầm môi trường thanh toán thật. Chỉ cho phép chọn từ danh sách có sẵn. |
| **BR-13** | **Trạng thái khởi tạo sau khi kết nối ngân hàng** | Khi gọi `/connect` hoặc `/confirm` thành công, liên kết ngân hàng (`ConfigBankId`) được tạo với trạng thái `IsConnected = true` (tài khoản đang hoạt động, có thể dùng để tạo bill). Đối với nghiệp vụ hủy/ngắt kết nối (`IsConnected = false`), áp dụng quy tắc tại `[[docs/tpaygate-disconnect-bank/srs/spec|T-PayGate Disconnect Bank SRS]]`. | Không được cập nhật sang `IsConnected = false` nếu không đi qua luồng Ngắt kết nối ngân hàng hợp lệ. |
| **BR-14** | **OTP phải đúng định dạng** | OTP do ngân hàng gửi thường gồm 6 chữ số. Validate `^[0-9]{4,8}$` trước khi gọi `/config-bank/confirm` để tránh request dư thừa. Độ dài OTP thực tế phụ thuộc từng ngân hàng — xác nhận với T-PayGate khi mở rộng sang ngân hàng mới. | *Lỗi:* _"OTP không hợp lệ. Vui lòng kiểm tra lại."_ (validate phía client, không gọi API). |
| **BR-15** | **Mã ngân hàng phải lấy từ API, không hardcode** | `bankCode` gửi lên `/config-bank/connect` phải là giá trị `Code` lấy từ response `GET /api/v1/public-api/bank` (chuẩn NAPAS). Không được hardcode danh sách mã ngân hàng trong code. | Nếu hardcode → có thể dùng sai mã khi ngân hàng thay đổi hoặc khi T-PayGate thêm/xóa ngân hàng. Phải load danh sách ngân hàng từ API trước khi hiển thị form. |
| **BR-16** | **Ngân hàng đã kết nối không hiển thị là "Chưa kết nối"** | Sau khi gọi `/connect` hoặc `/confirm` thành công, phải gọi lại `GET /config-bank/list` để đồng bộ danh sách trước khi cập nhật UI. Không được cập nhật trạng thái UI dựa thuần vào response `/connect` mà không có bước xác nhận. | Nếu bỏ qua bước xác nhận → hiển thị trạng thái không chính xác, đặc biệt với luồng không OTP (BR-09). |
| **BR-17** | **Mã ngân hàng trong body connect không được nhầm với `clientId` OAuth** | Trường `clientId` trong body `/config-bank/connect` là optional, là mã định danh merchant do **ngân hàng** cấp. Không phải `clientId` OAuth của T-PayGate. Đặt tên biến và label rõ ràng để tránh nhầm lẫn. | *Lỗi ngầm:* Gán nhầm → kết nối ngân hàng có thể thất bại mà không có thông báo rõ ràng, hoặc ngân hàng từ chối kết nối do sai định danh merchant. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Các sơ đồ đầy đủ tại `docs/tpaygate/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/tpaygate/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** T-PayGate có giới hạn số lượng liên kết ngân hàng tối đa trên 1 `tenantId` không?
- [ ] **OQ-02:** Khi ngân hàng đổi tên hoặc thay đổi `bankCode` trong hệ thống T-PayGate, các liên kết cũ có bị ảnh hưởng không? `ConfigBankId` cũ vẫn hoạt động bình thường?
- [ ] **OQ-03:** Nếu đối tác có tenant con (nhiều kênh/shop), `ConfigBankId` của tenant con có độc lập với tenant cha không?
- [ ] **OQ-04:** Chính sách rate limit của T-PayGate cho `/config-bank/connect` là bao nhiêu? Có nguy cơ bị HTTP 429 khi nhiều merchant onboard đồng thời không?
- [ ] **OQ-05:** Các field tùy chọn (`identity`, `prefix`, `encryptKey`, `secretKey`) — điều kiện nào quyết định ngân hàng nào yêu cầu field nào? T-PayGate có API trả về cấu hình field theo `bankCode` không, hay phải hỏi thủ công?
- [ ] **OQ-06:** Timeout gọi `/config-bank/connect` phía đối tác nên đặt là bao nhiêu giây? T-PayGate phải gọi sang ngân hàng để xử lý, có thể chậm hơn các endpoint khác.

---

## 6. References

- `docs/tpaygate/tai-lieu-tich-hop-cho-doi-tac-thu-3.md` — Tài liệu tích hợp T-PayGate v2.1 (nguồn chính)
- `docs/tpaygate/cap-phat-thong-so-cho-doi-tac-thu-3.md` — Quy trình cấp phát thông số onboarding
- `[[docs/tpaygate-disconnect-bank/srs/spec|T-PayGate Disconnect Bank SRS]]` — Đặc tả tính năng Ngắt kết nối ngân hàng đối tác
- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- T-PayGate Public API — `POST /api/v1/public-api/config-bank/connect`
- T-PayGate Public API — `POST /api/v1/public-api/config-bank/confirm`
- T-PayGate Public API — `GET /api/v1/public-api/config-bank/list`
- T-PayGate Public API — `POST /api/v1/public-api/config-bank/disconnect`
- T-PayGate Public API — `GET /api/v1/public-api/bank`
