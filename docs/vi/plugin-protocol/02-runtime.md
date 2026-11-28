# 02 · Runtime

[← Tổng quan](README.md)

Chương này định nghĩa môi trường thực thi của mã plugin: có thể dùng gì, khi nào được tải, được gọi thế nào, khi lỗi thì biểu đạt ra sao.

## 1. Engine

Giao thức không gắn với engine cụ thể, chỉ quy định khế ước môi trường. Host **phải** thỏa mãn:

- Mức ngôn ngữ: hỗ trợ đầy đủ **ES2022** (gồm module, `async`/`await`, trường của lớp, `BigInt`, chuỗi tùy chọn).
- Mỗi plugin chạy trong một **runtime engine và ngữ cảnh độc lập**: không chia sẻ bất kỳ đối tượng hay trạng thái toàn cục nào với các plugin khác, với giao diện.
  Ngân sách bộ nhớ, ngắt thực thi và điều độ phải độc lập; chỉ tạo nhiều context trong cùng một runtime không thỏa mãn yêu cầu.
- Ngoài ngữ cảnh không có khả năng môi trường: không có hệ thống tệp, không có mạng native, không có tiến trình, không có kênh tới giao diện nội bộ của host.
- Có thể đặt giới hạn bộ nhớ và giới hạn ngăn xếp cho một ngữ cảnh đơn lẻ, và có thể ngắt việc thực thi của nó từ bên ngoài (ngăn vòng lặp vô hạn làm kẹt ứng dụng).

> Khuyến nghị triển khai (không quy phạm): chọn engine nhúng thực thi kiểu thông dịch, không phụ thuộc JIT (dòng QuickJS).
> Nó có hành vi nhất quán trên năm nền tảng, có thể dùng trên iOS, và hỗ trợ sẵn giới hạn bộ nhớ cùng ngắt thực thi.
> Việc chạy plugin trong iframe hoặc Worker của WebView không thỏa mãn điều 2, 4 ở trên.

Việc điều độ độc lập đảm bảo việc thực thi đồng bộ của một plugin không chiếm luồng giao diện hay chặn các plugin khác; có thể dùng luồng chuyên dụng hoặc cơ chế tương đương,
không yêu cầu mỗi plugin một tiến trình hệ điều hành. Giao thức này cách ly vòng lặp vô hạn cấp script, cấp phát vượt mức và lỗi engine có thể khôi phục,
không cam kết cách ly trong cùng tiến trình các sự cố như engine native sụp đổ, tiến trình bị hệ thống kết thúc.

## 2. Đối tượng toàn cục

### 2.1 Đảm bảo tồn tại

Ngoài các đối tượng nội tại chuẩn của ES2022, host **phải** cung cấp:

| Đối tượng toàn cục | Mô tả |
| --- | --- |
| `console` | `debug` / `log` / `info` / `warn` / `error`, tương đương `hg.log.*` |
| `setTimeout`, `clearTimeout` | Thuộc về lần gọi tạo ra nó, dọn dẹp khi lần gọi đó kết thúc; không có `setInterval` |
| `queueMicrotask` | — |
| `TextEncoder`, `TextDecoder` | Đảm bảo UTF-8; charset khác cần đặc tính `charset.legacy` |
| `URL`, `URLSearchParams` | Ngữ nghĩa URL của WHATWG |
| `atob`, `btoa` | — |
| `AbortController`, `AbortSignal` | — |
| `structuredClone` | Chỉ hỗ trợ giá trị truyền được |
| `crypto.getRandomValues`, `crypto.randomUUID` | Nguồn ngẫu nhiên an toàn mật mã học |

### 2.2 Đảm bảo không tồn tại

`fetch`, `XMLHttpRequest`, `WebSocket`, DOM (`window`, `document`, `navigator`),
`localStorage` / `sessionStorage` / `indexedDB`, `WebAssembly`, `Worker`, `setInterval`,
`require` / `process` / `Buffer` của Node.js.

Việc nối mạng dùng `hg.http`, lưu trữ bền vững dùng `hg.storage`.

## 3. Lối vào

```js
export const meta = { id: "demo", name: "示例源", version: "1.0.0", protocol: 1 };
export default function (hg) { /* 返回插件对象 */ }
```

- Xuất mặc định là **hàm factory**: nhận đối tượng API của host `hg` (xem [04](04-host-api.md)),
  trả về đối tượng plugin hoặc một Promise phân giải thành đối tượng plugin.
