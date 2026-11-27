<p align="center">
  <img src="app-icon.svg" width="88" height="88" alt="Biểu tượng HGhub" />
</p>

<h1 align="center">HGhub</h1>

<p align="center">Một trình phát video đa năng kết nối nội dung thông qua plugin.</p>

<p align="center">
  <a href="#bắt-đầu-sử-dụng">Bắt đầu sử dụng</a> ·
  <a href="https://minireel.github.io/HGhub/">Tài liệu trực tuyến</a> ·
  <a href="docs/vi/developers/index.md">Phát triển plugin</a> ·
  <a href="docs/vi/DEVELOPMENT.md">Tham gia phát triển</a> ·
  <a href="https://github.com/Minireel/HGhub/issues">Báo lỗi</a>
</p>

HGhub đặt việc duyệt, tìm kiếm, phát và thư viện cá nhân vào cùng một ứng dụng. Thông qua việc nhập plugin JavaScript, bạn có thể kết nối các nguồn nội dung khác nhau, và dùng một giao diện thống nhất để xem và quản lý nội dung.

**Ứng dụng không tích hợp sẵn nguồn nội dung, không phân phối plugin của bên thứ ba, cũng không cung cấp danh sách nguồn chính thức.** Lần đầu sử dụng cần tự nhập plugin, không cần đăng ký tài khoản HGhub; khi nguồn yêu cầu đăng nhập, plugin tương ứng sẽ cung cấp lối vào đăng nhập.

## Trải nghiệm cốt lõi

- **Phát nhập vai**: Giao diện cho máy tính và di động, kết hợp cử chỉ và điều khiển phát, xem chi tiết, chọn tập và bình luận trong bảng điều khiển.
- **Duyệt và tìm kiếm đa nguồn**: Đề xuất, lọc theo danh mục và tìm kiếm tổng hợp, hỗ trợ xem kết quả theo từng nguồn, và giữ lịch sử tìm kiếm cục bộ.
- **Lựa chọn phát linh hoạt**: Hỗ trợ HLS và video thông thường, chuyển tuyến, độ nét và resolver, cũng có thể tìm ở nguồn khác để tiếp tục xem.
- **Thư viện của riêng mình**: Quản lý mục yêu thích cục bộ, lịch sử xem và tiến độ phát tiếp; hỗ trợ theo dõi cập nhật, và có thể nhập danh sách đám mây do plugin cung cấp vào mục yêu thích cục bộ.
- **Mở rộng theo nhu cầu**: Thông qua tệp, địa chỉ web hoặc danh sách plugin để nhập nguồn nội dung và resolver, kiểm tra cập nhật, và khi cập nhật thất bại sẽ khôi phục phiên bản khả dụng trước đó.
- **Nhiều hình thức nội dung hơn**: Ngoài VOD, còn có thể thông qua plugin kết nối phát trực tiếp, luồng thông tin và nội dung đồ văn.

Nội dung cụ thể, tài khoản, bình luận và khả năng tương tác phụ thuộc vào plugin đã cài đặt.

## Bắt đầu sử dụng

Bản plugin đang ở giai đoạn phát triển sớm, hiện có thể [chạy hoặc build từ mã nguồn](docs/vi/DEVELOPMENT.md). Trọng tâm hiện tại là hoàn thiện Windows và Android, các nền tảng khác vẫn đang thích ứng và kiểm chứng; xem chi tiết [trạng thái hỗ trợ](docs/vi/developers/status.md).

1. Mở **Trung tâm plugin → Nhập & đăng ký**. Phía di động vào Trung tâm plugin từ "Của tôi".
2. Chọn tệp plugin, dán mã, hoặc nhập địa chỉ plugin/danh sách; xác nhận cài đặt và bật lên.
3. Tìm nội dung từ trang chủ, tìm kiếm hoặc danh mục, bắt đầu phát. Khi cần đổi nguồn trang chủ, có thể chọn trong phần cài đặt.

HGhub duy trì trình phát và giao diện plugin; tính khả dụng của nội dung bên thứ ba do nguồn tương ứng quyết định.

## Tài liệu

**[Mở trang tài liệu trực tuyến →](https://minireel.github.io/HGhub/)**

| Tôi muốn… | Bắt đầu từ đây |
| --- | --- |
| Làm một plugin nguồn nội dung hoặc resolver | [Hướng dẫn phát triển plugin](docs/vi/developers/index.md) |
| Hiểu tệp plugin, API của host và mô hình nội dung | [Giao thức plugin HPP](docs/vi/plugin-protocol/README.md) |
| Để Agent hỗ trợ phát triển plugin | [Hướng dẫn phát triển bằng Agent](docs/vi/developers/agent.md) |
| Sửa client hoặc tự build | [Hướng dẫn phát triển client](docs/vi/DEVELOPMENT.md) |
| Xem các khả năng đã hỗ trợ và đang chờ hoàn thiện | [Trạng thái hỗ trợ](docs/vi/developers/status.md) |

Client được build bằng **Rust · Tauri 2 · React · TypeScript**. Giao thức plugin HPP vẫn đang tiến hóa, trước khi viết plugin hãy đọc trạng thái hỗ trợ hiện tại.

## Tham gia, trao đổi, đóng góp

Hoan nghênh bạn phản hồi vấn đề, đề xuất ý tưởng qua [Issues](https://github.com/Minireel/HGhub/issues), hoặc gửi [Pull Request](https://github.com/Minireel/HGhub/pulls) để tham gia cải thiện. Khi báo cáo vấn đề, hãy kèm phiên bản ứng dụng, hệ điều hành và các bước tái hiện.

Dự án do tôi độc lập duy trì; người duy trì sẽ xử lý phản hồi tùy theo thời gian và sức lực, không cam kết nội dung của plugin hay trang bên thứ ba luôn khả dụng. Phạm vi duy trì và cách phản hồi về quyền lợi xem tại [Tuyên bố dự án](docs/PROJECT_POLICY.md).

HGhub hoan nghênh đóng góp của bạn — bất kỳ giao thức nào cũng cần rất nhiều lần vấp ngã, thử sai và tích lũy kinh nghiệm lâu dài mới ngày càng hoàn thiện. Mỗi Issue và PR của bạn đều giúp ích rất nhiều cho HGhub!

Trao đổi cộng đồng: [Linux.DO](https://linux.do/).

## Giấy phép và lập trường phi thương mại

Mã nguồn của dự án được công khai, áp dụng [PolyForm Noncommercial 1.0.0](LICENSE). **Cấm dùng cho mục đích thương mại**, bao gồm bán, kiếm tiền từ quảng cáo, mã kích hoạt, VIP, mở khóa trả phí, và sử dụng như một phần của sản phẩm hoặc dịch vụ thu phí; chi tiết theo toàn văn giấy phép.

Người duy trì **không cấp phép thương mại, không nhận quảng cáo, quảng bá hay hợp tác kinh doanh**, và chưa từng ủy quyền cho bất kỳ ai thu phí dưới danh nghĩa dự án này. Đóng góp kỹ thuật và phản hồi vấn đề của cộng đồng không thuộc hợp tác kinh doanh.

Dự án không vận hành dịch vụ lưu trữ nội dung, không có quan hệ phụ thuộc hay bảo chứng chính thức nào với các nhà cung cấp nội dung bên ngoài. Biên giới trách nhiệm đầy đủ, chính sách duy trì và thuyết minh về phản hồi quyền lợi xem tại [Tuyên bố dự án](docs/PROJECT_POLICY.md).
