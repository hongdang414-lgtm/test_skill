---
type: srs-screens
feature: voice-ordering
updated: 2026-08-14
---

## Screen: voice-order-panel (Panel Voice Order — overlay trên màn hình bán hàng)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Nút mic (bấm giữ) | Button (hold) | — | Hiển thị theo quyền | - Chỉ hiển thị khi tài khoản có quyền `VOICE_ORDER`; disable kèm tooltip khi thiếu 1 trong các điều kiện: mic chưa kết nối, đơn không hợp lệ, thực đơn chưa đồng bộ 24 giờ.<br/>*Lỗi:* "Thực đơn chưa đồng bộ xong, vui lòng nhập tay."<br/>- Chỉ ghi âm trong lúc giữ nút (push-to-talk); giữ quá 60 giây tự cắt phiên ghi và xử lý đoạn đã ghi được.<br/>- Tạm disable 5 giây sau mỗi lần lỗi để tránh giữ lại ngay. |
| Phụ đề trực tiếp (live transcript) | Text (readonly, realtime) | — | Trống | - Hiển thị text chép lại trong lúc giữ nút mic, làm mới theo từng câu.<br/>- Câu không nhận diện được (nhiễu/trống): hiển thị thông báo mời đọc lại thay vì text rỗng.<br/>*Lỗi:* "Không nghe rõ, mời đọc lại hoặc chuyển nhập tay." |
| Giỏ tạm (danh sách món) | List (editable) | Có | Trống | - Mỗi dòng: tên món khớp, số lượng (1–99), ghi chú (≤ 100 ký tự), trạng thái xác nhận: **xanh** = tin cậy ≥ 85%, **vàng** = tin cậy 60–84% chờ xác nhận.<br/>- Dòng vàng cho phép sửa tên món/SL/ghi chú bằng chạm tay; còn dòng vàng nào thì không được "Xác nhận vào đơn".<br/>*Lỗi:* "Vẫn còn món chưa xác nhận, vui lòng kiểm tra các dòng đánh dấu vàng."<br/>- Món không khớp thực đơn: không tạo dòng mới, chỉ hiện gợi ý tối đa 3 món gần nhất.<br/>*Lỗi:* "Không tìm thấy món trong thực đơn."<br/>- Món đã có trong đơn và không kèm ghi chú mới: tăng số lượng dòng hiện có thay vì tạo dòng mới. |
| Tổng tiền tạm tính | Text (readonly) | — | 0 VNĐ | - Tính lại realtime theo bảng giá đang áp dụng mỗi khi giỏ tạm thay đổi; không bao gồm khuyến mãi, ngang giá với nhập tay. |
| Nút "Xác nhận vào đơn" | Button (Primary) | Có | Disable | - Chỉ enable khi giỏ tạm có ≥ 1 dòng và không còn dòng vàng.<br/>- Disable ngay sau lượt click đầu tiên đến khi có phản hồi (tránh gửi bếp trùng lặp).<br/>*Lỗi:* "Giao dịch chưa thành công, đơn giữ nguyên, vui lòng thử lại." |
| Nút "Hủy giỏ tạm" | Button (Secondary) | — | — | - Hiện xác nhận lần hai trước khi hủy; hủy = đóng panel, đơn không đổi. |
| Thanh trạng thái phiên | Status indicator | — | Sẵn sàng | - Các trạng thái: Sẵn sàng → Đang nghe → Đang xử lý (≤ 2 giây) → Chờ xác nhận hoặc Lỗi.<br/>- Quá 2 giây không phản hồi: tự chuyển Lỗi và mở Dialog lỗi phiên voice. |

