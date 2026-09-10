---
feature: open-shift
type: srs-spec
version: 1.0.0
created: 2026-09-08
updated: 2026-09-08
status: draft
authors: [BA Team]
changelog:
  - 2026-09-08 | /srs | cascade từ OQ-06 close-shift resolved: GĐ1 không làm cảnh báo ca mở quá lâu (OQ-08)
  - 2026-09-08 | /srs | cascade từ OQ-04 close-shift resolved: hạn mức giữ hằng số hệ thống, OOS cấu hình cửa hàng
  - 2026-09-08 | /srs | cascade từ close-shift: làm rõ "đang đóng dở" là trạng thái quy trình giao diện, không phải trạng thái dữ liệu (Mục 10.2)
  - 2026-09-08 | /srs | resolved OQ-01/02/03/05/07/08, hold OQ-04, OOS OQ-06; updated Mục 3.2, 6, 7.2, 9, 10.1, 10.2
  - 2026-09-08 | /srs | [spec] initialized open-shift specification (mở ca), 15 business rules
---

# Đặc tả tính năng: Mở ca (Open Shift)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Cho phép nhân viên bắt đầu một ca làm việc trên POS để hệ thống ghi nhận **người mở ca**, **thời điểm bắt đầu ca**, **tiền mặt đầu ca** (nếu cửa hàng quản lý tiền mặt), và xác định **ca hiện thời** mà mọi giao dịch phát sinh dòng tiền tiếp theo được ghi nhận vào. Mở ca thành công thì người dùng vào màn hình bán hàng và thực hiện giao dịch bình thường. **Phạm vi:** (1) điều kiện và quyền mở ca; (2) luồng mở ca chuẩn; (3) tiền mặt đầu ca — nhập, xác nhận, nguyên tắc điều chỉnh; (4) thời gian mở ca; (5) hành vi đa thiết bị – đa người dùng; (6) xử lý lỗi và tình huống biên. **Ranh giới (ngoài phạm vi):** cấu hình Bật/Tắt Quản lý ca — đặc tả tại `docs/shift-management/srs/spec.md` (gọi tắt **spec Bật/Tắt**); Đóng ca, đối soát tiền cuối ca, điều chỉnh tiền trong ca, lịch sử ca — mỗi nghiệp vụ đặc tả riêng. |
| **2. Actors (Tác nhân)** | **Chính:** Thu ngân (Cashier), Nhân viên bán hàng (Sales Staff) — trực tiếp mở ca trước khi bán; Quản lý cửa hàng / Chủ cửa hàng khi trực tiếp bán cũng mở ca như nhân viên. **Hệ thống:** CMS Backend — nguồn chân lý về ca: kiểm tra điều kiện, tạo ca, sinh mã ca, ghi mốc thời gian; POS Client (đa thiết bị) — gửi yêu cầu mở ca, hiển thị trạng thái ca; Audit Log — ghi nhận mở ca. |
| **3. Pre-conditions** | 1. Người dùng đã đăng nhập tài khoản nhân viên trên thiết bị POS.<br/>2. Quản lý ca đang **Bật** (BR-003).<br/>3. Tài khoản còn hiệu lực làm việc tại cửa hàng.<br/>4. Người dùng chưa có ca đang mở hoặc đang đóng dở (BR-002).<br/>5. Thiết bị kết nối được server (BR-012). |
| **4. Expected Results** | **Happy Path:**<br/>1. Thu ngân đăng nhập POS, vào màn hình bán hàng.<br/>2. Hệ thống đối chiếu cấu hình + ca hiện tại với server: Quản lý ca Bật, thu ngân chưa có ca → hiển thị banner "Chưa có ca đang mở", chặn giao dịch dòng tiền.<br/>3. Thu ngân chạm **Mở ca** → màn Mở ca hiển thị thông tin ca (cửa hàng, nhân viên, thiết bị, đồng hồ thời gian thực) và ô Tiền mặt đầu ca.<br/>4. Thu ngân nhập tiền đầu ca (hoặc để trống = 0) → hệ thống kiểm tra hợp lệ ngay tại máy.<br/>5. Thu ngân nhấn **Mở ca** → nút khóa ngay, gửi yêu cầu tạo ca.<br/>6. Server kiểm tra: cấu hình Bật, tài khoản hiệu lực, chưa có ca, tiền hợp lệ → tạo ca thành công, ghi người mở, mốc mở theo giờ server, cửa hàng, thiết bị mở, tiền đầu ca; ghi audit.<br/>7. POS hiển thị "Mở ca thành công" kèm tóm tắt ca → thu ngân vào màn bán hàng; mọi giao dịch dòng tiền tiếp theo thuộc ca (BR-010).<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Đã có ca đang mở trên thiết bị khác]:** không mở ca mới; hiển thị thông tin ca hiện tại, tiếp tục bán trên thiết bị đang dùng (BR-002, BR-014, Mục 7.2 Case 1).<br/>- **[B — Đang tạo đơn dở khi bị yêu cầu mở ca]:** giữ nguyên giỏ hàng; chặn ở bước thanh toán; mở ca xong thanh toán tiếp, tiền thu thuộc ca mới (BR-shift-management-013).<br/>- **[C — Quản lý ca bị Tắt khi đang ở màn Mở ca]:** màn Mở ca tự đóng; không còn yêu cầu ca (Mục 9 case 8).<br/>- **[D — Mất kết nối / timeout]:** không tạo ca cục bộ (BR-011, BR-012); giữ dữ liệu đã nhập; có kết nối thì thử tiếp.<br/>- **[E — Không có quyền / tài khoản bị khóa]:** từ chối tạo ca, thông báo rõ lý do (Mục 9 case 1, 9).<br/>- **[F — Nhấn Mở ca nhiều lần / 2 thiết bị cùng gửi]:** chỉ một ca được tạo; request thừa nhận lại ca hiện tại (BR-008). |

