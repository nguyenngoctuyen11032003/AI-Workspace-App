# KNOWN_ISSUES.md

| # | Vấn đề | Ảnh hưởng | Trạng thái |
|---|---|---|---|
| 1 | Codex CLI chưa cài trên máy dev | Không thể kiểm thử CodexAdapter bằng tài khoản thật | Chờ user cài |
| 2 | Windows: Codex/Claude không có OS-level sandbox (seatbelt/landlock) | Phải dựa vào permission policy + Docker cho build/test | Thiết kế trong docs/SECURITY.md |
