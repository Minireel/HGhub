# Build cục bộ và triển khai Pages

```powershell
pnpm docs:build
pnpm docs:test
pnpm docs:preview
```

Địa chỉ mặc định http://127.0.0.1:4174/HGhub/, sản phẩm ở dist-docs/, đường dẫn cơ sở /HGhub/. Với miền tùy chỉnh có thể đặt HGHUB_DOCS_BASE=/ rồi build.

`.github/workflows/docs.yml` chỉ kích hoạt thủ công: cài đặt các phụ thuộc đã khóa, build, kiểm tra liên kết và nội dung gói, tải lên sản phẩm tĩnh. Không có trigger push.

Địa chỉ tài liệu trực tuyến là https://minireel.github.io/HGhub/. Cài đặt Pages của kho dùng GitHub Actions; sau khi push mã nguồn, chạy thủ công Developer documentation; workflow này chỉ phát hành tài liệu tĩnh, không build ứng dụng hay tạo Release.

Nếu lần push mã nguồn này không cần chạy CI của ứng dụng, có thể dùng `[skip ci]` trong thông điệp commit, sau đó chạy riêng workflow tài liệu theo cách thủ công. Đừng tạo thẻ phiên bản cho việc phát hành tài liệu.

Trang chỉ sao chép các tài liệu và ví dụ công khai được liệt kê rõ ràng, không đóng gói local-plugins, test-results, handoff và project Android được sinh ra. Gói kỹ năng được sinh từ cùng các tệp nguồn giao thức, tránh việc bản sao thủ công bị lệch.
