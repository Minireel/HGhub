# 09 · Phiên media và truyền tải

[← Tổng quan](README.md)

Chương này định nghĩa host biến các tham chiếu tài nguyên của [05](05-content-model.md) thành hình ảnh, phụ đề và media có thể phát như thế nào.
Mô hình dữ liệu mô tả "tài nguyên là gì", chương này thỏa thuận "do ai yêu cầu, truyền tải thế nào, làm mới khi nào". Các cơ chế này do host thống nhất triển khai.

## 1. Phiên và thuộc tính tài nguyên

Khi host nhận được `ImageRef`, `Stream` hoặc rãnh phụ đề, nó đăng ký chúng thành tài nguyên trong phiên media, và trả về địa chỉ cục bộ mà giao diện dùng trực tiếp được.
Cách truyền tải cụ thể của địa chỉ cục bộ do triển khai nền tảng quyết định, có thể dùng giao thức ứng dụng hoặc dịch vụ HTTP cục bộ; phải qua kiểm chứng bằng trình phát thực tế,
không thể giả định trình phát trên năm đầu đều đọc được cùng một loại giao thức tùy chỉnh. ID phiên là tham chiếu mờ, không ghép header yêu cầu và khóa vào địa chỉ.

Mỗi tài nguyên ghi lại các thông tin nội bộ sau, plugin không cần điền:

- Nhà cung cấp tài nguyên: ID plugin của nguồn dữ liệu hoặc resolver trả về tài nguyên đó, bản sửa đổi mã và ảnh chụp tuyến trang.
- Địa chỉ gốc, địa chỉ cuối cùng, header yêu cầu rõ ràng, header mặc định của nhà cung cấp, thuộc tính kho Cookie và phép biến đổi byte.
- Phiên phát còn lưu tác phẩm/kênh, phân tập, danh sách phát, lựa chọn tuyến của từng giai đoạn và chuỗi phân giải, dùng cho việc làm mới tái sử dụng.

Hình ảnh, phụ đề, luồng trực tiếp do nguồn dữ liệu trả về thuộc về nguồn dữ liệu; luồng, phụ đề do resolver trả về thuộc về resolver.
Tài nguyên con mà danh sách HLS/DASH tham chiếu kế thừa nhà cung cấp và cấu hình biến đổi của danh sách, không vì tên miền CDN thay đổi mà thay đổi nhà cung cấp.
Phép biến đổi plugin luôn gọi `transformResource` của nhà cung cấp tài nguyên sở thuộc, không thể vì lối vào đến từ nguồn dữ liệu mà gọi nhầm plugin.

Môi trường yêu cầu của nhà cung cấp được hợp nhất theo [05 §7.4](05-content-model.md), Cookie tự động khớp theo URL đích.
`Cookie`, `Authorization` rõ ràng chỉ giữ lại trong yêu cầu con hoặc chuyển hướng cùng origin với URL tài nguyên gốc; khi nhảy sang origin khác thì loại bỏ,
không thể trong các bước nhảy sau lại từ header mặc định thêm trở lại. Các header khác tiếp tục xử lý theo quy tắc truyền tải. Set-Cookie của yêu cầu tài nguyên ghi trở lại kho của nhà cung cấp.

Việc phiên giữ mô tả tài nguyên không đồng nghĩa JS runtime thường trú. Header yêu cầu tĩnh, proxy và phép biến đổi AES native không cần tải plugin;
khi cần biến đổi hoặc làm mới bằng plugin thì host theo nhu cầu khôi phục runtime. Sau khi phiên được giải phóng, hủy các nhiệm vụ của nó và thu hồi tham chiếu tài nguyên; tài nguyên dùng chung vẫn còn người dùng khác thì giữ lại.

## 2. Hợp nhất kết quả resolver

1. Sau khi `resolvePlayback` hoặc `live.resolve` của nguồn dữ liệu trả về luồng `page`, host chọn định một resolver.
2. Host giao địa chỉ trang và các header yêu cầu không phải thông tin xác thực hiệu lực cho resolver; không chuyển giao kho Cookie, Authorization, Cookie rõ ràng hay lưu trữ của nguồn.
   Các header như Referer truyền vào dùng cho resolver, không tự động trở thành header mặc định cho mọi yêu cầu media tiếp theo của nó.
