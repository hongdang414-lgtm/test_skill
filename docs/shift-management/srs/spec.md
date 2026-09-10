---
feature: shift-management
type: srs-spec
version: 1.0.0
created: 2026-09-08
updated: 2026-09-08
status: draft
authors: [BA Team]
changelog:
  - 2026-09-08 | /srs | [screens] added 4 screens and navigation for shift-management
  - 2026-09-08 | /srs | [flows] added 2 sequences, 1 state, 1 flowchart for shift-management
  - 2026-09-08 | /srs | closed OQ-06 (đặt lịch bật/tắt) as out of scope
  - 2026-09-08 | /srs | resolved OQ-05 (mốc kỳ không quản lý ca): ghi chú chân báo cáo, updated BR-018, Mục 3.1, 4
  - 2026-09-08 | /srs | resolved OQ-04 (mô hình chuỗi): defer chuỗi phiên bản sau, updated BR-001, Mục 6.1
  - 2026-09-08 | /srs | resolved OQ-03 (ẩn/khóa thao tác ca khi Tắt): updated BR-010, Mục 3.1, 11
  - 2026-09-08 | /srs | closed OQ-02 (offline gán ca hồi tố) as out of scope
  - 2026-09-08 | /srs | resolved OQ-01 (đóng ca hộ): updated BR-009, Mục 1, 5.3, 6.1
  - 2026-09-08 | /srs | [spec] initialized shift-management specification (config toggle), 18 business rules
---

# Đặc tả tính năng: Bật/Tắt Quản lý ca (Shift Management Toggle)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép cửa hàng lựa chọn có sử dụng chức năng **Quản lý ca** hay không thông qua **một công tắc cấu hình duy nhất cấp cửa hàng**. Khi **Bật**: nhân viên phải chủ động **mở ca** trước khi thực hiện các giao dịch phát sinh dòng tiền tại quầy; hệ thống không bao giờ tự mở ca. Khi **Tắt**: vào màn hình bán hàng và thao tác bán hàng bình thường, không yêu cầu mở ca, không phát sinh nghiệp vụ mở ca/đóng ca, không quản lý doanh thu – tiền mặt – giao dịch theo ca. Phạm vi: (1) thiết kế cấu hình và phân quyền thay đổi; (2) hiệu lực và phạm vi ảnh hưởng của cấu hình đến các nghiệp vụ; (3) hành vi chuyển Tắt sang Bật và Bật sang Tắt; (4) đồng bộ cấu hình đa thiết bị – đa người dùng; (5) xử lý tình huống biên khi cấu hình thay đổi. **Ranh giới (ngoài phạm vi):** nghiệp vụ Mở ca, Đóng ca, Đối soát tiền, Lịch sử ca — mỗi nghiệp vụ đặc tả trong use case riêng; spec này chỉ định nghĩa "cửa vào" và ranh giới dữ liệu mà cấu hình chi phối. |
| **2. Actors (Tác nhân)** | **Chính:** Chủ cửa hàng (Owner) — bật/tắt cấu hình; Quản lý cửa hàng (Store Manager) — bật/tắt cấu hình, đóng ca hộ (bất kỳ lúc nào, có ghi lý do).<br/>**Bị chi phối:** Thu ngân (Cashier), Nhân viên bán hàng (Sales Staff) — phải mở ca trước khi thực hiện giao dịch dòng tiền khi cấu hình đang Bật.<br/>**Hệ thống:** CMS Backend — lưu cấu hình, nguồn chân lý về trạng thái cấu hình và ca; POS Client (đa thiết bị) — nhận và áp dụng cấu hình; Audit Log — ghi nhận mọi thay đổi cấu hình. |
| **3. Pre-conditions** | 1. Người dùng đã đăng nhập CMS/POS.<br/>2. Cửa hàng đã có danh sách nhân viên và gán vai trò.<br/>3. Muốn **Tắt**: cửa hàng không còn ca nào đang mở (BR-009).<br/>4. Muốn **Bật**: không yêu cầu thêm điều kiện — ca đầu tiên được nhân viên mở sau thời điểm bật. |
| **4. Expected Results** | **Happy Path:**<br/>1. Quản lý vào Cài đặt, mục Quản lý ca, thấy công tắc đang **Tắt**.<br/>2. Bật công tắc → hệ thống hiển thị xác nhận _"Từ thời điểm này, nhân viên phải mở ca trước khi thực hiện giao dịch có dòng tiền"_ → chọn **Xác nhận**.<br/>3. Hệ thống lưu cấu hình, ghi audit log (người đổi, thời điểm, giá trị trước–sau), hiệu lực **ngay lập tức**.<br/>4. Thu ngân ở thiết bị khác vào màn hình bán hàng → hệ thống yêu cầu **Mở ca** (chưa có ca đang mở).<br/>5. Thu ngân chủ động mở ca → bán hàng bình thường; mọi giao dịch phát sinh dòng tiền thuộc ca đang mở của thu ngân.<br/>6. Cuối kỳ, quản lý quay lại mục Cài đặt để **Tắt** → hệ thống kiểm tra còn ca mở → hiển thị danh sách ca đang mở kèm nút "Đóng ca".<br/>7. Sau khi tất cả ca được đóng → tắt thành công → bán hàng không còn yêu cầu ca; lịch sử ca và báo cáo ca phát sinh trước đó vẫn xem được đầy đủ.<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Tắt khi còn ca mở]:** Chặn thao tác tắt, hiển thị danh sách ca đang mở và lối tắt đóng ca (BR-009).<br/>- **[B — Đổi cấu hình khi thiết bị khác đang bán]:** Thiết bị kia giữ nguyên thao tác đang làm; giao dịch dòng tiền **tiếp theo** bị chặn yêu cầu mở ca (nếu vừa Bật) hoặc không còn bị chặn (nếu vừa Tắt) (BR-015).<br/>- **[C — Đơn đang tạo dở khi bật]:** Giữ nguyên giỏ hàng; chặn ở bước thanh toán — nhân viên mở ca xong thanh toán tiếp, tiền thu thuộc ca mới (BR-013).<br/>- **[D — Thiết bị offline khi cấu hình đổi]:** Thiết bị theo cấu hình đã lưu trong máy; khi kết nối lại, giao dịch phát sinh lệch cấu hình được đánh dấu "ngoài ca" (BR-016).<br/>- **[E — Hai quản lý đổi gần như đồng thời]:** Thay đổi lưu sau cùng được áp dụng; người còn lại nhận thông báo cấu hình vừa được cập nhật (BR-017).<br/>- **[F — Người không có quyền đổi cấu hình]:** Từ chối với mã lỗi phân quyền, ghi cảnh báo an ninh vào audit log (BR-003).<br/>- **[G — Bật lại sau thời gian tắt]:** Không "hồi sinh" ca cũ — ca mới bắt đầu từ thao tác mở ca đầu tiên sau mốc bật (BR-012). |

