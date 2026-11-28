# 05 · Mô hình nội dung

[← Tổng quan](README.md)

Chương này định nghĩa hình dạng dữ liệu truyền giữa plugin và host. Kiểu chi tiết của các trường lấy tệp định nghĩa kiểu cho ra sau này làm chuẩn,
chương này nói rõ ngữ nghĩa, tính tùy chọn và giới hạn trên độ dài.

## 1. Quy tắc chung

| Quy tắc | Mô tả |
| --- | --- |
| Định danh | Mọi `id` đều là **chuỗi mờ** trong phạm vi plugin (1–256 byte). Host không phân tích nội dung của nó, chỉ dùng để truyền lại và ghi cục bộ |
| Tính ổn định của định danh | `id` **phải** ổn định lâu dài: host dùng nó làm khóa cho thư viện, lịch sử và tiến độ. Đừng đưa chữ ký, token hay mốc thời gian sẽ hết hạn vào `id` |
| Thời gian | Thống nhất mili giây Unix |
| Thời lượng | Thống nhất giây (tên trường kết thúc bằng `Sec`) |
| Văn bản | Thống nhất kết xuất dưới dạng văn bản thuần, ngoại lệ duy nhất là các trường được đánh dấu văn bản giàu (§9) |
| Địa chỉ | Tài nguyên host yêu cầu là địa chỉ `https:` tuyệt đối, khi khai báo `allowInsecureHttp` có thể dùng `http:`; ngoại lệ của luồng external xem §7.3 |
| Tùy chọn | Ngoài những cái đánh dấu rõ ràng "bắt buộc" ra đều có thể bỏ qua. Host dựa trên trường có tồn tại hay không quyết định ẩn hiện của phần tử giao diện |
| Giới hạn trên độ dài | Xem §8; văn bản hiển thị bị cắt ngắn, trường xử lý bằng máy kiểm chứng thất bại, mục danh sách vượt lượng bị loại bỏ |

Trường tùy chọn dùng việc bỏ qua để biểu thị "không có"; chỉ khi kiểu cho phép rõ ràng mới có thể trả về `null`, như `next` của phân trang.
Trường số truyền `0` biểu thị "thật sự là 0". Định danh không được cắt ngắn; định danh quá dài làm mục sở thuộc mất hiệu lực, tránh việc các ID khác nhau bị cắt thành cùng một giá trị.

## 2. Loại media

```ts
type MediaType =
  | "movie"        // 电影
  | "series"       // 剧集
  | "anime"        // 番剧
  | "variety"      // 综艺
  | "documentary"  // 纪录片
  | "short"        // 短剧、短视频
  | "live"         // 直播
  | "other";
```

Loại ảnh hưởng hình thái mặc định của giao diện host, nhưng bộ chọn phân tập cuối cùng hiển thị theo số phân tập và danh sách phát thực tế, không thể chỉ vì movie mà ẩn nhiều phân tập.
Plugin **có thể** bỏ qua `type` trên mục,
host xử lý theo `other`; loại media không quen thuộc cũng hạ cấp thành `other`. Một plugin có thể đồng thời cung cấp nhiều loại.

## 3. Mục media (`MediaItem`)

Thẻ danh sách, kết quả tìm kiếm, mục thư viện đều dùng nó.

| Trường | Bắt buộc | Kiểu | Mô tả |
| --- | --- | --- | --- |
| `id` | Có | string | Định danh duy nhất trong plugin |
| `title` | Có | string | Tiêu đề |
| `type` | Không | MediaType | Loại media |
| `cover` | Không | ImageRef | Ảnh bìa dọc |
| `backdrop` | Không | ImageRef | Hình ngang, dùng cho đầu trang chi tiết |
| `description` | Không | string | Giới thiệu |
| `tags` | Không | string[] | Nhãn |
| `score` | Không | number | Điểm đánh giá, 0–10 |
| `year` | Không | number | Năm |
| `status` | Không | `"ongoing" \| "finished" \| "upcoming"` | Trạng thái cập nhật |
| `episodeCount` | Không | number | Số tập đã cập nhật |
| `totalEpisodes` | Không | number | Tổng số tập dự kiến |
| `durationSec` | Không | number | Thời lượng một tập |
| `stats` | Không | `{ plays?, hot?, likes?, favorites?, comments? }` | Số đếm dùng cho hiển thị |
| `author` | Không | Author | Tác giả, chủ kênh, phía sản xuất |
| `publishedAt` | Không | number | Thời gian phát hành |
| `updatedAt` | Không | number | Thời gian cập nhật gần nhất |
| `badge` | Không | string | Văn bản ngắn của huy hiệu, ≤ 8 ký tự (như "bản gốc") |
| `entryEpisodeId` | Không | string | Lối vào phát trực tiếp của luồng thông tin nhập vai: cho nó, host không cần lấy chi tiết trước vẫn phân giải phát được |