### 1.1. Vị trí của Mở ca trong tổng thể Quản lý ca

- Cấu hình Bật/Tắt là "cửa vào" (spec Bật/Tắt); **Mở ca là nghiệp vụ đầu tiên của vòng đời ca** — mọi giao dịch phát sinh dòng tiền (phạm vi Mục 1.1 spec Bật/Tắt) khi Bật phải thuộc ca đang mở của người thực hiện (BR-shift-management-006).
- Hệ thống không bao giờ tự mở ca (BR-shift-management-005) — mở ca luôn là thao tác chủ động của nhân viên.
- Spec này dùng chung nguyên tắc với spec Bật/Tắt: server là nguồn chân lý; việc tạo ca và đối chiếu ca thực hiện tại server ngay trước khi ghi nhận giao dịch (BR-shift-management-015).

---

## 2. Mô hình nghiệp vụ: ca gắn với ai

### 2.1. Ba phương án

- **Phương án A:** Ca gắn **Nhân viên** trong phạm vi Cửa hàng; thiết bị chỉ được ghi nhận như "nơi mở ca".
- **Phương án B:** Ca gắn **Thiết bị** — mỗi POS có ca riêng, ai đăng nhập bán thì giao dịch vào ca của máy.
- **Phương án C:** Ca gắn kết hợp **Nhân viên + Thiết bị** — mỗi cặp người–máy một ca.

### 2.2. Ma trận so sánh theo tình huống thực tế

| Tình huống thực tế | A — gắn nhân viên | B — gắn thiết bị | C — gắn người + máy |
| :--- | :--- | :--- | :--- |
| Quy trách nhiệm từng dòng tiền về một người | Rõ — người giữ ca chịu trách nhiệm đối soát | Mờ — nhiều người dùng chung một máy trong ngày, tiền quy về máy chứ không về ai | Rõ, nhưng số ca nhân lên theo số máy |
| Một nhân viên làm trên 2 máy cùng lúc | Tự nhiên — một ca duy nhất, giao dịch cả hai máy cùng thuộc ca | Bắt buộc mở 2 ca → vỡ nguyên tắc "một người một ca", tiền đầu ca phải đếm 2 lần | Tương tự B — 2 ca cho một người |
| Đổi máy giữa ca (máy lỗi, hết pin) | Tiếp tục dùng ca trên máy mới | Ca "kẹt" ở máy hỏng — phải dựng cơ chế chuyển ca đặc biệt | Tương tự B |
| Hai nhân viên dùng lần lượt một máy (đăng xuất – đăng nhập) | Mỗi người một ca riêng, sạch sẽ | Máy phải "trả ca" trước khi người sau vào — phát sinh nghiệp vụ ca theo máy | Ca của người trước vẫn mở trên máy; người sau vẫn mở ca riêng — mô hình thừa |
| Diện kiểm thử | Ít tổ hợp — theo người | Nhân theo số máy × người | Nhân theo số máy × người |
| Đối chiếu với spec Bật/Tắt | Khớp BR-shift-management-006 "ca của người thực hiện" | Xung đột — giao dịch thuộc ca máy, không phải ca người | Xung đột một phần |

### 2.3. Kết luận

**Chọn Phương án A: ca gắn Nhân viên trong phạm vi Cửa hàng** (BR-001). Lý do:

1. Giá trị cốt lõi của quản lý ca là đối soát tiền theo trách nhiệm cá nhân — tiền do ai thu thì người đó chốt.
2. Thiết bị chỉ là công cụ; nhân viên đổi máy giữa ca không làm vỡ ca hay mất dữ liệu.
3. Khớp trực tiếp với rule đã chốt ở spec Bật/Tắt: giao dịch thuộc ca của **người thực hiện** (BR-shift-management-006) — mô hình B và C tự mâu thuẫn với rule này.
4. Cũng là mô hình chung của các POS bán lẻ (Loyverse, Square quản lý ca theo nhân viên).

Hệ quả chấp nhận: một cửa hàng có thể có **nhiều ca đang mở song song** (mỗi nhân viên một ca) — hành vi mong muốn với cửa hàng nhiều quầy; và một nhân viên đã có ca mở vẫn bán được trên thiết bị khác (BR-014).

---