- Đối tượng plugin gồm một số **namespace khả năng**, namespace có tồn tại hay không tức là có triển khai khả năng đó hay không (xem [06](06-capabilities.md)).
- Factory **nên** nhẹ: chỉ làm xây dựng đối tượng và đọc cài đặt. Trong thời gian thực thi factory chỉ có thể dùng các module được chỉ định ở [04 §1](04-host-api.md).
- Tổng ngân sách thời gian của factory là 5 giây, bao gồm chờ Promise và microtask; quá thời gian chờ xử lý như tải thất bại với `TIMEOUT`. Promise treo mà không có nhiệm vụ chờ thực thi vẫn chờ phần ngân sách còn lại; mỗi lần await không đặt lại tổng ngân sách. Việc thực thi đồng bộ một đoạn vẫn chịu giới hạn 1 giây của §7.
- Lợi ích của việc tiêm `hg` làm tham số (thay vì biến toàn cục) là: tác giả có thể trong Node.js truyền vào một `hg` giả
  để kiểm thử đơn vị cho plugin, không cần khởi động host.

## 4. Giá trị truyền được

Tham số và giá trị trả về truyền giữa host và plugin **phải** là giá trị truyền được:

- `null`, boolean, số hữu hạn, chuỗi
- Mảng và đối tượng thông thường cấu thành từ các giá trị trên (không vòng, không phương thức nguyên mẫu tùy chỉnh)
- `Uint8Array` (có thể xuất hiện ở bất kỳ tầng nào)

Quy tắc:

- Thuộc tính đối tượng có giá trị `undefined` sẽ bị loại bỏ khi truyền.
- `Date`, `Map`, `Set`, thể hiện lớp, hàm, `Symbol`, `BigInt` không truyền được.
- Độ sâu lồng nhau của đối tượng ≤ 32; tham số hoặc giá trị trả về của một lần gọi ≤ 8 MiB.
- Việc truyền là **sao chép theo giá trị**: sau đó bất kỳ phía nào sửa đối tượng cũng không ảnh hưởng phía còn lại.

Các giới hạn trên áp dụng cho tham số nghiệp vụ, giá trị trả về nghiệp vụ của phương thức khả năng và dữ liệu bền vững. Dưới đây là các đối tượng gắn với runtime, không đi qua tuần tự hóa giá trị truyền được:
`hg`, `ctx` (gồm `AbortSignal`), `HttpRequest.signal`, `HtmlNode`, đối tượng lỗi và callback của bộ hẹn giờ.
Chúng không được nhúng vào giá trị trả về nghiệp vụ hoặc lưu trữ. Phương thức `void` được phép hoàn thành bằng `undefined`; phần tử mảng không được là `undefined`.
Cách tính kích thước là: thay mỗi Uint8Array thành null, loại bỏ thuộc tính đối tượng undefined, rồi tính số byte UTF-8 của JSON gọn,
cộng thêm tổng số byte nhị phân gốc; không thể chỉ tính số ký tự JS. Định nghĩa này dùng cho ngân sách, không yêu cầu cầu nối dùng mã hóa JSON.

## 5. Mô hình gọi

### 5.1 Hình dạng phương thức

Phương thức khả năng của plugin và phép biến đổi tài nguyên đều cùng một hình dạng (ngoại lệ hook gỡ bỏ dispose xem [06 §16](06-capabilities.md)):

```ts
method(params: object, ctx: CallContext): T | Promise<T>
```

- `params` luôn là một đối tượng đơn lẻ. Các phiên bản giao thức sau này chỉ **thêm trường tùy chọn** vào đó, plugin **không được** báo lỗi vì trường không quen thuộc.
- `ctx` là ngữ cảnh của lần gọi này:

| Trường | Mô tả |
| --- | --- |
| `requestId` | Định danh lần gọi, xuất hiện trong nhật ký của host |
| `deadline` | Thời hạn động chỉ đọc (mili giây Unix); trong thời gian khung xem web tạm dừng ngân sách của lần gọi thì là `null` |
| `signal` | Tín hiệu hủy bỏ: kích hoạt khi người dùng rời trang, lần gọi bị thay thế hoặc tới thời hạn |