```ts
interface Author {
  id: string;
  name: string;
  avatar?: ImageRef;
  description?: string;
  followers?: number;
}
```

## 4. Chi tiết (`MediaDetail`)

Trên cơ sở `MediaItem` thêm:

| Trường | Bắt buộc | Mô tả |
| --- | --- | --- |
| `playlists` | Có | Danh sách phân tập, ít nhất một mục (§5) |
| `seasons` | Không | Tên hiển thị của mùa: `{ number, label }[]`. Khi phân tập mang `season`, host nhóm theo mùa và dùng tên ở đây |
| `people` | Không | Diễn viên và đoàn làm phim: `{ name, role?, avatar? }[]` |
| `trailers` | Không | Đoạn giới thiệu: `Stream[]` (§7.3) |
| `interaction` | Không | Trạng thái của người dùng hiện tại: `{ liked?, favorited?, likedEpisodeIds? }` |
| `related` | Không | Đề xuất liên quan |

Phim điện ảnh, nội dung một tập cũng trả về `playlists`, chỉ là bên trong chỉ có một tập.

## 5. Danh sách phát (`Playlist`) và phân tập (`Episode`)

Nhiều nguồn, nhiều bản lồng tiếng, bản chính và phần phụ của cùng một tác phẩm, mỗi cái là một `Playlist`.

```ts
interface Playlist {
  id: string;
  label?: string;                    // 如"线路 1""国语""花絮"
  kind?: "main" | "dub" | "extra" | "other";   // 默认 main
  episodes: Episode[];
  next?: string | null;              // 分集未一次给完时的游标（配合 listEpisodes）
}

interface Episode {
  id: string;
  index: number;                     // 从 1 开始的正整数；同一 Playlist 内唯一的列表排序序号
  season?: number;                   // 季号，非负整数，0 可用于特别篇；给出后宿主按季分组展示
  episodeNumber?: number;            // 真实集号，非负有限数字，可为 12.5；用于匹配，不等于排序 index
  label?: string;                    // 展示用编号文本，如 "12"、"SP1"、"12.5"；省略时用 index
  title?: string;
  durationSec?: number;
  airDate?: number;                  // 播出时间
  thumbnail?: ImageRef;
  locked?: boolean;                  // 当前不可播放
  lockReason?: string;               // 置灰时展示的原因
}
```

- `season`, `episodeNumber` biểu đạt số mùa/tập thật, `index` chỉ biểu đạt thứ tự trong danh sách phát hiện tại, không chịu trách nhiệm khớp định danh xuyên nguồn.
- Khi chuyển danh sách phát, ưu tiên khớp theo số mùa, số tập rõ ràng; khi không có số rõ ràng có thể dùng `index` đề xuất ứng viên,
  nhưng không được coi các bản cắt khác nhau, bản chính và phần phụ, bản tách tập là khớp xác định. Khi có mơ hồ thì để người dùng chọn tập.
- Khi số phân tập rất lớn (như xem lại trực tiếp hàng nghìn tập), plugin có thể trong `playlists[].episodes` chỉ cho trang đầu tiên,
  và dùng `next` phối hợp khả năng `media.listEpisodes` để phân trang (xem [06](06-capabilities.md)).

## 6. Phân trang

```ts
interface PageParams { cursor?: string; limit?: number }   // limit 是宿主的期望条数提示（1–100）

interface Page<T> {
  items: T[];
  next?: string | null;     // 下一页游标；null 或省略表示没有更多
  total?: number;           // 已知总数（可选，仅用于展示）
}
```

Con trỏ mờ đối với host, plugin có thể mã hóa tùy ý (thượng nguồn dạng số trang thì ghi số trang vào con trỏ là được).
Con trỏ **không được** phụ thuộc vào trạng thái bộ nhớ của plugin (xem [02 §5.4](02-runtime.md)).

## 7. Tham chiếu tài nguyên và phát

### 7.1 Hình ảnh và các tài nguyên khác

```ts
type ImageRef = string | ResourceRef;

interface ResourceRef {
  url: string;
  headers?: Record<string, string>;      // 取回时附带的请求头
  transform?: ResourceTransform;         // 取回后的字节变换
}
```

Cho trực tiếp địa chỉ dạng chuỗi là hình thức thường gặp nhất; khi cần header yêu cầu tùy chỉnh (chống lấy cắp liên kết) hoặc giải mã thì dùng hình thức đối tượng.

### 7.2 Biến đổi byte