### 1.1. Phạm vi giao dịch thuộc quản lý ca

| Nhóm giao dịch | Ví dụ | Khi cấu hình Bật |
| :--- | :--- | :--- |
| Thanh toán đơn hàng tại quầy | Thu tiền mặt/chuyển khoản/quét mã cho đơn POS | Phải thuộc ca đang mở của người thực hiện |
| Phiếu thu | Thu tiền mặt không kèm đơn (tiền lẻ, thu khác) | Phải thuộc ca đang mở của người thực hiện |
| Phiếu chi | Chi tiền mua lẻ, hoàn ứng, chi khác tại quầy | Phải thuộc ca đang mở của người thực hiện |
| Thu nợ | Khách thanh toán công nợ đơn ghi sổ | Phải thuộc ca đang mở của người thực hiện |
| Hoàn tiền / hoàn đơn sau thanh toán | Trả lại tiền cho khách | Phải thuộc ca đang mở của người thực hiện |
| Nghiệm vụ khác phát sinh dòng tiền tại quầy | Nạp thẻ trả trước, đặt cọc… (danh mục mở rộng theo từng use case) | Theo nguyên tắc chung: phát sinh dòng tiền tại quầy thì thuộc ca |

---

## 2. Phân tích thiết kế cấu hình (Phương án A vs Phương án B)

### 2.1. Hai phương án

- **Phương án A:** Chỉ một cấu hình **"Quản lý ca: Bật/Tắt"**. Khi Bật, bắt buộc mở ca mới được bán hàng.
- **Phương án B:** Cấu hình **"Quản lý ca: Bật/Tắt"** cộng cấu hình phụ **"Bắt buộc mở ca trước khi bán: Bật/Tắt"**.

### 2.2. Ma trận trạng thái của Phương án B

| Quản lý ca | Bắt buộc mở ca | Hệ quả nghiệp vụ | Đánh giá |
| :--- | :--- | :--- | :--- |
| Tắt | (ẩn/vô hiệu) | Bán hàng tự do, không có ca | Hợp lệ |
| Bật | Bật | Mọi giao dịch dòng tiền thuộc ca | Hợp lệ — trùng đúng Phương án A |
| Bật | Tắt | Ca tồn tại nhưng mở ca là tự nguyện; giao dịch có thể nằm ngoài ca | **Lửng nghiệp vụ:** báo cáo ca không khớp doanh thu, tiền quầy khó đối soát |
| Tắt | Bật | Không quản lý ca mà vẫn bắt buộc mở ca | **Mâu thuẫn logic** — phải chặn bằng giao diện |

### 2.3. So sánh theo tiêu chí

| Tiêu chí | Phương án A — một công tắc | Phương án B — thêm cấu hình phụ |
| :--- | :--- | :--- |
| Độ dễ hiểu | Một khái niệm duy nhất: bật quản lý ca đồng nghĩa phải mở ca | Phải giải thích hai tầng: "quản lý ca" là gì, khác "bắt buộc mở ca" chỗ nào |
| Nguy cơ cấu hình mâu thuẫn | Không thể mâu thuẫn — chỉ 2 trạng thái | 4 tổ hợp, trong đó 2 tổ hợp gây lửng/mâu thuẫn (Mục 2.2) |
| Ngữ nghĩa dữ liệu | Khi Bật, 100% giao dịch dòng tiền thuộc ca — báo cáo ca khớp tiền, đối soát kín | Kỳ "có ca" nhưng doanh thu thiếu đoạn ngoài ca — báo cáo ca lệch doanh thu, đối soát không khớp |
| Độ phủ nhu cầu SMB | Đủ: cửa hàng nhỏ hoặc muốn đếm tiền theo ca, hoặc không cần — không có nhóm giữa đáng kể | Nhóm "muốn theo dõi ca tự nguyện" hiếm, thường thuộc chuỗi lớn có quy trình riêng |
| Chi phí kiểm thử & hỗ trợ | 2 trạng thái × đa thiết bị | 4 tổ hợp × đa thiết bị × nhân viên — diện kiểm thử và câu hỏi hỗ trợ tăng gấp đôi |

