---
type: srs-flows
feature: tpaygate-connect-bank
updated: 2026-07-30
---

## Flow: tpaygate-connect-bank-embedded-ui (Activity Diagram)

> Luồng nghiệp vụ kết nối ngân hàng qua **Embedded UI** — T-PayGate cung cấp giao diện trong popup/tab. Phù hợp cho web app, đơn giản về phía đối tác.

```mermaid
flowchart TB
    subgraph UserLane ["Người dùng đối tác"]
        Start([Bắt đầu]) --> OpenList["Mở màn hình\nDanh sách liên kết ngân hàng"]
        OpenList --> ClickAdd["Nhấn 'Thêm liên kết ngân hàng'"]
        SelectBank["Chọn ngân hàng từ danh sách"] --> OpenEmbedded["Hệ thống mở popup/tab\nT-PayGate Embedded UI"]
        OpenEmbedded --> UserFillForm["Người dùng nhập thông tin\ntài khoản + xác nhận OTP\n(trên giao diện T-PayGate)"]
        UserFillForm --> WaitSignal["Chờ tín hiệu postMessage\n'tabClosed' từ T-PayGate"]
        WaitSignal --> End([Kết thúc])
    end

    subgraph SystemLane ["Hệ thống đối tác"]
        ClickAdd --> CheckConfig{"Đã cấu hình\nthông số T-PayGate?"}
        CheckConfig -- "Chưa (BR-01)" --> ShowConfigError["Hiển thị banner:\nChưa cấu hình thông số\nLiên hệ quản trị viên"]
        ShowConfigError --> End2([Kết thúc])
        CheckConfig -- "Đã có" --> LoadBanks["GET /api/v1/public-api/bank\n(gọi ẩn danh — BR-03)"]
        LoadBanks --> BankOK{"API\nthành công?"}
        BankOK -- "Lỗi" --> ShowBankError["Hiển thị lỗi:\nKhông tải được danh sách ngân hàng"]
        ShowBankError --> End3([Kết thúc])
        BankOK -- "OK" --> ShowBankList["Hiển thị danh sách\nngân hàng (IsActive=true — BR-02)"]
        ShowBankList --> SelectBank
        OpenEmbedded --> ListenMessage["Lắng nghe window.postMessage\nKiểm tra event.origin (BR-11)"]
        ListenMessage --> MessageReceived{"Nhận\n'tabClosed'?"}
        MessageReceived -- "Nhận được" --> RefreshList["GET /config-bank/list\nXác nhận IsConnected (BR-10, BR-17)"]
        RefreshList --> UpdateUI["Cập nhật danh sách UI\nHiển thị kết nối mới"]
        UpdateUI --> WaitSignal
    end
```

---

## Flow: tpaygate-connect-bank-api-direct (Activity Diagram)

> Luồng kết nối ngân hàng qua **API trực tiếp** — đối tác tự thiết kế UI, ký HMAC-SHA256 và gọi API T-PayGate. Phù hợp cho mobile app hoặc khi cần UI tùy biến sâu.

```mermaid
flowchart TB
    subgraph UserLane ["Người dùng đối tác"]
        Start([Bắt đầu]) --> FillForm["Nhập thông tin tài khoản:\nbankCode, merchantName,\naccountName, accountNo\n(+ các field tùy chọn theo ngân hàng)"]
        FillForm --> ClickConnect["Nhấn 'Kết nối ngân hàng'"]
        OTPScreen["Màn hình nhập OTP\n(nếu IsOTPConfirmation=true)"] --> EnterOTP["Nhập OTP nhận được\nqua SMS"]
        EnterOTP --> ClickConfirm["Nhấn 'Xác nhận OTP'"]
        ConnectSuccess["Hiển thị kết quả:\nKết nối thành công\nSố VA: {VaNumber}"] --> End([Kết thúc])
    end

    subgraph SystemLane ["Hệ thống đối tác"]
        ClickConnect --> ValidateForm{"Validate form\nphía client (BR-06)"}
        ValidateForm -- "Lỗi" --> ShowFormError["Hiển thị lỗi inline\ntrên từng trường"]
        ShowFormError --> FillForm
        ValidateForm -- "Hợp lệ" --> CheckExisting{"Kiểm tra accountNo\nđã kết nối chưa? (BR-05)"}
        CheckExisting -- "Đã tồn tại (IsConnected=true)" --> ShowDuplicateError["Thông báo:\nTài khoản đã được kết nối"]
        ShowDuplicateError --> End2([Kết thúc])
        CheckExisting -- "Chưa tồn tại" --> CallConnect["POST /config-bank/connect\n(ký HMAC POST — BR-04)"]
        CallConnect --> ConnectResult{"Error == null?"}
        ConnectResult -- "Lỗi" --> ShowAPIError["Hiển thị thông báo lỗi\ntừ Error.Message"]
        ShowAPIError --> FillForm
        ConnectResult -- "OK" --> CheckOTP{"IsOTPConfirmation\n== true?"}
        CheckOTP -- "Có OTP" --> OTPScreen
        CheckOTP -- "Không OTP" --> IsConnectedDirect{"IsConnected\n== true?"}
        IsConnectedDirect -- "Chưa connected" --> WaitConfirm["Chờ xử lý\nT-PayGate phía sau"]
        IsConnectedDirect -- "Connected" --> FetchVaNumber["GET /config-bank/list\nlọc ConfigBankId → lấy VaNumber (BR-09)"]
        FetchVaNumber --> UpsertDB["UPSERT DB đối tác\ntheo ConfigBankId (BR-08)"]
        UpsertDB --> ConnectSuccess

        ClickConfirm --> ValidateOTP{"OTP hợp lệ?\n(BR-15)"}
        ValidateOTP -- "Không hợp lệ" --> ShowOTPError["Lỗi:\nOTP không hợp lệ"]
        ShowOTPError --> EnterOTP
        ValidateOTP -- "Hợp lệ" --> CallConfirm["POST /config-bank/confirm\n(ký HMAC POST — BR-04)"]
        CallConfirm --> ConfirmResult{"Error == null\n& IsConnected=true?"}
        ConfirmResult -- "Lỗi" --> ShowConfirmError["Hiển thị lỗi\n(sai OTP, hết hạn…)"]
        ShowConfirmError --> EnterOTP
        ConfirmResult -- "OK" --> SaveVaNumber["Lưu ConfigBankId + VaNumber\nvào DB (BR-08)"]
        SaveVaNumber --> ConnectSuccess
    end
```

