---
feature: stock-history
type: srs-spec
version: 1.0.0
updated: 2026-08-25
status: draft
authors: [BA Team]
changelog:
  - 2026-08-25 | /srs-spec | resolved OQ-01..06 (offline, quyền xem, combo, giá vốn, lưu trữ, import)
  - 2026-08-25 | /screen   | [screens] cascade từ OQ resolved: bỏ mốc offline, thêm ghi chú combo
  - 2026-08-25 | /srs-spec | [spec] initialized stock-history specification, 12 movement types, 18 business rules
  - 2026-08-25 | /srs-spec | [flows] added sequence + state diagrams for stock-history
  - 2026-08-25 | /screen   | [screens] added 3 screens and navigation for stock-history
---

# Đặc tả tính năng: Lịch sử thay đổi tồn kho (Stock Movement History)

## 1. Tổng quan nghiệp vụ (Business Overview)

| Thông tin nghiệp vụ | Chi tiết đặc tả |
| :--- | :--- |
| **1. Mục đích & Phạm vi** | Ghi nhận **tự động và đầy đủ** mọi biến động tồn kho theo từng **sản phẩm/biến thể** tại từng **kho/cửa hàng**, hoạt động như một **sổ cái tồn kho chỉ-đọc** (chỉ thêm bản ghi mới, không sửa/xóa bản ghi cũ) phục vụ truy vết, đối chiếu và kiểm soát chênh lệch. Phạm vi bao gồm: (1) quy tắc ghi nhận cho mọi nghiệp vụ tác động tồn — nhập hàng, hủy phiếu nhập, bán hàng, hủy/hoàn đơn, khách trả hàng, trả hàng nhà cung cấp, chuyển kho, kiểm kho, điều chỉnh trực tiếp; (2) màn hình tra cứu lịch sử với bộ lọc và điều hướng chứng từ nguồn; (3) quy tắc điều chỉnh tồn trực tiếp từ màn Sửa sản phẩm. **Ranh giới (ngoài phạm vi):** tính giá vốn xuất kho (FIFO/trung bình), đối soát kế toán cuối kỳ, dự báo nhu cầu nhập hàng, điều chỉnh tồn hàng loạt qua import (giai đoạn 2 — OQ-06), và việc thay đổi tồn từ từng nghiệp vụ nguồn (thuộc đặc tả nghiệp vụ đó). |
| **2. Actors (Tác nhân)** | **Chính:** Quản lý cửa hàng (Store Manager) — tra cứu lịch sử, điều chỉnh tồn trực tiếp, bật/tắt "Cho phép tồn âm" theo kho; Thu ngân/Nhân viên bán hàng (Cashier) — tạo biến động qua bán/hủy/hoàn đơn, tra cứu lịch sử.<br/>**Phụ trợ:** Nhân viên kho — tạo biến động qua nhập hàng, trả hàng NCC, chuyển kho, kiểm kho; Quản lý chuỗi (Chain Manager) — tra cứu mọi kho, cấu hình toàn hệ thống; Kế toán — chỉ xem phục vụ đối chiếu.<br/>**Hệ thống:** Sổ cái tồn kho (Stock Ledger) — nơi ghi bản ghi; các nguồn phát sinh — Phiếu nhập, Phiếu trả NCC, Phiếu trả hàng, Đơn hàng, Phiếu chuyển kho, Phiếu kiểm kho, màn Sửa sản phẩm. |
| **3. Pre-conditions** | 1. Sản phẩm/biến thể đã có trong hệ thống, có mã SKU và được khai báo tồn cho ít nhất một kho/cửa hàng.<br/>2. Người thao tác đã **đăng nhập** — mọi bản ghi lịch sử đều bắt buộc có người thực hiện.<br/>3. Chứng từ nguồn đã rời trạng thái nháp, tức bắt đầu tác động tồn (BR-05, BR-07).<br/>4. Người tra cứu được cấp quyền **"Xem lịch sử tồn kho"**; phạm vi dữ liệu là các kho người dùng được gán (OQ-03). |
| **4. Expected Results** | **Happy Path — bán rồi hủy đơn (kiểu đảo ngược, không xóa):**<br/>1. Thu ngân tạo đơn bán 2 sản phẩm tại quầy → hệ thống trừ tồn từng dòng và ghi 1 bản ghi: loại "Bán hàng", tồn trước 20, thay đổi −2, tồn sau 18, chứng từ #ĐH, người thực hiện.<br/>2. Khách đổi ý, thu ngân hủy đơn trước thanh toán → hệ thống ghi bản ghi NGƯỢC DẤU: loại "Hủy đơn trước thanh toán", 18 → +2 → 20, tham chiếu cùng #ĐH.<br/>3. Danh sách lịch sử hiển thị **2 bản ghi riêng biệt** — bản ghi bán không bị xóa hay sửa (BR-06).<br/><br/>**Alternate Branches (Luồng rẽ nhánh & Ngoại lệ):**<br/>- **[A — Điều chỉnh trực tiếp]:** Quản lý sửa số tồn 10 → 15 tại Sửa sản phẩm, chọn lý do → ghi 1 bản ghi "Điều chỉnh tồn kho" +5, tồn sau 15; sửa 10 → 7 ghi −3.<br/>- **[B — Số lượng không đổi]:** Lưu lại 10 → 10 → không sinh bản ghi nào (BR-11).<br/>- **[C — Chứng từ còn nháp]:** Phiếu nhập mới tạo chưa xác nhận → chưa ghi; hủy nháp → không có gì được ghi.<br/>- **[D — Hai thiết bị cùng lúc]:** Hai quầy bán cùng sản phẩm gần như đồng thời → ghi tuần tự 20 → 18 → 16, không xuất hiện 2 bản ghi cùng "tồn trước 20" (BR-14).<br/>- **[E — Chuyển kho đang vận chuyển]:** Đã xuất ở kho đi (−) nhưng kho đến chưa nhận → tồn kho đi đã giảm, kho đến chưa tăng; số đang chuyển không bán được ở cả hai kho (BR-10).<br/>- **[F — Chứng từ nguồn đã hủy]:** Mở bản ghi lịch sử của phiếu nhập đã hủy → bản ghi vẫn hiển thị; liên kết chứng từ mở được ở trạng thái "Đã hủy" (chứng từ không xóa cứng).<br/>- **[G — Vượt tồn khi chặn âm]:** Bán 5 khi tồn 3 ở kho chưa bật "Cho phép tồn âm" → chặn, thông báo tồn hiện tại (BR-13). |

