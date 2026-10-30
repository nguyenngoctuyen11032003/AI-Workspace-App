# AGENTS.md — Hướng dẫn cho mọi AI agent làm việc trên AI WORKSPACE

Dự án được quản lý qua nhiều phiên AI và nhiều agent. **Không dựa vào lịch sử hội thoại.** Mọi thứ cần biết nằm trong repo này.

## 1. Thứ tự đọc bắt buộc (trước khi làm bất cứ việc gì)
1. `PROJECT_STATUS.md` — phase hiện tại, cái gì xong, cái gì bị block.
2. `NEXT_SESSION.md` — chỉ dẫn chính xác cho phiên này.
3. `WORK_PLAN.md` và `TASK_BACKLOG.md` — danh sách task có ID, trạng thái, người phụ trách, dependency.
4. `DECISION_LOG.md` — quyết định đã duyệt và đang chờ duyệt. Chỉ mục "Đã duyệt" mới là ràng buộc.
5. `KNOWN_ISSUES.md` và `AGENT_WORK_LOG.md` — vấn đề đang gặp, việc agent trước làm dở.
6. Tài liệu kiến trúc liên quan đến task của bạn trong `docs/` (xem bảng mục 6).
7. `git status` và `git log -10` (khi repo đã khởi tạo git) để thấy thay đổi chưa commit.

## 2. Quy tắc làm việc
- Tiếp tục từ chỗ đang dở. Không làm lại việc đã hoàn thành nếu không có lý do ghi rõ.
- **Không tự ý đổi kiến trúc đã được phê duyệt.** Đề xuất của agent không phải quyết định của chủ sở hữu. Muốn đổi, viết ADR mới trạng thái "Đề xuất" và đưa vào danh sách cần duyệt.
- Phân biệt trong tài liệu: Đã xác minh, Đề xuất, Giả định, Chưa xác định, Cần người dùng phê duyệt. Không bịa thông tin để lấp chỗ trống.
- Không đánh dấu task `COMPLETED` khi chưa có bằng chứng kiểm thử (log, test, task ID). Không tuyên bố đã chạy song song khi chưa thực sự khởi chạy nhiều tiến trình.
- Mỗi task khai báo file được ghi, file chỉ đọc. **Không hai agent ghi cùng một file.** Dùng branch hoặc worktree riêng khi viết code.
- Sau mỗi thay đổi quan trọng cập nhật: `IMPLEMENTATION_LOG.md`, `PROJECT_STATUS.md`, `WORK_PLAN.md`/`TASK_BACKLOG.md`, và `NEXT_SESSION.md` trước khi kết thúc phiên hoặc khi sắp hết token. Không nói đã lưu tiến độ nếu chưa ghi file.
- Mọi quyết định kiến trúc lớn có ADR trong `docs/adr/`.

## 3. Ràng buộc sản phẩm không thương lượng
- Không dữ liệu mock hay tiến độ giả trong luồng cốt lõi. Mock adapter chỉ dùng trong CI, không thay luồng sản phẩm.
- Không nút không có chức năng. Không hardcode danh sách agent hay workflow. Không gắn chặt vào một provider.
- Chuyển động trong Virtual Office chỉ do sự kiện thật điều khiển.
- Credential do CLI tự quản. **Không bao giờ đọc** `~/.claude/.credentials.json` hay `~/.codex/auth.json`. Không đưa credential vào frontend, log, event, database dạng plaintext, hay chia sẻ giữa agent.
- Không chạy mã do AI sinh trực tiếp trên host ngoài các quy tắc ở `docs/SECURITY.md`. Không tự deploy production. Hành động nguy hiểm đi qua Approval.
- Không sao chép mã, prompt, layout hay asset từ `my-virtual-office` (AGPL-3.0). Mã từ ChatDev, Pixel Agents, OpenHands, herdr chỉ dùng kèm attribution theo `docs/research/LICENSE_AND_REUSE.md`.
- Không chỉnh sửa các repo tham chiếu (`ChatDev`, `OpenHands`, `pixel-agents`, `my-virtual-office`, `herdr`) nằm cạnh thư mục này.

## 4. Môi trường đã kiểm tra (2026-10-30)
Windows 11, Node 24.19, pnpm 10.34, Git 2.56, Docker client 29.8 (daemon chưa chạy lúc kiểm tra), Claude Code CLI 2.1.312 đã đăng nhập. Codex CLI chưa cài. Shell mặc định là Git Bash hoặc PowerShell. Không có Python thật trên PATH. Lệnh ghi file dài bằng heredoc có thể lỗi ENAMETOOLONG: dùng công cụ Write.

