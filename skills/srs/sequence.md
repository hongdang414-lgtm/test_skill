---
name: sequence
description: Phân tích nghiệp vụ hệ thống và sinh Sequence Diagram (biểu đồ tuần tự) mô tả tương tác chi tiết giữa các thành phần. Output được ghi hoặc append vào file duy nhất docs/{feature}/srs/flows.md.
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
user-invocable: true
argument-hint: "\"<desc>\" --feature <slug> [--update]"
---

# Skill: /sequence — Sequence Diagram Generator

## 1. Goal
Tự động hóa việc phân tích nghiệp vụ, thiết kế tương tác tuần tự mức kỹ thuật (Technical white-box) dưới dạng sơ đồ Mermaid.js [1], sau đó cập nhật trực tiếp vào phân đoạn tương ứng trong tệp đặc tả luồng nghiệp vụ của tính năng (`docs/{feature}/srs/flows.md`) [1].

## 2. Constraints (Ràng buộc cứng)
1. **Chỉ ghi vào một file duy nhất:** Luôn ghi nhận output vào `docs/{feature}/srs/flows.md` [1]. KHÔNG tự ý tạo file standalone hoặc đổi đường dẫn [1]. KHÔNG nhúng sơ đồ sequence vào các file Use Case (`docs/{feature}/usecases/...`) nhằm giữ vững ranh giới trừu tượng (Abstraction Boundary) [1].
2. **Từ chối nếu vi phạm Abstraction Boundary:** Nếu người dùng yêu cầu vẽ sequence diagram trực tiếp vào file Use Case, hãy từ chối lịch sự và giải thích: *"Sequence Diagram biểu diễn tương tác kỹ thuật mức hệ thống (Technical white-box), trong khi Use Case biểu diễn nghiệp vụ dạng hộp đen (Business black-box). Sơ đồ này sẽ được lưu tại `srs/flows.md`."* [1]
3. **Cơ chế Approval Gate:**
   - **L1 Plan:** Trước khi tiến hành Write/Append vào file, bắt buộc in ra bản xem trước (Preview) gồm: Đường dẫn file mục tiêu, hành động (tạo mới/append), danh sách các actor/system tham gia, và tóm tắt kịch bản. Chờ người dùng phản hồi `Y/n` [1].
   - **L2 Diff:** Nếu sử dụng tham số `--update` trên tệp đã tồn tại, hiển thị cấu trúc thay đổi (unified diff) của đoạn mã Mermaid cũ và mới [1]. Chờ xác nhận `Y/n` [1].
   - **L3 Iterate (Không áp dụng):** Do terminal/chat không thể hiển thị trực quan sơ đồ Mermaid đã render, bỏ qua L3 [1]. Người dùng sẽ tự xem sơ đồ từ trình đọc file (Obsidian/GitHub/IDE) [1]. Muốn điều chỉnh, người dùng sẽ chạy lại lệnh kèm flag `--update` [1].
4. **Refuse nếu thiếu `--update`:** Nếu file `flows.md` hoặc luồng (`## Flow: <Title>`) tương tự đã tồn tại mà lệnh gọi thiếu flag `--update`, từ chối thực hiện và thông báo lỗi rõ ràng [1].
5. **Changelog Routing (v2.6):**
   - File `flows.md` sử dụng slim frontmatter (không chứa trường `changelog` bên trong) [1].
   - Mọi lịch sử cập nhật từ lệnh `/sequence` phải được ghi nhận (route) trực tiếp lên file đặc tả cha: `docs/{feature}/srs/spec.md` [1].
   - Prefix ghi nhận lịch sử bắt buộc là: `[flows]` [1].
   - Định dạng dòng log: `- YYYY-MM-DD | /sequence | [flows] added <flow-name> sequence, <n> actors` [1].

## 3. Inputs
Cú pháp lệnh hợp lệ:
- Khởi tạo lần đầu: `/sequence "<desc>" --feature <slug>`
- Cập nhật luồng đã có: `/sequence "<desc>" --feature <slug> --update`

