---
type: srs-flows
feature: price-list
updated: 2026-07-23
---

## Flow: price-list-main-flow (Activity Diagram)

> Luồng nghiệp vụ tổng thể tạo và quản lý bảng giá từ góc nhìn các vai trò tham gia (Swimlane).

```mermaid
flowchart TB
    subgraph ManagerLane ["Quản lý cửa hàng"]
        Start([Bắt đầu]) --> OpenList["Mở Danh sách bảng giá<br/>trên CMS"]
        OpenList --> ClickAdd["Nhấn Thêm bảng giá"]
        FillBasic["Nhập thông tin cơ bản:<br/>Tên, Mã, Mô tả,<br/>Thời gian, Khung giờ,<br/>Nhóm KH"] --> AddItems["Thêm SP/Combo/Option<br/>vào bảng giá"]
        AddItems --> SetPrice["Thiết lập giá mới<br/>(Nhập trực tiếp / Chiết khấu % /<br/>Giảm giá trị cố định)"]
        SetPrice --> ClickSave["Nhấn Lưu"]
        SaveSuccess["Bảng giá tạo thành công<br/>Trạng thái: Chưa diễn ra"] --> End([Kết thúc])
        SaveError["Xem thông báo lỗi<br/>Sửa lại thông tin"] --> FillBasic
    end

    subgraph SystemLane ["Hệ thống CMS"]
        ClickAdd --> CheckPermission{"Có quyền<br/>MANAGE_PRICE_LIST?"}
        CheckPermission -- "Không" --> DenyAccess["Ẩn menu Bảng giá<br/>403 PERMISSION_DENIED"]
        CheckPermission -- "Có" --> OpenForm["Mở form tạo bảng giá"]
        OpenForm --> FillBasic
        ClickSave --> ValidateBasic{"Thông tin<br/>cơ bản hợp lệ?"}
        ValidateBasic -- "Không hợp lệ" --> ShowValidation["Hiển thị lỗi validation<br/>inline trên form"]
        ShowValidation --> SaveError
        ValidateBasic -- "Hợp lệ" --> CheckOverlap{"Trùng thời gian<br/>bảng giá khác?"}
        CheckOverlap -- "Trùng" --> ShowOverlapErr["Cảnh báo trùng lặp<br/>BR-05"]
        ShowOverlapErr --> SaveError
        CheckOverlap -- "Không trùng" --> CheckItems{"Có ≥1 SP/<br/>Combo/Option?"}
        CheckItems -- "Không" --> ShowItemErr["Lỗi: Cần ít nhất 1 mục<br/>BR-09"]
        ShowItemErr --> SaveError
        CheckItems -- "Có" --> ValidatePrice{"Giá mới<br/>hợp lệ (≥0)?"}
        ValidatePrice -- "Không" --> ShowPriceErr["Lỗi giá không hợp lệ<br/>BR-08"]
        ShowPriceErr --> SaveError
        ValidatePrice -- "Hợp lệ" --> SaveDB["Lưu vào Database"]
        SaveDB --> WriteAudit["Ghi Audit Log<br/>BR-20"]
        WriteAudit --> SaveSuccess
    end

    subgraph ResultLane ["Kết quả"]
        DenyAccess --> End2([Kết thúc])
    end
```

---

## Flow: price-list-lifecycle (Activity Diagram)

> Luồng vòng đời bảng giá — trạng thái tự động theo thời gian.

```mermaid
flowchart LR
    Created["Bảng giá được tạo"] --> Pending["Chưa diễn ra"]
    Pending -- "now ≥ Thời gian bắt đầu<br/>(Scheduler tự động)" --> Active["Đang diễn ra"]
    Active -- "now > Thời gian kết thúc<br/>(Scheduler tự động)" --> Ended["Đã kết thúc"]

    Pending -- "Xóa bảng giá<br/>(Thao tác thủ công)" --> Deleted["Đã xóa"]
    Pending -- "Sửa thông tin<br/>(Toàn bộ)" --> Pending
    Active -- "Kết thúc sớm<br/>(Admin — BR-16)" --> Ended
    Active -- "Sửa giá SP<br/>(Chỉ giá — BR-13)" --> Active
    Ended -- "Sao chép<br/>(BR-14)" --> Pending2["Bản sao mới<br/>Chưa diễn ra"]
    Ended -- "Xóa" --> Deleted
```

---

## Flow: price-list-system-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo thứ tự thời gian khi tạo bảng giá.

```mermaid
sequenceDiagram
    autonumber
    actor Manager as Quản lý cửa hàng
    participant UI as CMS UI
    participant API as API Gateway
    participant Auth as Auth Service
    participant Product as Product Service
    participant DB as Database
    participant Audit as Audit Log

    Manager->>UI: Mở Danh sách bảng giá
    UI->>API: GET /price-lists?page=1&status=all
    API->>Auth: Kiểm tra quyền MANAGE_PRICE_LIST
    alt Không có quyền
        Auth-->>API: 403 PERMISSION_DENIED
        API-->>UI: 403
        UI-->>Manager: Ẩn menu / Thông báo lỗi quyền
    else Có quyền
        Auth-->>API: 200 OK
        API->>DB: SELECT price_lists ORDER BY updated_at DESC
        DB-->>API: price_list[]
        API-->>UI: 200 + danh sách bảng giá
        UI-->>Manager: Hiển thị bảng danh sách

        Manager->>UI: Nhấn "Thêm bảng giá"
        UI-->>Manager: Mở form tạo mới

        Manager->>UI: Nhấn "Thêm sản phẩm"
        UI->>API: GET /products?search={q}&page=1
        API->>Product: Lấy danh sách SP từ catalog
        Product-->>API: products[]
        API-->>UI: 200 + DS sản phẩm
        UI-->>Manager: Hiển thị modal chọn SP

        Manager->>UI: Chọn SP + thiết lập giá → Xác nhận
        UI-->>Manager: Cập nhật bảng giá trên form

        Manager->>UI: Nhấn "Lưu"
        UI->>API: POST /price-lists (payload)

        API->>DB: Check tên unique
        API->>DB: Check mã unique
        API->>DB: Check overlap thời gian với bảng giá khác

        alt Validation lỗi
            API-->>UI: 422 VALIDATION_ERROR (details[])
            UI-->>Manager: Hiển thị lỗi inline
        else Trùng thời gian
            API-->>UI: 409 TIME_OVERLAP (conflicting_price_list)
            UI-->>Manager: Cảnh báo trùng lặp thời gian
        else Hợp lệ
            API->>DB: BEGIN TRANSACTION
            API->>DB: INSERT price_list + price_list_items
            API->>DB: COMMIT
            API->>Audit: POST /audit-log (CREATE, details)
            Audit-->>API: 200 recorded
            API-->>UI: 201 + {price_list}
            UI-->>Manager: Tạo thành công → Chuyển về danh sách
        end
    end
```
