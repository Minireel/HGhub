# Để Agent hỗ trợ phát triển

Tải [gói kỹ năng phát triển plugin HGhub](../../../downloads/hghub-plugin-dev.zip) và [trình cài đặt](../../../downloads/install.py), đặt trong cùng một thư mục. Gói chỉ chứa giao thức công khai, trạng thái hỗ trợ, kiểu dữ liệu và ví dụ hư cấu.

```powershell
python install.py --archive hghub-plugin-dev.zip --destination "$HOME/.codex/skills"
```

Cũng có thể sao chép thư mục hghub-plugin-dev trong ZIP vào thư mục kỹ năng của Agent. Trình cài đặt từ chối ghi đè thư mục cùng tên; trước khi nâng cấp hãy sao lưu thư mục cũ.

## Prompt có thể sao chép

```text
请从本页下载 HGhub 插件开发技能包，解压到我指定的 Agent 技能目录，阅读 SKILL.md 和 references/status.md。
使用 hghub-plugin-dev 为我指定的站点开发单文件 HPP 插件。先核实实际接口与播放链，再实现存在的能力；使用 hg.http、hg.storage 与资源对象，保留真实作品、播放列表和分集 ID。
交付插件 JS、manifest.json 和可重复验证步骤，分别说明模拟、真实宿主、真实上游与设备播放的结果。
如果当前对话没有目标站点或输出目录，请先问我；已有信息直接使用。不发布或上传，除非我明确要求。
```

Khối mã cung cấp nút sao chép. Liên kết tải xuống được phân giải tương đối với trang hiện tại; việc kiểm chứng build cục bộ không có nghĩa là địa chỉ công khai chưa phát hành đã có thể truy cập.
