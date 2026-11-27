# Gỡ lỗi và nghiệm thu

Đọc nhật ký trong "Cài đặt & nhật ký" của plugin, trước hết xác nhận khởi tạo, rồi mới gọi phương thức cụ thể. Ghi lại requestId, mã lỗi, đường dẫn trường, tránh ghi thông tin xác thực. Hiện chưa có tự kiểm tra tích hợp đầy đủ và xuất tệp nhật ký; nhật ký có thể chọn và sao chép.

| Triệu chứng | Kiểm tra |
| --- | --- |
| Cài xong chạy thất bại | I/O của factory, phương thức bắt buộc, requires |
| Tìm kiếm rỗng | Ánh xạ trường, kiểm tra các mục, ID thật |
| Trang thứ hai thất bại | Con trỏ, trạng thái closure, hủy bỏ |
| Hình ảnh hoặc video 403 | Header yêu cầu tài nguyên, nhà cung cấp Cookie, địa chỉ CDN |
| Làm mới không hiệu lực | reason:refresh, bộ nhớ đệm hết hạn, thời hạn hiệu lực mili giây |
| Luồng trang không phát được | Resolver khớp, trang có bị gắn nhầm là MP4 không |
| Danh sách Android thất bại | Mục tương đối trước hết dùng "Chọn thư mục chứa danh sách" để ủy quyền, kiểm tra tên tệp, thư mục con và có vượt biên không; ủy quyền đơn tệp không bao gồm tệp anh em |

## Kiểm thử khả năng phát

`hg.env.media` là ảnh chụp tải do frontend phát thực tế đăng ký. Khi chạy riêng host Rust/stdio, codecs/formats mặc định đều rỗng; kiểm thử dùng bộ chuyển đổi stdio có thể trước hết gọi hành động của host `host.media.configure`, tham số ví dụ `{ "codecs": ["h264"], "formats": ["hls", "progressive"] }`. Đây là giao diện kiểm thử của host, không phải API hg mà plugin gọi được. Host giả trong bộ nhớ thông qua `createMockHost({ env: { media: ... } })` tiêm khả năng kiểm thử một cách rõ ràng, không thể dựa vào đó tuyên bố thiết bị hỗ trợ.

`node scripts/test-media-capabilities.mjs` kiểm tra lựa chọn đường đi, giá trị trả về của API, giao thoa codec, chờ đăng ký khi khởi động và thử lại khi thất bại; Rust `tests/media_capabilities.rs` kiểm tra ảnh chụp QuickJS thật, tham số VOD/trực tiếp/resolver, đăng ký trùng và thay đổi khả năng. Chúng không thực hiện giải mã mẫu video hay nghiệm thu UI.

## Kiểm chứng trong kho

Chuyên đề nhập thư mục: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test distribution --test automatic_updates` bao quát bộ chuyển đổi URI không trong suốt, chỉ đọc các mục đã chọn, định danh không khớp, cách ly lỗi, hủy xem trước và hồi quy cập nhật. Sau khi chạy `node scripts/sync-android-network.mjs`, trong `src-tauri/gen/android` thực thi `./gradlew.bat :app:testX86_64DebugUnitTest -x :app:rustBuildX86_64Debug` để kiểm chứng đường đi tiếng Trung/mã hóa/thư mục con của Android và từ chối vượt biên. Trên thiết bị còn kiểm chứng ủy quyền hệ thống, hủy lựa chọn, các nhà cung cấp tệp khác nhau, ủy quyền lại sau khởi động lại và cài đặt giữ dữ liệu.

Chuyên đề nhật ký/gỡ bỏ: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib native::logs` kiểm tra 240 dòng/phút, giữ 2000 dòng, giới hạn 2 KiB mỗi dòng gồm details và đếm số bị loại bỏ; `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test log_limits` kiểm tra nhật ký plugin, cảnh báo kiểm tra và lỗi nghiệp vụ dùng chung hạn mức. `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib runtime::lifecycle_tests` kiểm tra thông báo gỡ bỏ bình thường, cấm I/O, thời gian ngừng hoạt động khi ném lỗi/vòng lặp vô hạn/Promise treo, và bỏ qua thông báo với runtime đã bị ngắt.

Chuyên đề khởi tạo: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test runtime_initialization` kiểm tra Promise treo và microtask liên tục đều bị giới hạn bởi tổng ngân sách 5 giây, nhiều đoạn đồng bộ ngắn có thể cộng dồn vượt 1 giây, vòng lặp đồng bộ vô hạn bị ngắt, và factory không gọi được HTTP/lưu trữ/Cookie/HTML/bộ hẹn giờ. `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib runtime::initialization_tests` kiểm tra sau khi hủy khởi tạo, luồng giải phóng tài nguyên.

Chuyên đề tầng cao nhất của module: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test module_initialization` kiểm tra await tầng cao nhất bị treo, microtask liên tục, vòng lặp đồng bộ vô hạn và biên thời gian của nhiều đoạn ngắn, đồng thời kiểm tra xem trước và chỉ cài đặt không chạy factory. Việc định giá module và kiểm tra siêu thông tin dùng chung 5 giây; khi bật, factory có thêm ngân sách 5 giây.

