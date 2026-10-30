# POC Drizzle (Phase 0, OWR-2 mục 5.4)

Ngày: 2026-10-30. Người thực hiện: Validation Engineer. Mức bằng chứng: **Partial: PGlite only** (chưa VERIFIED, chưa chạy PostgreSQL 16 thật).
Thư mục chạy PoC (scratch, ngoài repo): `C:\Users\TUYEN\AppData\Local\Temp\claude\d--Code-Tuyen-Agent-management\9dde48b7-c365-4f17-9f11-5331b00699a6\scratchpad\poc-drizzle`
Không sửa `docs/DATABASE_DESIGN.md` hay schema production. Không git add/commit/push.

## 1. Tóm tắt kết quả

| # | Hạng mục | Kết quả |
|---|---|---|
| 1 | Version facts (orm/kit/postgres.js) | PASS (driver postgres.js chưa chạy: NOT RUN) |
| 2 | PostgreSQL 16 thật | BLOCKED (Docker daemon tắt); dùng PGlite thay thế |
| 3a | `drizzle-kit generate` | PASS |
| 3b | `agent_events` partition tháng, PK gồm partition key, unique per-aggregate | PASS có giới hạn (mục 6) |
| 3c | Partial unique index `where deleted_at is null` | PASS |
| 3d | FK vòng departments <-> agents bằng custom SQL | PASS |
| 3e | Apply migration 2 lần (idempotent) | PASS |
| 3f | Drift: `generate` sau apply no-op | PASS |
| 3f' | Phát hiện sửa tay DB bằng `generate`/`check` | FAIL (không phát hiện); `pull` phát hiện (mục 7) |
| 3g | Trigger append-only `reject_mutation` | PASS |
| 3h | Insert/read qua ORM, jsonb, enum | PASS |

