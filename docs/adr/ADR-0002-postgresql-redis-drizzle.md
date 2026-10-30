# ADR-0002 — PostgreSQL 16 + Redis, ORM Drizzle, event store append-only phân vùng theo tháng

## 1. Metadata

- **ID:** ADR-0002
- **Ngày:** 2026-10-30
- **Decision ID:** D-002 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-1; OWR-2 không đổi trạng thái)
- **Nguồn:** DATABASE_DESIGN.md §1, research/TECHNOLOGY_EVALUATION.md TE-01, QA_REVIEW_PHASE0.md (P1-08)

## 2. Context

- Schema gồm 44 bảng, trong đó có event store append-only phân vùng theo tháng, partial unique index cho soft delete, trigger chống UPDATE/DELETE và transaction `appendEvent + update aggregate + outbox`.
- ORCHESTRATION_WORKFLOW_RESEARCH §3.2 viết schema bằng Prisma, xung đột với DATABASE_DESIGN (ARCHITECTURE_COMPARISON C-1).

## 3. Problem

- Chọn database và ORM sao cho biểu diễn được partition, trigger, CHECK và FK vòng bằng migration đọc được.
- QA (P1-08) phát hiện nhãn «Drizzle v1.x» tại DATABASE_DESIGN.md:29 không khớp npm: `drizzle-orm` bản ổn định `latest` là 0.45.4, còn v1.0 mới chỉ có tag beta (QA đo qua `npm view`; cần xác minh lại trước khi pin).

## 4. Options considered

### Phương án A — Drizzle ORM + drizzle-kit + postgres.js [ĐỀ XUẤT]

SQL thuần khi generate migration; hỗ trợ partial index, `jsonb().$type<T>()`, savepoint; không binary engine. Cần tự viết `DrizzleModule` (~30 dòng) cho NestJS.
- **Lý do chọn đề xuất:** Migration là SQL đọc/sửa được, hợp trigger `reject_mutation` và partition (qua `drizzle-kit generate --custom`), không binary engine.

### Phương án B — Prisma 6/7 [Bị loại]

Hệ sinh thái chín, `nestjs-prisma`, Prisma Studio.
- **Lý do loại:** Không mô tả bảng partition, trigger, CHECK trong schema nên phải migration tay và dễ drift; transaction timeout mặc định 5 s.

### Phương án C — TypeORM [Bị loại]

Quen với NestJS, có package chính thức.
- **Lý do loại:** Không tài liệu nào đánh giá; hỗ trợ partition/partial index hạn chế; chỉ là lựa chọn mở trong ENVIRONMENT_SETUP.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- PostgreSQL 16 + Redis; ORM Drizzle với driver postgres.js.
- Chính sách phiên bản (đề xuất): ghi đúng phiên bản thực tế (hiện QA thấy 0.45.x ổn định, 1.0 beta). Phải chạy `npm view drizzle-orm dist-tags` lại trước khi pin; pin chính xác (không caret) cả `drizzle-orm` và `drizzle-kit`, nâng cấp qua PR riêng. Dùng 1.0 beta chỉ khi spike xác nhận cần và chủ sở hữu chấp nhận rủi ro beta.
- **Bắt buộc spike trước Phase 1:** (a) bảng `agent_events` partition theo tháng + unique index chứa `seq`/khóa partition; (b) FK vòng giữa bảng bằng custom SQL (`drizzle-kit generate --custom`); (c) migrate hai lần để kiểm drift; (d) trigger `reject_mutation`.
- Sửa nhãn «Drizzle v1.x» trong DATABASE_DESIGN.md:29 theo kết quả (thuộc phạm vi tác nhân khác).
- Đường dẫn `prisma/**` trong seed AGENT_ORGANIZATION đổi thành `packages/db/**`.

## 6. Rationale

- Event store partition và partial index là yêu cầu cốt lõi; Prisma không biểu diễn được trong schema.
- Migration SQL đọc được, hợp trigger audit.
- Không binary engine giảm rủi ro đóng gói sidecar (ADR-0001).

## 7. Consequences

**Tích cực**
- Migration minh bạch, kiểm soát được partition/trigger.
- Không binary engine.

**Tiêu cực**
- Đội tự duy trì `DrizzleModule` và quy ước migration.
- Relational query API kém chín; một số truy vấn phải viết SQL.
- Nếu phải pin bản 0.45.x hoặc 1.0 beta, có nguy cơ breaking change khi nâng.

**Trung tính**
- Mã mẫu `prisma.$transaction` trong tài liệu orchestration phải dịch lại.
- Redis chỉ là cache/queue/pub-sub, không là nguồn sự thật.

## 8. Security implications

- Trigger append-only và quyền DB tách (role app không có DELETE trên bảng audit) là lớp bảo vệ audit; chưa kiểm chứng trên PG 16 với bảng partition.
- Truy vấn tham số hóa của ORM giảm SQL injection; SQL tùy chỉnh trong migration phải được review.
- Không lưu secret trong DB (ADR-0008); backup DB không chứa token OAuth.