### 1.1. Danh mục loại biến động (12 loại)

| # | Loại biến động | Chiều | Nguồn phát sinh |
| :--- | :--- | :--- | :--- |
| 1 | Tồn đầu kỳ | Cộng | Khởi tạo sản phẩm / gắn kho mới có nhập tồn ban đầu |
| 2 | Nhập hàng | Cộng | Phiếu nhập được xác nhận |
| 3 | Hủy phiếu nhập | Trừ | Hủy phiếu nhập đã xác nhận |
| 4 | Bán hàng | Trừ | Tạo đơn bán tại quầy |
| 5 | Sửa đơn đang mở | Cộng/Trừ | Sửa số lượng dòng trên đơn chưa thanh toán |
| 6 | Hủy đơn trước thanh toán | Cộng | Hủy đơn chưa thanh toán |
| 7 | Khách trả hàng | Cộng | Phiếu trả hàng sau thanh toán |
| 8 | Trả hàng nhà cung cấp | Trừ | Phiếu trả NCC được xác nhận |
| 9 | Chuyển kho — xuất | Trừ (tại kho đi) | Xác nhận xuất chuyển |
| 10 | Chuyển kho — nhập | Cộng (tại kho đến) | Kho đến xác nhận đã nhận |
| 11 | Kiểm kho | Cộng/Trừ | Phiếu kiểm kho được xác nhận |
| 12 | Điều chỉnh tồn kho | Cộng/Trừ | Sửa trực tiếp số tồn trên Sửa sản phẩm |

### 1.2. Bản đồ nghiệp vụ làm thay đổi tồn kho

