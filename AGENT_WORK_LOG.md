# AGENT_WORK_LOG.md

Nhật ký công việc của các agent. Chỉ ghi việc thực sự đã chạy. Giờ theo máy, ngày 2026-10-30.

## Đợt 1: nghiên cứu song song (10 agent, chạy thật bằng công cụ Agent của Claude Code)
Khởi chạy cùng lúc 10 agent nghiên cứu độc lập. Tất cả hoàn thành và ghi file.
| Agent | Phạm vi | Kết quả |
|---|---|---|
| ChatDev | CHATDEV_ANALYSIS.md | Hoàn thành |
| Pixel Agents | PIXEL_AGENTS_ANALYSIS.md | Hoàn thành |
| My Virtual Office | MY_VIRTUAL_OFFICE_ANALYSIS.md | Hoàn thành |
| OpenHands | OPENHANDS_ANALYSIS.md | Hoàn thành |
| Claude Code | CLAUDE_CODE_INTEGRATION.md, có smoke test thật (5 tiến trình `claude -p` song song, không lỗi) | Hoàn thành |
| Codex | CODEX_INTEGRATION.md, không chạy live (CLI chưa cài) | Hoàn thành |
| Orchestration | ORCHESTRATION_WORKFLOW_RESEARCH.md | Hoàn thành |
| Database | DATABASE_DESIGN.md | Hoàn thành |
| Security | SECURITY.md | Hoàn thành |
| License và stack | LICENSE_AND_REUSE.md | Hoàn thành |

## Đợt 2: tài liệu sản phẩm, thiết kế, hạ tầng (6 agent)
Khởi chạy khi có slot trống. Thiết kế (Design System), BA, kiến trúc 18 agent, DevSecOps, Skill System, UX Research.
- Design System, DevSecOps, Skill System: hoàn thành.
- BA, 18-agent, UX Research: bị ngắt giữa chừng do hết hạn mức 5 giờ của model (HTTP 429, mở lại 1:40pm). Kiểm tra sau đó cho thấy toàn bộ file chính của ba agent đã được ghi đủ. File còn thiếu duy nhất là USABILITY_AUDIT_CHECKLIST.md, do Orchestrator viết bù. Không có bằng chứng nội dung bị cắt dở, nhưng QA-001 sẽ kiểm tra lại.

## Đợt 3: tổng hợp kiến trúc (6 agent, model sonnet)
Kiến trúc hệ thống, agent/workflow/event, provider và Virtual Office, 5 tài liệu bảo mật chi tiết, so sánh/reuse/technology, dev docs/roadmap/rủi ro. Cả 6 hoàn thành và mỗi agent báo cáo các mâu thuẫn tìm thấy giữa tài liệu nguồn.

## Đợt 4: ADR và QA (2 agent, model sonnet)
ADR cho D-001..D-020 kèm danh sách mâu thuẫn, và review kỹ thuật độc lập. Đang chạy lúc ghi.

## Công việc do Orchestrator tự làm
Kiểm tra môi trường, DECISION_LOG, phân tích herdr theo yêu cầu bổ sung (HERDR_ANALYSIS.md, HERDR_DESIGN_INTEGRATION.md, D-017..D-020), USABILITY_AUDIT_CHECKLIST.md, AGENTS.md, các file theo dõi.

## Giới hạn đã gặp
- Tối đa 10 agent chạy đồng thời trong đợt 1 (đạt đúng 10). Chưa từng chạy 18 agent đồng thời, nên **chưa có bằng chứng** hệ thống hỗ trợ 18.
- Heredoc dài lỗi ENAMETOOLONG trên Windows, nhiều agent chuyển sang công cụ Write.
- Hạn mức tài khoản là ràng buộc thật khi chạy nhiều agent.

## Đợt 5: Finalization (7 agent song song, model sonnet) + 1 agent đồng bộ
| Agent | Phạm vi | Kết quả |
|---|---|---|
| BA phần 1 | AC cho FR-001..100 (76 FR) | Hoàn thành |
| BA phần 2 | AC cho FR-101..199 (62 FR) | Hoàn thành |
| BA phần 3 (thêm sau khi phát hiện bỏ sót) | AC cho FR-200..292 (61 FR) | Hoàn thành |
| Decision Analyst | Sổ quyết định, phân loại 55 mâu thuẫn | Hoàn thành |
| System Architect | Viết lại 20 ADR | Hoàn thành |
| Security Reviewer | Danh mục ký duyệt, rủi ro còn lại | Hoàn thành |
| Technical Validation | Lệnh đọc, 27 runbook | Hoàn thành |
| Documentation (đồng bộ có giới hạn) | 29 thay đổi, log | Hoàn thành |
| QA độc lập lần hai | Hoàn thành: PASS WITH FIXES (0 P0, 3 P1, 8 P2, 6 P3). Sau đó Orchestrator sửa F-01 (số liệu), F-02, F-03 (đặc tả), F-04, F-05; các sửa này chưa được QA kiểm lại | |
Số agent chạy đồng thời tối đa trong đợt này: 7. Chưa từng đạt 18.

## Hoàn tất công việc của 3 agent bị ngắt
Sau khi đối chiếu, 3 agent (BA, 18-agent, UX) không còn nội dung thiếu. Hai mục theo mẫu đã được Orchestrator bổ sung (NFR Acceptance Criteria, checklist usability Rủi ro). Trạng thái đỏ trên giao diện chỉ phản ánh lần dừng do hạn mức, không phải nội dung thiếu.

## Đợt 6: OWR-2 (6 agent song song, model sonnet)
| Agent | Phạm vi | Kết quả |
|---|---|---|
| Decision Records | register, DECISION_LOG, ADR, CONFLICTS | Hoàn thành: 16 APPROVED, 5 APPROVED_CONDITIONAL, 9 mục chờ làm rõ, 12 xung đột |
| Sync A | bảo mật, provider, git, roadmap | Hoàn thành: 44 thay đổi, 10 xung đột để dấu |
| Sync B | DB, event, workflow, thiết kế | Hoàn thành: 30 thay đổi, 6 xung đột |
| BA | AC cho FR-122/181/182/183, FR-109/113/164, D-021 | Hoàn thành: 4 đề xuất, 12 chờ Owner |
| Drizzle PoC | scratch, PGlite | 9 PASS, 1 FAIL, 1 BLOCKED |
| Skill/Resume PoC | scratch, 8 lần gọi haiku | SK-01/02/03/05 PASS mức script, SK-04 SIMULATED |
Số agent chạy đồng thời tối đa: 6. Chưa từng đạt 18.
