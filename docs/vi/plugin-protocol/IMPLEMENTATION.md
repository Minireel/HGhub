# Tiến độ triển khai và phạm vi kiểm chứng

Trang này theo dõi phiên bản phát triển của nhánh `main`. Phiên bản trong manifest của ứng dụng là **0.0.5**, giao thức runtime là `1.0.0-dev`. HPP 1.0 vẫn ở giai đoạn dự thảo, trạng thái triển khai và phạm vi kiểm chứng của từng khả năng được liệt kê dưới đây.

Giai đoạn này lấy Windows làm chính. Android đã kiểm chứng nhập đơn tệp, phát một MP4 thật và đăng nhập web.

Đây là một trình phát plugin hoàn toàn mới, giữ phong cách frontend hiện có của HGhub. Không đọc, không di trú tài khoản, mục yêu thích hay lịch sử cũ; ứng dụng dùng định danh độc lập `com.hghub.player`, lần khởi động đầu tiên thư viện trống. API của trang cũ, tuyến, đăng nhập và mã giải mã media cũ đã bị loại bỏ.

## Đã triển khai

Bổ nghiệm phụ đề 2026-11-20: QuickJS thật và proxy loopback đã kiểm chứng SRT chuyển thành VTT UTF-8, siêu thông tin rãnh, Range/HEAD sau chuyển đổi, cách ly phụ đề hỏng và giải phóng phiên. Một SRT từ nguồn thật qua proxy trả về 200 / text/vtt, 1778 cue, 112299 byte; điều này không thay thế nghiệm thu hiển thị và chuyển phụ đề trên thiết bị. Mã hóa được xử lý theo thứ tự BOM, charset của phản hồi, UTF-8, ASS bị bỏ qua; nhiều phụ đề mặc định thì ưu tiên phụ đề đầu tiên của resolver hiện tại, nếu không thì phụ đề đầu tiên của nguồn.

