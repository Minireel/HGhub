# 07 · Hệ sinh thái và tiến hóa

[← Tổng quan](README.md)

## 1. Phiên bản và tương thích

### 1.1 Số phiên bản

- Phiên bản giao thức là `chính.phụ.sửa đổi`, host công bố qua `hg.env.protocol` (như `"1.0.0"`).
- Plugin trong `meta.protocol` chỉ ghi **phiên bản chính** (số nguyên).

### 1.2 Phiên bản phụ chỉ làm tăng thêm

Trong cùng một phiên bản chính, phiên bản phụ mới **chỉ có thể**:

- Thêm namespace khả năng mới, thêm phương thức tùy chọn mới
- Thêm trường tùy chọn vào `params`, thêm trường tùy chọn vào kiểu trả về
- Thêm module hoặc phương thức `hg.*` mới, thêm đặc tính tùy chọn mới
- Thêm trường tùy chọn của `meta` mới, thêm mã lỗi mới

Từ đó được quy tắc tương thích hai chiều:

| Một phía | Phải làm được |
| --- | --- |
| Plugin | Bỏ qua các trường không nhận biết trong `params`; dùng `hg.env.features` và "phương thức có tồn tại hay không" để thăm dò, thay vì so sánh số phiên bản |
| Host | Bỏ qua các trường không nhận biết trong `meta` và giá trị trả về; phương thức tùy chọn plugin chưa triển khai thì hạ cấp theo "không hỗ trợ"; mã lỗi không quen thuộc xử lý theo `INTERNAL` |

Việc mở rộng tăng thêm đảm bảo plugin cũ tiếp tục chạy trên host mới; tiền đề để plugin bản mới chạy trên host cũ là nó đã cung cấp hạ cấp hiệu lực cho khả năng mới.
"Phiên bản chính giống nhau" không có nghĩa host có mọi chức năng plugin cần.

### 1.3 Khả năng bắt buộc và hạ cấp

- `meta.features` chỉ làm thuyết minh, không thể thay thế việc thăm dò lúc chạy. Chức năng có thể hạ cấp dùng `hg.env.features` hoặc tính tồn tại của phương thức để phán đoán.
- Chức năng không thể hạ cấp thì ghi vào `meta.requires`: `feature` đối chiếu `hg.env.features`, `api` đối chiếu đường dẫn công khai host đã triển khai,
  đường dẫn tương đối với hg, ví dụ `html.parse`, `crypto.rsaEncrypt`, không phải phương thức plugin, cũng không dùng ký tự đại diện.
- Host kiểm tra requires trước khi bật, khi có mục không quen thuộc hoặc thiếu chức năng thì giữ tệp cài đặt nhưng đánh dấu "không tương thích", hiển thị các mục còn thiếu, không thực thi factory.
  Host của giao thức 1.0 phải triển khai việc kiểm tra requires, không được bỏ qua như siêu thông tin không quen thuộc thông thường những nhu cầu không hiểu được trong requires đã biết.
- Sự tồn tại của API lõi 1.0 không muộn hơn định nghĩa kiểu; API thêm mới sau này là tùy chọn, và cung cấp thăm dò bằng đặc tính hoặc phương thức.
  Bản 1.0 chính thức không cam kết tương thích các triển khai trung gian của dự thảo này.

### 1.4 Phiên bản chính

Chỉ thay đổi phá vỡ (xóa hoặc thay đổi ngữ nghĩa của trường đã có) mới tăng phiên bản chính.
Sau khi tăng phiên bản chính, host **nên** tiếp tục hỗ trợ plugin của phiên bản chính trước đó một thời gian, và trong danh sách plugin ghi nhãn "hướng tới giao thức bản cũ".

## 2. Đối chiếu và lựa chọn

Tham khảo ba dự án mã nguồn mở lấy plugin làm cốt lõi (kết luận đến từ việc đọc mã nguồn và tài liệu của chúng): Cloudstream (video, plugin Kotlin),
Venera (truyện tranh, plugin JS), Miru (anime / truyện tranh / tiểu thuyết, plugin JS).