| Nghiệp vụ | Thời điểm ghi nhận | Số lượng ghi | Chứng từ tham chiếu | Khi chứng từ bị sửa | Khi chứng từ bị hủy |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Khởi tạo tồn đầu kỳ | Khi lưu sản phẩm mới / gắn kho mới có nhập tồn ban đầu | Tồn khởi tạo (> 0) | Màn Sửa sản phẩm (tạo mới) | Điều chỉnh lại tồn đầu = ghi bản ghi "Điều chỉnh tồn kho", không sửa bản ghi Tồn đầu kỳ (BR-01) | Chưa phát sinh giao dịch nào khác: xóa sản phẩm xóa luôn bản ghi; đã phát sinh: không xóa được (BR-16) |
| Nhập hàng | Khi phiếu nhập được **xác nhận** (nhận hàng vào kho), không phải lúc tạo nháp | (+) Số thực nhận từng dòng | Phiếu nhập | Không sửa số lượng sau xác nhận (BR-08) — hiệu chỉnh bằng hủy + lập phiếu mới | Hủy sau xác nhận: ghi đảo ngược (−) từng dòng; hủy khi còn nháp: không ghi gì |
| Bán hàng tại quầy | Khi **tạo đơn** (chọn món) — nhất quán với split-order BR-16 | (−) Số lượng từng dòng | Đơn hàng | Thay đổi số lượng trên đơn đang mở: xem dòng "Sửa đơn đang mở" | Hủy trước thanh toán: (+) hoàn từng dòng |
| Sửa số lượng trên đơn đang mở | Khi lưu thay đổi dòng | (+/−) đúng phần chênh lệch của dòng | Đơn hàng | Không áp dụng | Đơn bị hủy: xem dòng "Bán hàng tại quầy" |
| Thanh toán đơn | Không sinh bản ghi — tồn đã trừ khi tạo đơn | — | — | — | — |
| Khách trả hàng / hoàn đơn sau thanh toán | Khi phiếu trả hàng được xác nhận | (+) Số nhận lại còn bán được | Phiếu trả hàng | Không sửa sau xác nhận (BR-08) | Hủy phiếu trả đã xác nhận: (−) đảo ngược |
| Hàng hoàn bị hỏng (hoàn tiền, không nhập kho) | Không sinh bản ghi tồn (BR-18) | — | Phiếu trả hàng | Không áp dụng | Không áp dụng |
| Trả hàng nhà cung cấp | Khi phiếu trả NCC được xác nhận (xuất trả) | (−) Số trả từng dòng | Phiếu trả NCC | Không sửa sau xác nhận (BR-08) | Hủy sau xác nhận: (+) đảo ngược |
| Chuyển kho — xuất | Khi xác nhận xuất tại kho đi | (−) Số xuất | Phiếu chuyển kho | Không sửa số lượng sau xuất | Hủy sau xuất: (+) nhập lại kho đi; hủy trước xuất: không ghi |
| Chuyển kho — nhập | Khi kho đến xác nhận đã nhận | (+) Số thực nhận | Phiếu chuyển kho | Không áp dụng | Kho đến từ chối nhận: hàng quay lại kho đi |
| Kiểm kho | Khi phiếu kiểm kho được xác nhận | (+/−) hiệu số = số đếm thực − số trên hệ thống, từng dòng | Phiếu kiểm kho | Không sửa sau xác nhận (BR-09) — sai thì lập phiếu kiểm mới | Hủy phiếu kiểm đã xác nhận: đảo ngược từng dòng |
| Điều chỉnh tồn trực tiếp | Khi lưu Sửa sản phẩm với số mới ≠ số cũ | (+/−) = mới − cũ | Màn Sửa sản phẩm | Không áp dụng | Không có hủy — điều chỉnh lại = ghi bản ghi mới |

### 1.3. Nghiệp vụ KHÔNG làm thay đổi tồn kho

| Nghiệp vụ | Lý do không ghi lịch sử tồn |
| :--- | :--- |
| Đặt hàng NCC chưa nhập kho | Tồn chỉ đổi khi xác nhận nhập — đơn đặt hàng là cam kết, chưa phải biến động |
| Tách đơn | Tồn đã trừ khi tạo đơn gốc; tách chỉ đổi phân bổ đơn — [[docs/split-order/srs/spec\|split-order BR-16]] |
| Gộp đơn | Tương tự — không cộng/trừ thêm — [[docs/merge-order/srs/spec\|merge-order BR-20]] |
| Giữ hàng / đặt trước cho khách | Đổi "số có thể bán", không đổi tồn thực |
| Sửa thông tin sản phẩm (tên, giá, mô tả, hình ảnh) | Không chạm số lượng |
| Thanh toán đơn tại quầy | Đã trừ khi tạo đơn (Mục 1.2) |
| Xem / xuất báo cáo, tra cứu lịch sử | Thao tác đọc |

---

## 2. Quy tắc nghiệp vụ cốt lõi (Core Business Rules)

