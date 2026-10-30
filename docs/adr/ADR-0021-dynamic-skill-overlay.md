# ADR-0021 — Skill Overlay Directory: nạp skill Markdown động theo từng agent

## 1. Metadata
- **ID:** ADR-0021
- **Ngày:** 2026-10-30
- **Decision ID:** D-021 (`DECISION_LOG.md`)
- **Decision status:** APPROVED_CONDITIONAL (OWR-2, Q-18). Chấp thuận có điều kiện cho đến khi OWN-AC-1..3 được kiểm thử thành công
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** `docs/architecture/DYNAMIC_SKILL_INJECTION.md`, `docs/poc/POC_DYNAMIC_SKILL.md`, `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`

## 2. Context
Yêu cầu của Owner: agent không bị đóng băng prompt. Sau khi hệ thống chạy hoặc làm xong sản phẩm, Owner thêm, sửa, xóa file Markdown trong `.ai/skills/<agent_name>/` và task mới tự nhận nội dung, không restart và không biên dịch lại. Skill Library hiện có (D-013) chỉ có phiên bản bất biến qua quy trình publish.

## 3. Problem
1. Nạp nội dung file vào ngữ cảnh Claude Code mỗi lần giao task mà không đổi mã.
2. Claude Code mặc định ghi lại system prompt một lần mỗi hội thoại và dùng lại khi resume (`--system-prompt-snapshot`), nên thay đổi skill không có hiệu lực khi resume nếu không xử lý.
3. Agent có thể ghi vào thư mục overlay qua Bash trên host Windows; Owner và agent có thể dùng cùng một OS user, nên không phân biệt được người sửa.

## 4. Options considered
- **A. Overlay trên đĩa cộng Skill Library, quét lúc giao task, guard phát hiện thay đổi (đề xuất, được duyệt có điều kiện).**
- **B. Chỉ Skill Library, chỉnh qua giao diện.** Loại: không đáp ứng yêu cầu sửa file trực tiếp của Owner.
- **C. Hot-reload giữa task.** Loại: Claude Code giữ system prompt đã ghi, và làm tăng chi phí cache.

## 5. Decision status and what is being proposed
Được duyệt có điều kiện: A. Ba tiêu chí bắt buộc của Owner:
- **OWN-AC-1** Dynamic Skill Loading: task mới nhận nội dung mới, không restart hay recompile.
- **OWN-AC-2** Unauthorized Overlay Modification: ghi audit log, đánh dấu đáng ngờ, tạm dừng agent cho tới khi Manager/Owner xác nhận, không cho agent tự tiếp tục. Hệ thống **không** tuyên bố xác định danh tính người sửa; chỉ xác định thay đổi, thời điểm, phiên thực thi và bằng chứng audit.
- **OWN-AC-3** Resume Session Context Refresh: kiểm tra hash của Skill và System Prompt khi resume, nếu đổi thì tạo session mới hoặc refresh context bằng cơ chế đã được PoC chứng minh, ghi audit log.

## 6. Rationale
Quét tại thời điểm giao task không phụ thuộc bộ theo dõi tệp. Overlay không cấp quyền. Guard dựa trên băm nội dung và thời điểm, trung thực về giới hạn danh tính.

## 7. Consequences
- Tích cực: Owner chỉnh quy chuẩn tức thì cho task kế tiếp.
- Tiêu cực: thêm bộ quét, guard, trạng thái `PAUSED_PENDING_CONFIRMATION`; resume theo cách làm mới tốn thêm chi phí.
- Trung tính: cần thêm FR-293..298 (đề xuất) và AC-601.. (xem traceability).

## 8. Security implications
Overlay là văn bản đưa vào system prompt, nên: loại khóa quyền khỏi frontmatter, quét bí mật, từ chối symlink và `..`, `.ai/**` thuộc `forbidden` của manifest, cách ly giữa agent, không mount overlay vào container Tier 2. Rủi ro còn lại: cùng OS user ở Tier 1 thì tiến trình có thể ghi đè, chỉ phát hiện được sau sự việc. Owner đã chấp nhận có điều kiện ở Phase 0 và 1 (OWR-1).

## 9. Cost and operational implications
Chi phí quét không đáng kể (PoC: 0,103 s tổng thời gian quét và render). Resume làm mới bằng `--system-prompt-snapshot off` đo được tốn thêm khoảng 47% trong một phép đo, chỉ một mẫu, chưa đủ để kết luận.

## 10. Validation evidence
| Khẳng định | Bằng chứng | Trạng thái |
|---|---|---|
| Task mới nhận nội dung mới, không restart (SK-01, SK-02) | `docs/poc/POC_DYNAMIC_SKILL.md`, 8 lần gọi model, 0,006283 USD | Đã chạy bằng script cộng CLI; chưa qua Agent SDK |
| Resume giữ system prompt cũ, `--system-prompt-snapshot off` và hash-check làm mới (SK-03) | cùng báo cáo, mục 5 | PASS cơ chế, có điều kiện, một mẫu chi phí |
| Agent sửa overlay thì bị phát hiện, dừng (SK-04) | cùng báo cáo, mục 6 | **SIMULATED**: agent thật bị chặn quyền ghi hai lần, phần ghi tệp là script |
| Owner xác nhận mới tiếp tục (SK-05) | cùng báo cáo, mục 7 | PASS ở mức guard/CLI, chưa tích hợp Approval Center |
| Hành vi qua Claude Agent SDK | không | NOT TESTED |
Mức bằng chứng: **Partial**. Không có mục nào VERIFIED.

## 11. Owner approval requirement
Đã duyệt có điều kiện. Để chuyển sang VERIFIED cần: OWN-AC-1..3 kiểm thử đạt trên đường tích hợp thật (Agent SDK, agent thật cho SK-04, Approval Center), không chỉ script. Owner còn cần xác nhận FR-293..298 và vị trí thư mục (Q-18).

## 12. Reversal conditions
Đảo nếu Claude Code đổi hành vi snapshot khiến refresh không còn đáng tin, hoặc nếu guard sinh quá nhiều dương tính giả khi Owner sửa file lúc agent chạy.
