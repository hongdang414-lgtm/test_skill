---
feature: cash-drawer
type: srs-spec
version: 1.0.0
updated: 2026-08-12
status: draft
authors: [BA Team]
changelog:
  - 2026-08-12 | /srs-spec | bỏ dependency ca làm việc (phần mềm chưa có quản lý ca), 14→12 BR, cập nhật flows + screens
  - 2026-08-12 | /srs-spec | [spec] initialized cash-drawer specification, 14 business rules
  - 2026-08-12 | /srs-spec | [flows] added sequence + state diagrams for cash-drawer
  - 2026-08-12 | /screen   | [screens] added specification and navigation for cash-drawer screens
---

# Đặc tả tính năng: Kết nối két tiền & Tự động mở két khi thanh toán tiền mặt (Cash Drawer)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép cửa hàng **kết nối két tiền vật lý** với máy POS và **tự động mở két ngay khi thu ngân xác nhận thanh toán tiền mặt** cho đơn hàng, đồng thời hỗ trợ **mở két thủ công khi cần** (đổi tiền lẻ, nạp tiền, kiểm tra) và **ghi nhận lịch sử mở két** phục vụ theo dõi/kiểm soát. Phạm vi bao gồm: màn hình cấu hình kết nối két (quản lý), luồng mở két tự động gắn vào bước xác nhận thanh toán tiền mặt, nút mở két thủ công kèm lý do, và danh sách lịch sử mở két. **Ranh giới (ngoài phạm vi):** việc đếm tiền mặt thực tế và đối soát quỹ (nếu có) thuộc tính năng riêng; việc mở két KHÔNG áp dụng cho thanh toán thẻ, QR, chuyển khoản hay voucher. |
| **2. Actors (Tác nhân)** | **Chính:** Thu ngân (Cashier) — thực hiện thanh toán tiền mặt (kích hoạt mở tự động) và mở két thủ công khi cần.<br/>**Phụ trợ:** Quản lý cửa hàng (Store Manager) — cấu hình kết nối két, bật/tắt kích hoạt, xem lịch sử mở két để kiểm tra.<br/>**Hỗ trợ hệ thống:** Thiết bị két tiền (Cash Drawer hardware) — nhận lệnh mở và (nếu có cảm biến) phản hồi trạng thái; Máy POS/Terminal — thiết bị vật lý mà két đang gắn vào. |
| **3. Pre-conditions** | 1. Cửa hàng có thiết bị POS/terminal đang hoạt động và đã gắn két tiền vật lý.<br/>2. Quản lý đã **cấu hình kết nối két** trên POS (chọn thiết bị/kênh mở) và đặt trạng thái **"Đang kích hoạt"**.<br/>3. Thu ngân đã **đăng nhập** vào hệ thống (mọi lần mở két đều ghi nhận người mở và thời gian).<br/>4. Đối với mở tự động: tồn tại đơn hàng ở trạng thái chờ thanh toán và thu ngân chọn phương thức **"Tiền mặt"**.<br/>5. Đối với mở thủ công: thu ngân có quyền mở két và sẵn sàng chọn lý do mở. |
| **4. Expected Results** | **Happy Path — Mở tự động khi thanh toán tiền mặt:**<br/>1. Thu ngân chọn "Tiền mặt" tại màn hình thanh toán, nhập số tiền khách đưa.<br/>2. Thu ngân bấm **"Xác nhận thanh toán"**.<br/>3. POS kiểm tra đã cấu hình két và đang kích hoạt → gửi lệnh mở két tới thiết bị.<br/>4. Két mở thành công → POS ghi lịch sử mở (loại: tự động, thu ngân, mã đơn hàng, thời gian, kết quả: thành công).<br/>5. POS hoàn tất giao dịch, hiển thị **tiền thừa trả khách**, thu ngân đếm tiền trả lại rồi đóng két.<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Chưa cấu hình két / không kích hoạt]:** POS phát hiện chưa có kết nối két đang kích hoạt → KHÔNG gửi lệnh, vẫn hoàn tất thanh toán, hiển thị thông báo nhỏ *"Chưa kết nối két tiền, mở két thủ công nếu cần"* và ghi lịch sử mở (kết quả: thất bại — chưa cấu hình).<br/>- **[B — Mất kết nối / lỗi thiết bị khi mở tự động]:** POS gửi lệnh nhưng không nhận xác nhận mở (timeout/lỗi) → **vẫn hoàn tất giao dịch**, hiển thị cảnh báo rõ *"Két tiền không mở được, vui lòng mở thủ công"*, ghi lịch sử (kết quả: thất bại — lỗi kết nối), thu ngân có thể bấm "Mở két" thử lại ngay.<br/>- **[C — Két đã mở sẵn]:** POS nhận tín hiệu két đang mở (cảm biến) → KHÔNG gửi lệnh mở lại, hiển thị nhắc *"Két đang mở, vui lòng đóng lại"*, ghi lịch sử (kết quả: đã mở sẵn).<br/>- **[D — Mở thủ công khi cần]:** Thu ngân bấm "Mở két" → chọn lý do (Đổi tiền lẻ / Nạp tiền / Kiểm tra / Khác) → két mở → ghi lịch sử (loại: thủ công, thu ngân, lý do, kết quả: thành công, không gắn đơn hàng).<br/>- **[E — Mở thủ công lỗi]:** Tương tự nhánh B nhưng cho thao tác thủ công — cảnh báo + cho thử lại + ghi lịch sử thất bại.<br/>- **[F — Thanh toán KHÔNG phải tiền mặt]:** Thu ngân chọn thẻ/QR/chuyển khoản/voucher → két KHÔNG mở tự động, không có hành động nào. |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **[Constraint] Chỉ mở két cho thanh toán tiền mặt** | Két chỉ mở tự động khi phương thức thanh toán được chọn là **TIỀN MẶT**. Thẻ, QR, chuyển khoản, voucher hoặc thanh toán hỗn hợp không chứa tiền mặt đều không kích hoạt mở két. | Nếu thanh toán không phải tiền mặt → không gửi lệnh mở, không hiển thị gì về két, không ghi lịch sử mở tự động. |
| **BR-02** | **[Constraint] Mỗi terminal chỉ một kênh kết nối đang kích hoạt** | Một máy POS/terminal chỉ được có **một** kết nối két ở trạng thái "Đang kích hoạt" tại một thời điểm. | Nếu quản lý cấu hình/kích hoạt kết nối mới khi đã có kết nối đang kích hoạt → cảnh báo *"Đã có két đang kích hoạt, bật cái mới sẽ tắt cái cũ?"*, yêu cầu xác nhận trước khi chuyển. |
| **BR-03** | **[Action Enabler] Điều kiện mở tự động** | Mở két tự động chỉ kích hoạt khi đủ ba điều kiện: (1) đã cấu hình két và đang kích hoạt; (2) phương thức thanh toán = tiền mặt; (3) thu ngân bấm "Xác nhận thanh toán". | Thiếu điều kiện nào → đi vào nhánh tương ứng (A chưa cấu hình / F không tiền mặt). Không gửi lệnh mở khi thiếu điều kiện. |
| **BR-04** | **[Action Enabler] Điều kiện mở thủ công** | Mở thủ công kích hoạt khi: thu ngân đã đăng nhập AND có quyền mở két AND đã chọn lý do hợp lệ. | Chưa chọn lý do → vô hiệu hóa nút "Mở két", yêu cầu chọn lý do trước. |
| **BR-05** | **[State Transition] Trạng thái vật lý két** | Két có hai trạng thái vật lý: **Đóng ↔ Mở**. Chuyển sang Mở khi nhận lệnh mở thành công; chuyển về Đóng khi cảm biến báo đóng (nếu có) hoặc thu ngân đóng tay. | Nếu không có cảm biến trạng thái → hệ thống ghi nhận trạng thái "best-effort" (xem OQ-01), không bắt buộc theo dõi chính xác. |
| **BR-06** | **[State Transition] Trạng thái kết nối két** | Kết nối có ba trạng thái: **Chưa cấu hình → Đang kết nối (kích hoạt) ↔ Mất kết nối**. Quản lý cấu hình + bật kích hoạt để sang "Đang kết nối"; thiết bị không phản hồi → "Mất kết nối"; quản lý tắt kích hoạt/xóa → về "Chưa cấu hình". | Khi ở "Mất kết nối": hiển thị cảnh báo trạng thái trên thanh toán, không tự tắt kết nối, chờ quản lý cấu hình lại (nhánh B). |
| **BR-07** | **[Data Validation] Lý do mở thủ công** | Lý do mở thủ công phải thuộc danh sách: **Đổi tiền lẻ**, **Nạp tiền vào két**, **Kiểm tra/đếm tiền**, **Khác**. Khi chọn "Khác", bắt buộc nhập ghi chú tự do tối thiểu 3 ký tự. | Lý do không hợp lệ / chọn "Khác" mà bỏ trống ghi chú → vô hiệu hóa nút mở, thông báo *"Vui lòng chọn lý do / nhập ghi chú"*. |
| **BR-08** | **[Data Validation] Bản ghi lịch sử mở két** | Mỗi bản ghi lịch sử mở két phải đầy đủ: **loại mở** (tự động/thủ công), **thời gian**, **thu ngân**, **lý do** (thủ công) hoặc **mã đơn hàng** (tự động), **kết quả** (thành công/thất bại/đã mở sẵn) và **chi tiết lỗi** (nếu có). | Trường nào thiếu → không ghi bản ghi mồ côi; đánh dấu cảnh báo dữ liệu theo dõi thiếu, yêu cầu kiểm tra. |
| **BR-09** | **[Constraint] Lỗi mở két không chặn giao dịch** | Mọi lỗi mở két (chưa cấu hình, mất kết nối, lỗi thiết bị, đã mở sẵn) **KHÔNG được chặn** việc hoàn tất giao dịch tiền mặt. Giao dịch vẫn hoàn tất, cảnh báo hiển thị rõ ràng, thu ngân có thể mở thủ công ngay. | Tuyệt đối không giữ giao dịch ở trạng thái treo vì lỗi két; cảnh báo phải rõ ràng và có nút "Mở két" ngay tại chỗ. |
| **BR-10** | **[Constraint] Ghi lịch sử mọi lần mở** | Mọi lần mở két — tự động và thủ công, thành công và thất bại — đều được ghi vào lịch sử mở két để theo dõi/kiểm soát. Thu ngân không được xóa log; chỉ quản lý có quyền xem. | Nếu ghi lịch sử thất bại → cảnh báo dữ liệu theo dõi thiếu; vẫn cho phép giao dịch tiếp, đánh dấu bản cần kiểm tra thủ công. |
| **BR-11** | **[Constraint] Cảnh báo mất kết nối không tự động tắt** | Khi két đang kích hoạt nhưng không phản hồi (mất kết nối), POS hiển thị cảnh báo trạng thái trên màn hình thanh toán nhưng **không tự động tắt** kết nối. Việc cấu hình lại do quản lý quyết định. | Tránh tắt nhầm kết nối đang tạm lỗi; để quản lý xác nhận trước khi chuyển trạng thái. |
| **BR-12** | **[Constraint] Không mở lại khi đã mở sẵn** | Nếu cảm biến báo két đang mở, POS không gửi lệnh mở lại; hiển thị nhắc đóng két, ghi lịch sử (kết quả: đã mở sẵn). | Tránh lặp lệnh mở không cần thiết và nhắc thu ngân đóng két trước khi thao tác tiếp. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Sơ đồ đầy đủ tại `docs/cash-drawer/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/cash-drawer/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** Thiết bị két tiền thực tế có **cảm biến trạng thái** (báo mở/đóng, báo "đã mở sẵn") không? Nhiều dòng két tiêu chuẩn giá rẻ không có cảm biến. Nếu không có, các nhánh "két đã mở sẵn" (BR-12) và "xác nhận mở thành công" không thực thi được — cần xác nhận để giảm nhánh hoặc giả định best-effort (chỉ gửi lệnh và ghi kết quả gửi).
- [ ] **OQ-02:** Khi cấu hình kết nối, quản lý chọn **thiết bị/kênh mở két là gì** (ví dụ qua máy in hóa đơn tại quầy, hay kết nối riêng)? Câu trả lời ảnh hưởng trực tiếp màn hình "Cấu hình kết nối két" (danh sách thiết bị nào để chọn).
- [ ] **OQ-03:** Khi phần mềm có **quản lý ca / đối soát quỹ** trong tương lai, lịch sử mở két nên tích hợp ở điểm giao tiếp nào (vd màn hình chốt ca)? Hiện tại tính năng chỉ ghi lịch sử phục vụ theo dõi/kiểm soát — cần chốt sẵn điểm tích hợp để tránh phải làm lại sau.
- [ ] **OQ-04:** Lịch sử mở két **lưu bao lâu** và có cần **xuất báo cáo riêng** (PDF/Excel) không? Liên quan chính sách lưu trữ và nhu cầu báo cáo của quản lý.

---

## 6. References

- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- `@../screen/SKILL.md` — Skill Con phụ trách đặc tả màn hình
- `[[docs/qr-payment/srs/spec|QR Payment SRS]]` — tính năng thanh toán QR (nơi nhắc đến tiền mặt như phương thức thay thế)
- Tiêu chuẩn phần cứng két tiền POS — cơ chế mở thường qua máy in hóa đơn (drawer kickout), lệnh mở không bắt buộc xác nhận trạng thái nếu két không có cảm biến.