Host **phải** thiết lập phạm vi nhiệm vụ cho mỗi lần gọi. HTTP, khung xem web, bộ hẹn giờ và các thao tác host bất đồng bộ sinh ra trong lần gọi
mặc định thuộc về phạm vi đó, qua `await` và callback của bộ hẹn giờ vẫn giữ thuộc tính, không thể mượn một lần gọi khác của cùng plugin để tiếp tục sống.
Plugin có thể truyền tiếp `ctx.signal` hoặc cung cấp tín hiệu hủy bỏ sớm hơn; tín hiệu rõ ràng và việc hủy lần gọi lấy "bất kỳ cái nào kích hoạt thì hủy", không thể thay thế phạm vi lần gọi.

Khi phương thức hoàn thành, thất bại hoặc bị hủy bỏ, host dừng các nhiệm vụ còn lại của nó, loại bỏ kết quả đến muộn, từ chối callback đến muộn tiếp tục phát khởi thao tác host.
Thao tác ghi đã gửi cho thượng nguồn và việc ghi lưu trữ nguyên tử đã hoàn thành sẽ không bị thu hồi; host không tự động thử lại thao tác ghi.

### 5.2 Thời hạn

| Loại lần gọi | Ngân sách mặc định | Giới hạn trên |
| --- | --- | --- |
| Đọc như danh sách, chi tiết, tìm kiếm | 15 giây | 30 giây |
| Phân giải phát (gồm resolver) | 20 giây | 60 giây |
| Gửi đăng nhập | 30 giây | 60 giây |
| Biến đổi tài nguyên (`transformResource`) | 5 giây | 10 giây |
| Hàm factory | 5 giây | 5 giây (không điều chỉnh được) |

Plugin có thể trong `meta.timeouts` (khóa là `read`, `playback`, `login`, `transform`, đơn vị giây) điều chỉnh ngân sách,
dùng cho tình huống thượng nguồn hơi chậm. Host **phải** kẹp giá trị khai báo giữa 5 giây và giới hạn trên của bảng trên: nới rộng thì được, vượt giới hạn trên thì không.

Sau khi quá thời gian chờ, host từ bỏ kết quả, kích hoạt `signal`, và trả về `TIMEOUT` cho giao diện. Timeout bất đồng bộ bình thường không tính là sự cố runtime;
khi việc thực thi đồng bộ vượt ngân sách bị ngắt cưỡng bức, lần gọi kích hoạt cũng trả về TIMEOUT, nhưng runtime phải được xây dựng lại và cộng dồn số lần sự cố.
Thời gian xếp hàng tính vào ngân sách lần gọi; việc chờ liên tiếp nhiều bước yêu cầu mạng không được đặt lại tổng ngân sách.
Ngoại lệ tạm dừng ngân sách của khung xem web xem [04 §12](04-host-api.md): `ctx.deadline` trong thời gian tạm dừng là `null`, sau khi khôi phục phản ánh thời hạn đã cập nhật.

### 5.3 Đồng thời

- Nhiều lần gọi của cùng một plugin xen kẽ thực thi tại các điểm `await` (ngữ nghĩa vòng lặp sự kiện đơn luồng). Plugin **không được** giả định các lần gọi là nối tiếp.
- HGhub mặc định số lần gọi đang bay đồng thời của cùng một plugin ≤ 8, xếp hàng ≤ 32; khi vượt, host trực tiếp trả về `RATE_LIMITED`, không đi vào plugin.
- Host **có thể** hợp nhất các lần gọi chỉ đọc đang bay có tham số hoàn toàn giống nhau.
- Giữa các plugin khác nhau là song song thật sự, không chặn lẫn nhau.
- Việc hủy bỏ một lần gọi bất đồng bộ thường không ảnh hưởng các lần gọi khác của cùng plugin; nếu việc thực thi đồng bộ bị ngắt, bộ nhớ vượt mức hoặc engine không thể tiếp tục,
  host hủy toàn bộ runtime của plugin: lần gọi kích hoạt trả về lỗi tương ứng, các lần gọi đang bay khác trong cùng runtime kết thúc bằng `INTERNAL` và ghi lại nguyên nhân,
  các lần gọi chưa thực thi trong hàng đợi có thể tiếp tục sau khi khởi tạo lại, nhưng không được phát lại phương thức đã đi vào plugin.

### 5.4 Thỏa thuận không trạng thái

Plugin **bất cứ lúc nào cũng có thể bị gỡ bỏ và tải lại**:

- Biến cấp module và trạng thái trong closure chỉ được coi là **bộ nhớ đệm**, mất đi phải xây dựng lại được.
- Dữ liệu cần giữ qua các lần gọi thì ghi vào `hg.storage`.
- Trạng thái phân trang phải được mã hóa trong con trỏ, không được giữ trong bộ nhớ.

## 6. Vòng đời

```text
已导入/停用 ──启用──▶ 初始化 ──能力与线路就绪──▶ 就绪
                      │                  │
                      │ 初始化失败/超时    │ 空闲超时 · 应用挂起 · 用户改动设置
                      ▼                  ▼
                   初始化失败          卸载（下次调用再加载）

就绪 ──同步执行中断/内存超限/可恢复引擎错误──▶ 运行时重建 ──连续 3 次──▶ 已停用（提示用户，可手动恢复）
```

- **Khởi tạo khi bật, khôi phục lười biếng sau khi nhàn rỗi**: Việc kiểm tra siêu thông tin lúc nhập không thực thi factory; lần bật đầu tiên thực thi quy trình sau,
  khởi tạo thành công mới công bố khả năng khả dụng cho giao diện:
  1. Kiểm tra `requires`, thực thi factory với tuyến đã lưu (không có thì là `null`); factory không được yêu cầu lần thực thi đầu tiên đã có tuyến.
  2. Kiểm chứng khả năng và các phương thức bắt buộc của nó theo `kind`, đệm danh sách phương thức hiệu lực.
  3. Nếu triển khai `endpoints`, gọi `endpoints.list({ refresh: false })`; phương thức phải cho phép current là `null`,
     lúc này không được phụ thuộc vào `hg.endpoint.url()`, có thể dùng địa chỉ dẫn đường trong plugin để lấy danh sách.
  4. Lựa chọn đã lưu vẫn còn trong danh sách thì dùng tiếp, nếu không chọn mục đầu tiên; danh sách rỗng thì khởi tạo thất bại. Không có endpoints thì current là `null`.
  5. Khi tuyến hiện tại thay đổi thì xây dựng lại ngữ cảnh, kiểm chứng lại khả năng, rồi mới mở các lần gọi nghiệp vụ. Các lần gọi nghiệp vụ trong lúc khởi tạo tạm thời không thực thi.
  Plugin đã sẵn sàng sau khi bị gỡ bỏ vì nhàn rỗi, lần gọi tiếp theo dùng tuyến đã lưu để xây dựng lại; không cần mỗi lần nối mạng liệt kê tuyến.
- **Gỡ bỏ khi nhàn rỗi**: Khi plugin không có lần gọi đang bay, host **có thể** gỡ bỏ nó bất cứ lúc nào; nhàn rỗi đủ 5 phút **nên** gỡ bỏ.
- Sau khi người dùng sửa cài đặt, chuyển tuyến trang, host **phải** hủy và xây dựng lại ngữ cảnh.
- Sau khi cài đặt thay đổi, thực thi lại toàn bộ khởi tạo; việc chuyển tuyến dùng danh sách đã lấy và tuyến đã chọn để khôi phục, không đệ quy chọn lại tuyến mặc định.
  Trước khi xây dựng lại hủy các lần gọi đang bay; nếu có phiên phát đang hoạt động, phiên theo [09](09-media-transport.md) dừng rồi người dùng phát lại.
- Khi ứng dụng bị treo (suspend) và không có phát hay tải xuống đang hoạt động, host **nên** gỡ bỏ toàn bộ plugin. Phiên media đang hoạt động có thể theo nhu cầu đánh thức plugin thực thi làm mới,
  biến đổi và báo cáo tiến độ; đây là lần gọi do host điều độ, không trao cho plugin khả năng thực thi nền tự do. Giới hạn treo của nền tảng xem [09 §7](09-media-transport.md).
- Sau khi một phương thức nghiệp vụ hoàn thành thành công một lần, đặt lại số đếm sự cố runtime liên tiếp; lỗi nghiệp vụ thông thường, lỗi mạng và timeout bất đồng bộ bình thường không cộng dồn.

## 7. Hạn mức

Host **phải** thực thi ngân sách tài nguyên. Bảng dưới đây phân biệt giới hạn trên cố định của giao thức và chính sách điều độ mặc định của HGhub; chính sách mặc định có thể tinh chỉnh,
không thể làm căn cứ để plugin suy đoán hiệu năng thiết bị. Việc thay đổi giới hạn trên cố định của giao thức xử lý theo quy tắc tương thích.