---

## Flow: tpaygate-connect-bank-lifecycle (Activity Diagram)

> Vòng đời của một liên kết ngân hàng (`ConfigBankId`) từ khi tạo đến khi kết nối thành công. Nghiệp vụ ngắt kết nối được đặc tả riêng tại `[[docs/tpaygate-disconnect-bank/srs/flows|T-PayGate Disconnect Bank Flows]]`.

```mermaid
flowchart LR
    Created["Gọi /connect"] --> Pending["Chờ xác nhận OTP\n(IsOTPConfirmation=true)"]
    Created --> Connected["Đã kết nối\n(IsConnected=true)"]
    Pending -- "Gọi /confirm\n(OTP đúng)" --> Connected
    Pending -- "OTP sai / hết hạn" --> Failed["Kết nối thất bại"]
    Failed -- "Thử lại\n(/connect lại)" --> Created
    Connected -- "Xem luồng Ngắt kết nối\n(tpaygate-disconnect-bank)" --> Disconnected["Đã ngắt kết nối\n(IsConnected=false)"]
```

---

## Flow: tpaygate-connect-bank-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo thứ tự thời gian — Luồng API trực tiếp với OTP.

```mermaid
sequenceDiagram
    autonumber
    actor User as Người dùng đối tác
    participant UI as Giao diện đối tác
    participant API as Backend đối tác
    participant TPG as T-PayGate API
    participant Bank as Ngân hàng

    User->>UI: Mở màn hình danh sách liên kết ngân hàng
    UI->>TPG: GET /api/v1/public-api/bank (ẩn danh, không token — BR-03)
    TPG-->>UI: 200 + [{Code, Name, ShortName, Logo, Bin}]
    UI-->>User: Hiển thị danh sách ngân hàng hỗ trợ

    User->>UI: Chọn ngân hàng + nhập thông tin tài khoản
    User->>UI: Nhấn "Kết nối ngân hàng"

    UI->>API: Gửi form kết nối
    API->>API: Kiểm tra trùng lặp trong DB (BR-05)
    API->>API: Lấy Access Token (từ cache hoặc gọi OAuth)
    API->>API: Tính HMAC-SHA256 dạng POST (BR-04)
    API->>TPG: POST /api/v1/public-api/config-bank/connect<br/>[Authorization: Bearer, x-api-time, x-signature]
    TPG->>Bank: Đăng ký kết nối + tài khoản
    Bank-->>TPG: Gửi OTP về SĐT merchant
    TPG-->>API: 200 {Error:null, Data:{ConfigBankId, IsConnected:false, IsOTPConfirmation:true}}
    API-->>UI: Chuyển màn nhập OTP

    User->>UI: Nhập OTP (6 số)
    UI->>UI: Validate OTP phía client (BR-15)
    UI->>API: Gửi OTP + ConfigBankId
    API->>API: Tính HMAC-SHA256 dạng POST (BR-04)
    API->>TPG: POST /api/v1/public-api/config-bank/confirm<br/>{configBankId, otpNumber}
    TPG->>Bank: Xác thực OTP
    Bank-->>TPG: OK + VA Number

    alt OTP đúng
        TPG-->>API: 200 {Error:null, Data:{ConfigBankId, VaNumber, IsConnected:true}}
        API->>API: UPSERT record vào DB (BR-08)
        API-->>UI: Kết nối thành công + VaNumber
        UI-->>User: Hiển thị: Kết nối thành công — Số VA: {VaNumber}
    else OTP sai hoặc hết hạn
        TPG-->>API: 200 {Error:{Code:"TMT.TPayment.Code:300", Message:"..."}, Data:null}
        API-->>UI: Lỗi xác thực OTP
        UI-->>User: Hiển thị lỗi, cho phép nhập lại OTP
    end
```

---

## Flow: tpaygate-disconnect-bank-sequence (Sequence Diagram)

> **Lưu ý modular BA:** Sơ đồ tương tác và trình tự chi tiết cho nghiệp vụ Ngắt kết nối ngân hàng đã được tách riêng thành tính năng độc lập `tpaygate-disconnect-bank`.
> 
> Xem đầy đủ các sơ đồ Activity, Sequence và State Transition tại: `[[docs/tpaygate-disconnect-bank/srs/flows|T-PayGate Disconnect Bank Flows]]`.
