# Khả năng và API của host

## Chọn khả năng

| Kịch bản | Phương thức tối thiểu | Có thể thêm |
| --- | --- | --- |
| Video theo yêu cầu | media.getDetail, media.resolvePlayback | search, home, catalog, phân trang phân tập |
| Chỉ phát trực tiếp | live.getGroups, live.listChannels, live.resolve | Chương trình, đồ văn |
| Phân giải trang phát | resolver.resolve, meta.kind là resolver | matches, tài khoản, biến đổi tài nguyên |
| Đồ văn | articles.list, articles.getDetail | source hiện tại vẫn phải có media hoặc live |
| Đăng nhập | Cả bốn phương thức của auth | Biểu mẫu nhiều bước, đăng nhập web khi hỗ trợ `webview` |

Khai báo namespace thì phải triển khai các phương thức bắt buộc của nó, đừng dùng hàm rỗng để giả vờ hỗ trợ; bỏ qua các trường params không quen. Plugin chỉ trả về dữ liệu, không thể vẽ UI. Khi cần nhập liệu, dùng biểu mẫu cài đặt hoặc auth/ui. Xem chi tiết [Mô-đun khả năng](../plugin-protocol/06-capabilities.md).

Khi `hg.env.features.includes("webview")` là đúng, `auth.getMethods` có thể trả về cách đăng nhập web không có `form`. Sau khi người dùng bấm, trong `auth.submit` gọi `hg.webview.open`, dùng `doneWhen` chỉ định địa chỉ hoặc tên Cookie sau khi trang đăng nhập xong. Nếu cung cấp cả hai điều kiện, cả hai đều phải thỏa mãn. Sau khi cửa sổ hoàn tất chỉ trả về `finalUrl`, plugin lại dùng `hg.http.fetch` đọc giao diện tài khoản cùng origin, dựa trên ID người dùng và biệt danh mà trang trả về để tạo `UserProfile`. [Tham số và ví dụ](../plugin-protocol/04-host-api.md#12-hgwebview-tùy-chọn)

Host cách ly phiên đăng nhập web theo plugin và origin HTTPS, đồng thời xử lý `Set-Cookie` của các phản hồi tiếp theo. `hg.cookies.get/set` chỉ truy cập Cookie thông thường, localStorage của trang ở lại trong khung xem tạm thời. Khi mở lại khu vực tài khoản, `auth.getProfile` có thể khôi phục hiển thị tài khoản qua cùng giao diện đó. Khi đăng xuất, trong `finally` gọi `hg.cookies.clear({ url })` để dọn phiên cục bộ của trang đó.

## Gọi host

| Mục đích | Mô-đun | Lưu ý |
| --- | --- | --- |
| Mạng | hg.http | Không phải 2xx thì trả về phản hồi, plugin ánh xạ thành lỗi nghiệp vụ |
| Lưu trữ bền vững | hg.storage | 5 MiB/plugin, 256 KiB/giá trị, hỗ trợ TTL |
| Cookie | hg.cookies | Mỗi plugin một kho riêng, không thể chia sẻ tài khoản của nguồn khác |
| Tóm tắt/mã hóa | hg.crypto | HMAC, AES, mã hóa khóa công khai RSA, số ngẫu nhiên; phạm vi hỗ trợ RSA xem bên dưới |
| Mã hóa/giải mã | hg.codec | UTF-8, Base64, Hex, charset legacy |
| HTML | hg.html | Node chỉ dùng trong lần gọi hiện tại, không thể làm giá trị trả về nghiệp vụ |
| Cài đặt | hg.settings | Sau khi lưu, host xây dựng lại plugin |
| Nhật ký | hg.log, console | Nhật ký cục bộ, không ghi mật khẩu, Cookie, URL đã ký |

Factory chỉ xây dựng đối tượng và đọc cài đặt, không làm thao tác mạng, lưu trữ, Cookie hay HTML, cũng không thể tạo bộ hẹn giờ. Có thể trả về Promise; từ khi bắt đầu thực thi factory đến khi hoàn tất khám phá khả năng dùng chung ngân sách 5 giây, một đoạn thực thi đồng bộ tối đa 1 giây. await và microtask không đặt lại tổng ngân sách, Promise không bao giờ hoàn thành sẽ tải thất bại với TIMEOUT. Phương thức nhận `(params, ctx)`; đọc thường 15 giây, phát thường 20 giây. Truyền ctx.signal cho HTTP. Sau khi lần gọi kết thúc, nhiệm vụ được dọn dẹp, không có thực thi nền tự do.

Các GET ẩn danh đồng thời của cùng một plugin được host hợp nhất theo URL thực tế (gồm query) và header yêu cầu hiệu lực; không cần bật `cache`. Mặc định `credentials: "include"` nhưng khi không có Cookie khớp cũng có thể hợp nhất; yêu cầu mang Cookie hoặc Authorization được gửi độc lập. Mỗi lần gọi giữ timeout và signal của riêng mình; một lần gọi bị hủy không ảnh hưởng những bên chờ khác, sau khi tất cả thoát thì dừng yêu cầu thượng nguồn. Các giới hạn thân phản hồi, chế độ chuyển hướng hoặc chính sách Cookie khác nhau vẫn độc lập. Thử lại, đệm phản hồi và hợp nhất khi đang bay là các cơ chế khác nhau, đừng gắn signal của một lần gọi vào Promise chia sẻ xuyên lần gọi; hãy để host chịu trách nhiệm hợp nhất mạng và hủy bỏ.

API mặc định mỗi plugin đồng thời 6, cùng host tối đa 4, trong lúc đọc thân phản hồi vẫn chiếm giấy phép. Media dùng hàng đợi độc lập; khi nhận `RATE_LIMITED` hãy tôn trọng `retryAfterSec`, host sẽ làm nguội host đó xuyên plugin và hàng đợi theo phản hồi 429/503 thực tế.

Mỗi plugin dùng một QuickJS độc lập, các lần gọi có thể xen kẽ giữa các await. Đừng đặt tham số yêu cầu hiện tại hoặc signal vào biến closure chia sẻ; trạng thái cần bền vững thì dùng lưu trữ.

Dữ liệu nghiệp vụ cho phép đối tượng thông thường, mảng, số hữu hạn, chuỗi, boolean, null, Uint8Array. Thuộc tính undefined của đối tượng bị loại bỏ; mảng không thể có undefined/ô trống. Không được trả về Date, Map, thể hiện lớp, hàm hoặc vòng tham chiếu. Tổng giá trị tối đa 8 MiB, sâu 32 tầng.

## Nhật ký và gỡ bỏ

Nhật ký plugin cùng với cảnh báo kiểm tra và lỗi runtime của host dùng chung hạn mức mỗi phút 240 dòng, 2000 dòng gần nhất. Mỗi dòng gồm JSON UTF-8 của details tối đa 2 KiB; văn bản và details quá dài sẽ bị cắt, khi đọc nhật ký có thể thấy số lượng bị loại bỏ do vượt tốc độ. Ưu tiên ghi requestId, mã lỗi và đường dẫn trường, tránh ghi cả đoạn phản hồi thượng nguồn.

`dispose()` ở tầng cao nhất là tùy chọn, chỉ dùng để giải phóng tham chiếu bộ nhớ một cách đồng bộ hoặc ghi nhật ký, thực thi tối đa 1 giây. Lúc này không có phạm vi gọi nghiệp vụ, không thể nối mạng, lưu trữ bền vững, đọc ghi Cookie, thao tác HTML hay tạo bộ hẹn giờ, cũng không chờ Promise được trả về. Sự cố runtime có thể bỏ qua thông báo, trạng thái bền vững nên được lưu trong các lần gọi nghiệp vụ bình thường.

## Mã hóa khóa công khai RSA

`hg.crypto.rsaEncrypt` là API bắt buộc đồng bộ, chấp nhận PEM SPKI (`PUBLIC KEY`) hoặc PKCS#1 (`RSA PUBLIC KEY`) 1024–8192 bit. Phải chỉ định `pkcs1`, `oaep-sha1` hoặc `oaep-sha256`, đầu vào và đầu ra đều là `Uint8Array`; văn bản trước hết dùng `hg.codec.utf8Encode`.

```js
const ciphertext = hg.crypto.rsaEncrypt(
  { publicKey: publicKeyPem, padding: "oaep-sha256" },
  hg.codec.utf8Encode(password),
);
const encoded = hg.codec.base64Encode(ciphertext);
```

Cách đệm do giao diện thượng nguồn quyết định. MGF1 của OAEP dùng cùng hàm băm, label rỗng; đệm ngẫu nhiên do nguồn ngẫu nhiên của hệ điều hành cung cấp, bản mã của cùng một bản rõ thường khác nhau. Mỗi lần chỉ mã hóa một khối RSA, không tự động chia khối: giới hạn của khóa 2048 bit lần lượt là 245, 214, 190 byte, tính theo số byte sau khi mã hóa UTF-8. Khi định dạng, kiểu, phạm vi khóa hoặc độ dài bản rõ không hợp lệ thì ném `INVALID_ARGUMENT`, không cung cấp API giải mã bằng khóa riêng.

Plugin phụ thuộc vào phương thức này và không thể hạ cấp nên khai báo `meta.requires: [{ type: "api", name: "crypto.rsaEncrypt" }]`; các bản phát triển cũ sẽ đánh dấu không tương thích trước khi bật. Host giả trong bộ nhớ phải tiêm bộ chuyển đổi mã hóa, tính đúng đắn của thuật toán dùng host Rust/QuickJS thật để kiểm chứng.

## Phát

Trước hết đọc `hg.env.media`, phân giải phát chọn ứng viên theo thứ tự của `params.codecs`, không đoán khả năng hỗ trợ H.264/HEVC/AV1 theo tên nền tảng. Client hiện tại thăm dò đường đi thực tế của video thông thường và HLS trong WebView trước khi khởi tạo plugin; danh sách codec lấy giao thoa bảo thủ của các đường đi đã bật, có thể rỗng. API khả năng chỉ dùng để chọn ứng viên, không đảm bảo mọi tổ hợp profile, độ phân giải và rãnh âm thanh; nên giữ luồng dự phòng khả dụng. Không có đặc tính giao thức mới, DASH vẫn không dùng được.

Trả về `streams`, MP4 dùng progressive, HLS dùng hls. Luồng trang là kind:page, do resolver xử lý. Hình ảnh dùng ImageRef, mọi tài nguyên đi qua phiên media của plugin sở hữu.

Tuyến dự phòng dùng servers; theo serverId chỉ phân giải tuyến hiện tại. Thời hạn hiệu lực là mili giây Unix; reason:refresh phải cập nhật địa chỉ. Đừng dùng index để đoán cùng một tập giữa các nguồn, cũng đừng dùng resumeSec của thượng nguồn ghi đè tiến độ cục bộ của host.

Chữ ký đầy đủ xem ở [API của host](../plugin-protocol/04-host-api.md), [Mô hình nội dung](../plugin-protocol/05-content-model.md) và [Truyền tải media](../plugin-protocol/09-media-transport.md). Biểu diễn kiểu là khế ước thiết kế, khoảng trống thực tế xem ở [Trạng thái hỗ trợ](status.md).