## 3. Điều kiện mở ca, quyền và nguyên tắc "Đăng nhập ≠ Mở ca"

### 3.1. Điều kiện bắt buộc

| # | Điều kiện | Rule | Không đạt thì sao |
| :--- | :--- | :--- | :--- |
| 1 | Quản lý ca đang **Bật** tại thời điểm server xử lý | BR-003 | Từ chối tạo ca; khi Tắt thì không cần ca — vào bán hàng bình thường |
| 2 | Tài khoản đang hoạt động, thuộc phạm vi cửa hàng | — | Từ chối với thông báo tài khoản bị khóa (Mục 9 case 9) |
| 3 | Người dùng chưa có ca đang mở hoặc đang đóng dở trong cùng cửa hàng | BR-002 | Không tạo ca mới; hiển thị thông tin ca hiện tại, tiếp tục bán (BR-014) |
| 4 | Thiết bị kết nối được server | BR-012 | Không mở ca khi ngoại tuyến; chờ kết nối rồi thử lại |

### 3.2. Quyền mở ca

- Mọi vai trò trực tiếp bán hàng — Thu ngân, Nhân viên bán hàng, Quản lý cửa hàng, Chủ cửa hàng — đều được mở ca. Mở ca là **điều kiện vào làm việc** khi Quản lý ca Bật, không phải đặc quyền.
- Giai đoạn 1 không tách quyền mở ca riêng và **không có mở ca hộ** — người giữ ca luôn là người tự mở ca (OQ-05). Nếu sau này cần hạn chế cá nhân (vd nhân viên chỉ được bán, không được giữ tiền), xử lý qua màn phân quyền nhân viên — tái xem xét khi có nhu cầu thực.
- Tài khoản bị khóa/ngưng hiệu lực thì không mở được ca kể cả khi trước đó đã đăng nhập (Mục 9 case 9).

### 3.3. Đăng nhập ≠ Mở ca

Đăng nhập thành công **không** đồng nghĩa đang có ca. Khi Quản lý ca Bật và người dùng chưa có ca hợp lệ:

| Khu vực | Chưa mở ca có dùng được không |
| :--- | :--- |
| Duyệt sản phẩm, tra giá, tra tồn, tra khách hàng | Có |
| Tạo giỏ hàng / đơn đang tạo dở | Có — giỏ giữ nguyên; bị chặn ở bước thanh toán (BR-shift-management-013) |
| Cài đặt, xem thông tin tài khoản, đăng xuất | Có |
| Thanh toán đơn hàng, phiếu thu, phiếu chi, thu nợ, hoàn tiền — phạm vi Mục 1.1 spec Bật/Tắt | **Không** — chặn, yêu cầu mở ca (BR-shift-management-005, BR-shift-management-006) |
| Đóng ca | Không áp dụng — chưa có ca thì không có gì để đóng |

---

## 4. Luồng mở ca chuẩn (Happy Flow)

| # | Người dùng làm gì | Hệ thống phản hồi |
| :--- | :--- | :--- |
| 1 | Đăng nhập POS | Đăng nhập thành công, vào màn bán hàng; hệ thống đối chiếu cấu hình + ca với server (BR-shift-management-015); hiển thị banner "Chưa có ca đang mở" + nút Mở ca |
| 2 | (Tùy chọn) duyệt hàng, tạo giỏ | Cho phép; khi chạm Thanh toán thì chặn: "Cần mở ca trước khi thực hiện giao dịch này", giữ nguyên giỏ, mời mở ca |
| 3 | Chạm **Mở ca** (từ banner hoặc từ bước chặn thanh toán) | Mở màn Mở ca: tên cửa hàng, nhân viên, thiết bị, đồng hồ thời gian thực, ô Tiền mặt đầu ca (nếu quản lý tiền mặt), dòng tham khảo ca trước |
| 4 | Nhập tiền mặt đầu ca (hoặc để trống) | Kiểm tra ngay tại máy: là số, ≥ 0, trong hạn mức (BR-006); không hợp lệ thì báo lỗi ngay dưới ô, không gửi |
| 5 | Nhấn **Mở ca** | Nút chuyển "Đang mở ca..." và khóa để chống gửi trùng (BR-008); gửi yêu cầu tạo ca kèm tiền đầu ca |
| 6 | — | Server kiểm tra: cấu hình Bật (BR-003), tài khoản hiệu lực, chưa có ca (BR-002), tiền hợp lệ (BR-006) |
| 7 | — | Tạo ca thành công: sinh mã ca; ghi người mở, mốc mở = giờ server ghi nhận thành công (BR-004), cửa hàng, thiết bị mở, tiền đầu ca; ghi audit (BR-013) |
| 8 | — | POS nhận kết quả: thông báo "Mở ca thành công" + tóm tắt (mã ca, giờ mở, tiền đầu ca); banner ca đang mở; vào/tiếp tục màn bán hàng — giao dịch bị chặn trước đó thực hiện tiếp, thuộc ca mới (BR-010) |

