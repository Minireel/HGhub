# 08 · Giao diện host

[← Tổng quan](README.md)

Chương này nói rõ các thay đổi của tầng giao diện do việc plugin hóa gây ra. Giao diện không chứa bất kỳ tri thức nguồn dữ liệu nào: nó chỉ gọi lệnh host,
kết xuất dữ liệu plugin trả về theo [05 Mô hình nội dung](05-content-model.md).

## 1. Cấu trúc điều hướng

```text
首页      沉浸式播放器 / 从设置中选定的片源随机播放
分类      顶部按片源分栏 / 分类横向预览 / 全部与筛选
搜索      单选、多选或综合搜索
我的      收藏 / 来自片源 / 历史（追更收在收藏内）
插件      导入、开关、插件生成的表单、线路、日志、自检
设置      应用设置 + 插件相关设置
```

Nửa trên của thanh bên máy tính là trang chủ, danh mục, trung tâm plugin, cài đặt; ô tìm kiếm đặt riêng ở trên cùng; nửa dưới chỉ một lối vào "Thư viện của tôi".
Phía di động tái sử dụng thanh đáy hẹp dạng chữ thuần 40px của bản cũ: trang chủ, tìm kiếm, danh mục, của tôi. Trung tâm plugin đặt ở đầu nội dung của "Của tôi", vào trang cấp hai thì ẩn thanh đáy, quay lại thì khôi phục.

**Trang bảng xếp hạng bị xóa.** Bảng xếp hạng của thượng nguồn không còn trang độc lập: plugin dùng mục sắp xếp của `catalog` (`sorts`)
hoặc một phân khu trong `home` để biểu đạt (xem [06 §6](06-capabilities.md)). Lối vào bảng xếp hạng đã có bị loại bỏ khỏi điều hướng.

## 2. Trang plugin (mới)

Trung tâm plugin trong cùng một trang cung cấp hai phân cột "Đã cài đặt" và "Nhập & đăng ký". Cái trước quản lý nguồn dữ liệu, resolver, tài khoản, cấu hình, trạng thái và nhật ký; cái sau xử lý tệp, địa chỉ, mã nguồn, xem trước danh sách, đăng ký và cập nhật. Việc chuyển phân cột giữ lại nội dung nhập đang điền.

### 2.1 Danh sách

Chia hai nhóm theo `meta.kind`: **nguồn dữ liệu** và **resolver**. Mỗi hàng hiển thị:

| Phần tử | Mô tả |
| --- | --- |
| Tên và phiên bản | `meta.name`, `meta.version`; khi hướng tới giao thức bản cũ thì thêm dấu hiệu |
| Nguồn gốc | Đường dẫn cục bộ hoặc URL lúc nhập; khi đến từ danh sách plugin thì hiển thị tên danh sách |
| Công tắc | Bật / tắt. Sau khi tắt không tải nữa, cũng không xuất hiện trong các lựa chọn của trang chủ, danh mục, tìm kiếm |
| Trạng thái | Đã tắt / đang khởi tạo / sẵn sàng / khởi tạo thất bại / thiếu khả năng bắt buộc / vô hiệu hóa do sự cố runtime liên tiếp / có cập nhật |
| Thao tác | Kiểm tra cập nhật, gỡ bỏ (xóa dữ liệu của nó), vào chi tiết |

Plugin có `mature` là `true` mặc định bị ẩn, sau khi bật "hiển thị nội dung người lớn" trong cài đặt mới xuất hiện.

### 2.2 Nhập

| Cách | Giao diện |
| --- | --- |
| Tệp cục bộ | Bộ chọn tệp, hỗ trợ chọn nhiều `.js` cùng lúc |
| Danh sách plugin cục bộ | Chọn `manifest.json`, hiển thị danh sách mục để chọn |
| URL | Nhập địa chỉ; sau khi tải xuống nhận diện plugin JS hoặc danh sách JSON, không yêu cầu đuôi tệp |
| Dán mã nguồn | Ô văn bản dán mã nguồn plugin và nhập (tiện cho việc sao chép từ chat, diễn đàn) |

Danh sách tệp tương đối của Android thông qua "Chọn thư mục chứa danh sách" trao quyền truy cập thư mục, sau đó chọn danh sách JSON trong đó đi vào cùng một xem trước lựa chọn.
Khi việc chọn thư mục bị hủy thì không khởi động nhập; các mục vượt biên hoặc không tìm thấy hiển thị lỗi độc lập, không ảnh hưởng các mục còn lại.

