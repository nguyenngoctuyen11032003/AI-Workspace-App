# PHASE_0_FINAL_REPORT.md — AI WORKSPACE (phiên bản cập nhật, Finalization)

Ngày: 2026-10-30. Viết cho: Owner (Product Owner). Trạng thái: **các đầu việc kỹ thuật của Phase 0 hoàn tất, đang chờ QA độc lập lần hai và quyết định của Owner. Phase 1 chưa bắt đầu. Không quyết định nào được đánh dấu APPROVED.**

## 1. Kết luận ngắn
Phase 0 chưa được coi là đóng. Hoàn thành các đầu việc kỹ thuật không có nghĩa là Owner đã phê duyệt. QA độc lập lần hai cho kết quả **PASS WITH FIXES, 0 P0, 3 P1, 8 P2, 6 P3** (`PHASE_0_QA_REVIEW_2.md`). Không còn lỗi P0 về truy vết nghiệm thu, nhưng còn 16 FR chưa thể có tiêu chí vì thiếu thông tin, 21 quyết định kiến trúc (D-001 đến D-021) và 15 quyết định từ mâu thuẫn tài liệu đang chờ Owner, 13 rủi ro bảo mật mức "chưa chấp nhận được", và một số năng lực kỹ thuật chưa được kiểm chứng thực tế. Kết quả QA lần hai nằm ở `PHASE_0_QA_REVIEW_2.md`.

## 2. Việc đã làm trong đợt hoàn thiện
| Hạng mục | Kết quả | Bằng chứng |
|---|---|---|
| P0 truy vết nghiệm thu | Đủ 199 FR, mỗi FR đúng một dòng (kiểm tra bằng lệnh). 109 đã có AC, 74 AC mới (PROPOSED), 16 cần Owner làm rõ | `PHASE_0_ACCEPTANCE_CRITERIA_TRACEABILITY.md` |
| P0 enum provider | Đã sửa trong database design | `docs/DATABASE_DESIGN.md` |
| 55 mâu thuẫn | Phân loại lại: 15 cần Owner, 7 giải bằng bằng chứng, 33 do tài liệu cũ (2 đã sửa sẵn). Các mục chỉ đúng nếu một quyết định chờ duyệt được thông qua đã được hoãn, không đồng bộ | `docs/adr/CONFLICTS_TO_RESOLVE.md`, `PHASE_0_OWNER_DECISION_REGISTER.md` |
| Đồng bộ tài liệu nguồn | 29 thay đổi độc lập với quyết định chờ duyệt đã áp dụng, khoảng 37 nhóm mục hoãn, có nhật ký từng thay đổi | `docs/DOC_SYNC_LOG_PHASE0.md` |
| 20 ADR | Viết lại theo cấu trúc yêu cầu, tất cả PENDING_OWNER_APPROVAL. Mức bằng chứng: không ADR nào Verified, 10 Unverified, 10 Partial | `docs/adr/` |
| Kiểm chứng kỹ thuật | Chỉ chạy các lệnh đọc. 27 runbook kiểm thử cho Codex, Docker sandbox và concurrency, đều chờ Owner cho phép | `PHASE_0_TECHNICAL_VALIDATION_REPORT.md` |
| Rà soát bảo mật | Mô hình khuyến nghị, danh mục ký duyệt, 13 rủi ro NOT_ACCEPTABLE_UNTIL_MITIGATED | `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` |
| Rủi ro còn lại | 62 dòng, 17 dòng chặn Phase 1 | `PHASE_0_REMAINING_RISKS.md` |
| Yêu cầu mới: skill Markdown động | Đặc tả, quyết định D-021, 12 điểm cần sửa trong tài liệu hiện có, kế hoạch hiện thực với Claude Code | `docs/architecture/DYNAMIC_SKILL_INJECTION.md` |

## 3. Những gì tôi **không** làm, và lý do
- **Không cài Codex CLI, không bật Docker, không chạy prompt model** để kiểm chứng. Đó là thay đổi trạng thái máy hoặc tiêu quota chưa được cho phép rõ. Thay vào đó có runbook và danh sách hành động chờ Owner.
- **Không đồng bộ các giá trị phụ thuộc quyết định chưa duyệt** vào tài liệu nguồn (ORM, concurrency, tên event, bảng audit, timeout approval, cổng approval Claude, kích thước tile…). Làm vậy là biến đề xuất thành quyết định.
- **Không hợp nhất AC mới vào `ACCEPTANCE_CRITERIA.md` chính thức.** Chúng ở trạng thái PROPOSED, trong file truy vết riêng, có con trỏ từ file chính.
- **Không bắt đầu Phase 1, không viết mã ứng dụng.**

