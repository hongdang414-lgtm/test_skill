---
feature: close-shift
type: srs-flows
version: 1.0.0
created: 2026-09-08
updated: 2026-09-08
status: draft
authors: [BA Team]
changelog:
  - 2026-09-08 | /srs | cascade từ OQ-05 resolved: lưu ý Sequence 2 bổ sung in-app notification khi quay lại
  - 2026-09-08 | /srs | [flows] added 4 sequences, 1 state, 1 flowchart for close-shift
---

# Sơ đồ tương tác: Đóng ca (Close Shift)

> BR-00x trong file này là rule của spec Đóng ca (`spec.md`); rule của spec Bật/Tắt và spec Mở ca dẫn dạng đầy đủ `BR-shift-management-NNN`, `BR-open-shift-NNN`.

## 1. Sequence Diagram — Đóng ca thành công (happy path)

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân
    participant POS as POS Client
    participant SRV as Server (nguồn chân lý)
    participant AUD as Audit log

    Note over TN,SRV: Quản lý ca đang Bật — ca #23 mở từ 08:10, thu ngân kết thúc ca
    TN->>POS: Chạm "Đóng ca" trên banner ca đang mở
    POS->>SRV: Lấy tổng kết ca (snapshot + mốc snapshot)
    SRV-->>POS: Đầu ca 500.000 · thu vào 4.512.000 · chi ra 350.000 · kỳ vọng 4.662.000 · 42 giao dịch
    POS-->>TN: Màn Đóng ca: thông tin ca + tổng kết + ô "Tiền mặt đếm được"
    TN->>POS: Đếm tiền két — nhập 4.674.000
    POS->>POS: Kiểm tra tại máy (BR-007) — tính chênh lệch +12.000 thừa (BR-005)
    TN->>POS: Nhấn "Hoàn tất đóng ca" — nút khóa ngay (BR-010)
    POS->>SRV: Chốt: mã ca · tiền đếm · mốc snapshot người dùng đã xem
    SRV->>SRV: Validate lại: còn mở — quyền — tiền — snapshot không đổi (BR-003, BR-007, BR-009)
    SRV->>SRV: Chốt: Đã đóng — mốc đóng = giờ server (BR-001, BR-008) — chênh lệch theo snapshot mốc chốt (BR-005)
    SRV->>AUD: Ghi: mốc · người đóng · tiền cuối ca · chênh lệch · thiết bị (BR-014)
    SRV-->>POS: Ca đã đóng #23 · 17:42
    POS-->>TN: "Đã đóng ca" + chênh lệch — banner "Chưa có ca đang mở" — giao dịch tiếp theo yêu cầu mở ca mới (BR-013)
```

> **Lưu ý:** vào màn đối soát lúc 17:30 nhưng Hoàn tất lúc 17:42 thì mốc đóng là **17:42** — lấy tại server lúc chốt thành công (BR-008), đối xứng với BR-open-shift-004 của mở ca.

## 2. Sequence Diagram — Đóng ca hộ (quản lý, từ dialog chặn tắt)

```mermaid
sequenceDiagram
    autonumber
    actor QL as Quản lý (POS A)
    participant POSA as POS A
    participant SRV as Server (nguồn chân lý)
    actor TN as Thu ngân A (POS B — đang online)
    participant POSB as POS B
    participant AUD as Audit log

    Note over QL,POSB: Ca #23 của thu ngân A còn mở — cuối ngày quản lý cần Tắt Quản lý ca
    QL->>POSA: Tắt công tắc Quản lý ca
    POSA->>SRV: Yêu cầu tắt
    SRV-->>POSA: CHẶN — còn 1 ca mở + danh sách ca (BR-shift-management-009)
    QL->>POSA: Chạm "Đóng ca" trên dòng ca #23 (của thu ngân A)
    POSA->>SRV: Lấy tổng kết ca — chế độ đóng ca hộ
    POSA-->>QL: Màn Đóng ca: nhãn "Đóng ca hộ thu ngân A" + ô Lý do bắt buộc (BR-004)
    QL->>POSA: Đếm tiền két, nhập tiền đếm + lý do "A nghỉ gấp — chốt hộ cuối ngày"
    QL->>POSA: Hoàn tất đóng ca
    POSA->>SRV: Chốt: mã ca · tiền đếm · lý do đóng hộ
    SRV->>SRV: Validate: quyền đóng hộ — lý do — tiền — snapshot (BR-003, BR-004, BR-007)
    SRV->>SRV: Chốt: Đã đóng — mốc server (BR-001, BR-008)
    SRV->>AUD: Ghi: người đóng hộ + lý do + người giữ ca + tiền + chênh lệch (BR-014)
    SRV-->>POSA: Ca đã đóng #23 · 21:05
    SRV-->>POSB: Realtime: "Ca của bạn đã được đóng hộ 21:05 bởi QL — lý do: ..." (BR-004)
    POSB-->>TN: Banner ca biến mất — giao dịch tiếp theo yêu cầu mở ca mới (BR-013)
    QL->>POSA: Tắt lại Quản lý ca — thành công vì không còn ca mở (BR-shift-management-009)
