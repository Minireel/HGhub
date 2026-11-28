# 01 · Tệp plugin và nhập

[← Tổng quan](README.md)

## 1. Tệp plugin

**Một plugin là một tệp `.js`, tương ứng một nguồn dữ liệu hoặc resolver.** Không có gói nén, không có tệp đính kèm.

- ES module mã hóa UTF-8, tên tệp tùy ý (khuyến nghị dùng ID plugin, như `demo.js`).
- Tự chứa: không được dùng import tĩnh, `import()` động, hoặc tái xuất từ module bên ngoài. Khi tác giả phát triển bằng nhiều tệp hoặc thư viện bên thứ ba, tự đóng gói thành đơn tệp.
- Kích thước ≤ 2 MiB.

Tệp có và chỉ có hai xuất:

| Xuất | Mô tả |
| --- | --- |
| `export const meta` | Siêu thông tin: một đối tượng thông thường (§2) |
| `export default` | Hàm factory: nhận API của host `hg`, trả về đối tượng plugin (xem [02 §3](02-runtime.md)) |

### 1.1 Ví dụ tối thiểu

```js
export const meta = {
  id: "demo",
  name: "示例源",
  version: "1.0.0",
  protocol: 1,
  description: "一句话简介",
};

export default function (hg) {
  return {
    media: {
      async getDetail({ id }, ctx) {
        const res = await hg.http.fetch({
          url: `https://api.demo.example/series/${id}`,
          responseType: "json",
          signal: ctx.signal,
        });
        return toDetail(res.data);
      },
      async resolvePlayback({ mediaId, episodeId }, ctx) {
        /* … */
      },
    },
    search: {
      async search({ query, cursor }, ctx) {
        /* … */
      },
    },
  };
}
```

## 2. Siêu thông tin (`meta`)

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `id` | Có | Định danh duy nhất của plugin. Chữ thường, chữ số, `.`, `_`, `-`, bắt đầu bằng chữ cái hoặc chữ số, ≤ 64 ký tự |
| `name` | Có | Tên hiển thị, ≤ 60 ký tự |
| `version` | Có | Phiên bản ba đoạn số nguyên thập phân không âm (như `1.2.0`), mỗi đoạn là số nguyên an toàn, không có số 0 ở đầu thừa; không chấp nhận hậu tố tiền phát hành và build |
| `protocol` | Có | Phiên bản chính của giao thức mà plugin hướng tới, hiện là `1` |
| `description` | Không | Giới thiệu, ≤ 500 ký tự |
| `author` | Không | Tên tác giả |
| `homepage` | Không | Trang chủ hoặc địa chỉ phản hồi |
| `updateUrl` | Không | Địa chỉ lấy tệp mới khi kiểm tra cập nhật (§5.2) |
| `kind` | Không | `source` (mặc định) hoặc `resolver`; resolver xem [06 §3](06-capabilities.md) |
| `matches` | Bắt buộc với resolver | Mảng mẫu host, khai báo resolver này xử lý được những trang phát nào (cách viết xem [06 §3](06-capabilities.md)) |
| `defaultHeaders` | Không | Header yêu cầu mặc định cho yêu cầu media, độ ưu tiên xem [05 §7.4](05-content-model.md) |
| `timeouts` | Không | Ngân sách thời gian cho từng loại lần gọi (giây), giới hạn trên xem [02 §5.2](02-runtime.md) |
| `features` | Không | Mảng, khai báo các đặc tính tùy chọn của host đã dùng (như `["webview"]`), chỉ dùng cho gợi ý giao diện |
| `requires` | Không | Mảng khả năng bắt buộc của host: `{ type: "feature" \| "api", name: string }`; như `{ type: "feature", name: "webview" }` hoặc `{ type: "api", name: "html.parse" }`, khi thiếu thì giữ tệp nhưng không bật được |
| `allowInsecureHttp` | Không | Khi là `true` cho phép plugin dùng địa chỉ `http:` (giao diện và media); mặc định chỉ cho phép HTTPS |
| `linkDomains` | Không | Mảng tên host. Khi người dùng dán liên kết thuộc các tên miền này, host giao cho `media.matchUrl` của plugin này xử lý (xem [06 §2](06-capabilities.md)) |
| `mature` | Không | `true` biểu thị có nội dung người lớn; host mặc định ẩn loại plugin này, người dùng cần bật trong cài đặt |
| `settings` | Không | Các mục cài đặt người dùng điều chỉnh được (§2.1) |

Quy tắc:

- `meta` **phải** là một đối tượng thông thường có thể định giá bằng literal, việc định giá không được phụ thuộc vào `hg` hay sinh tác dụng phụ — host sẽ đọc nó mà không gọi hàm factory.
- `id` là định danh của plugin: thư viện, lưu trữ, cài đặt đều lấy nó làm khóa. Đổi `id` tương đương đổi một plugin khác.
- Host **phải** bỏ qua các trường không nhận biết, để host cũ đọc được plugin bản mới.
- Khi `protocol` cao hơn phiên bản chính mà host hỗ trợ, host từ chối nhập và gợi ý cần cập nhật ứng dụng.

Plugin đã triển khai những khả năng nào **không cần** khai báo trong `meta`: host khi bật lần đầu sẽ thực thi factory, kiểm chứng các phương thức khả năng và đệm kết quả,
rồi theo [02 §6](02-runtime.md) hoàn thành khởi tạo tuyến mặc định. Trước khi khởi tạo hoàn thành, không công bố plugin này cho các lối vào nghiệp vụ như trang chủ, tìm kiếm.
Bộ nhớ đệm gắn với phiên bản tệp plugin và bản sửa đổi nội dung; ghi đè cùng phiên bản cũng phải làm bộ nhớ đệm mất hiệu lực. Hành vi tương thích của `requires` xem ở [07 §1](07-ecosystem.md).

### 2.1 Header yêu cầu và ngân sách thời gian

```js
export const meta = {
  id: "demo", name: "示例源", version: "1.0.0", protocol: 1,
  // 90% 的图片与流都要同一个 Referer，逐条写是噪音，写漏一条就是 403
  defaultHeaders: { Referer: "https://www.example.com/", "User-Agent": "Mozilla/5.0 …" },
  // 上游慢时可以放宽；宿主会夹在上限内（02 §5.2）
  timeouts: { read: 25, playback: 45 },
};
```

- `defaultHeaders` chỉ tác dụng lên **yêu cầu media** (hình ảnh, danh sách phát, phân đoạn, phụ đề), không ảnh hưởng yêu cầu giao diện của `hg.http`.
  Độ ưu tiên là: `headers` của chính mục hoặc luồng > `defaultHeaders` > giá trị mặc định của host (xem [05 §7.4](05-content-model.md)).
- Các khóa của `timeouts` là `read`, `playback`, `login`, `transform`, đơn vị giây.

### 2.2 Mục cài đặt

```js
settings: [
  { key: "quality", type: "select", label: "默认清晰度", default: "auto",
    options: [{ value: "auto", label: "自动" }, { value: "1080", label: "1080P" }] },
  { key: "preferHevc", type: "switch", label: "优先 HEVC", default: false },
  { key: "apiKey", type: "password", label: "接口密钥" },
]
```

Mỗi mục gồm `key`, `type`, `label`, và tùy chọn `description`, `default`.

| `type` | Kiểu giá trị | Trường bổ sung |
| --- | --- | --- |
| `text` | Chuỗi | `placeholder` |
| `password` | Chuỗi (giao diện hiển thị che) | `placeholder` |
| `switch` | Boolean | — |
| `number` | Số | `min`, `max` |
| `select` | Chuỗi | `options: [{ value, label }]` |

Host chịu trách nhiệm kết xuất biểu mẫu cài đặt và lưu giá trị của người dùng; plugin đọc qua `hg.settings` (xem [04](04-host-api.md)).

## 3. Cách nhập

| Cách | Mô tả |
| --- | --- |
| Tệp `.js` cục bộ | Có thể chọn nhiều tệp cùng lúc |
| `manifest.json` cục bộ | Nhập theo lô theo danh sách trong đó (§4) |
| URL | Sau khi tải xuống nhận diện theo nội dung một plugin JS đơn lẻ hoặc một danh sách plugin JSON, hỗ trợ giao diện tải xuống không có đuôi tệp |
| Dán mã nguồn | Nhập một plugin bằng mã nguồn UTF-8; khi không có URL nguồn thì cần `updateUrl` mới kiểm tra cập nhật được |

Đuôi tệp của đường dẫn URL và `Content-Type` chỉ là gợi ý, không làm căn cứ từ chối. Host trước hết thử phân tích thành danh sách plugin JSON,
nếu không thì đọc siêu thông tin theo module JS; khi nhận diện thất bại báo lỗi cụ thể, không lưu trang lỗi HTML thành plugin.
Địa chỉ tương đối lấy URL danh sách cuối cùng sau chuyển hướng làm chuẩn; việc tải xuống vẫn chịu ràng buộc của quy tắc mạng của host và giới hạn kích thước tệp tương ứng.

Quy trình nhập một plugin đơn lẻ:

```text
取得文件内容 → 读取 meta 并检查必需字段与 protocol → 写入插件目录并记录来源 → 完成
```

- Việc nhập chỉ đọc `meta`, không gọi hàm factory.
- "Đọc" dùng việc định giá module bằng runtime tạm thời chịu ràng buộc ngân sách thời gian và bộ nhớ, không phải trích xuất tĩnh; tầng cao nhất của module không được thực thi I/O bên ngoài.
  Việc định giá tầng cao nhất và kiểm tra siêu thông tin dùng chung ngân sách 5 giây. `await` tầng cao nhất, ngoại lệ, vòng lặp vô hạn đều chịu ràng buộc của ngân sách này.
- Sau khi nhập thành công, mặc định bật và kích hoạt khởi tạo; khi người dùng chọn chỉ cài đặt thì giữ trạng thái vô hiệu hóa. Khởi tạo thất bại giữ lại tệp, hiển thị nguyên nhân và cho phép thử lại.
- Khi nhập theo lô, một plugin đơn lẻ thất bại không ảnh hưởng các plugin còn lại; kết thúc sẽ tổng hợp danh sách thành công và thất bại kèm nguyên nhân.
- Host ghi lại **nguồn gốc** của mỗi plugin (đường dẫn cục bộ hoặc URL), dùng cho việc kiểm tra cập nhật sau này.

## 4. Danh sách plugin (`manifest.json`)

```json
{
  "name": "某某的插件集",
  "description": "可选的说明",
  "plugins": [
    {
      "id": "demo",
      "name": "示例源",
      "version": "1.0.0",
      "description": "一句话简介",
      "url": "demo.js"
    },
    {
      "id": "another",
      "name": "另一个源",
      "version": "0.3.1",
      "url": "https://cdn.example.com/plugins/another.js"
    }
  ]
}
```

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `name` | Không | Tên danh sách |
| `description` | Không | Thuyết minh danh sách |
| `plugins` | Có | Mảng các mục plugin |
| `plugins[].url` | Có | Địa chỉ tệp plugin. Có thể là URL tuyệt đối, hoặc đường dẫn tương đối với vị trí của chính `manifest.json` |
| `plugins[].id` | Có | Nhất quán với `meta.id` trong tệp plugin |
| `plugins[].name` | Có | Tên hiển thị |
| `plugins[].version` | Có | Nhất quán với `meta.version` trong tệp plugin |
| `plugins[].description` | Không | Giới thiệu |

Quy tắc:

- `id`, `name`, `version`, `description` trong danh sách chỉ dùng để hiển thị cho người dùng và so sánh phiên bản **trước khi tải xuống**;
  sau khi nhập, mọi thứ lấy `meta` trong tệp plugin làm chuẩn. Nhưng khi `id`, `version` của tệp tải xuống không khớp với danh sách, mục này nhập thất bại,
  không được cài đặt hay ghi đè dưới một định danh khác; điều này thường có nghĩa danh sách và tệp phát hành không đồng bộ.
- Chuẩn phân giải đường dẫn tương đối là thư mục chứa `manifest.json` — khi nhập cục bộ là thư mục chứa tệp đó,
  khi nhập bằng URL là chính URL đó.
- Địa chỉ `content://` của nhà cung cấp tài liệu Android không phải đường dẫn hệ thống tệp. Mục tương đối cần người dùng ủy quyền thư mục chứa danh sách;
  host truy vấn từng tầng theo tên tệp của nhà cung cấp, không ghép ID tài liệu không trong suốt. Chỉ chọn tệp danh sách không trao quyền truy cập tệp anh em.
  Đường dẫn tương đối không được vượt ra ngoài thư mục đã ủy quyền, cũng không được mượn đường dẫn tuyệt đối, ký tự phân tách đã mã hóa, truy vấn hay fragment để thay đổi việc định vị tệp;
  mục HTTP(S) tuyệt đối vẫn xử lý như tải xuống từ xa. Ủy quyền thư mục thuộc về quy trình nhập của host, không mở API truy cập tệp cho plugin.