Ảnh bìa, phân đoạn của một số trang không phải hình ảnh hoặc video chuẩn, cần giải mã.

```ts
type ResourceTransform =
  | { type: "aes-cbc"; key: string; iv: string; padding?: "pkcs7" | "none" }  // key/iv 为 Base64
  | { type: "plugin"; name: string; state?: Json };
```

- `aes-cbc` do host thực thi native, không chiếm runtime của plugin, thích hợp cho các tài nguyên nhỏ tần suất cao như ảnh bìa.
- `plugin` do host gọi `transformResource` của plugin (xem [06](06-capabilities.md)),
  `name` và `state` được truyền lại nguyên trạng. Thích hợp cho thuật toán không chuẩn.
- Biến đổi phân đoạn cần host hỗ trợ đặc tính `transform.segment`.
- Đối tượng biến đổi lấy tài nguyên đầy đủ làm đơn vị, không phải callback dạng truyền phát. Đầu vào và đầu ra phải lần lượt thỏa mãn giới hạn trên kích thước toàn bộ lần gọi 8 MiB;
  host đặt ngân sách kích thước tài nguyên trước khi đọc vào đầy đủ, và dành chỗ cho các trường khác như URL, state.
  1.0 không hỗ trợ thực thi biến đổi plugin đối với toàn bộ video progressive, cũng không cam kết phân đoạn kích thước tùy ý đều biến đổi được.
  Thứ tự yêu cầu theo khoảng của media và biến đổi xem [09 §4](09-media-transport.md).

### 7.3 Thông tin phát

```ts
interface PlaybackInfo {
  streams: Stream[];              // 至少一条；宿主结合设备能力与用户设置挑选
  servers?: PlaybackServer[];     // 可切换的播放线路
  subtitles?: SubtitleTrack[];
  expiresAt?: number;             // 本次播放信息失效时间；刷新沿宿主记录的解析链进行（09 §5）
  durationSec?: number;
  resumeSec?: number;             // 上游记录的续播位置（宿主本地记录优先）
  skip?: { introEndSec?: number; outroStartSec?: number };
}

interface PlaybackServer {
  id: string;
  label?: string;
}

interface Stream {
  url: string;
  kind?: "media" | "page" | "external";    // 默认 media
  format?: "hls" | "dash" | "progressive"; // kind 为 media 时必需
  codec?: "h264" | "hevc" | "av1";
  label?: string;                          // 如 "1080P"
  height?: number;
  bitrate?: number;                        // bit/s
  serverId?: string;                       // 属于哪条播放线路
  live?: boolean;
  headers?: Record<string, string>;        // 取清单、密钥、分片时附带的请求头
  resolverId?: string;                     // kind 为 page 时，指定用哪个解析器插件
  transforms?: {
    playlist?: ResourceTransform;
    segment?: ResourceTransform;
    key?: ResourceTransform;
  };
  external?: { app?: string; mime?: string };   // kind 为 external 时的提示信息
}
```

Ý nghĩa của ba loại `kind`:

| `kind` | Ý nghĩa | Hành vi của host |
| --- | --- | --- |
| `media` | Địa chỉ media có thể phát trực tiếp | Giao cho trình phát |
| `page` | Chỉ có trang phát, cần phân giải | Giao cho plugin resolver đã cài ([06 §3](06-capabilities.md)); khi không có resolver khả dụng thì gợi ý người dùng |
| `external` | Không thể phát trong ứng dụng (liên kết magnet, trang bên ngoài) | Sau khi hỏi người dùng thì giao cho hệ thống hoặc ứng dụng khác mở |

`external.url` là địa chỉ mở bên ngoài, cho phép HTTP(S), `magnet:`, `ed2k:`, `rtsp:`, `rtmp:`;
HTTP ở đây chỉ giao cho ứng dụng bên ngoài, không vì `allowInsecureHttp` tắt mà bị từ chối như yêu cầu mạng của host.
Host có thể vì nền tảng không có chương trình xử lý mà gợi ý không hỗ trợ; không được giao các địa chỉ này cho proxy media yêu cầu, cũng không chấp nhận `javascript:`, `data:`, `file:`.
Các tài nguyên nghiệp vụ khác vẫn tuân theo quy tắc HTTP(S). Việc mở bên ngoài không truyền Cookie, header yêu cầu hay phép biến đổi mà host lưu.

**Tuyến phát**: Khi một tập có nhiều máy chủ lấy luồng, plugin liệt kê chúng trong `servers`, và trong `streams` cho
luồng của tuyến mặc định (hoặc tuyến được yêu cầu) đó. Khi người dùng chuyển tuyến, host mang `serverId` gọi lại `resolvePlayback`.
Như vậy việc hiển thị danh sách tuyến không cần yêu cầu từng tuyến một.