| Tài nguyên | Ngân sách | Tính chất |
| --- | --- | --- |
| Bộ nhớ heap | 64 MiB / plugin | Mặc định HGhub; bao gồm tài nguyên host như tài liệu HTML đã gắn |
| Ngăn xếp lần gọi | 1 MiB | Mặc định HGhub |
| Một đoạn thực thi đồng bộ (giữa hai lần nhường vòng lặp sự kiện) | 1 giây | Mặc định HGhub, sau khi ngắt xây dựng lại runtime |
| Lần gọi đang bay | 8, xếp hàng 32 | Mặc định HGhub (§5.3) |
| Tham số hoặc giá trị trả về của một lần gọi | 8 MiB | Giới hạn trên cố định của giao thức, gồm toàn bộ đối tượng chứ không chỉ trường nhị phân |
| Bộ hẹn giờ | 64 cái, một độ trễ ≤ 60 giây | Mặc định HGhub, vòng đời chịu ràng buộc ngân sách lần gọi |
| `hg.storage` | 5 MiB / plugin; khóa ≤ 256 byte; một giá trị ≤ 256 KiB | Giới hạn trên cố định của giao thức, không chia sẻ với bộ nhớ đệm HTTP/media |
| Nhật ký | Mỗi phút 240 dòng; một dòng ≤ 2 KiB; mỗi plugin giữ 2000 dòng gần nhất | Mặc định HGhub |

Hạn mức liên quan đến mạng xem [03 §3](03-host-rules.md) và [04 §2](04-host-api.md).

## 8. Mô hình lỗi

### 8.1 Cách biểu đạt

Plugin biểu đạt thất bại có thể dự kiến bằng cách ném `hg.PluginError`:

```js
throw new hg.PluginError("AUTH_REQUIRED", "请先登录");
throw new hg.PluginError("RATE_LIMITED", "请求过于频繁", { retryAfterSec: 30 });
```

- `message` sẽ hiển thị cho người dùng: **nên** ngắn gọn, dễ đọc, dùng ngôn ngữ của giao diện.
- `details` (tùy chọn) chỉ đi vào nhật ký plugin.
- Ném bất kỳ giá trị nào khác (Error thông thường, chuỗi, v.v.) đều coi là `INTERNAL`, thông điệp của nó không hiển thị cho người dùng, chỉ ghi vào nhật ký.

### 8.2 Mã lỗi

| Mã lỗi | Ý nghĩa | Xử lý mặc định của host | Có thể thử lại |
| --- | --- | --- | --- |
| `AUTH_REQUIRED` | Cần đăng nhập | Hướng dẫn đăng nhập | Sau khi đăng nhập |
| `AUTH_INVALID` | Thông tin xác thực sai | Gợi ý trong biểu mẫu đăng nhập | Không |
| `AUTH_EXPIRED` | Phiên mất hiệu lực | Xóa trạng thái đăng nhập và hướng dẫn đăng nhập lại | Sau khi đăng nhập |
| `NOT_FOUND` | Nội dung không tồn tại | Gợi ý; mục thư viện đánh dấu mất hiệu lực | Không |
| `CONTENT_LOCKED` | Nội dung tồn tại nhưng hiện không phát được | Hiển thị nguyên nhân; có thể thử đổi tuyến phát hoặc đổi nguồn | Không |
| `RATE_LIMITED` | Bị giới hạn tốc độ | Trong `retryAfterSec` cấm thử lại và gợi ý | Sau khi chờ |
| `UPSTREAM_ERROR` | Thượng nguồn trả về lỗi | Gợi ý; có thể thử lại | Có |
| `UPSTREAM_CHANGED` | Cấu trúc thượng nguồn không khớp với dự kiến của plugin | Gợi ý "plugin cần cập nhật", cung cấp lối vào kiểm tra cập nhật | Không |
| `INTERACTION_REQUIRED` | Cần người dùng hoàn thành thao tác như xác minh | Gợi ý vào trang plugin; host không suy đoán từ thông điệp lỗi hay details và tự động mở trang web | Sau thao tác của người dùng |
| `NETWORK` | Mạng không tới được, DNS, TLS thất bại | Gợi ý kiểm tra mạng | Có |
| `TIMEOUT` | Vượt thời hạn | Gợi ý; có thể thử lại | Có |
| `CANCELLED` | Lần gọi bị hủy bỏ | Im lặng | — |
| `INVALID_ARGUMENT` | Tham số không hợp lệ | Ghi vào nhật ký | Không |
| `STORAGE_FULL` | Hạn ngạch lưu trữ bền vững của plugin không đủ | Gợi ý dọn dẹp dữ liệu của plugin đó; không được tự động xóa trạng thái đăng nhập | Sau khi dọn dẹp |
| `UNSUPPORTED` | Tổ hợp khả năng hoặc tham số không được hỗ trợ | Im lặng hoặc gợi ý | Không |
| `INTERNAL` | Khuyết tật của plugin hoặc lỗi chưa phân loại | Gợi ý "plugin bị lỗi", có thể xem nhật ký | Có |