Tổng: 9 PASS (1, 3a-3e, 3f, 3g, 3h), 1 FAIL (3f' là hành vi công cụ, không phải lỗi schema), 1 BLOCKED (2), 0 NOT RUN trong phạm vi đã thử (postgres.js và PG16-specific ghi ở mục 10).

## 2. Phiên bản (npm registry, 2026-10-30)

```
npm view drizzle-orm dist-tags        -> latest: 0.45.4 (kèm nhiều tag 1.0.0-beta/rc, rc: 1.0.0-rc.5)
npm view drizzle-orm@0.45.4 version   -> 0.45.4   (TỒN TẠI)
npm view drizzle-kit dist-tags        -> latest: 0.31.11 (rc: 1.0.0-rc.5)
npm view drizzle-orm@0.45.4 peerDependencies.postgres -> >=3
npm view postgres dist-tags           -> latest: 4.0.0, beta: 3.0.0-rc.3
npm view @electric-sql/pglite dist-tags -> latest: 0.5.13
```

- drizzle-orm 0.45.4 tồn tại và là `latest` ổn định. drizzle-kit `latest` là 0.31.11 (kit không khai báo peer orm); cặp 0.45.4 + 0.31.11 chạy cùng nhau thành công trong PoC => tương thích thực nghiệm.
- Đã pin chính xác (`npm install --ignore-scripts --save-exact`, chỉ trong scratch): `drizzle-orm@0.45.4`, `drizzle-kit@0.31.11`, `@electric-sql/pglite@0.5.13`, `tsx@4.23.15` (chạy TS).
- postgres.js: orm 0.45.4 yêu cầu peer `postgres >=3`; mới nhất 4.0.0. Không cài, không chạy (không có PG thật): NOT RUN.
- Nhánh 1.0.0-beta/rc của orm/kit: không dùng, không đánh giá.

## 3. Cơ sở dữ liệu

```
docker info   -> Client 29.8.1; Server: failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine
where psql    -> INFO: Could not find files for the given pattern(s).
Get-Service *postgres* -> Running  postgresql-x64-17 (PostgreSQL 17 Server)
```

- Docker: BLOCKED (daemon tắt, không bật theo yêu cầu). Không có `psql` trong PATH.
- Có service PostgreSQL **17** cục bộ đang chạy, nhưng không phải PG16 và truy cập cần credentials (cấm dùng). Không kết nối, không dùng.
- Engine thay thế: `@electric-sql/pglite@0.5.13`, lưu đĩa (`./pgdata`) để 2 lần chạy là 2 tiến trình riêng.

```
select version() -> PostgreSQL 18.6 (PGlite 0.5.13) on wasm32-unknown-emscripten ... 32-bit
```

**PGlite 0.5.13 báo PostgreSQL 18.6, không phải 16.** Mọi kết quả chỉ áp dụng cho bản build Postgres của PGlite (WASM, 32-bit, một kết nối). Hành vi riêng PG16: NOT RUN. (Có thể thử PGlite bản cũ hơn nếu build PG16; chưa làm.)

## 4. Cấu trúc PoC

- `src/schema/core.ts`: pgEnum `agent_status`, `departments` (`head_agent_id uuid` không FK), `agents` (FK `department_id`), partial unique index `.where(sql\`deleted_at is null\`)` (theo DATABASE_DESIGN.md).
- `src/schema/events.ts`: khai báo `agentEvents` (PK `(id, occurred_at)`, unique gồm `occurred_at`, jsonb, bigserial) chỉ để ORM có kiểu; không nằm trong `schema` của drizzle-kit (kit sẽ sinh bảng không partition).
- `drizzle.config.ts`: `dialect: postgresql`, `driver: pglite`, `schema: ./src/schema/core.ts`, `out: ./drizzle`.
- `scripts/migrate.mts` (dùng `drizzle-orm/pglite/migrator`), `scripts/verify.mts`, `scripts/drift.mts`.

## 5. Migration (3a, 3b, 3c, 3d, 3e)

```
npx drizzle-kit generate --name=core
  2 tables / agents 10 columns 2 indexes 1 fks / departments 8 columns 1 indexes 0 fks
  [OK] drizzle\0000_core.sql
npx drizzle-kit generate --custom --name=events_partitioned   -> drizzle\0001_events_partitioned.sql (rỗng, điền tay)
npx drizzle-kit generate --custom --name=circular_fk          -> drizzle\0002_circular_fk.sql
```

Partial index do kit sinh (0000):
```
CREATE UNIQUE INDEX "agents_org_slug_uq" ON "agents" USING btree ("organization_id","slug") WHERE "agents"."deleted_at" is null;
```
0001 (viết tay): `CREATE TABLE agent_events (...) PARTITION BY RANGE ("occurred_at")`, `CONSTRAINT agent_events_pk PRIMARY KEY (id, occurred_at)`, 2 partition (2026_10, 2026_11), unique index per-aggregate (gồm `occurred_at`), hàm `reject_mutation()` và trigger `trg_agent_events_ro`. Cần `--> statement-breakpoint` giữa các câu lệnh.
0002 (viết tay): `ALTER TABLE departments ADD CONSTRAINT departments_head_agent_fk FOREIGN KEY (head_agent_id) REFERENCES agents(id) ON DELETE SET NULL`.

Apply 2 lần (3e):
```
rm -rf pgdata; npx tsx scripts/migrate.mts
  applied rows before=0 after=3 (newly applied=3)
npx tsx scripts/migrate.mts
  applied rows before=3 after=3 (newly applied=0)      # no-op => PASS
```

Catalog (`scripts/verify.mts`):
```
agent_events relkind=p pk=RANGE (occurred_at); agent_events_2026_10 / _11 relkind=r
agents_org_slug_uq      pred=(deleted_at IS NULL) indisunique=true
departments_org_slug_uq pred=(deleted_at IS NULL) indisunique=true
agents_department_id_departments_id_fk: agents(department_id) -> departments(id) ON DELETE SET NULL
departments_head_agent_fk: departments(head_agent_id) -> agents(id) ON DELETE SET NULL
```
Hành vi: slug trùng khi chưa xoá mềm => `duplicate key value violates unique constraint "agents_org_slug_uq"`; sau khi set `deleted_at` thì chèn lại slug được. `head_agent_id` sai => `violates foreign key constraint "departments_head_agent_fk"`.

## 6. Partition và unique per-aggregate (3b)

Thử với cùng `(aggregate_type, aggregate_id, sequence=1)`:
```
OK   cùng aggregate+seq, cùng tháng nhưng occurred_at khác (10-15 vs 10-20) -> INSERT thành công
OK   cùng aggregate+seq, tháng khác (11-05)                                  -> INSERT thành công
ERR  create unique index (aggregate_type, aggregate_id, sequence)            -> unique constraint on partitioned table must include all partitioning columns
ERR  insert occurred_at 2027-03-01 (chưa có partition)                       -> no partition of relation "agent_events" found for row
```
Kết luận:
- PK `(id, occurred_at)` và unique index chứa partition key hoạt động.
- Giới hạn: PostgreSQL bắt buộc mọi unique/PK trên bảng partition chứa cột partition. `(aggregate_type, aggregate_id, sequence)` **không** thể được DB đảm bảo duy nhất toàn cục; index có `occurred_at` chỉ chặn trùng khi `occurred_at` bằng nhau. Tính duy nhất của sequence per-aggregate phải được bảo đảm ở tầng ứng dụng (một writer/aggregate, `SELECT ... FOR UPDATE` hoặc advisory lock, hoặc bảng head riêng). Đây là điểm thiết kế để Owner/Architect xác nhận, không phải lỗi Drizzle.
- Không có partition DEFAULT: ghi ngoài khoảng đã tạo sẽ lỗi => cần job tạo partition trước (hoặc DEFAULT partition).
- Drizzle không mô tả được PARTITION BY trong schema; bảng phải khai báo kép (SQL tay + TS typing). Đưa `agentEvents` vào schema của kit sẽ sinh `CREATE TABLE` không partition (không thử).

## 7. Drift (3f, 3f')

```
npx drizzle-kit generate   (sau apply) -> "No schema changes, nothing to migrate"
npx drizzle-kit check                  -> "Everything's fine"
```
Sửa tay DB (`scripts/drift.mts`): `alter table agents add column rogue_col text; drop index agents_department_idx; create index rogue_idx on departments(name);`
```
npx drizzle-kit generate -> "No schema changes, nothing to migrate"    # KHÔNG phát hiện
npx drizzle-kit check    -> "Everything's fine"                        # KHÔNG phát hiện
npx drizzle-kit pull --config=pull.config.ts   (introspect vào ./pulled)
  schema.ts có rogueCol: text("rogue_col"); index("rogue_idx"); vắng "agents_department_idx"   # PHÁT HIỆN
```
Kết luận: `generate`/`check` chỉ so schema TS với snapshot JSON, không đọc DB nên không thấy drift DB. Phải dùng `pull` (diff thủ công) hoặc script CI so catalog. Ghi chú: `pull` coi partition con là bảng thường (`agent_events_2026_10/11`) và không xuất bảng cha partition `agent_events`; `pull --config` không dùng chung với `--out` (phải có config riêng).

## 8. Trigger append-only (3g)

```
ERR  UPDATE -> table agent_events_2026_10 is append-only
ERR  DELETE -> table agent_events_2026_10 is append-only
OK   INSERT vẫn thành công
```
`pg_trigger`: trigger nhân bản xuống từng partition (cha + 2 con). Thông báo hiển thị tên partition con (TG_TABLE_NAME), không phải tên bảng cha.

## 9. ORM (3h)

- Insert/select qua `drizzle-orm/pglite`: departments, agents (enum `status: 'working'`), cập nhật `head_agent_id` sau khi có agent (vòng FK), đọc lại đúng.
- Enum sai: `invalid input value for enum agent_status: "bogus"`.
- jsonb: ghi `{ nested: { n: [1,2], s: 'vi: tiếng Việt' } }`, đọc lại object; `payload->'nested'->>'s'` và `jsonb_array_length` đúng, Unicode giữ nguyên.
- `bigserial` mode number (`globalSeq: 1`), timestamptz trả về `Date`.
- Pruning (tham khảo): `explain` với `occurred_at` trong tháng 11 chỉ quét `agent_events_2026_11`.

## 10. Chưa kiểm chứng trên PostgreSQL 16 thật

- Toàn bộ kết quả trên PGlite 0.5.13 (PostgreSQL 18.6, WASM, 32-bit, 1 kết nối). Hành vi riêng PG16: NOT RUN.
- Partition pruning với dữ liệu lớn, plan thực tế, BRIN trên `occurred_at`: NOT RUN (chỉ 1 explain nhỏ).
- Extension (pgcrypto, pg_trgm, pgvector...): `gen_random_uuid()` có sẵn core; còn lại NOT RUN.
- Hiệu năng, đồng thời (nhiều kết nối, lock, `FOR UPDATE` cho sequence), `ALTER TYPE ... ADD VALUE` trong transaction: NOT RUN.
- Driver postgres.js (3.x/4.0.0) với orm 0.45.4 và `drizzle-orm/postgres-js/migrator`, `drizzle-kit migrate` (CLI): NOT RUN (dùng migrator ORM trên PGlite).
- Mới dựng 2 bảng core + `agent_events`; `usage_records`, `audit_logs` và trigger của chúng chưa thử.

## 11. Khuyến nghị tiếp theo (khi có Docker)

1. Chạy lại đúng PoC trên `postgres:16` (docker compose cục bộ) với `drizzle-kit migrate` và driver `postgres` thật; so sánh với báo cáo này.
2. Bổ sung: partition DEFAULT + job tạo partition (pg_partman hoặc script), `EXPLAIN` với >= 1M dòng, BRIN, nhiều writer đồng thời cho sequence per-aggregate.
3. Chốt cơ chế đảm bảo sequence per-aggregate ở tầng ứng dụng (mục 6) và ghi vào ADR.
4. Thêm kiểm tra drift vào CI (so `pg_catalog`/`pull` với snapshot) vì `generate`/`check` không đọc DB.
5. Quyết định ORM thuộc Owner; PoC này không có bằng chứng đủ mạnh để khuyến nghị đổi ORM. Các hạn chế (partition khai báo kép, không phát hiện drift DB) đã nằm trong lường trước của DATABASE_DESIGN.md và có cách xử lý.
