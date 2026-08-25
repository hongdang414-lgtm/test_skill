---
feature: stock-history
type: srs-screen
updated: 2026-08-25
status: draft
authors: [BA Team]
---

# Đặc tả màn hình: Stock History (Lịch sử thay đổi tồn kho)

## Screen: Danh sách lịch sử tồn kho

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Bộ lọc Kho/Cửa hàng | Dropdown (chọn nhiều) | Không | Tất cả kho người dùng có quyền | - Danh sách kho lọc theo quyền xem của người dùng.<br/>- Giữ nguyên lựa chọn khi quay lại màn hình trong phiên. |
| Tìm sản phẩm | Textbox + gợi ý | Không | Trống | - Gõ tên sản phẩm, tên biến thể hoặc SKU; gợi ý hiện từ 2 ký tự, hiển thị kèm biến thể.<br/>- Hỗ trợ quét mã vạch: tự điền SKU tương ứng.<br/>*Lỗi:* "Không tìm thấy sản phẩm phù hợp" — danh sách trống kèm gợi ý kiểm tra lại mã. |
| Khoảng thời gian | Date range + chọn nhanh | Không | 30 ngày gần nhất | - Chọn nhanh: Hôm nay / 7 ngày / 30 ngày / Tháng này / Tùy chọn.<br/>- Tùy chọn: ngày bắt đầu phải trước hoặc bằng ngày kết thúc.<br/>*Lỗi:* "Ngày bắt đầu phải trước hoặc bằng ngày kết thúc". |
| Loại biến động | Dropdown (chọn nhiều) | Không | Tất cả | - 12 loại theo Mục 1.1 trong spec (Tồn đầu kỳ, Nhập hàng, Hủy phiếu nhập, Bán hàng, Sửa đơn đang mở, Hủy đơn trước thanh toán, Khách trả hàng, Trả hàng NCC, Chuyển kho — xuất, Chuyển kho — nhập, Kiểm kho, Điều chỉnh tồn kho). |
| Người thực hiện | Dropdown | Không | Tất cả | - Danh sách nhân viên tải theo kho đang lọc; gồm cả hệ thống nếu có thao tác tự động. |
| Tìm chứng từ | Textbox | Không | Trống | - Nhập mã chứng từ (đơn hàng, phiếu nhập, phiếu trả, phiếu chuyển, phiếu kiểm) — tìm gần đúng, không phân biệt hoa/thường. |
| Danh sách kết quả | Bảng + phân trang | — | Sắp mới nhất trước | - Cột: Thời gian / Sản phẩm (biến thể) / SKU / Kho / Loại biến động / Tồn trước / Thay đổi / Tồn sau / Chứng từ / Người thực hiện.<br/>- Cột Thay đổi hiển thị dấu + (xanh) hoặc − (đỏ), canh phải, chữ số định dạng cố định.<br/>- Loại biến động hiển thị nhãn màu; nhóm "Hủy phiếu nhập / Hủy đơn / Trả hàng NCC" dễ phân biệt nhóm tăng.<br/>- Hiển thị 50 bản ghi/trang kèm tổng số bản ghi; bộ lọc áp dụng ngay khi thay đổi. |
| Dòng tổng kết nhanh | Thanh tổng kết | — | Theo bộ lọc | - Hiển thị tổng cộng (+), tổng trừ (−) và số bản ghi trong khoảng đang lọc — hỗ trợ đối chiếu nhanh. |
| Nút Xuất Excel | Button | — | — | - Xuất đúng bộ lọc hiện tại, giữ thứ tự sắp xếp.<br/>- Chặn click liên tục đến khi nhận phản hồi.<br/>*Lỗi:* "Xuất thất bại, vui lòng thử lại". |
| Liên kết chứng từ (trong cột Chứng từ) | Link | — | — | - Chứng từ còn tồn tại và người dùng có quyền: mở chi tiết chứng từ ở tab mới.<br/>- Chứng từ đã hủy: mở được, hiển thị trạng thái "Đã hủy" (không xóa cứng).<br/>- Không có quyền: link vô hiệu kèm tooltip "Không có quyền xem chứng từ".<br/>- Nguồn "Sửa sản phẩm": mở màn chi tiết sản phẩm. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Menu Kho hàng | Nhấn "Lịch sử tồn kho" | Danh sách lịch sử tồn kho | Vào từ phân mục tồn kho |
| Danh sách lịch sử | Nhấn 1 dòng | Drawer chi tiết bản ghi | Hiện đủ thông tin + lý do |
| Danh sách lịch sử | Nhấn mã chứng từ | Chi tiết chứng từ (tab mới) | Tôn trọng quyền và trạng thái chứng từ |
| Danh sách lịch sử | Nhấn tên sản phẩm | Chi tiết sản phẩm — tab Tồn kho | Lọc sẵn theo biến thể + kho |
| Danh sách lịch sử | Nhấn Xuất Excel | Tải file | Đúng bộ lọc hiện tại |
| Chi tiết sản phẩm — tab Tồn kho | Nhấn "Xem lịch sử tồn" | Danh sách lịch sử (lọc sẵn) | Lọc theo sản phẩm + kho đang xem |
| Drawer chi tiết bản ghi | Nhấn bản ghi đảo ngược liên quan | Cuộn tới bản ghi đó trong danh sách | Điều hướng theo cặp bản ghi gốc/đảo ngược |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Trạng thái tải:** Hiện skeleton cho bảng khi đổi bộ lọc; không nhảy layout.
- **Không phụ thuộc màu:** Cột Thay đổi luôn kèm dấu +/− rõ ràng, màu chỉ là hỗ trợ.
- **Thời gian:** Hiển thị giờ địa phương của kho, một mốc thống nhất từ máy chủ (OQ-01 đã chốt: không bán offline).
- **Sản phẩm ngưng kinh doanh:** Vẫn hiển thị trong lịch sử kèm nhãn "Ngưng kinh doanh" (BR-16).
- **Bản ghi từ combo:** hiển thị theo từng linh kiện; drawer chi tiết chỉ rõ dòng combo tương ứng trên đơn (OQ-04).
- **Bộ lọc giữ nguyên** khi quay lại từ màn khác trong cùng phiên làm việc.

