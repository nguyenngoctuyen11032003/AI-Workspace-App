# NEXT_SESSION.md

Cập nhật: 2026-10-30 (sau OWR-3). Đọc `AGENTS.md` (mục 8) và `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` trước.

## 1. Tóm tắt dự án
AI WORKSPACE: nền tảng điều phối nhiều AI agent mô phỏng công ty phần mềm. Claude Agent SDK là adapter chính (D-004). Codex và OpenHands đã hoãn khỏi MVP. Stack: React 19 + Vite, NestJS, pnpm monorepo, PostgreSQL 16 + Redis, Drizzle. Owner nghiệm thu cuối cùng.

## 2. Phase hiện tại
**Phase 0 MỞ. Phase 1 chưa được cho phép.** Không chuyển Phase 0 thành Completed, không bắt đầu Phase 1 khi chưa có xác nhận rõ ràng của Owner.

## 3. Tiến độ
Quyết định OWR-2 đã ghi nhận (16 APPROVED, 5 APPROVED_CONDITIONAL, 0 VERIFIED). Tài liệu đã đồng bộ phần có căn cứ (74 thay đổi). PoC Drizzle và Skill/Resume đã chạy (Partial). Docker BLOCKED. QA lần ba chưa chạy.

## 4. Đã hoàn thành
Xem `PROJECT_STATUS.md` và `PHASE_0_FINAL_REPORT.md` mục A đến D.

## 5. Chưa hoàn thành
- Câu hỏi PENDING_OWNER_CLARIFICATION (9 mục): danh sách event gốc, Q-07 ToS, FR-122/181/182/183, tile 16 hay 32, D-005 đổi MVP B, MIT asset.
- 12 FR NEEDS_OWNER_CLARIFICATION.
- Xung đột X-1..X-12 và các mục "Not synced" trong `docs/DOC_SYNC_LOG_OWR2_A.md`, `docs/DOC_SYNC_LOG_OWR2_B.md`.
- QA độc lập lần ba.
- Docker validation (`docs/poc/POC_DOCKER_SANDBOX.md`), Drizzle trên PG16, SK-04 với agent thật, test qua Agent SDK, Conductor PoC (Q-20).

## 6. Task đang làm dở
Không có agent chạy.

## 7. Quyết định quan trọng
`PHASE_0_OWNER_DECISION_REGISTER.md` mục 9 đến 12, `DECISION_LOG.md`, `docs/adr/` (ADR-0001..0021).

## 8. Vấn đề đang gặp
`KNOWN_ISSUES.md` (19 mục), `PHASE_0_REMAINING_RISKS.md`.

## 9. File đã thay đổi
Xem nhật ký đồng bộ `docs/DOC_SYNC_LOG_OWR2_A.md`, `docs/DOC_SYNC_LOG_OWR2_B.md` và `git log`. Thư mục `docs/poc/` và `docs/owner-responses/` là mới.

## 10. Chỉ dẫn cho phiên tiếp theo
1. Chờ Owner trả lời các mục chờ. Không tự điền thông tin thiếu.
2. Khi Owner tự mở Docker Desktop và báo `docker version`, `docker info` thành công: chạy VAL-DOCKER-01..09 theo `docs/poc/POC_DOCKER_SANDBOX.md` rồi cập nhật.
3. Chạy QA độc lập lần ba sau khi các xung đột được Owner quyết.
4. Chỉ khi Owner xác nhận rõ ràng mới bàn tới Phase 1; lộ trình theo `docs/IMPLEMENTATION_ROADMAP.md` (phụ lục OWR-2: Phase 2 chỉ mock/sandbox nếu cổng an ninh chưa đạt).

## 11. Agent cần khởi chạy
QA độc lập (không phải tác giả tài liệu); Documentation (đồng bộ theo quyết định mới); BA (AC cho FR sau khi Owner làm rõ); Validation Engineer (Docker, khi Docker sẵn sàng).

## 12. Chạy song song được
Đồng bộ theo miền không chung file. Các PoC độc lập nhau trừ Docker (cần Owner). Không chạy nhiều model song song khi đang đo quota.

## Cập nhật sau OWR-3
- Việc còn chờ Owner: X-13 (ba câu), P-16 và X-14 (bảng ánh xạ event), Q-21..Q-26, quyền commit, quyền Docker.
- Sau khi Owner trả lời: đồng bộ thân SECURITY mục 2 đến 5 và SANDBOX_DESIGN mục 4, ánh xạ event, rồi QA ngắn kiểm các sửa sau lần ba.
- Chưa bắt đầu Phase 1. Chưa push. Thay đổi OWR-3 chưa commit.