> Thời gian: mở màn hình lúc 08:00 nhưng đến 08:10 mới nhấn Mở ca thì ca bắt đầu **08:10** — mốc lấy tại server lúc tạo thành công, không lấy thời gian mở form (BR-004, Mục 6).

---

## 5. Tiền mặt đầu ca

### 5.1. Phân tích từng câu hỏi

| Câu hỏi | Đề xuất | Lý do |
| :--- | :--- | :--- |
| Có bắt buộc nhập không? | Không bắt buộc; để trống coi là **0** | Cửa hàng không dùng tiền mặt hoặc quầy không có tiền lẻ vẫn mở ca được; ép nhập khi không đếm tiền sẽ sinh số ảo |
| Giá trị mặc định? | **Để trống** — không tự điền số ca trước; hiển thị dòng tham khảo nhỏ "Ca trước của bạn: tiền cuối ca 500.000 (17:45 hôm qua)" | Tự điền tiền ca trước khuyến khích bấm xuyên không đếm tiền, tạo số liệu đối soát ảo; dòng tham khảo vẫn đủ tiện |
| Có cho nhập 0? | Có — 0 là giá trị hợp lệ (ca không dùng tiền quầy) | — |
| Có cho số âm? | Không (BR-006) | Tiền đầu ca là số tiền vật lý có trong quầy |
| Có giới hạn trên? | Theo hạn mức tiền mặt chung của hệ thống; vượt thì báo lỗi tại máy | Chặn lỗi nhập nhầm (thừa số 0) ngay từ đầu |
| Ai được sửa? | Chỉ người mở ca, và chỉ **trước khi nhấn Mở ca** | Ca chưa tồn tại thì chưa có gì để khóa |
| Sửa sau khi mở ca? | **Không sửa trực tiếp** (BR-007) | Tiền đầu ca là căn cứ đối soát cuối ca; sửa trực tiếp sau khi đã phát sinh giao dịch làm mất khả năng truy chênh lệch |
| Cần điều chỉnh thì làm sao? | Tạo **nghiệp vụ điều chỉnh tiền trong ca** (phiếu thu/chi điều chỉnh) — thuộc use case Đối soát/Điều chỉnh tiền | Giữ vết audit: tiền đầu ca gốc + phiếu điều chỉnh rõ người, lý do, thời điểm |

### 5.2. Kết luận

Một ô tiền, một quy tắc ngắn dễ giải thích với thu ngân: **"để trống = 0, không được âm, mở ca rồi không sửa — muốn chỉnh thì lập phiếu"**. Tester kiểm thử theo 3 biên: trống / 0 / vượt hạn mức.

---

## 6. Thời gian mở ca

- Mốc mở ca do **server ghi nhận tại thời điểm tạo ca thành công** (BR-004). Người dùng không nhập; đồng hồ trên màn Mở ca chỉ để tham khảo.
- Form mở 08:00, nhấn Mở ca 08:10 thì ca bắt đầu 08:10; giao dịch ghi nhận từ 08:10 trở đi thuộc ca.

| Tình huống biên | Xử lý |
| :--- | :--- |
| Mở form lâu, đếm tiền xong mới nhấn | Đồng hồ chạy trên form; mốc lấy lúc nhấn thành công — phản ánh thời điểm thật sự bắt đầu bán |
| Đồng hồ thiết bị lệch giờ | Server quyết định; mọi mốc nghiệp vụ lấy giờ server |
| Nhấn sát thời điểm đổi ngày (23:59) | Ca thuộc ngày của mốc server ghi nhận; ca đêm trải qua 2 ngày là hợp lệ — ca không bị chia theo ngày cứng; doanh thu theo ngày phân theo mốc ghi nhận của từng giao dịch (OQ-06) |
| Server phản hồi chậm | Mốc là lúc tạo thành công tại server, không phải lúc máy gửi yêu cầu |
| Hiển thị thời gian | Theo múi giờ cửa hàng |

---

## 7. Đa thiết bị — đa người dùng

### 7.1. Nguyên tắc chung

- Ca gắn nhân viên (Mục 2); server là nguồn chân lý; mọi giao dịch đối chiếu cấu hình + ca với server ngay trước khi ghi nhận (BR-shift-management-015).

### 7.2. Bốn tình huống

**Case 1 — Nhân viên A mở ca trên POS A, sau đó A đăng nhập POS B:**

- POS B đối chiếu server: A đã có ca đang mở nên **không mở ca mới** (BR-002); hiển thị "Bạn có ca đang mở — mở 07:30 trên POS A".
- A bán hàng bình thường trên POS B; giao dịch **thuộc ca đang mở** của A (BR-010, BR-014); thiết bị thực hiện được ghi trong từng giao dịch.
- Lưu ý tiền mặt vật lý: tiền đầu ca đếm tại ngăn kéo POS A; không giới hạn thanh toán tại POS B (OQ-03) — trách nhiệm tiền quy về người giữ ca, đếm tiền nhiều máy xử lý ở use case Đối soát.

**Case 2 — A đang có ca mở, nhân viên B đăng nhập POS B:**