### 2.4. Kết luận

**Chọn Phương án A.** Lý do:

1. Giá trị cốt lõi của quản lý ca là **đối soát tiền kín** — mọi dòng tiền thuộc một ca, một người chịu trách nhiệm. Cấu hình phụ phá vỡ chính bất biến này.
2. SMB dùng POS cần cấu hình "hiểu trong 5 giây"; một công tắc không có trạng thái mâu thuẫn giúp giảm đáng kể lỗi vận hành và câu hỏi hỗ trợ.
3. Nhu cầu "ca tự nguyện" nếu thực sự xuất hiện, hãy nâng cấp sau bằng một cấu hình phụ có kèm chuyển đổi dữ liệu có kiểm soát — không dựng trước cấu hình mà chưa có người dùng cần.

---

## 3. Phạm vi ảnh hưởng của cấu hình

### 3.1. Bảng ảnh hưởng theo khu vực nghiệp vụ

| Khu vực | Khi Tắt | Khi Bật |
| :--- | :--- | :--- |
| Màn hình bán hàng | Vào và bán hàng tự do | Vào màn hình khi chưa có ca → hệ thống yêu cầu Mở ca; chặn giao dịch dòng tiền đến khi có ca (BR-005, BR-006) |
| Quyền truy cập chức năng bán hàng | Không đổi — bán hàng theo quyền sẵn có | Không thêm quyền mới; "cửa" là **có ca đang mở**, không phải quyền hạn |
| Mở ca | Ẩn điểm truy cập nhanh trên màn bán hàng; tại màn chức năng ca hiển thị khóa + tooltip "Quản lý ca đang tắt. Liên hệ quản lý để bật" | Hiển thị; nhân viên chủ động mở, hệ thống không tự mở (BR-005) |
| Đóng ca | Ẩn điểm truy cập nhanh trên màn bán hàng; tại màn chức năng ca hiển thị khóa + tooltip tương tự Mở ca | Hiển thị; điều kiện tắt cấu hình là đã đóng hết ca (BR-009) |
| Lịch sử ca | **Vẫn xem được** dữ liệu ca đã phát sinh (chỉ đọc) | Xem đầy đủ |
| Báo cáo theo ca | Báo cáo kỳ đã quản lý ca **giữ nguyên số liệu**; đoạn thời gian tắt không có số liệu ca (phân loại "ngoài ca", kèm ghi chú chân báo cáo — BR-018) | Đầy đủ theo ca |
| Phiếu thu | Không gắn ca | Thuộc ca đang mở của người lập phiếu |
| Phiếu chi | Không gắn ca | Thuộc ca đang mở của người lập phiếu |
| Thu nợ | Không gắn ca | Thuộc ca đang mở của người thu |
| Hoàn tiền / hoàn đơn | Không gắn ca | Thuộc ca đang mở của người hoàn |
| Thanh toán đơn hàng | Không gắn ca | Thuộc ca đang mở của người thu tiền |
| Nghiệp vụ khác phát sinh dòng tiền | Không gắn ca | Theo nguyên tắc chung Mục 1.1 — thuộc ca |

### 3.2. Rule dữ liệu lịch sử (chốt)

> **Bật/Tắt Quản lý ca chỉ thay đổi việc có yêu cầu ca hay không đối với giao dịch phát sinh SAU mốc thay đổi. Dữ liệu ca đã phát sinh trước đó KHÔNG bị xóa, KHÔNG bị ẩn, KHÔNG bị sửa** — lịch sử ca và báo cáo ca vẫn truy cập được sau khi tắt, phục vụ tra cứu và đối soát (BR-011).

---

## 4. Chuyển đổi Tắt → Bật

| Câu hỏi | Phân tích | Hành vi đề xuất |
| :--- | :--- | :--- |
| Có hiệu lực ngay không? | Trạng thái cấu hình nằm trên server; việc "bị chặn" được kiểm tra tại thời điểm thực hiện giao dịch, không phụ thuộc phiên làm việc | Hiệu lực **ngay** tại thời điểm lưu thành công (BR-007) |
| Có cần đăng xuất/đăng nhập lại không? | Nếu ràng buộc lúc đăng nhập, người dùng đang online sẽ bị "sót"; kiểm tra mỗi giao dịch là chặt hơn | **Không cần** — mọi phiên đang hoạt động chịu tác động của cấu hình mới |
| Đang ở màn hình bán hàng thì sao? | Đá người dùng ra giữa chừng làm mất việc đang làm | Giữ nguyên màn hình; hiển thị banner trạng thái ca; **giao dịch dòng tiền đầu tiên** sau mốc bị chặn yêu cầu mở ca |
| Đơn hàng đang tạo/chưa thanh toán? | Giỏ hàng có giá trị với nhân viên, không được mất | Giữ nguyên giỏ; chặn ở bước thanh toán — mở ca xong thanh toán tiếp; **toàn bộ tiền thu của đơn thuộc ca mới** (BR-013) |
| Khi nào bắt đầu yêu cầu mở ca? | Mốc phải tuyệt đối rõ ràng để báo cáo phân kỳ | Tại mốc lưu cấu hình — mọi giao dịch phạm vi Mục 1.1 phát sinh sau mốc yêu cầu ca |
| Giao dịch trước mốc bật có thuộc ca mới không? | Tiền đã thu không thể "chia đôi" ca; gán hồi tố làm sai trách nhiệm nhân viên | **Không gán hồi tố** — giao dịch trước mốc có thông tin ca để trống, báo cáo phân loại "ngoài ca" (BR-008) |
| Có cần ghi nhận thời điểm bật? | Báo cáo doanh thu cần biết ranh giới "có ca/không ca" | **Có** — audit log lưu mốc; kỳ báo cáo chứa mốc bật/tắt hiển thị ghi chú chân báo cáo (BR-018) |

