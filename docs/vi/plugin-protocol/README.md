# Giao thức plugin HGhub (HPP) 1.0 · Dự thảo

> Trạng thái: **Dự thảo đường cơ sở tái cấu trúc**. Việc triển khai tiến hành theo tài liệu này; ở P0 đồng thời cho ra định nghĩa kiểu, ví dụ hư cấu và host giả, sau khi kiểm chứng vòng khép kín cốt lõi sẽ đóng băng 1.0.

Phạm vi đã triển khai, việc cần làm và lệnh kiểm chứng của bản phát triển Windows xem ở [Tiến độ triển khai](IMPLEMENTATION.md).

## 1. Định vị

HGhub là một **trình xem video đa năng**: phim điện ảnh, phim bộ, anime, chương trình tạp kỹ, phim ngắn, phát trực tiếp, dùng cùng một bộ giao diện để trình bày.
Bản thân nó **không chứa bất kỳ nguồn nội dung nào** — tìm gì, lấy từ đâu, phân giải thế nào, toàn bộ do plugin JS mà người dùng tự nhập cung cấp.

HPP (HGhub Plugin Protocol) định nghĩa thỏa thuận giữa plugin và host: tệp plugin trông như thế nào, nhập ra sao,
chạy trong môi trường nào, có thể gọi những gì, phải triển khai những gì.

## 2. Biên giới trách nhiệm

- HGhub **không tích hợp, không phân phối, không xét duyệt** bất kỳ plugin nào, cũng không cung cấp danh sách plugin chính thức.
- Plugin là mã của bên thứ ba do người dùng tự nhập. Nó truy cập những trang nào, xử lý tài khoản do người dùng nhập vào ra sao,
  do tác giả plugin và người sử dụng tự chịu trách nhiệm, host không bảo chứng.
- Vì vậy giao thức **không đặt** các cơ chế như ký, xét duyệt, xác nhận quyền, thu hồi.

Host chỉ gánh những việc liên quan đến sự ổn định của chính nó: plugin chạy trong một JS engine độc lập, không chạm được vào hệ thống tệp và giao diện nội bộ của ứng dụng;
vòng lặp vô hạn và vượt bộ nhớ của script plugin do host ngắt xử lý, không chặn giao diện và các plugin khác; không cam kết cách ly sự cố cấp tiến trình của chính engine native (xem [03](03-host-rules.md)).

## 3. Mục tiêu

| # | Mục tiêu | Điểm rơi |
| --- | --- | --- |
| G1 | **Lõi không tích hợp gì**: Trong mã, tài liệu, kiểm thử của host không xuất hiện bất kỳ tên miền, giao diện, khóa, logic phân giải nào của thượng nguồn | Toàn bộ |
| G2 | **Dễ phân phối**: Sản phẩm giao là một tệp `.js` duy nhất, có thể nhập trực tiếp hoặc nhập theo lô qua danh sách, không phụ thuộc dịch vụ trung tâm | [01](01-plugin-file.md) |
| G3 | **Runtime nhất quán trên năm đầu**: Windows / macOS / Linux / Android / iOS dùng cùng ngữ nghĩa JS và API của host; giải mã media, đặc tính tùy chọn được thăm dò theo thiết bị | [02](02-runtime.md), [09](09-media-transport.md) |
| G4 | **Dữ liệu có cấu trúc**: Mọi dữ liệu đều có kiểu, có kiểm chứng, không dùng ký tự phân tách ghép chuỗi để truyền cấu trúc | [05](05-content-model.md) |
| G5 | **Khả năng đầy đủ, triển khai theo nhu cầu**: Bao quát việc lấy nội dung và phát của phim ảnh, phim ngắn và phát trực tiếp; giao diện ẩn hiện theo khả năng, cho phép nguồn thuần VOD và nguồn thuần trực tiếp | [06](06-capabilities.md) |
| G6 | **Có thể tiến hóa**: Giao thức có phiên bản, có thăm dò đặc tính, khả năng mới không phá vỡ plugin cũ | [07](07-ecosystem.md) |

## 4. Không phải mục tiêu