- B là người khác nên **mở ca riêng** — cửa hàng có 2 ca đang mở song song; mỗi giao dịch thuộc ca của người thực hiện (BR-shift-management-006).
- Hợp lệ và mong muốn với cửa hàng nhiều quầy, nhiều nhân viên cùng bán.

**Case 3 — POS A đang có ca của A; A đăng xuất, B đăng nhập POS A:**

- Ca của A **không bị hủy, không tự đóng** khi đăng xuất — ca tồn tại độc lập phiên đăng nhập, chờ A đóng ca ở thiết bị nào cũng được (hoặc quản lý đóng ca hộ theo BR-shift-management-009).
- B **không được dùng ca của A** (BR-009); B muốn thực hiện giao dịch dòng tiền thì mở ca riêng.
- Kết quả: một thiết bị có thể cùng lúc liên quan tới nhiều ca đang mở (ca của A mở từ máy này + ca của B) — hợp lệ vì ca gắn người, không gắn máy.
- A đăng nhập lại trên máy bất kỳ thì thấy ca của mình vẫn mở, tiếp tục bán (đồng Case 1).

**Case 4 — Hai thiết bị cùng gửi yêu cầu mở ca cho cùng một nhân viên gần như đồng thời:**

- Server xử lý kiểm tra "đã có ca?" và tạo ca **nguyên tử** (một thao tác không tách được): request đến trước thắng; request đến sau nhận lại thông tin ca hiện tại, **không coi là lỗi** (BR-008).
- Máy đến sau hiển thị "Ca đã được mở 08:10:02 trên POS A" và vào bán hàng.
- Trên cùng một máy: nút Mở ca khóa ngay khi bắt đầu gửi nên nhấn nhiều lần cũng chỉ một yêu cầu.

---

## 8. Khi cấu hình vừa được bật (từ góc nhìn mở ca)

| Câu hỏi | Hành vi (đối chiếu spec Bật/Tắt) |
| :--- | :--- |
| Khi nào POS bắt đầu yêu cầu mở ca? | Tại **giao dịch dòng tiền đầu tiên** phát sinh sau mốc bật (BR-shift-management-013, BR-shift-management-015); banner trạng thái ca xuất hiện khi thiết bị nhận cấu hình mới qua realtime |
| Có ngắt màn hình đang thao tác không? | Không — giữ nguyên màn hình và thao tác đang làm |
| Đơn đang tạo dở thì sao? | Giữ nguyên giỏ hàng; chặn ở bước thanh toán; nhân viên mở ca xong thanh toán tiếp; toàn bộ tiền thu của đơn thuộc ca mới |
| Sau khi hoàn tất/hủy đơn hiện tại có bắt mở ca không? | Đơn đã ghi nhận trước mốc không thuộc ca (BR-shift-management-008); mọi **giao dịch tiếp theo** (thanh toán mới, phiếu thu, thu nợ…) vẫn bị chặn yêu cầu ca — kể cả trong cùng phiên làm việc |
| Giao dịch mới kiểm tra ca thế nào? | Server đối chiếu cấu hình + ca ngay trước khi ghi nhận (BR-shift-management-015); máy chưa nhận realtime vẫn bị chặn tại server và hiển thị yêu cầu mở ca |

---

## 9. Xử lý lỗi

| # | Tình huống (Trigger) | Hệ thống xử lý | Thông báo | Retry |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Không có quyền mở ca (tài khoản bị loại khỏi phạm vi bán / phân quyền tùy biến nếu có) | Chặn tại máy nếu biết trước; server từ chối `403 PERMISSION_DENIED`, ghi cảnh báo an ninh vào audit | "Tài khoản không có quyền mở ca. Vui lòng liên hệ quản lý." | Không — phải xử lý quyền/tài khoản trước |
| 2 | Ca đã được mở trước đó (phát hiện khi gửi) | Server không tạo ca mới; trả về thông tin ca hiện tại | "Bạn đã có ca đang mở từ {giờ} trên {thiết bị}" + nút Vào bán hàng | Không cần — dùng ca hiện tại |
| 3 | Một request mở ca khác thành công trước (gần đồng thời) | Như mục 2 — chỉ một ca tồn tại (BR-008) | "Ca đã được mở lúc {giờ} trên {thiết bị}" | Không cần |
| 4 | Mất kết nối server trước/trong khi gửi | Không gửi được; **không tạo ca cục bộ** (BR-012); giữ nguyên dữ liệu đã nhập trên form | "Mất kết nối. Không thể mở ca khi ngoại tuyến — vui lòng kiểm tra mạng." | Có — nút Thử lại khi có mạng; dữ liệu giữ nguyên |
| 5 | Server timeout — không rõ kết quả | Máy **đối chiếu lại trạng thái ca** với server: đã có ca thì vào bán hàng; chưa có thì cho gửi lại (BR-011) | "Đang kiểm tra kết quả mở ca…" rồi thông báo tương ứng | Có — tự đối chiếu, nhân viên không phải tự đoán |
| 6 | Tạo ca thất bại (lỗi hệ thống phía server) | Không có ca được tạo (BR-011); giữ dữ liệu form; ghi log hệ thống | "Không mở được ca lúc này. Vui lòng thử lại." kèm mã lỗi | Có |
| 7 | Tiền đầu ca không hợp lệ (âm / không phải số / vượt hạn mức / sai định dạng) | Chặn **ngay tại máy** trước khi gửi; server kiểm tra lại (BR-006) | Báo đỏ dưới ô nhập: "Tiền đầu ca không được âm" v.v. | Sửa tại chỗ rồi gửi — không phải retry |
| 8 | Quản lý ca bị Tắt trong lúc đang ở màn Mở ca | Realtime: màn Mở ca tự đóng, về bán hàng thường. Nếu nhấn Mở ca trước khi màn kịp đóng: server từ chối vì cấu hình Tắt (BR-003) | "Quản lý ca đã tắt — không cần mở ca." | Không cần — bán hàng không còn yêu cầu ca |
| 9 | Người dùng bị khóa/ngưng quyền trong lúc mở ca | Server từ chối tạo ca; phiên xử lý theo chính sách tài khoản | "Tài khoản đã bị khóa. Lý do: {lý do quản lý ghi khi khóa}. Vui lòng liên hệ quản lý." | Không |
| 10 | Nhấn Mở ca nhiều lần (double submit) | Nút khóa ngay khi bắt đầu gửi; server idempotent — request trùng không tạo ca thứ hai, trả ca hiện tại (BR-008) | Vô hình với người dùng nếu lần đầu thành công | — |

