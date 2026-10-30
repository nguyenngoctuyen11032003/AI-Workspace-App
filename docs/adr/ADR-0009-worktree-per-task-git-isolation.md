# ADR-0009 — Mỗi task một git worktree + branch, manifest quyền file, diff check là cổng cuối

## 1. Metadata

- **ID:** ADR-0009
- **Ngày:** 2026-10-30
- **Decision ID:** D-009 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** architecture/WORKSPACE_AND_GIT_ISOLATION.md, security/PERMISSION_MODEL.md §7

## 2. Context

- Tối đa 18 agent ghi code song song vào cùng repo gây xung đột và khó truy vết.
- Nhiều tài liệu dùng quy ước branch khác nhau (`wt/<run>/<node>/<attempt>`, `wf/<runId>/<nodeId>/r<rev>`); manifest JSON `readOnly` vs YAML `readonly` (CF-032, CF-046).

## 3. Problem

- Cần cách ly ghi, đơn vị attribution/revert rõ ràng và quy tắc không agent nào tự merge vào main.
- Cần enforcement không chỉ dựa vào hook.

## 4. Options considered

### Phương án A — 1 task = 1 worktree = 1 branch `agent/<slug>/<task-id>`; luồng `task → integration/<project> → main` [ĐỀ XUẤT]

Manifest quyền sở hữu file theo glob (`writable`/`readOnly`/`forbidden`); hook là lớp bắt buộc; diff check sau cùng là cổng quyết định; không tự merge vào main.
- **Lý do chọn đề xuất:** Đơn vị task trùng đơn vị retry, review, attribution; diff check bắt ghi ngoài `writable` kể cả khi hook bị bypass.

### Phương án B — Workspace chung cho mọi agent [Bị loại]

Đơn giản.
- **Lý do loại:** Không cách ly, xung đột ghi file, khó revert theo task, khó chứng minh ai sửa gì.

### Phương án C — Clone đầy đủ per agent [Bị loại]

Cách ly rất mạnh.
- **Lý do loại:** Tốn đĩa và đồng bộ object; worktree dùng chung object store rẻ hơn.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Tên branch theo D-009, thay mọi quy ước khác (CF-032); manifest JSON `schemaVersion: 1`, key `readOnly`, đặt ngoài worktree (CF-046).
- Ba lớp enforcement: hook, quyền thư mục, diff check Lớp 3.
- Cần cơ chế phát hiện giao glob giữa manifest (PERMISSION_MODEL §14 mục 5).

## 6. Rationale

- Integration branch cho phép review tự động trước khi con người duyệt merge vào main.
- Diff validation sau cùng chặn ghi ngoài phạm vi.
- Chi phí đĩa thấp nhờ object store chung.

## 7. Consequences

**Tích cực**
- Attribution/revert theo task.
- Cách ly ghi file.

**Tiêu cực**
- Phải quản lý vòng đời worktree (tạo, dọn, retry tạo branch mới).
- Merge vào integration cần xử lý conflict.
- Đường dẫn dài/khoảng trắng trên Windows phải xử lý.

**Trung tính**
- Worktree host-truth vs volume-truth chưa chốt (SANDBOX_DESIGN §14).
- Không áp dụng cho dữ liệu ngoài repo.

## 8. Security implications

- Agent chạm worktree khác qua đường dẫn tuyệt đối nếu lọt hook (R4): diff check và quyền thư mục là lớp thứ hai.
- Manifest đặt ngoài worktree để agent không tự sửa quyền của mình.
- Không agent nào có quyền push/merge vào main; credential git do người dùng quản.

## 9. Cost and operational implications

- Mỗi worktree tốn đĩa và thời gian tạo; chưa đo ở 5–18 worktree song song.
- Cần job dọn worktree/branch mồ côi.
- Chi phí review integration branch thuộc người dùng.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Mô hình one task = one worktree = one branch và lệnh `git worktree add -b` được mô tả | `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md §5` | Giả định |
| Mâu thuẫn schema manifest JSON vs YAML | `docs/security/PERMISSION_MODEL.md §7.1` | Đã xác minh |
| Mâu thuẫn tên branch M-O6 | `docs/MULTI_AGENT_ORCHESTRATION.md §15` | Đã xác minh |
| Worktree song song chạy ổn trên Windows | `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Mô hình git chuẩn nhưng chưa thử nghiệm worktree song song nào; chỉ xác minh được các mâu thuẫn tài liệu.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận quy ước branch và luồng `task → integration → main`, cấm tự merge vào main.
- Chấp nhận manifest JSON ngoài worktree.
- **Khả năng đảo ngược:** Trung bình: đổi sau khi có mã quản worktree tốn công nhưng cô lập trong một module.
- **Bị chặn trong lúc chờ duyệt:** Mã quản worktree, manifest enforcement, diff check.

## 12. Reversal conditions

- Thử nghiệm 5–18 worktree song song cho thấy chi phí không chấp nhận được: sparse checkout hoặc clone nông.
- Chủ sở hữu cho phép tự merge vào main sau gate đủ mạnh.
- Repo mục tiêu quá lớn khiến worktree không khả thi.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-009, §2 Q-04 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Mỗi task một worktree/branch riêng; review diff trước khi merge; không tự động merge vào `main`; Owner là người phê duyệt cuối cùng.
- Q-04: Agent merge vào `integration/*` khi vượt các gate; merge vào `main` luôn cần Owner approval; Lead Agent không thay thế Owner.
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
