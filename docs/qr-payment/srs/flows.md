---
type: srs-flows
feature: qr-payment
updated: 2026-08-03
---

## Flow: qr-payment-activity (Activity Diagram)

> Luồng nghiệp vụ Thanh toán đơn hàng qua mã QR ngân hàng — mô tả trình tự kiểm tra cấu hình ngân hàng, sinh bill QR qua T-PayGate, hiển thị mã QR cho khách, và xử lý webhook xác nhận thanh toán.

```mermaid
flowchart TB
    subgraph CashierLane ["Thu ngân (POS)"]
        Start([Bắt đầu]) --> SelectQR["Chọn phương thức 'Chuyển khoản QR'"]
        SelectQR --> CheckConfig{"Có configId hoặc\nConfigBankId IsConnected=true?\n(BR-01)"}
        CheckConfig -- "Không có cấu hình" --> ShowNoConfig["Hiển thị thông báo:\nChưa cấu hình tài khoản ngân hàng\nDisable phương thức QR (BR-01)"]
        ShowNoConfig --> End0([Kết thúc])
        CheckConfig -- "Hợp lệ" --> GenRefId["Sinh refTransactionId duy nhất:\n{storeCode}-{orderId}-{timestamp} (BR-02)"]
        GenRefId --> SignHMAC["Tính chữ ký HMAC-SHA256 dạng POST:\nclientId_tenantId_source_timestamp_rawBody (BR-04)"]
        SignHMAC --> CallBill["POST /api/v1/public-api/order/bill\n{refTransactionId, amount, description}"]
        CallBill --> BillResult{"Kết quả tạo bill?"}

        BillResult -- "Timeout / Lỗi mạng" --> CheckBillExist["GET /order/get-refTransactionId\nKiểm tra bill đã tạo chưa? (BR-03)"]
        CheckBillExist --> BillFound{"Bill đã tạo\ntrên T-PayGate?"}
        BillFound -- "Đã tạo" --> UseBillCode["Dùng BillCode từ vấn tin\nHiển thị màn hình QR"]
        BillFound -- "Chưa tạo" --> ShowNetError["Hiển thị lỗi kết nối\nCho phép thử lại (refTransactionId mới)"]
        ShowNetError --> End1([Kết thúc])

        BillResult -- "HTTP 403 (Chữ ký / Cấu hình)" --> ShowSigError["Hiển thị lỗi:\nLỗi kết nối thanh toán QR\nYêu cầu Quản lý kiểm tra (BR-04)"]
        ShowSigError --> End2([Kết thúc])

        BillResult -- "HTTP 200 (Error==null)" --> SaveBill["Lưu DB nội bộ:\nrefTransactionId, billCode, amount, configBankId\nstatus=Pending (BR-14)"]
        UseBillCode --> SaveBill
        SaveBill --> RenderQR{"QrDataURL có giá trị?\n(BR-05)"}
        RenderQR -- "Có" --> ShowQRFromURL["Hiển thị ảnh QR từ QrDataURL\n+ Thông tin InfoPayment (BR-06)"]
        RenderQR -- "Không" --> ShowQRFromEMV["Render QR từ chuỗi EMV QRBase64\n+ Thông tin InfoPayment (BR-06)"]
        ShowQRFromURL --> WaitPayment["Màn hình QR đang chờ thanh toán\nTimer đếm ngược (BR-07)"]
        ShowQRFromEMV --> WaitPayment

        WaitPayment --> QREvent{"Sự kiện xảy ra?"}
        QREvent -- "Timer hết hạn" --> ShowExpired["Ẩn QR, hiển thị nút\n'Tạo lại mã QR' (BR-07)"]
        ShowExpired --> End3([Kết thúc — Thu ngân tạo lại hoặc hủy])
        QREvent -- "Thu ngân nhấn Hủy" --> CancelQR["Đánh dấu BillCode cũ là\n'Đã hủy nội bộ' (BR-15)"]
        CancelQR --> End4([Đơn hàng về trạng thái chờ phương thức])
        QREvent -- "Webhook thanh toán về" --> WebhookFlow["Xử lý Webhook (BR-08 → BR-13)"]
        WebhookFlow --> PaymentResult{"Kết quả xử lý\nwebhook?"}
        PaymentResult -- "Đã thanh toán đủ" --> ShowSuccess["Màn hình Thanh toán thành công\nCập nhật đơn hàng (BR-10, BR-13)"]
        PaymentResult -- "Thanh toán một phần" --> ShowPartial["Hiển thị trạng thái:\nThanh toán một phần\n+ Số tiền còn thiếu (BR-10)"]
        ShowSuccess --> End5([Kết thúc — Thu ngân in hóa đơn])
        ShowPartial --> WaitPayment
    end
```

---

## Flow: qr-payment-sequence (Sequence Diagram)

> Tương tác chi tiết giữa các thành phần hệ thống theo trình tự thời gian cho luồng Thanh toán QR — bao gồm cả chiều tạo bill và chiều nhận webhook.