### 7.4 Header yêu cầu

Header yêu cầu của yêu cầu media (hình ảnh, danh sách phát, khóa, phân đoạn, phụ đề) được hợp nhất theo độ ưu tiên sau, ưu tiên cao ghi đè ưu tiên thấp:

| Độ ưu tiên | Nguồn |
| --- | --- |
| 1 | `headers` của chính mục hoặc luồng (`ResourceRef.headers`, `Stream.headers`, `SubtitleTrack.headers`) |
| 2 | `meta.defaultHeaders` của plugin (xem [01 §2.1](01-plugin-file.md)) |
| 3 | Giá trị mặc định của host: `User-Agent` là `HGhub/<phiên bản>`, còn lại không đặt |

- Hình ảnh và luồng của cùng một trang trong nguồn phim ảnh thường cần cùng một `Referer`, dùng giá trị mặc định khai báo là được, không cần lặp lại từng mục.
- Cả ba tầng này đều **không được** ghi đè các header yêu cầu do tầng truyền tải quyết định ([03 §2](03-host-rules.md)).
- **Quy tắc cùng origin có hiệu lực với mỗi tầng**: `Cookie` và `Authorization` dù đến từ tầng nào, cũng chỉ gửi cho địa chỉ **cùng origin** với địa chỉ tài nguyên
  (`Stream.url` hoặc `ResourceRef.url`), không đi theo danh sách chuyển tới CDN của bên thứ ba.
  Header mặc định không thể vượt qua quy tắc này — nếu không một header mặc định kiểu `Referer` cũng có thể mang thông tin xác thực tới CDN.
- `Range` của yêu cầu media do trình phát và host điều khiển, plugin và header mặc định đều không được chỉ định.
- Tên header yêu cầu không phân biệt chữ hoa thường. Yêu cầu media theo kho Cookie của nhà cung cấp tài nguyên khớp bình thường với URL đích; header Cookie rõ ràng ghi đè Cookie tự động.
  Cookie tự động không mượn xuyên plugin, tài nguyên xuyên miền chỉ mang Cookie mà chính nó khớp được. Quy tắc hợp nhất nhà cung cấp tài nguyên và resolver xem [09](09-media-transport.md).

### 7.5 Rãnh phụ đề

```ts
interface SubtitleTrack {
  url: string;
  format: "vtt" | "srt" | "ass";
  language?: string;      // BCP 47
  label?: string;
  default?: boolean;
  headers?: Record<string, string>;
}
```

Host chịu trách nhiệm chuyển đổi SRT → VTT và nhận diện mã hóa; rãnh `ass` có kết xuất hay không do host quyết định, khi không hỗ trợ thì bỏ qua rãnh đó.

## 8. Giới hạn trên độ dài

Host **phải** khi vượt giới hạn thì cắt ngắn văn bản hiển thị hoặc loại bỏ các mục danh sách vượt lượng, và ghi lại một cảnh báo; các trường xử lý bằng máy như định danh, con trỏ, địa chỉ, khóa
không được cắt ngắn, khi không hợp lệ làm mục hoặc lần gọi sở thuộc thất bại. Việc cắt ngắn danh sách không thể giả mạo con trỏ next; tính đầy đủ của phân trang do plugin đảm bảo.

| Trường | Giới hạn trên |
| --- | --- |
| `title`, `name`, `label` | 200 ký tự |
| `description`, `summary` | 5000 ký tự |
| `badge` | 8 ký tự |
| Một `tag` đơn lẻ | 30 ký tự |
| `tags` | 20 mục |
| Địa chỉ | 8 KiB |
| Nội dung chính văn bản giàu `contentHtml` | 512 KiB |
| Mục một trang | 100 |
| `playlists` | 20 |
| Một `playlists[].episodes` | 5000 |
| `streams` | 16 |
| `servers` | 20 |
| `subtitles` | 32 |

## 9. Văn bản giàu

Nội dung dạng đồ văn có thể trả về HTML:

- Host **phải** làm sạch trước rồi mới kết xuất: chỉ giữ định dạng cơ bản, liên kết và hình ảnh, loại bỏ script, style, thuộc tính sự kiện,
  style nội tuyến, `iframe` và biểu mẫu; liên kết thống nhất giao cho trình duyệt bên ngoài mở.
- Địa chỉ hình ảnh trong nội dung chính được host viết lại thành giao thức media cục bộ, cũng chịu ràng buộc quy tắc mạng.
- Plugin **nên** chỉ trả về đoạn nội dung chính (như `<p>`, `<img>`), đừng trả về HTML của cả trang.
