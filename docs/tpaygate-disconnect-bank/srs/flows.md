---
type: srs-flows
feature: tpaygate-disconnect-bank
updated: 2026-07-31
---

## Flow: tpaygate-disconnect-bank-activity (Activity Diagram)

> Luồng nghiệp vụ Ngắt kết nối ngân hàng đối tác — mô tả trình tự kiểm tra quyền hạn, xác nhận qua Dialog, ký HMAC-SHA256, gọi API T-PayGate và cập nhật trạng thái trong cơ sở dữ liệu.

```mermaid
flowchart TB
    subgraph UserLane ["Người dùng đối tác"]
        Start([Bắt đầu]) --> OpenList["Mở màn hình Danh sách liên kết ngân hàng"]
        OpenList --> ClickDisconnect["Nhấn nút 'Ngắt kết nối' trên dòng IsConnected=true (BR-01)"]
        ConfirmDialog["Dialog xác nhận ngắt kết nối"] --> UserConfirm["Người dùng nhấn 'Xác nhận ngắt kết nối' (BR-02)"]
        UserConfirm --> End([Kết thúc luồng thao tác UI])
    end

    subgraph SystemLane ["Hệ thống đối tác"]
        ClickDisconnect --> CheckPermission{"Người dùng có quyền\nQuản lý / Chủ cửa hàng? (BR-08)"}
        CheckPermission -- "Không có quyền" --> ShowPermError["Hiển thị lỗi:\nBạn không có quyền thực hiện thao tác này"]
        ShowPermError --> End2([Kết thúc])
        CheckPermission -- "Hợp lệ" --> ConfirmDialog
        
        UserConfirm --> ValidateConfigId{"ConfigBankId hợp lệ\nvà tồn tại trong DB? (BR-09)"}
        ValidateConfigId -- "Không hợp lệ" --> ShowValError["Hiển thị lỗi:\nMã liên kết ngân hàng không hợp lệ"]
        ShowValError --> End3([Kết thúc])
        ValidateConfigId -- "Hợp lệ" --> SignHMAC["Tính chữ ký HMAC-SHA256 dạng POST:\nclientId_tenantId_source_timestamp_rawBody (BR-03)"]
        
        SignHMAC --> CallAPI["POST /api/v1/public-api/config-bank/disconnect"]
        CallAPI --> APIResult{"Kết quả gọi API?"}
        
        APIResult -- "Timeout / Lỗi mạng" --> CallList["GET /api/v1/public-api/config-bank/list\nKiểm tra trạng thái thực tế (BR-06)"]
        CallList --> CheckRealState{"IsConnected trên\nT-PayGate?"}
        CheckRealState -- "true (chưa ngắt)" --> ShowTimeoutError["Thông báo lỗi kết nối:\nVui lòng thử lại thủ công"]
        ShowTimeoutError --> End4([Kết thúc])
        CheckRealState -- "false (đã ngắt)" --> UpdateDB["Cập nhật DB: IsConnected=false\nGIỮ NGUYÊN bản ghi DB (BR-04)"]
        
        APIResult -- "HTTP 403 (Already Disconnected)" --> SyncState["Đồng bộ trạng thái DB nội bộ:\nIsConnected=false (BR-07)"]
        SyncState --> UpdateUI["Cập nhật UI Danh sách:\nBadge xám 'Đã ngắt kết nối'\nẨn nút Ngắt kết nối (BR-01, BR-10)"]
        
        APIResult -- "HTTP 200 (Error==null)" --> UpdateDB
        UpdateDB --> UpdateUI
        UpdateUI --> NotifySuccess["Hiển thị Toast:\nĐã ngắt kết nối tài khoản thành công"]
        NotifySuccess --> End5([Kết thúc])
        
        APIResult -- "HTTP 403 (Sai chữ ký / Hết hạn)" --> ShowSigError["Hiển thị lỗi:\nChữ ký không hợp lệ hoặc request hết hạn (BR-03)"]
        ShowSigError --> End6([Kết thúc])
    end
```

---

## Flow: tpaygate-disconnect-bank-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo trình tự thời gian cho luồng Ngắt kết nối ngân hàng.

