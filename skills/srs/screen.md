---
name: screen
description: Tạo tài liệu đặc tả chi tiết các trường thông tin (Field Specs) và Bảng điều hướng (Navigation Map) chuẩn BA. AI tự động chạy ngầm Grep và Web Search để tối ưu hóa Rules & Lỗi và Luồng điều hướng. Đầu ra ghi vào docs/{feature}/srs/screens.md.
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebSearch]
user-invocable: true
argument-hint: "\"<desc>\" --feature <slug> [--update]"
---

# Skill: /screen — Standalone Screen Specification Generator

## 1. Goal
Hỗ trợ Business Analyst (BA) thiết lập tài liệu đặc tả chi tiết cho giao diện người dùng (UI Component) [1]. AI sẽ chạy ngầm các tác vụ rà soát để tự động thiết lập bảng đặc tả trường thông tin tinh giản (Field Specs) [1], đồng thời xây dựng bảng Bản đồ điều hướng (Navigation Map) mô tả rõ luồng di chuyển giữa các màn hình/dialog [1], triệt tiêu rủi ro bỏ sót trường hợp xử lý (miss case) [1].

## 2. Constraints (Ràng buộc cứng)
1. **Quy định tệp tin đầu ra:**
   - Kết quả đặc tả màn hình được ghi hoặc append vào một tệp duy nhất: `docs/{feature}/srs/screens.md` [1].
   - Mỗi màn hình được lưu dưới dạng một phân đoạn riêng bắt đầu bằng tiêu đề `## Screen: {Tên màn hình}` (Do đó, tệp không cần cột Tên màn hình trong bảng để tránh lặp dữ liệu) [1].
2. **Cơ chế Phân tích ngầm (Background Analysis):**
   - **Internal Scan:** Sử dụng `Grep` để tìm kiếm các thực thể hoặc định nghĩa dữ liệu liên quan trong `docs/` nhằm đảm bảo tên trường thông tin, kiểu dữ liệu và logic đồng bộ hoàn toàn với cơ sở dữ liệu hiện có [1].
   - **External Scan:** Sử dụng `WebSearch` để tìm kiếm tiêu chuẩn thiết kế biểu mẫu (Form UI Best Practices) cho loại màn hình này.
3. **Quy tắc thiết lập Rules & Lỗi tinh giản (Rigor UI/UX Rules):**
   - **Cấu trúc bảng đặc tả trường thông tin gồm đúng 5 cột:** `Tên trường thông tin`, `Control Type`, `Bắt buộc`, `Mặc định`, `Rules & Lỗi` [1].
   - **Cột Rules & Lỗi phải viết lồng trực tiếp:**
     - KHÔNG sử dụng các nhãn bôi đậm (như `**Kiểm tra rỗng:**`, `**Giá trị biên:**`, `**Ràng buộc phụ thuộc:**`, `**Định dạng Regex:**`) làm tiêu đề hoặc tiền tố trong các gạch đầu dòng [1].
     - Viết quy tắc logic dưới dạng các gạch đầu dòng ngắn, rõ ràng [1].
     - **ỨNG VỚI MỖI RULE:** Nếu quy tắc đó có phát sinh lỗi hoặc cần xử lý vi phạm, phải viết ngay dòng thông báo lỗi tương ứng trực tiếp ở phía dưới quy tắc đó, được bôi nghiêng và bắt đầu bằng tiền tố `*Lỗi:* "[Nội dung thông báo lỗi cụ thể]"` [1].
     - Đảm bảo bao phủ đầy đủ: điều kiện rỗng [3], giá trị biên [3], định dạng Regex [3] và các ràng buộc phụ thuộc giữa các trường.
4. **Bản đồ điều hướng bắt buộc (Navigation Map Requirement):**
   - AI bắt buộc phải tạo thêm phân đoạn `### 2. Điều hướng (Navigation Map)` ngay dưới phần đặc tả trường thông tin [1].
   - Bảng điều hướng phải gồm chính xác 4 cột: `Từ màn hình`, `Hành động`, `Đến màn hình`, `Ghi chú` [1].
5. **Cơ chế Approval Gate:**
   - **L1 Plan:** Bản xem trước (Preview) phải thể hiện rõ: Kế hoạch tạo/cập nhật `screens.md`, tóm tắt danh sách trường thông tin và các luồng điều hướng cốt lõi [1]. Chờ xác nhận `Y/n` từ người dùng [1].
   - **L2 Diff:** Khi cập nhật màn hình đã tồn tại (có flag `--update`), hiển thị định dạng unified diff của phần bảng thay đổi [1]. Chờ xác nhận `Y/n` [1].
   - **L3 Iterate (Không áp dụng):** Bỏ qua L3 [1].
6. **Changelog Routing (v2.6):**
   - Tệp `screens.md` sử dụng slim frontmatter (không chứa trường `changelog` riêng) [1].
   - Nhật ký thay đổi từ lệnh `/screen` phải được cập nhật trực tiếp vào trường `changelog` của tệp đặc tả cha: `docs/{feature}/srs/spec.md` [1].
   - Tiền tố ghi nhận bắt buộc: `[screens]` [1].
   - Định dạng dòng log: `- YYYY-MM-DD | /screen | [screens] added/updated specification and navigation for <screen-name>` [1].

## 3. Inputs
Cú pháp lệnh hợp lệ:
- Khởi tạo lần đầu: `/screen "<desc>" --feature <slug>`
- Cập nhật tài liệu: `/screen "<desc>" --feature <slug> --update`

## 4. Approach (Quy trình thực hiện chi tiết)