---

## 10. Dữ liệu ghi nhận của một ca & trạng thái ca

### 10.1. Thông tin nghiệp vụ của một ca

| Thông tin | Ghi nhận lúc | Ghi chú |
| :--- | :--- | :--- |
| Mã ca (Shift ID) | Mở | Định danh duy nhất toàn hệ thống; hiển thị thân thiện với thu ngân dạng "Ca #23 · mở 08:10" (số thứ tự tự tăng theo cửa hàng + giờ mở) |
| Cửa hàng | Mở | Phạm vi hoạt động của ca |
| Nhân viên mở ca | Mở | Người giữ ca — chịu trách nhiệm đối soát |
| Thời điểm mở | Mở | Mốc server ghi nhận tạo ca thành công (BR-004) |
| Thiết bị mở ca | Mở | Chỉ để audit — ca không khóa vào thiết bị (BR-001) |
| Tiền mặt đầu ca | Mở | Nếu cửa hàng quản lý tiền mặt; để trống = 0 (BR-006) |
| Liên kết các giao dịch thuộc ca | Suốt ca | Mỗi giao dịch dòng tiền tham chiếu mã ca (BR-010); thiết bị thực hiện ghi trong từng giao dịch |
| Thời điểm đóng, người đóng/đóng ca hộ + lý do, tiền cuối ca, chênh lệch | Đóng | Thuộc use case Đóng ca — liệt kê để thấy đủ bức tranh |

### 10.2. Trạng thái ca

| Trạng thái | Ý nghĩa | Cho phép gì |
| :--- | :--- | :--- |
| **Đang mở** | Ca đã tạo thành công, chưa bắt đầu đóng | Nhận giao dịch dòng tiền của người giữ ca (BR-010) |
| **Đang đóng dở** (đang đối soát) — *trạng thái quy trình trên giao diện, không phải trạng thái dữ liệu — server vẫn lưu "Đang mở" (đối chiếu use case Đóng ca)* | Người giữ ca/quản lý đang mở màn đối soát nhưng chưa Hoàn tất | **Vẫn tính là đang mở** với mọi ràng buộc (đối chiếu Edge case 5 spec Bật/Tắt): chưa mở được ca mới (BR-002), chưa tắt được cấu hình (BR-shift-management-009) — không đặt thời hạn ở phạm vi này (OQ-08) |
| **Đã đóng** | Đối soát hoàn tất, ca kết thúc | Chỉ đọc — xem lịch sử, báo cáo; không mở lại — muốn làm tiếp thì mở ca mới |

- Không có trạng thái "nháp" hay "hủy": mở ca thất bại đồng nghĩa **ca không tồn tại** (BR-011); ca đã đóng không được mở lại.

---

## 11. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

