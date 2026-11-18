# Fantastic Cafe — Hệ thống gọi món qua mã QR cho quán cà phê

Bài tập lớn học phần **Lập trình Web** — Học viện Công nghệ Bưu chính Viễn thông, Khoa An toàn thông tin.

Khách ngồi tại bàn quét mã QR để xem thực đơn, nhờ trợ lý AI gợi ý món, gọi món và theo dõi đơn ngay trên điện thoại, không cần cài ứng dụng hay tạo tài khoản. Nhân viên thu tiền trước khi pha chế, quầy pha chế nhận đơn theo thời gian thực trên màn hình KDS. Quản trị viên quản lý thực đơn, bàn, nhân sự, kho, khuyến mãi, doanh thu và tình trạng vận hành của hệ thống.

## Mục lục

1. [Điểm nổi bật](#1-điểm-nổi-bật)
2. [Tác nhân và ba cổng truy cập](#2-tác-nhân-và-ba-cổng-truy-cập)
3. [Chức năng chi tiết](#3-chức-năng-chi-tiết)
4. [Luồng nghiệp vụ chính](#4-luồng-nghiệp-vụ-chính)
5. [Kiến trúc hệ thống](#5-kiến-trúc-hệ-thống)
6. [Mô hình dữ liệu](#6-mô-hình-dữ-liệu)
7. [Thiết kế API](#7-thiết-kế-api)
8. [Bảo mật](#8-bảo-mật)
9. [Bảo đảm tính đúng đắn của dữ liệu](#9-bảo-đảm-tính-đúng-đắn-của-dữ-liệu)
10. [Công nghệ sử dụng](#10-công-nghệ-sử-dụng)
11. [Cấu trúc thư mục](#11-cấu-trúc-thư-mục)
12. [Cài đặt và chạy ở môi trường phát triển](#12-cài-đặt-và-chạy-ở-môi-trường-phát-triển)
13. [Chạy bản production cục bộ bằng Docker](#13-chạy-bản-production-cục-bộ-bằng-docker)
14. [Cấu hình](#14-cấu-hình)
15. [Dữ liệu demo và tài khoản](#15-dữ-liệu-demo-và-tài-khoản)
16. [Kiểm thử](#16-kiểm-thử)
17. [Các lệnh thường dùng](#17-các-lệnh-thường-dùng)
18. [Hạn chế đã biết](#18-hạn-chế-đã-biết)
19. [Chức năng đã gỡ](#19-chức-năng-đã-gỡ)
20. [Nguồn ảnh](#20-nguồn-ảnh)

---

## 1. Điểm nổi bật

- **Gọi món không cần ứng dụng:** quét QR là vào đúng bàn; nhiều điện thoại cùng bàn dùng chung một phiên phục vụ nhưng mỗi thiết bị chỉ thấy đơn của mình.
- **Thu tiền trước, pha chế sau:** đơn chỉ vào màn hình pha chế khi nhân viên xác nhận đã nhận đủ tiền; có khóa chống hai nhân viên cùng thu một bàn.
- **Thời gian thực:** KDS, danh sách yêu cầu và trang theo dõi đơn của khách cập nhật tức thời qua Socket.IO, sự kiện đi qua transactional outbox nên không mất khi sự cố giữa chừng.
- **Tiền và kho luôn đúng:** transaction MongoDB, khóa lạc quan, idempotency, bản chụp giá và hóa đơn bất biến; các tình huống tranh chấp đều có test chứng minh.
- **Kho nguyên liệu theo công thức:** giữ nguyên liệu khi đặt, trừ khi pha chế, trả lại khi hủy, tự ẩn món hết nguyên liệu ở mọi nơi khách nhìn thấy.
- **Voucher và tích điểm theo số điện thoại:** mọi con số giảm giá do server tính, không thể sửa phía trình duyệt.
- **AI Barista an toàn:** mô hình ngôn ngữ chỉ trích xuất ý định, backend tự chọn món từ thực đơn thật; có chế độ dự phòng theo luật khi không dùng AI.
- **Phân quyền và cô lập chặt:** ba cổng tách biệt, hai cụm API (khách / nội bộ), QR ký HMAC có thể thu hồi, chống CSRF, giới hạn tần suất bằng Redis.
- **Có thể vận hành thật:** Docker Compose nhiều tiến trình, worker nền có heartbeat, trang giám sát vận hành, phát hiện bất thường, sao lưu và khôi phục.

## 2. Tác nhân và ba cổng truy cập

| Tác nhân | Cổng (Docker) | Cổng (phát triển) | Xác thực | Vai trò chính |
|---|---|---|---|---|
| **Khách hàng** | 8080 | 5173 | Cookie phiên khách sau khi quét QR, không cần tài khoản | Xem thực đơn, đặt món, theo dõi đơn, gọi nhân viên, đánh giá, đặt bàn, tặng quà, chọn nhạc |
| **Thành viên** (khách đăng ký) | 8080 | 5173 | SĐT + mật khẩu, cookie phiên riêng | Dùng voucher, tích và đổi điểm, đặt bàn trước |
| **Nhân viên** (`STAFF`) | 8081 | 5173 | Email + mật khẩu, access token + refresh token | Bàn và phiên, thu tiền, KDS, yêu cầu của khách, ca thu ngân, chấm công |
| **Quản trị viên** (`ADMIN`) | 8082 | 5173 | Email + mật khẩu, access token + refresh token | Toàn bộ quản trị, báo cáo, vận hành |
| **Chủ quán** | 8082 | 5173 | Là tài khoản `ADMIN` đang hoạt động được tạo sớm nhất | Như quản trị viên, và là người duy nhất quản lý các tài khoản quản trị khác |

Khi chạy bằng Docker, mỗi cổng chỉ chuyển tiếp đúng nhóm API của vai trò đó; gọi API sai cổng nhận lỗi 404. Ở chế độ phát triển cả ba vai trò dùng chung cổng 5173.

## 3. Chức năng chi tiết

### 3.1. Khách hàng

**Vào bàn và phiên phục vụ**
- Quét mã QR cố định dán tại bàn (`/t/<token>`). Nếu bàn chưa có phiên, server tự mở phiên mới; nếu đã có, thiết bị tham gia phiên hiện tại với một `participantId` riêng. Hai thiết bị quét cùng lúc trên bàn trống chỉ tạo một phiên.
- Phiên có ba trạng thái: `OPEN` (đang phục vụ), `CHECKOUT` (đang thanh toán, không gọi thêm), `CLOSED` (đã đóng, hóa đơn đã chốt). Khi phiên đóng, quyền của mọi thiết bị khách bị thu hồi; lần quét sau mở phiên mới.
- Phiên tự mở nhưng không có đơn nào sau 60 phút sẽ tự đóng.

**Thực đơn và tìm kiếm**
- Phần đầu trang là 10 món bán chạy nhất (tính theo số lượng bán của các đơn đã trả tiền, 30 ngày gần nhất, giờ Việt Nam), dùng chung cách tính với mục Top sản phẩm trên Dashboard.
- Thực đơn theo danh mục, món có nhiều size với giá riêng, mức đường, mức đá, topping, ảnh, mô tả.
- Tìm kiếm tiếng Việt có hoặc không dấu, chấp nhận gõ sai nhẹ, hiểu nhóm món và ngân sách ("dưới 50k"); bộ lọc ngân sách, không caffeine, không sữa. Pipeline này chạy theo luật, không gọi mô hình AI.

**AI Barista**
- Khách nhập câu tự nhiên ("mình thích vị chua nhẹ, không cà phê, dưới 50 nghìn"), hệ thống trả tối đa 3 món đang bán kèm ảnh, giá và lý do.
- Nguyên tắc **AI chỉ đề xuất, backend quyết định**: mô hình chỉ nhận câu của khách và một danh sách tag giới hạn, trích ra ý định (khẩu vị, ngân sách, điều kiện); backend lọc cứng từ MongoDB (đang bán, đủ nguyên liệu, ngân sách, caffeine/sữa) rồi tự xếp hạng và viết lý do. Mô hình không bao giờ nhận cả thực đơn, và AI không tự đặt đơn.
- Khi không bật AI hoặc mô hình lỗi / quá thời gian, hệ thống chuyển sang chấm điểm theo luật và ghi rõ chế độ này trên giao diện (`mode: llm | fallback`, `route: rules | llm | llm_fallback`).
- Quản trị viên có nút "AI gợi ý tag" khi sửa món; gợi ý chỉ được lưu sau khi quản trị viên xem và bấm Lưu.

**Giỏ hàng và đặt món**
- Giỏ hàng lưu theo từng thiết bị (Zustand, lưu trong trình duyệt), giữ nguyên khi mất mạng nhưng chỉ gửi đơn khi online.
- Báo giá `POST /orders/quote`: server tính lại giá toàn bộ giỏ từ dữ liệu thật và trả báo giá có chữ ký HMAC gắn với thiết bị, phiên bàn, nội dung giỏ, ưu đãi và tổng tiền, hiệu lực 2 phút. Giá phía client chỉ để tham khảo.
- Đặt đơn `POST /orders` kèm báo giá và header `Idempotency-Key`; gửi lại cùng khóa trả đúng đơn cũ, không tạo đơn thứ hai. Giá hoặc tình trạng món vừa đổi trả `QUOTE_CHANGED` để khách xác nhận lại.
- Mỗi món tối đa 50 phần; mỗi thiết bị tối đa 3 đơn đang chờ thanh toán cùng lúc.

**Theo dõi đơn, gọi nhân viên, đánh giá**
- Trang "Đơn của tôi" cập nhật realtime. Khách tự hủy đơn chưa trả tiền; đơn đã trả tiền nhưng chưa pha thì gửi yêu cầu hủy kèm lý do cho nhân viên.
- Nút Gọi nhân viên và yêu cầu hỗ trợ thanh toán (tối đa 6 yêu cầu mỗi phút mỗi bàn).
- Đánh giá 1–5 sao kèm nhận xét cho từng đơn đã thanh toán và phục vụ; mỗi đơn một lần, chỉ thiết bị đã đặt mới đánh giá được.
- Sau khi phiên đóng, khách xem lại hóa đơn của thiết bị mình trong 12 giờ.

**Thành viên, voucher và điểm** (xem mục 3.4)

**Đặt bàn trước** (xem mục 3.5), **Tặng quà cho bàn khác**, **Jukebox**, **Ảnh in tem dán cốc và latte art** (xem mục 3.7)

### 3.2. Nhân viên

- **Bàn & phiên:** xem bàn trống, đang phục vụ, chờ thanh toán; mở phiên thủ công, chuyển bàn, đóng phiên (chỉ khi mọi đơn đã phục vụ hoặc hủy và không còn nợ), xem và in hóa đơn.
- **Thu tiền:** bấm "Kiểm tra & nhận tiền" để nhận quyền thu của bàn trong 2 phút (tự gia hạn mỗi 30 giây khi hộp thoại mở, tự trả khi đóng); nhân viên khác thấy thông báo bàn đang được thu. Thu từng phần hoặc đủ, theo phương thức tiền mặt, chuyển khoản đã kiểm tra hoặc khác. Thu đủ thì đơn chuyển từ `AWAITING_PAYMENT` sang `PENDING` và lên KDS.
- **KDS:** bốn cột Chờ xác nhận, Đã nhận, Đang pha, Sẵn sàng; mỗi thẻ có bàn, giờ đặt, số phút chờ, món, tùy chọn, ghi chú, topping. Đơn nằm quá lâu ở một bước bị gắn nhãn "Quá hạn" (quá 5 phút ở bước chờ / đã nhận / sẵn sàng, quá 10 phút ở bước đang pha). Yêu cầu hủy đơn đã trả tiền hiện ưu tiên ở đầu KDS. Lịch sử đơn đã hủy thu gọn bên dưới.
- **Hủy và hoàn tiền:** khi duyệt yêu cầu hủy đơn đã thu tiền, nhân viên xác nhận đã hoàn tiền thực tế; trong cùng transaction đơn chuyển `CANCELLED`, khoản thu chuyển `REFUNDED`, nguyên liệu, lượt voucher và điểm đã đổi được trả lại. Phần mềm chỉ ghi nhận, việc trả tiền do nhân viên thực hiện.
- **Điều chỉnh khoản thu:** ba loại — hủy ghi nhận, sửa phương thức, chuyển sang đúng đơn. Khoản gốc không bị xóa mà được đảo bằng bản ghi mới có liên kết; yêu cầu của nhân viên cần quản trị viên duyệt, quản trị viên tự điều chỉnh thì áp dụng ngay.
- **Yêu cầu của khách:** danh sách yêu cầu gọi nhân viên theo thời gian thực, bấm Xử lý để ghi người xử lý.
- **Hết món:** bật/tắt món, size, topping; trang ghi rõ món bị ẩn do thiếu nguyên liệu.
- **Ca thu ngân:** mở ca với tiền đầu ca, chốt ca với tiền đếm thực tế. Tiền dự kiến = tiền đầu ca + tiền mặt thu − tiền mặt hoàn; có chênh lệch thì bắt buộc nhập lý do. Toàn quán chỉ có một ca mở; nếu thu tiền khi chưa mở ca, hệ thống tự mở ca với tiền đầu ca 0đ.
- **Ảnh cốc & latte art**, **Jukebox** phát tại quầy, **Đặt bàn**, **Lịch ca & chấm công** (xem các mục sau).

### 3.3. Quản trị viên

- **Dashboard:** thực thu ròng = tiền đã thu − tiền đã hoàn ± điều chỉnh, so sánh với kỳ trước, số đơn và tỉ lệ hủy; doanh thu theo ngày và theo giờ, giờ cao điểm, tiền món theo danh mục, thực thu theo phương thức, Top 5 sản phẩm. Lọc theo khoảng ngày (giờ Việt Nam), xuất CSV (UTF-8 BOM, mở đúng bằng Excel) chạy ở worker nền, in / PDF.
- **Thực đơn:** danh mục, món (nhiều size, tag hương vị, caffeine / sữa, ảnh), topping. Món đã bán chỉ được lưu trữ và khôi phục, không xóa hẳn để đơn và hóa đơn cũ không mất liên kết; xóa danh mục trống hoặc chuyển toàn bộ món sang danh mục khác rồi xóa.
- **Bàn & QR:** xem và tải QR tĩnh có chữ ký HMAC; "Cấp lại khóa" làm mã đã in mất hiệu lực ngay (chỉ quản trị viên).
- **Nhân viên:** tạo, sửa, khóa, đặt lại mật khẩu, xóa mềm. Chỉ chủ quán được tạo hoặc quản lý tài khoản quản trị; quản trị viên không tự khóa hay tự hạ quyền chính mình.
- **Hóa đơn:** lịch sử và in lại hóa đơn theo bàn, thu ngân, khoảng ngày.
- **Khoản thu:** sổ khoản thu, duyệt điều chỉnh, lịch sử đảo và thay thế.
- **Đánh giá:** điểm trung bình, phân bố số sao, nhận xét.
- **Kho & công thức, Voucher, Thành viên, Đặt bàn, Lịch ca & hiệu suất, Jukebox, Ảnh cốc** (xem các mục sau).
- **Vận hành:** trạng thái MongoDB / Redis / worker, số bàn đang phục vụ, đơn đang xử lý, đơn chờ lâu nhất, danh sách sự kiện outbox và công việc nền lỗi để phát lại (có ghi audit), cảnh báo bất thường.

### 3.4. Kho nguyên liệu, voucher và thành viên

**Kho nguyên liệu theo công thức (BOM)** — `/admin/inventory`
- Khai báo nguyên liệu (g, ml hoặc cái), ngưỡng cảnh báo sắp hết, giá vốn tham khảo. Số lượng lưu dạng số nguyên theo 1/1000 đơn vị để tránh sai số số thực.
- Công thức cho từng món và topping trên một phần; size khác định lượng có công thức riêng thay thế công thức nền.
- Nhập kho, xuất khác / hao hụt, kiểm kê tồn thực tế — mỗi thao tác bắt buộc có lý do, gửi kèm `Idempotency-Key` và version, ghi vào sổ kho bất biến.
- **Giữ – trừ – trả:** đặt đơn giữ nguyên liệu (`RESERVE`), chuyển sang Đang pha thì trừ tồn thật (`CONSUME`), hủy trước pha thì trả phần giữ (`RELEASE`); mọi thay đổi nằm trong cùng transaction với đơn. Lượng có thể dùng = tồn thực − đang giữ.
- **Tự động báo hết món:** trạng thái còn bán được tính lại mỗi lần đọc thực đơn từ tồn kho mới nhất; size, món hoặc topping không đủ nguyên liệu bị ẩn ở thực đơn, tìm kiếm, AI Barista và bị từ chối khi đặt; nhập thêm hàng thì tự bán lại. Món nhân viên chủ động tạm ngừng thì không tự bật lại.
- Sổ kho phân trang, lọc theo nguyên liệu, ghi người thao tác và đơn liên quan. Nguyên liệu chưa từng dùng thì xóa được; đã dùng chỉ được "Ngừng dùng".

**Voucher** — `/admin/vouchers`
- Giảm số tiền cố định hoặc theo phần trăm; đơn tối thiểu, mức giảm tối đa, thời gian hiệu lực, tổng số lượt, giới hạn lượt mỗi thành viên, chỉ dành cho thành viên, cho phép dùng kèm điểm hay không.
- Đặt đơn giữ lượt voucher; chuyển sang Đang pha thì lượt thành "đã dùng"; hủy trước đó thì trả lượt đúng một lần. Voucher chưa dùng thì xóa được, đã dùng chỉ tạm dừng.

**Thành viên và điểm** — `/member`, `/admin/members`
- Đăng ký bằng tên, số điện thoại di động Việt Nam (dạng `0…` hoặc `+84…` được chuẩn hóa về một tài khoản) và mật khẩu; thông báo lỗi nêu lý do từng ô. Số điện thoại là tên đăng nhập, hệ thống không dùng SMS.
- Mật khẩu bcrypt, nhập sai 6 lần khóa đăng nhập 15 phút, đổi mật khẩu hoặc khóa tài khoản thu hồi mọi phiên cũ. Vào phiên bàn mới hoặc rời bàn thì đăng xuất thành viên.
- **Quy tắc điểm:** 1 điểm = 1.000đ; đổi tối đa 50% số tiền còn lại sau voucher và không vượt số dư; tích ⌊tổng sau ưu đãi / 10.000⌋ điểm, chỉ cộng khi đơn vừa **đã thanh toán** vừa **đã phục vụ**, đúng một lần. Trừ voucher trước rồi mới trừ điểm; chỉ dùng chung voucher và điểm khi voucher cho phép.
- Sổ điểm (`EARN` / `REDEEM` / `RETURN`) bất biến, quản trị viên xem lịch sử nhưng không có chức năng sửa tay số dư. Hóa đơn lưu bản chụp ưu đãi nên sửa hay tạm dừng voucher sau này không đổi hóa đơn cũ.

### 3.5. Đặt bàn trước theo khung giờ

Khách (đăng nhập thành viên) mở `/reservations`, chọn ngày, giờ, số người và chỉ thấy bàn còn trống đủ chỗ; nhân viên và quản trị viên quản lý tại `/staff/reservations`, `/admin/reservations`.

- Mỗi lượt giữ chỗ **60 phút**; giờ đến từ 15 phút đến 7 ngày tới; tối đa 30 khách, một bàn đủ chỗ, không ghép bàn; hai khoảng thời gian trên cùng một bàn không được chồng nhau (đặt tiếp đúng giờ lượt trước kết thúc vẫn được). Thành viên tối đa 3 lịch đang xử lý và không đặt hai bàn trùng giờ.
- **Chống đặt trùng:** khóa bàn trong transaction khi đặt; mỗi yêu cầu có `requestId` UUID và hash nội dung nên gửi lại không tạo lịch thứ hai.
- **Check-in:** nhân viên bấm "Khách đã đến" trong khoảng từ 15 phút trước giờ hẹn đến hết hạn đến; check-in mở đúng một phiên bàn, khách quét QR tại bàn để gọi món như thường.
- **Tự nhả bàn và no-show:** quá giờ hẹn 15 phút lịch hết hạn, bị đánh dấu `NO_SHOW` và nhả bàn; scheduler quét mỗi 30 giây, và việc tính bàn trống luôn bỏ qua lịch quá hạn kể cả khi tác vụ nền chưa chạy.
- **Trạng thái:** `CONFIRMED` → `CHECKED_IN` → `COMPLETED`; hoặc `CONFIRMED` → `CANCELLED` / `NO_SHOW`. Khách tự hủy lịch chưa check-in kèm lý do.
- **Liên kết với luồng chung:** bàn có lịch giữ chỗ trong 60 phút tới không bị QR tự mở, mở thủ công hay chuyển phiên vào; bàn đang phục vụ bị loại khỏi danh sách giờ sắp tới; chuyển phiên đã check-in sang bàn khác thì chuyển cả liên kết lịch; ngừng bàn hoặc giảm sức chứa khi còn lịch bị chặn; phiên đóng thì lịch hoàn tất.
- Nhân viên có huy hiệu số lịch đang hoạt động và thông báo khi có lịch mới (cập nhật realtime, kèm polling dự phòng).

### 3.6. Lịch ca, chấm công và hiệu suất nhân viên

Quản trị viên dùng `/admin/workforce`, nhân viên dùng `/staff/workforce`.

- **Xếp lịch ca theo tuần:** tạo, sửa, hủy ca (hủy cần lý do); ca tối đa 16 giờ, cho phép qua đêm; một nhân viên không có hai ca chồng nhau. **Sao chép lịch tuần** có bước xem trước và xác nhận, tạo cả tuần trong một transaction hoặc không tạo gì.
- **Chấm công:** nhân viên bấm Vào ca / Ra ca, dùng giờ của server (không nhận giờ từ trình duyệt); vào ca từ 30 phút trước giờ bắt đầu; mỗi người chỉ một ca đang làm. Phút trễ tính sau dung sai 5 phút; ca quá giờ chưa chấm hiển thị "Chưa vào ca", quá giờ kết thúc là "Vắng". Quản trị viên điều chỉnh giờ thực tế khi ghi sai (bắt buộc lý do, lưu giá trị trước và sau trong nhật ký, không sửa giờ bấm gốc).
- **KPI từng nhân viên:** số đơn ở từng công đoạn KDS (nhận / pha / sẵn sàng / giao) gắn với người bấm; thời gian pha chế trung bình; sao trung bình của các đơn đã giao; giờ theo lịch so với giờ thực tế; số ca vắng, trễ, chưa ra ca; thao tác ngoài ca. Mỗi công đoạn được ghi cùng transaction với việc chuyển trạng thái đơn, không có số liệu suy đoán cho đơn cũ.
- **Cảnh báo thiếu người:** quản trị viên đặt các khung giờ cần tối thiểu bao nhiêu người theo thứ trong tuần; hệ thống báo "Đã xếp x/y người · Thiếu n" (tính theo số người đồng thời thấp nhất trong toàn khung) và "Đang chấm công x/y" ở khung đang diễn ra, tô đỏ khi thiếu.
- **Gợi ý giờ cao điểm:** từ lịch sử 4 tuần hoàn chỉnh (ít nhất 20 đơn đã thanh toán, không hủy), đề xuất số người = ⌈trung bình đơn mỗi giờ ÷ định mức đơn mỗi người mỗi giờ⌉ (mặc định 6). Gợi ý chỉ là bản nháp, quản trị viên phải bấm Lưu.
- **Đối soát ca thu ngân:** danh sách ca thu ngân trong tuần với người mở, người chốt, tiền dự kiến, tiền kiểm đếm, chênh lệch và lời giải trình, kèm tổng chênh lệch của tuần. Chênh lệch gắn với ca thu ngân, không chia cho từng nhân viên.
- **Xuất bảng công CSV:** nhân viên chỉ xuất của mình, quản trị viên xuất toàn bộ; có bảo vệ chuỗi công thức.
- Mọi ngày giờ dùng múi giờ **Asia/Ho_Chi_Minh (UTC+7)**, không phụ thuộc máy người dùng; Admin và Staff tự cập nhật lẫn nhau qua thông báo realtime. Hệ thống không tính lương và không xác minh vị trí khi chấm công.

### 3.7. Các chức năng bổ sung của nhóm

- **Tặng đồ uống cho bàn khác** (`/gifts`): khách chọn bàn nhận, món và lời nhắn (tối đa 140 ký tự), chọn gửi **ẩn danh** hoặc **hiện bàn gửi**. Bàn nhận thấy lời mời trong 5 phút và bấm Nhận hoặc Từ chối; nhận xong người gửi có thêm 5 phút để xem giá và xác nhận đặt. Đơn **thuộc phiên của người gửi** (tiền, kho, hóa đơn đều tính ở bàn gửi) và KDS giao đúng bàn nhận. Có giới hạn số lời mời đồng thời và tần suất gửi.
- **Jukebox** (`/jukebox`, `/staff/jukebox`, `/admin/jukebox`): quản trị viên thêm bài từ link YouTube hoặc file âm thanh HTTPS (tối đa 100 bài). Mỗi phiên bàn có 1 lựa chọn chung và 5 lượt bình chọn, chỉ trừ lượt khi bài được quầy chọn phát. Hàng đợi xếp theo số bàn bình chọn; máy nối loa ở quầy giữ quyền phát (lease 30 giây, heartbeat 10 giây), có tạm dừng, âm lượng, bài tiếp theo và chọn trực tiếp một bài. Bộ điều khiển vẫn hiện khi chuyển sang chức năng khác.
- **Ảnh in tem dán cốc** (`/staff/cup-prints`, `/admin/cup-prints`): khách tải ảnh JPEG / PNG / WebP (tối đa 4 MB) cho từng dòng món; server giải mã thật, chuẩn hóa, bỏ EXIF, lưu trong MongoDB. Sau khi đơn đã thanh toán, nhân viên xem trước và in tem 60×80 mm (A4 hoặc giấy tem). Lệnh in chỉ được ghi nhận là "yêu cầu in".
- **Latte art từ ảnh:** từ ảnh gốc tạo 3 bản tông cà phê (Latte mềm / Cacao đậm / Chấm nghệ thuật) để khách chọn; nhân viên tải file PNG 1024×1024 cho máy in đồ uống (dự kiến). Không dùng AI, không gửi ảnh cho bên thứ ba.
- **Màn hình hiển thị QR tại bàn** (`/display`): trang tùy chọn hiển thị mã QR cố định của bàn.
- **Hỗ trợ khách hàng:** liên kết tới bot Telegram CSKH (cấu hình được).
- **PWA:** ứng dụng có thể thêm vào màn hình chính, giữ giao diện và giỏ khi mất mạng, hiển thị trạng thái kết nối; đơn chỉ được gửi khi online.

### 3.8. Giám sát và phát hiện bất thường

- **Trang Vận hành** (`/admin/operations`, tự làm mới 30 giây): kết nối MongoDB / Redis, heartbeat các worker, số bàn / đơn / yêu cầu đang xử lý, tiến độ đơn theo cột KDS, sự kiện outbox và công việc nền lỗi cần phát lại, chi tiết kỹ thuật gấp sẵn.
- **Metrics:** `GET /metrics` định dạng Prometheus (không công khai ra Nginx): số request, độ trễ, sự kiện nghiệp vụ, thời gian từng công đoạn đơn, số kết nối socket, tình trạng phụ thuộc. Nhãn có tập giá trị hữu hạn, log JSON có `requestId` và che thông tin nhạy cảm.
- **Phát hiện bất thường** theo luật và thống kê (không phải mô hình học máy): bốn bộ phát hiện — tỉ lệ lỗi HTTP, độ trễ p95 của API, thời gian pha chế p95, tỉ lệ hủy đơn. So cửa sổ 15 phút hiện tại với baseline 60 phút liền trước, cần tối thiểu 20 mẫu, chưa đủ dữ liệu thì ghi rõ thay vì suy diễn. Cảnh báo có bằng chứng, có thời gian chờ chống lặp, vòng đời `OPEN → ACKNOWLEDGED → CLOSED` do quản trị viên xử lý và được audit. Phần giải thích do AI viết (tùy chọn) chỉ nhận số liệu tổng hợp; thiếu AI thì dùng mẫu giải thích có sẵn.

## 4. Luồng nghiệp vụ chính

Một ngày vận hành đi qua chín bước nối tiếp:

1. **Chuẩn bị (quản trị viên):** tài khoản nhân viên, thực đơn và giá, bàn và mã QR, tồn kho, voucher.
2. **Mở ca (nhân viên):** đăng nhập cổng nhân viên, mở ca thu ngân với tiền đầu ca.
3. **Khách quét QR, chọn món, đặt món:** phiên bàn tự mở, báo giá có chữ ký, đặt đơn ở trạng thái chờ thanh toán; nguyên liệu và lượt voucher được giữ.
4. **Thu tiền (nhân viên):** nhận quyền thu bàn, ghi nhận khoản thu (từng phần hoặc đủ); đủ tiền thì đơn lên KDS.
5. **Pha chế (KDS):** Chờ xác nhận → Đã nhận → Đang pha (trừ nguyên liệu) → Sẵn sàng → Đã phục vụ (cộng điểm cho thành viên); khách theo dõi realtime.
6. **Gọi nhân viên và yêu cầu hỗ trợ.**
7. **Đóng phiên, chốt hóa đơn, đánh giá:** hóa đơn bất biến, in từ trình duyệt, quyền khách bị thu hồi.
8. **Các luồng ngoại lệ:** hủy đơn và hoàn tiền, điều chỉnh khoản thu có quản trị viên duyệt.
9. **Chốt ca thu ngân và đối soát, thống kê doanh thu, giám sát vận hành.**

### Máy trạng thái của đơn hàng

| Trạng thái | Ý nghĩa | Chuyển tiếp hợp lệ |
|---|---|---|
| `AWAITING_PAYMENT` | Đơn mới, chờ nhân viên thu tiền | Thu đủ → `PENDING`; hủy → `CANCELLED` |
| `PENDING` | Đã thu đủ, chờ quầy nhận | → `CONFIRMED`; hủy kèm hoàn tiền → `CANCELLED` |
| `CONFIRMED` | Quầy đã nhận đơn | → `PREPARING`; hủy kèm hoàn tiền → `CANCELLED` |
| `PREPARING` | Đang pha chế, nguyên liệu bị trừ khỏi kho | → `READY` |
| `READY` | Đã pha xong | → `SERVED` |
| `SERVED` | Đã phục vụ; thành viên được cộng điểm | Kết thúc |
| `CANCELLED` | Đã hủy; trả nguyên liệu, lượt voucher, điểm | Kết thúc |

Server kiểm tra trạng thái hiện tại và `version` trước mỗi lần chuyển, nên client không thể đặt trạng thái tùy ý, không nhảy cóc, không lùi bước, và hai nhân viên bấm cùng lúc chỉ một thao tác thành công.

## 5. Kiến trúc hệ thống

Hệ thống là **modular monolith**: một mã nguồn nhưng chạy thành nhiều tiến trình có vai trò khác nhau, dùng chung MongoDB và Redis.

```
Khách     :8080 ─► Nginx ─► Guest API (A, B) ───┐
Nhân viên :8081 ─► Nginx ─┐                      ├─► MongoDB (replica set)
Quản trị  :8082 ─► Nginx ─┴► Internal API (A, B)─┤
                                                 ├─► Redis
Worker: scheduler · realtime · report ◄──────────┘
```

- **Hai cụm API:** lưu lượng điện thoại của khách vào cụm Guest, nhân viên và quản trị viên vào cụm Internal (ưu tiên CPU cao hơn), nên khách đông không làm chậm màn hình thu tiền và pha chế. Biến `TRAFFIC_CLASS` khiến mỗi instance trả 404 khi bị gọi sai cụm.
- **Ba worker nền:**
  - `scheduler`: giữ quyền lãnh đạo (leader lease), chạy các tác vụ theo lịch (đóng phiên bỏ quên, quét no-show đặt bàn, hết hạn lời mời quà, phát hiện bất thường) và **relay outbox**.
  - `realtime`: lấy sự kiện từ hàng đợi Redis rồi phát qua Socket.IO bằng Redis emitter tới đúng phòng.
  - `report`: tạo báo cáo CSV theo yêu cầu để không chiếm tài nguyên của API.
- **MongoDB replica set** là bắt buộc vì đặt đơn, giữ nguyên liệu, giữ voucher, trừ điểm, ghi nhật ký và sự kiện phải cùng thành công hoặc cùng thất bại.
- **Redis:** giới hạn tần suất, cache thực đơn, hàng đợi công việc nền có retry và dead-letter, Socket.IO adapter, bộ đếm HTTP cho phát hiện bất thường.

### Kiến trúc bốn tầng của backend

| Tầng | Trách nhiệm | Ví dụ |
|---|---|---|
| Route | Khai báo đường dẫn, gắn middleware xác thực, phân quyền, CSRF, giới hạn tần suất | `guestRoutes.ts`, `paymentRoutes.ts` |
| Controller | Đọc request, kiểm tra bằng schema Zod, gọi service, trả response chuẩn | `orderController.ts` |
| Service | Toàn bộ nghiệp vụ, mở transaction, ghi audit và sự kiện outbox | `orderService.ts`, `inventoryService.ts` |
| Repository | Nơi duy nhất truy vấn MongoDB, phương thức đặt tên theo nghiệp vụ | `orderRepository.ts`, `voucherRepository.ts` |

Script `server/scripts/check-boundaries.ts` chạy cùng lệnh lint và báo lỗi nếu controller import model Mongoose, hoặc service / middleware / route import model để truy vấn thay vì đi qua repository.

### Thời gian thực bằng transactional outbox

Nếu phát sự kiện ngay lúc ghi dữ liệu, server có thể ghi xong đơn nhưng gặp sự cố trước khi phát khiến KDS không bao giờ thấy đơn. Hệ thống ghi sự kiện vào collection `outboxevents` **trong cùng transaction** với đơn hàng; relay (ở worker scheduler) nhận sự kiện bằng lease, đẩy sang hàng đợi Redis rồi mới đánh dấu đã phát; worker realtime phát qua Socket.IO. Nếu Redis lỗi, sự kiện vẫn nằm trong MongoDB để thử lại với backoff, sự kiện lỗi nhiều lần có thể được quản trị viên phát lại; phía nhận được thiết kế để chịu sự kiện trùng. Mỗi sự kiện có mã, phiên bản và phạm vi phát (`all`, `staff`, `session`, `guest`) nên khách chỉ nhận sự kiện của bàn và thiết bị mình. Khi mất kết nối socket, client vẫn lấy dữ liệu qua REST và polling dự phòng.

## 6. Mô hình dữ liệu

| Nhóm | Collection | Ghi chú thiết kế |
|---|---|---|
| Tài khoản | `users`, `refreshsessions` | `authVersion` tăng khi đổi quyền, mật khẩu hoặc khóa để thu hồi phiên cũ; refresh token chỉ lưu dạng băm |
| Thực đơn | `categories`, `products`, `toppings` | Món đã bán chỉ được lưu trữ; size có giá riêng; tag, caffeine, sữa |
| Bàn và phiên | `tables`, `tablesessions`, `guestsessions` | Chỉ một phiên hoạt động mỗi bàn (partial unique index); `staticQrEpoch` tăng khi cấp lại khóa; khách hết hạn sau 12 giờ |
| Đơn và tiền | `orders`, `payments`, `paymentcorrections`, `bills`, `cashshifts`, `reviews` | Đơn lưu bản chụp tên và giá; unique `(phiên, Idempotency-Key)`; khoản thu không xóa, chỉ đảo; hóa đơn chốt một lần cho mỗi phiên |
| Yêu cầu | `servicerequests`, `cancelrequests` | Ghi người xử lý và thời điểm |
| Kho | `ingredients`, `recipes`, `stockmovements` | `onHand` / `reserved` / `version`; sổ kho bất biến |
| Khuyến mãi | `vouchers`, `voucheruses`, `members`, `membersessions`, `pointentries` | Sổ điểm unique `(đơn, loại)`; mỗi đơn một lượt voucher |
| Đặt bàn | `reservations` | Unique `(người đặt, requestId)`; chỉ mục theo bàn và khoảng thời gian; unique phiên đã check-in |
| Nhân sự | `workshifts`, `staffingpolicies`, `workactivities`, `workschedulecopies` | Ca, giờ chấm gốc và hiệu lực, cấu hình nhu cầu, hoạt động KDS theo người và ca |
| Bổ sung | `gifts`, `jukebox*` (bài, trạng thái, bình chọn, thao tác), `cupphotos`, `cupprintattempts` | Ảnh lưu trong MongoDB có TTL |
| Hệ thống | `auditlogs`, `outboxevents`, `anomalyalerts`, `migrationstates` | Nhật ký ghi cùng transaction với dữ liệu nghiệp vụ |

Chỉ mục và collection được tạo bằng **migration có khóa và checksum** (`npm run db:migrate`); ở production Mongoose không tự tạo chỉ mục.

## 7. Thiết kế API

API kiểu REST, tiền tố `/api/v1`, phản hồi JSON thống nhất:

```jsonc
{ "success": true, "data": { } }
{ "success": false, "error": { "code": "…", "message": "…", "details": { } }, "requestId": "…" }
```

Mã lỗi nghiệp vụ được đặt tên rõ ràng (`IDEMPOTENCY_CONFLICT`, `STATE_TRANSITION_INVALID`, `PAYMENT_CLAIMED`, `QUOTE_CHANGED`, `INSUFFICIENT_STOCK`, `VOUCHER_EXHAUSTED`, `INGREDIENT_IN_USE`…). Lỗi kiểm tra dữ liệu (422) nêu lý do cụ thể cho từng ô bằng tiếng Việt.

| Nhóm | Ví dụ đường dẫn | Quyền truy cập |
|---|---|---|
| Xác thực nội bộ | `/auth/login`, `/auth/refresh`, `/auth/me`, `/auth/password` | Công khai (30 lần/phút) hoặc đã đăng nhập |
| Thực đơn | `/categories`, `/products`, `/products/best-sellers`, `/menu/search` | Công khai |
| Khách | `/table-sessions/join`, `/orders/quote`, `/orders`, `/orders/:id/cancel`, `/service-requests`, `/ai/recommendations`, `/receipts/current` | Cookie phiên khách, CSRF, giới hạn theo bàn |
| Thành viên | `/members/register`, `/members/login`, `/members/me`, `/members/points` | Công khai có CSRF; cookie thành viên |
| Đặt bàn | `/reservations`, `/reservations/availability`, `/staff/reservations/:id/check-in` | Thành viên (khách), STAFF / ADMIN |
| Tặng quà, Jukebox, ảnh cốc | `/gifts`, `/jukebox`, `/cup-photos`, `/staff/cup-prints` | Cookie phiên khách hoặc STAFF / ADMIN |
| Nhân viên | `/staff/tables`, `/staff/orders`, `/staff/orders/:id/status`, `/staff/orders/:id/payments`, `/staff/cash-shifts`, `/staff/workforce` | STAFF hoặc ADMIN |
| Quản trị | `/admin/users`, `/admin/products`, `/admin/inventory`, `/admin/vouchers`, `/admin/members`, `/admin/reports`, `/admin/operations`, `/admin/workforce`, `/admin/anomalies` | Chỉ ADMIN |
| Sức khỏe | `/healthz`, `/readyz` (nội bộ), `/metrics` (mạng riêng) | Không đi qua Nginx công khai |

## 8. Bảo mật

- **Mật khẩu:** băm bằng bcrypt; thành viên bị khóa tạm sau nhiều lần nhập sai.
- **Phiên nhân viên và quản trị:** access token JWT hạn 15 phút; refresh token xoay vòng sau mỗi lần làm mới, lưu trong cookie HttpOnly (`SameSite=Lax`), server chỉ lưu giá trị băm. `authVersion` tăng khi đổi mật khẩu, đổi quyền hoặc khóa tài khoản khiến mọi phiên cũ mất hiệu lực ngay.
- **Phân quyền ở server:** middleware `requireAuth` và `requireRole` cho từng API; một bài kiểm thử tự động duyệt toàn bộ bảng định tuyến để bảo đảm mọi route `/staff` yêu cầu STAFF hoặc ADMIN và mọi route `/admin` chỉ cho ADMIN. Kết nối Socket.IO cũng được kiểm tra vai trò và định kỳ kiểm tra lại.
- **Quy tắc chủ quán:** chỉ tài khoản quản trị đang hoạt động được tạo sớm nhất mới được tạo, sửa, khóa, đổi mật khẩu hoặc cấp quyền cho tài khoản quản trị; không ai tự khóa hay tự hạ quyền chính mình.
- **Mã QR:** token `qrs1` ký HMAC gắn mã bàn và thế hệ khóa, nên khách không thể sửa đường dẫn để vào bàn khác; quản trị viên cấp lại khóa để vô hiệu mã đã in.
- **Phía khách:** cookie phiên khách HttpOnly; mọi truy vấn đơn lọc theo `participantId`, nên khách cùng bàn không xem hay hủy được đơn của nhau. Quyền xem hóa đơn sau khi đóng phiên là cookie riêng, không dùng để đặt món.
- **CSRF:** thao tác ghi của khách so khớp `Origin` / `Referer`.
- **Giới hạn tần suất bằng Redis:** ví dụ vào bàn 20 lần/phút, đặt đơn 12 lần/phút mỗi bàn, báo giá 60 lần/phút, gọi nhân viên 6 lần/phút, đăng nhập 30 lần/phút, AI 20 lần/phút; phản hồi 429 bằng JSON tiếng Việt.
- **Khác:** `helmet`, CORS theo danh sách nguồn cho phép, giới hạn kích thước JSON 256 KB, log che `authorization`, cookie, mật khẩu, token, khóa API; ảnh tải lên được giải mã và chuẩn hóa thật, từ chối SVG / GIF / PDF; mọi thao tác quan trọng (thu tiền, hoàn tiền, điều chỉnh, kho, tài khoản, ca) đều ghi `auditlogs`.

## 9. Bảo đảm tính đúng đắn của dữ liệu

Ba nguyên tắc xuyên suốt:

1. **Backend là nguồn sự thật:** giá, trạng thái và quyền do server quyết định; giá client gửi lên bị bỏ qua, server đọc món, size, topping từ cơ sở dữ liệu và tính lại.
2. **Lưu bản chụp:** mỗi món trong đơn lưu tên, size và đơn giá lúc đặt; đơn lưu cả bản chụp ưu đãi và định lượng nguyên liệu; hóa đơn chốt khi đóng phiên là bản cố định. Đổi thực đơn hoặc voucher sau này không ảnh hưởng đơn và hóa đơn cũ.
3. **Chống trùng và tranh chấp:**
   - **Idempotency:** đặt đơn, thu tiền, nhập kho, đặt bàn, tạo ca... đều gửi kèm khóa; gửi lại cùng khóa và cùng nội dung trả kết quả cũ, cùng khóa khác nội dung trả 409; chỉ mục duy nhất ở mức cơ sở dữ liệu bảo đảm điều này.
   - **Khóa lạc quan:** đơn, khoản thu, nguyên liệu, voucher, phiên, ca đều có `version`; cập nhật chỉ thành công khi version khớp.
   - **Cập nhật có điều kiện:** giữ nguyên liệu chỉ khi `onHand − reserved ≥ số cần`; dùng voucher chỉ khi `usedCount < maxUses`; trừ điểm chỉ khi `points ≥ số trừ`.
   - **Transaction:** các thay đổi liên quan (đơn, khoản thu, kho, voucher, điểm, nhật ký, sự kiện) ghi trong một transaction MongoDB.

## 10. Công nghệ sử dụng

| Thành phần | Công nghệ |
|---|---|
| Giao diện | React 18, Vite, TypeScript, React Router, TanStack Query, Zustand, Tailwind CSS, Radix UI, Recharts, Lucide, Socket.IO client |
| Máy chủ | Node.js 24, Express, TypeScript, Mongoose, Socket.IO, Zod, bcryptjs, JSON Web Token, Pino, bộ đo Prometheus, sharp |
| Kiểm tra dữ liệu dùng chung | `packages/contracts` — schema Zod dùng cho cả client và server |
| Cơ sở dữ liệu | MongoDB 7 chạy replica set (hỗ trợ transaction nhiều tài liệu) |
| Bộ nhớ đệm, hàng đợi | Redis 7 (giới hạn tần suất, cache thực đơn, hàng đợi worker, Socket.IO adapter) |
| AI | API Chat Completions tương thích OpenAI (tùy chọn), có chế độ dự phòng theo luật |
| Triển khai | Docker Compose, Nginx, Cloudflare Tunnel (công khai link cho điện thoại, tùy chọn) |
| Kiểm thử | Vitest, Testing Library, Supertest, MongoMemoryReplSet, Playwright, k6 |
| CI | GitHub Actions (lint, typecheck, test, build) |

## 11. Cấu trúc thư mục

```
.
├── client/                    Giao diện React (khách, nhân viên, quản trị)
│   ├── public/                Ảnh món, phông chữ, service worker, manifest PWA
│   └── src/
│       ├── features/          guest · staff · admin · auth · ai · gifts · jukebox ·
│       │                      reservations · cupPhotos
│       ├── layouts/           GuestLayout, StaffLayout, AdminLayout, ReservationLayout
│       ├── components/ui/     Button, Card, Modal, Toast… dùng chung
│       ├── lib/               api, socket, portals, cn…
│       └── store/             giỏ hàng (Zustand)
├── server/
│   ├── src/
│   │   ├── routes/            Khai báo đường dẫn theo miền nghiệp vụ
│   │   ├── controllers/       Nhận request, kiểm tra bằng Zod
│   │   ├── services/          Nghiệp vụ, transaction
│   │   ├── repositories/      Toàn bộ truy vấn MongoDB
│   │   ├── models/            Schema Mongoose
│   │   ├── middlewares/       auth, guest, error
│   │   ├── realtime/          Socket.IO
│   │   ├── infrastructure/    mongo, redis, hàng đợi, metrics, logger, unit of work
│   │   ├── ai/ providers/     Lời nhắc, hook và nhà cung cấp AI
│   │   ├── seeds/             Dữ liệu demo
│   │   ├── server.ts          Tiến trình API
│   │   └── worker.ts          Tiến trình worker nền
│   ├── scripts/               migrate, wait-db, check-boundaries, drill hàng đợi…
│   └── e2e/                   Playwright
├── packages/contracts/        Schema Zod và kiểu dữ liệu dùng chung
├── docker/nginx/              Cấu hình Nginx tách ba cổng
├── scripts/                   Khởi tạo Mongo, sao lưu / khôi phục, k6, nạp dữ liệu mẫu
├── docs/data/                 Dữ liệu mẫu kho nguyên liệu
├── compose.yaml               MongoDB cho môi trường phát triển
├── compose.production.yaml    Production cục bộ: Nginx, 4 API, 3 worker, Mongo, Redis
├── compose.public.yaml        Bổ sung công khai qua đường hầm
└── .env.example               Mẫu cấu hình
```

## 12. Cài đặt và chạy ở môi trường phát triển

**Yêu cầu:** Node.js **24 trở lên** (xem `.nvmrc`), **Docker Desktop** đang chạy (cho MongoDB và Redis).

```bash
npm ci
npm run db:up                                          # MongoDB replica set
docker run -d --name maycafe-redis -p 6379:6379 redis:7-alpine
npm run db:wait                                        # chờ MongoDB sẵn sàng
npm run build:contracts                                # build schema dùng chung
npm run db:migrate                                     # tạo chỉ mục
npm run seed                                           # nạp dữ liệu demo — THAY dữ liệu trong DB, chỉ chạy lần đầu
npm run dev                                            # API + worker + web
```

- Web: <http://localhost:5173>; API: <http://localhost:4000>.
- Nhân viên và quản trị đăng nhập tại `/auth/login`.
- Không cần tạo file `.env`, các giá trị mặc định đủ để chạy; server đọc cấu hình dev từ `server/.env` nếu có.
- **Lần sau chạy lại:** bật Docker Desktop, chạy `docker start maycafe-mongo maycafe-redis`, rồi `npm run dev`. Không seed lại.
- **Windows báo `listen EACCES … :4000`** (cổng bị hệ thống giữ): chạy với cổng khác.
  ```powershell
  $env:PORT = "4100"
  $env:API_PROXY_TARGET = "http://localhost:4100"
  npm run dev
  ```

### Vào bàn như một khách

Đăng nhập quản trị → **Bàn & QR** → **Xem QR tĩnh** của một bàn → mở hoặc sao chép đường dẫn `/t/<token>` trên điện thoại (hoặc quét mã QR hiển thị trong hộp thoại). Để điện thoại truy cập được, đặt `PUBLIC_APP_URL` là địa chỉ máy tính trong mạng LAN (không dùng `localhost`).

### Bản đồ đường dẫn giao diện

| Cổng | Đường dẫn |
|---|---|
| Khách | `/t/:token` (vào bàn), `/menu`, `/cart`, `/orders`, `/receipt`, `/member`, `/ai`, `/gifts`, `/jukebox`, `/reservations`, `/reservations/mine`, `/display` |
| Nhân viên | `/staff/kds`, `/staff/tables`, `/staff/service-requests`, `/staff/availability`, `/staff/cash-shift`, `/staff/payments`, `/staff/cup-prints`, `/staff/jukebox`, `/staff/reservations`, `/staff/workforce`, `/staff/account` |
| Quản trị | `/admin/dashboard`, `/admin/products`, `/admin/categories`, `/admin/toppings`, `/admin/tables`, `/admin/users`, `/admin/inventory`, `/admin/vouchers`, `/admin/members`, `/admin/bills`, `/admin/payments`, `/admin/reviews`, `/admin/operations`, `/admin/workforce`, `/admin/reservations`, `/admin/cup-prints`, `/admin/jukebox`, `/admin/account` |

## 13. Chạy bản production cục bộ bằng Docker

Tạo file `.env` ở thư mục gốc (tham khảo `.env.example`) với hai khóa `JWT_ACCESS_SECRET` và `JWT_REFRESH_SECRET` **khác nhau**, mỗi khóa tối thiểu 32 ký tự, rồi:

```bash
docker compose -f compose.production.yaml up -d --build
```

| Container | Số lượng | Vai trò |
|---|---|---|
| `mongo` | 1 | MongoDB 7 replica set |
| `redis` | 1 | Redis 7 (AOF) |
| `migrate` | 1 (chạy một lần) | Áp migration trước khi API và worker khởi động |
| `server-guest-a`, `server-guest-b` | 2 | API cho khách |
| `server-internal-a`, `server-internal-b` | 2 | API cho nhân viên và quản trị |
| `worker-scheduler`, `worker-realtime`, `worker-report` | 3 | Tác vụ theo lịch + outbox, phát realtime, tạo báo cáo |
| `web` | 1 | Nginx phục vụ giao diện và tách ba cổng |

Ba cổng: khách <http://localhost:8080>, nhân viên <http://localhost:8081>, quản trị <http://localhost:8082>. Mỗi API giới hạn mặc định 0,75 CPU và 384 MiB; có thể chỉnh bằng các biến `GUEST_API_*`, `INTERNAL_API_*`, `WORKER_*`, `WEB_*`. Cập nhật phiên bản có thể làm lần lượt từng cụm API (A trước, B sau) để không gián đoạn.

Để khách quét QR bằng điện thoại qua mạng di động, script `scripts/public-web.ps1` khởi động Cloudflare Tunnel, lấy địa chỉ HTTPS công khai và cập nhật `PUBLIC_APP_URL` để mã QR trỏ tới đó. Sao lưu và kiểm tra khôi phục: `scripts/db/backup.ps1`, `scripts/db/restore-test.ps1`.

## 14. Cấu hình

Biến môi trường chính (đầy đủ trong `.env.example`):

| Nhóm | Biến | Mặc định / ý nghĩa |
|---|---|---|
| Máy chủ | `PORT`, `TRAFFIC_CLASS` | `4000`; `unified` / `guest` / `internal` |
| Địa chỉ | `PUBLIC_APP_URL`, `STAFF_APP_URL`, `ADMIN_APP_URL`, `SERVER_ORIGIN` | Nguồn được phép (CORS, CSRF) và địa chỉ trong mã QR |
| Cơ sở dữ liệu | `MONGODB_URI`, `REDIS_URL` | `mongodb://127.0.0.1:27017/maycafe?replicaSet=rs0…`, `redis://127.0.0.1:6379` |
| Xác thực | `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `ACCESS_TOKEN_TTL`, `REFRESH_TOKEN_TTL` | Production bắt buộc ≥ 32 ký tự; `15m`, `30d` |
| Cookie | `COOKIE_SECURE`, `COOKIE_DOMAIN` | Bật `COOKIE_SECURE` khi dùng HTTPS |
| Phiên bàn | `GUEST_AUTO_OPEN`, `SESSION_IDLE_TIMEOUT_MIN` | `true`; `60` (0 = tắt) |
| Giới hạn tần suất | `RATE_LIMIT_*` | Xem mục 8 |
| Báo giá, cache | `ORDER_QUOTE_TTL_MS`, `MENU_CACHE_TTL_SECONDS` | `120000`, `60` |
| Worker và outbox | `WORKER_ROLE`, `OUTBOX_*`, `BACKGROUND_JOB_*` | `all` / `scheduler` / `realtime` / `report` |
| AI | `AI_MODE`, `AI_PROVIDER`, `AI_BASE_URL`, `AI_MODEL`, `AI_API_KEY`, `AI_TIMEOUT_MS` | Mặc định `fallback` (không gọi AI) |
| Phát hiện bất thường | `ANOMALY_*` | Cửa sổ, baseline, số mẫu tối thiểu, ngưỡng; `ANOMALY_AI_MODE` điều khiển riêng phần giải thích |
| Giao diện (build) | `VITE_SHOW_DEMO_CREDENTIALS`, `VITE_SUPPORT_TELEGRAM_URL` | `false`; link bot CSKH (đặt rỗng để ẩn) |

**Bật AI Barista:** đặt `AI_MODE=live`, `AI_MODEL` (mã mô hình, bắt buộc tường minh) và `AI_API_KEY`; `AI_BASE_URL` là địa chỉ gốc (không kèm `/chat/completions`). Chỉ chọn mô hình hỗ trợ đủ Chat Completions với xác thực Bearer và chế độ JSON-object. Khóa API chỉ nằm ở backend. Muốn dùng AI giải thích cảnh báo bất thường, đặt riêng `ANOMALY_AI_MODE=live`.

## 15. Dữ liệu demo và tài khoản

`npm run seed` nạp dữ liệu mẫu: danh mục, món, topping, bàn, lịch sử đơn 30 ngày, một phiên mở sẵn, và in ra liên kết thiết lập màn hình hiển thị từng bàn.

| Tài khoản | Mật khẩu | Vai trò |
|---|---|---|
| `admin@maycafe.vn` | `MayCafe@2025` | Quản trị (chủ quán) |
| `staff.a@maycafe.vn` | `MayCafe@2025` | Nhân viên |
| `staff.b@maycafe.vn` | `MayCafe@2025` | Nhân viên |

Trang đăng nhập mặc định không hiện sẵn tài khoản demo; chỉ bật `VITE_SHOW_DEMO_CREDENTIALS=true` khi trình diễn. Kho nguyên liệu mẫu theo thực đơn nạp thêm bằng `scripts/setup-inventory-demo.mjs --apply` (cần đặt `INVENTORY_ADMIN_PASSWORD`), dùng dữ liệu `docs/data/inventory-demo.json`.

> **Cảnh báo:** `npm run seed` thay thế dữ liệu trong cơ sở dữ liệu; chỉ chạy trên cơ sở dữ liệu có thể xóa.

## 16. Kiểm thử

| Lớp | Công cụ | Phạm vi |
|---|---|---|
| Unit test server (20 file) | Vitest | Hàm thuần: định giá, chữ ký QR, phân quyền theo bảng định tuyến, thông báo lỗi, xuất báo cáo, tìm kiếm, AI, phát hiện bất thường |
| Integration test server (17 file) | Vitest, Supertest, MongoMemoryReplSet | Luồng thật qua API trên MongoDB replica set có transaction: đặt đơn, thu tiền, kho, voucher, tài khoản, đặt bàn, ca làm, quà, Jukebox, ảnh cốc… |
| Test giao diện (39 file) | Vitest, Testing Library (jsdom) | Các trang khách, nhân viên, quản trị |
| Kiểm tra tĩnh | TypeScript, ESLint, `check-boundaries` | Kiểu dữ liệu, quy tắc mã nguồn, ranh giới giữa các tầng |
| Trình duyệt | Playwright | Kiểm tra giao diện đáp ứng (cần bản production đang chạy) |
| Tải | k6 (`scripts/load/`) | Thực đơn, luồng khách, realtime, QR |

Integration test chạy trên MongoDB thật thay vì giả lập repository vì phần dễ sai nhất là transaction và khóa lạc quan. Nhiều test mô phỏng tranh chấp: hai yêu cầu song song chỉ một thành công, hoặc cố tình gây lỗi giữa chừng rồi kiểm tra mọi thay đổi được hoàn tác. Quy trình CI nằm ở `.github/workflows/ci.yml`.

## 17. Các lệnh thường dùng

| Lệnh | Mô tả |
|---|---|
| `npm run dev` | Chạy API, worker và web ở chế độ phát triển |
| `npm run build` | Build contracts, server và client |
| `npm run typecheck` | Kiểm tra kiểu TypeScript toàn bộ |
| `npm run lint` | ESLint và kiểm tra ranh giới giữa các tầng |
| `npm test` | Unit test, integration test và test giao diện |
| `npm run test:server` / `test:client` / `test:integration` | Chạy riêng từng nhóm test |
| `npm run test:e2e` | Playwright, cần bản production đang chạy |
| `npm run test:queue-drill` | Diễn tập khôi phục hàng đợi trên Redis cô lập |
| `npm run seed` | Nạp lại dữ liệu demo (xóa dữ liệu hiện có) |
| `npm run db:up` / `npm run db:down` | Bật / tắt MongoDB bằng Docker |
| `npm run db:wait` | Chờ MongoDB sẵn sàng |
| `npm run db:migrate` | Áp migration và chỉ mục |
| `npm run format` | Định dạng mã nguồn bằng Prettier |

## 18. Hạn chế đã biết

- **Chưa tích hợp thanh toán trực tuyến:** hệ thống chỉ ghi nhận khoản thu tại quầy (tiền mặt, chuyển khoản đã kiểm tra, khác); việc hoàn tiền thực tế do nhân viên thực hiện.
- **QR cố định không chứng minh khách đang ngồi tại quán:** nếu nghi ngờ mã bị chụp lại, quản trị viên cấp lại khóa; nhân viên cần đóng phiên khi khách rời bàn.
- **AI Barista phụ thuộc dịch vụ mô hình ngôn ngữ bên ngoài** khi bật chế độ live; khi không khả dụng, hệ thống dùng gợi ý theo luật nên kém linh hoạt hơn. Chưa đánh giá chất lượng và chi phí ở quy mô lớn. Phát hiện bất thường dựa trên luật và thống kê, không phải học máy.
- **Thành viên không xác minh số điện thoại bằng SMS** nên không có chức năng tự khôi phục mật khẩu.
- **Kho:** chưa quản lý lô và hạn dùng, chưa tự quy đổi đơn vị, chưa tính định lượng theo mức đường hoặc đá; đơn chờ thanh toán không tự hết hạn giữ kho (nhân viên hủy để giải phóng).
- **Đặt bàn:** chưa có đổi bàn hay đổi giờ cho lịch chưa check-in, chưa ghép bàn, chưa thu cọc hoặc phí no-show, chưa gửi SMS nhắc.
- **Ca làm:** không tính lương, không xác minh vị trí khi chấm công; chênh lệch tiền ca gắn với ca thu ngân chứ không chia cho từng người.
- **Ảnh in tem và latte art:** chưa kết nối máy in thực phẩm; hệ thống chỉ chuẩn bị file và ghi nhận yêu cầu in.
- **Jukebox:** phụ thuộc nguồn nhạc cho phép nhúng và việc trình duyệt cho phép tự phát âm thanh.
- **PWA không gửi đơn offline:** chỉ lưu giao diện và giỏ, thao tác ghi luôn cần mạng.
- **Chưa triển khai lên đám mây / HTTPS thật:** mới chạy production cục bộ bằng Docker Compose; hiệu năng đã thử trên Docker Desktop không phải cam kết năng lực cloud.
- **Dữ liệu demo là dữ liệu thử nghiệm:** các chỉ số như tỉ lệ hủy không đại diện cho một quán thật; việc kiểm thử trên nhiều điện thoại thật còn hạn chế.

## 19. Chức năng đã gỡ

Ngày 07/11/2026 nhóm gỡ các chức năng sau khỏi hệ thống; API tương ứng trả 404, dữ liệu đơn, khoản thu, kho và hóa đơn lịch sử không bị xóa:

- Mini-game "Ai trả tiền?"
- Đặt mang đi (Takeaway) và đặt trước nhận tại quầy (Pre-order Pickup)
- AI Freestyle (đồ tự chế ngoài thực đơn)
- Smart Insights và gợi ý bán kèm (phần giờ cao điểm và món bán chạy vẫn còn trên Dashboard)

## 20. Nguồn ảnh

Ảnh món trong `client/public/menu-images/` lấy từ Unsplash và Pexels, dùng cho mục đích học tập. Phông chữ Be Vietnam Pro theo giấy phép SIL OFL (`client/public/fonts/OFL.txt`).
