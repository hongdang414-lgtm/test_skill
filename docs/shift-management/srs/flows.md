---
feature: shift-management
type: srs-flows
updated: 2026-09-08
status: draft
authors: [BA Team]
---

# Sơ đồ tương tác: Shift Management (Bật/Tắt Quản lý ca)

## 1. Sequence Diagram — Bật cấu hình, lan tỏa đa thiết bị (đơn đang tạo dở ở POS B)

```mermaid
sequenceDiagram
    autonumber
    actor QL as Quản lý (POS A)
    participant CFG as Cấu hình cửa hàng (server)
    participant AUD as Audit log
    actor TN as Thu ngân (POS B)
    participant POSB as POS B - màn bán hàng

    Note over QL,POSB: Trước mốc: Quản lý ca = TẮT — POS B bán tự do, đơn đang tạo dở
    TN->>POSB: Đang tạo đơn (giỏ 5 món, chưa thanh toán)
    QL->>CFG: Bật công tắc + Xác nhận
    CFG->>CFG: Lưu thành công — mốc hiệu lực M (BR-007)
    CFG->>AUD: Ghi: người đổi, mốc M, Tắt sang Bật, thiết bị (BR-018)
    CFG-->>QL: Đã bật — hiệu lực ngay
    CFG->>POSB: Đẩy cấu hình mới theo thời gian thực (BR-015)
    POSB-->>TN: Banner trạng thái ca: "Chưa có ca đang mở"

    TN->>POSB: Nhấn Thanh toán
    POSB->>CFG: Đối chiếu cấu hình + ca trước khi ghi nhận (BR-015)
    alt Chưa có ca mở
        CFG-->>POSB: Chặn — yêu cầu mở ca (BR-005)
        POSB-->>TN: Giữ nguyên giỏ hàng, hiện yêu cầu Mở ca (BR-013)
        TN->>CFG: Mở ca — thủ công, hệ thống không tự mở (BR-005)
        CFG-->>POSB: Ca đang mở
        POSB->>CFG: Ghi nhận thanh toán — thuộc ca của thu ngân (BR-006)
        POSB-->>TN: Thanh toán thành công
    else Đã có ca mở
        POSB->>CFG: Ghi nhận thanh toán — thuộc ca (BR-006)
    end

    Note over CFG,POSB: Giao dịch phát sinh trước mốc M: KHÔNG thuộc ca, phân loại "ngoài ca" (BR-008)
```

> **Lưu ý:** Nếu realtime chưa kịp tới POS B thì bước "Đối chiếu trước khi ghi nhận" vẫn chặn tại server — lớp đối chiếu là bảo đảm cuối, realtime chỉ để trải nghiệm (BR-015).

## 2. Sequence Diagram — Tắt khi còn ca mở: chặn + đóng ca hộ (Hướng 1)

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân (người giữ ca)
    participant CA as Ca của cửa hàng
    actor QL as Quản lý (POS A)
    participant CFG as Cấu hình cửa hàng (server)

    Note over TN,CFG: Đang BẬT — ca mở từ 07:30, chưa đóng
    TN->>CA: Còn ca đang mở
    QL->>CFG: Tắt công tắc
    CFG->>CA: Kiểm tra số ca đang mở của cửa hàng
    alt Còn từ 1 ca mở (kể cả đang đóng dở)
        CFG-->>QL: CHẶN — "Không thể tắt Quản lý ca khi vẫn còn N ca đang mở" + danh sách ca (BR-009)
        QL->>CA: Đóng ca hộ — bắt buộc ghi lý do (BR-009)
        CA-->>TN: Thông báo "Ca được đóng hộ" kèm lý do
        QL->>CFG: Tắt lại công tắc
        CFG-->>QL: Tắt thành công — mốc M2 (BR-010)
    else Đã đóng hết ca
        CFG-->>QL: Tắt thành công — mốc M2 (BR-010)
    end

    Note over QL,CA: Sau mốc M2: không yêu cầu ca với giao dịch mới; lịch sử ca cũ vẫn xem được (BR-011)
```

> **Lưu ý:** Ca "đang đóng dở" (đang đối soát) vẫn tính là đang mở — phải hoàn tất đóng ca rồi mới tắt được (Edge case 5 trong spec).

## 3. State Diagram — Vòng đời cấu hình Bật/Tắt

```mermaid
stateDiagram-v2
    direction LR
    state "TẮT — bán hàng tự do, giao dịch không gắn ca" as OFF
    state "BẬT — giao dịch dòng tiền phải thuộc ca đang mở" as ON

    [*] --> OFF: Khởi tạo cửa hàng — mặc định Tắt (BR-001)
    OFF --> ON: Quản lý bật + xác nhận — hiệu lực ngay tại mốc M1 (BR-007)
    ON --> OFF: Quản lý tắt — chỉ khi đã đóng hết ca (BR-009, BR-010)
    ON --> ON: Thao tác tắt bị chặn khi còn ca mở (BR-009)
```

> **Phân thuộc giao dịch theo mốc:** trước M1 mọi giao dịch ngoài ca; từ M1 giao dịch dòng tiền phải thuộc ca (BR-006, BR-008). Bật lại sau thời gian tắt bắt đầu hoàn toàn mới, không hồi sinh ca cũ (BR-012). Dữ liệu ca phát sinh ở giai đoạn BẬT trước đó giữ nguyên khi về TẮT (BR-011).

## 4. Flowchart — Quyết định giao dịch có thuộc ca không (tại thời điểm ghi nhận)

```mermaid
flowchart TD
    A[Giao dịch phát sinh dòng tiền tại quầy] --> B{Thiết bị đã nhận cấu hình mới nhất?}
    B -->|Chưa nhận — offline hoặc realtime trễ| C[Áp dụng cấu hình đã lưu trong máy — BR-016]
    B -->|Đã nhận| D{Cấu hình cửa hàng đang BẬT?}
    D -->|TẮT| E[Ghi nhận — KHÔNG gắn ca]
    D -->|BẬT| F{Người thực hiện có ca đang mở?}
    F -->|Có| G[Ghi nhận — thuộc ca đó — BR-006]
    F -->|Chưa| H[CHẶN — yêu cầu mở ca — BR-005]
    C --> I{Khi đồng bộ — cấu hình server đã BẬT?}
    I -->|Vẫn TẮT| E
    I -->|Đã BẬT| J[Giữ nguyên — đánh dấu ngoài ca — BR-016 và BR-008]
```

> **Lưu ý:** Quyết định thuộc ca hay không lấy trạng thái cấu hình **tại thời điểm server ghi nhận** giao dịch — không phải lúc mở màn hình hay lúc bắt đầu tạo chứng từ (BR-014). Nhánh J không tự gán vào ca đang mở khi đồng bộ — tránh gán hồi tố (BR-008).