> ID đầy đủ có prefix feature theo quy ước đặt tên: `BR-001` dưới đây tương ứng `BR-open-shift-001`. Rule kế thừa từ spec Bật/Tắt được dẫn dạng đầy đủ `BR-shift-management-NNN`.

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-001** | **[Constraint] Ca gắn nhân viên trong cửa hàng** | Ca thuộc về một nhân viên trong phạm vi một cửa hàng; thiết bị chỉ được ghi nhận là nơi mở ca, không là điều kiện thuộc ca. Một cửa hàng có nhiều ca đang mở song song của các nhân viên khác nhau. | Không tồn tại ca "của thiết bị"; ca không khóa vào máy mở. |
| **BR-002** | **[Constraint] Một nhân viên — một ca đang mở** | Một nhân viên không được có nhiều ca đang mở đồng thời trong cùng một cửa hàng, kể cả khi đăng nhập trên nhiều thiết bị. Ca **đang đóng dở** vẫn tính là đang mở. | Yêu cầu tạo ca thứ hai: server từ chối, trả thông tin ca hiện tại (BR-008). |
| **BR-003** | **[Enablement] Chỉ tạo ca khi Quản lý ca đang Bật** | Không được tạo ca nếu cấu hình đang Tắt tại thời điểm server xử lý yêu cầu (kế thừa BR-shift-management-002, BR-shift-management-010). | Từ chối tạo ca; người dùng không cần ca — vào bán hàng bình thường. |
| **BR-004** | **[Derivation] Thời điểm mở ca theo server** | Thời điểm mở ca do server ghi nhận tại thời điểm tạo ca thành công; người dùng không nhập tay; không dùng thời gian mở màn hình. Hiển thị theo múi giờ cửa hàng. | Không áp dụng — không tồn tại đường nhập thời gian mở. |
| **BR-005** | **[Constraint] Mở ca là thao tác chủ động** | Hệ thống không tự mở ca theo lịch, theo đăng nhập hay ngầm định (kế thừa BR-shift-management-005); hệ thống chỉ hướng dẫn và chặn, người dùng tự mở. | Không áp dụng — không có cơ chế tự mở. |
| **BR-006** | **[Constraint] Tiền mặt đầu ca hợp lệ** | Tiền mặt đầu ca ≥ 0, là số hợp lệ, trong hạn mức tiền mặt của hệ thống; để trống coi là 0; không âm. | Chặn tại máy trước khi gửi; server kiểm tra lại khi nhận yêu cầu. |
| **BR-007** | **[Constraint] Không sửa tiền đầu ca sau khi mở** | Tiền đầu ca cố định sau khi ca tạo thành công. Cần điều chỉnh thì dùng nghiệp vụ điều chỉnh tiền trong ca (phiếu thu/chi điều chỉnh, thuộc use case Đối soát/Điều chỉnh) — không sửa trực tiếp. | Thao tác sửa trực tiếp: không tồn tại trong sản phẩm. |
| **BR-008** | **[Constraint] Mở ca idempotent — chống trùng ca** | Một yêu cầu mở ca trùng (nhấn nhiều lần, hai thiết bị cùng gửi) không tạo ca thứ hai: server kiểm tra "đã có ca?" và tạo ca trong một thao tác nguyên tử; request đến sau nhận lại thông tin ca hiện tại. Giao diện khóa nút trong lúc gửi. | Không xảy ra trùng ca; request thừa không bị coi là lỗi. |
| **BR-009** | **[Constraint] Không dùng ca của người khác** | Giao dịch chỉ thuộc ca của chính người thực hiện; không dùng, không "mượn", không chuyển nhượng ca giữa nhân viên. (Đóng ca hộ là nghiệp vụ khác — quản lý kết thúc ca hộ người giữ ca, không phải dùng ca.) | Giao dịch gắn ca của người không thực hiện: không tồn tại đường ghi nhận như vậy. |
| **BR-010** | **[Derivation] Giao dịch sau mốc mở thuộc ca** | Mọi giao dịch phát sinh dòng tiền (phạm vi Mục 1.1 spec Bật/Tắt) ghi nhận sau mốc mở ca thành công phải tham chiếu đúng mã ca của người thực hiện. | Giao dịch không tham chiếu ca khi bắt buộc: chặn tại server (đối chiếu BR-shift-management-006). |
| **BR-011** | **[Constraint] Thất bại = chưa có ca** | Mở ca thất bại (lỗi, timeout, mất kết nối) thì người dùng **không** được coi là đang có ca; trạng thái ca luôn lấy từ server, không "giả có ca" tại máy. | Không có ca cục bộ; mọi kiểm tra đối chiếu server. |
| **BR-012** | **[Constraint] Không xử lý mở ca offline** | Yêu cầu tạo ca phải tới được server; không mở ca khi ngoại tuyến, không xếp hàng chờ gửi khi có mạng lại. (Giao dịch offline nói chung theo quy trình offline hiện có — BR-shift-management-016.) | Không tạo ca cục bộ; chờ kết nối rồi gửi lại. |
| **BR-013** | **[Audit] Ghi nhận mở ca** | Mỗi lần mở ca (thành công lẫn từ chối quan trọng) ghi audit: người mở, mốc mở, cửa hàng, thiết bị, tiền đầu ca, kết quả. | Lỗi ghi log: vẫn trả kết quả mở ca cho người dùng, phát cảnh báo hệ thống. |
| **BR-014** | **[Constraint] Tiếp tục ca trên thiết bị khác** | Nhân viên đã có ca đang mở đăng nhập thiết bị khác: không mở ca mới, tiếp tục dùng ca hiện tại; giao dịch tại thiết bị mới vẫn thuộc ca. | Yêu cầu mở ca thứ hai trên thiết bị khác: từ chối, hiển thị thông tin ca hiện tại. |
| **BR-015** | **[Constraint] Đăng nhập ≠ mở ca** | Đăng nhập thành công không tạo ca và cũng không bắt đăng xuất khi chưa mở ca; khu vực được dùng khi chưa có ca liệt kê tại Mục 3.3, giao dịch dòng tiền bị chặn đến khi có ca. | Không áp dụng — thiết kế không có cửa "đăng nhập là có ca". |

