# ADR-0006 — OpenHands là provider thứ ba qua openhands-agent-server, dùng cho Tier 2

## 1. Metadata

- **ID:** ADR-0006
- **Ngày:** 2026-10-30
- **Decision ID:** D-006 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED (hoãn đến sau Phase 4)
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** research/OPENHANDS_ANALYSIS.md, PROVIDER_INTEGRATION.md §9, §25

## 2. Context

- Cần một provider chạy hoàn toàn trong container (Tier 2) và có agent native không phụ thuộc đăng nhập CLI.
- OpenHands (MIT) có agent-server và client TypeScript nhưng là Python; sub-mode ACP (Claude Code/Codex chạy bên trong OpenHands, materialize `auth.json`) mâu thuẫn D-008 (CF-024).

## 3. Problem

- Chọn cách tích hợp OpenHands mà không phá ranh giới sandbox và nguyên tắc credential.
- Docker daemon chưa chạy lúc khảo sát nên chưa thử khởi chạy agent-server.

## 4. Options considered

### Phương án A — Service ngoài `openhands-agent-server` + `@openhands/typescript-client` [ĐỀ XUẤT]

Chạy qua `uvx` hoặc `docker run ghcr.io/openhands/agent-server:<ver>-python`; backend nói chuyện REST/WS; MVP chỉ agent native + LLM key trong OS keychain, ACP tắt; không vendor mã.
- **Lý do chọn đề xuất:** Tách process/container, dùng lại contract MIT có attribution, cùng giao thức event qua `AgentAdapter`.

### Phương án B — Nhúng Python vào backend NestJS [Bị loại]

Kéo runtime Python/uv vào tiến trình backend.
- **Lý do loại:** Phá ranh giới sandbox, tăng bề mặt tấn công, vendoring nặng.

### Phương án C — Bỏ OpenHands [Bị loại]

Tier 2 tự xây runtime container.
- **Lý do loại:** Công sức cao hơn nhiều; là phương án đối chiếu, không nằm trong DECISION_LOG.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- OpenHands là provider thứ ba chỉ cho Tier 2, chạy như service ngoài; không vendor mã.
- ACP sub-mode tắt ở MVP; UI OpenHands chỉ là tham khảo (gắn HeroUI/PostHog/Cloud).
- Ghim phiên bản image/package; LLM key riêng lưu OS keychain.

## 6. Rationale

- Giữ chung contract adapter, không phá kiến trúc.
- License MIT cho phép dùng lại có attribution (ADR-0012).
- Docker là điều kiện sẵn có của Tier 2 (ADR-0007).

## 7. Consequences

**Tích cực**
- Có đường Tier 2 sẵn runtime.
- Không thêm phụ thuộc Python vào backend.

**Tiêu cực**
- Thêm một service ngoài cần quản vòng đời và ghim phiên bản.
- Cần LLM API key riêng (đụng FR-098/OI-06).

**Trung tính**
- Nếu Tier 2 không được dùng, bỏ provider này.
- Protocol agent-server có thể đổi giữa các phiên bản.

## 8. Security implications

- Agent-server khi chạy bằng `uvx` mặc định không sandbox (full access filesystem): chỉ cho phép dạng container.
- Cùng origin giữa openvscode-server và app có thể lộ localStorage (SELF_HOSTING).
- ACP tắt để không materialize `auth.json` vào container (ADR-0008).

## 9. Cost and operational implications

- Docker/WSL2 là phụ thuộc; image lớn, tốn RAM/đĩa chưa đo.
- LLM key trả theo token (khác quota subscription): chi phí biến đổi.
- Theo dõi cập nhật phiên bản agent-server.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Đã đọc mã/tài liệu repo OpenHands (agent-canvas v1.34.0) | `docs/research/OPENHANDS_ANALYSIS.md §5, §7` | Đã xác minh |
| License MIT từ file LICENSE | `docs/research/LICENSE_AND_REUSE.md §1.5` | Đã xác minh |
| Package `@openhands/typescript-client` 1.59.0 tồn tại | `docs/QA_REVIEW_PHASE0.md (bảng npm view)` | Đã xác minh |
| Mâu thuẫn ACP vs D-008 | `docs/PROVIDER_INTEGRATION.md §9, §25` | Đã xác minh |
| Khởi chạy agent-server thành công trong Docker | `docs/PROVIDER_INTEGRATION.md §22` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Mã nguồn, license và package được xác minh bằng đọc; chưa khởi chạy agent-server thật.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận OpenHands là provider thứ ba chỉ trong Tier 2 và ACP tắt ở MVP.
- Chấp nhận cần LLM API key riêng và chi phí tương ứng.
- **Khả năng đảo ngược:** Dễ: provider tùy chọn sau interface; bỏ không ảnh hưởng lõi.
- **Bị chặn trong lúc chờ duyệt:** Provider OpenHands, mọi test Tier 2 phụ thuộc nó.

## 12. Reversal conditions

- Chủ sở hữu duyệt ACP sub-mode: thiết kế lại credential handling và cập nhật ADR-0008.
- Agent-server không ổn định hoặc license/đóng gói đổi: tự xây runtime container tối giản.
- Tier 2 không dùng ở phase nào: bỏ provider này.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-006, §2 Q-02 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- OpenHands bị hoãn đến **sau Phase 4**; không tích hợp như provider thứ ba trong MVP. Q-02: không tích hợp OpenHands ACP trong MVP.
- Hệ quả: Tier 2 của ADR-0007 chưa có runtime OpenHands trong MVP (register mục 11, X-9). Cần Owner biết khi lập kế hoạch Tier 2.
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
