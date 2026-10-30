# ADR-0010 — Concurrency mặc định 5, cấu hình tối đa 18, nâng dần theo bằng chứng

## 1. Metadata

- **ID:** ADR-0010
- **Ngày:** 2026-10-30
- **Decision ID:** D-010 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-1; OWR-2 Q-07, Q-08)
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

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §2 Q-07, Q-08, §5.1, §8 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Giữ concurrency mặc định 5. Đây là **giới hạn mặc định của ứng dụng, KHÔNG phải xác nhận rằng subscription cho phép 5 session song song.** Không tự tăng lên 10 hoặc 18 khi chưa xác minh điều khoản provider. Xác minh ToS vẫn **mở** (PENDING_OWNER_CLARIFICATION).
- Q-08: không tự động failover provider khi hết quota; thông báo Owner và chờ quyết định.
- Không tuyên bố giới hạn concurrency cụ thể của subscription và không tuyên bố 18 Agent concurrent đã được kiểm thử (OWR-2 §5.1, §8). Cap Codex 3 trong ADR chỉ còn mang tính tham khảo vì Codex hoãn khỏi MVP (D-005).
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

## 14. Cập nhật OWR-3 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (OWR-3) (A.2, A.15, D). Các mục 1 đến 13 ở trên giữ làm lịch sử; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**. Việc đồng bộ tài liệu khác do agent khác thực hiện, xem `docs/DOC_SYNC_LOG_OWR3_A.md` và `docs/DOC_SYNC_LOG_OWR3_B.md`; ADR này không khẳng định đã đồng bộ.

- Decision status giữ **APPROVED**: mặc định 5. Không tăng lên 10 hoặc 18.
- A.2/D: 5 là **giới hạn scheduler nội bộ của ứng dụng, KHÔNG phải điều provider bảo đảm hay cho phép**. Owner nói đã kiểm tra điều khoản chính thức và không thấy căn cứ công khai đủ rõ cho 5, 10 hay 18 session song song (lời Owner, agent chưa kiểm chứng độc lập). Không chia sẻ credential hoặc seat giữa nhiều người; không tăng khi chưa xác minh điều khoản và usage thực tế. Bảng ngoại suy 5/10/18 trong ADR chỉ là tham khảo, chưa kiểm chứng.
- A.15 (quy tắc slot): `WAITING_APPROVAL` **không tính** vào 5 active workers nếu worker đã được giải phóng và trạng thái đã persist; nếu worker chưa được giải phóng (ví dụ approval mức tool giữ process) thì vẫn chiếm slot. Quota provider vẫn áp dụng. Xung đột CF-048 theo hướng này; đồng bộ do agent khác thực hiện.
- **Validation evidence (trung thực):** mức giữ nguyên (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
