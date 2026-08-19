---
feature: cash-drawer
type: srs-flows
updated: 2026-08-12
status: draft
authors: [BA Team]
---

# Sơ đồ tương tác: Cash Drawer (Kết nối két tiền & Tự động mở khi thanh toán tiền mặt)

## 1. Sequence Diagram — Mở két tự động khi xác nhận thanh toán tiền mặt

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân
    participant POS as Hệ thống POS
    participant KET as Két tiền (thiết bị)
    participant LOG as Lịch sử mở két

    Note over TN,LOG: HAPPY PATH — thanh toán tiền mặt
    TN->>POS: Chọn "Tiền mặt", nhập số tiền khách đưa, bấm "Xác nhận thanh toán"
    POS->>POS: Kiểm tra đã cấu hình két & đang kích hoạt?

    alt Đã cấu hình & đang kích hoạt
        POS->>KET: Gửi lệnh mở két
        alt Két mở thành công
            KET-->>POS: Xác nhận đã mở
            POS->>LOG: Ghi: tự động | thu ngân | mã đơn | thành công
            POS-->>TN: Hoàn tất giao dịch, hiển thị tiền thừa trả khách
        else Mất kết nối / lỗi thiết bị
            KET--xPOS: Timeout / lỗi
            POS->>LOG: Ghi: tự động | thu ngân | mã đơn | thất bại (lỗi kết nối)
            POS-->>TN: Vẫn hoàn tất giao dịch + cảnh báo "Két không mở được, mở thủ công"
            TN->>POS: Bấm "Mở két" (thủ công) → thử lại
        end
    else Chưa cấu hình / không kích hoạt
        POS->>LOG: Ghi: tự động | thu ngân | mã đơn | thất bại (chưa cấu hình)
        POS-->>TN: Hoàn tất giao dịch + thông báo "Chưa kết nối két, mở thủ công nếu cần"
    end

    Note over TN: Đếm tiền thừa trả khách → đóng két
```

## 2. Sequence Diagram — Mở két thủ công

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân
    participant POS as Hệ thống POS
    participant KET as Két tiền (thiết bị)
    participant LOG as Lịch sử mở két

    TN->>POS: Bấm "Mở két"
    POS->>POS: Kiểm tra thu ngân đã đăng nhập & quyền mở két
    POS-->>TN: Hiện dialog chọn lý do
    TN->>POS: Chọn lý do (Đổi lẻ / Nạp tiền / Kiểm tra / Khác)
    POS->>KET: Gửi lệnh mở

    alt Mở thành công
        KET-->>POS: Đã mở
        POS->>LOG: Ghi: thủ công | thu ngân | lý do | thành công
        POS-->>TN: Đã mở két
    else Lỗi mở
        KET--xPOS: Lỗi
        POS->>LOG: Ghi: thủ công | thu ngân | lý do | thất bại
        POS-->>TN: Cảnh báo + cho thử lại
    end
```

## 3. State Diagram — Trạng thái kết nối két tiền

```mermaid
stateDiagram-v2
    state "Chưa cấu hình" as ChuaCauHinh
    state "Đang kết nối (kích hoạt)" as DangKetNoi
    state "Mất kết nối" as MatKetNoi

    [*] --> ChuaCauHinh
    ChuaCauHinh --> DangKetNoi: Quản lý cấu hình + bật kích hoạt
    DangKetNoi --> MatKetNoi: Thiết bị không phản hồi (timeout/lỗi)
    MatKetNoi --> DangKetNoi: Thiết bị phản hồi lại / thử mở thành công
    DangKetNoi --> ChuaCauHinh: Quản lý tắt kích hoạt / xóa cấu hình
    MatKetNoi --> ChuaCauHinh: Quản lý cấu hình lại
```

> **Lưu ý:** Nhánh "két đã mở sẵn" (BR-12) và "xác nhận mở thành công" phụ thuộc việc thiết bị có cảm biến trạng thái — xem OQ-01. Nếu két không có cảm biến, hệ thống ghi nhận kết quả "best-effort" (chỉ ghi kết quả gửi lệnh).