| Điểm thiết kế | Cloudstream | Venera | Miru | Lựa chọn của HPP |
| --- | --- | --- | --- | --- |
| Đơn vị plugin | Gói bytecode đã biên dịch, cần project Gradle để build | Một `.js` duy nhất, `class X extends ComicSource` | Một `.js` duy nhất, siêu thông tin đầu chú thích + `export default class` | Một `.js` duy nhất, `export const meta` + hàm factory xuất mặc định |
| Đọc siêu thông tin | Manifest trong gói + chỉ mục kho | Regex tìm tên lớp rồi thực thi, đọc trường của thể hiện | Regex quét `@key value` trong **toàn bộ tệp** | Đối tượng có cấu trúc, có thể kèm mục cài đặt; không dựa vào regex |
| Chỉ mục theo lô | Tệp kho trỏ tới một số tệp danh sách plugin | Danh sách JSON của địa chỉ tùy ý | `<kho>/index.json` | `manifest.json`, hỗ trợ đường dẫn tương đối và URL |
| Cập nhật | Khi khởi động so sánh phiên bản và tự động cập nhật | So sánh phiên bản chỉ mục rồi tải lại | Không có đường đi cập nhật đầy đủ | So sánh phiên bản, địa chỉ nguồn hoặc `updateUrl` lấy tệp mới |
| Môi trường thực thi | **Cùng tiến trình, cùng quyền với ứng dụng**, có thể gọi bất kỳ giao diện nào của ứng dụng | QuickJS, **mọi plugin dùng chung một engine** | Các nền tảng khác nhau dùng engine khác nhau, mỗi cái một bộ mã cầu nối | Triển khai engine thống nhất, mỗi plugin một runtime, ngữ cảnh và điều độ độc lập, chỉ qua `hg.*` tương tác với bên ngoài |
| Timeout / giới hạn trên bộ nhớ | Timeout coroutine (mặc định 120 giây, plugin điều chỉnh được); không có giới hạn trên bộ nhớ | Không có | Không có | Có ([02 §7](02-runtime.md)) |
| Host gọi plugin thế nào | Gọi phương thức trực tiếp | Sinh mã nguồn JS thực thi | **Ghép** tham số vào chuỗi mã nguồn | Tham số đi qua cầu có cấu trúc, không ghép mã nguồn |
| Mạng | Plugin tự mang thư viện HTTP nối mạng trực tiếp | Đóng gói HTTP của host + `fetch` toàn cục | `this.request`, ngoài ra có thể bật `fetch` vượt qua đóng gói | Chỉ có `hg.http`, thống nhất qua lớp mạng của host |
| Phân trang | Số trang | Số trang + trang tối đa, hoặc token `next`, hai bộ phương thức | Chỉ số trang | Con trỏ mờ thống nhất |
| Trang chủ | Plugin khai báo một số phân khu, phân trang theo từng khu, có thể ghi nhãn hình ngang | Trang khám phá: trang phân khu / trang danh sách / hỗn hợp | Chỉ danh sách "mới nhất" | Phân khu `home` (có thể phân trang, có thể ghi nhãn hình ngang) + `feed` độc lập |
| Phân giải phát | Dạng callback chảy ra một lô liên kết; tách nhà lưu trữ video thành resolver tái sử dụng được | Việc lấy chương và cấu hình tải theo từng hình tách riêng | Bản mới tách "liệt kê bản sao" và "phân giải bản sao" | `servers` liệt kê tuyến phát, `resolvePlayback(serverId)` phân giải theo nhu cầu; resolver nhà lưu trữ độc lập thành plugin, định tuyến theo mẫu host |
| Đăng nhập | Giao diện nguồn dữ liệu không có quy trình đăng nhập thống nhất | Ba loại mật khẩu tài khoản / web / Cookie; thỏa thuận ném chuỗi `Login expired` | Sau khi đăng nhập web ghi Cookie | Luồng biểu mẫu (có thể nhiều bước) + khung xem web tùy chọn; `AUTH_EXPIRED` có kiểu |
| Cài đặt | Plugin tự vẽ giao diện cài đặt native | Khai báo trong plugin, gồm nút dạng callback | Đăng ký lúc chạy | Khai báo tĩnh `meta.settings` |
| Khả năng giao diện | Plugin có thể thao tác trực tiếp giao diện native | Plugin có thể bật thông điệp, hộp thoại, ô nhập | Không có | Plugin không vẽ giao diện (xem vấn đề mở ở §6) |
| Tổng hợp | Khi tìm kiếm truy vấn song song từng plugin, nhóm theo nguồn | — | — | Giống bên trái, ngoài ra có đổi nguồn khớp theo tiêu đề và năm |
| Hỗ trợ tác giả | Ngôn ngữ kiểu mạnh + công cụ kiểm thử khói tích hợp | Chỉ tệp stub JSDoc | Chỉ mô tả giao diện trong tài liệu | Dự định cung cấp định nghĩa kiểu, host giả và tự kiểm tra tích hợp |