- **Giảm độ khó phát triển plugin**: Không lấy chi phí học bằng không hay không cần build làm mục tiêu; tác giả có thể dùng công cụ kỹ thuật, cuối cùng giao đơn tệp là được. Việc lựa chọn giao diện lấy khả năng, tính đầy đủ ngữ nghĩa và khả năng bảo trì của host làm chuẩn.
- **DRM**: Không liên quan Widevine / FairPlay / PlayReady.
- **Plugin vẽ giao diện**: Plugin chỉ trả về dữ liệu. Giao diện do host thống nhất chịu trách nhiệm.
- **Nội dung đọc**: Truyện tranh, tiểu thuyết không nằm trong phạm vi 1.0.
- **Nguồn dữ liệu tăng cường trải nghiệm**: Thư viện bình luận chạy (danmaku), thư viện phụ đề, thư viện dữ liệu phim ảnh, đồng bộ lịch sử xem không nằm trong phạm vi 1.0.
  1.0 chỉ giải quyết "nội dung từ đâu đến". Rãnh phụ đề mà nguồn nội dung tự mang theo thuộc về dữ liệu phát, vẫn trong phạm vi.
- **Nguyên ngữ kiếm tiền**: Không có giao diện liên quan đến vị trí quảng cáo, thanh toán, mở khóa thành viên.

## 5. Nguyên tắc thiết kế

1. **Host sở hữu giao diện, plugin sở hữu dữ liệu.** Plugin trả về dữ liệu theo hình dạng cố định, host quyết định cách trình bày; khi thiếu khả năng, giao diện tự động thu lại lối vào tương ứng.
   Khi cần người dùng nhập liệu hoặc thao tác, plugin khai báo **biểu mẫu**, host kết xuất trong trang plugin (xem [06 §15](06-capabilities.md)).
2. **Một tệp, một plugin.** Nguồn dữ liệu hoặc resolver mỗi cái được giao thành một tệp `.js` tự chứa.
3. **Chỉ chạy khi được gọi.** Plugin không có nền; host không gọi, plugin không thực thi.
4. **Ưu tiên cục bộ.** Yêu thích, lịch sử, tiến độ do host lưu cục bộ, không đăng nhập cũng dùng được; đồng bộ đám mây là khả năng tùy chọn của plugin.
5. **Vào rộng, ra nghiêm.** Host kiểm chứng hình dạng và chịu lỗi đối với dữ liệu plugin trả về, một dữ liệu xấu đơn lẻ không kéo sập cả trang.

## 6. Kiến trúc

```text
界面（WebView）
  React：首页 / 分类 / 搜索 / 详情 / 播放器 / 片库 / 插件管理
  不含任何数据源知识
      │ IPC：通用命令（plugin_call、library_*、settings_* …）
      │ 本地媒体会话地址（应用协议或本地 HTTP，按平台实现）
      ▼
宿主核心（Rust）
  ├─ 插件管理器     导入 · 初始化/能力发现 · 更新检查 · 启停 · 移除
  ├─ 调用调度器     返回值校验 · 超时 · 取消
  ├─ 插件运行时     每插件独立运行时、内存预算与执行调度，通过 hg.* 与宿主交互
  ├─ 网络层         HTTP 客户端 · Cookie · 限速 · 缓存
  ├─ 媒体会话与代理  资源归属 · 解析链 · HLS 改写 · Range · 刷新 · 字节变换
  └─ 存储           插件键值存储 · 本地片库
      ▼
上游站点与 CDN
```

## 7. Thuật ngữ

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| Host | Bản thân ứng dụng HGhub, phía triển khai giao thức |
| Plugin | Một tệp `.js`, tương ứng một nguồn dữ liệu hoặc resolver |
| Danh sách plugin | Một `manifest.json`, liệt kê một số tệp plugin, dùng để nhập theo lô |
| Khả năng | Một nhóm phương thức liên quan mà plugin triển khai, như `search`, `comments` |
| Đặc tính | Chức năng tùy chọn của host, plugin có thể thăm dò, như `webview` |
| Giá trị truyền được | Giá trị có thể truyền giữa host và plugin: giá trị JSON và `Uint8Array` |
| Phiên media | Tham chiếu tài nguyên, nhà cung cấp, môi trường yêu cầu và thông tin làm mới phát do host quản lý, không phải nhiệm vụ nền của plugin |

Trong tiếng Trung, từ "线路" có ba nghĩa khác nhau; giao thức này đặt tên riêng cho từng nghĩa, không dùng lẫn:

| Thuật ngữ của giao thức này | Tiếng Anh | Ý nghĩa |
| --- | --- | --- |
| **Tuyến trang** | endpoint | Các tên miền lối vào khác nhau của cùng một thượng nguồn (trang chính, trang dự phòng, bản sao) |
| **Danh sách phát** | playlist | Nhiều bộ danh sách phân tập của cùng một tác phẩm (nguồn khác nhau, lồng tiếng, bản chính/phần phụ) |
| **Tuyến phát** | server | Nhiều máy chủ lấy luồng của cùng một tập |

## 8. Bản đồ tài liệu

| Tài liệu | Nội dung | Trạng thái |
| --- | --- | --- |
| [01 Tệp plugin và nhập](01-plugin-file.md) | Cấu trúc tệp, siêu thông tin, `manifest.json`, nhập và cập nhật | Dự thảo |
| [02 Runtime](02-runtime.md) | Engine và môi trường, mô hình gọi, vòng đời, mô hình lỗi, kiểm chứng giá trị trả về | Dự thảo |
| [03 Quy tắc của host](03-host-rules.md) | Cách ly, mạng, giới hạn tốc độ, quyền riêng tư | Dự thảo |
| [04 API của host](04-host-api.md) | Giao diện của các module `hg.*` | Dự thảo |
| [05 Mô hình nội dung](05-content-model.md) | Hình dạng dữ liệu của mục media, phân tập, phát, luồng, rãnh phụ đề | Dự thảo |
| [06 Mô-đun khả năng](06-capabilities.md) | Giao diện khả năng của plugin và ánh xạ giao diện host, tìm kiếm tổng hợp và đổi nguồn | Dự thảo |
| [07 Hệ sinh thái và tiến hóa](07-ecosystem.md) | Tương thích phiên bản, đối chiếu chuẩn, đường cơ sở client và lộ trình | Dự thảo |
| [08 Giao diện host](08-frontend.md) | Thay đổi của từng trang: trang chủ, danh mục, tìm kiếm, trang plugin, trang cài đặt | Dự thảo |
| [09 Truyền tải media](09-media-transport.md) | Thuộc tính tài nguyên, hợp nhất resolver, proxy, yêu cầu theo khoảng, làm mới và phát nền | Dự thảo |

## 9. Tóm tắt quyết định then chốt

| Quyết định | Lựa chọn | Lý do chính |
| --- | --- | --- |
| Hình thức plugin | Một tệp `.js` duy nhất, xuất `meta` và hàm factory | Phân phối và nhập không phụ thuộc môi trường build; siêu thông tin có cấu trúc, có thể kèm mục cài đặt |
| Phân phối | Nhập trực tiếp `.js`, hoặc nhập theo lô qua `manifest.json`; hỗ trợ tệp cục bộ và URL | Không phụ thuộc bất kỳ dịch vụ trung tâm nào; cộng đồng tự lưu trữ |
| Mô hình tin cậy | Không ký, không xét duyệt, không đặt xác nhận quyền | Plugin do cộng đồng làm, người dùng tự chọn, host không bảo chứng |
| Runtime | Mỗi plugin một runtime engine và điều độ thực thi độc lập | Ngữ nghĩa JS/API nhất quán xuyên đầu; cách ly với giao diện; có thể đặt timeout và giới hạn bộ nhớ |
| Giao diện host | Tiêm `hg` qua tham số vào hàm factory | Tác giả có thể trong Node dùng `hg` giả để kiểm thử đơn vị cho plugin |
| Mạng | Thống nhất qua `hg.http` | Bản thân engine không có khả năng mạng; giới hạn tốc độ và bộ nhớ đệm có thể triển khai thống nhất |
| Giao diện | Plugin chỉ trả về dữ liệu; khi cần người dùng nhập liệu thì khai báo biểu mẫu, host kết xuất | Trải nghiệm nhất quán; người duy trì tập trung vào giao diện |
| Hợp tác xuyên plugin | Host trung chuyển dữ liệu có kiểu: nguồn dữ liệu → resolver | Nguồn phim ảnh dùng chung một lô nhà cung cấp lưu trữ video rất nhiều, logic phân giải đáng được tái sử dụng |
| Dữ liệu | Kiểu mạnh, có cấu trúc | Có thể chẩn đoán, có thể tiến hóa |

## 10. Thuật ngữ quy phạm

Trong giao thức này, **phải**, **không được**, **nên**, **có thể** lần lượt tương ứng với MUST, MUST NOT, SHOULD, MAY của RFC 2119. Các từ giống vậy không in đậm được hiểu theo nghĩa thường ngày.
