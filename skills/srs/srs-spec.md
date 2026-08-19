---
name: srs-spec
description: Tạo đặc tả chi tiết tính năng (Feature Spec) chuẩn BA với bảng Quy tắc nghiệp vụ (Business Rules) gồm 4 cột nghiêm ngặt (#, Quy tắc, Mô tả, Xử lý vi phạm). AI tự động quét ngầm (Grep/Web Search), vẽ sơ đồ và TỰ ĐỘNG GỌI `/screen` để sinh đặc tả màn hình độc lập. Đầu ra ghi vào docs/{feature}/srs/spec.md.
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, WebSearch]
user-invocable: true
argument-hint: "\"<desc>\" --feature <slug> [--update]"
---

# Skill: /srs-spec — Modular Feature Specification (Parent Skill)

## 1. Goal
Hỗ trợ Business Analyst (BA) thiết lập bộ tài liệu đặc tả tính năng hoàn chỉnh theo mô hình Modular [1]. Skill này đóng vai trò là **Skill Cha (Parent Skill)**: chịu trách nhiệm khởi tạo tài liệu tổng quan (`spec.md`), vẽ sơ đồ tương tác (`flows.md`), sau đó **gọi lồng Skill Con `/screen`** để xử lý chi tiết giao diện màn hình và lưu vào tệp độc lập (`screens.md`) nhằm đảm bảo tính cấu trúc và nhất quán [1].

## 2. Constraints (Ràng buộc cứng)
1. **Ranh giới tài liệu (Abstraction Boundary):**
   - Nội dung tổng quan nghiệp vụ (Mục 1-4) và Bảng quy tắc nghiệp vụ cốt lõi (Mục 5) ghi vào: `docs/{feature}/srs/spec.md` [1].
   - Sơ đồ tương tác (Mermaid) ghi vào: `docs/{feature}/srs/flows.md` [1].
   - **Đặc tả màn hình:** KHÔNG được viết trực tiếp vào `spec.md` [1]. AI bắt buộc phải gọi Skill `/screen` để sinh tài liệu riêng tại `docs/{feature}/srs/screens.md` [1].
   - Tại `spec.md`, AI chỉ đặt link liên kết hoặc nhúng tệp dưới dạng transclusion `![[screens.md]]` [1].
2. **Cơ chế gọi lồng Skill (Skill Delegation):**
   - AI bắt buộc phải phân tách phần mô tả giao diện từ yêu cầu của người dùng, tự động định hình các tham số và thực thi lệnh `/screen "<mô tả màn hình>" --feature <slug> [--update]` ngay trong lượt xử lý [1].
3. **Cơ chế Approval Gate:**
   - **L1 Plan:** Bản xem trước (Preview) phải thể hiện rõ: Kế hoạch tạo `spec.md`, kế hoạch tạo `flows.md`, và **kế hoạch gọi lồng lệnh `/screen`** [1]. Chờ xác nhận `Y/n` từ người dùng [1].
   - **L2 Diff:** Khi cập nhật có flag `--update`, hiển thị định dạng unified diff của tệp `spec.md` [1]. Chờ xác nhận `Y/n` [1].
   - **L3 Iterate (Không áp dụng):** Bỏ qua L3 [1].
4. **Changelog Routing (v2.6):**
   - Ghi nhận lịch sử của cả quá trình khởi tạo `/srs-spec` trực tiếp vào trường `changelog` của tệp `docs/{feature}/srs/spec.md` [1].

## 3. Inputs
Cú pháp lệnh hợp lệ:
- Khởi tạo lần đầu: `/srs-spec "<desc>" --feature <slug>`
- Cập nhật tài liệu: `/srs-spec "<desc>" --feature <slug> --update`

## 4. Approach (Quy trình thực hiện chi tiết)

### Bước 1: Quét ngầm hệ thống & Nghiên cứu (Background Scan)
- Chạy ngầm `Grep` trên `docs/` để tìm các từ khóa liên quan đến nghiệp vụ này, kiểm tra xem có xung đột logic hoặc ràng buộc trạng thái nào với các tính năng cũ đã có hay không [1].
- Chạy ngầm `WebSearch` để thu thập các quy định pháp lý, tiêu chuẩn bảo mật (GDPR, PCI-DSS...), giới hạn số lượng giao dịch, chính sách timeout hoặc hạn mức (quotas) tiêu chuẩn ngành cho tính năng này.