### 2. Điều hướng (Navigation Map)
*Bản đồ mô tả luồng di chuyển và tương tác giữa các màn hình, dialog hoặc trạng thái giao diện của tính năng:*

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Màn hình bán hàng | Nhấn giữ nút mic | Panel Voice Order | Overlay, không rời màn hình đơn |
| Panel Voice Order | Thả nút mic, xuất hiện dòng tin cậy 60–84% | Dialog xác nhận món chưa chắc | Tự mở cho dòng cần xác nhận đầu tiên |
| Panel Voice Order | Dịch vụ lỗi / quá 2 giây không phản hồi | Dialog lỗi phiên voice | Kèm nguyên nhân lỗi |
| Panel Voice Order | Nhấn "Xác nhận vào đơn" | Màn hình bán hàng | Đóng panel + thông báo thành công, đơn cập nhật tiền + gửi bếp |
| Panel Voice Order | Nhấn "Hủy giỏ tạm" (đã xác nhận lần hai) | Màn hình bán hàng | Đơn không đổi |
| Panel Voice Order | Chạm vào dòng màu vàng | Dialog xác nhận món chưa chắc | Mở lại để sửa dòng đó |

### 3. Các điểm lưu ý trên trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- **Trạng thái chờ (Loading):** hiển thị "Đang xử lý" ngay khi thả nút mic, có kết quả trong tối đa 2 giây theo mục tiêu chất lượng.
- **Phản hồi âm thanh:** tiếng "tick" nhẹ mỗi khi một dòng món được điền, giúp thu ngân không phải nhìn màn hình liên tục.
- **Im lặng giữa các câu:** quá 10 giây không đọc tiếp vẫn giữ phiên; phụ đề mờ dần kèm gợi ý "Tiếp tục đọc hoặc Xác nhận".
- **Môi trường ồn:** khuyến nghị dùng mic tai nghe; hiện ghi chú khuyến nghị lần đầu mở panel trong ngày.
- **Khả năng tiếp cận:** mọi trạng thái màu (xanh/vàng) đi kèm icon + nhãn text, không phân biệt chỉ bằng màu.
- **Đóng panel đột ngột (nút X):** cảnh báo "Giỏ tạm chưa xác nhận sẽ bị hủy" trước khi đóng.

## Screen: dialog-confirm-item (Dialog xác nhận món chưa chắc — tin cậy 60–84%)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Câu gốc chép lại | Text (readonly) | — | Từ phụ đề | - Hiển thị nguyên văn câu nói để thu ngân đối chiếu, không cho chỉnh sửa. |
| Món hệ thống hiểu được | Dropdown (single) | Có | Gợi ý tương đồng cao nhất | - Danh sách tối đa 3 món gần nhất, sắp xếp theo độ tương đồng giảm dần, hiển thị dạng "Khớp 78%".<br/>- Bắt buộc chọn 1 món hoặc "Không món nào đúng".<br/>*Lỗi:* "Vui lòng chọn món phù hợp hoặc bỏ dòng này."<br/>- Khi độ tương đồng bằng nhau: ưu tiên món bán chạy của cửa hàng xếp trước. |
| Số lượng | Stepper | Có | SL từ câu nói; 1 nếu không xác định được | - Giới hạn min = 1, max = 99.<br/>*Lỗi:* "Số lượng chỉ từ 1 đến 99." |
| Ghi chú | Textbox | Không | Trống | - Tối đa 100 ký tự.<br/>*Lỗi:* "Ghi chú quá dài, đã tự cắt còn 100 ký tự." |
| Nút "Chọn món này" | Button (Primary) | Có | Disable | - Chỉ enable khi đã chọn món hợp lệ; lưu dòng về giỏ tạm và chuyển trạng thái xanh. |
| Nút "Bỏ dòng" | Button (Secondary) | — | — | - Xóa dòng khỏi giỏ tạm, không cần xác nhận lại. |
| Nút "Nhập tay" | Link button | — | — | - Đóng dialog, focus vào tên món trong giỏ tạm để sửa bằng tay. |