Khi nhập danh sách plugin, trước hết hiển thị các mục (tên, phiên bản, giới thiệu, đã cài đặt hay chưa, phiên bản đã cài), mặc định chọn toàn bộ, sau khi người dùng xác nhận lần lượt tải xuống nhập;
một cái đơn lẻ thất bại không ảnh hưởng các cái còn lại, kết thúc tổng hợp kết quả. Danh sách đã lưu thành đăng ký, có thể trong đầu trang plugin làm mới bằng một bấm.
Mặc định bật và khởi tạo plugin mới, cũng có thể chọn chỉ cài đặt. Việc nhập thành công và khởi tạo thành công hiển thị riêng, không thể ngụy trang khởi tạo thất bại thành tệp chưa cài đặt.
Plugin đang khởi tạo không tham gia lựa chọn nguồn nghiệp vụ; khi cần xử lý thủ công có thể vào thẳng trang chi tiết, không yêu cầu tìm kiếm trước để kích hoạt việc tải.

### 2.3 Chi tiết plugin

| Khu vực | Nội dung |
| --- | --- |
| Tổng quan | Tên, phiên bản, giới thiệu, tác giả, trang chủ, nguồn gốc, `protocol`, các đặc tính đã khai báo |
| **Biểu mẫu do plugin sinh ra** | §2.4 |
| Tuyến trang | Khi triển khai `endpoints` thì hiển thị: danh sách tuyến, thăm dò khả năng tới được (kích hoạt thủ công), làm mới, chuyển đổi |
| Cài đặt | `meta.settings` kết xuất biểu mẫu; sau khi gửi, host hủy lần gọi của plugin đó, kết thúc các phiên media liên quan và khởi tạo lại |
| Trạng thái và dữ liệu | Dung lượng lưu trữ chiếm, xóa bộ nhớ đệm, xóa dữ liệu (gồm trạng thái đăng nhập) |
| Nhật ký | Nhật ký gần nhất, có thể sao chép xuất ra |
| Tự kiểm tra | Theo khả năng chạy chuỗi VOD, trực tiếp hoặc resolver; khi cần người dùng cung cấp ID mục/trang phát; lần lượt hiển thị đạt, thất bại, bỏ qua và thiếu đầu vào |
| Cập nhật | Kiểm tra cập nhật, cập nhật lên phiên bản mới (khi có phiên bản mới) |

### 2.4 Biểu mẫu do plugin sinh ra

**Plugin khai báo biểu mẫu, host kết xuất, người dùng dùng trong trang plugin.** Đây là cách duy nhất plugin có thể ảnh hưởng giao diện.

Hai loại nguồn gốc, cách kết xuất nhất quán:

| Nguồn gốc | Nội dung |
| --- | --- |
| Khả năng `auth` ([06 §12](06-capabilities.md)) | Cách đăng nhập: mật khẩu tài khoản, mã xác minh, đăng nhập web, v.v. |
| Khả năng `ui` ([06 §15](06-capabilities.md)) | Hành động plugin tự định nghĩa: đổi tài khoản, tham số tùy chỉnh, chọn tuyến, v.v. |

Quy tắc kết xuất:

- Mỗi cách đăng nhập hoặc hành động là một nút bấm (`label` là chữ trên nút, `description` là thuyết minh phía dưới).
- Sau khi bấm triển khai biểu mẫu: các trường kết xuất theo `FormField.type` (ô văn bản, ô mật khẩu, danh sách thả xuống, số, hình mã xác minh, v.v.),
  trường `required` chưa điền thì vô hiệu hóa nút gửi.
- Sau khi gửi gọi `submit` tương ứng:
  - `{ status: "ok" }`: thu lại biểu mẫu, ở đầu trang plugin gợi ý một câu (`message` hoặc "đã hoàn thành").
  - `{ status: "next", form }`: kết xuất biểu mẫu mới ra, các giá trị đã điền ở các bước trước giữ lại và hợp nhất khi gửi.
  - Ném `AUTH_INVALID`: hiển thị thông điệp lỗi trong biểu mẫu, giữ lại nội dung đã điền.
  - Ném `AUTH_REQUIRED`: gợi ý đăng nhập trước.
