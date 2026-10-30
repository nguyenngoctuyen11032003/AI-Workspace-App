# ADR-0010 — Concurrency mặc định 5, cấu hình tối đa 18, nâng dần theo bằng chứng

## 1. Metadata

- **ID:** ADR-0010
- **Ngày:** 2026-10-30
- **Decision ID:** D-010 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED (Q-07 điều khoản subscription còn mở)
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (tin nhắn trong phiên; xem PHASE_0_OWNER_DECISION_REGISTER.md mục 8)
- **Nguồn:** architecture/RESOURCE_MANAGEMENT.md, research/MULTI_AGENT_SCALABILITY.md

## 2. Context

- Chủ sở hữu yêu cầu tối đa 18 agent đồng thời; giới hạn thực tế là quota subscription dùng chung, sau đó RAM/CPU, không phải số process.
- Nhiều tài liệu đặt mặc định khác nhau (10, 5, 2–3, 18), gây lệch cấu hình (CF-002).

## 3. Problem

- Chọn mặc định và trần concurrency mà bằng chứng hiện có ủng hộ.
- Chưa có số đo task thật (build/test, context 100k+); ToS nhiều phiên song song chưa xác minh (R-TOS-QUOTA).

## 4. Options considered

### Phương án A — Mặc định 5, trần 18, nâng 5 → 10 → 18 khi có bằng chứng [ĐỀ XUẤT]

`concurrency worker = min(resource_limits.max_concurrent_agents, 18)`; semaphore Redis Lua global + per-provider; stagger spawn; cap Claude 5, Codex 3.
- **Lý do chọn đề xuất:** Số đo thực chỉ có ở 1–3 process; nâng dần theo test plan MULTI_AGENT_SCALABILITY §9.

### Phương án B — Mặc định 18 [Bị loại]

Đáp ứng yêu cầu trần ngay.
- **Lý do loại:** Bão hòa CPU lúc khởi động (~12 thread, ~6–10 s theo ngoại suy) và vượt khả năng quota chưa rõ.

### Phương án C — Mặc định 10 [Bị loại]

Theo research Claude/ORCHESTRATION.
- **Lý do loại:** Chưa có số đo ở mức 10; không bằng chứng quota chịu được.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Khởi đầu 5; `provider_configs.max_concurrent = 2` ở DB chỉ là giá trị khởi tạo.
- Chỉ nâng sau khi chạy test plan §9 bằng session thật, có chủ sở hữu xác nhận từng bậc.
- Đồng bộ ORCHESTRATION (global 10), DATABASE_DESIGN (default 2), CLAUDE research (10) về một cấu hình duy nhất.

## 6. Rationale

- Tác giả đo: 1 process 5 975 ms, ~235 MB RSS; 3 process đồng thời tổng 8 172 ms, đỉnh 685,6 MB, không rate-limit (QA không tái hiện).
- Khởi động tốn ~3,5–3,9 s CPU/process nên spawn đồng loạt 18 sẽ nghẽn.
- Chi phí cố định ~48k token/spawn làm quota là nút thắt đầu tiên.

## 7. Consequences

**Tích cực**
- Bảo vệ máy và quota.
- Cấu hình được tới 18.

**Tiêu cực**
- Người dùng muốn 18 phải qua các bậc nâng.
- Phải đồng bộ lại nhiều tài liệu.

**Trung tính**
- Semaphore Redis phải rebuild được từ Postgres (ledger).
- RESOURCE §4.1 coi `WAITING_APPROVAL` không chiếm slot nhưng approval mức tool giữ process sống.

## 8. Security implications

- Giới hạn concurrency là kiểm soát chống lạm dụng 18 agent (THREAT_MODEL §3.5) và chống vượt quota gây khóa tài khoản.
- Cap per-provider tránh một provider chiếm toàn bộ slot.
- Cấu hình trần không được sửa bởi agent.

## 9. Cost and operational implications

- RAM ước tính ~1,2 GB ở 5 agent (5 × 235 MB), ngoại suy ~4,2 GB ở 18 (ước tính, chưa đo).
- Quota subscription (cửa sổ 5 giờ/tuần) là chi phí thật; chuyển API key thì chi phí theo token.
- Chi phí vận hành: theo dõi ledger và hiệu chỉnh ngưỡng.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Đo 1 và 3 process đồng thời (Acer Nitro i5-11400H, 15,77 GB, Claude Code 2.1.312) | `docs/architecture/RESOURCE_MANAGEMENT.md §3` | Chưa kiểm chứng |
| Số đo 1 và 3 process trong báo cáo scalability | `docs/research/MULTI_AGENT_SCALABILITY.md §4` | Chưa kiểm chứng |
| Bảng ngoại suy 5/10/18 | `docs/architecture/RESOURCE_MANAGEMENT.md §11` | Giả định |
| Điều khoản nhiều phiên song song | `docs/PROJECT_RISKS.md R-TOS-QUOTA` | Chưa kiểm chứng |
| Mặc định lệch giữa tài liệu (10, 5, 2–3) | `docs/adr/CONFLICTS_TO_RESOLVE.md CF-002` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Có số đo ở 1 và 3 process do tác giả báo cáo (QA không tái hiện); từ 5 trở lên là ngoại suy.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận mặc định 5, trần 18 và quy trình nâng bậc.
- Cho phép chạy test plan §9 với session thật (tốn quota).
- **Khả năng đảo ngược:** Dễ: chỉ là cấu hình; hạ hoặc nâng bất cứ lúc nào.
- **Bị chặn trong lúc chờ duyệt:** Chốt cấu hình mặc định trong DATABASE_DESIGN/ORCHESTRATION; scheduler concurrency.

## 12. Reversal conditions

- Test plan §9 chứng minh 10 hoặc 18 session ổn định và quota đủ: nâng mặc định.
- Chủ sở hữu xác nhận gói có ngưỡng thấp hơn: hạ trần.
- Chuyển API key trả theo token khiến quota không còn là nút thắt.