```

> **Lưu ý:** quản lý đếm tiền thực tế thay và chịu trách nhiệm số liệu nhập; người giữ ca xem được lý do trong chi tiết ca (Mục 6.3 spec). Nếu thu ngân không online: nhận in-app notification khi quay lại, kèm bản ghi trong chi tiết ca — không push notification hệ điều hành GĐ1 (OQ-05 resolved).

## 3. Sequence Diagram — Có giao dịch mới trong lúc đang đối soát (BR-009)

```mermaid
sequenceDiagram
    autonumber
    actor A as Thu ngân A — người giữ ca
    participant POSA as POS A (màn Đóng ca)
    participant POSB as POS B (màn bán hàng)
    participant SRV as Server (nguồn chân lý)

    Note over A,SRV: Ca #23 đang mở — A vào màn đối soát nhưng vẫn phụ khách ở máy khác
    A->>POSA: Chạm Đóng ca
    POSA->>SRV: Lấy tổng kết ca
    SRV-->>POSA: Snapshot S1 — mốc T1: kỳ vọng 4.662.000 · 42 giao dịch
    A->>POSB: Phụ khách: thanh toán tiền mặt 150.000
    POSB->>SRV: Ghi nhận — ca vẫn Đang mở nên giao dịch thuộc ca #23 (BR-open-shift-010)
    SRV-->>POSB: Ghi nhận thành công
    A->>POSA: Đếm tiền theo S1: nhập 4.674.000 — thấy chênh lệch +12.000
    A->>POSA: Nhấn Hoàn tất
    POSA->>SRV: Chốt: tiền đếm 4.674.000 · mốc snapshot T1
    SRV->>SRV: Tính lại snapshot: có 1 giao dịch mới sau T1 (BR-009)
    SRV-->>POSA: KHÔNG chốt — trả S2: kỳ vọng 4.812.000 + cảnh báo "1 giao dịch mới"
    POSA-->>A: "Ca có 1 giao dịch mới (+150.000 tiền mặt) — số liệu đã làm mới, kiểm tra lại tiền"
    A->>POSA: Đếm lại tiền: 4.824.000 — chênh lệch +12.000
    A->>POSA: Xác nhận Hoàn tất lần nữa (mốc snapshot T2)
    POSA->>SRV: Chốt: tiền đếm 4.824.000 · mốc T2
    SRV->>SRV: Snapshot khớp — chốt Đã đóng, tính chênh lệch +12.000 (BR-001, BR-005)
    SRV-->>POSA: Ca đã đóng #23 · 17:58 — chênh lệch +12.000 (thừa)