## 9. Cost and operational implications

- PostgreSQL và Redis chạy cục bộ (Docker hoặc cài native): thêm 2 service cho dev; chưa đo RAM.
- Cần job tạo partition hằng tháng và chính sách retention (EVENT_SYSTEM).
- Spike ước tính 1 ngày; chi phí đổi ORM sau này cao.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Mâu thuẫn Prisma vs Drizzle giữa các tài liệu | `docs/research/ARCHITECTURE_COMPARISON.md §6.3 C-1` | Đã xác minh |
| `drizzle-orm` latest ổn định = 0.45.4, v1.0 chỉ beta (cần xác minh lại trước khi pin) | `docs/QA_REVIEW_PHASE0.md P1-08` | Đã xác minh |
| Drizzle xử lý được partition + partial unique index + FK vòng bằng custom SQL | `docs/DATABASE_DESIGN.md §1.1–1.2` | Chưa kiểm chứng |
| Unique index partition với `seq` chạy đúng trên PG 16 | `docs/security/AUDIT_LOGGING.md §15` | Chưa kiểm chứng |
| Độ tin cậy Trung bình; phát biểu Drizzle v1.x/Prisma 7 từ web search | `docs/research/TECHNOLOGY_EVALUATION.md TE-01` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Chỉ phiên bản npm và mâu thuẫn tài liệu được xác minh; năng lực partition/FK vòng của Drizzle chưa prototype.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận Drizzle (không Prisma) và chính sách pin phiên bản ở trên, kể cả khả năng dùng 0.45.x thay vì «v1.x».
- Chấp nhận spike partition + FK vòng là điều kiện cổng trước khi freeze schema.
- **Khả năng đảo ngược:** Rẻ trước khi viết schema/migration; rất đắt sau khi có dữ liệu và 44 bảng.
- **Bị chặn trong lúc chờ duyệt:** Viết schema Drizzle, migration và mọi mã truy cập DB; freeze DATABASE_DESIGN.

## 12. Reversal conditions

- Spike phát hiện drift không sửa được: đánh giá lại Prisma + SQL thô.
- Chủ sở hữu duyệt Prisma vì lý do kỹ năng đội: dịch lại 44 bảng.
- Drizzle ngừng bảo trì hoặc phá vỡ API lớn; hoặc 1.0 beta không ổn định mà 0.45.x không đủ tính năng.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §5.4, §2 Q-06, Q-12, Q-13 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Trạng thái không đổi. OWR-2 §5.4 cho phép Drizzle PoC **trong thư mục scratch**, không ảnh hưởng production schema: kiểm tra Drizzle stable (ưu tiên pin `0.45.4` nếu xác minh tồn tại và tương thích), partition `agent_events`, partial unique index, circular foreign keys, `drizzle-kit generate --custom`, chạy migration hai lần, kiểm tra schema drift và idempotency. PoC **chưa chạy**; không có kết quả.
- Nếu PoC thất bại: báo cáo và đưa lựa chọn cho Owner; **không tự đổi ORM**.
- Q-13: `task_status` UPPERCASE, enum khác lowercase. Q-12: `agent_events.type` dạng dotted lowercase. Q-06: audit online 365 ngày, signed export 2 năm (ảnh hưởng retention/partition của audit).
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

## Cập nhật PoC (2026-10-30, OWR-2 mục 5.4)
Kết quả `docs/poc/POC_DRIZZLE.md`: 9 PASS, 1 FAIL, 1 BLOCKED. Engine là PGlite 0.5.13 (báo PostgreSQL 18.6), **không phải PostgreSQL 16**, vì Docker daemon tắt; một PostgreSQL 17 cục bộ đang chạy nhưng cần thông tin đăng nhập nên không dùng. Đã ghim `drizzle-orm@0.45.4` (tồn tại, là bản stable `latest`), `drizzle-kit@0.31.11`.
- PASS: migration, bảng phân vùng, partial unique index, FK vòng qua custom SQL, chạy migration hai lần, trigger append-only, đọc ghi qua ORM.
- **Giới hạn:** unique index trên bảng phân vùng phải chứa khóa phân vùng, nên `(aggregate_type, aggregate_id, sequence)` không thể unique ở mức DB; ứng dụng phải tự bảo đảm.
- **FAIL:** `drizzle-kit generate` và `check` không phát hiện thay đổi thủ công trong DB; `pull` thì có. Đây là hành vi công cụ, không phải lỗi schema; cần quy trình kiểm drift dùng `pull`.
- BLOCKED: PostgreSQL 16 thật, postgres.js driver (chưa cài), hành vi pruning.
Mức bằng chứng nâng từ Unverified lên **Partial (PGlite)**. Không VERIFIED. Quyết định ORM không đổi.