3. Luồng resolver trả về thay thế ứng viên page đã chọn hiện tại; các luồng nguồn khác vẫn có thể làm ứng viên. Resolver không được tiếp tục trả về page.
4. Phụ đề của nguồn dữ liệu và phụ đề của resolver được hợp nhất, mỗi cái giữ thuộc tính của mình; các rãnh có cùng URL, cùng nhà cung cấp và cấu hình yêu cầu hiệu lực giống nhau có thể khử trùng.
   Khi có nhiều default thì ưu tiên default đầu tiên của resolver hiện tại, nếu không dùng default đầu tiên của nguồn.
5. durationSec, resumeSec, skip do resolver hiện tại trả về được ưu tiên, không cho thì dùng tiếp giá trị của nguồn; tiến độ cục bộ luôn ưu tiên hơn resumeSec.
   Thời hạn hiệu lực lấy giá trị sớm nhất trong các expiresAt đã khai báo của từng giai đoạn trong chuỗi hiện tại, không trộn thời hạn hiệu lực của các luồng ứng viên chưa được chọn khác vào.

ID của tuyến nguồn và tuyến resolver không thuộc cùng một namespace. Host nội bộ dùng "giai đoạn + ID plugin + serverId gốc" ghi lại lựa chọn,
giao diện hiển thị nhãn riêng của tuyến nguồn và tuyến resolver; khi gọi chỉ truyền ID gốc cho giai đoạn tương ứng, không truyền serverId của resolver cho nguồn dữ liệu.
Khi resolver chỉ định không khả dụng thì gợi ý rõ ràng; sau khi người dùng chọn lại mới thay đổi chuỗi phân giải, không lặng lẽ đổi sang triển khai khác.

Một hành động phân giải phát dùng chung tổng ngân sách, mặc định 20 giây, giới hạn trên 60 giây, đặt theo ngân sách playback của nguồn dữ liệu lối vào.
Mỗi giai đoạn còn chịu ràng buộc ngân sách của chính nó, việc gọi resolver không thể đặt lại tổng đồng hồ; việc người dùng chủ động chọn resolver không chiếm ngân sách thực thi plugin,
nhưng rời trang sẽ hủy toàn bộ hành động. Quy tắc tạm dừng tương tác web vẫn theo [04 §12](04-host-api.md).

## 3. Media thông thường và HLS/DASH

Tài nguyên progressive không có biến đổi phải hỗ trợ proxy dạng truyền phát, không đọc toàn bộ video vào bộ nhớ:

- Theo nhu cầu của trình phát chuyển tiếp GET/HEAD và Range/If-Range, và giữ ngữ nghĩa 200/206/416 đúng.
- Chuyển tiếp hoặc tính lại đúng Content-Type, Content-Length, Content-Range, Accept-Ranges; không được cố định giả mạo thành 200 hoặc bytes.
- Khi thượng nguồn bỏ qua Range thì xử lý theo phản hồi thực tế của nó, không đánh dấu phản hồi đầy đủ thành phản hồi cục bộ; rời khỏi việc phát thì kịp thời hủy truyền tải thượng nguồn.

Proxy HLS phải xử lý danh sách chính và danh sách media, ít nhất bao quát danh sách phát con, danh sách âm thanh/phụ đề, phân đoạn, KEY/SESSION-KEY, MAP,
và PART/PRELOAD-HINT khi hỗ trợ phát độ trễ thấp. Mọi URI lấy địa chỉ cuối cùng sau khi danh sách đó chuyển hướng để phân giải,
rồi viết lại thành địa chỉ tài nguyên phiên; giữ nguyên tham số truy vấn gốc, không thể tự đoán chữ ký của danh sách cha thêm vào mọi tài nguyên con.
Giữ các ngữ nghĩa ảnh hưởng việc phát như BYTERANGE, MEDIA-SEQUENCE, DISCONTINUITY. Mỗi tài nguyên con tiếp tục thực thi quy tắc mạng.

AES-128 HLS chuẩn xử lý theo hỗ trợ của trình phát, host chịu trách nhiệm proxy khóa; đây là việc khác với phép biến đổi byte tùy chỉnh trước khi truyền tải tài nguyên, không thể giải mã trùng.
Thứ tự xử lý danh sách là: lấy byte → biến đổi playlist cần thiết → theo URL cuối cùng phân giải và viết lại URI → giao cho trình phát.
Đoạn khởi tạo MAP xử lý theo cấu hình segment, KEY xử lý theo cấu hình key.