---

## Screen: Chi tiết bản ghi lịch sử (Drawer)

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Toàn bộ thông tin bản ghi | Chỉ đọc | — | Theo bản ghi | - Hiện đủ 10 thông tin bắt buộc (BR-04): Thời gian, Sản phẩm/biến thể, SKU, Kho, Loại biến động, Tồn trước, Thay đổi, Tồn sau, Chứng từ/nguồn, Người thực hiện.<br/>- Không có nút sửa/xóa ở bất kỳ vai trò nào (BR-01). |
| Lý do / Ghi chú | Chỉ đọc | — | Theo bản ghi | - Hiển thị với loại Điều chỉnh tồn kho (lý do bắt buộc — BR-12), Kiểm kho và ghi chú tự do của nghiệp vụ nguồn. |
| Bản ghi đảo ngược liên quan | Link | Không | Trống | - Nếu bản ghi này bị đảo ngược (hoặc là bản đảo ngược của bản khác): hiện mã và thời gian bản kia, nhấn để nhảy tới.<br/>- Cặp bản ghi tham chiếu cùng một chứng từ. |
| Trạng thái chứng từ nguồn | Nhãn trạng thái | — | Theo chứng từ | - Hiện trạng thái hiện tại của chứng từ (Hợp lệ / Đã hủy) giúp người đọc hiểu vì sao tồn cộng lại. |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Danh sách lịch sử | Nhấn 1 dòng | Drawer chi tiết bản ghi | Không rời trang danh sách |
| Drawer chi tiết | Nhấn liên kết chứng từ | Chi tiết chứng từ (tab mới) | Tôn trọng quyền và trạng thái |
| Drawer chi tiết | Nhấn bản ghi đảo ngược | Danh sách — cuộn tới bản kia | Nhấn giữ ngữ cảnh bộ lọc |
| Drawer chi tiết | Nhấn Đóng | Danh sách lịch sử | |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Chỉ-đọc tuyệt đối:** Không render bất kỳ control chỉnh sửa nào — thể hiện rõ nguyên tắc audit trail.
- **Ngữ cảnh chuỗi:** Hiện "Tồn trước" của bản liền trước trong cùng chuỗi khi người dùng cần đối chiếu tay.

---

## Screen: Điều chỉnh tồn kho trên Sửa sản phẩm

### 1. Đặc tả trường thông tin (Field Specifications)

