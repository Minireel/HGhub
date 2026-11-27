# 06 · Mô-đun khả năng

[← Tổng quan](README.md)

Đối tượng plugin gồm một số **namespace khả năng**. Một namespace tồn tại tức là plugin đã triển khai khả năng đó;
host dựa vào đó quyết định hiển thị những lối vào nào trên giao diện.

```js
export default function (hg) {
  return {
    media:  { getDetail, resolvePlayback },   // 点播源必需；纯直播源可只实现 live
    search: { search },                        // 可选：有它才有搜索
    home:   { getSections },                   // 可选：有它才有首页内容
  };
}
```

Mọi phương thức khả năng và transformResource đều có hình dạng `(params, ctx)` (xem [02 §5](02-runtime.md));
hook gỡ bỏ dispose là ngoại lệ. Kiểu dữ liệu xem [05](05-content-model.md).
Phương thức có dấu `?` là tùy chọn trong khả năng đó.

## 1. Tổng quan khả năng

| Khả năng | Phương thức | Giao diện tương ứng |
| --- | --- | --- |
| **`media`** (bắt buộc với nguồn VOD) | `getDetail` `resolvePlayback` `listEpisodes?` `matchUrl?` | Trang chi tiết, trình phát |
| **`resolver`** (bắt buộc với resolver) | `resolve` | Không có giao diện độc lập; cung cấp địa chỉ phát cho nguồn dữ liệu |
| `home` | `getSections` `listSection?` | Trang chủ (phân khu) |
| `feed` | `list` | Luồng thông tin nhập vai (phát dọc liên tục) |
| `catalog` | `getCategories` `list` `getFilters?` | Trang danh mục và lọc |
| `search` | `search` `getFilters?` `suggest?` `trending?` | Trang tìm kiếm, tìm kiếm tổng hợp, đổi nguồn |
| `live` | `getGroups` `listChannels` `resolve` `getProgrammes?` | Kênh trực tiếp |
| `articles` | `list` `getDetail` | Đồ văn |
| `comments` | `list` `send?` `like?` | Bảng bình luận |
| `interact` | `like?` `favorite?` `follow?` | Nút lượt thích, yêu thích, theo dõi |
| `auth` | `getMethods` `submit` `getProfile` `logout` | Biểu mẫu đăng nhập ở trang plugin |
| `library` | `listMedia` `listFolders?` `listFollowing?` | Yêu thích và lịch sử đám mây |
| `progress` | `report` | Không có giao diện (báo cáo tiến độ phát) |
| `updates` | `check` | Huy hiệu "có cập nhật" của thư viện |
| `endpoints` | `list` | Lựa chọn tuyến trang ở trang plugin |
| `ui` | `getActions` `submit` | Biểu mẫu thao tác ở trang plugin |
| — | `transformResource?` `dispose?` | Hook tầng cao nhất (§16) |

- **Nguồn dữ liệu** (`meta.kind` là `source`, mặc định) ít nhất triển khai một trong `media` hoặc `live`, có thể triển khai cả hai.
  Khi triển khai `media` **nên** ít nhất có `home`, `feed`, `catalog`, `search` hoặc `media.matchUrl` làm lối vào nội dung.
  Lối vào của nguồn thuần trực tiếp là danh sách kênh, không cần giả mạo chi tiết VOD. Các khả năng trả về `MediaItem` VOD yêu cầu đồng thời triển khai `media`.
- **Resolver** (`meta.kind` là `resolver`) chỉ cần `resolver` và `meta.matches`.
- Host khi khởi tạo kiểm chứng các phương thức bắt buộc của mỗi namespace, và ghi lại các phương thức tùy chọn có tồn tại hay không, giao diện hiển thị nút theo phương thức cụ thể;
  không thể chỉ vì tồn tại một đối tượng `interact` rỗng mà hiển thị toàn bộ tương tác. Factory nên giữ tập khả năng ổn định, không thay đổi theo trạng thái đăng nhập; giới hạn đăng nhập dùng mã lỗi biểu đạt.

Phần giao diện xem [08 Giao diện host](08-frontend.md). Bảng xếp hạng không có trang độc lập: bảng xếp hạng của thượng nguồn dùng mục sắp xếp của `catalog`
hoặc một phân khu trong `home` để biểu đạt.

