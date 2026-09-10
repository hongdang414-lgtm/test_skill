---
feature: open-shift
type: srs-flows
version: 1.0.0
created: 2026-09-08
updated: 2026-09-08
status: draft
authors: [BA Team]
changelog:
  - 2026-09-08 | /srs | cascade từ close-shift: state diagram vòng đời ca về 2 trạng thái — "đang đối soát" là quy trình giao diện
  - 2026-09-08 | /srs | cascade từ OQ-03 resolved: cập nhật Lưu ý tiền mặt Sequence 3
  - 2026-09-08 | /srs | [flows] added 3 sequences, 1 state, 1 flowchart for open-shift
---

# Sơ đồ tương tác: Mở ca (Open Shift)

> BR-00x trong file này là rule của spec Mở ca (`spec.md`); rule của spec Bật/Tắt dẫn dạng đầy đủ `BR-shift-management-NNN`.

## 1. Sequence Diagram — Mở ca thành công (happy path)

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân
    participant POS as POS Client
    participant SRV as Server (nguồn chân lý)
    participant AUD as Audit log

    Note over TN,SRV: Quản lý ca đang BẬT — thu ngân chưa có ca
    TN->>POS: Đăng nhập, vào màn bán hàng
    POS->>SRV: Đối chiếu cấu hình + ca hiện tại (BR-shift-management-015)
    SRV-->>POS: Đang Bật + chưa có ca
    POS-->>TN: Banner "Chưa có ca đang mở" — chặn giao dịch dòng tiền (BR-shift-management-005)
    TN->>POS: Chạm "Mở ca"
    POS-->>TN: Màn Mở ca: cửa hàng, nhân viên, thiết bị, đồng hồ, ô tiền đầu ca
    TN->>POS: Nhập tiền mặt đầu ca (để trống = 0 — BR-006)
    POS->>POS: Kiểm tra tại máy: ≥ 0, trong hạn mức
    TN->>POS: Nhấn "Mở ca" — nút khóa ngay (BR-008)
    POS->>SRV: Yêu cầu tạo ca: nhân viên, cửa hàng, thiết bị, tiền đầu ca
    SRV->>SRV: Kiểm tra: Bật (BR-003) — tài khoản hiệu lực — chưa có ca (BR-002) — tiền hợp lệ (BR-006)
    SRV->>SRV: Tạo ca — mốc mở = giờ server ghi nhận thành công (BR-004)
    SRV->>AUD: Ghi mở ca: người, mốc, thiết bị, tiền đầu ca (BR-013)
    SRV-->>POS: Ca đã mở: mã ca, mốc mở, tiền đầu ca
    POS-->>TN: "Mở ca thành công" + vào bán hàng — giao dịch tiếp theo thuộc ca (BR-010)
```

> **Lưu ý:** mở màn hình lúc 08:00 nhưng nhấn Mở ca lúc 08:10 thì mốc mở là **08:10** — lấy tại server lúc tạo thành công, không lấy thời gian mở form (BR-004).

## 2. Sequence Diagram — Chống trùng ca: hai thiết bị gửi gần như đồng thời (Case 4)

```mermaid
sequenceDiagram
    autonumber
    actor A as Thu ngân A
    participant POSA as POS A
    participant POSB as POS B
    participant SRV as Server

    Note over A,SRV: A đăng nhập trên 2 máy, cả hai cùng gửi yêu cầu mở ca
    A->>POSA: Nhấn "Mở ca"
    A->>POSB: Nhấn "Mở ca" (gần như đồng thời)
    POSA->>SRV: Yêu cầu tạo ca
    POSB->>SRV: Yêu cầu tạo ca
    Note over SRV: Kiểm tra "đã có ca?" + tạo ca là một thao tác nguyên tử (BR-008)
    SRV-->>POSA: Thành công — ca mở lúc 08:10:02
    SRV-->>POSB: Đã có ca — trả thông tin ca hiện tại, KHÔNG tạo ca thứ 2 (BR-002)
    POSA-->>A: Mở ca thành công
    POSB-->>A: "Ca đã được mở 08:10:02 trên POS A" + vào bán hàng
