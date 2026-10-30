# Architecture Decision Records (ADR) — AI WORKSPACE

Ngày: 2026-10-30. Phase 0, chỉ tài liệu.

Mỗi ADR chuyển một dòng của `DECISION_LOG.md` (D-001..D-020) thành bản ghi có ngữ cảnh, phương án, hệ quả, bảo mật, chi phí, bằng chứng và điều kiện đảo ngược. Số ADR trùng số D. **Cập nhật OWR-2 (2026-10-30):** 20 ADR đã có phản hồi của Owner (OWR-1, OWR-2, `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`): 16 APPROVED (ADR-0001, 0002, 0004, 0005 hoãn Codex khỏi MVP, 0006 hoãn OpenHands, 0008, 0009, 0010, 0012, 0013, 0014, 0015, 0016, 0017, 0018, 0020; riêng 0014 còn PENDING_OWNER_CLARIFICATION về danh sách event) và 4 APPROVED_CONDITIONAL (ADR-0003, 0007, 0011, 0019). D-021 chưa có ADR riêng (APPROVED_CONDITIONAL trong register). Không ADR nào VERIFIED. Mỗi ADR có mục "13. Cập nhật OWR-2" ở cuối. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

Nội dung gốc "PENDING_OWNER_APPROVAL" của các ADR nằm ở các mục 1 đến 12 được giữ làm lịch sử; mục 13 và cột Status dưới đây là hiện hành. Khi Owner đổi quyết định, cập nhật Metadata của ADR và cột Phê duyệt của `DECISION_LOG.md`.

Cấu trúc bắt buộc của mọi ADR: Metadata, Context, Problem, Options considered, Decision status and what is being proposed, Rationale, Consequences, Security implications, Cost and operational implications, Validation evidence, Owner approval requirement, Reversal conditions (xem [TEMPLATE.md](TEMPLATE.md)).

## Cách tính mức bằng chứng

- **Verified:** các claim cốt lõi của quyết định đã được đọc mã/chạy lệnh thật và truy vết được; không còn claim cốt lõi nào chỉ là giả định.
- **Partial:** một phần claim cốt lõi đã xác minh (đọc mã, spot-check của QA, hoặc số đo do tác giả tài liệu báo cáo), phần còn lại là giả định hoặc chưa kiểm chứng.
- **Unverified:** claim cốt lõi dựa trên tài liệu nghiên cứu, web search hoặc ngoại suy, chưa prototype hay chạy thật.
- QA Phase 0 không chạy lại live test nào, nên **chưa ADR nào đạt Verified**. Mức hiện tại là mức trung thực, không phải mức mong muốn. Sau OWR-2 mức bằng chứng không đổi: chưa có PoC report (không giả định `docs/poc/` tồn tại); khi có báo cáo PoC thật mới được nâng mức. Status (quyết định của Owner) và Validation evidence (bằng chứng kiểm thử) là hai trục độc lập.