### 2.1 Cách làm đáng học

- **Plugin đơn tệp + chỉ mục độc lập**: Venera và Miru đều chứng minh đây là hình thái cộng đồng dễ chấp nhận nhất.
- **Phân khu trang chủ do plugin điều khiển** (Cloudstream): plugin khai báo một số phân khu, mỗi phân khu có thể lật trang độc lập, và có thể ghi nhãn dùng hình ngang.
  Tương ứng `home`.
- **Tách việc phân giải nhà lưu trữ thành module tái sử dụng được** (extractor của Cloudstream): nguồn phim ảnh dùng một lô nhà cung cấp lưu trữ video rất nhiều,
  việc viết lại logic phân giải theo từng nguồn vừa lãng phí vừa dễ mất hiệu lực. Tương ứng plugin resolver ([06 §3](06-capabilities.md)).
- **Tách "liệt kê các mục có thể phát" và "phân giải một mục trong đó"**: tránh việc vì hiển thị danh sách tuyến mà yêu cầu từng tuyến một,
  cũng khiến việc đặt header yêu cầu theo tuyến trở nên tự nhiên.
- **Định danh mục ổn định** (Cloudstream vì điều này đặt riêng một trường): địa chỉ trang có thể mang tham số sẽ hết hạn,
  khóa của thư viện phải tách khỏi nó. Tương ứng yêu cầu tính ổn định của `id` ở [05 §1](05-content-model.md).
- **Chia mùa, diễn viên và đoàn làm phim, đoạn giới thiệu** (Cloudstream): trang chi tiết của trình xem đa năng cần chúng, đều là trường tùy chọn.
- **Tìm kiếm tổng hợp nhóm theo nguồn, đổi nguồn khớp theo tiêu đề và năm** (Cloudstream): tương ứng [06 §17](06-capabilities.md).
- **Dán liên kết mở trực tiếp** (`link` của Venera): tương ứng `media.matchUrl` và `meta.linkDomains`.
- **Nhóm thư mục yêu thích đám mây** (Venera): tương ứng `library.listFolders`.
- **Kiểm tra cập nhật theo từng mục** (`checkUpdate` của Miru): tương ứng `updates.check`, đổi thành theo lô để giảm yêu cầu.
- **Kiểm thử khói tích hợp** (Cloudstream) và **bảng nhật ký trong ứng dụng** (Miru): hai công cụ giúp tác giả plugin nhiều nhất, đưa vào lộ trình.

### 2.2 Các hố cần tránh