Chuyên đề điều độ mạng: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib network::scheduling_tests` dùng phản hồi HTTP chậm thật để kiểm tra hợp nhất GET ẩn danh, query thực tế/header hiệu lực, hủy và timeout độc lập, cách ly yêu cầu xác thực, giới hạn đồng thời thân phản hồi 4 mỗi host/6 mỗi plugin, và hàng đợi media độc lập cùng làm nguội chia sẻ. `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test media_scheduling --test playback_performance` kiểm tra phát tiền cảnh ngắt tải trước, bỏ qua tải trước mới trong lúc xếp hàng/truyền tải, khôi phục tải trước sau khi kết thúc, và tái sử dụng bộ nhớ đệm MP4/HLS.

Chuyên đề làm mới chia sẻ: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test transport shared_refresh` kiểm tra cùng phiên bản đồng thời chỉ phân giải một lần, hủy một bên chờ và toàn bộ bên chờ, giải phóng phiên, giữ ánh xạ cũ và thử lại khi thất bại, tài nguyên con HLS cũ mất hiệu lực và định địa chỉ phiên bản mới. `node scripts/test-playback-state.mjs` kiểm tra cùng một phiên đổi phiên bản sẽ tải lại video và khôi phục số giây hiện tại, yêu cầu phát cũ không thể ghi đè trạng thái mới; `node scripts/test-next-episode.mjs` kiểm tra tái sử dụng và hủy bỏ của chuẩn bị phát nối tiếp. Tất cả đều không thay thế nghiệm thu tính liên tục phát trên thiết bị.

Chuyên đề luồng bên ngoài nằm trong `tests/playback_selection.rs`: sáu giao thức được phép, phát bình thường/tải trước/làm mới đều chỉ trả về thông tin xác nhận, không tạo phiên media đang hoạt động, không rò rỉ header và Cookie; từ chối địa chỉ script/file/data và tài khoản mật khẩu trong URL, plugin không thể giả mạo mục tiêu xác nhận của host. Trên thiết bị còn kiểm chứng xác nhận, hủy bỏ, thiếu chương trình xử lý, quay lại ứng dụng sau khi mở bên ngoài; biên dịch hoặc kiểm thử host không bằng nghiệm thu khởi động hệ thống.

Chuyên đề phụ đề: `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib media::subtitles` kiểm tra trục thời gian SRT, UTF-8, BOM UTF-16, charset GBK và đầu vào không hợp lệ; `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test transport srt_tracks` kiểm tra từ QuickJS thật đến proxy media về khai báo định dạng, siêu thông tin rãnh, Range/HEAD sau chuyển đổi, rãnh hỏng không ảnh hưởng video và giải phóng phiên. Trên thiết bị còn cần kiểm chứng phụ đề mặc định, chuyển nhiều rãnh, tắt phụ đề và video tiếp tục phát; kiểm thử chuyên đề không đại diện nghiệm thu thiết bị.

```powershell
pnpm install --frozen-lockfile
pnpm build
$env:CARGO_INCREMENTAL = '0'
cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --locked --lib --tests -j 2
cargo build --manifest-path src-tauri/crates/hpp/Cargo.toml --locked --example dev_host -j 2
node scripts/test-plugins.mjs
pnpm docs:build
pnpm docs:test
```

Bộ kiểm thử trình duyệt dùng Rust/QuickJS thật và React bản sản xuất, chỉ dùng stdio thay cho Tauri IPC; media do cục bộ tổng hợp. Nó không bằng nghiệm thu cửa sổ native hoặc trang thực tế. Trong lúc kiểm thử đừng build lại dev_host.exe của Windows.

Trải nghiệm phát có thể chạy riêng `pnpm test:experience` (hoặc trên bản build dev_host đã có, chạy `node scripts/test-plugins.mjs --experience`). Bao quát xem trước nhẹ ở trang chủ, xác nhận chi tiết, phân trang chọn tập, tiếp nối phiên xem trước, phát toàn cửa sổ trên máy tính, điều khiển di động và khôi phục khi thoát; các lối vào danh mục, tìm kiếm, yêu thích và lịch sử nên hiển thị chi tiết trước, không tự động phát. Ảnh chụp màn hình xuất ra `output/playwright/watch-experience/`. Hồi quy tìm kiếm, cài đặt và cử chỉ di động gốc chạy `pnpm test:frontend`.

Kiểm tra trước phát hành: chọn cài đơn tệp và danh sách, nhập trùng, nhất quán phiên bản; tìm kiếm/trang chủ → chi tiết → phân tập thật → giải mã thực tế; trang thứ hai, chuyển tuyến, kéo, làm mới, hủy bỏ; tải lại và lưu trữ bền vững. Mỗi nền tảng tuyên bố hỗ trợ đều phải kiểm chứng chuỗi đầy đủ từ phiên cục bộ đến trình phát.