| ID | Tiêu đề | Decision ID | Status | Owner approval needed | Validation evidence |
|---|---|---|---|---|---|
| [ADR-0001](ADR-0001-monorepo-typescript-web-first.md) | Monorepo TypeScript, web app + backend Node cục bộ, Tauri 2 ở phase sau | D-001 | APPROVED (Owner, 2026-10-30; OWR-2 Q-16) | yes | Partial |
| [ADR-0002](ADR-0002-postgresql-redis-drizzle.md) | PostgreSQL 16 + Redis, ORM Drizzle, event store append-only phân vùng theo tháng | D-002 | APPROVED (Owner, 2026-10-30) | yes | Partial (PGlite, chưa PG16) |
| [ADR-0003](ADR-0003-conductor-workflow-engine.md) | Workflow engine tự xây (Conductor) trên BullMQ + Postgres, approval là WAITING_HUMAN | D-003 | APPROVED_CONDITIONAL (OWR-2: PoC Q-20) | yes | Unverified |
| [ADR-0004](ADR-0004-claude-code-agent-sdk.md) | Tích hợp Claude Code bằng @anthropic-ai/claude-agent-sdk, cổng phê duyệt canUseTool + PreToolUse | D-004 | APPROVED (OWR-2 Q-17) | yes | Partial |
| [ADR-0005](ADR-0005-codex-app-server.md) | Tích hợp Codex qua codex app-server (JSON-RPC stdio), dự phòng codex exec --json | D-005 | APPROVED, hoãn Codex khỏi MVP (OWR-2) | yes | Unverified |
| [ADR-0006](ADR-0006-openhands-agent-server-provider.md) | OpenHands là provider thứ ba qua openhands-agent-server, dùng cho Tier 2 | D-006 | APPROVED, hoãn sau Phase 4 (OWR-2) | yes | Partial |
| [ADR-0007](ADR-0007-two-tier-sandbox.md) | Mô hình sandbox hai tầng (Tier 1 host + hook, Tier 2 container hóa tùy chọn) | D-007 | APPROVED_CONDITIONAL (OWR-1, OWR-2 §8) | yes | Unverified |
| [ADR-0008](ADR-0008-credentials-managed-by-cli.md) | Credential do CLI tự quản, secret khác lưu OS keychain | D-008 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0009](ADR-0009-worktree-per-task-git-isolation.md) | Mỗi task một git worktree + branch, manifest quyền file, diff check là cổng cuối | D-009 | APPROVED (OWR-2) | yes | Unverified |
| [ADR-0010](ADR-0010-concurrency-default-5-max-18.md) | Concurrency mặc định 5, cấu hình tối đa 18, nâng dần theo bằng chứng | D-010 | APPROVED (Owner, 2026-10-30; Q-07 ToS còn mở) | yes | Partial |
| [ADR-0011](ADR-0011-phaser-virtual-office.md) | Virtual Office dùng Phaser 3.90, React ↔ Phaser qua Zustand + event bus | D-011 | APPROVED_CONDITIONAL (OWR-2: benchmark) | yes | Unverified |
| [ADR-0012](ADR-0012-license-and-reuse-policy.md) | Chính sách giấy phép và tái sử dụng (Apache-2.0/MIT có attribution, AGPL chỉ tham khảo) | D-012 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0013](ADR-0013-agent-skills-standard.md) | Skill theo chuẩn Agent Skills, mở rộng qua skill.yaml, Memory tách khỏi Skill | D-013 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0014](ADR-0014-event-system-outbox-redis.md) | Event system: outbox giao dịch → Redis pub/sub (Socket.IO) + Redis Stream cho Virtual Office | D-014 | APPROVED (OWR-2); PENDING_OWNER_CLARIFICATION danh sách event | yes | Unverified |
| [ADR-0015](ADR-0015-design-ai-accent-purple-inter.md) | Design: accent AI tím, Inter + JetBrains Mono, mật độ Comfortable, giao diện mặc định tiếng Anh (OWR-2: ĐỔI thành mặc định tiếng Việt, tiếng Anh là locale thứ hai) | D-015 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0016](ADR-0016-agent-roles-as-seed-data.md) | 18 vai trò agent mặc định là dữ liệu seed; tách Agent/Provider/User Identity | D-016 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0017](ADR-0017-attention-state-layer.md) | Mô hình 5 attention state làm lớp hiển thị trên 12 lifecycle state | D-017 | APPROVED (OWR-2) | yes | Unverified |
| [ADR-0018](ADR-0018-agent-rail-theme-tokens.md) | Agent Rail (sidebar hai vùng) và cấu trúc token theme kiểu herdr | D-018 | APPROVED (OWR-2) | yes | Unverified |
| [ADR-0019](ADR-0019-agent-driven-control-aiw-cli.md) | Điều khiển bởi agent: biến môi trường AIW_*, CLI aiw trả JSON, token API riêng theo agent | D-019 | APPROVED_CONDITIONAL (OWR-2: Security Review) | yes | Unverified |
| [ADR-0020](ADR-0020-herdr-design-reference-only.md) | herdr là tham chiếu thiết kế; HerdrRuntime chỉ là tùy chọn sau PoC | D-020 | APPROVED (OWR-2) | yes | Partial |
| [ADR-0021](ADR-0021-dynamic-skill-overlay.md) | Skill Overlay Directory: nạp skill Markdown động theo agent | D-021 | APPROVED_CONDITIONAL (OWR-2) | yes | Partial (script và CLI, chưa SDK) |

Tệp liên quan: [TEMPLATE.md](TEMPLATE.md) (mẫu ADR mới), [CONFLICTS_TO_RESOLVE.md](CONFLICTS_TO_RESOLVE.md) (mâu thuẫn giữa các tài liệu Phase 0 cần chủ sở hữu giải quyết).
