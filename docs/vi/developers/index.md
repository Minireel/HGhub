# Phát triển plugin HGhub

Đưa nội dung vào trình phát, bắt đầu từ một tệp JavaScript.

HGhub quản lý giao diện, phát, mạng, cách ly và thư viện cục bộ; plugin chịu trách nhiệm về dữ liệu của trang. Ứng dụng không tích hợp sẵn nguồn nội dung, cũng không có danh sách nguồn nội dung chính thức.

## Bắt đầu từ đây

1. [Làm plugin đầu tiên](quickstart.md): Nhập, tìm kiếm, chi tiết và phát.
2. [Khả năng và API của host](host-api.md): Chọn khả năng theo trang, xử lý yêu cầu, lưu trữ và hủy bỏ.
3. [Gỡ lỗi và nghiệm thu](testing.md): Từ host giả đến QuickJS thật và trình phát.
4. [Nhập, danh sách và cập nhật](distribution.md): Đơn tệp, danh sách, phiên bản và đăng ký.
5. [Để Agent hỗ trợ phát triển](agent.md): Cài gói kỹ năng công khai và sao chép prompt.

Ứng dụng hiện tại **0.0.5 / HPP 1.0.0-dev**. Giao thức là dự thảo, trước khi phát triển hãy đọc [Trạng thái hỗ trợ hiện tại](status.md).

[Tổng quan giao thức](../plugin-protocol/README.md) · [API đầy đủ của host](../plugin-protocol/04-host-api.md) · [Mô hình nội dung](../plugin-protocol/05-content-model.md) · [Build và triển khai cục bộ](deployment.md)

Trang web không gửi dữ liệu phân tích, không tải phông chữ bên thứ ba. Nội dung tải về chỉ gồm giao thức công khai, kiểu dữ liệu và ví dụ hư cấu, không chứa nguồn nội dung hay tài khoản trên máy này.
