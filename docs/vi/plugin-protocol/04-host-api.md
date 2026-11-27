# 04 · API của host

[← Tổng quan](README.md)

API của host là con đường **duy nhất** để plugin tương tác với bên ngoài. Nó được truyền cho hàm factory dưới dạng đối tượng `hg` (xem [02 §3](02-runtime.md)).
Mọi plugin nhìn thấy cùng một bộ `hg`.

## 1. Tổng quan các module

| Module | Công dụng | Đồng bộ/Bất đồng bộ |
| --- | --- | --- |
| [`hg.env`](#3-hgenv) | Thông tin môi trường chạy | Đồng bộ chỉ đọc |
| [`hg.endpoint`](#4-hgendpoint) | Tuyến trang hiện tại | Đồng bộ chỉ đọc |
| [`hg.http`](#2-hghttp) | Yêu cầu mạng | Bất đồng bộ |
| [`hg.cookies`](#5-hgcookies) | Kho Cookie | Bất đồng bộ |
| [`hg.storage`](#6-hgstorage) | Lưu trữ khóa–giá trị bền vững | Bất đồng bộ |
| [`hg.crypto`](#7-hgcrypto) | Nguyên ngữ mật mã học | Đồng bộ |
| [`hg.codec`](#8-hgcodec) | Chuyển đổi mã hóa | Đồng bộ |
| [`hg.html`](#9-hghtml) | Phân tích và truy vấn HTML | Đồng bộ |
| [`hg.settings`](#10-hgsettings) | Đọc cài đặt của người dùng | Đồng bộ chỉ đọc |
| [`hg.log`](#11-hglog) | Nhật ký | Đồng bộ |
| [`hg.webview`](#12-hgwebview-tùy-chọn) | Mời người dùng hoàn thành thao tác trong trang web | Bất đồng bộ |
| `hg.PluginError` | Kiểu lỗi | — |

Quy tắc chung:

- Khi giao diện bất đồng bộ thất bại thì ném `hg.PluginError` (mã lỗi xem [02 §8](02-runtime.md)).
- Dữ liệu nghiệp vụ dùng giá trị truyền được; ngoại lệ của các đối tượng runtime như `signal`, handle node, lỗi xem [02 §4](02-runtime.md).
- Thao tác bất đồng bộ tự động gắn với phạm vi lần gọi hiện tại, không cần plugin truyền signal mới hủy theo lần gọi được; signal rõ ràng chỉ thêm điều kiện hủy sớm.
- Trong thời gian thực thi hàm factory, ngoài `hg.env`, `hg.endpoint`, `hg.settings`, `hg.log`, `hg.crypto`, `hg.codec`
  ra, các module khác không dùng được.

## 2. `hg.http`

```ts
hg.http.fetch(request: HttpRequest): Promise<HttpResponse>
```

### 2.1 Yêu cầu

```ts
interface HttpRequest {
  url: string;                       // 绝对 https 地址
  method?: "GET" | "HEAD" | "POST" | "PUT" | "PATCH" | "DELETE";   // 默认 GET
  headers?: Record<string, string>;
  query?: Record<string, string | number | boolean | undefined>;   // 追加到 url
  body?: string | Uint8Array | { json: Json } | { form: Record<string, string> };
  responseType?: "text" | "json" | "bytes";                         // 默认 text
  charset?: string;                  // text/json 的解码字符集
  timeoutMs?: number;                // 默认 15000，上限 30000
  redirect?: "follow" | "manual";    // 默认 follow
  maxBytes?: number;                 // 响应体上限，默认 2 MiB，最大 8 MiB
  cache?: { ttlSec: number; staleIfErrorSec?: number };
  retry?: 0 | 1 | 2;                 // 默认 0
  credentials?: "include" | "omit";  // 是否使用 Cookie 罐，默认 include
  signal?: AbortSignal;
}
```

| Trường | Quy tắc |
| --- | --- |
| `url` | Độ dài ≤ 8 KiB; đích chịu ràng buộc của quy tắc mạng ở [03 §2](03-host-rules.md), khi bị chặn thì ném `NETWORK` |
| `headers` | Tổng kích thước ≤ 16 KiB; các header do tầng truyền tải quyết định sẽ bị loại bỏ ([03 §2](03-host-rules.md)). Khi không chỉ định `User-Agent` thì là `HGhub/<phiên bản>` |
| `query` | Khóa có giá trị `undefined` bị bỏ qua; mã hóa theo quy tắc biểu mẫu rồi thêm vào |
| `body` | ≤ 1 MiB. `{ json }` tự động đặt `Content-Type: application/json`; `{ form }` tự động đặt `application/x-www-form-urlencoded`; `Content-Type` cho rõ ràng được ưu tiên. `GET`, `HEAD` không được mang thân yêu cầu |
| `charset` | Khi bỏ qua thì lấy charset trong header phản hồi, quay về UTF-8; charset khác cần đặc tính `charset.legacy` |
| `maxBytes` | Thân phản hồi vượt quá thì dừng lại và ném `UPSTREAM_ERROR` |

### 2.2 Phản hồi

```ts
interface HttpResponse<T> {
  status: number;
  ok: boolean;                       // 200–299
  url: string;                       // 最终地址
  headers: Record<string, string>;   // 键小写；多值以 ", " 连接
  setCookies: string[];              // 对插件可见的原始 Set-Cookie，逐条
  data: T;                           // 按 responseType：string / 解析后的 JSON / Uint8Array
  fromCache: boolean;
}
```

- **Không phải 2xx không ném lỗi.** Plugin dựa trên `status` tự ánh xạ thành `PluginError` thích hợp.
- Khi `responseType: "json"` mà thân phản hồi không phải JSON hợp lệ thì ném `UPSTREAM_CHANGED`.
- Việc nén `gzip`, `br` do host xử lý.
- Khi origin đích có phiên đăng nhập web do host quản lý, toàn bộ `Set-Cookie` trong phản hồi do host xử lý, `headers` bỏ qua `set-cookie`, `setCookies` rỗng. `credentials: "omit"` bỏ qua việc tự động gửi và ghi, đồng thời ẩn `Set-Cookie` của trang đó. Phản hồi có `Set-Cookie` dù đã ẩn header đó cũng không thể đệm.

### 2.3 Chuyển hướng

| Yêu cầu | Hành vi |
| --- | --- |
| `redirect: "manual"` | Không đi theo; phản hồi 3xx trả về nguyên trạng, `headers.location` đọc được |
| Mang thân yêu cầu (`POST`, v.v.) | Luôn xử lý theo `manual`, tránh gửi trùng |
| Khác | Đi theo, tối đa 5 bước nhảy; mỗi bước nhảy qua lại quy tắc mạng |

Khi chuyển hướng xuyên origin thì loại bỏ Cookie và Authorization rõ ràng, không đính kèm lại header thông tin xác thực gốc cho đích xuyên nguồn;
Cookie tự động mỗi bước nhảy khớp lại kho Cookie của plugin sở thuộc theo URL đích.

### 2.4 Bộ nhớ đệm

`cache` để host đệm phản hồi thay, giảm yêu cầu trùng lặp tới thượng nguồn.

- Chỉ có hiệu lực với yêu cầu có phương thức `GET`, không mang header `Authorization`, và không dùng Cookie; không thỏa mãn thì lặng lẽ bỏ qua.
- Khóa bộ nhớ đệm gồm plugin, địa chỉ yêu cầu lối vào (gồm chuỗi truy vấn) và toàn bộ header yêu cầu hiệu lực, và phân biệt phản hồi theo `Vary`;
  phản hồi có `Vary: *`, `Cache-Control: no-store` hoặc đặt Cookie thì không đệm. Đích sau chuyển hướng chịu ràng buộc quy tắc mạng mỗi bước nhảy,
  không được chỉ lấy địa chỉ cuối cùng để hợp nhất các yêu cầu có ngữ nghĩa lối vào khác nhau.
- Chỉ đệm 2xx; giới hạn trên của `ttlSec` là 86400.
- `staleIfErrorSec`: khi yêu cầu mới thất bại vì `NETWORK`, `TIMEOUT` hoặc 5xx, có thể trả về phản hồi cũ hết hạn không vượt quá thời lượng đó
  (giới hạn trên 7 ngày), lúc này `fromCache` là `true`.
- Bộ nhớ đệm HTTP dùng hạn ngạch có thể loại bỏ độc lập, không chiếm 5 MiB của `hg.storage`; bộ nhớ đệm media cũng quản lý độc lập.
  Bộ nhớ đệm không đủ chỉ ảnh hưởng tỷ lệ trúng, không được loại bỏ dữ liệu bền vững hay khiến việc ghi lưu trữ thất bại; việc người dùng "xóa bộ nhớ đệm" không xóa lưu trữ và trạng thái đăng nhập.

### 2.5 Thử lại

`retry` chỉ có hiệu lực với `GET`, `HEAD`, khi `NETWORK`, `TIMEOUT` hoặc 5xx tự động thử lại, khoảng cách 400 mili giây, 1200 mili giây.
Việc thử lại cũng tiêu hao hạn ngạch giới hạn tốc độ; host đang trong thời gian làm nguội thì không thử lại.

### 2.6 Các lỗi có thể ném

| Mã lỗi | Tình huống |
| --- | --- |
| `NETWORK` | DNS, kết nối, TLS thất bại, hoặc đích bị quy tắc mạng chặn |
| `RATE_LIMITED` | Xếp hàng quá thời gian chờ, hoặc host đang trong thời gian làm nguội (kèm `retryAfterSec`) |
| `TIMEOUT` | Vượt `timeoutMs` hoặc thời hạn của lần gọi |
| `CANCELLED` | `signal` bị kích hoạt |
| `UPSTREAM_ERROR` | Thân phản hồi vượt `maxBytes` |
| `UPSTREAM_CHANGED` | Phân tích `responseType: "json"` thất bại |
| `INVALID_ARGUMENT` | Địa chỉ không hợp lệ, thân yêu cầu quá lớn, `GET` mang thân yêu cầu, v.v. |

## 3. `hg.env`

```ts
interface HostEnv {
  protocol: string;                    // 宿主实现的协议版本，如 "1.0.0"
  app: { name: string; version: string };
  plugin: { id: string; version: string };
  platform: "windows" | "macos" | "linux" | "android" | "ios";
  locale: string;                      // 界面语言，BCP 47
  features: readonly string[];         // 宿主支持的可选特性
  media: {
    codecs: readonly ("h264" | "hevc" | "av1")[];   // 当前播放路径实际可用，按偏好排序
    formats: readonly ("hls" | "dash" | "progressive")[];
  };
}
```

- Toàn bộ là ảnh chụp lúc plugin tải. Sau khi ngôn ngữ giao diện, v.v. thay đổi, host tải lại plugin.
- Plugin **nên** dùng `features` và `media` để thăm dò khả năng, thay vì suy đoán theo `platform` hoặc `app.version`.
- Khả năng media lấy việc thăm dò bằng API khả năng của đường đi phát hiện tại làm căn cứ; tên codec là họ codec, không đảm bảo mọi profile,
  độ phân giải, tốc độ bit hay tổ hợp âm thanh/hình ảnh của họ đó đều phát được, cũng không tương đương nghiệm thu giải mã mẫu thực tế. Khi chưa thăm dò hoặc không có kết quả hỗ trợ đáng tin thì cho phép danh sách rỗng,
  không được mặc định tuyên bố H.264 hay đoán theo tên hệ điều hành. Cách thăm dò hiện tại của HGhub và phạm vi bảo thủ xem [Thuyết minh triển khai](IMPLEMENTATION.md#thăm-dò-khả-năng-phát-2026-11-16).
- `params.codecs` mà VOD, trực tiếp và resolver nhận được phải nhất quán với `hg.env.media.codecs` trong ảnh chụp tải của plugin đó,
  bao gồm phát bình thường, tải trước và làm mới. Khi kết quả thăm dò của host thay đổi thì xây dựng lại runtime, không thể sửa tại chỗ ảnh chụp chỉ đọc đã có.
- Với thượng nguồn cần "số thiết bị", hãy tự sinh giá trị ngẫu nhiên và lưu vào `hg.storage`.

### 3.1 Đặc tính tùy chọn

| Tên đặc tính | Ý nghĩa |
| --- | --- |
| `webview` | Hỗ trợ `hg.webview` ([§12](#12-hgwebview-tùy-chọn)) |
| `charset.legacy` | `TextDecoder`, `hg.codec.decodeText`, `charset` của `hg.http` hỗ trợ GBK / GB18030 / Big5 / Shift_JIS / EUC-KR |
| `transform.segment` | Hỗ trợ biến đổi tài nguyên đối với phân đoạn video (xem [05](05-content-model.md)) |
| `format.dash` | Trình phát hiện tại và proxy MPD/phân đoạn của host đều hỗ trợ MPEG-DASH (xem 09) |

Plugin khai báo các đặc tính đã dùng trong `meta.features` (mảng), chỉ dùng cho gợi ý giao diện; lúc chạy **nên** tự thăm dò và hạ cấp.

## 4. `hg.endpoint`

```ts
interface EndpointApi {
  readonly current: {
    id: string;
    label?: string;
    url: string;           // 纯 origin，如 "https://api.example.com"
  } | null;
  url(path: string): string;   // 把以单个 "/" 开头的路径拼到当前线路上；拒绝 "//host" 等跨 origin 形式
}
```

- Áp dụng cho tình huống thượng nguồn có nhiều tên miền lối vào. Sau khi plugin triển khai khả năng `endpoints` (xem [06](06-capabilities.md)),
  host hiển thị danh sách tuyến trong trang chi tiết plugin và lưu lựa chọn của người dùng, plugin đọc qua đây.
- Khi plugin không triển khai khả năng đó hoặc chưa có lựa chọn mặc định thì `current` là `null`, `url()` ném `UNSUPPORTED`.
- Khi bật lần đầu, trước hết lấy danh sách tuyến, lưu lựa chọn mặc định, khi cần xây dựng lại ngữ cảnh, quy trình đầy đủ xem [02 §6](02-runtime.md).
- Việc chuyển tuyến sẽ khởi động lại ngữ cảnh plugin; lưu trữ thông thường không chuyển theo, việc thông tin xác thực có gắn với origin hay không do plugin quyết định.

## 5. `hg.cookies`

```ts
hg.cookies.get(params: { url: string }): Promise<Cookie[]>
hg.cookies.set(params: { url: string; cookie: Cookie }): Promise<void>
hg.cookies.clear(params?: { url?: string }): Promise<void>

interface Cookie {
  name: string; value: string;
  domain?: string; path?: string;
  hostOnly?: boolean;      // get 返回匹配属性；set 省略 domain 时为 true
  expiresAt?: number;      // Unix 毫秒；省略为会话 Cookie
  secure?: boolean; httpOnly?: boolean;
}
```

- Kho Cookie thông thường được cách ly và lưu bền vững theo **plugin**; trong kho khớp theo các quy tắc Cookie thông thường như domain, host-only, path, secure, expires của URL yêu cầu,
  không chuyển toàn bộ theo sự thay đổi của tuyến trang hiện tại. Việc cùng một dịch vụ qua các bản sao có đăng nhập được hay không do miền Cookie của thượng nguồn và logic đăng nhập của plugin quyết định.
- `get({ url })` trả về các Cookie trong kho thông thường khớp với URL đó; domain của `set` phải là cái mà URL đã cho có thể đặt;
  `clear({ url })` xóa các Cookie khớp với URL đó, không truyền url thì xóa toàn bộ Cookie của plugin đó.
- Khi `set` bỏ qua domain thì gắn với host của URL; khi cho domain thì mặc định không phải host-only, có thể đặt rõ ràng thành true để giới hạn ở host đó.
- `hg.http.fetch` tự động lưu `Set-Cookie` của phản hồi và đính kèm trong các yêu cầu sau, trừ khi `credentials: "omit"`.
- Cookie phiên **vẫn được giữ** sau khi plugin bị gỡ bỏ (plugin bất cứ lúc nào cũng có thể bị gỡ bỏ, không thể dùng điều này mô phỏng việc kết thúc phiên trình duyệt);
  xóa khi plugin đăng xuất hoặc người dùng xóa dữ liệu.

### 5.1 Phiên đăng nhập web do host quản lý

- Phiên mà khung xem một lần bàn giao thành công được lưu theo **ID plugin + origin HTTPS bắt đầu**. Origin gồm host và cổng hiệu lực. Trang hoàn tất phải quay về origin bắt đầu, Cookie của nhà cung cấp định danh bên thứ ba chỉ ở lại trong khung xem tạm thời.
- `credentials: "include"` (mặc định) và yêu cầu media của plugin đó, khi origin đó có phiên đăng nhập web thì dùng kho chuyên dụng của nó, vẫn khớp theo domain, host-only, path, secure và thuộc tính hết hạn của chính Cookie. Mỗi lần chuyển hướng chọn lại kho của đích. Header `Cookie` rõ ràng dùng quy tắc ghi đè hiện có.
- `hg.cookies.get/set` chỉ truy cập kho thông thường. Cookie đăng nhập web và `Set-Cookie` cùng origin tiếp theo đều ở lại trong kho do host quản lý, plugin dùng giao diện tài khoản cùng origin để đọc thông tin người dùng. Nguồn và resolver có thuộc tính độc lập.
- Đăng nhập lại thành công với cùng một origin thì thay thế nguyên tử phiên đăng nhập web trước đó. Thất bại hoặc hủy bỏ giữ phiên cũ. `hg.cookies.clear({ url })` xóa toàn bộ phiên đăng nhập web của origin đó (gồm các đường dẫn khác), và theo quy tắc thông thường xóa các Cookie thông thường khớp URL. Xóa không tham số thì xóa cả hai loại kho của plugin đó. Các yêu cầu đã phát ra trước khi đăng xuất hoặc đổi tài khoản, phản hồi đến muộn của chúng không thể khôi phục hay sửa phiên cũ.

## 6. `hg.storage`

```ts
hg.storage.get<T>(key: string): Promise<T | undefined>
hg.storage.set(key: string, value: TransferValue, options?: { ttlSec?: number }): Promise<void>
hg.storage.delete(key: string): Promise<void>
hg.storage.keys(prefix?: string): Promise<string[]>
hg.storage.clear(): Promise<void>
```

- Khóa: chuỗi UTF-8 1–256 byte. Giá trị: bất kỳ giá trị truyền được nào (`Uint8Array` cũng hỗ trợ), một cái ≤ 256 KiB, tổng ≤ 5 MiB.
- `TransferValue` chỉ giá trị truyền được của [02 §4](02-runtime.md). Namespace chỉ chia theo **ID plugin**,
  việc chuyển tuyến trang không chuyển lưu trữ; việc di trú nội bộ của plugin, định danh ngẫu nhiên của thiết bị và cấu hình xuyên tuyến có thể dùng bền vững.
- `ttlSec`: sau khi hết hạn `get` trả về `undefined` (mục hết hạn do host thu hồi), thích hợp để đặt bộ nhớ đệm có thể xây dựng lại.
- Việc ghi là nguyên tử: sau khi `set` trả về, dữ liệu đã được lưu bền vững, plugin sau đó bị gỡ bỏ cũng không mất.
- Lưu trữ là văn bản thuần (trong thư mục dữ liệu ứng dụng). Việc token có chia sẻ qua các bản sao hay không do plugin quyết định; khi cần gắn với origin,
  có thể dùng tiền tố khóa kiểu `auth:<origin>`, host không cưỡng bức quyết định phạm vi xác thực thay plugin.
- Dữ liệu bền vững không tự động bị loại bỏ; khi vượt hạn ngạch `set` ném `STORAGE_FULL`, các giá trị trước đó giữ nguyên vẹn.
  Việc đọc-rồi-ghi đồng thời không phải giao dịch; plugin cần tự phối hợp việc cập nhật phức hợp trên cùng một khóa.
- Khi gỡ bỏ plugin thì xóa toàn bộ.

## 7. `hg.crypto`

Các nguyên ngữ đồng bộ được triển khai native. Tham số vào dạng chuỗi được mã hóa theo UTF-8.

```ts
type Hash = "md5" | "sha1" | "sha256" | "sha512";

hg.crypto.digest(algorithm: Hash, data: Uint8Array | string): Uint8Array
hg.crypto.hmac(algorithm: Hash, key: Uint8Array | string, data: Uint8Array | string): Uint8Array
hg.crypto.aesEncrypt(params: AesParams, data: Uint8Array): Uint8Array
hg.crypto.aesDecrypt(params: AesParams, data: Uint8Array): Uint8Array
hg.crypto.rsaEncrypt(params: RsaParams, data: Uint8Array): Uint8Array
hg.crypto.randomBytes(length: number): Uint8Array      // ≤ 65536
hg.crypto.randomUUID(): string

interface AesParams {
  mode: "cbc" | "ecb" | "ctr" | "gcm";
  key: Uint8Array;                  // 16 / 24 / 32 字节
  iv?: Uint8Array;                  // cbc、ctr：16 字节；gcm：12 字节；ecb：省略
  padding?: "pkcs7" | "none";       // 仅 cbc、ecb，默认 pkcs7
  aad?: Uint8Array;                 // 仅 gcm
}

interface RsaParams {
  publicKey: string;                // PEM（SPKI 或 PKCS#1）
  padding: "pkcs1" | "oaep-sha1" | "oaep-sha256";
}
```

- Thẻ xác thực 16 byte của GCM đính ở cuối bản mã; kiểm chứng thất bại ném `INVALID_ARGUMENT`.
- `rsaEncrypt` chỉ làm mã hóa khóa công khai (một số thượng nguồn yêu cầu dùng khóa công khai mã hóa mật khẩu đăng nhập). Modulus của khóa công khai là 1024–8192 bit;
  nhãn PEM là `PUBLIC KEY` (SPKI) hoặc `RSA PUBLIC KEY` (PKCS#1), không chấp nhận khóa riêng.
- `pkcs1` của RSA biểu thị RSAES-PKCS1-v1_5; OAEP dùng hàm băm đã chọn và MGF1 cùng hàm băm, label rỗng.
  Nguồn ngẫu nhiên an toàn của hệ thống cung cấp tính ngẫu nhiên cho phần đệm, độ dài trả về là bản mã có số byte bằng modulus, không tự động chia khối.
- Đặt số byte modulus là k, giới hạn trên bản rõ một lần của RSA: `pkcs1` là k−11, `oaep-sha1` là k−42,
  `oaep-sha256` là k−66. `data` phải là `Uint8Array`; văn bản trước hết mã hóa UTF-8, rồi tính độ dài theo byte.
  PEM/khóa công khai không hợp lệ, cách đệm, kiểu hoặc bản rõ quá dài ném `INVALID_ARGUMENT`; nguồn ngẫu nhiên thất bại ném `INTERNAL`.
- Các thuật toán ngoài bảng trên do plugin tự đóng gói triển khai JS thuần.

## 8. `hg.codec`

```ts
hg.codec.base64Encode(data: Uint8Array | string, options?: { urlSafe?: boolean; pad?: boolean }): string
hg.codec.base64Decode(text: string): Uint8Array      // 同时接受标准与 URL 安全字母表
hg.codec.hexEncode(data: Uint8Array | string): string
hg.codec.hexDecode(text: string): Uint8Array
hg.codec.utf8Encode(text: string): Uint8Array
hg.codec.utf8Decode(data: Uint8Array): string
hg.codec.decodeText(data: Uint8Array, charset: string): string
```

Khi đầu vào không hợp lệ thì ném `INVALID_ARGUMENT`. `decodeText` đối với charset ngoài UTF-8 cần đặc tính `charset.legacy`.

## 9. `hg.html`

Không ít thượng nguồn chỉ có trang để phân tích. Host cung cấp phân tích HTML và truy vấn bộ chọn CSS dạng native,
tránh việc mỗi plugin tự đóng gói thư viện phân tích khổng lồ.

```ts
hg.html.parse(html: string): HtmlNode

interface HtmlNode {
  readonly tag: string;                       // 小写标签名；文档根为 "#document"
  select(selector: string): HtmlNode[];
  first(selector: string): HtmlNode | null;
  text(): string;                             // 后代文本拼接并折叠空白
  ownText(): string;                          // 仅直接子文本节点
  attr(name: string): string | null;
  attrs(): Record<string, string>;
  html(): string;                             // 内部 HTML
  outerHtml(): string;
  parent(): HtmlNode | null;
  children(): HtmlNode[];
}
```

- Phân tích theo quy tắc HTML5; **không thực thi script, không tải bất kỳ tài nguyên con nào**.
- Bộ chọn hỗ trợ CSS Selectors Level 3, cộng thêm `:has()`; không hợp lệ thì ném `INVALID_ARGUMENT`.
- Đầu vào ≤ 8 MiB.
- Handle node là tham chiếu tới đối tượng của host: chỉ có thể dùng **trong lần gọi đã tạo ra nó**, sau khi lần gọi kết thúc sẽ mất hiệu lực;
  không thể truyền cho các giao diện `hg.*` khác, không thể làm giá trị trả về.
- Dữ liệu cần thực thi script của trang mới lấy được, plugin tự khôi phục thuật toán của nó; giao thức không cung cấp việc thực thi script trong trang.

## 10. `hg.settings`

```ts
hg.settings.get<T>(key: string): T | undefined
hg.settings.getAll(): Record<string, Json>
```

- Trả về giá trị người dùng đặt cho plugin đó; khi chưa đặt thì trả về `default` trong `meta.settings`; khóa chưa định nghĩa trả về `undefined`.
- Cài đặt là chỉ đọc: chỉ người dùng mới có thể sửa qua giao diện host.
- Sau khi người dùng sửa cài đặt, host thống nhất hủy các lần gọi đang bay của plugin đó và khởi tạo lại, không cung cấp hook `onSettingsChanged`.
  Giao diện cài đặt có thể hợp nhất nhiều thay đổi trong một lần gửi biểu mẫu, tránh tải lại lặp đi lặp lại theo từng trường.

## 11. `hg.log`

```ts
hg.log.debug(...args: unknown[]): void
hg.log.info(...args: unknown[]): void
hg.log.warn(...args: unknown[]): void
hg.log.error(...args: unknown[]): void
```

Các tham số được định dạng thành một dòng văn bản; đối tượng được tuần tự hóa JSON, `Uint8Array` chỉ ghi độ dài.
Hạn mức và tính khả kiến xem [02 §10](02-runtime.md). Nhật ký vượt tốc độ bị loại bỏ và đếm.

## 12. `hg.webview` (tùy chọn)

Cần đặc tính `webview`. Mục này định nghĩa việc đăng nhập web có điều kiện hoàn tất.

```ts
hg.webview.open(params: {
  url: string;                 // 起始页
  title?: string;              // 外框标题，≤ 60 字符
  doneWhen: {
    urlPrefix?: string;        // 导航到以此为前缀的地址时视为完成
    cookieNames?: string[];    // 这些 Cookie 全部出现时视为完成
  };
  timeoutSec?: number;         // 默认 300，上限 600
}): Promise<{ finalUrl: string }>
```

- Host hiển thị ở tiền cảnh một khung xem web có khung ngoài (hiển thị tên miền hiện tại và nút đóng), người dùng tự hoàn thành các thao tác trên trang.
- Khung xem web dùng thư mục dữ liệu một lần, độc lập, khác origin với giao diện ứng dụng, và không thể truy cập `hg`.
- Trang web được tải bằng ngăn xếp mạng native của nền tảng, việc khai báo khả năng không có nghĩa mọi yêu cầu đều qua hg.http; phạm vi tác dụng của chính sách mạng xem [03 §2](03-host-rules.md).
- `doneWhen` ít nhất cho một điều kiện; khi cho đồng thời điều kiện URL và Cookie thì cả hai đều phải thỏa mãn.
  Điều kiện URL khớp cùng origin HTTPS và tiền tố đoạn đường dẫn đầy đủ. Ví dụ `/account` khớp `/account` và `/account/me`, không khớp `/accounting`. Tham số truy vấn và fragment không tham gia khớp đường dẫn.
  Chỉ điều hướng khung chính sau điều hướng bắt đầu hoặc thay đổi trạng thái Cookie mới có thể kích hoạt hoàn tất, không thể vì URL ban đầu đã khớp mà đóng ngay.
  Điều kiện Cookie phán đoán theo địa chỉ khung chính hiện tại của origin bắt đầu. Khi hoàn tất chỉ gửi các Cookie khớp với địa chỉ đó, và lưu ở host theo [§5.1](#51-phiên-đăng-nhập-web-do-host-quản-lý). `open` trả về `finalUrl`, plugin sau đó dùng giao diện tài khoản cùng origin để xác nhận trạng thái đăng nhập. localStorage của trang không đi vào giá trị trả về.
- Sau khi host xác nhận `doneWhen` thỏa mãn thì hoàn tất đăng nhập. Người dùng đóng khung xem trước khi hoàn tất thì ném `CANCELLED`, quá thời gian chờ ném `TIMEOUT`.
  Trong thời gian mở, tạm dừng ngân sách còn lại của lần gọi này, `ctx.deadline` động là `null`, sau khi đóng khôi phục theo ngân sách còn lại. Người dùng rời đi, hủy bỏ và ứng dụng bị treo vẫn có thể kết thúc lần gọi này và đóng khung xem.
- Cùng một plugin đồng thời tối đa một khung xem web. HGhub chỉ cho phép mở từ `auth.submit` khi người dùng gửi biểu mẫu xác thực, các lối vào khác ném `UNSUPPORTED`. Khi cửa sổ đã mở, yêu cầu mở lần thứ hai của cùng plugin ném `RATE_LIMITED`.

Sau khi người dùng bấm cách đăng nhập web của plugin, `auth.submit` có thể dùng quy trình sau.

```js
await hg.webview.open({ url: hg.endpoint.url("/login"), title: "登录账号",
  doneWhen: { urlPrefix: hg.endpoint.url("/account"), cookieNames: ["session"] } });
const user = await api.getProfile(ctx); // 通过同源账号接口查询
if (!user) throw new hg.PluginError("AUTH_INVALID", "站点尚未确认登录");
return { status: "ok", user };
```

## 13. `hg.PluginError`

```ts
new hg.PluginError(code: ErrorCode, message?: string, options?: {
  retryAfterSec?: number;
  retryable?: boolean;
  details?: Json;
  cause?: unknown;
})
```

`hg.PluginError` là lớp con của `Error`, thể hiện mang `code`, `retryable`, `retryAfterSec`, `details`.
Ngữ nghĩa và mã lỗi xem [02 §8](02-runtime.md).
