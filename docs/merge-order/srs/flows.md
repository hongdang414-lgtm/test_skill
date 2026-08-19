---
type: srs-flows
feature: merge-order
updated: 2026-07-23
---

## Flow: merge-order-main-flow (Activity Diagram)

> Luồng nghiệp vụ tổng thể từ góc nhìn các vai trò tham gia (Swimlane).

```mermaid
flowchart TB
    subgraph CashierLane ["Thu ngân"]
        Start([Bắt đầu]) --> OpenDetail["Mở chi tiết đơn hàng gốc<br/>trạng thái ACTIVE"]
        OpenDetail --> ClickMerge["Nhấn nút Ghép đơn"]
        SelectTarget["Chọn đơn đích<br/>từ danh sách"] --> ReviewPreview["Xem preview tài chính<br/>đơn đích sau ghép"]
        ReviewPreview --> CashierDecision{"Xác nhận<br/>ghép đơn?"}
        CashierDecision -- "Huỷ" --> CancelMerge["Đóng, không thay đổi"]
        CashierDecision -- "Xác nhận" --> WaitSystem["Chờ hệ thống xử lý"]
        MergeSuccess["Mở chi tiết<br/>đơn đích sau ghép"] --> End([Kết thúc])
        MergeError["Xem thông báo lỗi"] --> End
        CancelMerge --> End
    end

    subgraph SystemLane ["Hệ thống POS"]
        ClickMerge --> CheckPermission{"Có quyền<br/>MERGE_ORDER?"}
        CheckPermission -- "Không" --> HideBtn["Ẩn nút Ghép đơn"]
        CheckPermission -- "Có" --> CheckEligible{"Đơn gốc đủ<br/>điều kiện?"}
        CheckEligible -- "Không đủ" --> DisableBtn["Vô hiệu hóa nút<br/>+ tooltip giải thích"]
        CheckEligible -- "Đủ" --> LoadTargets["Tải DS đơn có thể ghép<br/>WHERE status=ACTIVE"]
        LoadTargets --> CheckTargets{"Có đơn đích<br/>hợp lệ?"}
        CheckTargets -- "Không" --> ShowEmpty["DS rỗng + thông báo"]
        CheckTargets -- "Có" --> SelectTarget
        WaitSystem --> ValidateConditions{"Kiểm tra<br/>điều kiện lần cuối"}
        ValidateConditions -- "Không hợp lệ" --> ShowError["Thông báo lỗi chi tiết"]
        ValidateConditions -- "Hợp lệ" --> BeginTx["BEGIN TRANSACTION"]
        BeginTx --> TransferItems["Chuyển items gốc sang đích<br/>BR-06 BR-09 BR-13"]
        TransferItems --> RecalcTarget["Tính lại tài chính đơn đích<br/>BR-10 BR-11 BR-12"]
        RecalcTarget --> MarkMerged["Đánh dấu đơn gốc MERGED<br/>BR-14"]
        MarkMerged --> CheckDiffTable{"Khác bàn?"}
        CheckDiffTable -- "Có" --> FreeTable["Giải phóng bàn gốc<br/>BR-17"]
        CheckDiffTable -- "Không" --> SyncKDS
        FreeTable --> SyncKDS["Đồng bộ KDS<br/>BR-16"]
        SyncKDS --> WriteLog["Ghi Audit Log<br/>BR-19"]
        WriteLog --> CommitTx["COMMIT TRANSACTION"]
        CommitTx --> MergeSuccess
        ShowError --> MergeError
    end

    subgraph ResultLane ["Kết quả"]
        HideBtn --> End2([Kết thúc])
        DisableBtn --> End2
        ShowEmpty --> End2
    end
```

---

## Flow: merge-order-system-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo thứ tự thời gian.

```mermaid
sequenceDiagram
    autonumber
    actor Cashier as Thu ngân
    participant UI as POS UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant Finance as Finance Engine
    participant DB as Database
    participant KDS as KDS Service
    participant Audit as Audit Log

    Cashier->>UI: Mở chi tiết đơn gốc → Nhấn Ghép đơn

    UI->>API: GET /orders/{id}/merge-eligibility
    API->>Auth: Kiểm tra quyền MERGE_ORDER
    alt Không có quyền
        Auth-->>API: 403 PERMISSION_DENIED
        API-->>UI: 403
        UI-->>Cashier: Ẩn nút Ghép đơn
    else Có quyền
        Auth-->>API: 200 OK
        API->>DB: SELECT order WHERE id={id} AND status=ACTIVE
        DB-->>API: order data

        alt Đơn không đủ điều kiện
            API-->>UI: 422 ORDER_NOT_MERGEABLE (reason)
            UI-->>Cashier: Disable nút + tooltip
        else Đơn hợp lệ
            API->>DB: SELECT orders WHERE status=ACTIVE AND id != {id} AND status != MERGED
            DB-->>API: target_orders[]
            API-->>UI: 200 + danh sách đơn có thể ghép
            UI-->>Cashier: Hiển thị danh sách đơn đích

            Cashier->>UI: Chọn đơn đích
            UI->>API: POST /orders/{source}/merge/preview (target_id)
            API->>Finance: Tính toán ghép items + discounts + tax
            Finance-->>API: preview {target_after_merge, warnings[]}
            API-->>UI: 200 + preview + warnings
            UI-->>Cashier: Hiển thị preview + cảnh báo

            Cashier->>UI: Nhấn Xác nhận Ghép đơn
            UI->>API: POST /orders/{source}/merge/confirm (target_id)

            API->>DB: Re-validate trạng thái cả 2 đơn
            alt Điều kiện thay đổi
                API-->>UI: 422 MERGE_CONDITIONS_CHANGED
                UI-->>Cashier: Lỗi, yêu cầu thử lại
            else Hợp lệ
                API->>DB: BEGIN TRANSACTION
                API->>DB: UPDATE target SET items+=source.items, recalc totals
                API->>DB: UPDATE source SET status=MERGED, merged_into={target}
                API->>DB: COMMIT

                par Đồng bộ song song
                    API->>KDS: PATCH /kds/orders — chuyển items
                    KDS-->>API: 200 OK
                and
                    API->>Audit: POST /audit-log (merge details)
                    Audit-->>API: 200 recorded
                end

                API-->>UI: 200 + {target_order_final}
                UI-->>Cashier: Ghép thành công → Mở chi tiết đơn đích
            end
        end
    end
```