| # | Quy tắc | Mô tả | Xử lý vi phạm |
| :--- | :--- | :--- | :--- |
| **BR-01** | **[Constraint] Sổ chỉ-đọc vĩnh viễn** | Bản ghi lịch sử tồn không được sửa hoặc xóa bởi bất kỳ vai trò nào (thu ngân, quản lý cửa hàng, quản lý chuỗi, kể cả admin). Hệ thống không cung cấp thao tác sửa/xóa trên màn hình lịch sử; dữ liệu lưu trữ không giới hạn thời gian (OQ-02). | Không tồn tại đường thao tác trong sản phẩm; mọi yêu cầu chỉnh dữ liệu lịch sử thuộc quy trình kỹ thuật đặc biệt có ghi nhận phê duyệt — ngoài phạm vi tính năng. |
| **BR-02** | **[Derivation] Bất biến chuỗi bản ghi** | Mỗi bản ghi thỏa: **Tồn sau = Tồn trước + Số lượng thay đổi**; tồn sau của bản ghi là tồn trước của bản kế tiếp cùng sản phẩm-biến thể-kho. Mỗi chuỗi có số thứ tự bản ghi tăng dần để phát hiện đứt đoạn. | Khi ghi mới mà công thức không khớp: chặn ghi, giữ nguyên tồn, cảnh báo dữ liệu lệch cho kỹ thuật xử lý; tuyệt đối không ghi đè. |
| **BR-03** | **[Constraint] Mức ghi nhận theo biến thể × kho** | Một bản ghi tương ứng một SKU-biến thể tại một kho/cửa hàng. Sản phẩm nhiều biến thể ghi từng biến thể riêng; giao dịch nhiều dòng sinh nhiều bản ghi. Bán combo/gói: sinh bản ghi riêng cho từng linh kiện, tham chiếu cùng dòng combo trên đơn; combo không có tồn riêng (OQ-04). | Giao dịch gộp nhiều dòng/số kho: hệ thống tự tách ghi đúng mức biến thể-kho; không tồn tại bản ghi mức sản phẩm cha. |
| **BR-04** | **[Data Validation] Đủ 10 thông tin bắt buộc** | Mỗi bản ghi bắt buộc: (1) thời gian thay đổi, (2) sản phẩm/biến thể, (3) SKU, (4) kho/cửa hàng, (5) loại biến động (12 loại Mục 1.1), (6) tồn trước, (7) số lượng thay đổi (+/−), (8) tồn sau, (9) chứng từ/nguồn phát sinh, (10) người thực hiện. Lý do bắt buộc bổ sung với loại Điều chỉnh tồn kho (BR-12). | Thiếu trường nào: chặn hoàn tất thao tác gốc, hiển thị thông báo lỗi; không tạo bản ghi thiếu thông tin ("bản ghi mồ côi"). |
| **BR-05** | **[Action Enabler] Chỉ ghi khi tồn thật sự đổi** | Chứng từ ở trạng thái nháp/chờ xác nhận chưa ghi; thao tác giữ nguyên số lượng không ghi. | Yêu cầu ghi rỗng bị bỏ qua; chứng từ chỉ bắt đầu ghi khi chuyển sang trạng thái đã tác động tồn (BR-07). |
| **BR-06** | **[Constraint] Hoàn tác ghi đảo ngược, không xóa** | Hủy/hoàn chứng từ đã tác động tồn sinh **bản ghi mới ngược dấu** tại thời điểm hủy, tham chiếu cùng chứng từ; bản ghi gốc giữ nguyên. Ví dụ: bán 20 → 18, hủy đơn 18 → 20 — hai bản ghi cùng tồn tại. | Nghiệp vụ nguồn không được xóa/sửa bản ghi gốc (bị chặn theo BR-01); chỉ được ghi bổ sung bản đảo ngược. |
| **BR-07** | **[State Transition] Trạng thái chứng từ nguồn** | Mỗi chứng từ thuộc 1 trong 2 nhóm: **Chưa tác động tồn** (nháp, chờ xác nhận) và **Đã tác động tồn** (đã xác nhận). Chuyển một chiều từ Chưa sang Đã; từ Đã chỉ có thể hủy (kèm đảo ngược), không quay về nháp. | Thao tác đưa chứng từ đã xác nhận về nháp: không tồn tại; hệ thống hướng dẫn hủy + lập phiếu mới. |
| **BR-08** | **[Constraint] Không sửa số lượng sau xác nhận** | Phiếu nhập, phiếu trả NCC, phiếu trả hàng đã xác nhận không cho sửa số lượng dòng. Hiệu chỉnh bằng hủy phiếu (ghi đảo ngược toàn bộ) + lập phiếu mới. | Nút sửa số lượng vô hiệu sau xác nhận, kèm tooltip hướng dẫn hiệu chỉnh. |
| **BR-09** | **[Derivation] Kiểm kho ghi hiệu số từng dòng** | Thay đổi = số thực đếm − số trên hệ thống tại thời điểm xác nhận phiếu kiểm; dòng khớp không sinh bản ghi. Phiếu kiểm đã xác nhận không sửa — phát hiện sai thì lập phiếu kiểm mới. | Sửa phiếu kiểm đã xác nhận: bị chặn; hướng dẫn lập phiếu mới. |
| **BR-10** | **[State Transition] Chuyển kho hai bước** | Xác nhận xuất: trừ kho đi ngay (loại "Chuyển kho — xuất"). Kho đến xác nhận nhận: cộng kho đến (loại "Chuyển kho — nhập"). Giữa hai bước, số đang chuyển **không bán được ở cả hai kho**. Hủy khi đang chuyển: cộng nhập lại kho đi. | Bán hàng trên số đang chuyển: chặn với thông báo số chưa về kho, không thể bán. |
| **BR-11** | **[Derivation] Điều chỉnh trực tiếp** | Trên Sửa sản phẩm: thay đổi = số mới − số hiện tại tại thời điểm lưu; lưu cùng giá trị không sinh bản ghi. Nguồn phát sinh ghi "Sửa sản phẩm". | Không áp dụng — quy tắc suy diễn thuần, không có thao tác vi phạm. |
| **BR-12** | **[Data Validation] Lý do điều chỉnh bắt buộc** | Điều chỉnh tồn trực tiếp phải chọn lý do thuộc danh mục: **Nhập sai ban đầu / Hỏng, hết hạn / Thất lạc, thất thoát / Kiểm đếm lại / Khác** (nhập tự do tối thiểu 3 ký tự). | Chưa chọn lý do hoặc "Khác" bỏ trống: chặn lưu, con trỏ nhảy về trường lý do. |
| **BR-13** | **[Constraint] Chặn tồn âm mặc định** | Mỗi kho có cấu hình "Cho phép tồn âm" (mặc định TẮT). Kho chưa bật: giao dịch làm tồn sau < 0 bị chặn trước khi ghi. Kho đã bật: cho phép ghi âm, tồn hiển thị màu cảnh báo. | Thao tác làm âm ở kho chưa bật: chặn, hiển thị tồn hiện tại + gợi ý kiểm tra hoặc liên hệ quản lý bật cấu hình. |
| **BR-14** | **[Constraint] Đồng thời nhiều thiết bị** | Các thay đổi trên cùng sản phẩm-biến thể-kho được xử lý **tuần tự** (xếp hàng): mỗi bản ghi nhận tồn trước từ kết quả của bản ghi ngay trước đó — chuỗi không đứt, không đè số. | Hệ thống phát hiện trùng số thứ tự hoặc đứt chuỗi: cảnh báo dữ liệu lệch, tạm chặn ghi chuỗi đó đến khi xử lý xong. |
| **BR-15** | **[Action Enabler] Cảnh báo tồn đã đổi khi điều chỉnh** | Khi lưu điều chỉnh trực tiếp, nếu tồn hiện tại khác giá trị hiển thị lúc mở màn hình: cảnh báo "Tồn đã thay đổi từ X thành Y bởi ai, lúc nào", yêu cầu xác nhận số mới tuyệt đối trước khi ghi (ví dụ 10 → 15 nhưng tồn thật đã về 8: ghi 8 → 15). | Lưu mà chưa xác nhận cảnh báo: không ghi; màn hình làm mới tồn hiện tại. |
| **BR-16** | **[Constraint] Sản phẩm ngưng kinh doanh** | Lịch sử của sản phẩm ngưng kinh doanh vẫn tra cứu đầy đủ kèm nhãn trạng thái; không xóa cứng sản phẩm/biến thể đã có lịch sử (chỉ ẩn khỏi danh mục bán). | Thao tác xóa: chặn với thông báo "Sản phẩm đã phát sinh lịch sử tồn kho"; hướng dẫn chuyển sang ngưng kinh doanh. |
| **BR-17** | **[Constraint] Khóa đơn vị tính** | Không đổi đơn vị tính của sản phẩm/biến thể đã phát sinh lịch sử — đổi đơn vị làm mất nghĩa của các bản ghi cũ. Cách xử lý: tạo sản phẩm/biến thể mới. | Trường đơn vị tính vô hiệu kèm giải thích khi sản phẩm đã có lịch sử. |
| **BR-18** | **[Constraint] Hàng hoàn hỏng không nhập lại kho** | Hoàn tiền cho khách nhưng hàng không còn bán được: không cộng tồn; bắt buộc ghi chú trên phiếu trả hàng. Biến động tiền ghi ở phiếu trả, không có bản ghi tồn. | Xác nhận trả hàng mà không chọn "nhập lại kho": phiếu vẫn hoàn tiền, không sinh bản ghi tồn — ghi chú bắt buộc. |