| Hố | Nguồn gốc | Cách làm của HPP |
| --- | --- | --- |
| Plugin cùng tiến trình cùng quyền với ứng dụng, từng xuất hiện plugin bật quảng cáo chuyển hướng, kéo ứng dụng khác lên | Cloudstream | Plugin trong engine độc lập, không có giao diện và giao diện hệ thống, về cấu trúc không làm được |
| Trường loại trong chỉ mục là văn bản tự do, xuất hiện giá trị viết không nhất quán | Cloudstream | Kiểm chứng liệt kê (enum), giá trị không nhận biết xử lý theo `other` |
| Trường số phiên bản giao diện tồn tại nhưng không có hiệu lực, không tương thích phải tới lúc chạy báo lỗi mới lộ ra | Cloudstream | Khi nhập kiểm tra `meta.protocol`; lúc chạy dùng thăm dò đặc tính |
| Mọi plugin dùng chung một engine và đối tượng toàn cục, một vòng lặp vô hạn làm kẹt toàn bộ, plugin đọc được dữ liệu của nhau | Venera | Mỗi plugin một runtime, điều độ thực thi và lưu trữ độc lập |
| Không có timeout lần gọi và giới hạn trên bộ nhớ | Venera, Miru | Thời hạn, giới hạn trên bộ nhớ, vô hiệu hóa sau khi sụp đổ |
| Ghép tham số lần gọi vào mã nguồn JS, dấu ngoặc kép trong từ khóa tìm kiếm là phá được lần gọi | Miru | Tham số truyền qua cầu có cấu trúc |
| Các nền tảng khác nhau dùng engine khác nhau, duy trì hai bộ cầu nối, từng nền tảng riêng lẻ không dùng được | Miru | Một engine nhúng duy nhất |
| Dùng regex lấy siêu thông tin trong toàn bộ tệp; trường bắt buộc bị thiếu thì sụp đổ trực tiếp | Miru | `meta` có cấu trúc; thiếu trường cho lỗi nhập rõ ràng |
| Giám sát thư mục plugin tự động tải lại, xuất hiện plugin "bị mất", "hiển thị chưa cài đặt" | Miru | Nhập, cập nhật, gỡ bỏ đều là thao tác rõ ràng |
| Không có đường đi cập nhật | Miru | So sánh phiên bản + địa chỉ cập nhật rõ ràng |
| Tên trường của tài liệu và triển khai không nhất quán | Venera | Tệp định nghĩa kiểu làm nguồn chân lý duy nhất, tài liệu đối chiếu từ nó |
| Thỏa thuận ném chuỗi cụ thể biểu thị đăng nhập mất hiệu lực | Venera | Mã lỗi có kiểu |

## 3. Đường cơ sở triển khai client

HGhub 0.0.1 là trình phát video plugin mới giữ phong cách frontend hiện có, không tiếp nối nghiệp vụ nền tảng cũ, cũng không di trú tài khoản và thư viện cũ.

- `src-tauri/crates/hpp` cung cấp runtime độc lập, mạng dùng chung, kiểm chứng, quản lý plugin, phiên media và thư viện cục bộ.
- `src-tauri/src` chỉ giữ bộ chuyển đổi Tauri, cài đặt ứng dụng, nhật ký và khả năng cửa sổ.
- Giao diện qua lệnh host dùng chung gọi plugin, không chứa giao diện trang, tên miền, thông tin xác thực hay tri thức giải mã.
- Thư viện cục bộ bắt đầu từ thư viện rỗng, khóa mục là "ID plugin + ID mục"; tiến độ phân tập còn gồm ID danh sách phát và ID phân tập thật.
- Lối vào bảng xếp hạng và tài khoản nền tảng hiện có không giữ lại; bảng xếp hạng do phân khu home hoặc sắp xếp catalog biểu đạt, tài khoản thuộc về plugin.

Mục tiêu đầy đủ xem [08](08-frontend.md), phạm vi đã hoàn thành thực tế xem [Tiến độ triển khai](IMPLEMENTATION.md).

## 4. Điểm chính triển khai (không quy phạm)

- **Cầu mỏng + script dẫn đầu JS**: Phía native chỉ lộ một lối vào thông điệp có cấu trúc, phần lớn đóng gói tiện lợi của `hg.*` dùng một đoạn script dẫn đầu JS phát hành cùng host
  để triển khai. Bề mặt native nhỏ, việc tiến hóa giao diện chủ yếu sửa script.
- **Đọc `meta`**: Trong một runtime độc lập dùng xong bỏ, không có I/O bên ngoài, định giá module có giới hạn thời gian, chỉ lấy xuất `meta`,
  không gọi xuất mặc định; không thể coi "chưa thực thi factory" là đồng nghĩa "chưa thực thi JS tầng cao nhất".
- **Truyền tham số**: Khi host gọi phương thức plugin, tham số truyền vào dưới dạng giá trị có cấu trúc, không sinh mã nguồn JS.
- **Thư mục plugin**: Mỗi plugin một tệp `<id>.js`, bên cạnh là lưu trữ và cài đặt của nó; host lưu riêng một chỉ mục ghi lại nguồn gốc và thời gian nhập.
- **Phiên media**: Quy tắc truyền tải đầy đủ xem [09](09-media-transport.md). Việc đăng ký tài nguyên phải giữ nhà cung cấp, bản sửa đổi mã và chuỗi phân giải,
  không thể chỉ lưu URL và header yêu cầu. Nền tảng có thể chọn triển khai truyền tải cục bộ khác nhau, giao diện plugin công khai giữ nhất quán.