Host giả dùng cho ánh xạ trường, không thể chứng minh cách ly, hạn ngạch, Cookie, proxy media hay giải mã native. Báo cáo ghi riêng kết quả mô phỏng, host thật, thượng nguồn và kiểm chứng thiết bị.

## Đăng nhập web

`createMockHost({ webview })` có thể tiêm fixture đăng nhập trả về `{ finalUrl }`. Host giả sẽ đặt đặc tính `webview` theo tiêm này, phù hợp để kiểm thử ẩn hiện cách đăng nhập và phản hồi tài khoản của trang.

`cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --test web_login` dùng QuickJS thật, IPC biểu mẫu xác thực và cửa sổ đăng nhập mô phỏng để kiểm chứng hành động của người dùng, timeout, hủy bỏ và khôi phục ngân sách. `cargo test --manifest-path src-tauri/crates/hpp/Cargo.toml --lib network::web_login_tests` kiểm tra cách ly phiên, yêu cầu cùng origin, đăng xuất và đổi tài khoản.

Trên client hỗ trợ đăng nhập web, có thể nghiệm thu trang thật theo các bước sau.

1. Trong trang plugin mở "Cài đặt & nhật ký", ở "Tài khoản plugin" chọn đăng nhập web, xác nhận đầu cửa sổ hiển thị trang dự kiến.
2. Hoàn tất đăng nhập tại trang. Cửa sổ có thể tự động quay lại sau khi thỏa mãn `doneWhen`. Nếu cửa sổ Windows vẫn mở, bấm "Đóng" ở trên cùng hoặc dấu × trên thanh tiêu đề, host sẽ đối chiếu điều kiện hoàn tất trước khi đóng cửa sổ.
3. Quay lại "Tài khoản plugin" đối chiếu biệt danh, thu gọn rồi mở lại "Cài đặt & nhật ký", xác nhận `auth.getProfile` vẫn lấy lại được cùng một tài khoản từ giao diện cùng origin.
4. Bấm "Đăng xuất", xác nhận khu vực tài khoản hiển thị lại các cách đăng nhập. Khởi phát một lần đăng nhập nữa, lần này bấm "Đóng" trước khi điền tài khoản, xác nhận thao tác này kết thúc bằng `CANCELLED`.

Môi trường nghiệm thu thực tế của Windows và Android xem ở [Phạm vi triển khai](../plugin-protocol/IMPLEMENTATION.md).

## Proxy và bản release Android

Cài đặt → Mạng → Truy cập proxy hỗ trợ theo hệ thống, tùy chỉnh và kết nối trực tiếp. Tùy chỉnh hỗ trợ HTTP, HTTPS, SOCKS; sau khi lưu, yêu cầu mới có hiệu lực, video đã đệm sẽ không tải lại, khi cần thì làm mới phát. Kiểm tra kết nối chỉ dùng bản nháp hiện tại, không lưu. Kết nối trực tiếp tắt proxy tầng ứng dụng, nhưng không vượt qua VPN / TUN của hệ điều hành.

Chế độ hệ thống dùng giá trị ghi đè `HGHUB_PROXY` hoặc proxy cố định của nền tảng, biến môi trường proxy; Android đọc thuộc tính proxy HTTP của hệ thống, Linux còn hỗ trợ proxy thủ công của GNOME. Script cấu hình tự động PAC không nằm trong phạm vi hỗ trợ hiện tại. Lựa chọn proxy vẫn giữ kiểm tra địa chỉ đích và chuyển hướng.

Ảnh bìa và media của Android được cung cấp qua dịch vụ vòng lặp HTTP trong ứng dụng. `scripts/sync-android-network.mjs` trước khi build đồng bộ cấu hình mạng chịu quản lý phiên bản vào project được sinh, chỉ cho phép kết nối văn bản thuần `127.0.0.1` và `localhost`. Sau khi thực thi lại Android init cũng cần đồng bộ; khi nghiệm thu bản release, kiểm tra manifest và XML trong APK, không thể chỉ dựa vào debug phát được mà phán đoán release bình thường.

Cùng script đồng bộ đó cũng sao chép `OrientationPlugin.kt` của Android. Trình phát thông qua lệnh `set_orientation` đã đăng ký để chuyển hướng hệ thống của Activity, khi thoát khôi phục cài đặt hướng trước khi vào; màn hình ngang không còn phụ thuộc vào xoay CSS. Trình duyệt chỉ mô phỏng được thay đổi kích thước khung xem, hành vi hướng hệ thống và thanh trạng thái cần xác nhận trên Android.

Cấu hình phát triển của manifest Cargo chính của ứng dụng tắt biên dịch tăng dần, để tránh khi hệ thống tệp E: hiện tại không hỗ trợ liên kết cứng xuất hiện ký hiệu chưa phân giải của đối tượng tăng dần cũ. Nếu đã có sản phẩm bị hỏng, có thể trước hết thực thi `cargo clean --manifest-path src-tauri/Cargo.toml -p hghub-hpp`, rồi chạy `pnpm app:dev`.