- Người dùng có thể trong giao diện danh sách chọn các plugin muốn nhập, mặc định chọn toàn bộ.
- Host **phải** bỏ qua các trường không nhận biết.
- Số mục ≤ 500; tệp ≤ 1 MiB.

### 4.1 Đăng ký

Sau khi nhập danh sách plugin từ URL, host **có thể** lưu địa chỉ đó thành "đăng ký": sau đó người dùng có thể làm mới bằng một bấm,
thấy các plugin mới thêm trong danh sách và các plugin có cập nhật. Đây chỉ là lớp vỏ giao diện cho §3, §5, không đưa vào khái niệm giao thức mới.

## 5. Ghi đè và cập nhật

### 5.1 Ghi đè cùng ID

Khi nhập một plugin mà `id` đã tồn tại:

| Phiên bản của tệp mới | Hành vi của host |
| --- | --- |
| Cao hơn | Cập nhật trực tiếp |
| Giống nhau | Ghi đè (tiện cho tác giả gỡ lỗi) |
| Thấp hơn | Gợi ý đây là hạ cấp, người dùng xác nhận |

Việc ghi đè giữ lại lưu trữ, cài đặt và bản ghi thư viện của plugin đó. `key` không còn tồn tại trong `settings` của phiên bản mới thì giá trị tương ứng bị xóa;
khi kiểu giá trị hoặc tùy chọn của khóa được giữ lại đã không hợp lệ thì quay về giá trị mặc định mới. Việc cập nhật plugin thay thế tệp nguyên tử, làm bộ nhớ đệm khả năng mất hiệu lực và khởi tạo lại;
khi khởi tạo thất bại, cung cấp thao tác khôi phục tệp khả dụng trước đó, không khôi phục các thao tác thượng nguồn hay ghi lưu trữ mà plugin đã thực thi.
Phiên phát đang hoạt động dùng plugin đó sẽ dừng trước khi thay thế và gợi ý phát lại, tránh tài nguyên cũ dùng lẫn mã mới; việc kiểm tra cập nhật không ảnh hưởng việc phát.

