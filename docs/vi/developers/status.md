# Trạng thái hỗ trợ hiện tại

Trang này ghi lại tình hình hỗ trợ của nhánh `main` đối với HPP `1.0.0-dev`. Ngày 2026-11-18 đã cập nhật kết quả nghiệm thu đăng nhập web của Windows và Android.

| Phạm vi | Trạng thái hiện tại | Biên giới kiểm chứng |
| --- | --- | --- |
| Nhập đơn tệp/danh sách/URL | Đã triển khai | Rust và UI Windows; Android đơn tệp đo thực tế |
| Đăng ký và cập nhật tự động | Đã triển khai | Làm nguội, bỏ qua thủ công, hồi quy hoãn khi phát |
| Runtime độc lập và hủy bỏ | Lõi đã triển khai | Đồng thời/ngắt/Promise chia sẻ; kiểm tra tầng cao nhất của module và factory mỗi cái có tổng ngân sách 5 giây, đoạn đồng bộ 1 giây và hủy khởi tạo có hồi quy QuickJS thật; cầu nội bộ không lộ cho plugin |
| HTTP/Cookie/lưu trữ | Lõi đã triển khai | Hồi quy chính sách mạng (gồm DNS hỗn hợp IPv4 được phép và IPv6 bị hạn chế), bộ nhớ đệm, hạn ngạch và lưu trữ bền vững; chưa kiểm chứng môi trường mạng này trong cửa sổ Tauri thực tế |
| Khả năng nội dung và thư viện cục bộ | Đã nối vào UI | Nguồn hư cấu trên giao diện bản sản xuất tích hợp đầy đủ, 25 nhóm đạt |
| MP4/HLS/resolver | Lõi đã triển khai | Hồi quy Range, khóa, thuộc tính tài nguyên và hủy bỏ |
| Chuẩn bị phát nối tiếp và tải trước theo byte | Đã triển khai chiến lược có giới hạn | Tái sử dụng chuẩn bị tập tiếp theo; bộ nhớ đệm tài nguyên bắt đầu phát của MP4 theo khoảng và HLS tĩnh; kiểm thử quay về khi hủy bỏ, hết hạn và thay đổi phiên bản; người dùng đã phản hồi việc phát và phát nối tiếp trên Windows/Android mượt, không phải nghiệm thu đầy đủ toàn nền tảng |
| RSA | API bắt buộc đã triển khai | SPKI/PKCS#1, ba loại đệm; kiểm chứng mã hóa/giải mã bằng QuickJS thật, tính ngẫu nhiên, biên tham số và độ dài, hồi quy requires; khóa 1024–8192 bit |
| Thăm dò khả năng thiết bị | Đã nối vào API khả năng của WebView hiện tại | Đường đi thực tế của HLS và video thông thường được thăm dò riêng, giao thoa codec bảo thủ; hồi quy tính nhất quán của ảnh chụp tải/tham số phát, không đại diện cho mọi profile hay nghiệm thu giải mã trên năm đầu |
| Phụ đề | Đã nối VTT, SRT chuyển thành VTT UTF-8 | Chuyển đổi theo định dạng của rãnh, hỗ trợ BOM hoặc charset của phản hồi; host thật bao quát siêu thông tin nhiều rãnh, Range/HEAD, cách ly phụ đề hỏng; chuyển đổi một rãnh từ nguồn thật đã được kiểm chứng, hiển thị/chuyển phụ đề trên thiết bị chờ nghiệm thu, ASS chưa khai báo |
| Luồng phát bên ngoài | Đã nối mở có xác nhận | VOD, trực tiếp, đồ văn/luồng thông tin hiển thị đích và do người dùng xác nhận rồi giao cho hệ thống; HTTP(S), magnet, ed2k, rtsp, rtmp; kiểm chứng host thật và hồi quy không có phiên media đạt, chương trình xử lý của nền tảng và tương tác thiết bị chờ nghiệm thu |
| Làm mới chia sẻ của host | Đã nối | Cùng phiên/phiên bản dùng chung làm mới; sau khi thành công thay thế tài nguyên nguyên tử, địa chỉ của phiên bản cũ mất hiệu lực; hủy một bên chờ không ảnh hưởng các bên tiêu thụ khác; giữ vị trí phát; trải nghiệm khôi phục trên thiết bị chờ nghiệm thu |
| Điều độ | API và lõi media đã triển khai | GET ẩn danh được hợp nhất theo plugin và yêu cầu hiệu lực của mỗi bước nhảy, hủy bỏ/timeout độc lập; API mỗi plugin 6, cùng host 4, media/hình ảnh 6/2, giấy phép bao phủ thân phản hồi; việc phát hiện tại ngắt tải trước và ngăn tải trước mới chen hàng. Hồi quy đồng thời HTTP thật và ưu tiên media đạt; tải xuống chưa được hỗ trợ |
| Bộ nhớ HTML | Số lượng/đầu vào có giới hạn | Chưa hợp nhất tính vào ngân sách heap 64 MiB |
| Nhật ký và gỡ bỏ | Lõi đã triển khai | Mọi lối vào nhật ký dùng chung giới hạn tốc độ/byte/vòng, khi đọc hiển thị số lượng bị loại bỏ; ngữ cảnh khỏe mạnh dispose đồng bộ tối đa 1 giây, ném lỗi/timeout không ngăn việc ngừng hoạt động, chuyên đề host thật đạt |
| Tự kiểm tra/xuất nhật ký/dung lượng và dọn dẹp | Còn thiếu | Đọc nhật ký khả dụng, xuất tệp chờ được nối vào |
| Danh sách tương đối của Android | Đã nối ủy quyền thư mục, thiết bị chờ nghiệm thu | Sau khi chọn thư mục hệ thống thì phân giải đường dẫn con theo tên; kiểm thử đơn vị đường dẫn và hồi quy Rust về chọn cài, định danh, hủy bỏ; ủy quyền đơn tệp không mở rộng tới các tệp anh em |
| Đăng nhập WebView | Windows, Android được hỗ trợ đã nối | Người dùng đã kiểm chứng đăng nhập trang thật ở cả hai đầu. Windows sau khi đăng nhập có thể bấm "Đóng" ở đầu cửa sổ để quay lại plugin; trình giả lập Android sau khi cài đặt lại mà giữ dữ liệu vẫn hiển thị được biệt danh tài khoản. Hai đầu còn có kiểm thử cách ly cùng origin dạng native; biểu mẫu xác thực, hủy bỏ và tạm dừng ngân sách có bao quát hồi quy ngoại tuyến |
| DASH/ASS | Tùy chọn chưa được hỗ trợ | Không khai báo các đặc tính này |
| Xuyên đầu/nền | Chưa hoàn thành | Trình duyệt Windows, việc nhập trên Android và phát một MP4 thật có bằng chứng |

Các kiểm thử native của Windows và Android đều bao quát Cookie và localStorage cùng origin. API 35 / WebView 110 trong tiến trình đăng nhập độc lập sử dụng thư mục dữ liệu mới, đã đạt kiểm thử cách ly với khung xem chính và hai lần đăng nhập liên tiếp. Client khai báo `webview` theo khả năng native. Bộ chuyển đổi hiện tại yêu cầu Windows WebView2 88 trở lên, hoặc Android 28 trở lên và hỗ trợ `GET_COOKIE_INFO`.

Chi tiết xem ở [Thuyết minh triển khai](../plugin-protocol/IMPLEMENTATION.md).