## 4. Context (Dynamic)
Các lệnh kiểm tra trạng thái tệp tin thực tế trong quá trình thực thi:
- Ngày hiện tại: !`date +%Y-%m-%d` [1]
- Danh sách các tệp flows hiện có: !`for d in docs/*/srs/flows.md; do [ -f "$d" ] && echo "$d"; done` [1]
- Các tệp spec.md hoạt động để route changelog: !`for d in docs/*/srs/spec.md; do [ -f "$d" ] && echo "$d"; done` [1]

## 5. Approach (Quy trình thực hiện)
1. **Parse & Validate:** 
   - Trích xuất mô tả nghiệp vụ `<desc>`, mã tính năng `--feature <slug>`, và trạng thái `--update` [1].
   - Kiểm tra xem thư mục `docs/{slug}/` đã tồn tại chưa. Nếu chưa, cảnh báo người dùng tạo cấu trúc tính năng trước.
2. **Auto-detect & Align:** 
   - Đọc tệp `docs/{slug}/srs/spec.md` hoặc các file sơ đồ usecase hiện có để thu thập thông tin về Actors, hệ thống bên thứ ba, và các thực thể nghiệp vụ nhằm đồng nhất danh sách định danh (Identifiers).
3. **Phân tích thiết kế hệ thống:**
   - Xác định rõ các bên tham gia: Người dùng (Actor), Giao diện (UI/Frontend), Cổng xử lý (API Gateway/Backend), Cơ sở dữ liệu (DB), Hệ thống thứ ba (Payment Gateway, SMS/Email service...).
   - Xác định luồng thành công (Happy path) và các luồng rẽ nhánh lỗi (Exception paths: Lỗi xác thực, hết hạn OTP, lỗi DB...).
4. **Soạn thảo mã Mermaid.js:**
   - Đảm bảo tuân thủ cú pháp `sequenceDiagram` [1].
   - Sử dụng `autonumber` để tự động đánh số thứ tự bước thực hiện [1].
   - Thiết lập các khối điều kiện `alt / else` hoặc `opt` rõ ràng cho các nhánh xử lý ngoại lệ [1].
5. **L1 Plan Approval:**
   - Hiển thị thông tin tóm tắt kế hoạch ghi file [1].
   - Đưa ra câu hỏi xác nhận: `Bạn có đồng ý thực hiện ghi file flows.md này không? (Y/n)` [1].
6. **Thực hiện Write / Append:**
   - Nếu tạo mới: Khởi tạo file với slim frontmatter và thêm tiêu đề `## Flow: {Tên luồng nghiệp vụ}` kèm khối mã Mermaid.js.
   - Nếu append: Định vị cuối file `flows.md`, chèn thêm tiêu đề `## Flow: {Tên luồng nghiệp vụ}` và khối mã mới.
7. **Changelog Routing:**
   - Đọc tệp `docs/{slug}/srs/spec.md`, định vị khu vực `changelog` trong frontmatter [1].
   - Chèn dòng nhật ký thay đổi với tiền tố `[flows]` [1].
8. **Báo cáo kết quả:**
   - Thông báo file đã được ghi thành công [1].
   - Cung cấp hướng dẫn ngắn gọn cách người dùng có thể xem sơ đồ bằng các công cụ tương thích (như Obsidian, GitHub Markdown Reader) [1].

## 6. Mermaid Syntax Reference (Mẫu chuẩn)
Sử dụng cấu trúc cú pháp tuần tự an toàn dưới đây để tham chiếu:
```mermaid
sequenceDiagram
    autonumber
    actor User as Khách hàng
    participant UI as Giao diện Web
    participant API as API Gateway
    database DB as Cơ sở dữ liệu

    User->>UI: Yêu cầu thực hiện hành động
    UI->>API: Gửi yêu cầu xác thực (Request)
    API->>DB: Kiểm tra trạng thái tài khoản
    alt Tài khoản hợp lệ
        DB-->>API: Trả về thông tin (Active)
        API-->>UI: Xác thực thành công (Token)
        UI-->>User: Hiển thị màn hình chính
    else Tài khoản bị khóa / Không tồn tại
        DB-->>API: Trả về lỗi
        API-->>UI: Thông báo lỗi xác thực
        UI-->>User: Hiển thị thông báo "Tài khoản không hợp lệ"
    end