- Khi đã đăng nhập, khu vực đăng nhập hiển thị tài khoản hiện tại (kết quả của `getProfile`) và "Đăng xuất".
- Người dùng trong "Đã cài đặt" triển khai "Cài đặt & nhật ký" của plugin, là có thể trong "Tài khoản plugin" chọn cách đăng nhập. Cách không có `form` sau khi bấm gọi trực tiếp `auth.submit`. Nếu plugin mở khung xem web, host sau khi quay lại theo `AuthResult` hiển thị tài khoản hoặc biểu mẫu bước tiếp theo. [Quy trình đăng nhập web](04-host-api.md#12-hgwebview-tùy-chọn)
- Biểu mẫu chỉ xuất hiện ở trang plugin; các vị trí khác của giao diện không bật biểu mẫu plugin.

## 3. Trang chủ

Trang chủ là sân khấu nhập vai mở ra là phát được, không hiển thị thác thẻ đề xuất, tiêu đề trang hay phụ đề quảng bá.

### 3.1 Lựa chọn nguồn phát

Lối vào nằm ở **Cài đặt → Nguồn phát trang chủ**, trang chủ không đặt lối vào đổi nguồn. Cung cấp ba chế độ chọn một, chọn nhiều, chọn toàn bộ, lựa chọn được ghi nhớ trên máy này, không ảnh hưởng lẫn nhau với phạm vi tìm kiếm. Chỉ các nguồn phát đã bật và có khả năng phát cùng lấy nội dung mới tham gia; resolver không làm nguồn phát nội dung.

### 3.2 Phát ngẫu nhiên

- Ưu tiên gọi feed.list của nguồn phát; không có feed thì từ home.getSections lấy tác phẩm; chỉ có catalog thì chọn ngẫu nhiên một danh mục lấy mẫu.
- Nội dung được xáo trộn theo nguồn phát và tác phẩm, từ nhiều nguồn phát xây dựng hàng đợi phát. Các nguồn đến độc lập, thất bại không chặn việc phát của nguồn khác.
- Dùng entryEpisodeId hoặc ID danh sách phát / phân tập thật trong chi tiết để bắt đầu phát, hỗ trợ phân tập phân trang; không đoán ID phân tập.
- Phím lên xuống, con lăn hoặc vuốt dọc trên di động để chuyển tác phẩm; hỗ trợ mục trước, mục sau. Mục trùng lặp được khử trùng, con trỏ trùng lặp thì kết thúc, phân trang thất bại có thể thử lại và giữ tác phẩm hiện tại.
- Sau khi kết thúc có chuyển mục tiếp theo hay không tuân theo cài đặt ứng dụng; tiến độ xem và yêu thích lưu vào cục bộ.
- Khi không có nguồn phát, lựa chọn rỗng, tải thất bại cung cấp trạng thái rỗng và lối vào có thể thực thi tương ứng.

### 3.3 Tương tác phát

Tái sử dụng `Feed`, `PlayerControls`, `PlaybackLock`, `useSeekInput`, Sheet ở đáy và cấu trúc nhóm cài đặt của Git `5f88442`; `VideoSurface` đổi sang nối với phiên media host đã chuẩn bị, giữ cùng một giao diện điều khiển phát.

Trang chủ, tìm kiếm, danh mục, yêu thích và lịch sử dùng chung một trình phát, không còn dùng trang chi tiết tác phẩm độc lập. Chi tiết, chọn tập, bình luận trên máy tính vươn ra từ phía phải và nhường không gian video, phía di động dùng ngăn kéo ở đáy. Bản plugin không cung cấp nút chia sẻ.

Phía di động dùng tiếp chạm một lần xóa màn hình, chạm hai lần tạm dừng/phát, nhấn giữ tốc độ 2x, nhấn giữ vuốt lên khóa tốc độ, kéo ngang định vị, chụm hai ngón phóng to thu nhỏ và dịch chuyển, vuốt dọc đổi mục; khi xóa màn hình và toàn màn hình thì ẩn thanh đáy.

Trình phát cung cấp phát / tạm dừng, thanh tiến độ kéo được, âm lượng và tắt tiếng, tốc độ, phụ đề, toàn màn hình, hình trong hình khi nền tảng hỗ trợ, và khóa cảm ứng. Máy tính hỗ trợ Space phát, phím trái phải nhảy, M tắt tiếng, F toàn màn hình.
Thanh điều khiển khi nhàn rỗi mờ dần, di chuyển chuột, chạm hoặc tiêu điểm bàn phím khôi phục hiển thị. Khi khóa thì dừng việc đổi mục bằng cử chỉ và phím tắt. Thông tin tác phẩm và cài đặt tuyến phát giữ ở tầng thứ cấp; phân tập, thêm thông tin và yêu thích cục bộ thao tác từ trong khung hình.

## 4. Trang danh mục

- Thanh nguồn phát cố định ở trên cùng chuyển theo plugin, dù chỉ có một nguồn phát cũng giữ lại.
- Mỗi danh mục tổ chức theo "biểu tượng / tên danh mục / giới thiệu — xem trước ảnh bìa ngang — toàn bộ"; các mục có thể cuộn ngang, cuối cung cấp "xem thêm".
- Bấm "toàn bộ" vào lưới đầy đủ của danh mục đó, giữ danh mục con, bộ lọc, sắp xếp và phân trang con trỏ; cung cấp "toàn bộ danh mục" quay lại tổng quan.
- Xem trước của từng danh mục tải độc lập, thử lại độc lập; việc chuyển nguồn phát hủy các yêu cầu đã hết hạn, tránh trộn vào nguồn khác.
- Các lối vào phân khu đề xuất, trực tiếp, đồ văn và luồng thông tin đơn nguồn gốc đặt dưới nguồn phát tương ứng, tiếp tục hiển thị theo khả năng; không tích hợp tri thức của bất kỳ nền tảng đơn lẻ nào.

## 5. Trang tìm kiếm

- Tái sử dụng thanh tìm kiếm và bố cục thẻ áp phích của phiên bản lịch sử `5f88442`, không còn hiển thị tiêu đề lớn phụ thêm của trang. Nút bên phải thanh tìm kiếm triển khai **phạm vi và lọc**: chọn một plugin, chọn nhiều plugin, hoặc "tổng hợp" (toàn bộ plugin đã bật và triển khai `search`).
- Khi chưa tìm kiếm hiển thị lịch sử tìm kiếm cục bộ (tối đa 20 mục, khử trùng đưa lên đầu, bấm tìm lại, có thể xóa sạch); xóa nội dung nhập có thể quay lại lịch sử. Tìm kiếm hot chỉ hiển thị các từ plugin thực tế trả về, không tích hợp đề tài hay bảng xếp hạng.
- Lựa chọn phạm vi và lựa chọn nguồn của trang chủ độc lập với nhau, mỗi cái ghi nhớ riêng.
- Gợi ý liên tưởng trong quá trình nhập (`suggest`) chỉ nhằm vào plugin đang chọn một đó; ở chế độ tổng hợp không yêu cầu gợi ý.
- Từ khóa tìm kiếm hot (`trending`) hiển thị theo cột plugin.

Hiển thị kết quả:

Kết quả đơn nguồn và tổng hợp đều dùng thẻ áp phích tỷ lệ bản cũ: ảnh bìa, dấu hiệu cập nhật, tiêu đề, tác giả hoặc nguồn phát, và số lượt phát/lượt thích plugin thực tế cung cấp. Thẻ hợp nhất ưu tiên dùng nguồn có ảnh bìa, tải qua phiên hình ảnh host của nguồn đó; máy tính ba tới bốn cột, di động hai cột, giữ khoảng cách gọn.

| Chế độ | Hiển thị |
| --- | --- |
| Chọn một | Trực tiếp hiển thị kết quả phân trang của plugin đó |
| Chọn nhiều | Mỗi plugin một phân cột (nhãn ở trên cùng), mỗi cái phân trang riêng |
| Tổng hợp | Phân cột đầu tiên là "toàn bộ", theo quy tắc của [06 §17.1](06-capabilities.md) hợp nhất cùng một tác phẩm thành một thẻ, trên thẻ ghi nhãn số nguồn; các phân cột tiếp theo là kết quả gốc của từng plugin |

- Sau khi bấm thẻ hợp nhất vào "lựa chọn nguồn": liệt kê các mục của từng nguồn, người dùng chọn định một cái xong trực tiếp vào trình phát thống nhất, chỉ thêm nút quay lại.
  Đây là một trong các lối vào của việc đổi nguồn.
- Kết quả của từng plugin đến độc lập, kết xuất độc lập; một plugin nào đó thất bại chỉ ảnh hưởng phân cột của chính nó.

## 6. Của tôi

Thư viện cung cấp ba phân cột loại trừ lẫn nhau: **Yêu thích, Đến từ nguồn phát, Lịch sử**. Trong yêu thích cung cấp "chỉ xem đang theo dõi" và kiểm tra cập nhật. Thẻ ảnh bìa hiển thị nguồn phát, tiến độ xem hoặc trạng thái nguồn không khả dụng. "Đến từ nguồn phát" mới tải tài khoản và danh sách đám mây, nguồn cần đăng nhập thì sau khi đăng nhập mới tải; thao tác đăng nhập vẫn đi tới trung tâm plugin.

- **Thư viện cục bộ**: Yêu thích, lịch sử, tiến độ. Dữ liệu ở cục bộ, khóa là "ID plugin + ID mục", không phụ thuộc đăng nhập.
  Khi plugin mà mục thuộc về bị gỡ bỏ thì giữ lại và đánh dấu "nguồn không khả dụng".
- **Theo dõi cập nhật**: Tác phẩm được đánh dấu đang theo dõi trong thư viện, dùng `updates.check` kiểm tra theo lô, khi có cập nhật hiển thị huy hiệu (`latestLabel`).
- **Khu vực tài khoản của từng plugin**: Plugin triển khai `auth` mỗi cái chiếm một khối, hiển thị trạng thái đăng nhập (`getProfile`) và danh sách đám mây
  (`library`: yêu thích / lượt thích / lịch sử / theo dõi), và cung cấp "nhập vào thư viện cục bộ".
- Việc đổi nguồn thiết lập liên kết nguồn, giữ lại yêu thích cũ và tiến độ cũ; ưu tiên căn theo số mùa/tập rõ ràng, khi có mơ hồ thì chọn tập, số giây phát do người dùng xác nhận có dùng lại hay không.
- 0.0.1 làm trình phát plugin hoàn toàn mới bắt đầu từ thư viện rỗng, không đọc hay di trú thư viện của nền tảng cũ.

## 7. Trình phát thống nhất và ngăn kéo chi tiết

- **Danh sách phân tập**: Khi `playlists` nhiều hơn một mục thì cung cấp chuyển đổi (như "Tuyến 1", "Lồng tiếng", "Phần phụ"), theo số mùa/tập thật cố gắng giữ vị trí, khi mơ hồ thì chọn tập.
  Phân tập mang `season` thì nhóm theo mùa.
- **Tuyến phát**: Khi `PlaybackInfo.servers` nhiều hơn một mục, trong trình phát cung cấp chuyển đổi; chuyển đổi tức là mang `serverId`
  phân giải lại. Tuyến nguồn và tuyến resolver hiển thị tách biệt, lưu lựa chọn riêng; việc thử lại có giới hạn sau sự cố tuân theo [09 §5](09-media-transport.md), không luân phiên vô hạn.
- **Luồng cần resolver**: Khi `Stream.kind` là `page` và không có resolver khả dụng, gợi ý "nguồn này cần cài resolver",
  và cung cấp lối vào nhảy tới trang plugin; khi có nhiều resolver khả dụng thì để người dùng chọn, có thể chọn "ghi nhớ lựa chọn cho trang này".
- **Phát bên ngoài**: Khi `Stream.kind` là `external`, sau khi hỏi người dùng dùng hệ thống hoặc ứng dụng khác mở.
- **Đổi nguồn**: Ngăn kéo "Cài đặt phát" của trình phát cung cấp lối vào "đổi nguồn", đi theo quy trình của [06 §17.2](06-capabilities.md).
- **Phụ đề**: Khi `subtitles` nhiều hơn một thì cung cấp lựa chọn; cái có `default` là `true` mở mặc định.
- **Yêu thích**: Yêu thích cục bộ luôn khả dụng, không phụ thuộc `interact`; yêu thích đám mây, lượt thích, theo dõi lần lượt hiển thị theo phương thức tương ứng có tồn tại hay không, khi chưa đăng nhập hướng dẫn tới trang plugin.
- **Nguồn thuần trực tiếp**: Từ danh sách kênh vào phát, không gọi phương thức media không tồn tại; tuyến kênh, resolver, làm mới dùng chung cơ chế phiên media.
- **Thay đổi phiên**: Cập nhật, vô hiệu hóa, gỡ bỏ plugin hoặc sửa cài đặt/tuyến trang của nó sẽ dừng việc phát liên quan và gợi ý phát lại; cập nhật tự động thực thi sau khi việc phát kết thúc.

## 8. Trang cài đặt

| Phân loại | Hạng mục |
| --- | --- |
| Ngoại hình và phát | Nguồn phát trang chủ (chọn một / chọn nhiều / chọn toàn bộ), chủ đề, tắt tiếng khi khởi động, ghi nhớ âm lượng, tùy chọn độ nét mặc định, tự động phát mục tiếp theo |
| Mạng | Chế độ bảo mật nghiêm ngặt (nói rõ tác dụng lên API plugin và yêu cầu media của host, khung xem web dùng ngăn xếp mạng nền tảng độc lập), chẩn đoán mạng |
| Plugin | Hiển thị nội dung người lớn (`mature`), tự động kiểm tra cập nhật plugin khi khởi động, các plugin mặc định tham gia tìm kiếm tổng hợp, tùy chọn resolver (lựa chọn ghi nhớ theo trang, có thể xóa sạch) |
| Lưu trữ | Dọn dẹp bộ nhớ đệm (gồm bộ nhớ đệm plugin), dung lượng dữ liệu chiếm |
| Về | Phiên bản, kiểm tra cập nhật, giấy phép mã nguồn mở, xuất nhật ký |

Điều chỉnh:

- **Loại bỏ** các hạng mục gắn với một nguồn dữ liệu đơn lẻ: lựa chọn tuyến (chuyển tới trang plugin), thông tin tài khoản (chuyển tới "Của tôi" và trang plugin).
- Các hạng mục "Mạng" và "Bảo mật" gốc được giữ lại, cách diễn đạt nói rõ phạm vi tác dụng thực tế của chúng đối với API plugin, media và khung xem web.

## 9. Các lệnh host giao diện dùng

Giao diện chỉ tương tác với host qua các lệnh này, không tiếp xúc trực tiếp đối tượng plugin.

| Lệnh | Công dụng |
| --- | --- |
| `plugin_call(pluginId, method, params, requestId)` | Gọi phương thức khả năng đã công bố, requestId do giao diện sinh trước dùng cho hủy bỏ và liên kết nhật ký |
| `plugin_list()` | Danh sách plugin: siêu thông tin, trạng thái bật, trạng thái khởi tạo, nguồn gốc, khả năng khả dụng và phương thức cụ thể, khả năng bắt buộc còn thiếu |
| `plugin_import(source)` | Nhập: đường dẫn cục bộ, URL, mã nguồn đã dán, danh sách plugin |
| `plugin_set_enabled(id, enabled)` / `plugin_remove(id)` | Công tắc và gỡ bỏ |
| `plugin_check_update(id?)` / `plugin_apply_update(id)` | Kiểm tra và thực thi cập nhật |
| `plugin_form_actions(id)` / `plugin_form_submit(id, target, values)` | Biểu mẫu trang plugin: `target` là id phương thức của `auth` hoặc id hành động của `ui` |
| `plugin_settings_get(id)` / `plugin_settings_set(id, key, value)` | Cài đặt plugin |
| `plugin_logs(id)` / `plugin_self_test(id)` | Nhật ký và tự kiểm tra |
| `plugin_retry_init(id)` | Thử lại khởi tạo hoặc kiểm tra lại sau khi bổ sung đủ khả năng bắt buộc |
| `endpoint_list(id, refresh?)` / `endpoint_set(id, endpointId)` / `endpoint_probe(id)` | Tuyến trang |
| `resolver_preference_get()` / `resolver_preference_set(host, pluginId)` | Tùy chọn resolver |
| `library_*` / `history_*` / `progress_*` | Thư viện cục bộ và tiến độ |
| `media_session_register(...)` | Nội bộ: đăng ký `ImageRef` / `Stream` thành địa chỉ media cục bộ, cho `<img>` và trình phát dùng |
| `plugin_call_cancel(requestId)` | Hủy lần gọi dùng chung; việc đổi từ tìm kiếm, rời trang, v.v. dùng cùng một cơ chế |
| `media_session_release(sessionId)` | Giải phóng phiên media, hủy yêu cầu tài nguyên không ai dùng |

Tìm kiếm tổng hợp và hàng đợi phát trang chủ do giao diện phát khởi song song nhiều `plugin_call` thực hiện, kết quả của từng nguồn kết xuất độc lập;
việc giới hạn tốc độ và khử trùng phía host trong suốt đối với giao diện (xem [03 §3](03-host-rules.md)).
`plugin_call` chấp nhận requestId do giao diện sinh trước, và trong nhật ký, hủy bỏ và trả về của cùng một lần gọi dùng nó;
chỉ cho phép các phương thức khả năng đã công bố, không cho phép qua đường dẫn đối tượng tùy ý truy cập factory, dispose hoặc đối tượng nội bộ của host.
