# Hướng dẫn phát triển client

[Quay lại trang chủ dự án](../../README.vi.md)

Tài liệu này dành cho các cộng tác viên tham gia phát triển client. Tác giả plugin hãy đọc [Hướng dẫn phát triển plugin](developers/index.md) trước.

## Công nghệ và thư mục

| Lớp | Công nghệ |
| --- | --- |
| Khung ứng dụng | Rust, Tauri 2 |
| Giao diện | React, TypeScript, Vite, Tailwind CSS |
| Thành phần phát | hls.js |
| Mạng | reqwest, rustls |

```text
src/                    界面、交互、状态管理与组件
src-tauri/              原生能力与 Rust 实现
scripts/                开发、构建与验证脚本
tests/                  本地测试页面与回归用例
docs/                   技术说明与维护记录
```

## Phát triển cục bộ

Môi trường cơ bản: Node.js 22, pnpm 10, Rust 1.90 trở lên, cùng môi trường phát triển Tauri cần cho nền tảng đích.

```bash
pnpm install --frozen-lockfile
pnpm dev             # 前端开发
pnpm app:dev         # 桌面应用开发
pnpm build           # 类型检查与前端构建
pnpm app:build       # 桌面应用构建与打包
```

Cấu hình cho từng nền tảng nằm ở `src-tauri/tauri*.conf.json`. Phát triển di động và macOS cần SDK và toolchain của nền tảng tương ứng.

Build Android còn cần JDK 17, Android SDK / NDK và libclang. Đặt `JAVA_HOME`, `ANDROID_HOME`, `NDK_HOME`, và cho `LIBCLANG_PATH` trỏ tới thư mục chứa thư viện động libclang; engine plugin sẽ dùng header của NDK để sinh binding cho Android. Lần đầu chạy `pnpm tauri android init --ci`, sau đó có thể dùng `pnpm tauri android build --debug --target x86_64 --apk --ci` để build gói cài cho trình giả lập (thiết bị ARM64 thì đổi target thành `aarch64`).

- Hệ thống tệp không hỗ trợ liên kết tượng trưng như exFAT cần giữ `node-linker=hoisted` trong `.npmrc`; một số bước build native vẫn có thể bị giới hạn bởi hệ thống tệp.
- Danh sách cho phép của các script cài đặt phụ thuộc xem ở `package.json` và `pnpm-workspace.yaml`.
- Lệnh kiểm chứng và phạm vi hỗ trợ hiện tại xem ở [Tiến độ triển khai](plugin-protocol/IMPLEMENTATION.md).

## Trang tài liệu cho nhà phát triển

```bash
pnpm docs:build
pnpm docs:test
pnpm docs:preview
```

Địa chỉ xem trước mặc định là `http://127.0.0.1:4174/HGhub/`. Các trang tĩnh, ví dụ và gói kỹ năng Agent công khai được xuất ra `dist-docs/`.
Cấu hình Pages và các bước triển khai xem ở [Thuyết minh triển khai trang tài liệu](developers/deployment.md).

## Đọc thêm

- [Hướng dẫn phát triển plugin](developers/index.md): Viết plugin nguồn nội dung hoặc resolver.
- [Giao thức plugin](plugin-protocol/README.md): Giao diện và quy ước dữ liệu.
- [Phạm vi hỗ trợ hiện tại](developers/status.md): Trạng thái triển khai và biên giới kiểm chứng theo nền tảng.