- Khi `retryable` của `PluginError` bị bỏ qua thì suy đoán theo bảng trên.
- Khi giao diện `hg.*` thất bại, cái ném ra cũng là `PluginError` (như `hg.http.fetch` ném `NETWORK`, `TIMEOUT`, `CANCELLED`).
  Plugin không bắt thì cứ ném lên nguyên trạng.
- Host gặp mã lỗi không quen thuộc **phải** xử lý theo `INTERNAL`.

### 8.3 `UPSTREAM_ERROR` và `UPSTREAM_CHANGED`

Sự khác biệt này quyết định người dùng nên làm gì, plugin **nên** phân biệt nghiêm túc:

- Thượng nguồn nói rõ là thất bại (HTTP 5xx, mã lỗi nghiệp vụ) → `UPSTREAM_ERROR`: thử lại sau có thể khôi phục.
- Phản hồi thành công nhưng cấu trúc không nhận ra (trường bị thiếu, kiểu không khớp, bộ chọn không lấy được node) → `UPSTREAM_CHANGED`:
  thử lại vô ích, cần cập nhật plugin.

## 9. Kiểm chứng giá trị trả về

Trước khi host giao dữ liệu plugin trả về cho giao diện **phải** kiểm chứng:

- **Cấu trúc**: Trường bắt buộc bị thiếu hoặc kiểu không khớp → lần gọi này thất bại bằng `INTERNAL`, nhật ký ghi lại đường dẫn của trường bị lỗi.
- **Chịu lỗi danh sách**: Khi một mục đơn lẻ trong danh sách không hợp lệ thì loại bỏ mục đó và ghi cảnh báo, không làm cả trang thất bại.
- **Trường không quen thuộc**: Loại bỏ, không truyền cho giao diện.
- **Chuỗi**: Văn bản hiển thị vượt giới hạn trên độ dài của trường thì cắt ngắn (giới hạn trên xem [05](05-content-model.md));
  các trường xử lý bằng máy như ID, địa chỉ, con trỏ và khóa không cắt ngắn, mà báo lỗi hoặc loại bỏ mục theo loại sở thuộc.
  Văn bản được kết xuất dưới dạng văn bản thuần; ngoại lệ duy nhất là các trường được đánh dấu rõ ràng là văn bản giàu (rich text), chúng phải qua làm sạch mới đi vào trang.
- **Số**: Phải hữu hạn; trường đếm phải là số nguyên không âm.
- **Địa chỉ**: Kiểm chứng theo công dụng của trường; tài nguyên host yêu cầu chỉ chấp nhận địa chỉ HTTPS tuyệt đối (khi khai báo `allowInsecureHttp` thì cho phép HTTP);
  ngoại lệ của luồng `external` xem [05 §7.3](05-content-model.md). Độ dài địa chỉ ≤ 8 KiB, quá dài thì loại bỏ hoặc báo lỗi, không cắt ngắn.

## 10. Nhật ký

- `hg.log.*`, `console.*`, cảnh báo kiểm chứng của host và lỗi runtime dùng chung hạn mức nhật ký và bộ đệm vòng của plugin đó, người dùng có thể xem và xuất trong trang chi tiết plugin. Các dòng vượt tốc độ bị loại bỏ và đếm, khi đọc nhật ký có thể thấy số lượng tích lũy; ảnh chụp vẫn không vượt 2000 dòng. Một dòng bản ghi JSON UTF-8 (gồm details) không vượt 2 KiB, văn bản quá dài cắt theo biên ký tự, details quá dài có thể trở thành văn bản JSON bị cắt.
- Nhật ký chỉ lưu trên máy này, host **không được** tự động tải lên.