---

## 5. Chuyển đổi Bật → Tắt (khi còn ca chưa đóng)

### 5.1. Ba hướng xử lý

- **Hướng 1 — Chặn:** Không cho tắt khi còn ca đang mở. Thông báo: _"Không thể tắt Quản lý ca khi vẫn còn ca đang mở. Vui lòng đóng tất cả ca trước khi tiếp tục."_
- **Hướng 2 — Tự động đóng:** Cho phép tắt, hệ thống tự động đóng các ca đang mở.
- **Hướng 3 — Ca treo:** Cho phép tắt, các ca đang mở vẫn tồn tại chờ được đóng sau.

### 5.2. Ma trận so sánh

| Tiêu chí | Hướng 1 — Chặn | Hướng 2 — Tự động đóng | Hướng 3 — Ca treo |
| :--- | :--- | :--- | :--- |
| An toàn dữ liệu | **Cao** — mọi ca kết thúc bằng quy trình đóng có đối soát đầy đủ | Trung bình — ca bị đóng "ép", số liệu chốt tại thời điểm máy đóng, thiếu xác nhận tiền của người giữ ca | Thấp — ca mở vô thời hạn, không rõ kết thúc khi nào |
| Khả năng audit | **Cao** — mỗi ca có người đóng rõ ràng, trách nhiệm rõ | Trung bình — ghi chú "đóng tự động do tắt tính năng", người mở ca không xác nhận số liệu | Kém — khó truy trách nhiệm chênh lệch phát sinh sau này |
| Rủi ro tài chính | **Gần như không** — tiền được đếm, đối soát trước khi tắt | Cao — chênh lệch tiền chỉ phát hiện sau khi ca đã "đóng", quy trách nhiệm rất khó | Cao — tiền nằm trong ca không chốt, kéo dài vô hạn |
| Trải nghiệm người dùng | Kém nhất nếu quên đóng ca — giảm ma sát bằng danh sách ca mở + nút đóng ngay (Mục 5.3) | Tốt nhất — một cú click là xong | Trung bình — tưởng tiện nhưng để lại "nợ" phải xử lý sau |
| Độ phức tạp hệ thống | **Thấp** — chỉ cần kiểm tra điều kiện + dialog | Trung bình — cần nghiệp vụ đóng ca tự động, ghi chú đặc biệt, xử lý giao dịch đang dở của ca | Cao — phải xử lý ca treo khi bật lại, ca quá hạn, báo cáo lệch kỳ |

### 5.3. Đề xuất

**Chọn Hướng 1 (Chặn khi còn ca mở)**, kèm 3 giải pháp giảm ma sát:

1. Dialog tắt hiển thị **danh sách ca đang mở** (người mở, thời điểm mở, thiết bị) kèm nút **"Đóng ca"** đi thẳng đến thao tác đóng — quản lý không phải đi tìm từng ca.
2. **Quản lý được đóng ca hộ** thu ngân bất kỳ lúc nào, bắt buộc ghi lý do; người mở ca xem được thông tin "ca được đóng hộ" kèm lý do (BR-009) — tránh tình trạng nhân viên nghỉ ca không đóng được thì cửa hàng không bao giờ tắt được tính năng. Chi tiết quy trình đóng ca thuộc use case Đóng ca.
3. Thông báo nêu rõ số ca đang mở và hướng dẫn: _"Không thể tắt Quản lý ca khi vẫn còn {N} ca đang mở. Vui lòng đóng tất cả ca trước khi tiếp tục."_

Lý do chọn: giao dịch tiền mặt phải được chốt bằng **đối soát có con người**; đóng ca tự động hoặc để ca treo đều làm mất khả năng quy trách nhiệm chênh lệch tiền — đúng giá trị cốt lõi của quản lý ca. Đây cũng là chuẩn chung của các POS bán lẻ (Square, Loyverse yêu cầu đóng ca/closing the drawer trước khi đổi cấu hình liên quan).

*Tùy chọn dành cho giai đoạn sau:* nếu phản hồi thực tế cho thấy phiền, bổ sung "Ép đóng ca có xác nhận + lý do" (biến thể của Hướng 2 có kiểm soát) — không đưa vào giai đoạn 1.

---

## 6. Phân quyền

### 6.1. Ma trận quyền theo vai trò

| Vai trò | Đổi cấu hình Bật/Tắt | Xem trạng thái cấu hình | Chịu ràng buộc mở ca |
| :--- | :--- | :--- | :--- |
| Chủ cửa hàng (Owner) | Có | Có | Có (khi trực tiếp bán) |
| Quản lý cửa hàng (Store Manager) | Có | Có | Có (khi trực tiếp bán) |
| Thu ngân (Cashier) | Không | Có | Có |
| Nhân viên bán hàng (Sales Staff) | Không | Có | Có |

