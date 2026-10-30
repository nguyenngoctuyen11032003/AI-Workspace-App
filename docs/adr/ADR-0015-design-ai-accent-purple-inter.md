# ADR-0015 — Design: accent AI tím, Inter + JetBrains Mono, mật độ Comfortable, giao diện mặc định tiếng Anh

## 1. Metadata

- **ID:** ADR-0015
- **Ngày:** 2026-10-30
- **Decision ID:** D-015 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** design/DESIGN_TOKENS.md

## 2. Context

- Cần ngôn ngữ thị giác nhất quán cho SaaS cao cấp, phân biệt phần do AI tạo với phần của người dùng.
- Tailwind v4 `@theme` là static trong khi theme phải đổi runtime; màu blocked lệch giữa SCREEN_SPEC (đỏ) và token (cam).

## 3. Problem

- Chọn accent, font, mật độ và locale mặc định.
- Quyết định Inter vs Geist được đánh dấu cần phê duyệt trong DESIGN_TOKENS §3.3.

## 4. Options considered

### Phương án A — Accent AI tím (`--ai-accent`), Inter + JetBrains Mono, Comfortable, avatar agent bo vuông / người tròn, mặc định EN (VI thứ hai) [ĐỀ XUẤT]

Primitive oklch, semantic hex; theme runtime qua `[data-theme]`; self-host font; 8 status + agent-state tái dùng status.
- **Lý do chọn đề xuất:** Tím riêng, không đụng màu trạng thái; Inter phủ glyph tiếng Việt tốt.

### Phương án B — Accent xanh [Bị loại]

Quen thuộc.
- **Lý do loại:** Trùng ngữ nghĩa màu `info/running`, khó tách «AI» khỏi trạng thái.

### Phương án C — Font Geist [Bị loại]

Hiện đại.
- **Lý do loại:** Chưa có lý do thay Inter; độ phủ glyph tiếng Việt cần kiểm.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Dùng token như phương án đề xuất; đổi tên `--ai-accent` để tránh va chạm shadcn `--accent`.
- Màu blocked = orange, failed = red (CF-021).
- Locale mặc định EN, VI là locale thứ hai; token theo cấu trúc herdr (ADR-0018).

## 6. Rationale

- Tách rõ màu hành động/AI và màu trạng thái.
- oklch nội suy đều; semantic hex cho QA đo contrast.
- Self-host font phù hợp offline desktop.

## 7. Consequences

**Tích cực**
- Nhận diện AI rõ.
- Contrast text-tertiary `#71717A` ≥ 4,5:1 theo tác giả.

**Tiêu cực**
- Font self-host tăng bundle.
- Tím có thể bị xem là «AI cliché» nếu lạm dụng.
- Mặc định EN có thể không hợp người dùng VI đầu tiên.

**Trung tính**
- Cần cơ chế runtime var cho Tailwind v4 (T2).
- Màu primitives là đề xuất.

## 8. Security implications

- Không ảnh hưởng bảo mật trực tiếp; font self-host tránh gọi CDN bên thứ ba (rò rỉ IP/riêng tư).
- Màu trạng thái phải phân biệt được cho người mù màu để tránh bỏ sót cảnh báo bảo mật (blocked/failed).
- Giấy phép font Inter/JetBrains Mono (OFL) phải ghi trong THIRD_PARTY_NOTICES.

## 9. Cost and operational implications

- Bundle font; chi phí thiết kế token một lần.
- Chi phí thay đổi sau khi component đã dùng token thấp nếu giữ semantic layer.
- Dịch EN/VI tốn công nội dung.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Spacing 4px và tính WCAG của text-tertiary | `docs/design/DESIGN_TOKENS.md §3.4, §4 T4` | Giả định |
| Cơ chế Tailwind v4 `@theme` static | `docs/design/DESIGN_TOKENS.md §4 T2` | Giả định |
| Màu primitives và chọn font là đề xuất | `docs/design/DESIGN_TOKENS.md §3.2, §3.3` | Giả định |
| Lệch màu blocked giữa SCREEN_SPEC và token | `docs/VIRTUAL_OFFICE.md §21 mục 7` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Mâu thuẫn tài liệu xác minh được; phần lớn token và contrast do tác giả tính, chưa được QA đo lại.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chọn accent tím và font Inter (hoặc Geist), locale mặc định EN hay VI.
- Xác nhận mật độ Comfortable.
- **Khả năng đảo ngược:** Dễ nếu mọi màu đi qua token; thay đổi lan rộng nếu hard-code.
- **Bị chặn trong lúc chờ duyệt:** Component library, theme CSS, bản dịch.

## 12. Reversal conditions

- Kiểm thử người dùng cho thấy tím gây nhầm với trạng thái: đổi accent.
- Chủ sở hữu chọn Geist hoặc locale VI mặc định.
- Brand guideline tổ chức yêu cầu bảng màu khác.
