---
type: srs-flows
feature: split-order
updated: 2026-07-21
---

## Flow: split-order-main-flow (Activity Diagram)

> Luồng nghiệp vụ tổng thể từ góc nhìn các vai trò tham gia (Swimlane).

```mermaid
flowchart TB
    subgraph CashierLane ["Thu ngân"]
        Start([Bắt đầu]) --> SelectOrder["Chọn đơn hàng gốc\ntrạng thái ACTIVE"]
        SelectOrder --> ClickSplit["Nhấn nút Tách đơn"]
        OpenUI["Màn hình chọn\nsản phẩm tách"] --> SelectItems["Chọn sản phẩm/SL\nmuốn tách sang đơn mới"]
        SelectItems --> ReviewPreview["Xem preview tài chính\n2 đơn sau tách"]
        ReviewPreview --> CashierDecision{"Xác nhận\nthao tác?"}
        CashierDecision -- "Huỷ" --> CancelSplit["Đóng màn hình\nkhông thay đổi"]
        CashierDecision -- "Xác nhận" --> WaitSystem["Chờ hệ thống\nxử lý"]
        SplitSuccess["Hiển thị 2 đơn\nhàng độc lập"] --> End([Kết thúc])
        SplitError["Xem thông báo lỗi\nthử lại hoặc thoát"] --> End
        CancelSplit --> End
    end

    subgraph SystemLane ["Hệ thống POS"]
        ClickSplit --> CheckPermission{"Có quyền\nTách đơn?"}
        CheckPermission -- "Không có quyền" --> HideBtn["Ẩn nút Tách đơn"]
        CheckPermission -- "Có quyền" --> CheckEligible{"Đơn đủ\nđiều kiện?"}
        CheckEligible -- "Không đủ\nSL hoặc sai TT" --> DisableBtn["Vô hiệu hóa nút\n+ tooltip giải thích"]
        CheckEligible -- "Đủ điều kiện" --> OpenUI
        SelectItems --> ValidateSelection{"Lựa chọn\nhợp lệ?"}
        ValidateSelection -- "Chọn toàn bộ\nhoặc SL=0" --> ShowInlineError["Hiển thị lỗi inline\ncho phép sửa lại"]
        ShowInlineError --> SelectItems
        ValidateSelection -- "Hợp lệ" --> CalcPreview["Tính toán preview\ngiảm giá + thuế + phí DV\ntheo BR-07 đến BR-10"]
        CalcPreview --> ReviewPreview
        WaitSystem --> BeginTx["BEGIN TRANSACTION"]
        BeginTx --> UpdateOriginal["Cập nhật đơn gốc\nxóa item đã tách\ntính lại tổng tiền"]
        UpdateOriginal --> CreateNew["Tạo đơn mới\nparent_order_id=gốc\ntính lại tổng tiền"]
        CreateNew --> CheckBalance{"Tổng tiền\ncân bằng?"}
        CheckBalance -- "Lệch > 1 VND" --> Rollback["ROLLBACK\nBáo lỗi FINANCIAL_INCONSISTENCY"]
        CheckBalance -- "Cân bằng" --> CommitTx["COMMIT TRANSACTION"]
        CommitTx --> SyncKDS["Đồng bộ KDS\ntheo đơn mới"]
        SyncKDS --> WriteLog["Ghi Audit Log\nchi tiết actor + items + amounts"]
        WriteLog --> UpdateMapping["Cập nhật mapping\ntồn kho - không trừ thêm"]
        UpdateMapping --> SplitSuccess
        Rollback --> SplitError
        HideBtn --> End2([Kết thúc])
        DisableBtn --> End2
    end
```

---

## Flow: split-order-system-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo thứ tự thời gian.

