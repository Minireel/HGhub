# Làm plugin đầu tiên

Cung cấp một ES module UTF-8, chỉ xuất `meta` và factory mặc định. Tệp không vượt quá 2 MiB, không dùng import bên ngoài. Tải [ví dụ hư cấu đầy đủ](../../../examples/plugin-hello.js), trong phần cài đặt plugin điền địa chỉ MP4 HTTPS do chính bạn kiểm soát rồi kiểm chứng việc phát. Ví dụ không phụ thuộc vào video kiểm thử bên ngoài.

```js
export const meta = { id: 'example.hello', name: '虚构示例', version: '1.0.0', protocol: 1 };
export default hg => ({
  media: {
    getDetail({ id }) {
      return { id, title: '示例短片', playlists: [{ id: 'main', episodes: [{ id: 'first', index: 1, label: '第一集' }] }] };
    },
    resolvePlayback() {
      throw new hg.PluginError('UNSUPPORTED', '请先配置自己的媒体地址');
    },
  },
  search: { search({ query }) { return { items: [{ id: 'demo', title: query || '示例短片' }], next: null }; } },
});
```

Đoạn mã trên kiểm chứng cài đặt, tìm kiếm và chi tiết. Ví dụ đầy đủ còn cung cấp phương thức phát có thể cấu hình và trang chủ.

## Kết nối một trang

Trước hết hãy quan sát yêu cầu và phản hồi khi truy cập trang bình thường, ghi lại danh sách, chi tiết, phân tập và phân giải phát một lần. Đừng đoán giao diện hoặc coi địa chỉ phát sắp hết hạn là ID của tác phẩm.

```js
const response = await hg.http.fetch({
  url: 'https://api.example.com/search', query: { q: query },
  responseType: 'json', signal: ctx.signal,
});
if (!response.ok) throw new hg.PluginError('UPSTREAM_ERROR', '来源暂时不可用');
if (!Array.isArray(response.data.items)) throw new hg.PluginError('UPSTREAM_CHANGED', '来源结构已变化');
return { items: response.data.items.map(row => ({ id: String(row.id), title: row.title })), next: response.data.next || null };
```

Tên miền giữ chỗ phải được thay bằng địa chỉ đã kiểm chứng. Giữ nguyên ID thật của tác phẩm, danh sách phát và phân tập; index chỉ biểu thị thứ tự. Con trỏ phân trang phải vẫn dùng được sau khi plugin được tải lại. Đặt media và các header yêu cầu cần thiết vào đối tượng tài nguyên, do host xử lý.

## Làm mới địa chỉ phát

Khi địa chỉ thượng nguồn sẽ hết hạn, trả về `expiresAt` chính xác theo mili giây. Khi nhận được `reason: "refresh"`, lấy lại địa chỉ còn hiệu lực, không trả về bộ nhớ đệm đã biết là hết hạn. Host sẽ hợp nhất các lần làm mới đồng thời của cùng một phiên phát, thực thi lại chuỗi phân giải hiện tại từ nguồn dữ liệu; sau khi thành công sẽ chuyển toàn bộ phiên bản tài nguyên. Phát bình thường, tải trước và làm mới đều phải chuyển tiếp `ctx.signal`, sau khi hủy thì không tiếp tục truy cập thượng nguồn. Phụ đề, header yêu cầu và thời hạn hiệu lực được cập nhật cùng với `PlaybackInfo` mới.

## Phát bên ngoài

Khi không thể phát trong ứng dụng, trả về `{ kind: "external", url, external: { app: "播放器名称" } }`. Cho phép HTTP(S), magnet, ed2k, rtsp, rtmp; `app` và `mime` chỉ là gợi ý, không chỉ định chương trình thực thi. Host hiển thị địa chỉ đích, sau khi người dùng bấm "xác nhận mở" sẽ giao cho chương trình xử lý mặc định của hệ thống, thiếu chương trình thì hiển thị thất bại. Mở bên ngoài không đính kèm header yêu cầu tài nguyên, Cookie hay phép biến đổi; những liên kết không thể truy cập nếu thiếu các điều kiện này không phù hợp với đường đi này. Tải trước sẽ không mở ứng dụng bên ngoài.

## Phụ đề

Trả về địa chỉ phụ đề và `format` thực tế (`srt` hoặc `vtt`) trong `PlaybackInfo.subtitles`, có thể kèm `language`, `label`, `default` và `headers`. SRT dù được trả về dưới dạng `application/octet-stream`, host cũng chuyển đổi theo khai báo thành VTT UTF-8; giữ thông tin rãnh để trình phát lựa chọn. Mã hóa ưu tiên đọc BOM, tiếp theo là `charset` của phản hồi, không có cả hai thì theo UTF-8. Phụ đề mã hóa cũ cần cung cấp charset đúng; không đoán mã hóa không có dấu hiệu. Đọc hoặc chuyển đổi phụ đề thất bại chỉ ảnh hưởng rãnh đó, ASS hiện không được kết xuất.

## Nhập vào

Mở "Trung tâm plugin → Nhập & đăng ký", chọn tệp hoặc dán mã nguồn. Xác nhận trạng thái "đã sẵn sàng", từ tìm kiếm vào chi tiết và chọn tập. Sửa mã có thể nhập lại với cùng ID, cùng phiên bản; bản sửa mới sẽ dừng phiên media cũ. Theo [các bước kiểm chứng](testing.md) kiểm tra đường đi bình thường, lỗi, hủy bỏ, trang thứ hai và tải lại.
