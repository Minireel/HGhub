# 03 · Quy tắc của host

[← Tổng quan](README.md)

Chương này liệt kê các ràng buộc cấu trúc của host đối với plugin và quy tắc hành vi của chính host. Các quy tắc này không phải xét duyệt bảo mật, cũng không phán xét công dụng của plugin:
chúng chỉ đảm bảo "một plugin xảy ra vấn đề không kéo sập ứng dụng", và "ứng dụng không gây phiền cho thượng nguồn".

## 1. Cách ly

Được đảm bảo bởi cấu trúc runtime (xem [02 §1](02-runtime.md)):

- Plugin không chạm được vào hệ thống tệp, tiến trình, giao diện nội bộ của ứng dụng và dữ liệu của các plugin khác; mọi việc bên ngoài nó có thể làm đều qua `hg.*`.
- Mỗi plugin một runtime, ngân sách bộ nhớ và điều độ thực thi độc lập; vòng lặp vô hạn của script, vượt bộ nhớ do host ngắt và xây dựng lại runtime đó.
  Không đảm bảo cách ly sự cố cấp tiến trình của engine native, không yêu cầu hộp cát cấp tiến trình.
- Plugin không thể vẽ giao diện: nó trả về dữ liệu, giao diện do host kết xuất. Ngoại lệ duy nhất là khung xem web do người dùng chủ động mở ([04 §12](04-host-api.md)).