```mermaid
sequenceDiagram
    autonumber
    actor Cashier as Thu ngân
    participant UI as POS UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant Finance as Finance Engine
    participant DB as Database (Orders)
    participant KDS as KDS Service
    participant Audit as Audit Log

    Cashier->>UI: Chọn đơn hàng gốc → Nhấn Tách đơn

    UI->>API: GET /orders/{id}/split-eligibility
    API->>Auth: Kiểm tra quyền SPLIT_ORDER của actor
    alt Không có quyền
        Auth-->>API: 403 PERMISSION_DENIED
        API-->>UI: 403
        UI-->>Cashier: Ẩn nút Tách đơn
    else Có quyền
        Auth-->>API: 200 OK
        API->>DB: SELECT order + items WHERE id={id}
        DB-->>API: order data + status + items[]

        alt Đơn không đủ điều kiện (status != ACTIVE hoặc tổng SL < 2)
            API-->>UI: 422 ORDER_NOT_SPLITTABLE (reason)
            UI-->>Cashier: Vô hiệu hóa nút + tooltip lý do
        else Đơn hợp lệ
            API-->>UI: 200 + order_data + items[]
            UI-->>Cashier: Mở màn hình chọn sản phẩm tách

            Cashier->>UI: Chọn sản phẩm + số lượng muốn tách
            UI->>API: POST /orders/{id}/split/preview (items_to_split[])
            API->>Finance: Tính toán phân bổ giảm giá BR-07 BR-08
            Finance->>Finance: Tính thuế + phí DV cho 2 đơn BR-09
            Finance-->>API: preview {order_goc_updated, order_moi}
            API-->>UI: 200 + preview data
            UI-->>Cashier: Hiển thị preview tài chính 2 đơn

            Cashier->>UI: Nhấn Xác nhận Tách đơn
            UI->>API: POST /orders/{id}/split/confirm (items_to_split[])

            API->>Finance: Tính toán cuối cùng + validate BR-10
            Finance-->>API: Kết quả tài chính + balance_check

            alt Tổng tiền lệch > 1 VNĐ
                API-->>UI: 500 FINANCIAL_INCONSISTENCY
                UI-->>Cashier: Lỗi hệ thống, đơn gốc giữ nguyên
            else Tài chính hợp lệ
                API->>DB: BEGIN TRANSACTION
                API->>DB: UPDATE orders SET items=remaining, totals=recalc WHERE id={goc_id}
                API->>DB: INSERT INTO orders (items=split_items, parent_order_id={goc_id}, totals=recalc)
                API->>DB: COMMIT

                par Đồng bộ song song
                    API->>KDS: PATCH /kds/orders — sync items to new order
                    KDS-->>API: 200 OK (hoặc warning nếu lỗi)
                and
                    API->>Audit: POST /audit-log (actor_id, order_goc, order_moi, items, amounts, timestamp)
                    Audit-->>API: 200 recorded
                end

                API-->>UI: 200 + {order_goc_final, order_moi_final}
                UI-->>Cashier: Thông báo tách thành công + hiển thị 2 đơn
            end
        end
    end
```

---

## Flow: split-order-combo-edge-case (Activity Diagram)

> Luồng xử lý riêng cho sản phẩm combo khi thực hiện tách đơn.

```mermaid
flowchart TB
    Start([Bắt đầu chọn SP tách]) --> CheckItemType{"Loại sản phẩm?"}

    CheckItemType -- "Sản phẩm thường" --> NormalItem["Hiển thị trường\nNhập số lượng tách"]
    CheckItemType -- "Sản phẩm Combo" --> ComboItem["Hiển thị Combo\nnhư 1 đơn vị nguyên"]

    NormalItem --> InputQty["Thu ngân nhập SL\nmin=1 max=SL_goc"]
    ComboItem --> InputComboQty["Thu ngân chọn SL combo\nmin=1 max=SL_combo_goc"]

    InputQty --> ValidateQty{"SL hợp lệ?\n0 < SL <= SL_goc"}
    InputComboQty --> ValidateComboQty{"SL hợp lệ?\n0 < SL <= SL_combo_goc"}

    ValidateQty -- "Không hợp lệ" --> ShowQtyError["Lỗi: Số lượng\nkhông hợp lệ"]
    ValidateQty -- "Hợp lệ" --> AddToSplitList["Thêm vào danh\nsách tách"]
    ValidateComboQty -- "Không hợp lệ" --> ShowComboError["Lỗi: Số lượng\ncombo không hợp lệ"]
    ValidateComboQty -- "Hợp lệ" --> AddComboToList["Thêm nguyên combo\nvào danh sách tách\nKHÔNG tách thành phần"]

    ShowQtyError --> InputQty
    ShowComboError --> InputComboQty
    AddToSplitList --> CheckTotal{"Còn ít nhất 1 SP\ntrong đơn gốc?"}
    AddComboToList --> CheckTotal

    CheckTotal -- "Không còn\nSP trong gốc" --> DisableConfirm["Vô hiệu hóa\nnút Xác nhận\n+ cảnh báo inline"]
    CheckTotal -- "Còn ít nhất 1 SP" --> EnableConfirm["Kích hoạt\nnút Xác nhận"]

    DisableConfirm --> End([Chờ thu ngân điều chỉnh])
    EnableConfirm --> End
```