Quyền đổi cấu hình là quyền riêng **`MANAGE_SHIFT_SETTING`**, mặc định gán cho Chủ cửa hàng và Quản lý cửa hàng; có thể gán thêm/thu hồi qua màn phân quyền nếu hệ thống cho phép tùy biến vai trò (BR-003).
Riêng quyền **đóng ca hộ** (đóng ca của nhân viên khác, BR-009): áp cho Chủ cửa hàng và Quản lý cửa hàng, độc lập với quyền đổi cấu hình — chi tiết quy trình thuộc use case Đóng ca.
Giai đoạn 1, mỗi cửa hàng độc lập hoàn toàn — Quản lý chuỗi không có quyền can thiệp cấu hình ca của cửa hàng con; mô hình chuỗi defer phiên bản sau.

### 6.2. Nhân viên không có quyền đổi cấu hình

- **Vẫn được xem trạng thái hiện tại** (chỉ đọc): cần thiết để nhân viên biết mình có đang bị ràng buộc ca không. Thể hiện qua: banner trạng thái ca trên màn bán hàng khi Bật; mục Cài đặt hiển thị trạng thái khóa kèm nhãn "Chỉ quản lý được thay đổi" (BR-004).

### 6.3. Quản lý đổi cấu hình khi nhân viên đang bán hàng ở thiết bị khác

- Server là nguồn chân lý; cấu hình mới có hiệu lực với **giao dịch tiếp theo** của mọi thiết bị.
- Thiết bị đang bán: không bị đá ra giữa chừng — xử lý như nhánh [B] và [C] Mục 1: giao dịch đã ghi nhận giữ nguyên, giao dịch tiếp theo chịu cấu hình mới (BR-015).
- Chi tiết đồng bộ: xem Mục 7.

---

## 7. Multi-device / Multi-user

### 7.1. Xác định kiến trúc cấu hình

| Câu hỏi | Kết luận | Lý do |
| :--- | :--- | :--- |
| Cấu hình cấp cửa hàng hay cấp thiết bị? | **Cấp cửa hàng** | Ca gắn với người + cửa hàng; doanh thu cần tổng hợp theo cửa hàng. Nếu mỗi thiết bị một cấu hình, cùng một lúc có giao dịch thuộc ca và không thuộc ca — phá đối soát (trùng lập luận Mục 2.2) |
| Thời điểm có hiệu lực? | Tại mốc server ghi nhận thay đổi | Mọi thiết bị quy về một mốc duy nhất để báo cáo phân kỳ chính xác |
| Có cần đồng bộ realtime không? | **Ưu tiên có, nhưng không phụ thuộc** | Realtime (đẩy về các POS) cho trải nghiệm tốt; nhưng lớp bảo đảm cuối là **đối chiếu cấu hình + ca với server ngay trước khi ghi nhận giao dịch** thuộc phạm vi Mục 1.1 |
| Thiết bị chưa nhận cấu hình mới thì ưu tiên hành vi nào? | Theo cấu hình server **tại thời điểm giao dịch được ghi nhận** | Giao dịch đã ghi nhận không thu hồi; nếu thiết bị dùng cấu hình cũ dẫn đến lệch → đánh dấu "ngoài ca" vào báo cáo ngoại lệ, không gán hồi tố tự động (BR-016) |

### 7.2. Bảng tình huống đa thiết bị

| Tình huống | Xử lý |
| :--- | :--- |
| Nhiều máy POS cùng cửa hàng, nhiều nhân viên đang đăng nhập | Cùng một giá trị cấu hình; mỗi nhân viên tự chịu ràng buộc ca với chính ca của mình |
| Quản lý bật từ POS A trong khi POS B đang bán hàng | POS B nhận đẩy realtime → banner trạng thái ca xuất hiện, giao dịch dòng tiền tiếp theo bị chặn yêu cầu mở ca; nếu realtime trễ, lớp đối chiếu khi ghi nhận chặn tại server |
| Quản lý tắt từ POS A trong khi POS B đang có ca mở | **Bị chặn ngay từ bước lưu** — ca mở là của cửa hàng chứ không của thiết bị nào (BR-009); không tồn tại tình huống "tắt được ở A mà B vẫn còn ca" |
| Thiết bị offline khi cấu hình thay đổi | Thiết bị theo cấu hình đã lưu trong máy; vẫn bán theo quy trình offline hiện có |
| Thiết bị nhận cấu hình mới sau khi kết nối lại (reconnect) | Đối chiếu ngay cấu hình + trạng thái ca với server; giao dịch phát sinh trong lúc lệch được đánh dấu "ngoài ca (phát sinh khi offline)" (BR-016) |

---

## 8. Edge cases

