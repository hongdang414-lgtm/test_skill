---
feature: tpaygate-disconnect-bank
type: srs-spec
version: 1.0.0
updated: 2026-07-31
status: draft
authors: [BA Team]
changelog:
  - 2026-07-31 | /srs-spec   | [spec] initialized tpaygate-disconnect-bank specification, 10 business rules
  - 2026-07-31 | /activity   | [flows] added activity, sequence, and lifecycle diagrams for disconnect-bank
  - 2026-07-31 | /screen     | [screens] added specification for disconnect-bank-dialog and disconnected item state
---

# Đặc tả tính năng: Ngắt kết nối ngân hàng đối tác (T-PayGate Disconnect Bank)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép đối tác thứ 3 (POS, ERP, eCommerce…) **ngắt/hủy liên kết tài khoản ngân hàng của merchant với T-PayGate** thông qua API `POST /api/v1/public-api/config-bank/disconnect`. Khi thực hiện ngắt kết nối, bản ghi `ConfigBankId` được chuyển sang trạng thái `IsConnected = false` (không xóa cứng trong CSDL để phục vụ đối soát, tra cứu lịch sử hóa đơn và kiểm toán). Không cho phép tạo hóa đơn (bill) mới với `ConfigBankId` đã disconnect. Phạm vi: chức năng ngắt kết nối trên màn hình Quản lý liên kết ngân hàng trong hệ thống đối tác (front-end đối tác), giao tiếp với **Public API của T-PayGate**. |
| **2. Actors (Tác nhân)** | **Chính:** Người dùng đối tác (Quản lý cửa hàng / Chủ cửa hàng có quyền hạn quản lý cấu hình thanh toán).<br/>**Hỗ trợ:** T-PayGate API (xử lý hủy liên kết, trả về kết quả), Ngân hàng đối tác (xác nhận hủy đăng ký tài khoản liên kết), Database phía đối tác (cập nhật trạng thái `IsConnected = false`), T-PayGate Admin. |
| **3. Pre-conditions** | 1. Liên kết ngân hàng đang tồn tại trong hệ thống đối tác với trạng thái `IsConnected = true`.<br/>2. Người dùng thực hiện có quyền hạn (permission) Quản lý cửa hàng / Chủ cửa hàng.<br/>3. Đối tác có `Access Token` hợp lệ từ endpoint OAuth (`POST /api/v1/oauth/token`) và đủ 4 thông số onboarding (`clientId`, `tenantId`, `source`, `clientSecret`) để tính chữ ký HMAC-SHA256.<br/>4. Đồng hồ hệ thống máy chủ đối tác đồng bộ NTP (lệch < 5 phút so với T-PayGate). |
| **4. Expected Results** | **Happy Path — Luồng ngắt kết nối thành công:**<br/>1. Người dùng mở màn hình Danh sách liên kết ngân hàng, nhấn nút **"Ngắt kết nối"** trên một dòng liên kết đang hoạt động (`IsConnected = true`).<br/>2. Hệ thống hiển thị Dialog xác nhận ngắt kết nối (hiển thị rõ tên ngân hàng, số tài khoản, chủ tài khoản và cảnh báo về hậu quả nghiệp vụ).<br/>3. Người dùng xác nhận ngắt kết nối trên Dialog.<br/>4. Hệ thống đối tác gửi request `POST /api/v1/public-api/config-bank/disconnect` với body `{ "configBankId": "..." }` kèm chữ ký HMAC-SHA256 hợp lệ.<br/>5. T-PayGate gọi sang ngân hàng hủy đăng ký, trả HTTP 200 `{ "Error": null, "Data": ... }`.<br/>6. Hệ thống đối tác cập nhật trạng thái bản ghi trong DB sang `IsConnected = false` (giữ nguyên record để đối soát).<br/>7. UI cập nhật hiển thị dòng liên kết sang badge xám **"Đã ngắt kết nối"** và ẩn/disable nút Ngắt kết nối.<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Lỗi chữ ký hoặc hết hạn request]:** T-PayGate trả HTTP 403 `#TPayGate: Invalid signature.` hoặc `#TPayGate: Request expired.` → Không cập nhật DB, hiển thị lỗi rõ ràng trên giao diện để người dùng kiểm tra cấu hình hoặc thử lại.<br/>- **[B — ConfigBankId không tồn tại hoặc đã ngắt kết nối trước đó]:** T-PayGate trả HTTP 403 `#TPayGate: configBankId does not exist or is disconnected.` → Hệ thống tự động cập nhật trạng thái DB đối tác sang `IsConnected = false` (đồng bộ trạng thái) và thông báo tài khoản đã ngắt kết nối.<br/>- **[C — Request Timeout / Mạng không ổn định]:** Không tự động thử lại (no retry) khi gọi POST `/disconnect`. Bắt buộc gọi `GET /api/v1/public-api/config-bank/list` để kiểm tra trạng thái thực tế của `ConfigBankId` trước khi quyết định xử lý tiếp.<br/>- **[D — Hóa đơn cũ sau khi ngắt kết nối]:** Các hóa đơn (`billCode`) đã tạo trước thời điểm ngắt kết nối vẫn tiếp tục nhận thanh toán và webhook từ T-PayGate bình thường; chỉ việc tạo hóa đơn mới (`POST /api/v1/public-api/order/bill`) với `x-config-id` đã disconnect mới bị chặn từ chối (HTTP 403). |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **Chỉ được phép ngắt kết nối khi liên kết đang hoạt động** | Nút và chức năng "Ngắt kết nối" chỉ áp dụng cho các liên kết ngân hàng đang có trạng thái `IsConnected = true`. Các bản ghi đã ngắt kết nối (`IsConnected = false`) không được thực hiện lại hành động này. | Disable hoặc ẩn nút "Ngắt kết nối" đối với các dòng có `IsConnected = false`. Nếu gọi API nội bộ, từ chối request với lỗi: _"Liên kết ngân hàng đã bị ngắt kết nối trước đó."_ |
| **BR-02** | **Bắt buộc xác nhận từ người dùng qua Dialog trước khi gửi request** | Trước khi gọi API ngắt kết nối sang T-PayGate, hệ thống bắt buộc phải hiển thị Dialog xác nhận với thông tin chi tiết (tên ngân hàng, số tài khoản, tên chủ tài khoản) và lời cảnh báo về hậu quả nghiệp vụ. | Không được phép gọi thẳng API `POST /config-bank/disconnect` ngay sau một thao tác click mà không qua Dialog xác nhận. |
| **BR-03** | **Request ngắt kết nối phải được ký HMAC-SHA256 theo chuẩn POST** | Endpoint `POST /api/v1/public-api/config-bank/disconnect` yêu cầu chữ ký HMAC-SHA256 gửi trong header `x-signature` và thời gian Unix timestamp trong `x-api-time`. Chuỗi ký theo công thức POST: `{clientId}_{tenantId}_{source}_{timestamp}_{rawBody}` (4 dấu gạch dưới `_`, có kèm chuỗi JSON nguyên văn của body). | Thiếu hoặc sai chữ ký → T-PayGate trả HTTP 403 `#TPayGate: Invalid signature.`<br/>`x-api-time` lệch > 5 phút → HTTP 403 `#TPayGate: Request expired.`<br/>Hệ thống hiển thị lỗi và ghi log cảnh báo bảo mật. |
| **BR-04** | **Cập nhật trạng thái sau khi ngắt kết nối và tuyệt đối không xóa bản ghi DB** | Khi ngắt kết nối thành công, hệ thống đối tác chỉ được cập nhật trường `IsConnected = false` (hoặc `Status = Disconnected`) trong cơ sở dữ liệu. Tuyệt đối không xóa bản ghi (`DELETE`), vì `ConfigBankId` là khóa ngoại bắt buộc để đối soát, tra cứu hóa đơn và kiểm toán các giao dịch trong quá khứ. | Nếu xóa cứng bản ghi, các hóa đơn cũ sẽ bị mất liên kết thông tin tài khoản ngân hàng thụ hưởng. Bắt buộc sử dụng cơ chế soft-update/chuyển trạng thái. |
| **BR-05** | **Chặn tạo hóa đơn mới với ConfigBankId đã ngắt kết nối** | Mọi yêu cầu tạo hóa đơn thanh toán mới (`POST /api/v1/public-api/order/bill`) sử dụng `x-config-id` tương ứng với liên kết đã ngắt (`IsConnected = false`) đều bị cấm. Hệ thống đối tác phải kiểm tra trạng thái tại DB nội bộ trước khi gọi API T-PayGate. | *Lỗi validation:* _"Tài khoản ngân hàng này đã ngắt kết nối. Vui lòng chọn tài khoản khác hoặc kết nối lại để tạo hóa đơn."_ Nếu cố tình gọi lên T-PayGate, T-PayGate trả HTTP 403 `configBankId does not exist or is disconnected.` |
| **BR-06** | **Không tự động retry khi POST /config-bank/disconnect gặp timeout** | Nếu request gọi `POST /api/v1/public-api/config-bank/disconnect` gặp sự cố mạng hoặc timeout, hệ thống không được tự động gọi lại (retry). Bắt buộc phải gọi `GET /api/v1/public-api/config-bank/list` để xác định trạng thái thực tế của `ConfigBankId` trên T-PayGate trước khi xử lý. | Nếu `/list` trả về `IsConnected = false` → cập nhật DB nội bộ thành công mà không cần gọi `/disconnect` lại. Nếu vẫn `IsConnected = true` → hiển thị thông báo yêu cầu người dùng thử lại thủ công. |
| **BR-07** | **Đồng bộ trạng thái khi nhận lỗi HTTP 403 đã ngắt kết nối từ T-PayGate** | Khi gọi `/disconnect` hoặc bất kỳ API nào mà T-PayGate trả về lỗi HTTP 403 với thông báo `#TPayGate: configBankId does not exist or is disconnected.`, hệ thống đối tác tự động cập nhật trạng thái bản ghi trong DB sang `IsConnected = false` để đồng bộ với T-PayGate. | Cập nhật ngầm CSDL nội bộ sang `IsConnected = false`, đồng thời hiển thị thông báo cho người dùng: _"Tài khoản ngân hàng này đã được ngắt kết nối trên hệ thống T-PayGate."_ |
| **BR-08** | **Kiểm tra quyền hạn thực hiện ngắt kết nối** | Thao tác ngắt kết nối tác động trực tiếp đến khả năng thu tiền của cửa hàng. Chỉ tài khoản người dùng có quyền Quản lý cửa hàng / Chủ cửa hàng (Merchant Admin / Owner) mới được phép thực hiện hành động này. | Nếu người dùng không có quyền (nhân viên thu ngân, nhân viên kho...) → ẩn nút "Ngắt kết nối" hoặc hiển thị lỗi: _"Bạn không có quyền thực hiện ngắt kết nối ngân hàng. Vui lòng liên hệ Quản lý."_ |
| **BR-09** | **Xác thực tính hợp lệ của configBankId trước khi xử lý** | Trường `configBankId` gửi trong body của request `/disconnect` phải là chuỗi UUID hợp lệ, thuộc quyền sở hữu của `tenantId` hiện tại và tồn tại trong CSDL đối tác. | *Lỗi:* _"Mã liên kết ngân hàng không hợp lệ hoặc không tồn tại."_ Từ chối request tại client/server nội bộ trước khi gọi T-PayGate. |
| **BR-10** | **Không thể tái kích hoạt liên kết đã ngắt, phải tạo kết nối mới** | Một `ConfigBankId` đã chuyển sang trạng thái `IsConnected = false` sẽ vĩnh viễn không thể chuyển lại thành `IsConnected = true`. Khi muốn sử dụng lại tài khoản ngân hàng đó, người dùng phải thực hiện quy trình "Thêm liên kết ngân hàng" từ đầu, hệ thống sẽ sinh ra một `ConfigBankId` mới. | Không hiển thị nút "Kết nối lại" trên dòng đã ngắt kết nối. Hiển thị thông báo hướng dẫn: _"Để sử dụng lại tài khoản này, vui lòng nhấn 'Thêm liên kết ngân hàng' để tạo kết nối mới."_ |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Các sơ đồ đầy đủ tại `docs/tpaygate-disconnect-bank/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/tpaygate-disconnect-bank/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** Khi ngắt kết nối, các giao dịch đang ở trạng thái chờ thanh toán của hóa đơn cũ có bị ảnh hưởng gì tới việc nhận webhook từ T-PayGate không? *(Tài liệu T-PayGate nêu tạo bill mới bị 403, nhưng webhook cho bill cũ đã tạo trước thời điểm ngắt kết nối có đảm bảo gửi về không?)*
- [ ] **OQ-02:** T-PayGate có giới hạn thời gian lưu trữ record `ConfigBankId` đã ngắt kết nối trong `GET /config-bank/list` không *(ví dụ sau 90 ngày hoặc 1 năm có bị ẩn/xóa khỏi response list không)*?
- [ ] **OQ-03:** Nếu ngân hàng đối tác chủ động hủy kết nối từ phía ngân hàng (ví dụ tài khoản bị đóng/khóa), T-PayGate có gửi webhook hoặc notification báo về cho đối tác để tự động chuyển `IsConnected = false` không, hay đối tác chỉ phát hiện khi tạo bill bị lỗi 403?

---

## 6. References

- `docs/tpaygate/tai-lieu-tich-hop-cho-doi-tac-thu-3.md` — Tài liệu tích hợp T-PayGate v2.1 (nguồn chính)
- `docs/tpaygate/cap-phat-thong-so-cho-doi-tac-thu-3.md` — Quy trình cấp phát thông số onboarding
- `[[docs/tpaygate/srs/spec|T-PayGate Connect Bank SRS]]` — Đặc tả tính năng Thêm liên kết ngân hàng đối tác
- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- `@../screen/SKILL.md` — Skill Con phụ trách đặc tả màn hình
- T-PayGate Public API — `POST /api/v1/public-api/config-bank/disconnect`
- T-PayGate Public API — `GET /api/v1/public-api/config-bank/list`