- **Công cụ tác giả**:
  - Bảng nhật ký và bản ghi yêu cầu mạng ở trang chi tiết plugin (địa chỉ, mã trạng thái, thời gian, có trúng bộ nhớ đệm hay không).
  - "Tải lại từ tệp": tác giả sửa xong tệp cục bộ bấm một cái tải lại, không cần nhập lại.
  - Tự kiểm tra plugin: theo khả năng thực tế chọn đường đi VOD, trực tiếp hoặc resolver; khi không có lối vào nội dung thì để người dùng cung cấp ID mục, liên kết hoặc trang phát,
    không giả mạo query, cũng không cam kết plugin tùy ý đều tự động chạy xong được. Các bước có thể bỏ qua, phân biệt thiếu đầu vào, không hỗ trợ và thất bại thật sự.
  - Bảng yêu cầu mạng: địa chỉ, mã trạng thái, thời gian, có trúng bộ nhớ đệm hay không.
  - Tệp định nghĩa kiểu, và một `hg` giả có thể chạy trong Node.js, dùng cho kiểm thử đơn vị.

## 5. Lộ trình

| Giai đoạn | Nội dung | Dấu hiệu hoàn thành |
| --- | --- | --- |
| P0 | Kiểu và kiểm chứng giao thức, hg giả, nguồn JSON/HTML hư cấu và fixture nguồn, resolver phụ thuộc Cookie; runtime độc lập, hủy bỏ và ngân sách; `hg.http/storage/cookies/log/crypto/codec/html`; nhập cục bộ, khởi tạo và endpoints; `media/search/resolver`; phiên media, HLS và progressive | Chuỗi hư cấu chạy thông nhập → khởi tạo khả năng/tuyến → tìm kiếm → chi tiết → phát → làm mới; kiểm chứng hủy bỏ, kéo MP4, khóa HLS và thuộc tính tài nguyên của resolver; năm đầu kiểm chứng truyền tải cốt lõi, hoàn thành thì đóng băng khế ước lõi 1.0 |
| P1 | Thư viện plugin hoàn toàn mới; `feed/catalog/comments/interact/auth/ui`, biến đổi ảnh bìa; trang plugin, nhật ký cơ sở, tự kiểm tra; loại bỏ tri thức trang của host | Chức năng VOD dùng qua plugin bên ngoài, khóa chính đa nguồn đã sẵn sàng; cây công khai và sản phẩm phát hành không có tri thức thượng nguồn thật |
| P2 | `manifest.json` và nhập URL tùy ý, kiểm tra cập nhật, đăng ký; `home` và biên tập tổng hợp; cột danh mục, phạm vi tìm kiếm và hiển thị tổng hợp; loại bỏ bảng xếp hạng độc lập | Đồng thời bật nhiều plugin, thất bại không kéo lẫn nhau; ghi đè cùng phiên bản, nâng cấp, hạ cấp và hoãn cập nhật tự động trong lúc phát khả dụng |
| P3 | Đổi nguồn và liên kết nguồn, khớp mùa/tập; `articles/library/progress/updates`; bảng yêu cầu mạng và tài liệu tác giả hoàn thiện | Đổi nguồn giữ nguồn cũ và tiến độ, không khớp sai mùa/tập; tác giả có thể dựa trên kiểu và mẫu hư cấu triển khai plugin độc lập |
| P4 | `live`, đường đi nguồn thuần trực tiếp; `hg.webview` tùy chọn; DASH và biến đổi phân đoạn đã khai báo hỗ trợ; nghiệm thu khả năng xuyên đầu | Phát/làm mới kênh, đăng nhập web và các đặc tính tùy chọn đã khai báo đạt nghiệm thu trên nền tảng đích, hoàn thành triển khai khả năng 1.0 |

Các giai đoạn này là thứ tự triển khai, không có nghĩa có thể tuyên bố ra ngoài các module chưa triển khai là đã hỗ trợ. Host phát hành thành 1.0 đầy đủ sau khi đóng băng phải thỏa mãn mọi khế ước bắt buộc;
các bản build trung gian từ P0 tới P3 ghi nhãn là bản phát triển. Đặc tính tùy chọn chỉ khai báo sau khi triển khai thực tế và qua nghiệm thu.