| Lớp | Triển khai |
| --- | --- |
| Host độc lập | `src-tauri/crates/hpp`, không phụ thuộc API, thông tin xác thực, logic giải mã media của nền tảng video ngắn gốc |
| Quản lý plugin | Chọn nhiều tệp native, dán, URL và chọn nhập từ manifest cục bộ/từ xa; hiển thị giới thiệu và tên danh sách nguồn; kiểm chứng siêu thông tin và phiên bản, hạ cấp phải chọn rõ ràng, bật/tắt, thử lại, gỡ bỏ, khôi phục chỉ mục |
| Cập nhật plugin | Kiểm tra thủ công và chọn cài đặt; ưu tiên updateUrl, so sánh phiên bản theo số, lưu trữ bền vững bản ghi nguồn; xem trước không thay thế plugin hiện có, thay đổi bản sửa đổi sẽ từ chối cập nhật đã hết hạn; sau thất bại khôi phục tệp khả dụng trước đó, giữ lại cài đặt và lưu trữ |
| Khởi tạo | Định giá tầng cao nhất của module và kiểm tra siêu thông tin dùng chung ngân sách 5 giây; factory có thêm tổng ngân sách 5 giây; cả hai hỗ trợ Promise/microtask, giới hạn 1 giây mỗi đoạn đồng bộ và timeout khi treo; hủy khởi tạo sẽ dừng luồng; lần đầu lấy tuyến với endpoint rỗng, sau khi chọn tuyến mặc định sẽ xây dựng lại; thay đổi cài đặt và tuyến trang sẽ xây dựng lại |
| Thực thi | Mỗi plugin một QuickJS runtime, luồng riêng, heap 64 MiB; 8 lần gọi đồng thời, tổng 40 khe xếp hàng, ngân sách lần gọi và ngắt đồng bộ |
| Vòng đời | Thuộc tính ngữ cảnh của Promise/await, hủy lần gọi, hủy HTTP khi thiếu signal, dọn dẹp bộ hẹn giờ, gỡ bỏ khi nhàn rỗi, dừng sau lỗi liên tiếp và thử lại rõ ràng; khi gỡ bỏ ngữ cảnh đang chạy được, cố gắng gọi dispose đồng bộ, tối đa 1 giây, không chờ Promise, runtime lỗi thì bỏ qua |
| Nhật ký | Nhật ký plugin, cảnh báo kiểm chứng và lỗi runtime dùng chung giới hạn mỗi phút 240 dòng / 2000 dòng gần nhất / mỗi dòng 2 KiB JSON UTF-8; số lượng bị loại bỏ do vượt tốc độ có thể thấy khi đọc nhật ký; details quá dài bị cắt, không vượt qua giới hạn văn bản |
| Cầu dữ liệu | Giá trị truyền được có nhãn đầy đủ; hỗ trợ Uint8Array, không xung đột với đối tượng thông thường; độ sâu, ngân sách byte và kiểm chứng kết quả |
| Lưu trữ bền vững | Lưu trữ khóa–giá trị cách ly theo ID plugin, nhị phân, TTL, ghi xuống đĩa nguyên tử, giới hạn dung lượng; kho Cookie độc lập và khớp domain/path |
| HTTP | Kiểm tra DNS/chuyển hướng, tái sử dụng chính sách mạng hiện có của ứng dụng; khi cùng nhóm DNS có IPv4 được phép thì bỏ qua IPv6 bị hạn chế, không kết nối địa chỉ bị hạn chế; hủy bỏ, giới hạn yêu cầu/phản hồi, charset, giới hạn tốc độ, làm nguội Retry-After; thử lại GET/HEAD và bộ nhớ đệm phản hồi GET ẩn danh có giới hạn; tái sử dụng nhóm kết nối theo plugin, trang nguồn và chính sách proxy/bảo mật, định kỳ kiểm chứng lại DNS |
| Điều độ mạng | API mỗi plugin đồng thời 6, cùng host 4, giấy phép bao phủ thân phản hồi; GET ẩn danh của cùng plugin được hợp nhất theo địa chỉ thực tế và header hiệu lực của mỗi bước nhảy, timeout/hủy độc lập, sau khi mọi bên chờ thoát thì dừng thượng nguồn; media không chiếm hàng đợi API và hạn ngạch tốc độ, làm nguội host được chia sẻ xuyên plugin/hàng đợi |
| Công cụ native | URL, URLSearchParams, mã hóa/giải mã, tóm tắt/HMAC, AES CBC/ECB/CTR/GCM, mã hóa khóa công khai RSA (SPKI/PKCS#1, PKCS1/OAEP-SHA1/OAEP-SHA256, 1024–8192 bit), số ngẫu nhiên, bộ chọn HTML/CSS native |
| Media | Dịch vụ HTTP loopback cổng ngẫu nhiên, proxy truyền phát progressive và Range/HEAD/206/416 thật, viết lại danh sách chính/phụ HLS và thuộc tính URI |
| Thuộc tính tài nguyên | Khớp/lựa chọn resolver; Cookie, header yêu cầu và phép biến đổi độc lập của resolver; phụ đề của nguồn và resolver giữ nhà cung cấp của riêng mình; cùng một hành động phân giải dùng chung ngân sách |
| Phụ đề SRT | Đọc theo format của rãnh và chuyển thành VTT UTF-8, không phụ thuộc đuôi tệp hay MIME thượng nguồn; giữ ngôn ngữ, nhãn, mục mặc định và môi trường yêu cầu của nhà cung cấp; đầu vào/đầu ra mỗi cái giới hạn 8 MiB, Range/HEAD tính trên nội dung sau chuyển đổi |
| Luồng bên ngoài | Sau khi nguồn/resolver trả về external, host sinh mục tiêu xác nhận; VOD và trình phát dùng chung hiển thị địa chỉ, bấm xác nhận mới giao hệ thống mở; hủy bỏ, hết hạn và mở bằng hệ thống thất bại đều có thể thấy, không truyền header/Cookie/phép biến đổi, không thiết lập phiên proxy; tải trước không kích hoạt mở bên ngoài |
| Biến đổi byte | AES-CBC native cho hình ảnh/danh sách/phân đoạn/khóa có giới hạn, plugin transformResource; tính phạm vi cục bộ trên đầu ra |
| Phiên | Phiên lô hình ảnh và phiên phát; giải phóng sẽ hủy truyền tải, giải phóng khi plugin phụ thuộc bị vô hiệu hóa/gỡ bỏ/cập nhật/thay đổi cài đặt; kiểm chứng bản sửa đổi ngăn phiên cũ dùng môi trường mới |
| Trang Windows | Vỏ trình phát nhập vai cổ điển; trang chủ phát trực tiếp, phạm vi nguồn phát chọn trong cài đặt; danh sách đề xuất giữ thứ tự và ẩn hiện của nguồn/phân khu; điều hướng danh sách đầy đủ của phân khu; cây danh mục đệm kết quả thành công theo bản sửa đổi của plugin, lọc và sắp xếp; phân trang cuộn cho đề xuất/danh mục/tìm kiếm/phân khu, thất bại giữ nội dung và thử lại với con trỏ gốc, khử trùng ID và kết thúc khi con trỏ trùng; tìm kiếm đơn nguồn/đa nguồn/tổng hợp, lựa chọn nguồn; quản lý plugin, cài đặt, tuyến, nhật ký |
| Tuyến trang | Làm mới thủ công, thăm dò khả năng tới được và chuyển đổi; sau khi làm mới giữ tuyến vẫn còn tồn tại, nếu không chọn mục đầu tiên và xây dựng lại; thăm dò không mang header mặc định hay Cookie của plugin, không tự động chuyển |
| Thư viện cục bộ | Rust ghi xuống đĩa nguyên tử, yêu thích/theo dõi/lịch sử cách ly theo plugin và tác phẩm; tiến độ còn gồm danh sách phát và ID tập thật; lưu khi rời đi/tạm dừng/kết thúc, chọn lại tập để phát tiếp; sau khi gỡ plugin, bản ghi vẫn được giữ |
| Chi tiết và phát | Nhóm theo mùa, khi season/episodeNumber rõ ràng là duy nhất thì định vị ứng viên, không dùng index để đoán; phân trang phân tập, làm mới, phát HLS/video thông thường; bản sửa đổi ghi đè cùng phiên bản làm trang và phiên mất hiệu lực |
| Đổi nguồn xuyên nguồn | Trang chi tiết và trình phát dùng chung lối vào đổi nguồn; tìm kiếm song song các nguồn khả dụng khác, sắp xếp theo tiêu đề/năm/loại, thử lại khi thất bại độc lập; chi tiết ứng viên, phân trang phân tập và người dùng xác nhận; chỉ khi khớp mùa/tập rõ ràng là duy nhất mới chọn trước, việc dùng lại số giây cũ mặc định tắt, tiến độ của nguồn mới được ưu tiên; liên kết nguồn được lưu nguyên tử, trong "Của tôi" có thể xem nguồn liên kết, yêu thích/theo dõi/lịch sử cũ được giữ độc lập |
| Phát trực tiếp | Nguồn thuần trực tiếp có thể chỉ triển khai `live`; lối vào trang chủ hiển thị nhóm kênh, phân trang kênh, biểu tượng kênh, danh sách chương trình tải theo nhu cầu; phân giải trực tiếp tái sử dụng HLS/luồng thông thường, lựa chọn tuyến/luồng và phiên media, lần gọi chỉ truyền channelId |
| Luồng thông tin nhập vai | Trang danh mục hiển thị lối vào feed theo phương thức cụ thể; trang chủ dùng chung phát nhập vai; phát dọc từng mục, nút mục trước/sau và phím mũi tên, phân trang con trỏ và khử trùng ID; dùng trực tiếp entryEpisodeId, khi thiếu thì lấy chi tiết và lô phân tập đầu tiên cần thiết; yêu thích cục bộ, bình luận và tương tác đám mây; lưu tiến độ theo tập thật, tự động mục tiếp theo tuân theo tùy chọn; khi trang che phủ hoặc rời đi thì giải phóng việc phát |
| Đồ văn | Danh sách và chi tiết độc lập `articles.list/getDetail`; nội dung chính được xây dựng lại theo thẻ cho phép, hình ảnh đi qua phiên media của plugin sở hữu, liên kết ngoài dùng trình duyệt bên ngoài; hỗ trợ video trong bài, tác phẩm VOD liên kết, bình luận và tương tác |
| Của tôi | Thông tin tài khoản plugin, phân trang yêu thích/lượt thích/lịch sử đám mây và nhập cục bộ tại trang này; kiểm tra theo dõi cập nhật thủ công theo lô mỗi lô ≤50, bỏ qua các mục không thuộc lô này |
| Danh mục đám mây và theo dõi | Danh mục yêu thích đám mây yêu cầu theo folderId gốc, danh sách theo dõi phân trang con trỏ độc lập, ảnh đại diện đi qua proxy media của plugin sở hữu; chỉ khi triển khai interact.follow mới hiển thị bỏ theo dõi, thành công thì loại bỏ tác giả tương ứng |
| Bình luận và trả lời | Trang chi tiết mở theo nhu cầu bình luận của tác phẩm/tập thật hiện tại, trả lời dùng replies/commentId; phân trang độc lập và khử trùng ID, văn bản thuần, proxy ảnh đại diện, lượt thích bình luận; gửi mà có trả về bình luận thì chèn vào, không có thì làm mới; thất bại giữ bản nháp, rời trang hủy lần gọi chưa hoàn thành; mục tiêu/thao tác không được hỗ trợ sẽ ẩn độc lập |
| Tương tác đám mây | Trang chi tiết hiển thị theo phương thức cụ thể lượt thích/yêu thích/theo dõi tác giả đám mây, thành công thì cập nhật trạng thái; yêu thích cục bộ được giữ riêng; xác thực mất hiệu lực sẽ xóa trạng thái tương tác và dùng hướng dẫn đăng nhập của plugin sở hữu |
| Báo cáo tiến độ thượng nguồn | playing/pause/ended thực tế sinh ra start/pause/end; một người tiêu thụ phát đơn lẻ theo đồng hồ đơn điệu mỗi 30 giây tối đa một heartbeat; làm mới/chuyển luồng không lặp lại start, kèm episodeId thật và số giây hữu hạn; thất bại im lặng, lỗi xác thực thì xóa bộ nhớ đệm tài khoản, không bật lên gợi ý phát |
| Biểu mẫu plugin | Kiểm tra phương thức bắt buộc của `auth`/`ui`, kiểm chứng trạng thái/trường/thông tin trả về; trong bảng cài đặt và nhật ký của plugin hiển thị đăng nhập, tài khoản, đăng xuất và thao tác tùy chỉnh; tám loại trường, hợp nhất giá trị nhiều bước, cập nhật thử thách ẩn, kiểm chứng bắt buộc, proxy media cho mã xác minh/ảnh đại diện |
| Trạng thái đăng nhập | Plugin có thể lưu token, host lưu phiên đăng nhập web theo plugin và origin HTTPS. Nhập liệu biểu mẫu chỉ giữ trong bộ nhớ. AUTH_INVALID giữ lại giá trị đã điền, AUTH_REQUIRED/EXPIRED xóa bộ nhớ đệm tài khoản của trang và hướng dẫn đăng nhập. Sau khi đăng xuất thất bại, truy vấn lại thông tin plugin |
| Khôi phục phát | Trang yêu cầu host làm mới chia sẻ theo thời gian hết hạn/lỗi phát chí mạng; cùng một phiên bản của phiên chỉ chạy một chuỗi phân giải, giữ lựa chọn nguồn/resolver và vị trí phát hiện tại; thành công thì thay thế tài nguyên nguyên tử, địa chỉ cũ trả về mất hiệu lực; một bên chờ hủy không ảnh hưởng lẫn nhau, toàn bộ bên chờ hủy hoặc phiên được giải phóng sẽ dừng làm mới; thất bại giữ ánh xạ cũ, hỗ trợ thử lại thủ công |
| Chuẩn bị phát nối tiếp | Sau khi việc phát hiện tại ổn định, chuẩn bị tập tiếp theo và trang phân tập cần thiết; dùng reason:preload hiện có, phiên chuẩn bị không đánh dấu đang phát; chuyển tập thủ công, tự động và bằng cử chỉ tiếp quản cùng một nhiệm vụ/phiên chuẩn bị; hết hạn hoặc thất bại quay về chuẩn bị bình thường |
| Tải trước media có giới hạn | Một nhiệm vụ ưu tiên thấp duy nhất; MP4 đầu 2 MiB và cuối 512 KiB; HLS tĩnh biến thể đầu tiên, rãnh âm thanh tương ứng, hai phân đoạn tại điểm phát tiếp và key/map; bộ nhớ đệm cách ly theo phiên, một phiên 16 MiB, giữ toàn cục 64 MiB, mất hiệu lực sau 120 giây |
| Lựa chọn phát | Điều khiển luồng ứng viên và tuyến phát riêng của nguồn/resolver; lưu serverId gốc theo giai đoạn và ID plugin; chuyển thủ công giữ số giây hiện tại, làm mới giữ lựa chọn; sau khi giải mã thất bại làm mới một lần rồi chuyển luồng ứng viên có giới hạn theo cài đặt |
| Tùy chọn trang của resolver | Host của trang phát do host trích xuất; VOD và trình phát dùng chung cung cấp lựa chọn resolver và tùy chọn "ghi nhớ" mặc định tắt; sau khi chọn thành công lưu nguyên tử, qua trang khác không tái sử dụng lựa chọn thủ công; resolver do luồng chỉ định được ưu tiên, ghi nhớ bị vô hiệu hóa/gỡ bỏ/không khớp không tham gia định tuyến tự động; trong cài đặt có thể quên từng mục hoặc xóa toàn bộ |
| Tài liệu phát triển | Kiểu dự thảo đầy đủ `src/plugins/protocol.ts`; host giả trong bộ nhớ có thể tiêm bộ chuyển đổi truyền tải/mã hóa/HTML `src/plugins/mockHost.ts`; nguồn hư cấu thuần và bộ chuyển đổi kiểm thử stdio với host thật |

Chuyên đề khởi tạo `tests/runtime_initialization.rs` bao quát Promise treo, thời hạn tổng 5 giây của microtask liên tục, factory bất đồng bộ vượt 1 giây nhưng mỗi đoạn không timeout, vòng lặp đồng bộ vô hạn và cấm I/O của factory. `runtime::initialization_tests` kiểm chứng sau khi hủy khởi tạo bị treo/vòng lặp đồng bộ/vòng lặp microtask, luồng chuyên dụng kịp thời giải phóng tài nguyên mạng của nó.

`tests/module_initialization.rs` cũng bao quát await tầng cao nhất của module bị treo, microtask liên tục và vòng lặp đồng bộ vô hạn một đoạn; nhiều đoạn đồng bộ ngắn cộng dồn vượt 1 giây có thể nhập bình thường, xem trước và chỉ cài đặt đều không gọi factory. Kiểm tra nhập sẽ không báo trước một Promise không có nhiệm vụ chờ thực thi thành lỗi engine.

Chuyên đề nhật ký và gỡ bỏ: `native::logs` kiểm tra tốc độ thống nhất, dung lượng vòng, giới hạn byte tuần tự hóa gồm details và số lượng bị loại bỏ; `tests/log_limits.rs` từ QuickJS thật đồng thời sinh ra console, kiểm chứng và lỗi nghiệp vụ, xác nhận dùng chung cùng một hạn mức. `runtime::lifecycle_tests` kiểm chứng dọn dẹp bình thường, cấm I/O khi gỡ bỏ, thông báo ném lỗi/vòng lặp vô hạn/Promise treo đều có thể hoàn thành việc ngừng hoạt động, runtime đã bị ngắt thì bỏ qua thông báo. Việc xuất tệp nhật ký vẫn chưa được nối vào giao diện.

Các phương thức hiện được phát hiện và điều độ là:

- `media.getDetail`、`media.resolvePlayback`、`media.listEpisodes`、`media.matchUrl`
- `resolver.resolve`
- `search.search`、`search.getFilters`、`search.suggest`、`search.trending`
- `home.getSections`、`home.listSection`、`catalog.getCategories`、`catalog.getFilters`、`catalog.list`
- `library.listMedia`、`library.listFolders`、`library.listFollowing`、`progress.report`、`updates.check`
- `comments.list`、`comments.send`、`comments.like`、`interact.like`、`interact.favorite`、`interact.follow`
- `feed.list`、`endpoints.list`、`transformResource`
- `live.getGroups`、`live.listChannels`、`live.resolve`、`live.getProgrammes`
- `articles.list`、`articles.getDetail`
- `auth.getMethods`、`auth.submit`、`auth.getProfile`、`auth.logout`
- `ui.getActions`、`ui.submit`

Kiểm tra bật hiện tại yêu cầu source triển khai toàn bộ phương thức bắt buộc của `media` hoặc `live`, resolver triển khai `resolver.resolve`.
Nguồn thuần trực tiếp không được khai báo các khả năng home/catalog/search/feed/library trả về MediaItem VOD mà không triển khai media.
Khi khai báo `auth` phải triển khai cả bốn phương thức, khi khai báo `ui` phải triển khai hai phương thức; namespace không đầy đủ sẽ khởi tạo thất bại.
Các phương thức trên đã được nối vào các quy trình người dùng tương ứng của chi tiết VOD, trình phát, luồng thông tin và "Của tôi".
Các đặc tính tùy chọn chưa triển khai vẫn chưa tuyên bố hỗ trợ.

Quản lý biểu mẫu tái sử dụng IPC `hpp_manage`: target của `form.actions` là `auth` hoặc `ui`;
target của `form.submit` là `{ kind: "auth" | "ui", id }`, values là đối tượng khóa–giá trị chuỗi,
cả hai đều dùng requestId do giao diện sinh trước, đi vào cùng phạm vi hủy bỏ/thời hạn như khả năng thông thường.
Một trường không hợp lệ duy nhất trong biểu mẫu làm cả biểu mẫu mất hiệu lực; các mục xấu trong danh sách hành động/cách đăng nhập bị loại bỏ độc lập.
Nhật ký kiểm chứng chỉ chứa đường dẫn trường và nguyên nhân, không chứa giá trị nhập vào. Các kiểm chứng dưới đây dùng trang ngoại tuyến và dấu hiệu tổng hợp.

- Windows đã kiểm chứng trong cửa sổ Tauri/WebView2 thật với ba thư mục dữ liệu độc lập. Ba cửa sổ đồng thời mở cùng một origin, lần lượt ghi Cookie và localStorage, rồi đọc lại giá trị của riêng mình. Có thể dùng `cargo run --manifest-path src-tauri/Cargo.toml --example webview2_login_isolation -- <thư mục tạm mới>` để kiểm tra lại, sau khi tiến trình thoát thì loại bỏ thư mục kiểm thử đó.
- Android API 35 / WebView 110.0.5481.154.1 đo thực tế `MULTI_PROFILE=false`, `GET_COOKIE_INFO=true`. `ProcessLoginIsolationTest` giữ khung xem chính mở, lần lượt liên kết hai tiến trình đăng nhập cùng gói khởi động lạnh, mỗi lần đặt hậu tố thư mục dữ liệu ngẫu nhiên trước khi tạo WebView. Hai phiên đều bắt đầu từ trạng thái rỗng, đều ghi được dấu hiệu của riêng mình. Khung xem chính sau hai lần đăng nhập kết thúc đều giữ giá trị gốc. Dịch vụ kiểm thử chỉ đi vào bản build debug. Kiểm thử này bao quát đăng nhập nối tiếp.
- Trình giả lập Android API 37 trước đó đã đạt kiểm thử cách ly cùng origin với khung xem mặc định và hai Profile tạm thời của AndroidX.

Lớp mạng Rust đã triển khai phiên do host quản lý được lưu theo plugin và origin HTTPS, bao quát tương thích kho thông thường, gửi thay cùng origin, ẩn header phản hồi, đăng xuất và đổi tài khoản trong lúc đang bay. Runtime Rust chỉ mở cửa sổ đăng nhập cho biểu mẫu xác thực do người dùng gửi. Kiểm thử ngoại tuyến QuickJS thông qua fixture bộ chuyển đổi native kiểm chứng khóa lẫn nhau đồng thời, tạm dừng và khôi phục ngân sách, hủy bỏ, timeout cửa sổ, kiểm chứng địa chỉ hoàn tất và trường trả về.

Client Windows dùng cửa sổ InPrivate WebView2 độc lập, mỗi lần đăng nhập tạo thư mục dữ liệu tạm thời. Android 28 trở lên và WebView hỗ trợ `GET_COOKIE_INFO` thì client dùng tiến trình độc lập, trước khi tạo WebView đặt hậu tố thư mục dữ liệu cho lần đăng nhập này. Hai đầu này khi bộ chuyển đổi native khả dụng đều khai báo `webview`. Cách đăng nhập không có `form` có thể do người dùng bấm để gửi trực tiếp.

Đăng nhập trang thật đã được người dùng kiểm chứng thủ công trên Windows và Android. Cửa sổ đăng nhập Windows chạy trong luồng STA độc lập, phía trên hiển thị trang hiện tại và nút "Đóng". Khi bấm nút đó hoặc dấu × trên thanh tiêu đề, host trước khi hủy khung xem sẽ đọc địa chỉ và Cookie hiện tại, theo `doneWhen` phán đoán hoàn tất hay hủy bỏ. Trình giả lập Android API 35 / WebView 110 sau khi cài đặt lại giữ dữ liệu, khu vực tài khoản plugin vẫn hiển thị được biệt danh tài khoản và "Đăng xuất" qua yêu cầu cùng origin.

Lỗi xác thực trên chuỗi phân giải phát được host ghi chú nhà cung cấp thực tế, giao diện dựa vào đó đi vào khu vực tài khoản của nguồn dữ liệu hoặc resolver;
không tin tưởng định danh plugin khác tự báo trong details của plugin, cũng không dùng lẫn trạng thái đăng nhập của resolver với của nguồn.

## Vẫn cần hoàn thành

1. **Hoàn thiện nền tảng phân phối**: Đăng ký, kiểm tra tự động khi khởi động, tùy chọn cập nhật tự động, làm nguội 24 giờ và hoãn cài đặt trong lúc phát đã triển khai và có hồi quy Rust. Việc đọc đơn tệp qua content URI của Android đã sửa và đo thực tế; danh sách tương đối đã nối vào ủy quyền thư mục hệ thống và phân giải từng tầng theo tên tệp, không ghép ID tài liệu của nhà cung cấp. Hồi quy Rust bao quát chỉ đọc plugin đã chọn, kiểm chứng định danh, cách ly thất bại của mục và hủy bỏ; kiểm thử đơn vị đường dẫn Android bao quát tiếng Trung, mã hóa, thư mục con và vượt biên. Việc nghiệm thu bộ chọn thư mục và các nhà cung cấp khác nhau trên thiết bị vẫn chờ hoàn thành, phạm vi sử dụng xem ở [Hướng dẫn phân phối](../developers/distribution.md).
2. **Các khả năng khác**: Mục yêu thích đám mây hiện dùng thư mục mặc định của thượng nguồn; lựa chọn thư mục trong tương tác chi tiết, lượt thích theo tập và liên động trạng thái tài khoản đầy đủ hơn sẽ hoàn thiện sau. Luồng thông tin tái sử dụng cử chỉ và bộ điều khiển của trình phát cổ điển, vẫn cần thêm nghiệm thu thiết bị.
3. **Hoàn thiện việc phát**: Tùy chọn độ nét mặc định, định vị lại chỉ mục ứng viên đã hết hạn, chuyển đổi khi máy chủ thất bại; xử lý định dạng phụ đề đầy đủ. Việc chuyển luồng ứng viên có giới hạn hiện tại chỉ xử lý lỗi chí mạng của trình phát, lỗi xác thực/khóa ở giai đoạn phân giải sẽ dừng và hướng dẫn người dùng xử lý.
4. **Nghiệm thu trải nghiệm làm mới**: Làm mới chia sẻ cùng phiên bản của phiên bởi host, thay thế tài nguyên nguyên tử và hủy bỏ độc lập đã triển khai; khôi phục nền và tính liên tục phát trên thiết bị vẫn cần nghiệm thu, các bên tiêu thụ tải xuống chưa triển khai không nằm trong phạm vi hỗ trợ.
5. **Ngân sách media và bộ nhớ đệm**: Việc phát và hình ảnh lần lượt giới hạn 6/2 lần truyền tải; yêu cầu media hiện tại hủy bỏ tải trước mang tính suy đoán, trong lúc xếp hàng hoặc truyền thân phản hồi thì bỏ qua tải trước mới, tải trước cũng chiếm giấy phép truyền tải media. Tập tiếp theo đã có bộ nhớ đệm có giới hạn. Bộ nhớ đệm bền vững xuyên phiên, phát trực tiếp dài và kiểm chứng HLS độ trễ thấp vẫn chưa hoàn thành; hàng đợi tải xuống cùng với khả năng tải xuống chưa được hỗ trợ được giữ làm khoảng trống tùy chọn.
6. **Biến đổi khoảng byte**: HLS BYTERANGE có biến đổi byte hiện trả về rõ ràng UNSUPPORTED; không giải mã trực tiếp bản mã bị cắt ngắn.
7. **API và chức năng tùy chọn**: API bắt buộc RSA đã triển khai, và đã qua hồi quy với QuickJS thật về hai loại PEM/ba loại đệm, tính ngẫu nhiên, biên bản rõ/khóa và requires; đăng nhập web đã nối vào Windows và các thiết bị Android được hỗ trợ, các nền tảng còn lại chưa khai báo `webview`. DASH và tải xuống là khả năng tùy chọn, hiện chưa triển khai. hg.env.media đã nối vào việc thăm dò API khả năng của đường đi phát WebView hiện tại; vẫn cần nghiệm thu media thực tế và khôi phục nền trên từng nền tảng.
8. **Chức năng ứng dụng**: Tự kiểm tra plugin, dọn dẹp và dung lượng dữ liệu, cập nhật ứng dụng. Hạng mục này không phát triển việc di trú dữ liệu nền tảng cũ.
9. **Nghiệm thu nền tảng**: Cửa sổ native Windows và hồi quy media đầy đủ trên năm đầu vẫn chưa hoàn thành; Android đã định vị và sửa lỗi nhập URI của bộ chọn tài liệu thất bại, việc dán và nhập tệp với APK mới đo thực tế thành công.

Bộ nhớ đệm phản hồi API hiện chỉ tiếp nhận GET có `credentials:"omit"` rõ ràng và không có Authorization/Cookie;
với các yêu cầu mặc định include nhưng lúc đó vừa không có Cookie, áp dụng cách xử lý bảo thủ là không trúng bộ nhớ đệm, không thay đổi ngữ nghĩa phản hồi. Việc hợp nhất đang bay không yêu cầu bật bộ nhớ đệm phản hồi: mặc định include và yêu cầu hiện tại không mang Cookie cũng có thể hợp nhất; sau khi chuyển hướng tới đích có Cookie, các bên chờ lần lượt yêu cầu riêng.

Chuyên đề mạng `network::scheduling_tests` dùng phản hồi HTTP chậm thật bao quát hợp nhất, hủy/timeout độc lập, cách ly xác thực và header hiệu lực, giới hạn đồng thời của thân phản hồi và làm nguội xuyên hàng đợi. `tests/media_scheduling.rs` kiểm chứng việc phát hiện tại ngắt tải trước, sau khi sáu thân phản hồi media chiếm đầy thì xếp hàng và khôi phục tải trước sau khi truyền tải kết thúc; `tests/playback_performance.rs` tiếp tục kiểm chứng trúng bộ nhớ đệm tải trước MP4/HLS.
Host giả không mô phỏng cách ly QuickJS, hạn ngạch đầy đủ hay ngữ cảnh lần gọi bất đồng bộ; những điều này phải dùng kiểm thử host Rust.

### Thăm dò khả năng phát (2026-11-16)

- WebView hoàn thành thăm dò trước lệnh plugin đầu tiên và đăng ký với host; mọi lệnh lô đầu dùng chung nhiệm vụ đăng ký, khi thất bại không tiếp tục gọi plugin bằng khả năng mặc định, thao tác tiếp theo có thể thử lại. Việc khởi động Tauri không còn khởi tạo plugin sớm bằng khả năng cố định.
- Video thông thường dùng `HTMLMediaElement.canPlayType`, chỉ chấp nhận `probably` của loại codec. Các codec MP4 đại diện là H.264 `avc1.42E01E`, HEVC `hvc1.1.6.L93.B0` / `hev1.1.6.L93.B0`, AV1 `av01.0.04M.08`; đồng thời kiểm tra AAC `mp4a.40.2`. Các container, rãnh âm thanh và profile khác không được cam kết toàn diện dựa trên điều này.
- HLS và VideoSurface dùng chung lựa chọn đường đi: ưu tiên đường đi MSE tương ứng với `Hls.isSupported()`, và dùng `Hls.getMediaSource().isTypeSupported` kiểm tra codec và AAC; bao gồm ManagedMediaSource do Hls.js chọn. Nếu không thì kiểm tra hỗ trợ MIME HLS native, và tái sử dụng thăm dò codec native. Host không có proxy DASH, vẫn không khai báo DASH.
- Một đường đi ít nhất phát hiện được một codec mới được liệt kê vào formats. Vì giao thức không có ma trận codec theo từng định dạng, codecs lấy giao thoa của các đường đi đã bật, sắp xếp theo H.264, HEVC, AV1; giao thoa có thể rỗng. Đây là tập khả dụng bảo thủ, không phải hợp của mọi bộ giải mã hệ thống. Sẽ không vì video native phát được HEVC mà tuyên bố đường đi Hls.js cũng phát được.
- Kết quả đăng ký chỉ giữ trong tiến trình host hiện tại. Factory của plugin lấy ảnh chụp chỉ đọc sâu; codecs của VOD, trực tiếp và resolver do host điền theo ảnh chụp runtime tương ứng, ghi đè giá trị cũ của bên gọi, phát bình thường/tải trước/làm mới nhất quán. Đăng ký trùng kết quả giống nhau không xây dựng lại, không ngắt phiên; khi kết quả thay đổi thì giải phóng phiên cũ, hủy runtime cũ và cập nhật bản sửa đổi, lần dùng tiếp theo khởi tạo lại. Dữ liệu người dùng, cài đặt plugin và lưu trữ không bị xóa.
- Host Rust/stdio không có frontend phát và host giả trong bộ nhớ mặc định báo cáo khả năng rỗng, kiểm thử cần tiêm rõ ràng. Việc thăm dò native/frontend và truyền ảnh chụp có hồi quy logic; không có việc tải thêm hay phát thử video thăm dò, không thể lấy API khả năng hay kết quả biên dịch làm bằng chứng giải mã thực tế trên năm đầu. Tùy chọn lựa chọn luồng và khôi phục lỗi vẫn xử lý theo quy trình hiện có.

### Tối ưu phát nối tiếp và phát lần đầu (2026-11-16)

- Phiên chuẩn bị giữ tối đa 90 giây và không vượt quá thời hạn hiệu lực của nguồn; gần kết thúc có thể cập nhật kết quả đã hết hạn. Bản sửa đổi plugin, lựa chọn tuyến/phân giải, thay đổi tài khoản, rời đi và vào nền sẽ hủy bỏ hoặc làm mất hiệu lực. Tải trước không bật lên giao diện xác thực hay lựa chọn, khi phát chính thức dùng xử lý lỗi bình thường.
- Tải trước media một lần tối đa 15 giây; sau khi bị việc phát hiện tại ngắt, thử tối đa 3 lần, khoảng cách ít nhất 10 giây, tài nguyên đã hoàn thành được tái sử dụng. Bộ nhớ đệm không chia sẻ xuyên phiên; sau khi chính sách proxy/bảo mật thay đổi sẽ không trúng bộ nhớ đệm cũ, giải phóng phiên hủy truyền tải.
- Nguồn MP4 bỏ qua Range, khi cần biến đổi toàn bộ tài nguyên thì bỏ qua tải trước theo byte. Chỉ khi ETag mạnh hoặc Last-Modified và kiểm chứng Content-Range đầy đủ đều đạt mới ghép tiền tố bộ nhớ đệm với hậu tố thượng nguồn, nếu không yêu cầu lại bình thường. Danh sách media động HLS không làm bộ nhớ đệm tĩnh, phân đoạn BYTERANGE không tải trước; không cam kết mọi tốc độ bit thích ứng đều trúng.
- Khi chuyển tập, phiên mới chuẩn bị xong mới giải phóng phiên cũ; việc đọc tiến độ cục bộ và chuẩn bị diễn ra song song, giữ nguyên phần tử video, HLS đặt startPosition phát tiếp, tài nguyên Worker rõ ràng và backBufferLength 30 giây. Tải trước không tạo trình phát thứ hai.
- `media.preload` / `media.preload.cancel` là IPC nội bộ của host, không thêm API plugin; `reason:preload` vốn có của giao thức không cần mở rộng. Chẩn đoán chỉ giữ tối đa 200 mục ID phiên, giai đoạn và thời gian, không ghi URL lấy luồng, Cookie hay token.
- Bằng chứng tự động: `scripts/test-next-episode.mjs` kiểm tra tái sử dụng nhiệm vụ, hết hạn và hủy bỏ chuẩn bị/tiếp quản/tải trước; `tests/playback_performance.rs` dùng dịch vụ HTTP thật cục bộ kiểm tra tái sử dụng kết nối, trúng bộ nhớ đệm MP4/HLS, tái sử dụng tải trước trùng lặp, quay về khi thay đổi phiên bản/bỏ qua Range và hủy bỏ. Người dùng đã phản hồi việc phát và phát nối tiếp trên Windows/Android mượt; không có so sánh thời gian khung hình đầu tiên trước/sau với trang thật, không đại diện nghiệm thu đầy đủ toàn nền tảng.
- Trạng thái đang tải được quản lý theo phiên hiện tại, khi chuẩn bị phiên mới không hiển thị trạng thái tạm dừng của phiên cũ; bỏ qua thất bại đến muộn của yêu cầu phát cũ và AbortError, việc từ chối quyền tự động phát thực tế vẫn cung cấp nút phát. `scripts/test-playback-state.mjs` bao quát cạnh tranh yêu cầu phát, tạm dừng rõ ràng, nhảy khi đang tạm dừng và hủy thành phần.

Yêu cầu phân phối dùng môi trường tải xuống của host độc lập, ẩn danh; danh sách từ xa sau khi chuyển hướng phân giải đường dẫn tương đối theo URL cuối cùng,
chỉ tải xuống các mục đã chọn, khi định danh tệp không khớp với danh sách thì chỉ mục đó thất bại. Trong lúc tải xuống, kiểm tra và chờ khóa cài đặt đều hỗ trợ hủy bỏ;
một lần cài đặt đơn lẻ mà việc thay thế tệp đã bắt đầu sẽ hoàn thành rồi mới dừng các mục tiếp theo, không hiểu việc hủy bỏ là khôi phục thao tác thượng nguồn hay ghi lưu trữ.
Bản xem trước còn lại sau khi rời trang quay lại sẽ được giải phóng ngay. Script gỡ lỗi riêng của nền tảng gốc chỉ giữ trong thư mục cục bộ bị bỏ qua,
lối vào kiểm thử công khai chỉ phụ thuộc nguồn hư cấu, không phụ thuộc plugin riêng tư.

## Khế ước thực thi được của lựa chọn phát

### Phạm vi và kích hoạt

Tương ứng 05 §7, 06 §2/3, 08 §7 và 09 §2/5. Tại đây ghi lại IPC của host, PlaybackParams của plugin không thêm trường resolver.

### Chữ ký

`hpp_manage("media.prepare", { id, requestId, args, streamIndex?, resolverId?, resolverStreamIndex?, resolverServers? })`

### Yêu cầu, phản hồi và môi trường

- `args.serverId` chỉ truyền cho `media.resolvePlayback` của nguồn; `resolverServers` là đối tượng chuỗi ánh xạ ID plugin resolver tới ID tuyến gốc, chỉ mục đã chọn của resolver mới được truyền cho `resolver.resolve` của nó.
- Hai `*streamIndex` là số nguyên 0–15, bỏ qua thì dùng 0; phạm vi lựa chọn do các luồng mà giai đoạn này trả về lần này quyết định.
- Trả về `PreparedMedia.choices.source` và `choices.resolver` tùy chọn: `pluginId`, `servers`, `serverId`, `streamIndex`, `streams`; mô tả luồng chỉ gồm index, label, height, bitrate, codec, kind, format, serverId.
- Mô tả lựa chọn không chứa URL tài nguyên, header yêu cầu hay khóa biến đổi. Địa chỉ phiên vẫn do proxy media cung cấp. Không cần biến môi trường mới.
- Đã lấy được ứng viên của nguồn nhưng luồng ưu tiên có định dạng không được hỗ trợ, resolver không khả dụng hoặc phân giải thất bại, thì lỗi đính kèm `playbackChoices` do host sinh, giao diện giữ lối vào để chọn thủ công ứng viên khác; trường này không chấp nhận plugin tự báo.
- localStorage `hpp.playback.servers` dùng giá trị tuần tự hóa của mảng JSON `["source" | "resolver", pluginId]` làm khóa, giá trị là serverId gốc; chỉ ghi lại tuyến thực tế đã chọn sau khi phân giải thành công.

### Kiểm chứng và lỗi

| Điều kiện | Kết quả |
| --- | --- |
| Chỉ mục là số âm, chuỗi hoặc lớn hơn 15 | INVALID_ARGUMENT, không thực thi plugin |
| Chỉ mục vượt danh sách ứng viên lần này | Báo lỗi rõ ràng, giữ điều khiển chọn thủ công |
| Luồng khai báo resolverId, đồng thời tồn tại lựa chọn UI cũ | Định tuyến theo khai báo của luồng; resolver đã khai báo không khả dụng thì báo lỗi rõ ràng |
| resolverServers chứa mục của resolver khác | Bỏ qua các mục không phải nhà cung cấp hiện tại, không chuyển cho nguồn |
| Nguồn/resolver trả về mục tuyến không hợp lệ | Loại bỏ độc lập; ID trùng giữ mục đầu tiên; loại bỏ trường không quen thuộc |
| AUTH_REQUIRED, AUTH_EXPIRED, CONTENT_LOCKED | Dừng phân giải, không tự động luân phiên ứng viên |

### Trường hợp bình thường, nền tảng và bất thường

- Bình thường: ứng viên page thứ 2 của nguồn → luồng trực tiếp thứ 2 của resolver chỉ định; nguồn và resolver dùng ID tuyến khác nhau, và hiển thị riêng.
- Nền tảng: chỉ có một luồng nguồn và không có resolver, ẩn điều khiển phụ; không gửi serverId của resolver.
- Bất thường: lỗi giải mã chí mạng trước hết làm mới một lần, khi bật "sau khi phát thất bại thử các luồng phát khác" thì mỗi ứng viên còn lại trong vòng này thử tối đa một lần, hết thì gợi ý thao tác thủ công; người dùng thử lại mới mở vòng mới.

### Khẳng định hồi quy

Rust `tests/playback_selection.rs` kiểm tra thuộc tính tham số hai giai đoạn, ưu tiên resolver chỉ định, hợp nhất thông tin phân giải, thông tin lựa chọn không rò rỉ cấu hình tài nguyên, xử lý chỉ mục không hợp lệ và mục xấu.
Kiểm thử tích hợp Edge kiểm tra chuyển luồng/tuyến thật giữ số giây, lựa chọn hai giai đoạn giữ lại sau làm mới, bền vững xuyên tập và tải lại, một lần làm mới và thử lại ứng viên có giới hạn, sau khi định dạng ưu tiên không được hỗ trợ thì phát thủ công ứng viên dự phòng.

### Ví dụ sai và đúng

Sai: đưa tuyến của resolver vào `args.serverId`, khiến nguồn nhận được ID tuyến của plugin khác.

Đúng: `args.serverId = "source-backup"`, `resolverServers = { "fixture.resolver": "resolver-backup" }`, mỗi giai đoạn chỉ nhận ID gốc của chính mình.

## Quy ước gọi phát trực tiếp và đồ văn

### Phạm vi áp dụng

Tương ứng 05 §9, 06 §8/9, giao diện trang chủ và chi tiết của 08, thuộc tính media của 09. Trang chủ hiển thị lối vào trực tiếp/đồ văn của nguồn theo phương thức cụ thể, nguồn bị vô hiệu hóa không hiển thị.

### Chữ ký gọi

- `hpp_manage("live.prepare", {id,requestId,args:{channelId,serverId?,reason,codecs},streamIndex?,resolverId?,resolverServers?,resolverStreamIndex?})`
- `hpp_manage("article.prepare", {id,requestId,args:{articleId,reason,codecs},streamIndex?,resolverId?,resolverServers?,resolverStreamIndex?})`
- Cả hai trả về `PreparedMedia` giống như VOD, việc giải phóng vẫn dùng `media.release`.

### Khế ước tham số và tài nguyên

Phát trực tiếp chỉ truyền cho `live.resolve` channelId gốc, tham số phát, không xây dựng mediaId/episodeId. Danh sách chương trình dùng cửa sổ mili giây Unix, sau khi người dùng mở mới yêu cầu 24 giờ tới.
Việc phát đồ văn yêu cầu lại `articles.getDetail({id:articleId})`, chỉ lấy streams trong đó đi vào chuỗi media dùng chung; các trường nội dung chính/tác giả không đi vào cấu hình phát.
Bình luận và tương tác của đồ văn lần lượt dùng `{kind:"article",articleId}`, không tái sử dụng mục tiêu media. Hai loại phát đều dùng cách ly nguồn/resolver, hủy bỏ, bản sửa đổi và thuộc tính phụ đề như cũ, không có biến môi trường mới.

### Kiểm chứng và xử lý thất bại

| Điều kiện | Kết quả |
| --- | --- |
| source chỉ có live đầy đủ | Bật bình thường, không yêu cầu media vỏ rỗng |
| Namespace live/articles thiếu phương thức bắt buộc | Khởi tạo thất bại, giữ bản ghi cài đặt có thể xem |
| ID kênh không hợp lệ/number, thời gian chương trình thiếu hoặc đảo ngược | Loại bỏ mục xấu độc lập, ghi lại đường dẫn |
| Nội dung chính của đồ văn không phải văn bản | Kiểm chứng kết quả thất bại |
| Nội dung chính vượt 512 KiB | Cắt theo biên ký tự UTF-8 và cảnh báo |
| streams là [] | Nội dung chính hiển thị bình thường, không hiển thị lối vào video |
| Danh sách chương trình thất bại | Không ảnh hưởng việc phát kênh, cung cấp thử lại độc lập |
| Lỗi phát chí mạng | Thử một lần reason:refresh, thất bại thì cho phép chuyển/thử lại thủ công |

### Trường hợp bình thường và bất thường

- Bình thường: nguồn thuần trực tiếp → phân trang kênh → bấm kênh → proxy native HLS → chuyển theo ID signal gốc → làm mới thủ công.
- Nền tảng: bài viết chỉ có văn bản thuần, kết xuất theo văn bản thuần, không cần khả năng video hay bình luận.
- Bất thường: bài viết chứa script/style/iframe/form, thuộc tính sự kiện hoặc liên kết javascript, toàn bộ node hoặc thuộc tính nguy hiểm không tham gia xây dựng node có thể nhìn thấy; hình ảnh trong nội dung chính phải đăng ký URL proxy trước, không thể điền trực tiếp src từ xa.

### Hành vi phải kiểm chứng

`tests/extra_content.rs` bao quát khởi tạo thuần trực tiếp, phương thức bị thiếu, mục độc lập không hợp lệ, ngân sách nội dung chính, streams rỗng, phiên media và mô tả lựa chọn công khai.
`scripts/test-plugin-extra-content.mjs` bao quát lối vào trang chủ của App bản sản xuất, phân trang kênh/bài viết, HLS/video thông thường, danh sách chương trình yêu cầu theo nhu cầu, thuộc tính tham số gốc, làm sạch nội dung chính và lệnh liên kết ngoài.
Ảnh bìa danh sách đăng ký lô hình ảnh tập trung, tránh mỗi hàng chiếm một phiên media; tập hợp vượt 200 hình ảnh đăng ký theo lô và khi trang bị hủy giải phóng toàn bộ phiên.

### Đối chiếu ví dụ triển khai

Sai: plugin trực tiếp giả mạo episodeId của VOD; HTML bài viết đưa nguyên trạng vào giao diện và giữ img src thượng nguồn.

Đúng: kênh chỉ truyền channelId; bài viết dùng phân tích mẫu lười biếng, rồi từ các node/thuộc tính được phép xây dựng phần tử React, hình ảnh thống nhất thay bằng địa chỉ cục bộ do assets.prepare trả về.

## Xác nhận xuyên nguồn và liên kết thư viện

Tương ứng 06 §17.2, 08 §6/7, không thay đổi chữ ký công khai của search/media của plugin. Sau khi người dùng mở bảng đổi nguồn, truy vấn song song theo tiêu đề hiện tại các nguồn đã bật, có thể nhìn thấy và triển khai search/media khác; năm và loại dùng cho việc sắp xếp của host, không thêm tham số tìm kiếm mà plugin chưa thỏa thuận.

### Lệnh liên kết

- `hpp_manage("library.sources.get", {pluginId,mediaId}) -> LibraryEntry[]`: trả về các mục khác trong nhóm liên kết, gồm các mục lịch sử mà nguồn không khả dụng. Giao diện dựa trên nguồn khả dụng hiện tại và tùy chọn nội dung người lớn quyết định có thể hiển thị và mở hay không.
- `hpp_manage("library.sources.link", {from:{pluginId,item:MediaItem},to:{pluginId,item:MediaItem}}) -> null`: chỉ gọi sau khi người dùng xác nhận, kiểm chứng hai phía rồi ghi nguyên tử một lần; đã có hai nhóm liên kết sẽ hợp nhất, liên kết trùng là lũy đẳng (idempotent).
- `library.json.associations` là ánh xạ từ khóa mục tới tập khóa mục của nhóm liên kết; khóa vẫn dùng `[pluginId,mediaId]` được tuần tự hóa JSON, không ghép bằng ký tự phân tách. favorite, tracking, lastProgress và progress theo từng tập gốc đều không chuyển, không xóa.
- Không có biến môi trường mới; không cần, cũng không thực thi việc di trú dữ liệu HGhub cũ.

### Phát sau khi xác nhận

Trong mọi danh sách phát đã trả về của chi tiết ứng viên, `season + episodeNumber` rõ ràng chỉ khớp một tập và không có phân trang chưa tải mới có thể chọn trước; cả hai phía đều thiếu season thì cho phép chọn trước theo episodeNumber rõ ràng. index giống nhau, định danh bị thiếu, nhiều khớp hoặc phân tập chưa trả về hết thì không tự động chọn tập.

Người dùng có thể chọn thủ công tập của nguồn mới, tải các trang tiếp theo của nó, hoặc chỉ mở chi tiết nguồn mới. Tập bị khóa không chọn được.
Lựa chọn xuyên trang chỉ lưu playlistId/episodeId thật; trang mới lấy lại chi tiết và dọc theo con trỏ gốc tìm tập đó, con trỏ trùng thì kết thúc, phân tập thay đổi thì dừng và yêu cầu chọn lại.

Tập đích đã có tiến độ cục bộ thì ưu tiên dùng bản ghi đích. Chưa có thì "dùng lại vị trí phát của nguồn gốc" mặc định không chọn; người dùng chọn mới sao chép số giây hữu hạn, không âm lúc xác nhận, nếu không bắt đầu từ 0.
Trước khi xác nhận, trước hết lưu tiến độ hiện tại của nguồn gốc, rồi mới thiết lập liên kết và thay thế tuyến chi tiết hiện tại; hủy bỏ, tìm kiếm, xem ứng viên đều không thiết lập liên kết, cũng không kích hoạt phân giải nguồn mới.

### Kiểm chứng và biên giới thất bại

| Tình huống | Hành vi |
| --- | --- |
| Định danh của bất kỳ mục liên kết nào hoặc MediaItem không hợp lệ | INVALID_ARGUMENT, bản ghi hai phía và đĩa đều không đổi |
| Liên kết tới chính mình | INVALID_ARGUMENT |
| Một nguồn tìm kiếm thất bại | Lỗi và thử lại độc lập của nguồn này, các ứng viên khác có thể tiếp tục chọn |
| Chi tiết ứng viên hoặc trang phân tập thất bại | Ở lại nguồn hiện tại, hiển thị thử lại, không phân giải media ứng viên |
| Đọc tiến độ của nguồn mới thất bại | Dừng việc xác nhận phát, tránh ghi đè tiến độ khi bản ghi ưu tiên chưa biết |
| Lưu liên kết thất bại | Giữ chi tiết và việc phát gốc, hiển thị thất bại |
| Tập của nguồn mới đã bị loại bỏ, bị khóa hoặc sau con trỏ trùng vẫn không tìm thấy | Không tự động phát, gợi ý người dùng chọn lại tập |
| Plugin bị gỡ bỏ | Liên kết và thư viện cũ được giữ, giao diện đánh dấu nguồn không khả dụng |

Trường hợp bình thường: S2E1 của A → S2E1 rõ ràng của B (dù index là 41), sau khi người dùng xác nhận phát bằng ID thật của B. Trường hợp nền tảng: không chọn dùng lại vị trí và B không có tiến độ, thì phát từ 0. Trường hợp bất thường: trong B có hai S2E1, yêu cầu chọn tập thủ công; index giống nhau không được coi là khớp.

Rust `content_library` kiểm tra tính đối xứng/hợp nhất/lũy đẳng của nhóm liên kết, khôi phục sau khởi động lại, tính nguyên tử của ghi không hợp lệ, và tính độc lập của yêu thích/theo dõi/tiến độ của từng nguồn. `scripts/test-plugin-source-switch.mjs` dùng QuickJS thật và giao diện bản sản xuất kiểm tra sắp xếp, cách ly thất bại, hủy bỏ, khớp mùa/tập, ưu tiên tiến độ, dùng lại số giây rõ ràng, chọn tập xuyên trang và lối vào liên kết trong "Của tôi".

Ví dụ sai: dựa trên `index:1` cũ coi mục đầu tiên của nguồn mới là cùng một tập, và trực tiếp ghi đè bản ghi cũ.
Ví dụ đúng: chỉ dùng định danh mùa/tập rõ ràng để đề xuất ứng viên, người dùng xác nhận nguồn mới và tập, mỗi nguồn tiếp tục ghi theo ID thật của chính mình.

## Quy ước gọi luồng thông tin

Tương ứng 05 §3, 06 §5, 08 §7 và quy ước phiên media của 09, không thêm khả năng hay trường mới cho plugin.

- Các mục của `feed.list({cursor?,limit:20})` được khử trùng theo ID; trang tiếp theo thất bại giữ mục đang phát hiện tại, thử lại dùng con trỏ gốc. Con trỏ trùng dừng việc tải tiếp; trang chỉ có mục trùng không đẩy chỉ mục tới mục rỗng.
- Khi tồn tại `entryEpisodeId` thì gọi trực tiếp `media.prepare`, `args` là `mediaId`, `episodeId` thật và tham số phát thông thường, bỏ qua `playlistId` chưa biết; sẽ không vì việc phát mà gọi thêm `media.getDetail`.
- Khi thiếu lối vào trực tiếp thì yêu cầu `media.getDetail({id})`, lấy tập đầu tiên của danh sách phát đầu tiên. Lô đầu của danh sách rỗng và có `next` thì theo `playlistId` và con trỏ gốc gọi một lần `media.listEpisodes`; bị khóa hoặc vẫn không có phân tập thì hiển thị lỗi và lối vào chi tiết, không dùng `index` giả mạo ID phân tập.
- Tiến độ của lối vào trực tiếp không có định danh danh sách phát dùng `[pluginId,mediaId,null,episodeId]` làm khóa, lưu riêng với tiến độ chứa `playlistId` thật; không yêu cầu di trú dữ liệu nền tảng cũ.
- Sau khi bắt đầu phát, lưu tiến độ cục bộ theo chu kỳ, tạm dừng, kết thúc và rời đi. Báo cáo thượng nguồn dùng sự kiện phát thực tế và quy ước heartbeat 30 giây, làm mới không lặp lại start; báo cáo thất bại im lặng.
- Mỗi lúc chỉ gắn một bên tiêu thụ phát; chuyển đổi, che phủ hoặc rời đi giải phóng phiên media, khi quay lại tiếp tục từ vị trí đã lưu. Tự động mục tiếp theo chịu sự điều khiển của tùy chọn ứng dụng; trang cuối không lặp vòng.
- Hồi quy dùng `tests/fixtures/feed.js` và `scripts/test-plugin-feed.mjs`, kiểm chứng qua QuickJS thật, proxy media và trang React bản sản xuất. Lối vào trình duyệt bật React StrictMode nhất quán với bản sản xuất, siêu dữ liệu danh mục không đệm Promise yêu cầu đã bị hủy.

## Biên tập trang chủ và phân trang cuộn

Tương ứng 08 §3–5 và 03 §3, không thêm phương thức plugin.

- `hpp.preferences` ghi nhớ thứ tự nguồn `order`, nguồn ẩn ở trang chủ `hiddenHomeIds`, thứ tự phân khu `sectionOrder` và ẩn hiện phân khu `hiddenSections`; ẩn hiện trang chủ không thay đổi phạm vi tìm kiếm. Chế độ nhiều nguồn vẫn giữ thứ tự lựa chọn, khi kéo nguồn sẽ đồng bộ thứ tự đã chọn của chế độ đó.
- "Điều chỉnh" của danh sách đề xuất cung cấp tay cầm kéo HTML native và nút lên/xuống có thể truy cập; trang chủ hiện tại phát trực tiếp, danh sách đề xuất đi vào từ lối vào trang danh mục. Danh mục dùng thứ tự nguồn; "tham gia toàn bộ" chỉ khôi phục phân khu của nguồn đó, không sửa các nguồn khác.
- Phân khu đề xuất, danh mục và danh sách phân khu dùng chung `usePagedPlugin`, tìm kiếm giữ khóa yêu cầu và chuỗi con trỏ độc lập của từng nguồn. Khử trùng theo ID gốc, con trỏ đã tiêu thụ không thể yêu cầu lại; nguồn trả về con trỏ trùng thì dừng và hiển thị gợi ý.
- Điểm neo cuối dùng container cuộn hiện tại làm gốc để tải trước, đồng thời giữ nút bấm. Trang tiếp theo thất bại sẽ không xóa nội dung đã có, cũng không tự động thử lại vô hạn; thử lại dùng con trỏ gốc lúc thất bại. Làm mới trang chủ thất bại giữ đề xuất hiện có.
- Khi truy vấn, lọc, sắp xếp, bản sửa đổi plugin hoặc phạm vi tìm kiếm thay đổi thì hủy yêu cầu cũ, phản hồi cũ không thể ghi đè kết quả mới. Cây danh mục chỉ đệm kết quả thành công, khóa chứa bản sửa đổi plugin; không đệm hay chia sẻ xuyên trang Promise yêu cầu chưa hoàn thành.
- `scripts/test-plugin-browsing.mjs` dùng thao tác kéo, cuộn thật và trang bản sản xuất để kiểm chứng, fixture ở `tests/fixtures/browsing.js` (trong crate HPP); toàn trình dùng con trỏ mờ, không chuyển nó thành số trang.

## Ghi nhớ resolver theo trang phát

Tương ứng 06 §3, 08 §7/8, không thay đổi `PlaybackParams` của plugin hay tham số resolver.

### Lệnh của host và mô tả lựa chọn công khai

- `resolver.preferences.get({}) -> Record<string,string>`: trả về ánh xạ từ tên host chính xác tới ID plugin resolver.
- `resolver.preferences.set({host,pluginId:string|null}) -> null`: chỉ chấp nhận resolver khả dụng hiện tại; null là quên host đó. Từ chối URL, cổng, đường dẫn, ký tự đại diện và kiểu không hợp lệ; tên host được chuẩn hóa thành chữ thường/IDNA, loại bỏ dấu chấm cuối DNS.
- `resolver.preferences.clear({}) -> null`: xóa toàn bộ ghi nhớ. `resolver-preferences.json` độc lập với dữ liệu plugin, trước hết ghi nguyên tử thành công rồi mới thay thế trong bộ nhớ; ghi thất bại giữ giá trị gốc. Không có biến môi trường mới.
- `PreparedMedia.choices.resolverRoute` và `playbackChoices.resolverRoute` do host sinh chứa `host`, `forced`, `selectedId?`, `resolvers:[{id,name}]`. Giao diện chỉ lấy host và các tùy chọn khả dụng, không lấy đường dẫn trang phát, tham số truy vấn, header yêu cầu hay khóa.
- `resolverHost` tùy chọn của `media/live/article.prepare` giới hạn `resolverId` thủ công hiện tại trong host tương ứng. Sau khi làm mới hoặc chuyển tập, nguồn đổi sang trang phát khác thì lựa chọn giao diện cũ không tham gia định tuyến trang mới; hai trường này sẽ không được truyền cho plugin nguồn hay resolver.

### Ưu tiên và lưu trữ bền vững

1. `resolverId` do luồng khai báo được ưu tiên, khi plugin chỉ định không khả dụng thì thất bại rõ ràng, không tự động đổi sang resolver khác.
2. Lựa chọn thủ công lần này cho host hiện tại ưu tiên hơn ghi nhớ.
3. Khi chưa chỉ định thủ công, từ các ứng viên khả dụng và `matches` khớp lấy ghi nhớ của host chính xác đó; ghi nhớ bị vô hiệu hóa, gỡ bỏ hoặc không khớp sẽ bị bỏ qua. Khớp duy nhất dùng trực tiếp, nhiều khớp mà không có ghi nhớ khả dụng thì yêu cầu lựa chọn.
4. Khi không có khớp, liệt kê các resolver đã cài khả dụng để chọn thủ công; không tự động áp dụng tùy chọn lịch sử khớp với host khác.
5. "Ghi nhớ lựa chọn cho trang này" mặc định tắt, chỉ khi lần phân giải này thành công, nhà cung cấp thực tế nhất quán với mục đã chọn và host không thay đổi mới lưu; thất bại hoặc hủy bỏ không ghi đè ghi nhớ cũ. Khi plugin chỉ định resolver thì không cung cấp lối vào ghi nhớ ghi đè.

Chi tiết VOD và phát trực tiếp/đồ văn/luồng thông tin dùng chung quy trình lựa chọn này; cài đặt cung cấp quên từng mục và xóa toàn bộ.
`tests/resolver_preferences.rs` kiểm chứng cách ly host chính xác, khởi động lại xuyên host, ưu tiên chỉ định, tùy chọn mất hiệu lực và thất bại ghi xuống đĩa;
`scripts/test-plugin-resolver-preferences.mjs` kiểm chứng VOD/trực tiếp thực tế, mặc định không ghi nhớ, chuyển tập xuyên trang, thất bại không ghi đè, tải lại trang và dọn dẹp cài đặt.

## Kiểm chứng và tái hiện

Thực thi tại thư mục gốc của kho:

```powershell
cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml
cargo check --manifest-path src-tauri/Cargo.toml
cargo build --manifest-path src-tauri/Cargo.toml
npm.cmd run build
npm.cmd run test:plugins
```

Khi phụ thuộc đã được đệm có thể thêm `--offline` cho Cargo. Kiểm thử trình duyệt mặc định dùng Windows Edge;
vị trí cài đặt khác có thể đặt `BROWSER_EXECUTABLE`. Thư mục dữ liệu kiểm thử được tạo tạm thời, không đọc plugin chính thức, trạng thái đăng nhập hay thượng nguồn.
Thư mục làm việc exFAT không hỗ trợ liên kết cứng; nếu MSVC báo thông tin gỡ lỗi bộ nhớ đệm tăng dần bị hỏng, có thể đặt `$env:CARGO_INCREMENTAL = '0'`
rồi build lại. Khi tài nguyên eo hẹp thêm `-j 2` cho Cargo.

`test:plugins` chạy host Rust/QuickJS thật và App, trang React bản sản xuất, chỉ thay Tauri IPC bằng truyền tải stdio.
Trình duyệt ghi cục bộ hình ảnh H.264 tổng hợp, rồi qua proxy media thực tế kiểm chứng kéo video thông thường, phát đoạn khởi tạo/phân đoạn HLS và làm mới.
Hồi quy mới bao quát điều hướng mới, trang chủ, lọc/sắp xếp danh mục, hợp nhất tìm kiếm đa nguồn và cách ly thất bại đơn nguồn, yêu thích/theo dõi cục bộ, phát tiếp phân tập thực tế.
Đồng thời dùng `src-tauri/crates/hpp/tests/fixtures/forms.js` kiểm chứng biểu mẫu tài khoản/thao tác, đăng nhập nhiều bước, khôi phục và mất hiệu lực trạng thái đăng nhập,
dọn dẹp phiên cục bộ sau khi giao diện đăng xuất thất bại, và hủy bỏ gửi khi rời bảng. Lối vào đăng ký chỉ kiểm chứng lệnh liên kết ngoài thông thường, không truy cập trang bên ngoài.
`src-tauri/crates/hpp/tests/fixtures/social.js` và `scripts/test-plugin-social.mjs` kiểm chứng bình luận/trả lời, tương tác đám mây, phân trang thư mục và theo dõi, hủy bình luận, và báo cáo sự kiện phát thực tế; heartbeat dùng đồng hồ đơn điệu có thể tiêm để kiểm chứng biên 30 giây.
Hồi quy trực tiếp và đồ văn dùng `src-tauri/crates/hpp/tests/fixtures/extra-content.js`; nguồn thuần trực tiếp không khai báo media VOD, để tránh vỏ kiểm thử che giấu ràng buộc khởi tạo.
Đường cơ sở phát triển riêng cục bộ kiểm chứng độc lập, không đưa vào client, kiểm thử công khai hay sản phẩm trang tài liệu.
Sản phẩm nằm ở `test-results/plugins/<thời gian>/` bị bỏ qua khi commit, gồm báo cáo kiểm thử, ảnh chụp màn hình và bản ghi yêu cầu cục bộ.
Kiểm thử này không tương đương kiểm chứng cửa sổ Tauri WebView2 thực tế/đóng gói.

## Kiểm chứng lịch sử (2026-11-13, giao diện sớm)

- Kiểm thử Rust HPP 46 mục toàn bộ đạt, gồm cách ly và lưu trữ bền vững tiến độ luồng thông tin không có playlist, hợp nhất/lũy đẳng/tính nguyên tử của nguồn liên kết, lưu trữ bền vững/ưu tiên/quay về khi mất hiệu lực của tùy chọn trang resolver; hồi quy bình luận/tương tác, lọc kết quả trực tiếp/đồ văn, khám phá khả năng thuần trực tiếp, phiên media, cài đặt, tài khoản, phát và mạng đều đạt.
- Kiểm thử nhật ký Rust của ứng dụng Tauri 3 mục đạt.
- Kiểm tra kiểu frontend, build bản sản xuất và build tệp thực thi debug Windows đều đạt.
- Phát thực tế trên Edge và hồi quy chế độ nghiêm ngặt của App bản sản xuất đạt, báo cáo: `test-results/plugins/2026-11-13T05-32-01-958Z/report.json`, gồm 25 nhóm kết quả đạt.
- Bao quát trang chủ/danh mục, hợp nhất đa nguồn và lựa chọn nguồn, cách ly thất bại đơn nguồn, yêu thích/theo dõi/phát tiếp phân tập, biểu mẫu tài khoản,
  nhập danh sách đám mây, lọc thuộc tính kết quả theo dõi, gỡ plugin giữ bản ghi nguồn không khả dụng, điều hướng danh sách đầy đủ của phân khu, làm mới và thăm dò tuyến.
- Phân phối bao quát truyền tải lựa chọn tệp, danh sách tương đối cục bộ, danh sách từ xa không đuôi sau chuyển hướng, giới thiệu và tên nguồn, chỉ tải xuống các mục đã chọn,
  cách ly mục định danh không khớp, chỉ cài đặt, kiểm tra cập nhật không thay thế tệp, khôi phục cập nhật thất bại và cài đặt phiên bản cao hơn đã chọn.
- Việc phát bao quát chuyển luồng và tuyến của nguồn/resolver, giữ số giây, giữ lựa chọn sau làm mới và xuyên tập, bền vững sau tải lại, chuyển luồng ứng viên có giới hạn sau một lần làm mới,
  và phát thủ công luồng dự phòng sau khi định dạng ưu tiên không được hỗ trợ; cài đặt mới có thể tắt chuyển luồng ứng viên tự động.
- Bình luận bao quát mục tiêu tác phẩm/phân tập/trả lời, phân trang độc lập, hiển thị văn bản thuần, chưa đăng nhập giữ bản nháp, chèn bình luận trả về và làm mới khi không có trả về, hủy bỏ gửi khi rời trang, UNSUPPORTED lần lượt ẩn lối vào mục tiêu, gửi hoặc lượt thích.
- Tương tác đám mây bao quát lượt thích/yêu thích/theo dõi tác giả và bỏ theo dõi, ID gốc của thư mục yêu thích, phân trang danh sách theo dõi; tiến độ bao quát start/pause/end của việc phát thực tế, im lặng khi xác thực thất bại, biên heartbeat 30 giây và ức chế sự kiện trùng lặp.
- Trực tiếp/đồ văn bao quát nguồn thuần trực tiếp, nhóm/phân trang kênh, danh sách chương trình theo nhu cầu, chuyển tuyến thời gian thực, phân trang đồ văn, danh sách cho phép nội dung chính, proxy media hình ảnh, mở liên kết ngoài, video trong bài và tương tác bình luận bài viết.
- Luồng thông tin bao quát lối vào trực tiếp không lấy chi tiết, quay về phân trang danh sách phát/phân tập thật, thử lại khi thất bại giữ mục, khử trùng và kết thúc khi con trỏ trùng, yêu thích và phát tiếp khi tạm dừng/rời đi, làm mới không lặp lại start, công tắc tự động mục tiếp theo, giải phóng việc phát ở trang che phủ, và điều hướng luôn nằm trong cửa sổ; ảnh chụp màn hình là `feed-player.png` cùng thư mục.
- Sửa lỗi trang danh mục khi gắn lại trong React StrictMode tái sử dụng yêu cầu đã bị hủy khiến không tải được; lối vào bản sản xuất và kiểm thử trình duyệt dùng cùng một chế độ nghiêm ngặt.
- Đổi nguồn bao quát sắp xếp ứng viên chính xác/tương tự, cách ly thất bại tìm kiếm đơn nguồn, hủy bỏ không thiết lập liên kết, khớp mùa/tập rõ ràng (index khác nhau), ưu tiên tiến độ đích, mặc định không dùng lại số giây cũ, dùng lại rõ ràng, chọn thủ công khi số tập mơ hồ, định vị lại phân tập xuyên trang thật, và lối vào liên kết trong "Của tôi" cùng việc giữ quan hệ sau khi gỡ cài đặt; ảnh chụp màn hình `source-switch-confirmation.png`.
- Duyệt bao quát kéo nguồn/phân khu thật, di chuyển bằng nút, ẩn hiện/tham gia toàn bộ, bền vững sau tải lại và thứ tự nguồn danh mục; phân trang kích hoạt bằng cuộn, trang tiếp theo thất bại giữ nội dung, thử lại với con trỏ gốc, khử trùng ID, kết thúc khi con trỏ trùng, hủy bỏ lọc/sắp xếp/truy vấn, thất bại độc lập đa nguồn và làm mới trang chủ thất bại giữ nội dung. Cây danh mục xuyên điều hướng tái sử dụng bộ nhớ đệm thành công, thay thế cùng phiên bản có bản sửa đổi mới thì lấy lại; ảnh chụp màn hình `home-arrangement.png`, `browsing-pagination.png`.
- Tùy chọn resolver bao quát mặc định không ghi nhớ, chỉ khi chọn và thành công mới lưu, thất bại không ghi đè tùy chọn cũ, cách ly chuyển tập xuyên host phát, ưu tiên resolver chỉ định, định tuyến tự động sau tải lại trang, lối vào dùng chung cho VOD/trực tiếp, và quên/xóa từng mục trong cài đặt cùng tải lại sau khi xóa; ảnh chụp màn hình `resolver-site-preference.png`, `resolver-preference-settings.png`.
- Kiểm thử chỉ truy cập dịch vụ hư cấu cục bộ, không có yêu cầu tới trang nội dung bên ngoài. Hồi quy cầu stdio trình duyệt không tương đương nghiệm thu cửa sổ WebView2 thực tế.

## Kiểm toán 2026-11-15 và hồi quy giao diện cổ điển

Trang chủ và lối vào chi tiết trên máy tính hiện tại dùng chung trình phát nhập vai cổ điển, chọn tập, bình luận, chi tiết mở qua ngăn kéo; di động dùng thanh đáy dạng chữ và cử chỉ trình phát. entryEpisodeId của luồng thông tin phân giải trực tiếp, khi mở ngăn kéo chi tiết/chọn tập mới yêu cầu chi tiết, không tự thêm playlistId. Gợi ý lỗi không che yêu thích và chọn tập.

Vòng này toàn bộ bộ kiểm thử Rust (gồm runtime_contract và public_example) đạt; hồi quy chuyên đề xã hội, trực tiếp/đồ văn, luồng thông tin, đổi nguồn và resolver của UI cổ điển đạt. Android sau khi sửa nhập tệp qua content URI đã đo thực tế; APK cuối cùng dán/nhập tệp và phát một MP4 thật (1920×804) đạt. Frontend đầy đủ 25 nhóm, Rust 52 mục đạt. Khoảng trống còn lại xem thuyết minh hạn chế của tài liệu này, không lấy kiểm thử giao diện lịch sử làm nghiệm thu hiện tại.

## Bản vá engine

Promise reaction/await thông thường của `rquickjs-sys 0.12.2` không cung cấp hook Before/After đầy đủ.
Bản vá vendor cục bộ bắt ngữ cảnh đăng ký cho mỗi continuation, tránh dùng lẫn requestId khi các lần gọi đồng thời hoặc chia sẻ Promise.
Chi tiết xem `src-tauri/vendor/rquickjs-sys/HGHUB-PATCH.md`; khi nâng cấp engine phải giữ kiểm thử hồi quy tương ứng.
