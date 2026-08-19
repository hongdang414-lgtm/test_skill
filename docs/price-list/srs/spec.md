---
feature: price-list
type: srs-spec
version: 1.0.0
updated: 2026-07-23
status: draft
authors: [BA Team]
changelog:
  - 2026-07-23 | /srs-spec | [spec] initialized price-list specification, 20 business rules
  - 2026-07-23 | /activity | [flows] added price-list-main-flow activity, 6 decisions
  - 2026-07-23 | /sequence | [flows] added price-list-system-sequence sequence, 6 actors
  - 2026-07-23 | /screen   | [screens] added specification for 4 screens
---

# Đặc tả tính năng: Thêm Bảng giá (Price List)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép doanh nghiệp **tạo và quản lý nhiều mức giá bán khác nhau** cho cùng một sản phẩm, combo hoặc tùy chọn (option/add-on), sau đó áp dụng mức giá phù hợp tùy theo nhóm khách hàng hoặc khung thời gian bán hàng. Phạm vi: CMS Back-Office (quản lý trên web). Bảng giá hỗ trợ thiết lập giá mới theo 2 cách: **nhập trực tiếp** hoặc **chiết khấu % / giảm giá trị cố định** so với giá gốc. Hệ thống đảm bảo tại mỗi thời điểm chỉ có tối đa **1 bảng giá** ở trạng thái "Đang diễn ra", tránh xung đột giá. Hỗ trợ **khung giờ áp dụng** (happy hour) trong khoảng thời gian hiệu lực. |
| **2. Actors (Tác nhân)** | **Chính:** Quản lý cửa hàng (Store Manager), Admin hệ thống.<br/>**Hỗ trợ:** CMS Backend, Product Service (cung cấp danh sách SP/Combo/Option), Database, Audit Log. |
| **3. Pre-conditions** | 1. Người dùng đã đăng nhập CMS với quyền `MANAGE_PRICE_LIST`.<br/>2. Hệ thống đã có ít nhất 1 sản phẩm/combo/option trong danh mục.<br/>3. Đã thiết lập ít nhất 1 nhóm khách hàng (nếu muốn gán phạm vi). |
| **4. Expected Results** | **Happy Path:**<br/>1. Quản lý mở Danh sách bảng giá trên CMS.<br/>2. Nhấn "Thêm bảng giá" → hệ thống mở form tạo mới.<br/>3. Nhập thông tin cơ bản: Tên, Mã, Mô tả, Thời gian áp dụng (từ – đến), Khung giờ (nếu có), Nhóm khách hàng.<br/>4. Thêm sản phẩm/combo/option vào bảng giá, thiết lập giá mới cho từng mục.<br/>5. Nhấn "Lưu" → Bảng giá được tạo ở trạng thái "Chưa diễn ra".<br/>6. Khi đến thời gian bắt đầu → hệ thống tự chuyển sang "Đang diễn ra" và áp dụng giá mới.<br/>7. Khi hết thời gian → tự chuyển "Đã kết thúc", giá trở về giá gốc.<br/><br/>**Alternate Branches:**<br/>- **[A — Trùng thời gian]:** Nếu khoảng thời gian trùng với bảng giá khác đang tồn tại → cảnh báo, không cho lưu.<br/>- **[B — Không có sản phẩm]:** Lưu bảng giá không có SP/Combo/Option → cảnh báo, yêu cầu thêm ít nhất 1 mục.<br/>- **[C — Sản phẩm bị xóa]:** SP đã thêm vào bảng giá bị xóa khỏi danh mục → tự động loại khỏi bảng giá, ghi audit log.<br/>- **[D — Sửa bảng giá đang diễn ra]:** Chỉ cho phép sửa giá sản phẩm, không cho sửa thông tin cơ bản (tên, mã, thời gian). |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **[Constraint] Phân quyền quản lý bảng giá** | Chỉ tài khoản có quyền `MANAGE_PRICE_LIST` mới được tạo, sửa, xóa bảng giá. | Ẩn menu "Bảng giá" khỏi sidebar CMS. API: `403 PERMISSION_DENIED`. |
| **BR-02** | **[Constraint] Tên bảng giá bắt buộc và unique** | Tên bảng giá là trường bắt buộc, độ dài 1–100 ký tự (Unicode), không trùng với bảng giá khác đang tồn tại (so sánh case-insensitive sau khi trim). | Lỗi: _"Tên bảng giá không được để trống."_<br/>Lỗi: _"Tên bảng giá đã tồn tại. Vui lòng chọn tên khác."_ |
| **BR-03** | **[Constraint] Mã bảng giá unique** | Mã bảng giá có 2 chế độ: **Tự động sinh** (mặc định, format `BG-{YYYYMMDD}-{SEQ}`) hoặc **Tự nhập** (format `^[A-Za-z0-9_-]{3,20}$`). Mã phải unique toàn hệ thống. | Lỗi: _"Mã bảng giá đã tồn tại."_<br/>Lỗi: _"Mã chỉ chấp nhận chữ cái, số, dấu gạch ngang và gạch dưới (3–20 ký tự)."_ |
| **BR-04** | **[Constraint] Thời gian áp dụng hợp lệ** | Bắt buộc nhập Thời gian bắt đầu và Thời gian kết thúc. Thời gian kết thúc phải sau Thời gian bắt đầu. Thời gian bắt đầu phải ≥ thời điểm hiện tại (không tạo bảng giá trong quá khứ). | Lỗi: _"Thời gian kết thúc phải sau thời gian bắt đầu."_<br/>Lỗi: _"Thời gian bắt đầu không được ở trong quá khứ."_ |
| **BR-05** | **[Constraint] Không trùng lặp thời gian** | Tại mỗi thời điểm, hệ thống chỉ cho phép tối đa **1 bảng giá** ở trạng thái "Chưa diễn ra" hoặc "Đang diễn ra" có khoảng thời gian trùng lặp (overlap). Khi tạo/sửa bảng giá, hệ thống kiểm tra overlap với tất cả bảng giá chưa kết thúc. | Lỗi: _"Thời gian áp dụng trùng với bảng giá [{TÊN}] ({THỜI GIAN}). Tại mỗi thời điểm chỉ được có 1 bảng giá hoạt động."_ |
| **BR-06** | **[Constraint] Không xóa bảng giá đang diễn ra** | Bảng giá ở trạng thái "Đang diễn ra" không được phép xóa. Chỉ cho phép xóa bảng giá "Chưa diễn ra" hoặc "Đã kết thúc". | Lỗi: _"Không thể xóa bảng giá đang diễn ra. Vui lòng chờ kết thúc hoặc điều chỉnh thời gian."_ |
| **BR-07** | **[Constraint] Sản phẩm không trùng trong 1 bảng giá** | Mỗi sản phẩm/combo/option chỉ được xuất hiện **1 lần** trong cùng 1 bảng giá. Nếu thêm mục đã tồn tại → cảnh báo và cho phép cập nhật giá thay vì thêm trùng. | Lỗi: _"Sản phẩm [{TÊN}] đã có trong bảng giá. Bạn có muốn cập nhật giá?"_ |
| **BR-08** | **[Constraint] Giá mới không âm** | Giá mới (sau khi tính toán từ bất kỳ phương thức nào) phải ≥ 0. Không cho phép giá âm. | Lỗi: _"Giá không được nhỏ hơn 0."_ |
| **BR-09** | **[Constraint] Bảng giá phải có ít nhất 1 mục** | Khi lưu bảng giá, phải có ít nhất 1 sản phẩm, combo hoặc option đã được thêm vào. | Lỗi: _"Vui lòng thêm ít nhất 1 sản phẩm, combo hoặc tùy chọn vào bảng giá."_ |
| **BR-10** | **[Derivation] Tính giá bán áp dụng** | Khi bán hàng: nếu có bảng giá "Đang diễn ra" (thỏa cả khoảng thời gian + khung giờ nếu có) → giá bán = giá trong bảng giá. Nếu không → giá bán = giá gốc trong danh mục sản phẩm. |
| **BR-11** | **[Derivation] Phương thức tính giá mới** | Hỗ trợ 3 phương thức thiết lập giá trong bảng giá:<br/>1. **Nhập trực tiếp:** Nhập giá bán mới (VNĐ, số nguyên ≥ 0).<br/>2. **Chiết khấu %:** Giảm X% so với giá gốc. `Giá mới = Giá gốc × (1 − X/100)`, làm tròn xuống đến hàng nghìn.<br/>3. **Giảm giá trị cố định:** Giảm Y VNĐ. `Giá mới = Giá gốc − Y`, min = 0. | Nếu chiết khấu > 100%: lỗi _"Chiết khấu không được vượt quá 100%."_<br/>Nếu giảm giá trị > giá gốc: cảnh báo _"Giá sau giảm là 0đ. Bạn có chắc chắn?"_ |
| **BR-12** | **[Derivation] Giá tùy chọn (Option/Add-on)** | Giá tùy chọn trong bảng giá được **ghi đè độc lập** so với giá gốc. Nếu option không có trong bảng giá → giữ nguyên giá gốc. Option thuộc combo: giá option trong bảng giá chỉ áp dụng khi bán lẻ option, không ảnh hưởng giá combo đã set riêng. | — |
| **BR-13** | **[Action Enabler] Sửa bảng giá đang diễn ra** | Bảng giá "Đang diễn ra" chỉ cho phép: thêm/xóa sản phẩm, sửa giá sản phẩm. **Không cho phép** sửa: Tên, Mã, Mô tả, Thời gian áp dụng, Khung giờ, Nhóm KH. | Các trường cơ bản hiển thị ở chế độ readonly. Tooltip: _"Không thể sửa thông tin cơ bản khi bảng giá đang diễn ra."_ |
| **BR-14** | **[Action Enabler] Sao chép bảng giá** | Cho phép sao chép (duplicate) bảng giá từ bất kỳ trạng thái nào. Bản sao: tên thêm hậu tố " (Bản sao)", mã tự sinh mới, trạng thái "Chưa diễn ra", thời gian cần nhập lại, danh sách SP/giá giữ nguyên. | — |
| **BR-15** | **[State Transition] Trạng thái tự động theo thời gian** | Trạng thái bảng giá được hệ thống xác định tự động dựa trên thời gian hiện tại:<br/>- `now < Thời gian bắt đầu` → **Chưa diễn ra**<br/>- `Thời gian bắt đầu ≤ now ≤ Thời gian kết thúc` → **Đang diễn ra**<br/>- `now > Thời gian kết thúc` → **Đã kết thúc**<br/>Người dùng không tự chuyển trạng thái thủ công. | — |
| **BR-16** | **[State Transition] Hủy bảng giá chưa diễn ra** | Bảng giá "Chưa diễn ra" có thể bị **xóa** (xóa hoàn toàn khỏi hệ thống) hoặc sửa thời gian. Bảng giá "Đang diễn ra" có thể được **kết thúc sớm** bằng cách chỉnh Thời gian kết thúc = thời điểm hiện tại (chỉ admin). | Xóa: hiển thị dialog xác nhận. Kết thúc sớm: _"Bảng giá sẽ kết thúc ngay lập tức. Giá bán sẽ trở về giá gốc. Bạn có chắc chắn?"_ |
| **BR-17** | **[State Transition] Đã kết thúc là trạng thái cuối** | Bảng giá "Đã kết thúc" không thể quay lại "Chưa diễn ra" hay "Đang diễn ra". Chỉ cho phép: xem chi tiết, sao chép, xóa. | Các nút sửa bị ẩn/disable. |
| **BR-18** | **[Data Validation] Mô tả tối đa 500 ký tự** | Trường Mô tả là tùy chọn (không bắt buộc), tối đa 500 ký tự Unicode. Hiển thị bộ đếm ký tự. | Lỗi: _"Mô tả tối đa 500 ký tự."_ |
| **BR-19** | **[Data Validation] Khung giờ áp dụng (Happy Hour)** | Tùy chọn (không bắt buộc). Nếu bật: nhập Giờ bắt đầu (HH:mm) và Giờ kết thúc (HH:mm), Giờ kết thúc > Giờ bắt đầu. Chọn các ngày trong tuần áp dụng (checkbox, mặc định tất cả). Bảng giá chỉ có hiệu lực trong khung giờ được chọn, các giờ khác → giá gốc. | Lỗi: _"Giờ kết thúc phải sau giờ bắt đầu."_<br/>Lỗi: _"Vui lòng chọn ít nhất 1 ngày trong tuần."_ |
| **BR-20** | **[Data Validation] Audit Log** | Mọi thao tác tạo, sửa, xóa bảng giá đều ghi vào bảng `price_list_audit_log` với: `price_list_id`, `action` (CREATE/UPDATE/DELETE/DUPLICATE), `actor_id`, `timestamp`, `changes_detail` (JSON diff trước-sau), `ip_address`. | Lỗi ghi log: vẫn commit thao tác chính, ghi cảnh báo system error log. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Các sơ đồ đầy đủ tại `docs/price-list/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/price-list/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** Khi sản phẩm trong bảng giá bị vô hiệu hóa (không xóa), bảng giá có tự động loại SP đó khỏi danh sách không?
- [ ] **OQ-02:** Có giới hạn số lượng sản phẩm/combo/option tối đa trong 1 bảng giá không? (ví dụ: max 500 mục)
- [ ] **OQ-03:** Bảng giá "Đã kết thúc" có tự xóa sau N ngày hay giữ lại vĩnh viễn để tra cứu?
- [ ] **OQ-04:** Khi bảng giá đang diễn ra và giá gốc sản phẩm thay đổi — chiết khấu % có tự cập nhật theo giá gốc mới không?

---

## 6. References

- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- `@../screen/SKILL.md` (Skill Con phụ trách đặc tả màn hình)
- KiotViet — Thiết lập bảng giá: https://support.kiotviet.vn
- Sapo POS — Chính sách giá: https://support.sapo.vn