| Tên trường thông tin | Control Type | Bắt buộc | Mặc định | Rules & Lỗi |
| :--- | :--- | :--- | :--- | :--- |
| Số lượng tồn (từng kho) | Number input — một ô cho mỗi kho | Có | Tồn hiện tại của kho | - Hiển thị tồn hiện tại từng kho tại lúc tải màn hình; điều chỉnh ghi thay đổi = mới − cũ (BR-11).<br/>- Chỉ nhận số nguyên; không nhận ký tự khác chữ số.<br/>- Giữ nguyên giá trị: không sinh bản ghi.<br/>*Lỗi:* "Số lượng không hợp lệ". |
| Lý do điều chỉnh | Dropdown + Textbox khi chọn "Khác" | Có — khi số lượng thay đổi | Trống | - Danh mục: Nhập sai ban đầu / Hỏng, hết hạn / Thất lạc, thất thoát / Kiểm đếm lại / Khác (BR-12).<br/>- Chọn "Khác": bắt buộc nhập tối thiểu 3 ký tự.<br/>*Lỗi:* "Vui lòng chọn lý do điều chỉnh tồn kho".<br/>*Lỗi:* "Nhập ghi chú tối thiểu 3 ký tự". |
| Cảnh báo xung đột tồn | Dialog xác nhận (tự hiện) | — | — | - Kích hoạt khi tồn hiện tại khác giá trị lúc mở màn hình (BR-15).<br/>- Nội dung: "Tồn đã thay đổi từ X thành Y (bởi người/thiết bị, lúc giờ). Áp dụng số mới Z?"<br/>- Chọn Áp dụng: ghi điều chỉnh từ Y sang Z. Chọn Hủy: không ghi, màn làm mới tồn Y. |
| Cảnh báo tồn âm | Dialog cảnh báo (tự hiện) | — | — | - Kích hoạt khi kết quả < 0 ở kho chưa bật "Cho phép tồn âm" (BR-13).<br/>- Chặn lưu ở kho chưa bật; kho đã bật: hiện xác nhận tiếp tục và tồn hiển thị màu cảnh báo.<br/>*Lỗi:* "Tồn sau điều chỉnh bị âm (−N). Vui lòng kiểm tra số hoặc liên hệ quản lý". |
| Nút Lưu | Button | Có | — | - Trước khi ghi: hiện tóm tắt "10 → 15 (+5) — Lý do: ..." để xác nhận.<br/>- Chặn click liên tục đến khi có phản hồi.<br/>- Thành công: toast "Đã ghi điều chỉnh tồn kho +5". |

### 2. Điều hướng (Navigation Map)

| Từ màn hình | Hành động | Đến màn hình | Ghi chú |
| :--- | :--- | :--- | :--- |
| Sửa sản phẩm | Sửa số tồn + Lưu | Dialog tóm tắt + lý do | Không rời màn hình |
| Dialog tóm tắt | Xác nhận | (có xung đột) Dialog cảnh báo tồn đã đổi | BR-15 |
| Dialog tóm tắt | Xác nhận (không xung đột) | Sửa sản phẩm + toast thành công | Bản ghi đã ghi vào sổ |
| Dialog cảnh báo tồn đã đổi | Áp dụng số mới | Sửa sản phẩm + toast | Ghi từ tồn thật hiện tại |
| Dialog cảnh báo tồn đã đổi | Hủy | Sửa sản phẩm (làm mới tồn) | Không ghi gì |
| Dialog tóm tắt | Hủy | Sửa sản phẩm | Không ghi gì |

### 3. Các điểm lưu ý về trải nghiệm & Logic nâng cao (UI/UX & Edge Logic)

- **Hiển thị tồn "lúc mở màn hình" và "hiện tại"** cạnh nhau khi khác nhau — giúp quản lý hiểu cảnh báo xung đột (BR-14, BR-15).
- **Nhiều kho:** mỗi kho một ô riêng, mỗi ô thay đổi sinh một bản ghi độc lập (BR-03); tóm tắt liệt kê từng kho.
- **Đơn vị tính** hiển thị cạnh ô số lượng; khóa đổi đơn vị tính khi đã có lịch sử (BR-17).
- **Phân quyền:** chỉ quản lý cửa hàng trở lên được sửa số tồn trực tiếp; nhân viên chỉ xem.