### Bước 1: Quét ngầm hệ thống nội bộ (Background Internal Scan)
- Sử dụng `Grep` quét thư mục `docs/` để tìm xem các trường dữ liệu trên màn hình này đã được định nghĩa ở đâu chưa để đồng bộ tên gọi [1].

### Bước 2: Nghiên cứu ngầm tiêu chuẩn giao diện (Background Web Search)
- Sử dụng `WebSearch` tìm kiếm tiêu chuẩn thiết kế biểu mẫu (Form design patterns) và luồng người dùng (User Flows) phổ biến của tính năng tương ứng.

### Bước 3: L1 Plan Review
- Xuất kế hoạch ghi tệp, liệt kê danh sách màn hình, các trường thông tin và tóm tắt luồng điều hướng, chờ xác nhận `Y/n` từ người dùng [1].

### Bước 4: Soạn thảo và Ghi tệp tin
- Tạo hoặc append phân đoạn đặc tả màn hình vào tệp `screens.md` gồm đầy đủ bảng đặc tả trường thông tin, bảng điều hướng và lưu ý UI/UX nâng cao [1].

---

## 5. Cấu trúc Output chuẩn hóa (Template đầu ra cho AI)

Nội dung ghi vào tệp `docs/{feature}/srs/screens.md` tuân thủ chính xác cấu trúc sau:

```markdown
## Screen: [Tên màn hình cụ thể]

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| [Ví dụ: Tỉnh/Thành phố] | [Dropdown] | Có | [Trống] | - Bắt buộc chọn một tùy chọn.<br/>*Lỗi:* "Vui lòng chọn Tỉnh/Thành phố".<br/>- Danh sách tùy chọn lấy từ hệ thống (API danh mục Tỉnh/Thành phố Việt Nam). |
| [Ví dụ: Quận/Huyện] | [Dropdown] | Có | [Vô hiệu hóa] | - Chỉ được kích hoạt (enabled) sau khi đã chọn giá trị tại trường `Tỉnh/Thành phố`. Nếu thay đổi lại `Tỉnh/Thành phố`, trường này tự động xóa trắng giá trị và tải lại danh sách mới tương ứng.<br/>- Bắt buộc chọn một tùy chọn.<br/>*Lỗi:* "Vui lòng chọn Quận/Huyện". |
| [Ví dụ: Số điện thoại] | [Textbox] | Có | [Trống] | - Không được để trống [3].<br/>*Lỗi:* "Số điện thoại không được để trống" [1].<br/>- Phải nhập chính xác 10 chữ số [3].<br/>*Lỗi:* "Số điện thoại phải chứa 10 chữ số" [1].<br/>- Định dạng: `^0[3|5|7|8|9]\d{8}$` (Đầu số Việt Nam hợp lệ) [3].<br/>*Lỗi:* "Số điện thoại phải bắt đầu bằng 03, 05, 07, 08 hoặc 09" [1]. |
| [Ví dụ: Mật khẩu] | [Textbox - Password] | Có | [Trống] | - Không được để trống [3].<br/>*Lỗi:* "Mật khẩu không được để trống" [1].<br/>- Độ dài từ 8 đến 20 ký tự [3].<br/>- Phải chứa ít nhất 1 chữ cái viết hoa, 1 chữ cái viết thường và 1 chữ số [3].<br/>*Lỗi:* "Mật khẩu phải dài từ 8-20 ký tự, bao gồm ít nhất 1 chữ hoa, 1 chữ thường và 1 số" [1]. |
| [Ví dụ: Nút Thanh toán] | [Button] | Có | [N/A] | - Chỉ cho phép click sau khi tất cả các trường bắt buộc đã điền hợp lệ.<br/>- Chặn click liên tục (Disable button ngay sau lượt click đầu tiên cho đến khi có phản hồi từ API hệ thống thanh toán) để tránh trùng lặp giao dịch.<br/>*Lỗi:* "Giao dịch không thành công, vui lòng kiểm tra lại số dư và thử lại" [1]. |

### 2. Điều hướng (Navigation Map)
*Bản đồ mô tả luồng di chuyển và tương tác giữa các màn hình, dialog hoặc trạng thái giao diện của tính năng:*

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| DS Phương thức TT (`payment-methods`) | Nhấn Card NH liên kết | Chi tiết NH liên kết | [Ví dụ: Xem chi tiết ngân hàng đang kết nối] |
| DS Phương thức TT | Nhấn `⋮` -> **Hủy kết nối** | Dialog Xác nhận hủy | [Ví dụ: Mở hộp thoại xác nhận từ danh sách] |
| Chi tiết NH liên kết | Nhấn **Hủy kết nối** | Dialog Xác nhận hủy | [Ví dụ: Mở hộp thoại xác nhận từ trang chi tiết] |
| Chi tiết NH liên kết | Nhấn **Quay lại** | DS Phương thức TT | |
| Dialog Xác nhận hủy | Nhấn **Hủy** | Đóng dialog (giữ nguyên màn hình) | |
| Dialog Xác nhận hủy | Hủy thành công | Danh sách/Chi tiết | [Ví dụ: Trở lại danh sách sau khi ngắt kết nối thành công] |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)
- **Trạng thái tải dữ liệu (Loading State):** Khi người dùng thực hiện hành động gửi yêu cầu, hiển thị spinner hoặc skeleton loader tương ứng.
- **Bảo mật dữ liệu:** Trường Mật khẩu có biểu tượng Mắt (Ẩn/Hiện mật khẩu) [3]. Mật khẩu nhập vào phải được ẩn dưới dạng dấu chấm mặc định.
- **Tối ưu hóa bàn phím trên thiết bị di động:** Trường Số điện thoại phải tự động kích hoạt bàn phím số (Numeric keyboard) khi người dùng chạm vào.