## 2. `media` (bắt buộc với nguồn VOD)

```ts
media: {
  getDetail(params: { id: string }, ctx): MediaDetail

  resolvePlayback(params: {
    mediaId: string;
    episodeId: string;
    playlistId?: string;
    serverId?: string;                         // 用户选定的播放线路；省略由插件选默认
    reason: "play" | "preload" | "refresh";
    codecs: ("h264" | "hevc" | "av1")[];       // 当前播放路径实际可用，按偏好排序
    maxHeight?: number;
  }, ctx): PlaybackInfo

  listEpisodes?(params: PageParams & { mediaId: string; playlistId: string }, ctx): Page<Episode>

  matchUrl?(params: { url: string }, ctx): { mediaId: string; episodeId?: string } | null
}
```

- **`getDetail`**: Trả về `MediaDetail`, trong đó `playlists` ít nhất một mục.
- **`resolvePlayback`**: Trả về ít nhất một `streams`.
  - `reason: "play"` người dùng sắp xem; `"preload"` host tải trước cho mục tiếp theo, plugin có thể đối xử như nhau;
    `"refresh"` phân giải lại sau khi địa chỉ hết hạn giữa chừng hoặc lấy luồng thất bại, plugin **không nên** trả về địa chỉ cũ đã đệm.
  - Host hiện tại đã nối tải trước tập tiếp theo `preload`: kết quả có thể bị hủy bỏ, cũng có thể dùng trực tiếp cho việc phát, khi trúng phiên chuẩn bị hiệu lực sẽ không gọi lại một lần `play` nữa. Vì vậy tải trước nên trả về `PlaybackInfo` đầy đủ khả dụng, khai báo chính xác `expiresAt`, thao tác mạng dùng `ctx.signal`.
    Tải trước không có nghĩa người dùng đã xem; plugin hỗ trợ báo cáo tiến độ nên lấy sự kiện phát thực tế của `progress.report` để ghi việc xem. Khi cần đăng nhập hoặc lựa chọn thì trả về lỗi giao thức bình thường, tải trước không bật lên giao diện tương tác, sau khi người dùng chọn tập đó mới đi xử lý bình thường. Phạm vi tải trước theo byte có giới hạn và các tình huống chưa hỗ trợ xem [Thuyết minh triển khai](IMPLEMENTATION.md#tối-ưu-phát-nối-tiếp-và-phát-lần-đầu-2026-11-16).
  - Khi có nhiều tuyến phát thì liệt kê trong `servers`; khi `serverId` được cho thì trả về luồng của tuyến đó.
  - Khi không lấy được địa chỉ phát trực tiếp, chỉ có trang phát, thì trả về luồng `kind: "page"` giao cho resolver (§3).
  - Khi không lấy được bất kỳ địa chỉ nào thì ném `CONTENT_LOCKED` (giới hạn của thượng nguồn) hoặc `UPSTREAM_CHANGED` (cấu trúc đã thay đổi).
- **`listEpisodes`**: Chỉ khi `playlists[].next` không rỗng mới được host gọi, dùng cho nội dung có nhiều phân tập.
- **`matchUrl`**: Khi người dùng dán một liên kết của trang thượng nguồn, ánh xạ nó thành mục.
  Plugin khai báo trong `meta.linkDomains` (mảng tên host) mình nhận biết những tên miền nào, host dựa vào đó chọn plugin, không cần tải toàn bộ plugin.
  Khi không nhận biết địa chỉ đó thì trả về `null`.

## 3. `resolver` (resolver)

Rất nhiều nguồn phim ảnh dùng cùng một lô nhà cung cấp lưu trữ video. Việc biến "trang phát của một nhà lưu trữ nào đó → địa chỉ có thể phát" thành plugin độc lập,
nhiều nguồn dữ liệu có thể dùng chung một resolver.

```ts
meta: {
  kind: "resolver",
  matches: ["*.host-a.example", "player.host-b.example"],
  name: "某某解析器",
}

resolver: {
  resolve(params: {
    url: string;                               // 待解析的播放页地址
    headers?: Record<string, string>;          // 数据源给出的非凭据请求头
    serverId?: string;                         // 仅属于当前解析器的播放线路
    reason: "play" | "preload" | "refresh";
    codecs: ("h264" | "hevc" | "av1")[];
    maxHeight?: number;
  }, ctx): PlaybackInfo
}
```

`meta.matches` là mẫu host mà resolver **xử lý được** (cách viết xem dưới đây). Thời điểm host gọi:

1. Nguồn dữ liệu trả về luồng `kind: "page"`.
2. Host chọn resolver theo host: khi `resolverId` của luồng chỉ định plugin thì dùng nó; nếu không thì khớp `matches`;
   khi trúng nhiều cái thì dùng lựa chọn người dùng đã ghi nhớ cho host đó, không có ghi nhớ thì để người dùng chọn (có thể chọn "ghi nhớ").
3. Host dùng `url` và header yêu cầu không phải thông tin xác thực mà nguồn dữ liệu đưa ra để gọi `resolver.resolve`, coi các luồng trả về là luồng của tập đó giao cho trình phát.
4. Khi không có resolver khả dụng, gợi ý người dùng "nguồn này cần cài resolver", và liệt kê các resolver đã cài hiện tại để chọn.

Biên giới:

- Resolver và nguồn dữ liệu không nhìn thấy nhau: nó không lấy được lưu trữ, Cookie và thông tin xác thực của nguồn dữ liệu, chỉ lấy được địa chỉ trang và header yêu cầu.
  Header yêu cầu `Authorization` và `Cookie` **không được** chuyển giao cho resolver.
- Yêu cầu mạng của chính resolver do nó tự chịu trách nhiệm, độc lập với yêu cầu của nguồn dữ liệu.
- Resolver 1.0 chỉ có thể trả về luồng `media` hoặc `external`, không được trả về `page` nữa; không làm phân giải đệ quy.
- Thuộc tính tài nguyên của kết quả phân giải, hợp nhất phụ đề và thời hạn hiệu lực, thứ tự làm mới xem [09 §2](09-media-transport.md);
  `resolverId` không được trỏ tới plugin đã vô hiệu hóa, không tương thích hoặc khởi tạo thất bại; khi chỉ định không khả dụng thì gợi ý rõ ràng, không lặng lẽ đổi sang resolver khác.
- Địa chỉ trang có thể chứa tham số truy vấn liên quan đến phiên (địa chỉ có chữ ký do chính nguồn dữ liệu sinh ra rất thường gặp).
  Nguồn dữ liệu **nên** tránh đưa token hiệu lực lâu dài vào địa chỉ giao cho resolver.

Cách viết mẫu host:

| Cách viết | Khớp |
| --- | --- |
| `api.example.com` | Chỉ host đó |
| `*.example.com` | Tên miền con bất kỳ tầng nào của `example.com`, không gồm chính `example.com` |
| `*` | Host bất kỳ. Chỉ có ý nghĩa với resolver, sẽ khiến nó trở thành ứng viên trên mọi luồng `page` |

## 4. `home`

```ts
home: {
  getSections(params: PageParams, ctx): Page<Section>
  listSection?(params: PageParams & { sectionId: string }, ctx): Page<MediaItem>
}

interface Section {
  id: string;
  title?: string;
  layout: "banner" | "row" | "grid" | "list";
  imageAspect?: "portrait" | "landscape";   // 卡片用竖版封面还是横版图，默认 portrait
  items: MediaItem[];
  more?: boolean;        // 为 true 时显示"查看更多"，宿主调用 listSection
}
```

| `layout` | Trình bày |
| --- | --- |
| `banner` | Băng chuyền ở trên cùng, dùng `backdrop` (thiếu thì quay về `cover`) |
| `row` | Một hàng thẻ trượt ngang |
| `grid` | Lưới nhiều cột |
| `list` | Danh sách dọc có giới thiệu |

- Bản thân phân khu có thể phân trang (`getSections` trả về `next`), thích hợp cho "luồng đề xuất kéo xuống vô hạn".
- Bố cục là **gợi ý**: host có thể điều chỉnh theo kích thước màn hình, nhưng **nên** giữ thứ tự phân khu.
- Host ở chế độ tổng hợp trang chủ sẽ ghép các phân khu của nhiều plugin lại hiển thị, và ghi nhãn nguồn (xem [08 §3](08-frontend.md)).

## 5. `feed`

```ts
feed: {
  list(params: PageParams, ctx): Page<MediaItem>
}
```

Luồng thông tin nhập vai: mỗi nội dung hiển thị toàn màn hình và tự động phát. Mục **nên** mang `entryEpisodeId`,
nếu không host cần gọi `getDetail` trước mới phát được. Thích hợp cho thượng nguồn dạng phim ngắn, video ngắn.

## 6. `catalog`

```ts
catalog: {
  getCategories(params: {}, ctx): Category[]
  list(params: PageParams & {
    categoryId: string;
    filters?: Record<string, string[]>;   // 过滤器 id → 已选项 id
    sort?: string;
  }, ctx): Page<MediaItem>
  getFilters?(params: { categoryId: string }, ctx): { filters?: Filter[]; sorts?: FilterOption[] }
}

interface Category {
  id: string;
  label: string;
  children?: Category[];        // 至多两层
  filters?: Filter[];
  sorts?: FilterOption[];
}

interface Filter {
  id: string;
  label: string;
  options: FilterOption[];
  multiple?: boolean;           // 默认单选
}

interface FilterOption { id: string; label: string }
```

- Bộ lọc có thể cho tĩnh theo danh mục (`Category.filters`), cũng có thể lấy theo nhu cầu (`getFilters`, thích hợp cho thượng nguồn mà mục lọc thay đổi theo danh mục).
  Khi cả hai đều có thì lấy kết quả của `getFilters` làm chuẩn.
- `sorts` dùng để biểu đạt cách sắp xếp của thượng nguồn (mới nhất, hot nhất, điểm đánh giá, v.v.). Bảng xếp hạng của thượng nguồn có thể làm thành một danh mục cố định cộng một nhóm mục sắp xếp.
- Host **nên** đệm cây danh mục, ít nhất trong một phiên không yêu cầu trùng lặp.
- Trang danh mục hiển thị theo cột plugin (xem [08 §4](08-frontend.md)).

## 7. `search`

```ts
search: {
  search(params: PageParams & {
    query: string;
    filters?: Record<string, string[]>;
  }, ctx): Page<MediaItem>
  getFilters?(params: {}, ctx): Filter[]
  suggest?(params: { query: string }, ctx): string[]     // 输入联想
  trending?(params: {}, ctx): string[]                    // 热搜词
}
```

- `search` là cơ sở của tìm kiếm tổng hợp và đổi nguồn (§17). Plugin muốn tham gia tổng hợp **nên** đảm bảo nó trả về trong vài giây.
- `suggest` được host gọi sau khi chống rung (debounce), và chỉ nhằm vào plugin hiện tại, không tổng hợp xuyên plugin.
- Plugin **không nên** ghi từ khóa tìm kiếm vào nhật ký.

## 8. `live`

```ts
live: {
  getGroups(params: {}, ctx): LiveGroup[]
  listChannels(params: PageParams & { groupId?: string }, ctx): Page<Channel>
  resolve(params: {
    channelId: string; serverId?: string; codecs: ("h264" | "hevc" | "av1")[];
    maxHeight?: number; reason: "play" | "preload" | "refresh";
  }, ctx): PlaybackInfo
  getProgrammes?(params: { channelId: string; from: number; to: number }, ctx): Programme[]
}

interface LiveGroup { id: string; label: string }

interface Channel {
  id: string;
  name: string;
  logo?: ImageRef;
  number?: number;           // 频道号
  groupId?: string;
  now?: string;              // 正在播出的节目名
}

interface Programme {
  title: string;
  startAt: number;
  endAt: number;
  description?: string;
}
```

- `streams` mà `resolve` trả về **nên** mang `live: true`. Nhiều nguồn tín hiệu dùng `servers` biểu đạt.
- `reason: "refresh"` giống như VOD, phải lấy lại địa chỉ hiệu lực; kênh trực tiếp có thể thiết lập phiên media và chuỗi phân giải giống VOD.
- Danh sách chương trình (`getProgrammes`) là tùy chọn; host chỉ yêu cầu khi người dùng mở chi tiết kênh.

## 9. `articles`

```ts
articles: {
  list(params: PageParams, ctx): Page<ArticleItem>
  getDetail(params: { id: string }, ctx): ArticleDetail
}

interface ArticleItem {
  id: string;
  title: string;
  summary?: string;
  cover?: ImageRef;
  tags?: string[];
  author?: Author;
  stats?: { likes?: number; comments?: number; favorites?: number; hot?: number };
  publishedAt?: number;
  mediaId?: string;          // 关联的作品，宿主提供跳转
}

interface ArticleDetail extends ArticleItem {
  contentHtml?: string;      // 富文本正文，宿主清洗后渲染（05 §9）
  contentText?: string;      // 纯文本正文
  streams?: Stream[];        // 文内附带的视频
  interaction?: { liked?: boolean; favorited?: boolean };
}
```

## 10. `comments`

```ts
comments: {
  list(params: PageParams & { target: CommentTarget }, ctx): Page<Comment>
  send?(params: { target: CommentTarget; text: string }, ctx): Comment | void
  like?(params: { commentId: string; on: boolean }, ctx): void
}

type CommentTarget =
  | { kind: "media"; mediaId: string; episodeId?: string }
  | { kind: "article"; articleId: string }
  | { kind: "replies"; commentId: string };      // 某条评论的回复

interface Comment {
  id: string;
  author: { id?: string; name: string; avatar?: ImageRef };
  text: string;
  createdAt?: number;
  likes?: number;
  liked?: boolean;
  replyCount?: number;
  replies?: Comment[];       // 内联的前几条回复
  pinned?: boolean;
  region?: string;           // 上游提供的属地文本
}
```

- Trả lời một bình luận: `target` của `send` dùng `{ kind: "replies", commentId }`.
- Khi `send` trả về bình luận mới, host chèn trực tiếp vào danh sách; không trả về thì host làm mới danh sách.
- Khi plugin không hỗ trợ một loại `target` nào đó thì ném `UNSUPPORTED`, host ẩn lối vào tương ứng.

## 11. `interact`

```ts
interact: {
  like?(params: { target: InteractTarget; on: boolean }, ctx): void
  favorite?(params: { target: InteractTarget; on: boolean; folderId?: string }, ctx): void
  follow?(params: { authorId: string; on: boolean }, ctx): void
}

type InteractTarget =
  | { kind: "media"; mediaId: string; episodeId?: string }
  | { kind: "article"; articleId: string };
```

Đây là các thao tác **phía tài khoản**, sẽ thay đổi dữ liệu của thượng nguồn, cần đăng nhập; khi chưa đăng nhập plugin ném `AUTH_REQUIRED`.
Yêu thích và lịch sử cục bộ của host không phụ thuộc vào chúng (nguyên tắc thiết kế "ưu tiên cục bộ").

## 12. `auth`

```ts
auth: {
  getMethods(params: {}, ctx): { methods: AuthMethod[]; registerUrl?: string }
  submit(params: { methodId: string; values: Record<string, string> }, ctx): AuthResult
  getProfile(params: {}, ctx): UserProfile | null
  logout(params: {}, ctx): void
}

interface AuthMethod {
  id: string;
  label: string;             // 如"账号密码登录""网页登录"
  form?: Form;               // 省略表示无需填表（如网页登录），宿主直接调用 submit
}

interface Form {
  title?: string;
  description?: string;
  fields: FormField[];
  submitLabel?: string;
}

interface FormField {
  key: string;
  type: "text" | "password" | "email" | "number" | "otp" | "select" | "hidden" | "captcha";
  label?: string;
  placeholder?: string;
  required?: boolean;
  value?: string;                                // 预填值
  options?: { value: string; label: string }[];  // select
  image?: ImageRef;                              // captcha 的题面图片
}

type AuthResult =
  | { status: "ok"; user: UserProfile }
  | { status: "next"; form: Form };              // 还需要一步（验证码、短信码等）

interface UserProfile {
  id: string;
  name: string;
  avatar?: ImageRef;
  description?: string;
  stats?: Record<string, number>;   // 展示用计数，键为展示标签，如 { "关注": 3 }
}
```

Quy trình:

```text
宿主调用 getMethods → 用户在插件页的登录表单里选一种方式并填写 → submit
   ├─ { status: "ok" }   → 登录完成，宿主记录登录态并展示 user
   ├─ { status: "next" } → 宿主渲染新表单，再次 submit（此前各步的值与本步合并后一起提交）
   └─ 抛 AUTH_INVALID    → 宿主在表单内展示错误消息
```

- Token do plugin tự lấy có thể lưu vào `hg.storage`, cũng có thể dựa vào kho Cookie thông thường. Host không lưu mật khẩu người dùng nhập vào.
- Nếu hỗ trợ đặc tính `webview`, có thể cung cấp cách đăng nhập không có `form`. Sau khi người dùng bấm, plugin trong `auth.submit` gọi `hg.webview.open`. Host quản lý phiên đăng nhập cùng origin của plugin đó, plugin đọc `UserProfile` từ giao diện tài khoản. [Quy trình đăng nhập web](04-host-api.md#12-hgwebview-tùy-chọn)
- `getProfile`: dùng phiên đã lưu lấy lại thông tin; chưa đăng nhập trả về `null`; phiên mất hiệu lực ném `AUTH_EXPIRED`.
  Host gọi nó ở trang plugin và "Của tôi" để hiển thị trạng thái đăng nhập.
- `logout`: thông báo thượng nguồn và dọn dẹp phiên cục bộ; dù yêu cầu thượng nguồn thất bại cũng **phải** hoàn thành việc dọn dẹp cục bộ.
- `registerUrl`: host cung cấp lối vào "đăng ký" bên cạnh biểu mẫu đăng nhập, dùng trình duyệt bên ngoài mở.
- Khi bất kỳ lần gọi nào ném `AUTH_EXPIRED`, host xóa bộ nhớ đệm trạng thái đăng nhập của giao diện và hướng dẫn đăng nhập lại;
  token mất hiệu lực do plugin dọn dẹp, không cho phép host đoán khóa lưu trữ hoặc trực tiếp xóa toàn bộ dữ liệu của plugin đó.

## 13. Dữ liệu phía tài khoản

### 13.1 `library`

```ts
library: {
  listMedia(params: PageParams & {
    kind: "favorites" | "likes" | "history";
    folderId?: string;
  }, ctx): Page<MediaItem>
  listFolders?(params: {}, ctx): { id: string; label: string; count?: number }[]
  listFollowing?(params: PageParams, ctx): Page<Author>
}
```

Yêu thích, lượt thích, lịch sử trong tài khoản thượng nguồn. Host trong "Của tôi" hiển thị nó tách biệt với thư viện cục bộ, và cung cấp "nhập vào cục bộ".

### 13.2 `progress`

```ts
progress: {
  report(params: {
    mediaId: string;
    episodeId: string;
    positionSec: number;
    durationSec?: number;
    event: "start" | "heartbeat" | "pause" | "end";
  }, ctx): void
}
```

Báo cáo tiến độ phát lên thượng nguồn. Host gọi với tần suất giới hạn (xem [03 §3](03-host-rules.md)); thất bại im lặng, không ảnh hưởng việc phát.

### 13.3 `updates`

```ts
updates: {
  check(params: {
    items: { mediaId: string; episodeCount?: number; updatedAt?: number }[];   // ≤ 50
  }, ctx): { mediaId: string; episodeCount?: number; updatedAt?: number; latestLabel?: string }[]
}
```

Host giao theo lô các mục "đang theo dõi cập nhật" trong thư viện cùng trạng thái đã biết cho plugin, plugin trả về các mục có thay đổi.
Chỉ trả về các mục có cập nhật là được; host dựa vào đó hiển thị huy hiệu. Khi chưa triển khai khả năng này, host nhân lúc người dùng mở trang chi tiết mà so sánh.

## 14. `endpoints`

```ts
endpoints: {
  list(params: { refresh?: boolean }, ctx): Endpoint[]
}

interface Endpoint {
  id: string;
  label?: string;
  url: string;          // 纯 origin，如 "https://www.example.com"
}
```

Dùng cho tình huống thượng nguồn có nhiều tên miền lối vào (trang chính, trang dự phòng, bản sao).

- **Mục đầu tiên** của danh sách là tuyến mặc định, id và origin trong danh sách lần lượt duy nhất. Tuyến cố định trả về trực tiếp; danh sách động có thể đọc bộ nhớ đệm bền vững,
  lần khởi tạo đầu tiên không có bộ nhớ đệm cho phép dùng địa chỉ dẫn đường trong plugin để nối mạng. Khi `refresh: true` thì bỏ qua bộ nhớ đệm danh sách cũ để cập nhật.
- Host gọi khi bật lần đầu, khởi tạo lại sau khi thay đổi cài đặt, người dùng mở bảng tuyến; `refresh: true` chỉ truyền khi người dùng bấm làm mới.
  `hg.endpoint.current` có thể là `null`, phương thức này phải có thể trả về danh sách không rỗng khi không có tuyến hiện tại, không phụ thuộc vào việc factory nối mạng.
- Khi không có lựa chọn cũ hoặc lựa chọn cũ đã bị loại bỏ, khởi tạo chọn mục đầu tiên. Khi người dùng chủ động làm mới loại bỏ tuyến hiện tại, trước hết dừng các phiên liên quan,
  chọn tuyến mặc định mới và xây dựng lại; việc mở bảng thông thường có thể dùng danh sách đã đệm.
- Host chịu trách nhiệm: hiển thị danh sách, thăm dò khả năng tới được của từng tuyến (phát một yêu cầu tới đường dẫn gốc của tuyến, chỉ báo kết quả, không tự động chuyển),
  lưu lựa chọn của người dùng, khởi động lại plugin sau khi chuyển.
- Plugin đọc tuyến hiện tại qua `hg.endpoint.current` (xem [04 §4](04-host-api.md)).

## 15. `ui`

Plugin không thể vẽ giao diện. Khi cần người dùng nhập liệu hoặc thao tác (đổi tài khoản, điền tham số tùy chỉnh, chọn tùy chọn),
plugin khai báo **biểu mẫu**, do host kết xuất trong trang plugin (xem [08 §2.4](08-frontend.md)).

```ts
ui: {
  getActions(params: {}, ctx): ActionDescriptor[]
  submit(params: { actionId: string; values: Record<string, string> }, ctx): ActionResult
}

interface ActionDescriptor {
  id: string;
  label: string;             // 按钮文字，如"切换账号""自定义线路"
  description?: string;      // 按钮下方的说明
  form: Form;                // 与 auth 的 Form 同构（§12）
}

type ActionResult =
  | { status: "ok"; message?: string }     // 完成后可在界面上提示一句话
  | { status: "next"; form: Form };        // 还需要一步，宿主渲染新表单后再次 submit
```

- Host trong trang plugin kết xuất mỗi hành động thành một nút bấm; sau khi bấm triển khai biểu mẫu, gửi xong gọi `submit`.
- Hành động là **do người dùng kích hoạt**: host chỉ gọi `getActions` và `submit` vào những thời điểm này.
- Hành động cần đăng nhập mới thực thi được, plugin ném `AUTH_REQUIRED`; host gợi ý đăng nhập trước.
- **Không cung cấp** khả năng bật cửa sổ hỏi giữa chừng trong lúc gọi: trong thời gian thực thi phương thức, plugin không thể yêu cầu người dùng nhập liệu.
  Lối vào người dùng và điều kiện hoàn tất của đăng nhập web xem [04 §12](04-host-api.md). Việc khai báo độc lập và lối vào hoàn thành thủ công cho xác minh người-máy của trang vẫn chờ định nghĩa.

## 16. Hook tầng cao nhất

```ts
transformResource?(params: {
  kind: "image" | "playlist" | "segment" | "key";
  name: string;              // 来自 ResourceTransform.name
  state?: Json;              // 来自 ResourceTransform.state
  url: string;               // 资源的最终地址
  contentType?: string;
  data: Uint8Array;
}, ctx): { data: Uint8Array; contentType?: string }

dispose?(): void
```

- **`transformResource`**: Xử lý phép biến đổi byte của `{ type: "plugin" }` (xem [05 §7.2](05-content-model.md)).
  Nó được gọi với tần suất cao khi tải hình ảnh và khi phát, **nên** giữ tính toán thuần, không nối mạng.
- Việc sửa cài đặt được host thống nhất hủy các lần gọi đang bay và khởi tạo lại, không cung cấp callback thay đổi cài đặt.
- **`dispose`**: Thông báo đồng bộ cố gắng hết sức trước khi gỡ bỏ (ngân sách 1 giây), không đảm bảo được gọi; không được dựa vào nó để lưu trạng thái. Runtime đã bị ngắt hoặc không thể tiếp tục thực thi thì bỏ qua thông báo. Khi thông báo, đã không còn phạm vi lần gọi nghiệp vụ, không thể phát khởi thao tác mạng, lưu trữ, Cookie, HTML hay bộ hẹn giờ; có thể giải phóng tham chiếu bộ nhớ của plugin và ghi nhật ký. Host không chờ Promise nó trả về, việc ném lỗi hoặc timeout không ngăn việc gỡ bỏ.

## 17. Tìm kiếm tổng hợp và đổi nguồn

Đây là chức năng của host, xây dựng trên khả năng `search` và `media` của từng plugin, không cần plugin triển khai thêm bất cứ thứ gì.
Chi tiết giao diện xem [08](08-frontend.md).

### 17.1 Tìm kiếm tổng hợp

Sau khi người dùng gửi tìm kiếm, host gọi song song `search` của các plugin người dùng đã chọn (chọn một, chọn nhiều hoặc toàn bộ).
Thời điểm kích hoạt và giới hạn trên đồng thời xem [03 §3](03-host-rules.md).

Ở chế độ tổng hợp, kết quả trước hết chia cột theo nguồn, còn có một cột "toàn bộ" hợp nhất cùng một tác phẩm thành một thẻ:

1. Chuẩn hóa tiêu đề: loại bỏ khoảng trắng và dấu câu, thống nhất toàn chiều rộng/nửa chiều rộng, bỏ qua chữ hoa thường.
2. Các mục có tiêu đề sau chuẩn hóa giống nhau, và `year` cùng `type` không xung đột thì hợp nhất.
3. Thẻ đã hợp nhất ghi nhãn số nguồn, bấm vào có thể chọn phát từ nguồn nào.

### 17.2 Đổi nguồn

Khi nguồn hiện tại không phát được (`CONTENT_LOCKED`, `NOT_FOUND`, `UPSTREAM_CHANGED`, lấy luồng thất bại) hoặc người dùng chủ động chọn:

1. Host dùng `title` của mục hiện tại (và `year`, `type`) tìm kiếm trong các plugin khác.
2. Sắp xếp theo độ tương tự tiêu đề, tiêu đề hoàn toàn nhất quán và năm, loại không xung đột xếp phía trước.
3. **Người dùng xác nhận** mục ứng viên; host không tự động thay thế.
4. Sau khi xác nhận, host thiết lập liên kết nguồn mới và giữ lại nguồn cũ, yêu thích cũ và tiến độ cũ; mỗi nguồn vẫn ghi riêng theo "ID plugin + ID mục".
5. Số tập ưu tiên căn theo `season + episodeNumber` rõ ràng; cả hai phía đều không có số mùa thì có thể theo số tập rõ ràng đề xuất ứng viên.
   `index`, độ tương tự tiêu đề chỉ làm gợi ý dự phòng, khi khớp bị thiếu hoặc có mơ hồ thì để người dùng chọn tập.
6. Nguồn mới đã có tiến độ cục bộ thì ưu tiên dùng bản ghi của nguồn đó; chưa có thì có thể đề xuất dùng lại vị trí phát cũ,
   nhưng không thể mặc định coi số giây của bản tách tập, bản cắt và bản đầy đủ là tương đương, sau khi người dùng xác nhận mới định vị.

Để việc đổi nguồn đáng tin, plugin **nên**:

- Trả về tiêu đề gốc của thượng nguồn, đừng đính kèm trong `title` các tu từ như "cập nhật tới tập x", "HD" (đặt vào `badge` hoặc `description`).
- Cố gắng cung cấp `year` và `type`.
- Dùng `Episode.index` biểu đạt thứ tự danh sách, cố gắng cung cấp `season`, `episodeNumber` thật, đừng đoán tạo định danh mùa/tập từ văn bản hiển thị.