```mermaid
sequenceDiagram
    autonumber
    actor Cashier as Thu ngân (POS)
    participant UI as Màn hình POS
    participant POS as Backend POS
    participant TPG as T-PayGate API
    participant Bank as Ngân hàng đối tác
    actor Customer as Khách hàng

    Cashier->>UI: Chọn "Chuyển khoản QR" tại màn hình Thanh toán
    UI->>POS: Yêu cầu tạo QR (orderId, totalAmount)
    POS->>POS: Kiểm tra configId / ConfigBankId IsConnected=true (BR-01)
    POS->>POS: Sinh refTransactionId duy nhất (BR-02)
    POS->>POS: Cache / refresh Access Token OAuth (TTL 55 phút)
    POS->>POS: Tính chữ ký HMAC-SHA256 dạng POST (BR-04)
    POS->>TPG: POST /api/v1/public-api/order/bill<br/>[x-config-id, x-api-time, x-signature, Authorization]<br/>{refTransactionId, amount, description}

    alt Tạo bill thành công (Happy Path)
        TPG->>TPG: Tạo BillCode, sinh chuỗi EMV QR gắn VA
        TPG-->>POS: 200 {Error: null, Data: {BillCode, QRBase64, QrDataURL, InfoPayment}}
        POS->>POS: Lưu DB: refTransactionId, billCode, amount, status=Pending (BR-14)
        POS-->>UI: Trả về dữ liệu QR
        UI-->>Cashier: Hiển thị màn hình QR + countdown timer (BR-05, BR-06, BR-07)
        Customer->>UI: Quét QR bằng app ngân hàng
        Customer->>Bank: Xác nhận thanh toán trong app ngân hàng
        Bank-->>TPG: Ghi có tài khoản VA + xác nhận thanh toán
        TPG->>POS: POST {webhook_url}<br/>[x-api-time, x-signature]<br/>{RefTransactionId, BillCode, Amount, VirtualAccount, ActualAccount, PaymentTime}
        POS->>POS: Verify chữ ký webhook (BR-08) — raw body trước khi parse
        POS->>POS: Kiểm tra chống replay |now − x-api-time| ≤ 300s (BR-09)
        POS->>POS: Tra cứu đơn hàng theo RefTransactionId (BR-11)
        POS->>POS: Đối chiếu Amount với totalAmount (BR-10)
        POS->>POS: Cập nhật DB: status=Paid, paymentTime (BR-11)
        POS-->>TPG: HTTP 200 {"MessageError": null} (BR-12)
        POS->>UI: Phát sự kiện WebSocket / SSE (BR-13)
        UI-->>Cashier: Màn hình tự động chuyển "Thanh toán thành công"

    else Tạo bill lỗi chữ ký hoặc cấu hình (BR-04)
        TPG-->>POS: HTTP 403 {Error: {Message: "#TPayGate: Invalid signature."}}
        POS-->>UI: Lỗi kết nối thanh toán QR
        UI-->>Cashier: Thông báo lỗi, yêu cầu kiểm tra cấu hình

    else Tạo bill lỗi x-config-id không hợp lệ (BR-01)
        TPG-->>POS: HTTP 403 {Error: {Message: "#TPayGate: configBankId does not exist or is disconnected."}}
        POS-->>UI: Lỗi tài khoản ngân hàng đã ngắt kết nối
        UI-->>Cashier: Thông báo lỗi cấu hình, liên hệ Quản lý

    else Tạo bill Timeout / Lỗi mạng (BR-03)
        POS-xTPG: Timeout (không nhận phản hồi)
        POS->>TPG: GET /api/v1/public-api/order/get-refTransactionId?refTransactionId={ref}
        TPG-->>POS: 200 {Data: [{BillCode: "...", Amount: ...}]} hoặc []
        POS->>POS: Nếu đã có bill → dùng BillCode từ vấn tin, hiển thị QR
        POS->>POS: Nếu chưa có → thông báo lỗi, cho phép thử lại

    else Webhook chữ ký sai hoặc replay (BR-08, BR-09)
        TPG->>POS: POST {webhook_url} với chữ ký sai / x-api-time lệch
        POS->>POS: Verify thất bại → không cập nhật DB
        POS-->>TPG: HTTP 200 {"MessageError": "Invalid signature"} (BR-12)
    end
```

---

## Flow: qr-payment-lifecycle (State Transition Diagram)

> Sơ đồ chuyển đổi trạng thái vòng đời của một Bill QR (`BillCode`) trong hệ thống POS — từ khi tạo đến khi thanh toán hoàn tất hoặc bị hủy nội bộ.

