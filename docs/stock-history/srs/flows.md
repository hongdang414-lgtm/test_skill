---
feature: stock-history
type: srs-flows
updated: 2026-08-25
status: draft
authors: [BA Team]
---

# Sơ đồ tương tác: Stock History (Lịch sử thay đổi tồn kho)

## 1. Sequence Diagram — Bán rồi hủy đơn trước thanh toán (đảo ngược, không xóa)

```mermaid
sequenceDiagram
    autonumber
    actor TN as Thu ngân
    participant POS as Hệ thống POS
    participant TON as Tồn kho hiện tại
    participant SO as Sổ lịch sử tồn

    Note over TN,SO: HAPPY PATH — tồn ban đầu 20
    TN->>POS: Tạo đơn bán 2 sản phẩm
    POS->>TON: Trừ tồn từng dòng (2)
    TON-->>POS: Tồn mới = 18
    POS->>SO: Ghi bản ghi: Bán hàng | trước 20 | −2 | sau 18 | #ĐH | thu ngân
    POS-->>TN: Đơn tạo thành công

    Note over TN: Khách đổi ý — hủy trước thanh toán
    TN->>POS: Hủy đơn
    POS->>TON: Hoàn tồn từng dòng (2)
    TON-->>POS: Tồn mới = 20
    POS->>SO: Ghi bản ghi: Hủy đơn trước thanh toán | trước 18 | +2 | sau 20 | cùng #ĐH
    POS-->>TN: Đã hủy đơn

    Note over SO: 2 bản ghi cùng tồn tại — bản ghi bán KHÔNG bị xóa/sửa (BR-06)
```

## 2. Sequence Diagram — Điều chỉnh trực tiếp trên Sửa sản phẩm (kèm xung đột thiết bị khác)

```mermaid
sequenceDiagram
    autonumber
    actor QL as Quản lý cửa hàng
    participant POS as Hệ thống POS
    participant QB as Quầy B (thiết bị khác)
    participant TON as Tồn kho hiện tại
    participant SO as Sổ lịch sử tồn

    QL->>POS: Mở Sửa sản phẩm — màn hình hiển thị tồn 10
    Note over QB,TON: Trong lúc đó, quầy B bán 2
    QB->>TON: Trừ tồn (2)
    TON-->>SO: Ghi: Bán hàng | trước 10 | −2 | sau 8
    Note over TON: Tồn thật hiện tại = 8

    QL->>POS: Sửa số lượng 10 → 15, bấm Lưu
    POS->>POS: Số mới ≠ số cũ → cần ghi điều chỉnh (BR-11)
    POS->>POS: Tồn hiện tại (8) ≠ giá trị lúc mở màn hình (10)?

    alt Có xung đột
        POS-->>QL: Cảnh báo: "Tồn đã thay đổi từ 10 thành 8 (bán tại quầy B). Áp dụng 15?"
        alt Xác nhận áp dụng 15
            POS->>TON: Đặt tồn = 15
            POS->>SO: Ghi: Điều chỉnh tồn kho | trước 8 | +7 | sau 15 | lý do
            POS-->>QL: Lưu thành công
        else Hủy thao tác
            POS-->>QL: Không ghi gì — màn hình làm mới tồn 8 (BR-15)
        end
    else Không xung đột
        POS->>TON: Đặt tồn = 15
        POS->>SO: Ghi: Điều chỉnh tồn kho | trước 10 | +5 | sau 15 | lý do (BR-12)
        POS-->>QL: Lưu thành công
    end

    Note over QL,SO: Nếu nhập lại đúng 10 → không sinh bản ghi nào (BR-11)
```

## 3. State Diagram — Vòng đời phiếu chuyển kho & ảnh hưởng tồn hai kho

```mermaid
stateDiagram-v2
    state "Nháp" as Nhap
    state "Đã xuất (hàng đang chuyển)" as DaXuat
    state "Đã nhận" as DaNhan
    state "Hủy từ nháp (không ghi gì)" as HuyNhap
    state "Hủy khi đang chuyển (nhập lại kho đi)" as HuySauXuat

    [*] --> Nhap
    Nhap --> DaXuat: Xác nhận xuất — kho đi: trừ, ghi "Chuyển kho - xuất"
    Nhap --> HuyNhap: Hủy khi còn nháp — tồn chưa đổi, không ghi
    DaXuat --> DaNhan: Kho đến xác nhận nhận — kho đến: cộng, ghi "Chuyển kho - nhập"
    DaXuat --> HuySauXuat: Hủy khi đang chuyển — kho đi: cộng nhập lại
    DaNhan --> [*]
    HuyNhap --> [*]
    HuySauXuat --> [*]
```

> **Lưu ý:** Ở trạng thái "Đã xuất", số hàng đang chuyển **không bán được ở cả hai kho** — kho đi đã trừ, kho đến chưa cộng (BR-10). Phiếu chuyển bị hủy sau khi xuất không xóa bản ghi "Chuyển kho — xuất" đã ghi, mà sinh bản ghi cộng nhập lại (BR-06).