## 4. Phát hiện kỹ thuật mới đáng chú ý
1. **Skill động và resume:** `claude --help` bản 2.1.312 cho biết system prompt, kể cả phần `--append-system-prompt`, được ghi lại một lần mỗi hội thoại và dùng lại nguyên văn khi resume (`--system-prompt-snapshot`, mặc định bật). Với task mới (hội thoại mới) cơ chế quét và nạp hoạt động. Với resume, thay đổi skill sẽ không có hiệu lực nếu không xử lý. Cần PoC. (Đã xác minh bằng help, chưa kiểm thử hành vi.)
2. **Lộ trình phase có lỗ hổng an toàn:** roadmap cho agent chạy thật (Phase 2) trước khi có policy, xác thực API và audit (Phase 4). Reviewer bảo mật xếp đây là lỗ hổng hàng đầu. Cần Owner xếp lại thứ tự.
3. **Bằng chứng thực nghiệm mỏng:** mọi số đo live trong tài liệu nghiên cứu (RAM mỗi tiến trình, thời gian, 5 tiến trình song song) đều do một agent đo một lần và chưa được tái hiện. Máy hiện chỉ còn 3,9 GB RAM trống so với 6,4 GB ghi trong tài liệu.
4. **Mã FR không liên tục** (tới FR-292, 199 FR). Các cụm số AC trong `TESTING.md` và roadmap có khoảng trống đánh số cần dọn khi Owner chốt.
5. **Codex:** extension VS Code có bản codex 0.164 đã đăng nhập ChatGPT; CLI độc lập chưa cài.

## 5. Kiến trúc đề xuất (không đổi so với báo cáo trước, vẫn là đề xuất)
Monorepo TypeScript, NestJS, React 19 + Vite, PostgreSQL + Redis, Drizzle, workflow engine tự xây trên BullMQ, Claude qua Agent SDK, Codex qua `codex app-server`, sandbox hai tầng, Phaser 3 cho Virtual Office, concurrency mặc định 5 tối đa 18, thiết kế tham chiếu herdr. Chi tiết: `docs/SYSTEM_ARCHITECTURE.md`. Chưa có phần nào được Owner duyệt.

## 6. Việc cần Owner quyết định
Chi tiết, kèm phương án, hệ quả và khuyến nghị, nằm ở **`PHASE_0_OWNER_DECISION_REGISTER.md`**. Thứ tự đề xuất theo mức mở khóa công việc: D-001, D-002, D-014 (cùng Q-12, Q-13, Q-03), D-004 (cùng D-010 và Q-07), D-007 (cùng Q-01).

| Nhóm | Số lượng | Nơi xem |
|---|---|---|
| Quyết định kiến trúc | 21 (D-001..D-021) | Sổ quyết định mục 2, 3, 7 |
| Quyết định phát sinh từ mâu thuẫn | 15 (Q-01..Q-15) | Sổ quyết định mục 4 |
| FR cần làm rõ để viết được AC | 16 | Truy vết mục 2 |
| Rủi ro bảo mật cần xác nhận hoặc giảm thiểu | 13 NOT_ACCEPTABLE, 13 ACCEPTABLE nếu Owner thừa nhận | `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` |
| Hành động chỉ Owner thực hiện được | Cài Codex CLI, bật Docker, cho phép các runbook kiểm chứng, kiểm tra điều khoản subscription | Báo cáo kiểm chứng kỹ thuật |

## 7. Rủi ro còn lại
Xem `PHASE_0_REMAINING_RISKS.md` (62 dòng, 17 chặn Phase 1). Nổi bật: không có sandbox cấp hệ điều hành cho CLI trên Windows, file đăng nhập OAuth đọc được bởi tiến trình cùng người dùng, quota subscription dùng chung, chưa có bằng chứng 18 agent đồng thời, cấu hình git của orchestrator có thể chạy mã do agent ghi (phân tích của reviewer, chưa kiểm chứng), license sprite JIK-A-4 và `vendor/`, `assets/sounds` của herdr chưa rõ.

## 8. Điều kiện đóng Phase 0: hiện trạng
| Điều kiện | Trạng thái |
|---|---|
| Không còn P0 chưa xử lý | Đạt theo QA lần hai (0 P0) |
| AC hoàn thiện và truy vết | 183 trên 199 FR có AC hoặc AC đề xuất; 16 chờ Owner |
| Tài liệu đã đồng bộ | **Một phần**: đã áp dụng mục độc lập; mục phụ thuộc quyết định hoãn có chủ đích |
| Mâu thuẫn đã giải quyết hoặc ghi nhận chính xác | Có, theo `CONFLICTS_TO_RESOLVE.md` |
| ADR có trạng thái minh bạch | Có |
| QA độc lập lần hai đạt | PASS WITH FIXES, 0 P0. Ba P1: (F-01) 4 FR Must (FR-122, 181, 182, 183) chưa có AC vì cần Owner làm rõ, số liệu trong tài liệu đã sửa; (F-02) đã đổi `audit_events` thành `audit_logs`; (F-03) đã bổ sung cơ chế toàn vẹn cho overlay nhưng rủi ro ghi đè bởi tiến trình cùng người dùng ở Tier 1 vẫn còn, cần Owner thừa nhận. **Các sửa này chưa được QA kiểm lại.** |
| Rủi ro chưa giải quyết được trình bày | Có |
| Owner có đủ thông tin để quyết định | Có, qua sổ quyết định |

**Dừng tại đây.** Không bắt đầu Phase 1 cho đến khi có xác nhận rõ ràng của Owner.