```

> **Lưu ý:** nếu không có cơ chế này, server chốt theo snapshot mới (kỳ vọng 4.812.000) trong khi người dùng nhập số đếm theo snapshot cũ (4.674.000) — chênh lệch hiển thị −138.000 **thiếu** một cách sai lệch. Cơ chế không ép đếm lại; nó đảm bảo người đóng xác nhận trên số liệu mới nhất trước khi chốt (Mục 8.2 spec).

## 4. Sequence Diagram — Hai yêu cầu hoàn tất gần đồng thời (BR-010)

```mermaid
sequenceDiagram
    autonumber
    actor A as Thu ngân A (POS A — tự đóng)
    actor QL as Quản lý (POS B — đóng ca hộ)
    participant SRV as Server (nguồn chân lý)

    Note over A,QL: Cả hai cùng thao tác trên ca #23 gần như đồng thời
    A->>SRV: Hoàn tất — tự đóng
    QL->>SRV: Hoàn tất — đóng hộ + lý do
    Note over SRV: Kiểm tra "ca còn Đang mở?" + chốt là một thao tác nguyên tử (BR-010)
    SRV-->>A: THÀNH CÔNG — ca đã đóng 17:42:05 bởi A
    SRV-->>QL: Ca ĐÃ ĐÓNG — trả kết quả, không chốt lần 2, không báo lỗi
    QL-->>QL: "Ca đã được đóng 17:42:05 bởi A" — form tự đóng
```

> **Lưu ý:** request thua cuộc không phải lỗi — hiển thị kết quả ca đã đóng kèm người đóng và mốc (BR-010, Mục 7.2 Case 2 spec).

## 5. State Diagram — Vòng đời ca

```mermaid
stateDiagram-v2
    direction LR
    state "ĐANG MỞ — nhận giao dịch dòng tiền của người giữ ca (BR-open-shift-010)" as Mo
    state "ĐÃ ĐÓNG — chỉ đọc: xem lịch sử, báo cáo; không mở lại (BR-013)" as Done

    [*] --> Mo: Mở ca thành công — mốc server (use case Mở ca)
    Mo --> Done: Hoàn tất đóng ca — server chốt thành công (BR-001, BR-008)
    Done --> [*]
```

> **Lưu ý:** quá trình nhập đối soát **không** làm ca rời trạng thái Đang mở (BR-002) — "đang đối soát" là trạng thái quy trình trên màn hình, không phải trạng thái dữ liệu của ca. Thoát giữa chừng: ca vẫn Đang mở. Không có trạng thái nháp/hủy/đóng dở; ca đã đóng không được mở lại — muốn làm tiếp thì mở ca mới.

## 6. Flowchart — Quyết định Hoàn tất đóng ca (tại server)

```mermaid
flowchart TD
    A[Nhận yêu cầu Hoàn tất đóng ca] --> B{Ca tồn tại?}
    B -->|Không| X1[Lỗi không tìm thấy ca]
    B -->|Có| C{Ca còn Đang mở? — kiểm tra + chốt nguyên tử BR-010}
    C -->|Đã đóng| X2[Trả kết quả ca đã đóng — idempotent, không lỗi]
    C -->|Còn mở| D{Người yêu cầu là người giữ ca hoặc có quyền đóng hộ? BR-003 BR-004}
    D -->|Không| X3[Từ chối 403 — ghi cảnh báo an ninh audit]
    D -->|Có| E{Đóng hộ: có lý do? BR-004}
    E -->|Thiếu| X4[Từ chối — yêu cầu nhập lý do]
    E -->|Đủ hoặc tự đóng| F{Tiền đếm hợp lệ: số, ≥ 0, trong hạn mức? BR-007}
    F -->|Không| X5[Từ chối — lỗi dữ liệu tiền]
    F -->|Hợp lệ| G{Snapshot hiện tại khớp mốc người dùng đã xem? BR-009}
    G -->|Không — có giao dịch mới| X6[KHÔNG chốt — trả snapshot mới + cảnh báo, chờ xác nhận lại]
    G -->|Khớp| H[Chốt: Đã đóng — mốc server BR-001 BR-008 — tính chênh lệch BR-005 — audit BR-014]
    H --> I[Ca chỉ đọc — giao dịch tiếp theo của người giữ ca yêu cầu mở ca mới BR-013]
```

> **Lưu ý:** hoàn tất không kiểm tra cấu hình Bật/Tắt (BR-012) — ca đang mở luôn đóng được; Tắt Quản lý ca chỉ thành công khi không còn ca mở (BR-shift-management-009) nên hai thao tác không xung đột.