| # | Tình huống | Xử lý đề xuất | Rule |
| :--- | :--- | :--- | :--- |
| 1 | Bật Quản lý ca nhưng người dùng chưa có ca | Vào màn hình bán hàng → hiển thị yêu cầu Mở ca; chặn mọi giao dịch phạm vi Mục 1.1 cho đến khi mở ca; hệ thống không tự mở thay | BR-005, BR-006 |
| 2 | Người dùng đang tạo đơn thì quản lý bật | Giữ nguyên giỏ hàng; chặn ở bước thanh toán — mở ca xong thanh toán tiếp, tiền thu thuộc ca mới | BR-013 |
| 3 | Người dùng đang thanh toán dở thì cấu hình thay đổi | Giao dịch **đã được server ghi nhận** trước mốc: giữ nguyên (ngoài ca nếu trước lúc bật); **chưa ghi nhận**: chịu cấu hình mới | BR-014 |
| 4 | Có ca đang mở nhưng quản lý muốn tắt | Chặn + danh sách ca đang mở + nút đóng ca ngay tại dialog | BR-009 |
| 5 | Ca đang đóng dở (đang đối soát) thì quản lý tắt | Ca vẫn tính là "đang mở" đến khi đóng hoàn tất — chặn như trường hợp 4; hoàn tất đóng ca rồi mới tắt được | BR-009 |
| 6 | Thiết bị mất mạng khi cấu hình thay đổi | Theo cấu hình trong máy; khi online đối chiếu; giao dịch lệch đánh dấu "ngoài ca" | BR-016 |
| 7 | Hai quản lý cùng đổi cấu hình gần như đồng thời | Mỗi lần lưu là một thay đổi nguyên vẹn; **thay đổi sau cùng được áp dụng**; cả hai lần đổi đều ghi audit; người đổi trước nhận thông báo cấu hình vừa được người khác cập nhật | BR-017 |
| 8 | Người dùng không có quyền nhưng gọi API đổi cấu hình | Từ chối với lỗi phân quyền `403 PERMISSION_DENIED`; ghi cảnh báo an ninh vào audit log | BR-003 |
| 9 | Bật lại Quản lý ca sau một khoảng thời gian đã tắt | Bắt đầu mới hoàn toàn: ca đầu tiên là ca được mở sau mốc bật; không hồi sinh ca cũ | BR-012 |
| 10 | Lịch sử ca cũ sau khi tắt | Vẫn xem đầy đủ (chỉ đọc); menu lịch sử ca không bị ẩn | BR-011 |
| 11 | Báo cáo ca cũ sau khi tắt | Số liệu kỳ đã quản lý ca giữ nguyên; đoạn thời gian tắt không có số liệu ca, doanh thu hiển thị phân loại "ngoài ca" | BR-008, BR-011 |

---

## 9. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

