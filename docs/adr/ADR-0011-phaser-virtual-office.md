# ADR-0011 — Virtual Office dùng Phaser 3.90, React ↔ Phaser qua Zustand + event bus

## 1. Metadata

- **ID:** ADR-0011
- **Ngày:** 2026-10-30
- **Decision ID:** D-011 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** research/PIXEL_AGENTS_ANALYSIS.md, research/LICENSE_AND_REUSE.md §5.1, research/TECHNOLOGY_EVALUATION.md TE-05, VIRTUAL_OFFICE.md, QA_REVIEW_PHASE0.md (P2-09)

## 2. Context

- Virtual Office 2D cần 10–50 nhân vật, tilemap tối đa ~100×100, zoom/pan, layout editor, tích hợp React 19.
- Pixel Agents (MIT) chạy Canvas thuần; SCREEN_SPEC ghi «PixiJS/Canvas 2D», lệch với D-011 (CF-018); tile 32 vs 16 và số animation cũng lệch (CF-019, CF-020).
- QA (P2-09, `npm view`): `phaser@3.90.0` tồn tại nhưng `latest` = 4.2.1; D-011 chỉ nói «cân nhắc Phaser 4 sau» mà không ghi lý do ở lại 3.x.

## 3. Problem

- Chọn engine render Office và ghi lý do giữ 3.x thay vì 4.x cùng điều kiện nâng cấp.
- Chưa có số đo hiệu năng; ngưỡng p95 16,6 ms với 18 agent + 8 helper chỉ là đề xuất.

## 4. Options considered

### Phương án A — Phaser 3.90 sau interface `OfficeRenderer` [ĐỀ XUẤT]

Tilemap (Tiled), camera, input, animation có sẵn; React giao tiếp qua Zustand `getState()/subscribe` + event bus; tham khảo logic Pixel Agents (BFS, FSM, layout editor, seat).
- **Lý do chọn đề xuất:** Giảm công sức viết camera/editor; cô lập sau interface giữ đường lui rẻ.

### Phương án B — PixiJS 8 + pixi-viewport + easystarjs [Bị loại]

Nhẹ (~150–300 KB).
- **Lý do loại:** Phải ghép tilemap/viewport từ plugin cộng đồng.

### Phương án C — Canvas 2D thuần, dùng `renderer.ts` của Pixel Agents [Bị loại]

Không phụ thuộc, đã chạy 10–20 nhân vật.
- **Lý do loại:** Tự viết camera, input, tilemap, editor.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Phaser 3.90.x, pin chính xác, bọc sau `OfficeRenderer`; không tạo `Phaser.Game` trong render body (tạo trong `useEffect`, hủy ở cleanup).
- **Lý do ở 3.x (đề xuất, cần chủ sở hữu xác nhận):** 3.90.0 là bản đã dùng trong nghiên cứu và tài liệu/ví dụ/plugin chủ yếu nhắm 3.x; Phaser 4 là major mới (QA thấy `latest` 4.2.1; ngày phát hành 4.0.0 trong tài liệu cũ chưa kiểm) nên rủi ro breaking API và plugin chưa được đánh giá trong repo này.
- **Điều kiện nâng lên 4.x:** (a) spike chạy tilemap 40×30 + 30 sprite + BFS + zoom/pan trên 4.x đạt p95 ≤ 16,6 ms; (b) plugin/tilemap loader đang dùng hỗ trợ 4.x; (c) bundle không tệ hơn; (d) `OfficeRenderer` có test hợp đồng; (e) nâng qua ADR mới, không đổi âm thầm.
- Mọi chuyển động phải có `causedByEventId`; không wander ngẫu nhiên (FR-160). Sửa SCREEN_SPEC §3.3 thành Phaser (CF-018).

## 6. Rationale

- Camera, input, animation sẵn có giảm công sức editor.
- Cô lập sau interface cho phép đổi sang Pixi/Canvas nếu bundle/hiệu năng không đạt.
- Zustand cho phép scene Phaser đọc state ngoài React.

## 7. Consequences

**Tích cực**
- Phát triển editor nhanh hơn.
- Đường lui rẻ.

**Tiêu cực**
- Bundle lớn hơn Pixi/Canvas (~1,2 MB gzip cho v3, chưa tự kiểm).
- Chỉ port logic Pixel Agents, bỏ `renderer.ts`/`spriteCache.ts`.
- Nâng 3 → 4 sau này có chi phí.

**Trung tính**
- Xung đột tile 32 vs 16 và số animation cần chốt riêng.
- Số phiên bản Phaser/Pixi trong tài liệu cũ chưa tự kiểm.

## 8. Security implications

- Asset đóng gói chỉ CC0/MIT/OFL, cấm asset/layout/prompt My Virtual Office (ADR-0012).
- Layout/sprite do người dùng nhập là input không tin cậy: validate schema, giới hạn kích thước.
- Dependency lớn: ghim version, quét license/CVE trong CI.

## 9. Cost and operational implications

- Bundle lớn ảnh hưởng thời gian tải; ngân sách chưa đặt.
- Spike 2–3 ngày đề xuất (TE-05).
- Chi phí nâng 4.x nếu phải viết lại phần phụ thuộc API.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Pixel Agents dùng Canvas thuần (đã đọc mã) | `docs/research/PIXEL_AGENTS_ANALYSIS.md §0, §4.1` | Đã xác minh |
| `phaser@3.90.0` tồn tại; `latest` = 4.2.1 | `docs/QA_REVIEW_PHASE0.md (bảng npm view, P2-09)` | Đã xác minh |
| Số phiên bản Phaser 4.0.0 ngày 10/04/2026 và PixiJS 8.19.0 | `docs/research/LICENSE_AND_REUSE.md §5.1` | Chưa kiểm chứng |
| Phaser 3.90 đạt 60 fps với 30 sprite + tilemap 40×30 | `docs/research/TECHNOLOGY_EVALUATION.md TE-05` | Chưa kiểm chứng |
| Mâu thuẫn engine/tile/animation | `docs/VIRTUAL_OFFICE.md §21` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Chỉ xác minh được Pixel Agents dùng Canvas và sự tồn tại của hai phiên bản; chưa prototype Phaser hay đo hiệu năng.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận ở lại Phaser 3.90 và điều kiện nâng 4.x ở trên.
- Chấp nhận Office tách sau `OfficeRenderer`.
- **Khả năng đảo ngược:** Dễ nếu giữ interface `OfficeRenderer`; đắt nếu rò rỉ API Phaser ra ngoài.
- **Bị chặn trong lúc chờ duyệt:** Mã Office/Phaser, spike hiệu năng; không chặn Phase 1 lõi.

## 12. Reversal conditions

- Spike không đạt 60 fps trên máy tham chiếu: chuyển Pixi.
- Bundle vượt ngân sách tải: chuyển Pixi hoặc Canvas.
- Chủ sở hữu yêu cầu đổi engine, hoặc điều kiện nâng 4.x thỏa.