### 2. Điều hướng (Navigation Map)
*Bản đồ mô tả luồng di chuyển và tương tác giữa các màn hình, dialog hoặc trạng thái giao diện của tính năng:*

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog xác nhận món | Nhấn "Chọn món này" | Panel Voice Order | Dòng chuyển trạng thái xanh |
| Dialog xác nhận món | Nhấn "Bỏ dòng" | Panel Voice Order | Nếu còn dòng vàng khác, mở tiếp dialog cho dòng kế tiếp |
| Dialog xác nhận món | Nhấn "Nhập tay" | Panel Voice Order | Focus sửa tên món trong giỏ tạm |
| Dialog xác nhận món | Nhấn nút X / vùng ngoài dialog | Panel Voice Order | Giữ nguyên dòng vàng, không tự bỏ dòng |

### 3. Các điểm lưu ý trên trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- Dialog **không chặn đọc tiếp**: thu ngân có thể bấm ra ngoài để đọc câu mới, dialog tự thu nhỏ thành badge số dòng chờ xác nhận.
- Điểm tương đồng hiển thị bằng ngôn ngữ dễ hiểu ("Khớp 78%"), không dùng thuật ngữ kỹ thuật (confidence score).
- Sau khi xử lý dòng cuối cùng, dialog tự đóng và focus về nút "Xác nhận vào đơn".

## Screen: dialog-voice-error (Dialog lỗi phiên voice / chuyển nhập tay)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Thông điệp lỗi | Text (readonly) | Có | Theo nguyên nhân | - 3 nhóm thông điệp theo nguyên nhân: "Không nghe rõ" (nhiễu/trống), "Dịch vụ nhận diện đang bận" (quá 2 giây không phản hồi), "Mất kết nối dịch vụ" (lỗi mạng).<br/>- Mỗi nhóm kèm đúng 1 gợi ý xử lý tương ứng, không dùng thông báo chung chung.<br/>*Lỗi:* hiển thị đúng nhóm nguyên nhân đã xảy ra. |
| Số dòng sẽ bị hủy | Text (readonly) | — | 0 | - Nếu giỏ tạm còn dòng đã điền: hiển thị "Có {N} dòng đã điền sẽ bị hủy nếu chuyển nhập tay". |
| Nút "Thử lại" | Button (Primary) | Có | — | - Giữ nguyên giỏ tạm hiện tại, quay về trạng thái Sẵn sàng.<br/>- Sau 3 lần lỗi liên tiếp: nút "Chuyển nhập tay" được đổi thành nút chính (Primary). |
| Nút "Chuyển nhập tay" | Button (Secondary) | — | — | - Đóng toàn bộ panel voice và hủy giỏ tạm chưa xác nhận (kể cả dòng xanh chưa commit), focus ô tìm món trên màn hình bán hàng. |

### 2. Điều hướng (Navigation Map)
*Bản đồ mô tả luồng di chuyển và tương tác giữa các màn hình, dialog hoặc trạng thái giao diện của tính năng:*

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Dialog lỗi phiên voice | Nhấn "Thử lại" | Panel Voice Order | Giữ nguyên giỏ tạm |
| Dialog lỗi phiên voice | Nhấn "Chuyển nhập tay" | Màn hình bán hàng | Hủy giỏ tạm, cảnh báo số dòng bị hủy nếu > 0 |
| Panel Voice Order | Lỗi liên tiếp lần thứ 3 | Dialog lỗi phiên voice | "Chuyển nhập tay" thành nút chính |

### 3. Các điểm lưu ý trên trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- Không tự đóng dialog theo thời gian — thu ngân có thể đang xử lý việc khác, tự đóng đồng nghĩa hủy giỏ tạm.
- Ở lần lỗi thứ 3, thêm một dòng đề xuất: "Thử mic tai nghe hoặc kiểm tra mạng cửa hàng".
- Mọi lỗi đều ghi vào dashboard chất lượng nhận diện của quản lý cửa hàng (kèm nguyên nhân), không chặn nghiệp vụ bán hàng.