DASH là đặc tính tùy chọn `format.dash`: chỉ khi trình phát, việc định địa chỉ tài nguyên con MPD và proxy của host đều khả dụng mới khai báo.
Triển khai phải bao quát BaseURL, đoạn khởi tạo, phân đoạn media và yêu cầu theo khoảng của các hình thức MPD được chấp nhận; khi không xử lý được thì báo UNSUPPORTED,
không thể sau khi khai báo hỗ trợ lại để tài nguyên con mang header chuyên dụng cho trình phát yêu cầu xuyên miền trực tiếp. Khả năng rãnh âm thanh lấy danh sách mà thiết bị thực tế phát được làm chuẩn.

## 4. Biến đổi byte và yêu cầu theo khoảng

Phép biến đổi của 1.0 lấy tài nguyên đầy đủ làm đơn vị, cả biến đổi AES native và plugin đều không được tác dụng trực tiếp lên byte bản mã sau khi cắt tùy ý.
Tài nguyên có thể biến đổi là hình ảnh, danh sách phát, khóa, phân đoạn media và đoạn khởi tạo có giới hạn; không hỗ trợ biến đổi toàn bộ video progressive.
Tham số đầy đủ và kết quả của phép biến đổi plugin lần lượt chịu ràng buộc giới hạn trên 8 MiB, biến đổi native cũng phải đặt ngân sách đọc tài nguyên có giới hạn.

Khi cần yêu cầu byte cục bộ của tài nguyên sau biến đổi, host trước hết lấy tài nguyên đầy đủ chờ biến đổi, thực thi biến đổi, rồi cắt trên đầu ra và tính header phản hồi theo khoảng.
Khóa bộ nhớ đệm phải phân biệt bản sửa đổi mã của nhà cung cấp, môi trường yêu cầu, name/state/key/iv của phép biến đổi và tài nguyên đầu vào, không thể chỉ đệm theo URL.
Khi vượt ngân sách hoặc không xác định được biên giới biến đổi đầy đủ thì báo lỗi rõ ràng, không lặng lẽ đổi thành giải mã cục bộ trên bản mã.

Đối với HLS BYTERANGE, khoảng byte thượng nguồn mà mục đó chỉ định chính là biên giới tài nguyên chờ biến đổi: trước hết lấy khoảng đầy đủ, rồi biến đổi.
Chỉ khi chính khoảng đó có thể giải mã độc lập mới có thể dùng phép biến đổi; 1.0 không hỗ trợ thuật toán chia sẻ trạng thái mật mã qua nhiều khoảng.
Địa chỉ cục bộ phải phân biệt các khoảng này, và loại bỏ hoặc viết lại BYTERANGE gốc trong danh sách, khiến trình phát không áp thêm một lần độ lệch bản mã lên đầu ra đã giải mã.
Khi không làm được thì trả về UNSUPPORTED.

## 5. Thời hạn hiệu lực và khôi phục phát

- Đối với phiên phát/tải xuống đang hoạt động, trước khi expiresAt sớm nhất tới hạn thì làm mới theo chính sách host; khi không khai báo thời hạn hiệu lực, các thất bại lấy luồng có thể khôi phục như 401/403,
  địa chỉ mất hiệu lực cũng có thể kích hoạt một lần làm mới. Hủy bỏ, nội dung bị gỡ rõ ràng và mạng ngoại tuyến không kích hoạt làm mới vô hạn.
- Việc làm mới bắt đầu từ nguồn dữ liệu lối vào, với reason: "refresh" gọi `media.resolvePlayback` hoặc `live.resolve`,
  rồi theo địa chỉ page mới thực thi resolver đã chọn, resolver cũng nhận reason: "refresh". Cả hai giai đoạn đều không được trả về bộ nhớ đệm đã biết là hết hạn.
  Khi việc chỉ định resolver hoặc thay đổi host khiến định tuyến không còn áp dụng thì chọn lại; khi cần người dùng lựa chọn thì tạm dừng phát và hiển thị lối vào.