> ID đầy đủ có prefix feature theo quy ước đặt tên: `BR-001` dưới đây tương ứng `BR-shift-management-001`.

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-001** | **[Constraint] Cấu hình cấp cửa hàng, mặc định Tắt** | Quản lý ca là một cấu hình duy nhất áp dụng cho toàn bộ cửa hàng; mọi thiết bị POS của cửa hàng dùng chung một giá trị. Cửa hàng mới khởi tạo mặc định **Tắt**. Không có cấu hình cấp thiết bị, cấp nhân viên. Giai đoạn 1: mỗi cửa hàng độc lập hoàn toàn — Quản lý chuỗi không đặt lại/khóa cấu hình ca của cửa hàng con; chính sách cấp chuỗi xem xét ở phiên bản sau. | Không tồn tại đường thao tác đặt cấu hình riêng theo thiết bị/người dùng. |
| **BR-002** | **[Constraint] Một công tắc duy nhất — không cấu hình phụ** | Chỉ tồn tại cấu hình "Quản lý ca: Bật/Tắt" (kết quả phân tích Mục 2). Khi Bật, mọi giao dịch phạm vi Mục 1.1 bắt buộc thuộc ca đang mở. Không có cấu hình "bắt buộc mở ca" tách riêng. | Không áp dụng — thiết kế đã loại bỏ cấu hình phụ. |
| **BR-003** | **[Constraint] Phân quyền thay đổi cấu hình** | Chỉ tài khoản có quyền `MANAGE_SHIFT_SETTING` (mặc định: Chủ cửa hàng, Quản lý cửa hàng) được bật/tắt. | Ẩn/vô hiệu công tắc với vai trò khác; API từ chối `403 PERMISSION_DENIED` và ghi cảnh báo an ninh vào audit log. |
| **BR-004** | **[Constraint] Trạng thái cấu hình hiển thị cho mọi nhân viên** | Mọi tài khoản đăng nhập đều xem được trạng thái hiện tại (chỉ đọc): banner trạng thái ca trên màn bán khi Bật; mục Cài đặt hiển thị khóa kèm nhãn "Chỉ quản lý được thay đổi". | Không áp dụng — quyền xem là mặc định cho tất cả. |
| **BR-005** | **[Constraint] Hệ thống không tự động mở ca** | Khi Bật và người dùng chưa có ca đang mở: hệ thống hiển thị yêu cầu mở ca và chặn giao dịch; không có cơ chế tự mở ca theo lịch, theo đăng nhập hay ngầm định. | Thao tác giao dịch khi chưa có ca: chặn, thông báo _"Cần mở ca trước khi thực hiện giao dịch này."_ |
| **BR-006** | **[Derivation] Phạm vi giao dịch thuộc ca** | Khi Bật, các giao dịch phát sinh dòng tiền tại quầy (Mục 1.1: thanh toán đơn, phiếu thu, phiếu chi, thu nợ, hoàn tiền, và nghiệp vụ dòng tiền tại quầy khác) phải thuộc **ca đang mở của người thực hiện** tại thời điểm ghi nhận. | Ghi nhận giao dịch khi người thực hiện không có ca mở: chặn tại server, giao dịch không được ghi. |
| **BR-007** | **[State Transition] Hiệu lực tức thì khi Bật** | Cấu hình có hiệu lực ngay tại thời điểm server ghi nhận lưu thành công; không cần đăng xuất/đăng nhập lại; mọi phiên đang hoạt động của mọi thiết bị chịu tác động với giao dịch phát sinh sau mốc. | Không áp dụng — không tồn tại chế độ "chờ hiệu lực". |
| **BR-008** | **[Derivation] Không gán hồi tố** | Giao dịch phát sinh trước mốc bật có thông tin ca **để trống**, không được gán vào ca mở sau đó kể cả có cùng người thực hiện; báo cáo phân loại các giao dịch này là "ngoài ca". | Thao tác gán hồi tố giao dịch vào ca: không tồn tại trong sản phẩm (đối soát hồi tố nếu có thuộc use case Đối soát, phải qua quản lý). |
| **BR-009** | **[State Transition] Chặn tắt khi còn ca mở** | Không thể chuyển Bật sang Tắt khi cửa hàng còn **một hoặc nhiều ca đang mở** (bao gồm ca đang trong quá trình đóng). Dialog tắt hiển thị số ca mở, danh sách (người mở, thời điểm, thiết bị) và nút "Đóng ca" đi thẳng đến thao tác đóng. Quản lý được đóng ca hộ bất kỳ lúc nào: bắt buộc ghi lý do, người mở ca xem được thông tin "được đóng hộ" kèm lý do (chi tiết quy trình thuộc use case Đóng ca). | Nút Tắt hiển thị cảnh báo _"Không thể tắt Quản lý ca khi vẫn còn {N} ca đang mở. Vui lòng đóng tất cả ca trước khi tiếp tục."_ |
| **BR-010** | **[State Transition] Hiệu lực tức thì khi Tắt** | Sau mốc tắt: không yêu cầu ca với giao dịch mới; các điểm truy cập nhanh mở ca/đóng ca trên màn bán hàng bị ẩn, tại màn chức năng ca (Đối soát, Lịch sử ca) hiển thị trạng thái khóa kèm tooltip "Quản lý ca đang tắt. Liên hệ quản lý để bật"; giao dịch phát sinh sau mốc không gắn ca. Phiên đang mở ca được kết thúc bình thường bằng thao tác đóng ca trước khi tắt (theo BR-009). | Không áp dụng. |
| **BR-011** | **[Constraint] Tắt không xóa dữ liệu ca** | Việc tắt không xóa, không ẩn, không sửa lịch sử ca, đối soát và báo cáo ca đã phát sinh. Menu Lịch sử ca vẫn truy cập được sau khi tắt (chỉ đọc). Dữ liệu lưu theo chính sách lưu trữ chung của hệ thống. | Thao tác xóa dữ liệu ca khi tắt: không tồn tại trong sản phẩm. |
| **BR-012** | **[State Transition] Bật lại không hồi sinh ca cũ** | Khi bật lại sau thời gian tắt: lịch sử ca cũ giữ nguyên; ca hoạt động mới bắt đầu từ thao tác mở ca đầu tiên sau mốc bật; không tự nối tiếp ca trước thời điểm tắt. | Không áp dụng — không tồn tại cơ chế nối ca cũ. |
| **BR-013** | **[Action Enabler] Đơn đang thao tác khi cấu hình đổi** | Đơn hàng đang tạo/chưa thanh toán tại thời điểm cấu hình đổi: **giữ nguyên giỏ hàng**; khi thanh toán, nếu cấu hình đang Bắt → yêu cầu mở ca trước, toàn bộ tiền thu của đơn thuộc ca mở đó. Đơn đã hoàn tất thanh toán trước mốc: không bị ảnh hưởng. | Thanh toán đơn dở khi chưa mở ca (vừa bật): chặn ở bước thanh toán, mở ca xong tiếp tục — không mất giỏ hàng. |
| **BR-014** | **[Derivation] Mốc ghi nhận quyết định thuộc ca hay không** | Một giao dịch thuộc ca hay không quyết định bởi trạng thái cấu hình **tại thời điểm server ghi nhận giao dịch** — không phải thời điểm bắt đầu mở màn hình hay bắt đầu tạo chứng từ. Giao dịch đã ghi nhận không thay đổi thuộc tính ca khi cấu hình đổi. | Không áp dụng — quy tắc suy diễn thuần. |
| **BR-015** | **[Constraint] Đồng bộ đa thiết bị** | Server là nguồn chân lý. Hệ thống ưu tiên đẩy cấu hình mới về các thiết bị theo thời gian thực để cập nhật giao diện; bất kể đã nhận được hay chưa, **mọi giao dịch phạm vi Mục 1.1 đều đối chiếu cấu hình + ca với server ngay trước khi ghi nhận**. | Giao dịch đến server khi người thực hiện không có ca mở (theo cấu hình mới nhất): chặn, thiết bị hiển thị yêu cầu mở ca. |
| **BR-016** | **[Constraint] Thiết bị offline** | Thiết bị offline theo cấu hình đã lưu trong máy. Giao dịch phát sinh khi offline được ghi nhận theo quy trình offline hiện có; khi đồng bộ, nếu cấu hình server đã khác (ví dụ đã Bật nhưng giao dịch không thuộc ca): **giữ nguyên không thu hồi**, đánh dấu "ngoài ca — phát sinh khi offline" và liệt kê trong báo cáo ngoại lệ cho quản lý xem. | Không tự gán vào ca đang mở khi đồng bộ — tránh gán hồi tố vi phạm BR-008. |
| **BR-017** | **[Constraint] Thay đổi gần như đồng thời** | Hai quản lý cùng đổi cấu hình trong khoảng thời gian sát nhau: mỗi lần lưu là một thay đổi nguyên vẹn có audit riêng; **thay đổi lưu sau cùng là giá trị hiệu lực**. Người đổi trước (đang mở màn cài đặt) nhận thông báo _"Cấu hình vừa được {NGƯỜI} cập nhật"_ và màn hình làm mới giá trị. | Không khóa toàn màn cài đặt — chỉ đối chiếu lại giá trị tại lúc lưu. |
| **BR-018** | **[Audit] Ghi log mọi lần thay đổi cấu hình** | Mỗi lần bật/tắt ghi audit: người thay đổi, thời điểm chính xác (mốc hiệu lực), giá trị trước–sau, thiết bị thực hiện. Mốc này là căn cứ phân kỳ "có ca/không ca" trong báo cáo doanh thu: khi kỳ báo cáo chứa mốc bật/tắt, báo cáo hiển thị ghi chú chân trang nêu khoảng thời gian không quản lý ca (ví dụ: "Kỳ này có đoạn không quản lý ca từ 01/09–05/09"); không vẽ dải phân kỳ trên dòng thời gian. | Lỗi ghi log: vẫn xác nhận thay đổi cấu hình, ghi cảnh báo hệ thống cho kỹ thuật xử lý. |