### 5.1 Tài sản kiểm chứng P0

Ví dụ hư cấu chỉ kết nối fixture kiểm thử cục bộ hoặc giữ tên miền ví dụ, dùng ID, khóa và phản hồi hư cấu, không cung cấp bất kỳ nguồn phim ảnh thật nào.
Fixture kiểm thử thông qua cấu hình kiểm thử nới lỏng chính sách địa chỉ cục bộ, không thay đổi quy tắc mạng mặc định của bản sản xuất. Ví dụ giao thức không phải plugin nội dung phân phối cho người dùng.

Ít nhất bao quát: nguồn VOD JSON, nguồn VOD HTML, nguồn dữ liệu cần Cookie → resolver độc lập, endpoints lần đầu không có lựa chọn,
lần gọi xen kẽ và hủy bỏ của cùng plugin, xây dựng lại khi runtime vượt mức, khôi phục phân trang, thuộc tính tài nguyên con HLS, Range, làm mới thời hạn hiệu lực và cập nhật plugin làm phiên cũ kết thúc.
Định nghĩa kiểu và kiểm chứng giá trị trả về duy trì đồng bộ; việc kiểu biên dịch được không thay thế nghiệm thu chuỗi media thực tế.

## 6. Mở rộng ứng viên và vấn đề mở

### 6.1 Ứng viên sau 1.0

| Hướng | Mô tả |
| --- | --- |
| Khung xem web lấy chuỗi | Host tải trang phát ở nền và chặn yêu cầu media trong đó, giao địa chỉ lại cho plugin. Cloudstream dùng cách này đối phó script trang khó khôi phục; chi phí triển khai trên năm đầu cao, cần đánh giá riêng |
| Liên kết nhập một bấm | Đăng ký liên kết ứng dụng (như `hghub://import?url=…`), trên trang web bấm một cái là nhập được plugin hoặc danh sách plugin |
| Nguồn dữ liệu tăng cường trải nghiệm | Bình luận chạy (danmaku), thư viện phụ đề, dữ liệu phim ảnh, đồng bộ lịch sử xem |
| Nội dung đọc | Truyện tranh, tiểu thuyết; cần thêm một nhóm khả năng, định dạng tệp và runtime không đổi |
| Tải xuống | 1.0 không yêu cầu giao diện tải xuống; sau này tái sử dụng phiên media, chuỗi phân giải và ngữ nghĩa làm mới, không giả định một URL tĩnh là đủ để hoàn thành việc tải xuống dài hạn |

### 6.2 Vấn đề mở

| Vấn đề | Hiện trạng | Phương án dự phòng |
| --- | --- | --- |
| Hook media theo từng yêu cầu | Chỉ có header yêu cầu tĩnh và biến đổi byte | Cho phép plugin viết lại động địa chỉ hoặc header yêu cầu cho mỗi yêu cầu phân đoạn (Cloudstream có giao diện này); cái giá là trong lúc phát đánh thức plugin với tần suất cao |
| Rãnh âm thanh độc lập | Phụ thuộc rãnh âm thanh mà danh sách HLS / DASH tự mang | Thêm danh sách rãnh âm thanh gắn ngoài trong `PlaybackInfo` |
| Nhập liệu của người dùng giữa lần gọi | Không hỗ trợ: trong thời gian thực thi phương thức, plugin không thể bật cửa sổ hỏi | Dùng biểu mẫu của trang plugin (`ui`) hoặc khung xem web; nếu thật sự cần, có thể để `INTERACTION_REQUIRED` mang một biểu mẫu |
| Nơi cất token | Lưu trữ văn bản thuần cấp plugin, phạm vi xác thực do plugin quyết định | Lưu trữ an toàn của hệ thống làm khả năng tùy chọn sau này, không làm tiền đề của việc tái cấu trúc |
| Giá trị mặc định giới hạn tốc độ | API mặc định 5 lần/giây, bùng nổ 15; media xếp hàng độc lập | Tinh chỉnh chính sách host theo vận hành thực tế, không thay đổi khế ước dữ liệu cố định của giao thức |