### 5.2 Kiểm tra cập nhật

Host xác định địa chỉ cập nhật của một plugin theo thứ tự sau: `meta.updateUrl` → URL nguồn lúc nhập → không có (nhập cục bộ và chưa khai báo thì không kiểm tra).

- Khi khai báo `meta.updateUrl` thì lấy trực tiếp tệp JS đó; nếu không, khi nguồn là danh sách plugin thì làm mới danh sách và so sánh phiên bản mục,
  có phiên bản cao hơn mới tải xuống; khi nguồn là một địa chỉ JS đơn lẻ thì lấy trực tiếp tệp và so sánh phiên bản.
- Mặc định do người dùng kích hoạt thủ công. Host **có thể** cung cấp công tắc "tự động kiểm tra khi khởi động", khoảng cách tự động kiểm tra cùng một địa chỉ không ngắn hơn 24 giờ.
- Sau khi phát hiện phiên bản mới, có tự động cài đặt hay không do người dùng quyết định trong cài đặt; mặc định chỉ gợi ý. Cập nhật tự động chỉ chấp nhận phiên bản cao hơn của cùng ID,
  không tự động ghi đè cùng phiên bản hay hạ cấp; trong lúc đang phát thì hoãn cài đặt. Khoảng cách 24 giờ chỉ ràng buộc kiểm tra tự động, kiểm tra thủ công của người dùng không bị giới hạn này.

Việc so sánh phiên bản tiến hành từng đoạn theo số (`1.10.0` cao hơn `1.9.0`); phiên bản không hợp lệ sẽ báo lỗi rõ ràng khi nhập hoặc kiểm tra cập nhật, không đoán là 0.

### 5.3 Gỡ bỏ

Khi gỡ bỏ plugin, xóa tệp, lưu trữ, Cookie, bộ nhớ đệm và cài đặt của nó.
Mục yêu thích và lịch sử trong thư viện cục bộ của host tham chiếu plugin đó **nên** được giữ lại và đánh dấu "nguồn không khả dụng", để khôi phục sau khi nhập lại.