Plugin có thể đọc ghi bền vững lưu trữ và Cookie thông thường của chính mình. Phiên do khung xem đăng nhập bàn giao được host sử dụng theo plugin và origin ([04 §5](04-host-api.md#5-hgcookies)). Plugin cũng có thể đọc cài đặt, môi trường chạy của chính mình và các tham số nghiệp vụ host truyền vào.
Nó có thể qua giao diện `hg.*` đã công khai lấy dữ liệu bên ngoài, không thể truy cập trực tiếp dữ liệu nội bộ của host.

**Resolver** là sự hợp tác xuyên plugin duy nhất ([06 §3](06-capabilities.md)): host giao địa chỉ trang phát mà nguồn dữ liệu đưa ra và
header yêu cầu **không phải thông tin xác thực** cho resolver, resolver trả về địa chỉ có thể phát. Hai phía không chia sẻ lưu trữ, Cookie và trạng thái đăng nhập,
header yêu cầu `Authorization` và `Cookie` sẽ không được chuyển giao.
Nhà cung cấp của tài nguyên media, chuỗi phân giải và môi trường yêu cầu thực tế do host ghi lại, xem [09](09-media-transport.md).

## 2. Mạng

Yêu cầu API của plugin và toàn bộ media host lấy (gồm hình ảnh, phụ đề và video) đều qua lớp mạng của host.
Khung xem web tùy chọn dùng ngăn xếp mạng native của nền tảng, không cam kết mọi tài nguyên con của trang đều qua HTTP client này hay cơ chế cố định DNS;
nó là môi trường duyệt độc lập do người dùng chủ động mở, không thể truy cập giao diện cầu nối của ứng dụng. Việc truyền tải phiên cục bộ của chính host không được coi là yêu cầu ra ngoài tới thượng nguồn.

| Quy tắc | Mô tả |
| --- | --- |
| Giao thức | Chỉ cho phép `https:`; `http:` chỉ khi plugin khai báo `allowInsecureHttp` |
| Hình thức địa chỉ | Không cho phép địa chỉ chứa tên người dùng mật khẩu; không cho phép ký tự điều khiển |
| Bảo vệ mạng nội bộ | Mặc định từ chối các đích phân giải tới vòng lặp, mạng riêng, link-local, địa chỉ dành riêng, và các tên host như `localhost`, `.local`, `.internal`; đoạn FakeIP của phần mềm proxy 198.18.0.0/15 được cho qua. Nếu cùng nhóm đáp án DNS chứa IPv4 được phép và IPv6 bị hạn chế, bỏ qua IPv6 bị hạn chế, chỉ cố định tới địa chỉ được phép; không có địa chỉ được phép hoặc lẫn IPv4 bị hạn chế thì vẫn từ chối |
| DNS | Kết quả phân giải đã qua kiểm tra được dùng trực tiếp để kết nối (cố định DNS), không phân giải lại sau khi kiểm tra |
| Chuyển hướng | Tối đa 5 bước nhảy, mỗi bước nhảy kiểm tra lại; cấm hạ cấp HTTPS → HTTP |
| Header yêu cầu | Loại bỏ các header do tầng truyền tải quyết định như `Host`, `Content-Length`, `Connection`, `Transfer-Encoding`, `Accept-Encoding` |
| Proxy | Tuân theo cài đặt proxy của hệ thống; người dùng cũng có thể dùng biến môi trường để chỉ định cưỡng bức |

Bảo vệ mạng nội bộ và "chế độ bảo mật nghiêm ngặt" hiện có của ứng dụng là cùng một bộ cài đặt: khi người dùng vì dịch vụ tự xây (NAS, thư viện media tự lưu trữ)
tắt chế độ nghiêm ngặt, lưu lượng API của plugin và media cũng được nới lỏng theo. Host **nên** trong giao diện nói rõ phạm vi tác dụng, không mô tả khung xem web thành cùng một ngăn xếp mạng.

## 3. Thân thiện với thượng nguồn

Bảng dưới đây là chính sách điều độ mặc định của HGhub, do host thực thi, plugin không cần triển khai; tốc độ, đồng thời và dung lượng bộ nhớ đệm không làm hằng số giao thức cố định,
có thể tinh chỉnh theo vận hành thực tế. Việc hủy bỏ, không có thực thi nền tự do và biên giới kích hoạt bởi người dùng vẫn là thỏa thuận giao thức.

| Hạng mục | Giá trị mặc định |
| --- | --- |
| Tốc độ yêu cầu giao diện | 5 lần/giây, bùng nổ 15, đồng thời một plugin 6 (một host ≤ 4) |
| Gặp 429 / 503 | Theo `Retry-After` làm nguội host đó (giới hạn trên 10 phút), trong thời gian làm nguội yêu cầu thất bại ngay và gợi ý thử lại sau |
| Khử trùng đang bay | GET ẩn danh có tham số giống nhau được hợp nhất thành một yêu cầu |
| Hoạt động nền | Không có. Host không gọi, plugin không chạy |
| Tải trước thông tin phát | Tối đa sớm một mục; khi mạng bị hạn chế hoặc hạn ngạch không đủ thì không tải trước |
| Tìm kiếm tổng hợp | Chỉ kích hoạt khi người dùng gửi tìm kiếm; gợi ý nhập liệu không xuyên plugin |
| Báo cáo tiến độ | Mỗi 30 giây tối đa một lần, cộng thêm bắt đầu, tạm dừng, kết thúc mỗi cái một lần |
| Kiểm tra cập nhật | Tự động kiểm tra cùng một plugin cùng một địa chỉ không ngắn hơn 24 giờ một lần; kiểm tra thủ công của người dùng không bị giới hạn khoảng cách này |

Yêu cầu API, phát tương tác, tải xuống lần lượt xếp hàng; phân đoạn phát không tính vào hạn ngạch 5 lần/giây của API, tránh việc đệm media chặn đăng nhập và tìm kiếm.
Việc làm nguội `Retry-After` của cùng một host tài nguyên được chia sẻ giữa các hàng đợi và plugin; chỉ phản hồi thực tế hiện tại mới kích hoạt làm nguội, không thể dùng lỗi plugin để tùy tiện làm nguội host khác.
Host đặt đồng thời có giới hạn riêng cho media, ưu tiên việc phát hiện tại, rồi mới xét tải trước và tải xuống. Tải xuống xếp hàng theo từng tập, không phân giải đồng thời theo lô toàn bộ thư viện.
Việc khử trùng GET ẩn danh giới hạn trong cùng một plugin, phán đoán theo địa chỉ yêu cầu thực tế của mỗi bước nhảy và header yêu cầu hiệu lực; yêu cầu mang Authorization hoặc Cookie không hợp nhất. Một bên chờ hủy bỏ hoặc timeout không thể ngắt yêu cầu mà các bên khác vẫn cần; khi toàn bộ bên chờ thoát thì dừng yêu cầu thượng nguồn. Việc giải mã phản hồi thực thi riêng, các yêu cầu có chế độ chuyển hướng, chính sách Cookie và giới hạn trên thân phản hồi khác nhau không hợp nhất. Giấy phép đồng thời kéo dài tới khi đọc xong thân phản hồi hoặc yêu cầu bị hủy, không thể giải phóng sớm khi nhận được header phản hồi.

## 4. Quyền riêng tư

- Plugin có thể lấy thông tin chạy, cài đặt, lưu trữ, Cookie, tham số nghiệp vụ trong phạm vi của chính nó và nội dung bên ngoài lấy qua API công khai.
- Host không lộ cho plugin định danh thiết bị, hệ thống tệp, dữ liệu của các plugin khác hay toàn bộ thư viện cục bộ.
  `updates.check` và `progress.report` chỉ nhận các mục cần thiết của plugin sở thuộc và tiến độ phát hiện tại, không được mượn đó liệt kê các nguồn khác.
- Host không thu thập, không tải lên bất kỳ dữ liệu sử dụng nào; nhật ký chỉ tồn tại trên máy này.
