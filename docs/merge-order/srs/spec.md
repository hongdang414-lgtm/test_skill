---
feature: merge-order
type: srs-spec
version: 1.0.0
updated: 2026-07-23
status: draft
authors: [BA Team]
changelog:
  - 2026-07-23 | /srs-spec | [spec] initialized merge-order specification, 20 business rules
  - 2026-07-23 | /activity | [flows] added merge-order-main-flow activity, 5 decisions
  - 2026-07-23 | /sequence | [flows] added merge-order-system-sequence sequence, 7 actors
  - 2026-07-23 | /screen   | [screens] added specification for 5 screens
  - 2026-07-23 | /screen   | [screens] updated specification and navigation for all 5 screens — migrated to 5-column format
---

# Đặc tả tính năng: Ghép đơn hàng (Merge Order)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép nhân viên thu ngân/quản lý cửa hàng **ghép nội dung một đơn hàng gốc vào một đơn hàng đích đang hoạt động**, biến 2 đơn thành 1. Sau ghép: đơn đích chứa toàn bộ sản phẩm của cả 2 đơn và được tính lại tài chính; đơn gốc đánh dấu `MERGED`. Phạm vi: POS F&B (đơn tại bàn, mang đi). Không áp dụng cho đơn đã thanh toán, đã hủy, đang giao, hoặc đã ghép. |
| **2. Actors (Tác nhân)** | **Chính:** Thu ngân (Cashier), Quản lý cửa hàng (Store Manager).<br/>**Hỗ trợ:** POS Backend, Finance Engine, KDS, Audit Log. |
| **3. Pre-conditions** | 1. Đơn gốc tồn tại, trạng thái `ACTIVE`.<br/>2. Tồn tại ≥1 đơn đích khác cũng `ACTIVE`.<br/>3. Người dùng có quyền `MERGE_ORDER`.<br/>4. Không có partial payment pending trên cả 2 đơn. |
| **4. Expected Results** | **Happy Path:**<br/>1. Thu ngân mở chi tiết đơn gốc `ACTIVE`.<br/>2. Nhấn "Ghép đơn" → hệ thống hiển thị DS đơn có thể ghép.<br/>3. Chọn đơn đích.<br/>4. Hệ thống kiểm tra điều kiện, hiển thị preview tài chính đơn đích sau ghép.<br/>5. Xác nhận → chuyển dữ liệu, tính lại đơn đích, đánh dấu đơn gốc `MERGED`, đồng bộ KDS.<br/>6. Mở chi tiết đơn đích sau ghép.<br/><br/>**Alternate Branches:**<br/>- **[A - Không có đơn đích]:** Thông báo "Không tìm thấy đơn đủ điều kiện."<br/>- **[B - Khác bàn]:** Cảnh báo, yêu cầu xác nhận bổ sung.<br/>- **[C - Giảm giá gốc bị hủy]:** Cảnh báo trước khi xác nhận.<br/>- **[D - Lỗi hệ thống]:** Rollback, 2 đơn giữ nguyên. |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **[Constraint] Điều kiện kích hoạt** | Đơn gốc phải `ACTIVE`. Đơn đã thanh toán, hủy, đang giao, hoặc `MERGED` không được ghép. | Ẩn/disable nút. API: `422 ORDER_NOT_MERGEABLE`. |
| **BR-02** | **[Constraint] Đơn đích phải ACTIVE** | Đơn đích bắt buộc `ACTIVE`. DS chỉ hiện đơn thỏa điều kiện. | Đơn không ACTIVE không xuất hiện. Race condition: `422 TARGET_ORDER_INVALID`. |
| **BR-03** | **[Constraint] Phân quyền** | Chỉ tài khoản có quyền `MERGE_ORDER` mới thực hiện được. | Ẩn nút. API: `403 PERMISSION_DENIED`. |
| **BR-04** | **[Constraint] Không ghép chính nó** | Đơn gốc không được chọn chính mình làm đơn đích. | Loại khỏi DS. |
| **BR-05** | **[Constraint] Không ghép khi có partial payment** | Nếu đơn gốc HOẶC đơn đích có `PAYMENT_PENDING`: không cho phép. | Thông báo: _"Không thể ghép khi có giao dịch đang xử lý."_ |
| **BR-06** | **[Constraint] Combo chuyển nguyên khối** | Combo từ đơn gốc chuyển **nguyên bộ** sang đơn đích. | Không cần chọn lẻ. |
| **BR-07** | **[Constraint] Đơn MERGED không ghép tiếp** | Đơn `MERGED` không làm gốc/đích cho lần ghép sau. | Loại khỏi DS. API: `422 ORDER_ALREADY_MERGED`. |
| **BR-08** | **[Constraint] Giới hạn 50 line items** | Tổng dòng SP sau ghép ≤ 50 (hiệu năng KDS + in bill). | Lỗi: _"Vượt giới hạn ({N}/50)."_ |
| **BR-09** | **[Derivation] Giảm giá item-level** | Giảm giá gắn vào SP đi theo SP sang đơn đích. Giữ nguyên giá trị. | Nếu giảm giá > giá SP: cảnh báo audit log. |
| **BR-10** | **[Derivation] Giảm giá order-level đơn gốc bị hủy** | Giảm giá cấp đơn (%, VNĐ) của đơn gốc **bị hủy bỏ** khi ghép. | Cảnh báo cashier: _"Giảm giá [TÊN] sẽ bị hủy sau ghép."_ |
| **BR-11** | **[Derivation] Giữ giảm giá đơn đích** | Giảm giá đơn đích giữ nguyên, áp trên tổng mới. % → tính lại; VNĐ → giữ nguyên. | — |
| **BR-12** | **[Derivation] Tính lại tổng tiền đơn đích** | Công thức:<br/>`Tạm tính = Σ(Đơn giá × SL)` (cả gốc + đích)<br/>`Thuế = (Tạm tính − GG_SP − GG_đơn) × Thuế suất`<br/>`Phí DV = (Tạm tính − GG_SP − GG_đơn) × %DV`<br/>`Cần TT = Tạm tính − GG_SP − GG_đơn + Thuế + Phí DV` | Nếu kết quả âm: min = 0, cảnh báo audit log. |
| **BR-13** | **[Derivation] Gộp SP trùng** | Cùng SKU + cùng topping → cộng dồn SL. Khác topping → dòng riêng. | SL > 999: lỗi `QUANTITY_OVERFLOW`. |
| **BR-14** | **[State Transition] Đơn gốc → MERGED** | Đơn gốc → `MERGED`. Không thể: thanh toán, sửa, tách, ghép tiếp. Lưu `merged_into_order_id`. | Rollback nếu thất bại. |
| **BR-15** | **[State Transition] Đơn đích giữ ACTIVE** | Đơn đích giữ `ACTIVE`, cập nhật items + totals + `merged_from_order_ids[]`. | Rollback nếu thất bại. |
| **BR-16** | **[State Transition] KDS sau ghép** | Món `DONE` → giữ nguyên, chuyển mapping. Món `IN_PROGRESS`/`PENDING` → đồng bộ lại. | KDS lỗi: ghép vẫn thành công, cảnh báo audit log. |
| **BR-17** | **[State Transition] Giải phóng bàn** | Ghép khác bàn: bàn đơn gốc giải phóng nếu không còn đơn ACTIVE khác. | Còn đơn ACTIVE khác: chỉ xóa mapping đơn gốc. |
| **BR-18** | **[Action Enabler] Cảnh báo khác bàn** | Khi 2 bàn khác nhau: cảnh báo _"Đơn gốc (Bàn {X}) và đơn đích (Bàn {Y}) khác bàn."_ | Yêu cầu xác nhận bổ sung. |
| **BR-19** | **[Data Validation] Audit Log** | Ghi: `source_order_id`, `target_order_id`, `actor_id`, `timestamp`, `items_transferred[]`, `financial_before`, `financial_after`. | Lỗi ghi log: vẫn commit, cảnh báo system log. |
| **BR-20** | **[Data Validation] Tồn kho không đổi** | Ghép không thay đổi tồn kho. Chỉ cập nhật mapping đơn-SP. | Sai lệch: cảnh báo audit log, vẫn cho ghép. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Các sơ đồ đầy đủ tại `docs/merge-order/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/merge-order/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [ ] **OQ-01:** Đơn con đã tách (`parent_order_id` != null) có được ghép?
- [ ] **OQ-02:** Voucher/coupon bind vào mã đơn gốc bị hủy hay chuyển sang đích?
- [ ] **OQ-03:** Giới hạn số lần ghép vào 1 đơn đích? (ví dụ: max 5)
- [ ] **OQ-04:** Hoàn tác ghép bằng "Tách đơn" hay cần Undo riêng?
- [ ] **OQ-05:** Đơn đích sau ghép có badge "Đã nhận ghép"?

---

## 6. References

- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- Tính năng liên quan: [Split Order](../split-order/srs/spec.md)
- KiotViet: https://support.kiotviet.vn
- Sapo POS: https://support.sapo.vn