```

## 3. Sequence Diagram — Có ca mở, đăng nhập thiết bị khác (Case 1)

```mermaid
sequenceDiagram
    autonumber
    actor A as Thu ngân A
    participant POSA as POS A
    participant POSB as POS B
    participant SRV as Server

    Note over A,SRV: A đã có ca mở trên POS A từ 07:30 (BR-001: ca gắn người)
    A->>POSB: Đăng nhập POS B, vào màn bán hàng
    POSB->>SRV: Đối chiếu cấu hình + ca của A (BR-shift-management-015)
    SRV-->>POSB: Đang Bật + A đã có ca đang mở (BR-002)
    POSB-->>A: Banner "Ca đang mở từ 07:30 (mở trên POS A)" — không yêu cầu mở ca
    A->>POSB: Bán hàng bình thường (BR-014)
    POSB->>SRV: Ghi nhận giao dịch — tham chiếu ca của A (BR-010)
    SRV-->>POSB: Ghi nhận thành công — giao dịch thuộc ca
    Note over POSA,POSB: Ca không gắn máy: đóng ca có thể thực hiện ở thiết bị bất kỳ
```

> **Lưu ý tiền mặt vật lý:** tiền thu tại POS B nằm ở ngăn kéo POS B trong khi tiền đầu ca đếm tại POS A — không giới hạn thanh toán (OQ-03); đếm tiền nhiều máy xử lý ở use case Đối soát.

## 4. State Diagram — Vòng đời ca

```mermaid
stateDiagram-v2
    direction LR
    state "ĐANG MỞ — nhận giao dịch dòng tiền của người giữ ca (BR-010)" as Mo
    state "ĐÃ ĐÓNG — chỉ đọc: lịch sử, báo cáo" as Done

    [*] --> Mo: Mở ca thành công — mốc server (BR-004)
    Mo --> Done: Hoàn tất đóng ca — server chốt thành công (use case Đóng ca)
    Done --> [*]
```

> **Lưu ý:** không có trạng thái "nháp"/"hủy" — mở ca thất bại (lỗi, timeout, mất kết nối) đồng nghĩa ca **không tồn tại** (BR-011). Quá trình nhập đối soát không làm ca rời trạng thái Đang mở — "đang đóng dở" là trạng thái quy trình trên màn hình, không phải trạng thái dữ liệu (chi tiết use case Đóng ca). Ca đã đóng không được mở lại; muốn làm tiếp thì mở ca mới.

## 5. Flowchart — Quyết định trước khi tạo ca (tại server)

```mermaid
flowchart TD
    A[Nhận yêu cầu mở ca] --> B{Thiết bị kết nối được server? BR-012}
    B -->|Mất kết nối| X1[Chặn — không mở ca offline, giữ dữ liệu form chờ thử lại]
    B -->|Có| C{Quản lý ca đang Bật? BR-003}
    C -->|Tắt| X2[Từ chối — không cần ca, vào bán hàng bình thường]
    C -->|Bật| D{Tài khoản còn hiệu lực?}
    D -->|Bị khóa| X3[Từ chối — thông báo liên hệ quản lý]
    D -->|Có| E{Người này đã có ca đang mở hoặc đang đóng dở? BR-002}
    E -->|Có| F[Không tạo ca mới — trả thông tin ca hiện tại BR-008]
    E -->|Chưa| G{Tiền đầu ca hợp lệ: không âm, trong hạn mức? BR-006}
    G -->|Không| X4[Từ chối — báo lỗi dữ liệu tiền, sửa tại chỗ]
    G -->|Hợp lệ| H[Tạo ca — ghi người, mốc server, thiết bị, tiền đầu ca BR-004 BR-013]
    H --> I[Thành công — giao dịch tiếp theo thuộc ca BR-010]
```