> **Ánh xạ với đề xuất ban đầu:** BR-SHIFT-OPEN-01 → BR-002, 02 → BR-003, 03 → BR-004, 04 → BR-006, 05 → BR-008, 06 → BR-009, 07 → BR-010, 08 → BR-011, 09 → BR-012. Sáu rule bổ sung: BR-001, BR-005, BR-007, BR-013, BR-014, BR-015.

---

## 12. Sơ đồ tương tác (Interaction Diagram)

*(Sơ đồ đầy đủ tại `docs/open-shift/srs/flows.md`)*

![[flows.md]]

---

## 13. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/open-shift/srs/screens.md`)*

![[screens.md]]

---

## 14. Open Questions

- [x] **OQ-01:** Tiền mặt đầu ca — có cần cấu hình "bắt buộc đếm tiền đầu ca" cho cửa hàng muốn chặt không? — **Resolved:** Không bắt buộc; giữ một hành vi duy nhất (để trống = 0), không dựng cấu hình phụ — cửa hàng muốn chặt xử lý bằng quy định nội bộ (Mục 5.1, BR-006).
- [x] **OQ-02:** Mã ca hiển thị với thu ngân dạng nào (số thứ tự #, giờ mở, hay không hiển thị)? — **Resolved:** Kết hợp "Ca #23 · mở 08:10" — số thứ tự tự tăng theo cửa hàng kèm giờ mở; mã nội bộ Shift ID giữ nguyên cho hệ thống (Mục 10.1).
- [x] **OQ-03:** Nhân viên tiếp tục ca trên thiết bị khác (BR-014) — tiền vật lý ở ngăn kéo máy cũ; có cần giới hạn thanh toán tiền mặt khi bán ở máy khác không? — **Resolved:** Không giới hạn phương thức thanh toán; trách nhiệm tiền quy về người giữ ca; việc đếm tiền ở nhiều máy xử lý ở use case Đối soát (Mục 7.2 Case 1, BR-014).
- [~] **OQ-04:** Hạn mức trên tiền đầu ca lấy theo cấu hình nào — hằng số hệ thống hay cấu hình theo cửa hàng? — **Out of scope:** không xây cấu hình hạn mức theo cửa hàng; giữ hạn mức **hằng số hệ thống** — chỉ để chặn nhập nhầm (BR-006). Chốt chung với OQ-04 use case Đóng ca (resolved via /srs).
- [x] **OQ-05:** Có cần "mở ca hộ" (quản lý mở ca thay, người giữ ca vẫn là nhân viên)? — **Resolved:** Không có mở ca hộ — người giữ ca luôn là người tự mở; quản lý cần bán thì mở ca của chính mình. Chủ ý không đối xứng với đóng ca hộ (BR-shift-management-009): đóng hộ là giải cứu ca quên đóng, mở hộ là tạo trách nhiệm tiền thay người khác (Mục 3.2).
- [~] **OQ-06:** Ca đêm trải 2 ngày — báo cáo doanh thu theo ngày có cần quy ước phân ca đêm không? — **Out of scope (chuyển use case Báo cáo/Lịch sử ca):** giả định làm việc — doanh thu theo ngày phân theo mốc ghi nhận của từng giao dịch (giao dịch sau 00:00 thuộc ngày sau dù cùng ca đêm); ca là nhóm trách nhiệm, không phải khung thời gian báo cáo (Mục 6).
- [x] **OQ-07:** Tài khoản bị khóa giữa chừng (Mục 9 case 9) — có hiển thị lý do khóa cho người dùng không? — **Resolved:** Có, hiển thị rõ lý do: "Tài khoản đã bị khóa. Lý do: {lý do quản lý ghi khi khóa}. Vui lòng liên hệ quản lý." (Mục 9 case 9).
- [x] **OQ-08:** Ca đang đóng dở chặn mở ca mới (BR-002) — thời gian đóng dở tối đa bao lâu trước khi cảnh báo quản lý? — **Resolved:** Không đặt thời hạn trong phạm vi Mở ca — BR-002 giữ nguyên tắc "đang đóng dở = đang mở"; cảnh báo ca mở quá lâu/quên đóng: giai đoạn 1 không làm, nếu làm sau này thì ở Lịch sử ca/Báo cáo (OQ-06 use case Đóng ca, resolved via /srs).

---

## 15. References

- `@../../rules/ba-conventions.md`
- `@../../rules/approval-gate.md`
- Spec Bật/Tắt Quản lý ca: `docs/shift-management/srs/spec.md` (các rule dẫn dạng `BR-shift-management-NNN`)
- Use case kế tiếp (chưa có): Đóng ca, Đối soát tiền, Lịch sử ca
- Loyverse — Shift management: https://loyversehelp.com
- Square — Cash drawer shift reports: https://www.squareup.com
- Sapo POS — giao ca: https://support.sapo.vn