## 5. Cách chạy nhiều agent
Giới hạn đồng thời mặc định 5, tối đa 18 (D-010, chưa duyệt). Giới hạn thật là quota của subscription. Chi tiết: `docs/architecture/AGENT_ORGANIZATION.md`, `AGENT_SCHEDULER.md`, `RESOURCE_MANAGEMENT.md`. Mỗi agent có phạm vi và file sở hữu riêng; Orchestrator tích hợp và kiểm tra diff.

## 6. Bản đồ tài liệu
| Cần gì | Đọc |
|---|---|
| Yêu cầu, user flow, AC | `docs/product/` |
| Kiến trúc tổng thể, backend, frontend, API | `docs/SYSTEM_ARCHITECTURE.md`, `BACKEND_ARCHITECTURE.md`, `FRONTEND_ARCHITECTURE.md`, `API_DOCUMENTATION.md` |
| Database | `docs/DATABASE_DESIGN.md` |
| Agent, orchestration, workflow, event | `docs/AGENT_ARCHITECTURE.md`, `MULTI_AGENT_ORCHESTRATION.md`, `WORKFLOW_ENGINE.md`, `EVENT_SYSTEM.md`, `docs/architecture/` |
| Provider (Claude, Codex, OpenHands) | `docs/PROVIDER_INTEGRATION.md`, `docs/research/CLAUDE_CODE_INTEGRATION.md`, `CODEX_INTEGRATION.md` |
| Skill và memory | `docs/architecture/SKILL_SYSTEM.md` |
| Virtual Office | `docs/VIRTUAL_OFFICE.md` |
| Bảo mật | `docs/SECURITY.md`, `docs/security/` |
| UI/UX | `docs/design/` (đọc `HERDR_DESIGN_INTEGRATION.md` cho các sửa đổi mới nhất) |
| Roadmap, rủi ro | `docs/IMPLEMENTATION_ROADMAP.md`, `docs/PROJECT_RISKS.md` |
| Tái sử dụng và license | `docs/research/REUSE_DECISIONS.md`, `LICENSE_AND_REUSE.md` |
| Phát triển, kiểm thử, triển khai | `docs/development/` |
| Quyết định | `DECISION_LOG.md`, `docs/adr/`, `docs/adr/CONFLICTS_TO_RESOLVE.md` |

## 7. Khi hết việc hoặc sắp hết token
Cập nhật `NEXT_SESSION.md` theo đúng 12 mục của nó, ghi `AGENT_WORK_LOG.md`, rồi dừng. Phase 0 kết thúc bằng việc chờ chủ sở hữu xác nhận trước khi bắt đầu Phase 1.

## 8. Cập nhật sau OWR-2 và OWR-3 (2026-10-30)
- **Đọc trước:** `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` là nguồn quyết định của Owner. Mọi thay đổi tài liệu phải dẫn mã Q, D hoặc FR kèm OWR-2.
- **Từ vựng trạng thái:** PENDING_OWNER_APPROVAL, PENDING_OWNER_CLARIFICATION, APPROVED, APPROVED_CONDITIONAL, REJECTED, VERIFIED. VERIFIED chỉ khi có bằng chứng kiểm thử; hiện chưa mục nào VERIFIED. AC mới là PROPOSED, không tự thành APPROVED.
- **Phase 0 vẫn mở. Phase 1 chưa được cho phép.** Không bắt đầu Phase 1, không chuyển Phase 0 thành Completed.
- **Cấm tự ý:** cài Codex CLI; chạy toàn bộ 27 runbook; đổi production schema; sửa WSL hoặc cấu hình Docker; dùng credential thật; tăng concurrency (mặc định 5 là giới hạn ứng dụng, không phải xác nhận điều khoản subscription); commit hoặc push khi chưa được yêu cầu.
- **PoC được phép:** Drizzle, Dynamic Skill/Resume trong thư mục scratch, và Docker sandbox validation sau khi Owner tự mở Docker. Kết quả ở `docs/poc/`.
- **Codex và OpenHands đã hoãn khỏi MVP** (D-005, D-006); giữ abstraction `AgentAdapter`. Claude Agent SDK là adapter chính (D-004).
- **Event naming:** database dùng `lowercase.dot.notation`; `UPPER_SNAKE_CASE` chỉ là ánh xạ ở lớp ngoài, một hàm ánh xạ duy nhất.
- **Khi commit được yêu cầu:** tạo branch mới, không ghi Claude vào tác giả hay trailer, không push nếu chưa được cho phép.
- **OWR-3:** đọc `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md`. Event canonical là dotted lowercase; 16 event gốc ở `docs/EVENT_SYSTEM.md` mục 0, không thêm, xóa, đổi tên. Mặc định concurrency 5 là giới hạn nội bộ, không phải bảo đảm của provider. Không mở Docker, không chạy Conductor PoC, không test agent thật qua SDK nếu Owner chưa cho phép. Không push; không commit khi chưa được cho phép.