- Cùng một phiên chỉ chạy một nhiệm vụ làm mới, các yêu cầu khác chờ kết quả của nó. Sau khi chuỗi mới thành công, thay thế nguyên tử ánh xạ tài nguyên trong phiên,
  yêu cầu tài nguyên cũ đã mất hiệu lực không tiếp tục thử lại vô hạn; khi trình phát cần tải lại thì lưu vị trí phát hiện tại rồi khôi phục.
- VOD lấy vị trí cục bộ hiện tại làm chuẩn, không lặp lại việc dùng resumeSec của thượng nguồn ghi đè; trực tiếp khôi phục tới vị trí trực tiếp khả dụng.
- Một vòng sự cố tối đa tự động làm mới một lần; vẫn thất bại thì theo cấu hình thử các tuyến phát khác hoặc gợi ý người dùng thử lại/đổi nguồn.
  Các tuyến ứng viên khác nhau mỗi vòng thử tối đa một lần, AUTH_REQUIRED, AUTH_EXPIRED, CONTENT_LOCKED không tự động luân phiên thử tuyến.
  Việc tải xuống chỉ tiếp tục truyền khi tính liên tục của container và khoảng cho phép, nếu không khởi động lại tệp hiện tại hoặc gợi ý thất bại, không thể ghép các byte không tương thích.

## 6. Điều độ và vòng đời

API, phát tương tác và tải xuống lần lượt xếp hàng; việc phát hiện tại được ưu tiên, hình ảnh không thể chiếm đầy hàng đợi phân đoạn, tải trước và tải xuống nhường cho phát tiền cảnh.
Tốc độ, đồng thời, kích thước bộ nhớ đệm cụ thể thuộc về chính sách host; việc làm nguội Retry-After của host và quy tắc mạng áp dụng cho toàn bộ hàng đợi.
Việc hủy bỏ một bên chờ của tài nguyên dùng chung không nên ngắt việc truyền tải mà các phiên khác vẫn dùng.

Sau khi gỡ bỏ vì nhàn rỗi thông thường, phiên vẫn có thể dùng cùng bản sửa đổi plugin khôi phục biến đổi và làm mới. Khi vô hiệu hóa, gỡ bỏ, cập nhật plugin, hoặc sửa cài đặt, tuyến trang của nó,
host trước hết dừng các phiên phụ thuộc vào nó và gợi ý phát lại, rồi mới hủy lần gọi, thay thế mã/môi trường yêu cầu và xây dựng lại;
không lặng lẽ áp dụng môi trường plugin mới cho phiên cũ. Việc kiểm tra cập nhật, xem nhật ký và xóa bộ nhớ đệm tự nó không dừng việc phát.
Lịch sử, yêu thích và tiến độ của từng nguồn được lưu độc lập, không vì phiên kết thúc mà bị xóa.

## 7. Nghiệm thu xuyên đầu và hành vi nền

"Cùng một JS runtime" chỉ đảm bảo ngữ nghĩa ngôn ngữ và API nhất quán, không đảm bảo mọi thiết bị giải mã cùng một loại media.
`hg.env.media` phải phản ánh khả năng mà đường đi phát hiện tại khả dụng; không thể chỉ dựa vào tên bộ giải mã hệ thống tuyên bố WebView phát được.
HLS và progressive phát được là đường đi phát cốt lõi, DASH, kết xuất ASS, phát nền, tải xuống nền xử lý theo hỗ trợ thực tế của nền tảng.

Ít nhất kiểm chứng: hình ảnh có header, media có Cookie, danh sách HLS nhiều cấp và khóa, kéo MP4, phụ đề, làm mới địa chỉ, hủy bỏ,
và DASH/biến đổi phân đoạn thật sự đã khai báo. Mỗi nền tảng đích đều phải kiểm chứng chuỗi đầy đủ từ địa chỉ phiên cục bộ tới trình phát thực tế.

Khi ứng dụng vào nền, host có thể trong phạm vi nền tảng cho phép giữ phiên media đang hoạt động, theo nhu cầu gọi plugin làm mới, biến đổi hoặc báo cáo tiến độ.
Khi không có nhiệm vụ media thì không chạy plugin; sau khi hệ thống thật sự treo tiến trình không cam kết JS, mạng hay bộ hẹn giờ tiếp tục thực thi.
Khi khôi phục, xây dựng lại runtime cần thiết, kiểm tra thời hạn hiệu lực của phiên rồi mới phát tiếp, không lấy việc thực thi nền liên tục làm tiền đề giao thức.
