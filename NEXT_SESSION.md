# NEXT_SESSION.md

Cập nhật: 2026-10-30 (Phase 0 Finalization). Đọc `AGENTS.md` trước, rồi file này.

## 1. Tóm tắt dự án
AI WORKSPACE là nền tảng điều phối nhiều AI agent mô phỏng một công ty phần mềm, mỗi agent dùng Claude Code hoặc Codex CLI làm bộ não. Có Virtual Office 2D điều khiển bằng sự kiện thật, workflow node-based, Approval Center, sandbox, Skill Library và (đề xuất D-021) thư mục skill Markdown động theo từng agent. Owner giữ quyền nghiệm thu cuối cùng.

## 2. Phase hiện tại
**Phase 0: các đầu việc kỹ thuật hoàn tất, chờ QA độc lập lần hai và quyết định của Owner.** Phase 1 **không được bắt đầu** khi chưa có xác nhận rõ ràng của Owner. Hoàn thành đầu việc kỹ thuật không có nghĩa là Owner đã phê duyệt.

## 3. Tiến độ tổng thể
Nội dung Phase 0 xong. Chưa đóng vì thiếu quyết định Owner, QA lần hai chưa có kết quả cuối, và 16 FR chưa có AC.

## 4. Việc đã hoàn thành
Xem `PROJECT_STATUS.md` mục "Đã hoàn thành" và `TASK_BACKLOG.md` (FIN-001..FIN-007 DONE).

## 5. Việc chưa hoàn thành
- QA lần hai: `PHASE_0_QA_REVIEW_2.md` (đang chạy lúc ghi).
- 16 FR cần Owner làm rõ trước khi viết AC.
- Đồng bộ các giá trị phụ thuộc quyết định vào tài liệu nguồn (hoãn có chủ đích, xem `docs/DOC_SYNC_LOG_PHASE0.md` bảng "Deferred").
- PoC kiểm chứng skill động và resume (FIN-009), kiểm chứng Codex, Docker sandbox, concurrency (runbook trong `PHASE_0_TECHNICAL_VALIDATION_REPORT.md`), tất cả cần Owner cho phép.

## 6. Task đang làm dở
Không có agent nào đang chạy, trừ QA lần hai nếu chưa kết thúc. Kiểm tra `TASK_BACKLOG.md` mục FIN-008.

## 7. Quyết định quan trọng
`PHASE_0_OWNER_DECISION_REGISTER.md` (D-001..D-021 và Q-01..Q-15, **tất cả PENDING_OWNER_APPROVAL**), `DECISION_LOG.md`, `docs/adr/`. Thứ tự nên quyết trước: D-001, D-002, D-014, D-004 cùng D-010, D-007.

## 8. Vấn đề đang gặp
`KNOWN_ISSUES.md` (14 mục), `PHASE_0_REMAINING_RISKS.md`, `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md`. Chính: Codex CLI chưa cài, Docker daemon chưa chạy, lộ trình phase đặt agent chạy thật trước policy và audit, `--system-prompt-snapshot` ảnh hưởng skill động khi resume, điều khoản subscription cho nhiều phiên.

## 9. File đã thay đổi
Tất cả tài liệu trong `docs/`, các file `PHASE_0_*.md`, `AGENTS.md`, `DECISION_LOG.md`, `TASK_BACKLOG.md`, `PROJECT_STATUS.md`, `KNOWN_ISSUES.md`, `IMPLEMENTATION_LOG.md`, `AGENT_WORK_LOG.md`, `NEXT_SESSION.md`. Chi tiết thay đổi tài liệu nguồn: `docs/DOC_SYNC_LOG_PHASE0.md`. Không có mã ứng dụng. Repo đã là git repo (main, remote origin, 2 commit do Owner), nhiều thay đổi chưa commit; agent không commit khi chưa có chính sách.

## 10. Chỉ dẫn chính xác cho phiên tiếp theo
1. Đọc `PHASE_0_FINAL_REPORT.md`, `PHASE_0_QA_REVIEW_2.md`, `PHASE_0_OWNER_DECISION_REGISTER.md`.
2. **Chờ Owner phản hồi.** Không tự coi đề xuất là đã duyệt, không bắt đầu Phase 1.
3. Khi Owner ghi APPROVED hoặc REJECTED kèm ngày cho từng quyết định: cập nhật `DECISION_LOG.md`, sổ quyết định và ADR liên quan; rồi đồng bộ các mục "Deferred" trong `DOC_SYNC_LOG_PHASE0.md` đúng theo quyết định đó.
4. Nếu Owner cho phép: chạy các runbook kiểm chứng (Codex, Docker, concurrency, PoC skill động) và ghi bằng chứng thật.
5. Hợp nhất AC mới vào `docs/product/ACCEPTANCE_CRITERIA.md` chỉ sau khi Owner duyệt.
6. Chạy lại QA độc lập ngắn trước khi đề xuất đóng Phase 0.

## 11. Agent cần khởi chạy
| Việc | Vai trò | Ghi chú |
|---|---|---|
| Đồng bộ theo quyết định đã duyệt | Documentation + Architect | Mỗi agent một nhóm file, không ghi chung |
| AC cho 16 FR sau khi Owner trả lời | BA | ID AC không trùng dải 201-433 |
| Re-review | QA độc lập, không phải tác giả tài liệu | |
| Phase 1 sau khi được phép | Theo `docs/IMPLEMENTATION_ROADMAP.md` | Xem xét xếp lại phase an toàn (KNOWN_ISSUES #14) |

## 12. Task chạy song song được
Đồng bộ theo miền (DB và Event, Security, Provider và Office, Design) nếu Owner duyệt từng nhóm. Các runbook kiểm chứng độc lập nhau trừ khi dùng chung quota subscription: chạy nối tiếp khi đo concurrency.
Việc phải chờ: mọi thay đổi giá trị chính tắc chờ quyết định của Owner.