---

## 10. Sơ đồ tương tác (Interaction Diagram)

*(Sơ đồ đầy đủ tại `docs/shift-management/srs/flows.md`)*

![[flows.md]]

---

## 11. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/shift-management/srs/screens.md`)*

![[screens.md]]

---

## 12. Open Questions

- [x] **OQ-01:** Đóng ca hộ — quản lý được đóng ca của thu ngân trong trường hợp nào, có bắt buộc ghi lý do không, người mở ca có được xem ghi chú "đóng hộ" không? — **Resolved:** Quản lý đóng ca hộ bất kỳ lúc nào; bắt buộc ghi lý do; người mở ca xem được thông tin "được đóng hộ" kèm lý do (BR-009, Mục 5.3).
- [~] **OQ-02:** Giao dịch offline lệch cấu hình khi đồng bộ — chỉ đánh dấu "ngoài ca" vĩnh viễn (đề xuất hiện tại) hay cần cơ chế quản lý xác nhận rồi gán vào ca hồi tố có kiểm soát? — **Out of scope:** không xây cơ chế gán ca hồi tố trong phạm vi này; giữ hành vi đánh dấu "ngoài ca" theo BR-016. Tái xác nhận khi đặc tả use case Đối soát.
- [x] **OQ-03:** Khi Tắt, các thao tác mở ca/đóng ca ẩn hoàn toàn hay hiển thị ở trạng thái khóa kèm tooltip giải thích? — **Resolved:** Kết hợp — ẩn điểm truy cập nhanh trên màn bán hàng; tại màn chức năng ca (Đối soát, Lịch sử ca) hiển thị khóa + tooltip "Quản lý ca đang tắt. Liên hệ quản lý để bật" (BR-010).
- [x] **OQ-04:** Mô hình chuỗi: mỗi cửa hàng tự bật/tắt độc lập (đã chốt cấp cửa hàng) — nhưng quản lý chuỗi có được quyền đặt lại cấu hình cho từng cửa hàng con, và cấu hình chuỗi có bị khóa (override) không? — **Resolved:** Giai đoạn 1 mỗi cửa hàng độc lập hoàn toàn — Quản lý chuỗi không can thiệp cấu hình ca (không đặt lại, không khóa chính sách); mô hình chuỗi defer phiên bản sau (BR-001).
- [x] **OQ-05:** Báo cáo doanh thu có cần hiển thị trực quan mốc "kỳ không quản lý ca" (đoạn giao dịch ngoài ca) như một chú thích dòng thời gian không? — **Resolved:** Trung gian — không vẽ dải phân kỳ trên dòng thời gian; khi kỳ báo cáo chứa mốc bật/tắt, hiển thị ghi chú chân báo cáo dạng text nêu khoảng thời gian không quản lý ca (BR-018).
- [~] **OQ-06:** Có cần đặt lịch bật/tắt theo thời điểm (ví dụ "bật từ đầu tháng sau") thay vì chỉ hiệu lực ngay? — **Out of scope:** không xây đặt lịch; cấu hình chỉ hiệu lực tức thì theo BR-007/BR-010. Tái xem xét nếu có yêu cầu thực từ cửa hàng.

---

## 13. References

- `@../../rules/ba-conventions.md`
- `@../../rules/approval-gate.md`
- Use case kế tiếp (chưa có): Mở ca, Đóng ca, Đối soát tiền, Lịch sử ca — `docs/shift-management/usecases/`
- Sapo POS — giao ca/quản lý ca: https://support.sapo.vn
- Loyverse — Shift management: https://loyversehelp.com
- Square — Cash drawer & shift management: https://www.squareup.com
