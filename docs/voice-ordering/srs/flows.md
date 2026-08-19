---
type: srs-flows
feature: voice-ordering
updated: 2026-08-14
---

## Flow: voice-ordering-main-flow (Activity Diagram)

> Luồng nghiệp vụ tổng thể từ góc nhìn các vai trò tham gia (Swimlane).

```mermaid
flowchart TB
    subgraph CashierLane ["Thu ngân (Cashier)"]
        Start([Bắt đầu]) --> OpenOrder["Mở đơn ACTIVE\nhoặc tạo đơn mới"]
        OpenOrder --> HoldMic["Nhấn giữ nút mic\nđọc món + số lượng"]
        HoldMic --> Release["Thả nút mic\nkết thúc câu nói"]
        Release --> Review["Xem giỏ tạm\n+ phụ đề chép lại"]
        Review --> FixCheck{"Dòng nào\ncần chỉnh sửa?"}
        FixCheck -- "Sửa tay / xoá dòng" --> Review
        FixCheck -- "Không cần chỉnh" --> AnotherRound{"Đọc tiếp câu sau\nhay xác nhận?"}
        AnotherRound -- "Đọc tiếp" --> HoldMic
        AnotherRound -- "Xác nhận vào đơn" --> Confirm["Nhấn Xác nhận\nvào đơn"]
    end

    subgraph SpeechLane ["Speech Service"]
        Release --> ASR["Nhận diện giọng nói\nthành text"]
        ASR --> ASRCheck{"Nhận diện\nthành công?"}
        ASRCheck -- "Nhiễu / trống" --> RetryMsg["Thông báo không nghe được\nmời đọc lại"]
        RetryMsg --> HoldMic
        ASRCheck -- "OK" --> NLU["Tách tên món\nsố lượng + ghi chú"]
        NLU --> Match{"Khớp menu\n+ alias?"}
        Match -- "Tin cậy >= 85%" --> AutoFill["Tự điền giỏ tạm\nhighlight xanh"]
        Match -- "Tin cậy 60-84%" --> YellowFill["Điền + dấu chờ xác nhận\nhighlight vàng"]
        Match -- "Dưới 60% / không khớp" --> Suggest["Gợi ý <= 3 món gần nhất\nhoặc hỏi lại"]
        AutoFill --> Review
        YellowFill --> Review
        Suggest --> Review
        ASRCheck -- "Dịch vụ lỗi / quá 2 giây" --> Fallback["Thông báo lỗi\nchuyển nhập tay"]
    end

    subgraph SystemLane ["Hệ thống POS"]
        Confirm --> Validate{"Đơn hợp lệ\n& SL 1-99?"}
        Validate -- "Không hợp lệ" --> Review
        Validate -- "Hợp lệ" --> Commit["Ghi món vào đơn\ntính lại tổng tiền"]
        Commit --> SendKDS["Gửi món mới sang KDS"]
        SendKDS --> WriteAudit["Ghi audit log\n(phụ đề text, không audio)"]
        WriteAudit --> ShowSuccess["Thông báo thành công"]
    end

    subgraph ResultLane ["Kết quả"]
        ShowSuccess --> End([Kết thúc])
        Fallback --> End
    end
```

## Flow: voice-ordering-system-sequence (Sequence Diagram)

> Tương tác chi tiết giữa thu ngân, POS và các dịch vụ trong 1 phiên đặt món bằng giọng nói.

```mermaid
sequenceDiagram
    autonumber
    actor Cashier as Thu ngân
    participant POS_UI as POS UI
    participant Speech as Speech Service
    participant Menu as Menu Service
    participant API as POS API Gateway
    participant KDS as KDS
    participant Audit as Audit Log

    Note over Cashier,Speech: Sẵn sàng: quyền VOICE_ORDER, mic hoạt động, menu đã đồng bộ 24h
    Cashier->>POS_UI: Nhấn giữ nút mic, đọc món kèm số lượng
    POS_UI->>Speech: Stream audio trong lúc giữ nút (push-to-talk)
    Speech-->>POS_UI: Phụ đề text realtime
    Cashier->>POS_UI: Thả nút mic, kết thúc câu
    POS_UI->>Speech: Yêu cầu phân tích câu nói
    Speech->>Menu: Tra tên món, alias, giá theo bảng giá đang áp dụng
    Menu-->>Speech: Danh sách món khớp + mức tin cậy
    Speech-->>POS_UI: Danh sách (món, SL, ghi chú, tin cậy)
    alt Tin cậy >= 85%
        POS_UI-->>Cashier: Tự điền giỏ tạm, highlight xanh
    else Tin cậy 60-84%
        POS_UI-->>Cashier: Điền kèm dấu chờ xác nhận, highlight vàng
    else Dưới 60% hoặc không khớp
        POS_UI-->>Cashier: Hỏi lại, gợi ý tối đa 3 món gần nhất
    else Dịch vụ lỗi / quá 2 giây không phản hồi
        POS_UI-->>Cashier: Thông báo lỗi, chuyển nhập tay
    end
    Cashier->>POS_UI: Sửa giỏ tạm (chạm tay) rồi Nhấn Xác nhận vào đơn
    POS_UI->>API: Gửi danh sách món đã xác nhận (kèm mã phiên voice)
    API->>API: Kiểm tra quyền, trạng thái đơn, SL 1-99
    alt Hợp lệ
        API->>KDS: Gửi món mới sang bếp
        API->>Audit: Ghi log (người dùng, đơn, món, phụ đề text)
        API-->>POS_UI: OK, đơn cập nhật, tổng tiền mới
        POS_UI-->>Cashier: Hiển thị thành công
    else Không hợp lệ
        API-->>POS_UI: Lỗi 422, giữ nguyên giỏ tạm
        POS_UI-->>Cashier: Thông báo lỗi, mời sửa giỏ tạm
    end
    Note over POS_UI,Audit: Audio không lưu sau phiên - chỉ log text 90 ngày (NĐ 13/2023)
```