```mermaid
sequenceDiagram
    autonumber
    actor User as Người dùng đối tác (Quản lý)
    participant UI as Giao diện đối tác
    participant API as Backend đối tác
    participant TPG as T-PayGate API
    participant Bank as Ngân hàng đối tác

    User->>UI: Nhấn "Ngắt kết nối" trên liên kết IsConnected=true (BR-01)
    UI->>UI: Kiểm tra quyền quản lý cấu hình thanh toán (BR-08)
    UI-->>User: Hiển thị Dialog xác nhận ngắt kết nối (BR-02)
    User->>UI: Xác nhận ngắt kết nối

    UI->>API: Gửi yêu cầu ngắt kết nối (configBankId)
    API->>API: Kiểm tra tính hợp lệ của configBankId (BR-09)
    API->>API: Lấy Access Token hợp lệ từ OAuth cache
    API->>API: Tính chữ ký HMAC-SHA256 dạng POST (BR-03)
    API->>TPG: POST /api/v1/public-api/config-bank/disconnect<br/>[Authorization: Bearer, x-api-time, x-signature]<br/>{configBankId: "..."}

    alt Trường hợp thành công (Happy Path)
        TPG->>Bank: Gọi API hủy đăng ký liên kết phía ngân hàng
        Bank-->>TPG: OK (Đã hủy liên kết)
        TPG-->>API: 200 {Error: null, Data: ...}
        API->>API: Cập nhật DB nội bộ: IsConnected = false (giữ bản ghi - BR-04)
        API-->>UI: Thành công
        UI-->>User: Hiển thị Toast thành công + Badge xám "Đã ngắt kết nối"
    else Trường hợp liên kết đã bị ngắt từ trước (BR-07)
        TPG-->>API: 403 {Error: {Code: 403, Message: "#TPayGate: configBankId does not exist or is disconnected."}}
        API->>API: Cập nhật DB nội bộ: IsConnected = false (đồng bộ trạng thái T-PayGate)
        API-->>UI: Thông báo tài khoản đã ngắt kết nối trên T-PayGate
        UI-->>User: Cập nhật UI sang trạng thái "Đã ngắt kết nối"
    else Trường hợp lỗi chữ ký hoặc hết hạn request (BR-03)
        TPG-->>API: 403 {Error: {Code: 403, Message: "#TPayGate: Invalid signature." / "Request expired."}}
        API-->>UI: Lỗi xác thực T-PayGate
        UI-->>User: Hiển thị thông báo lỗi, không cập nhật DB
    else Trường hợp Timeout / Sự cố mạng (BR-06)
        API-xTPG: Timeout (không nhận được phản hồi sau 30s)
        API->>TPG: GET /api/v1/public-api/config-bank/list (Kiểm tra trạng thái thực tế)
        TPG-->>API: 200 {Data: [{ConfigBankId: "...", IsConnected: false}]}
        API->>API: Cập nhật DB nội bộ theo thực tế IsConnected
        API-->>UI: Cập nhật giao diện Danh sách
    end
```

---

## Flow: tpaygate-disconnect-bank-lifecycle (State Transition Diagram)

> Sơ đồ chuyển đổi trạng thái vòng đời của một liên kết ngân hàng (`ConfigBankId`) và ảnh hưởng đến việc tạo hóa đơn thanh toán.

```mermaid
stateDiagram-v2
    [*] --> Connected: Kết nối thành công qua luồng /connect hoặc /confirm
    
    state Connected {
        [*] --> ActiveForBilling
        note right of ActiveForBilling: IsConnected = true\nCho phép tạo hóa đơn mới (POST /order/bill)\nNhận webhook thanh toán cho mọi đơn hàng
    }
    
    Connected --> Disconnected: Gọi POST /config-bank/disconnect\n(Người dùng Quản lý xác nhận)
    
    state Disconnected {
        [*] --> ArchivedInDB
        note right of ArchivedInDB: IsConnected = false\nGIỮ NGUYÊN bản ghi DB để đối soát (BR-04)\nChặn tạo hóa đơn mới (BR-05)\nHóa đơn cũ trước đó vẫn nhận webhook bình thường
    }
    
    Disconnected --> [*]: Không thể tái kích hoạt bản ghi cũ (BR-10)\nMuốn sử dụng lại phải tạo kết nối mới
```
