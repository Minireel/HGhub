# Nhập, danh sách và cập nhật

Hỗ trợ mã nguồn, JS cục bộ, danh sách JSON cục bộ và URL. Sau khi tải xuống sẽ nhận diện nội dung, không phụ thuộc vào đuôi tệp. Việc phân phối bản sản xuất dùng HTTPS.

```json
{
  "name": "我的插件集",
  "plugins": [
    { "id": "example.hello", "name": "虚构示例", "version": "1.0.0", "url": "plugin-hello.js" }
  ]
}
```

Tên chuẩn là manifest.json. Tên tệp khác không thay đổi việc nhận diện nội dung, khuyến nghị thống nhất dùng tên chuẩn. Danh sách tối đa 1 MiB, 500 mục; JS tối đa 2 MiB.

Mục cục bộ tương đối với thư mục chứa danh sách; mục từ xa lấy URL cuối cùng của danh sách sau khi chuyển hướng làm chuẩn. Xem trước chỉ hiển thị siêu thông tin, sau khi xác nhận chỉ tải xuống các mục đã chọn. Khi meta.id/version của tệp không khớp với danh sách thì chỉ mục đó thất bại.

Khi nhập trên Android một danh sách cục bộ chứa mục tương đối, trong "Nhập & đăng ký" bấm "Chọn thư mục chứa danh sách", dùng bộ chọn hệ thống để ủy quyền thư mục, rồi bấm vào danh sách JSON trong đó. Chỉ chọn một tệp danh sách đơn lẻ sẽ không có quyền truy cập các tệp anh em; lúc này vẫn có thể dùng mục HTTPS tuyệt đối hoặc chọn JS riêng lẻ.

Các mục trong thư mục được tìm từng tầng theo tên tệp, hỗ trợ thư mục con, tiếng Trung và mã hóa phần trăm của URL. Dùng `/` để phân tách thư mục; `..` không được vượt ra ngoài thư mục gốc đã ủy quyền; đường dẫn tệp tuyệt đối, `/` hoặc `\` đã mã hóa, phần truy vấn, fragment và ký tự điều khiển đều bị từ chối. Khi tệp cùng tên trong thư mục không duy nhất, tệp bị thiếu, hoặc loại tệp/thư mục không khớp, thì chỉ mục tương ứng thất bại. Xem trước truy vấn đường dẫn nhưng không đọc nội dung JS; sau khi xác nhận chỉ đọc các mục đã chọn.

Bộ chọn hiển thị các tệp JSON mà thư mục đã chọn chứa trực tiếp, tối đa 500; một lần liệt kê tối đa 5000 mục thư mục. Hãy đặt danh sách và plugin trong thư mục chuyên dụng. Ánh xạ ủy quyền chỉ giới hạn trong phiên ứng dụng hiện tại, sau khi khởi động lại phải chọn lại thư mục; việc cập nhật của bản nhập cục bộ vẫn cần `meta.updateUrl`. Hồi quy phân phối của Rust và kiểm thử đơn vị đường dẫn Android bao quát hủy bỏ, chọn cài và quy tắc vượt biên; trải nghiệm thiết bị của bộ chọn hệ thống và nhà cung cấp tệp vẫn cần nghiệm thu.

## Cập nhật

Phiên bản được so sánh theo từng đoạn số. Cùng ID mà phiên bản cao hơn thì cập nhật; ghi đè cùng phiên bản dùng cho gỡ lỗi; hạ cấp cần xác nhận. Cập nhật giữ lại cài đặt đang hiệu lực và dữ liệu bền vững; bản sửa đổi mới làm phiên cũ mất hiệu lực. Cài đặt thành công và khởi tạo thành công là hai trạng thái khác nhau; khi thất bại có thể khôi phục tệp khả dụng trước đó.

Địa chỉ cập nhật: meta.updateUrl → JS nguồn hoặc danh sách nguồn → không có. Đăng ký có thể làm mới danh sách thủ công. Kiểm tra khi khởi động với cùng một địa chỉ cách nhau ít nhất 24 giờ; kiểm tra thủ công không bị giới hạn. Tự động cài đặt mặc định tắt; trong lúc phát sẽ hoãn cài đặt. Xem chi tiết [giao thức tệp](../plugin-protocol/01-plugin-file.md).
