# ADR-0013 — Skill theo chuẩn Agent Skills, mở rộng qua skill.yaml, Memory tách khỏi Skill

## 1. Metadata

- **ID:** ADR-0013
- **Ngày:** 2026-10-30
- **Decision ID:** D-013 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** architecture/SKILL_SYSTEM.md

## 2. Context

- Cần hệ thống Skill/Teach Agent dùng chung cho Claude Code, Codex và OpenHands; mỗi CLI discover skill ở đường dẫn khác nhau.
- Chuẩn mở chỉ cho 6 field frontmatter (field lạ gây hard error trên claude.ai).

## 3. Problem

- Agent không được tự ghi kiến thức thẳng vào repo; cần luồng duyệt và pin version.
- Windows: junction thay symlink.

## 4. Options considered

### Phương án A — Chuẩn Agent Skills; mở rộng trong `skill.yaml` + `metadata.aiw-*`; Claude nạp `--add-dir`, Codex junction vào `.agents/skills` [ĐỀ XUẤT]

Memory tách khỏi Skill, có luồng duyệt; progressive disclosure 3 tầng; pin version; validate bằng `skills-ref validate`.
- **Lý do chọn đề xuất:** Tương thích hệ sinh thái; tách duyệt khỏi mã.

### Phương án B — Ghi trực tiếp skill/memory vào repo [Bị loại]

Đơn giản.
- **Lý do loại:** Không kiểm soát, lẫn kiến thức chưa duyệt vào mã nguồn và lịch sử git.

### Phương án C — Định dạng skill riêng của AI WORKSPACE [Bị loại]

Toàn quyền thiết kế.
- **Lý do loại:** Không dùng lại hệ sinh thái và phải tự viết projection cho mọi CLI.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Extension chỉ nằm trong `skill.yaml`/`metadata.aiw-*`, không thêm field lạ vào frontmatter.
- Skill pin theo version; chọn skill theo relevance; Memory có luồng duyệt riêng.
- Projection layer riêng cho từng provider; OpenHands projection ngoài MVP.

## 6. Rationale

- Chuẩn mở được Claude Code và Codex hỗ trợ (tác giả đọc spec ngày 2026-10-30).
- Tách Memory khỏi Skill cho phép duyệt và thu hồi không đổi mã.
- Progressive disclosure giữ chi phí token ban đầu thấp (~100 token/skill).

## 7. Consequences

**Tích cực**
- Tái dùng skill giữa provider.
- Có kiểm duyệt.

**Tiêu cực**
- Phải xây projection layer.
- Giới hạn description + when_to_use cắt ở 1 536 ký tự; compaction giữ 5 000 token/skill.

**Trung tính**
- Trùng `name` Codex không merge.
- `~/.codex/skills` cũ còn hoạt động hay không là giả định.

## 8. Security implications

- Skill là nguồn prompt injection/supply-chain: chỉ nạp skill đã duyệt, pin version, kiểm hash.
- Memory chưa duyệt không được vào context agent khác.
- Junction có thể bị AV/quyền chặn; không tạo link ra ngoài workspace.

## 9. Cost and operational implications

- Tốn token mỗi skill nạp (~100 token metadata).
- Chi phí duy trì projection và test discover cho từng provider.
- Luồng duyệt Memory tốn thời gian người dùng.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Spec agentskills.io fetch 2026-10-30; discover của Claude/Codex | `docs/architecture/SKILL_SYSTEM.md §3.1–3.3` | Chưa kiểm chứng |
| Cấu trúc repo ChatDev/OpenHands và §3.4–3.6 đã được QA đối chiếu | `docs/QA_REVIEW_PHASE0.md (mục chỉ spot-check)` | Đã xác minh |
| Codex discover qua junction trên Windows | `docs/architecture/SKILL_SYSTEM.md §4.3` | Giả định |
| Schema `skill.yaml` | `docs/architecture/SKILL_SYSTEM.md §4.1.2` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Một phần thiết kế được QA đối chiếu nguồn; §3.1–3.3 và projection chưa kiểm, Codex chưa cài.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận tách Memory khỏi Skill với luồng duyệt.
- Chấp nhận schema `skill.yaml` làm extension.
- **Khả năng đảo ngược:** Dễ: định dạng nằm trong thư mục skill, projection thay được.
- **Bị chặn trong lúc chờ duyệt:** Mã Skill/Memory system và projection.

## 12. Reversal conditions

- Chuẩn Agent Skills đổi phá vỡ tương thích: cập nhật projection.
- Một provider chính ngừng hỗ trợ chuẩn: định dạng riêng cho provider đó.
- Luồng duyệt Memory quá nặng: auto-approve theo trust level.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-013 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- `SKILL.md` là chuẩn; metadata có thể mở rộng bằng `skill.yaml`. Skill và Memory là hai thành phần riêng; Memory cần cơ chế quản lý và approval phù hợp.
- Phần nạp Skill cho Codex qua junction chỉ áp dụng khi Codex quay lại (D-005 hoãn). Overlay động thuộc D-021 (APPROVED_CONDITIONAL).
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
