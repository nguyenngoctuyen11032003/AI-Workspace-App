# ADR-0016 — 18 vai trò agent mặc định là dữ liệu seed; tách Agent/Provider/User Identity

## 1. Metadata

- **ID:** ADR-0016
- **Ngày:** 2026-10-30
- **Decision ID:** D-016 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** architecture/AGENT_ORGANIZATION.md, AGENT_ARCHITECTURE.md §3.1

## 2. Context

- Chủ sở hữu yêu cầu thêm/xóa/thay thế vai trò agent mà không deploy lại.
- Một đăng nhập provider duy nhất phục vụ nhiều agent nên quota/cost phải tính theo credential chứ không theo agent.
- DB hiện chỉ có `agents.is_template`, thiếu template version (AGENT_ARCHITECTURE §3.1.1).

## 3. Problem

- Chọn mô hình dữ liệu cho vai trò và định danh sao cho thay đổi tổ chức không cần đổi mã.
- Hai nguồn sự thật `agents.status` vs `tasks.status`; ánh xạ 5 level quyền → enum 4 giá trị (CF-008).

## 4. Options considered

### Phương án A — 18 vai trò là seed data trong DB, không enum trong code; tách Agent/Provider/User Identity; template có version, instance pin [ĐỀ XUẤT]

Không `switch(role)`; ownership theo glob; thêm template thứ 19 không đổi mã; thêm `agent_templates` + `agent_template_versions` (CF-036).
- **Lý do chọn đề xuất:** Khớp yêu cầu động; Provider Identity theo credential cho quota/cost.

### Phương án B — Danh sách vai trò cố định trong code [Bị loại]

Đơn giản.
- **Lý do loại:** Đổi tổ chức phải deploy; trái yêu cầu của chủ sở hữu.

### Phương án C — Gộp Reviewer/Release vào QA [Bị loại]

Ít vai trò hơn.
- **Lý do loại:** Mất chủ thể độc lập cho gate «COMPLETED cần bằng chứng» (AGENT_ORGANIZATION §9).

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Seed 18 vai trò; template L4 do người dùng tạo hay chỉ seed là câu hỏi mở (AGENT_ARCHITECTURE §7).
- Audit ghi bộ ba (userId, agentId, providerCredentialId); instance pin `templateVersion`.
- Đổi `prisma/**` trong seed thành `packages/db/**`.

## 6. Rationale

- Khớp yêu cầu thêm/xóa/thay thế vai trò.
- Một tài khoản team phục vụ mọi agent nên Provider Identity phải tách (quan sát do tác giả báo cáo).
- Pin version tránh phá execution đang chạy.

## 7. Consequences

**Tích cực**
- Linh hoạt, không deploy lại.
- Quota/cost theo credential.

**Tiêu cực**
- Cần migration thêm bảng template.
- Prompt role phụ thuộc skill; drift nếu mapping không được test.

**Trung tính**
- Cơ chế pause/resume thật của CLI trên Windows chưa kiểm.
- Ánh xạ permission 5 level → 4 giá trị cần chốt.

## 8. Security implications

- Template/seed quyết định quyền ghi file và tool: thay đổi template phải qua duyệt và audit, không để agent tự sửa template của mình.
- Tách Provider Identity ngăn nhầm lẫn khi tính quota/audit theo credential.
- Role Reviewer/Release độc lập giữ nguyên tắc tách nhiệm vụ.

## 9. Cost and operational implications

- Chi phí migration và quản lý version template.
- Chi phí test mapping role → skill.
- Không có chi phí hạ tầng thêm.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| `claude auth status` trả team subscription, phục vụ mọi agent | `docs/architecture/AGENT_ORGANIZATION.md §8` | Chưa kiểm chứng |
| Thiếu template version và mâu thuẫn M-01..M-08 | `docs/AGENT_ARCHITECTURE.md §3.1.1, §8` | Đã xác minh |
| Các quyết định tổ chức còn lại là đề xuất | `docs/architecture/AGENT_ORGANIZATION.md §9` | Giả định |
| Pause/resume trên Windows | `docs/AGENT_ARCHITECTURE.md §7 mục 4` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Mâu thuẫn tài liệu xác minh được và có một quan sát thực do tác giả báo cáo; mô hình tổ chức là thiết kế chưa thử.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận 18 vai trò là seed data và thêm bảng template.
- Chốt ai được tạo template (chỉ seed hay người dùng).
- **Khả năng đảo ngược:** Trung bình: đổi mô hình dữ liệu sau khi có agent chạy cần migration.
- **Bị chặn trong lúc chờ duyệt:** Schema agents/templates, seed data, UI quản lý vai trò.

## 12. Reversal conditions

- Seed 18 vai trò không đủ linh hoạt: nhà thiết kế role đầy đủ cho người dùng.
- Multi-user khiến Provider Identity theo người dùng.
- Số vai trò thực tế sau Phase 1 khác xa 18.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-016 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- 18 Agent roles là seed data, không hardcode. Phân tách Agent Identity, Provider Identity, User Identity.
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
