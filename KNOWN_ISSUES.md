# KNOWN_ISSUES.md

Cập nhật: 2026-10-30.

| # | Vấn đề | Ảnh hưởng | Trạng thái |
|---|---|---|---|
| 1 | Codex CLI chưa cài (extension VS Code đã đăng nhập ChatGPT, `codex login status` báo đã đăng nhập) | Chưa test CodexAdapter live | Chờ chủ sở hữu cài |
| 2 | Windows native: Claude Code không có sandbox cấp OS. Quyền CLI chỉ là lớp chính sách | Rủi ro R1 trong SECURITY.md | Thiết kế hai tầng, cần chủ sở hữu xác nhận rủi ro |
| 3 | Docker daemon chưa chạy lúc kiểm tra | Mọi thông số sandbox container là giả định | Chờ bật Docker |
| 4 | Hạn mức tài khoản 5 giờ bị chạm khi chạy nhiều agent, làm 3 agent dừng giữa chừng | Giới hạn thật của "18 agent" là quota | Đã ghi vào D-010, MULTI_AGENT_SCALABILITY |
| 5 | Điều khoản subscription cho nhiều phiên song song chưa xác minh | Rủi ro ToS | Chờ chủ sở hữu kiểm tra |
| 6 | Truy vết AC: 199 FR đã có bảng truy vết, 74 AC mới PROPOSED. Còn 16 FR chờ Owner làm rõ, trong đó 4 Must (FR-122, 181, 182, 183). `TESTING.md` và `IMPLEMENTATION_ROADMAP.md` vẫn nêu "AC-001..130" có khoảng trống đánh số | Không viết được AC cho 16 FR | Chờ Owner |
| 7 | QA P0-2: enum provider_kind thiếu codex/openhands | Đã sửa trong DATABASE_DESIGN | Đã sửa |
| 8 | 55 mâu thuẫn giữa tài liệu (`docs/adr/CONFLICTS_TO_RESOLVE.md`), 15 quyết định Owner, xem PHASE_0_OWNER_DECISION_REGISTER.md | Người triển khai có thể đọc nhầm giá trị | Chờ chủ sở hữu |
| 9 | Nhãn "Đã xác minh" đôi khi dùng cho "khớp với tài liệu khác" thay vì đã chạy thật | Có thể gây tin quá mức | Cần rà trong lần vá tới |
| 10 | Python thật không có trên PATH, heredoc dài lỗi ENAMETOOLONG trên Windows | Hạn chế công cụ | Dùng công cụ Write |
| 11 | herdr: `vendor/` và `assets/sounds` chưa kiểm tra license; chưa thử socket API | Không được sao chép các phần đó | Mở |
| 12 | AI-Workspace-App là git repo (main, remote origin). Owner đã tự commit và đẩy toàn bộ Phase 0 (commit `1aec2b3`). Agent không commit trừ khi Owner yêu cầu; nếu yêu cầu thì tạo branch mới và không ghi Claude vào tác giả hay trailer | Không còn thay đổi chưa commit trước phản hồi của Owner | Đã chốt cách làm |
| 13 | Claude Code mặc định ghi snapshot system prompt một lần mỗi hội thoại và dùng lại khi resume (`--system-prompt-snapshot`, xác minh bằng `claude --help` 2.1.312). Skill overlay đổi sau đó không có hiệu lực khi resume nếu không xử lý | Ảnh hưởng D-021 | Cần PoC, xem DYNAMIC_SKILL_INJECTION.md mục 4.5 |
| 14 | Lộ trình cho agent chạy thật (Phase 2) trước khi có policy, API auth và audit (Phase 4) | Rủi ro bảo mật, từ PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md | Mở, cần Owner xếp lại thứ tự phase |
| 15 | Drizzle chỉ kiểm trên PGlite (engine báo PostgreSQL 18.6), không PG16. Unique theo `sequence` trên bảng phân vùng không ép được ở DB. `drizzle-kit generate/check` không phát hiện thay đổi thủ công, `pull` có | Bằng chứng ADR-0002 chỉ Partial | Lặp PoC trên PG16 khi có Docker; dùng `pull` để kiểm drift |
| 16 | SK-04 chỉ SIMULATED: agent thật bị chặn quyền ghi overlay hai lần. Cùng OS user nên không phân biệt Owner và agent | OWN-AC-2 chưa chứng minh trên agent thật | Cần test với cấu hình quyền phù hợp sau cổng an ninh |
| 17 | Docker daemon vẫn tắt (kiểm tra 13:39); RAM trống 4,04 GB trên 15,77 GB | VAL-DOCKER-01..09 BLOCKED | Chờ Owner tự mở Docker, theo `docs/poc/POC_DOCKER_SANDBOX.md` |
| 18 | D-005 hoãn Codex làm đổi nhóm chấp nhận MVP B của yêu cầu gốc; D-007 Tier 2 dựa OpenHands bị hoãn sau Phase 4 | Xung đột phạm vi | Chờ Owner xác nhận |
| 19 | Danh sách event gốc (Prompt 1 mục 8.4) chưa nằm trong repo | Không đóng băng event schema | PENDING_OWNER_CLARIFICATION |
| 20 | Chỉ thị A.10 (không chạy mã tùy ý trên host, sandbox bắt buộc, fail closed) chặt hơn D-007 Tier 1 đã duyệt. Thân SECURITY mục 2 đến 5 chưa viết lại | Không có thực thi agent thật cho tới khi có sandbox cô lập; Docker là điều kiện cứng nhưng chưa được phép bật | Chờ Owner trả lời X-13, Q-23, Q-26 |
| 21 | Bảng ánh xạ event dotted chưa có. 16 event gốc đã chép; ví dụ `task.started`, `agent.blocked` không thuộc 16 event gốc | Không đóng băng event schema | Chờ Owner (P-16, X-14) |
| 22 | Tài liệu còn mô tả lưu secret bằng OS keychain và kind API trong enum, so với FR-088 và FR-098 | QA ba P1 F3-01 | Banner đã thêm; chờ Q-21 |
| 23 | Provider terms: Owner nói không có căn cứ công khai cho số phiên song song; agent không xác minh độc lập | Không tăng concurrency | Giữ 5, không tuyên bố |