---

## 3. Sơ đồ tương tác (Interaction Diagram)

*(Sơ đồ đầy đủ tại `docs/stock-history/srs/flows.md`)*

![[flows.md]]

---

## 4. Đặc tả màn hình (Screen Specifications)

*(Đặc tả chi tiết tại `docs/stock-history/srs/screens.md`)*

![[screens.md]]

---

## 5. Open Questions

- [x] **OQ-01:** POS có chế độ **bán hàng offline** (mất mạng vẫn bán, đồng bộ sau) không? — **Resolved:** Không, POS chỉ bán khi có mạng; mọi bản ghi dùng một mốc thời gian thống nhất theo máy chủ (BR-14 áp dụng trực tiếp).
- [x] **OQ-02:** Lịch sử tồn kho **lưu trữ bao lâu** và có cần **xuất báo cáo tổng hợp** không? — **Resolved:** Lưu trữ không giới hạn thời gian; giai đoạn 1 chỉ cần xuất Excel danh sách theo bộ lọc (đã có trong screens). Báo cáo tổng hợp theo kỳ để làm sau nếu kế toán yêu cầu.
- [x] **OQ-03:** **Phạm vi xem** của từng vai trò? — **Resolved:** Truy cập điều khiển bằng quyền riêng **"Xem lịch sử tồn kho"** cấp theo vai trò: người được cấp quyền xem lịch sử các kho mình được gán. Mặc định cấp cho quản lý cửa hàng và quản lý chuỗi; có thể cấp thêm cho nhân viên bán hàng.
- [x] **OQ-04:** Hệ thống có **combo/gói sản phẩm** không, ghi tồn thế nào? — **Resolved:** Có. Bán 1 combo sinh nhiều bản ghi — mỗi linh kiện 1 bản ghi tham chiếu cùng dòng combo trên đơn; combo không có tồn riêng (nhất quán nhánh B split-order).
- [x] **OQ-05:** **Giá vốn xuất kho** có nằm trong tính năng này không? — **Resolved:** Tách tính năng riêng. Sổ tồn chỉ ghi số lượng; khi triển khai giá vốn, chuỗi bản ghi này là nguồn dữ liệu đầu vào — không phải bổ sung trường gì hôm nay.
- [x] **OQ-06:** Có cần **điều chỉnh tồn hàng loạt** (import Excel) không? — **Resolved:** Thuộc giai đoạn 2. Khi làm: mỗi dòng import sinh 1 bản ghi Điều chỉnh tồn kho riêng theo BR-03; lý do áp dụng cho cả loạt theo BR-12.

---

## 6. References

- `@../../rules/approval-gate.md`
- `@../../rules/diagram-selection.md`
- `@../screen/SKILL.md` — Skill Con phụ trách đặc tả màn hình
- `[[docs/split-order/srs/spec|Split Order SRS]]` — BR-16 xác định tồn bị trừ khi tạo đơn, tách đơn không đổi tồn
- `[[docs/merge-order/srs/spec|Merge Order SRS]]` — BR-20 xác định gộp đơn không đổi tồn
- Thông tư 200/2014/TT-BTC (Bộ Tài chính) — chuẩn mực kế toán Việt Nam về hàng tồn kho: yêu cầu theo dõi tồn theo từng kho và đối chiếu được với chứng từ gốc; sổ cái tồn kho của phần mềm nên hỗ trợ nguyên tắc này.