```mermaid
stateDiagram-v2
    [*] --> BillCreated: POST /order/bill thành công\nLưu DB nội bộ: status=Pending (BR-14)

    state BillCreated {
        [*] --> QRDisplayed
        note right of QRDisplayed: BillCode + QR đang hiển thị\nTimer đếm ngược (BR-07)\nChờ khách quét và thanh toán
    }

    BillCreated --> QRExpired: Timer hết thời gian\nPOS ẩn QR (BR-07)
    BillCreated --> CancelledInternal: Thu ngân nhấn Hủy hoặc\nChọn phương thức khác (BR-15)
    BillCreated --> PartialPaid: Webhook về với Amount < totalAmount (BR-10)
    BillCreated --> FullyPaid: Webhook về với Amount ≥ totalAmount (BR-10)

    state PartialPaid {
        [*] --> WaitingMore
        note right of WaitingMore: Đã nhận một phần tiền\nChờ thanh toán bổ sung\nQR vẫn có thể dùng lại
    }

    PartialPaid --> FullyPaid: Webhook bổ sung về\nAmount ≥ totalAmount (BR-10)

    state FullyPaid {
        [*] --> OrderCompleted
        note right of OrderCompleted: Đơn hàng đã thanh toán đủ\nCập nhật DB + phát WebSocket (BR-13)\nThu ngân có thể in hóa đơn
    }

    state CancelledInternal {
        [*] --> BillVoidedLocally
        note right of BillVoidedLocally: BillCode cũ đánh dấu Đã hủy nội bộ (BR-15)\nT-PayGate vẫn giữ bill (không có API hủy)\nNếu webhook cũ về → POS từ chối, ack 200
    }

    state QRExpired {
        [*] --> AwaitingRecreate
        note right of AwaitingRecreate: QR đã hết thời gian hiển thị\nThu ngân có thể tạo lại QR mới\n(refTransactionId mới - BR-07)
    }

    QRExpired --> BillCreated: Thu ngân nhấn "Tạo lại QR"\nSinh refTransactionId mới (BR-02, BR-07)
    FullyPaid --> [*]
    CancelledInternal --> [*]
```

---

## Flow: qr-payment-webhook (Webhook Processing Detail)

> Luồng xử lý chi tiết khi nhận webhook thanh toán từ T-PayGate — tập trung vào bảo mật, idempotency và đối chiếu nghiệp vụ.

```mermaid
flowchart TB
    WebhookIn(["Nhận POST webhook từ T-PayGate"]) --> ReadRawBody["Đọc raw body\nTRƯỚC khi parse JSON (BR-08)"]
    ReadRawBody --> CheckHeaders{"Có đủ header\nx-signature và x-api-time?"}
    CheckHeaders -- "Thiếu header" --> Reject1["Từ chối\n{MessageError: 'Missing headers'}"]
    Reject1 --> AckTPG([Trả HTTP 200])

    CheckHeaders -- "Đủ header" --> AntiReplay{"Kiểm tra replay:\n|now − x-api-time| ≤ 300s\n(BR-09)"}
    AntiReplay -- "Lệch > 5 phút" --> Reject2["Từ chối\n{MessageError: 'Request expired'}"]
    Reject2 --> AckTPG

    AntiReplay -- "Trong cửa sổ" --> VerifySig{"Verify HMAC-SHA256:\nclientId_tenantId_Name_timestamp_rawBody\n(BR-08) — đoạn 3 là Name tenant gốc"}
    VerifySig -- "Sai chữ ký" --> Reject3["Từ chối\n{MessageError: 'Invalid signature'}"]
    Reject3 --> AckTPG

    VerifySig -- "Hợp lệ" --> ParseBody["Parse JSON body\n{RefTransactionId, BillCode, Amount,\nVirtualAccount, ActualAccount, PaymentTime}"]
    ParseBody --> FindOrder{"Tra đơn hàng theo\nRefTransactionId (BR-11)"}
    FindOrder -- "Không tìm thấy" --> Reject4["Từ chối\n{MessageError: 'Transaction not found'}"]
    Reject4 --> AckTPG

    FindOrder -- "Tìm thấy" --> CheckStatus{"Trạng thái đơn hàng\nhiện tại?"}
    CheckStatus -- "Đã thanh toán (Paid)" --> SkipDup["Bỏ qua (Idempotency)\n{MessageError: null}"]
    SkipDup --> AckTPG

    CheckStatus -- "Đã hủy nội bộ (CancelledInternal)" --> SkipCancel["Bỏ qua (BillCode đã hủy nội bộ)\n{MessageError: null} (BR-15)"]
    SkipCancel --> AckTPG

    CheckStatus -- "Pending / PartialPaid" --> CompareAmount{"Đối chiếu số tiền:\nwebhook.Amount vs totalAmount\n(BR-10)"}
    CompareAmount -- "Amount ≥ totalAmount" --> UpdatePaid["Cập nhật DB:\nstatus=Paid, paymentTime, amountPaid\n(Atomic update / Lock - BR-11)"]
    CompareAmount -- "Amount < totalAmount" --> UpdatePartial["Cập nhật DB:\nstatus=PartialPaid, amountPaid\nGhi log cảnh báo (BR-10)"]

    UpdatePaid --> NotifyRealtime["Phát sự kiện WebSocket/SSE:\nĐơn hàng đã thanh toán (BR-13)"]
    UpdatePartial --> NotifyPartial["Phát sự kiện WebSocket/SSE:\nThanh toán một phần (BR-13)"]

    NotifyRealtime --> AckSuccess["{MessageError: null} (BR-12)"]
    NotifyPartial --> AckSuccess
    AckSuccess --> AckTPG
```