### Bước 2: Phân tích Quy tắc Nghiệp vụ nghiêm ngặt (Rigor Business Rules Extraction)
AI bắt buộc phải phân tách mô tả nghiệp vụ thành các luật nguyên tử (atomic statements) theo 5 nhóm bắt buộc:
1. **Constraint (Luật ràng buộc):** Giới hạn cứng (ví dụ: tuổi từ 18-65, nhập sai tối đa 5 lần).
2. **Derivation (Luật suy diễn/Tính toán):** Công thức, điều kiện chiết khấu, thuế suất.
3. **Action Enabler (Luật kích hoạt):** Điều kiện để một hành động/quy trình được phép bắt đầu.
4. **State Transition (Luật trạng thái):** Bản đồ chuyển đổi trạng thái của thực thể (ví dụ: Chờ thanh toán -> Đang giao).
5. **Data Validation (Luật định dạng dữ liệu):** Cấu trúc dữ liệu nhập vào (ví dụ: regex, độ dài, checksum).

### Bước 3: Thiết kế sơ đồ tương tác (Diagram Selection)
- Đánh giá nghiệp vụ để chọn Sequence Diagram hoặc Activity Diagram dựa trên quy tắc tại `diagram-selection.md` [1].
- Thiết lập sơ đồ tương tác và ghi vào tệp `docs/{feature}/srs/flows.md` [1].

### Bước 4: L1 Plan Review (Báo cáo kế hoạch gọi lồng)
- Xuất kế hoạch ghi tệp `spec.md`, `flows.md`, và nêu rõ tham số sẽ dùng để **gọi tiếp Skill `/screen`** [1]. Chờ người dùng xác nhận `Y/n` [1].

### Bước 5: Soạn thảo tệp đặc tả chính (spec.md)
- Tạo tệp `docs/{feature}/srs/spec.md` theo cấu trúc chuẩn (chứa Tổng quan nghiệp vụ có bổ sung Quy tắc nghiệp vụ chi tiết, Sơ đồ, và liên kết tới đặc tả màn hình) [1].

### Bước 6: Kích hoạt gọi lồng Skill Con (/screen)
- Tự động thực thi lệnh: `/screen "<phần mô tả màn hình tách lọc từ yêu cầu>" --feature <slug> [--update]` để tạo hoặc cập nhật tệp `screens.md` độc lập [1].

### Bước 7: Changelog Routing
- Ghi nhật ký thay đổi vào phần frontmatter của `spec.md` [1].

---

## 5. Cấu trúc Output chuẩn hóa của tệp Đặc tả chính (spec.md)

Nội dung ghi vào tệp `docs/{feature}/srs/spec.md` tuân thủ cấu trúc sau:

```markdown
# Đặc tả tính năng: [Tên tính năng]

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi dự án** | [Mô tả rõ ràng mục đích của tính năng và ranh giới giải quyết vấn đề] |
| **2. Actors (Tác nhân)** | [Danh sách các vai trò, hệ thống tham gia vào luồng xử lý này] |
| **3. Pre-conditions (Điều kiện tiên quyết)** | [Các điều kiện bắt buộc phải thỏa mãn trước khi bắt đầu luồng] |
| **4. Expected Results (Kết quả mong đợi)** | **Happy Path (Luồng xử lý chính):**<br/>1. Bước 1...<br/>2. Bước 2...<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ - Đã bao phủ các trường hợp biên phát hiện từ phân tích ngầm):**<br/>- **[Nhánh rẽ A]:** Nếu [Điều kiện A] xảy ra, hệ thống sẽ...<br/>- **[Ngoại lệ B]:** Nếu [Lỗi B] xảy ra, hệ thống hiển thị thông báo lỗi... |

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | [Loại luật - Tên quy tắc] | [Mô tả chi tiết luật nghiệp vụ] | [Cách hệ thống xử lý khi người dùng vi phạm luật này] |

## 3. Sơ đồ tương tác (Interaction Diagram)
*(Sơ đồ dưới đây được lưu tại `docs/{feature}/srs/flows.md`)*

```mermaid
[Mã nguồn Mermaid.js được tự động sinh theo quy tắc chọn loại sơ đồ dựa trên phân tích ngầm]
```

## 6. References
- `@../../rules/approval-gate.md` [1]
- `@../../rules/diagram-selection.md` [1]
- `@../screen/SKILL.md` (Skill Con phụ trách đặc tả màn hình) [1]
```