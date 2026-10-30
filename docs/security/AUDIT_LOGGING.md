# AUDIT LOGGING

> Security Architect · Phase 0 (documentation only) · 2026-10-30 · Trạng thái: PROPOSAL v1
> Mở rộng [`../SECURITY.md`](../SECURITY.md) §8, `docs/operations/DEVSECOPS_GUIDE.md` §9, `docs/DATABASE_DESIGN.md` bảng `audit_logs` (dòng 989–1008, trigger `reject_mutation` dòng 1172, quy ước dòng 1253–1256).
> Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Đặc tả audit log đủ để (a) tái dựng "ai/agent nào muốn gì, rule nào quyết định, ai duyệt, chạy ở đâu, kết quả ra sao", (b) phát hiện sửa/xoá log (tamper-evidence), (c) đáp ứng retention/redaction/export, (d) cảnh báo sự kiện nguy hiểm, (e) kiểm thử được. Hợp nhất ba nguồn đang dùng tên khác nhau: `audit_events` (SECURITY, DEVSECOPS) và `audit_logs` (DATABASE_DESIGN) — xem §4.1.

## 2. Phạm vi

- Trong: sự kiện bảo mật và quản trị (auth, approval, policy, tool call nhạy cảm, container, secret, config, git, emergency stop), schema, ghi trước/sau, hash chain, retention, redaction, export, alerting, test.
- Ngoài: telemetry vận hành (metrics OTel), transcript đầy đủ (lưu ở `agent_events`/`artifacts`, DEVSECOPS §9), log ứng dụng pino.
- Phân biệt: **audit** = bằng chứng pháp y, append-only, hash chain; **agent_events** = luồng sự kiện cho UI/Virtual Office (có thể rơi sự kiện, replay); **app log** = chẩn đoán. Không dùng chung bảng.

## 3. Danh mục sự kiện được ghi

Cột `cat` = `category`. Mức `sev` theo enum `audit_severity` hiện có (`info|warning|...` — danh sách giá trị cụ thể chưa có trong phần đã đọc; đề xuất `info|notice|warning|critical`).

| cat | action (ví dụ) | Khi nào | Ghi intent trước? | sev mặc định |
| --- | --- | --- | --- | --- |
| `auth` | `auth.login_check`, `auth.api_token_fail`, `auth.hook_token_fail`, `provider.needs_login`, `provider.reconnected` | mọi lần xác thực thất bại; đổi trạng thái provider | Không | warning khi fail |
| `approval` | `approval.requested`, `approval.decided`, `approval.expired`, `approval.revoked`, `approval.toctou_detected` | mọi bước vòng đời | Có (requested) | info; critical với toctou |
| `tool_call` | `tool.pre` (intent), `tool.post` (outcome) cho mỗi Bash/Edit/Write/WebFetch/MCP | **Chỉ** lệnh thuộc risk tag hoặc quyết định ≠ `allow` thường; lệnh `host_safe` được gom đếm (xem §5.4) | Có | info |
| `policy_deny` | `policy.deny` (rule_id, risk_tags) | mọi deny | — | warning (critical với `credential_access`, `self_modify`) |
| `rewrite` | `exec.rewrite` (command gốc hash → `aiw-exec`) | mọi rewrite | Có | info |
| `file_change` | `file.write_outside_manifest`, `diff.validate` (pass/fail), `diff.override` | validate diff; ngoại lệ | Có (validate) | notice/warning |
| `git` | `git.commit`, `git.merge`, `git.push`, `git.branch_delete`, `worktree.create/remove` | mọi thao tác orchestrator làm | Có (merge/push) | info/notice |
| `container` | `container.create`, `container.exec`, `container.kill`, `container.inspect_fail`, `egress.request` | vòng đời container + request proxy (tổng hợp domain/bytes) | Có (create/exec) | info; warning khi inspect fail |
| `secret_access` | `secret.use`, `secret.set`, `secret.rotate`, `secret.compromise`, `secret.expiry` | mỗi lần giải mã/ghi (không ghi value) | Có | notice; critical với compromise |
| `config_change` | `policy.change`, `settings.spawn_snapshot`, `manifest.create/widen`, `permission_level.change`, `provider.config` | thay đổi policy, level, manifest, `--settings` checksum | Có | warning |
| `emergency_stop` | `emergency.stop.intent`, `emergency.stop.outcome` | STOP ALL | Có | critical |
| `api_request` | `api.mutation` (do `AuditInterceptor`, DATABASE_DESIGN dòng 1256) | mọi mutation REST/WS của user | Không | info |
| `audit` | `audit.verify`, `audit.chain_fail`, `audit.spool_replay`, `audit.export`, `audit.purge` | tự quan sát | Có | critical với chain_fail |
| `agent` | `agent.spawn` (argv hash, env keys, settings sha256, hook verified), `agent.kill`, `agent.pause`, `agent.blocked` | vòng đời CLI | Có (spawn) | info |

Không ghi vào audit: nội dung prompt, nội dung file, stdout đầy đủ, giá trị secret (chỉ `sha256`, độ dài, loại). Hành động agent không nhạy cảm đã có trong `agent_events` (DATABASE_DESIGN dòng 1256); hành động nhạy cảm (git push, shell exec bị ask/deny/rewrite, approval, memory approve) ghi **cả hai** với cùng `correlation_id`.

## 4. Schema

### 4.1 Tên bảng và hợp nhất

Dùng **`audit_logs`** (đã định nghĩa, có trigger `reject_mutation`, partition theo tháng, migration `0006_usage_audit_partitioned`). Thuật ngữ `audit_events` trong SECURITY §8, DEVSECOPS §9, INFRA §8 được coi là bí danh cũ → các doc đó cần đổi tên (xem §15). [Đề xuất]

### 4.2 Cột hiện có (Đã xác minh — DATABASE_DESIGN dòng 989–1003)

`id uuid`, `organization_id`, `actor_user_id`, `actor_agent_id`, `action text`, `entity_type text`, `entity_id uuid`, `severity`, `before jsonb`, `after jsonb`, `ip_address inet`, `user_agent text`, `request_id text`, `created_at timestamptz`.

### 4.3 Cột bổ sung đề xuất (migration `0006b_audit_security_columns`)

| Cột | Kiểu | Mục đích | Ánh xạ SECURITY §8 |
| --- | --- | --- | --- |
| `seq` | `bigint not null` | số thứ tự trong chuỗi per-org (không gap) | `id bigserial` |
| `category` | `text not null` (CHECK theo §3) | phân loại | `category` |
| `phase` | `text` (`intent`\|`outcome`\|`single`) | ghi trước/sau | "intent/outcome" |
| `correlation_id` | `uuid` | nối intent↔outcome↔`agent_events`↔`approvals` | — |
| `session_id` | `uuid` | agent session | `session_id` |
| `task_id`, `project_id` | `uuid` | tra cứu timeline per task | `task_id`, `project_id` |
| `actor_kind` | `text` (`user`\|`agent`\|`system`) | actor | `actor{user\|agent\|system}` |
| `target` | `text` (đã redact, ≤ 2 KB) | path/command/branch | `target` |
| `input_hash` | `bytea(32)` | SHA-256 input chuẩn hoá | `input_hash` |
| `decision` | `text` (`allow`\|`deny`\|`ask_approved`\|`ask_denied`\|`ask_pending`\|`expired`\|`rewrite`) | quyết định | `decision` |
| `rule_id` | `text` | rule thắng | `rule_id` |
| `risk_tags` | `text[]` | tag rủi ro | `risk_tags[]` |
| `duration_ms` | `integer` | thời gian | `duration_ms` |
| `exit_code` | `integer` | mã thoát | `exit_code` |
| `meta` | `jsonb` (đã redact, ≤ 8 KB) | chi tiết bổ sung (tên env bị loại, domain proxy…) | — |
| `epoch` | `smallint not null default 1` | thế hệ HMAC key/chain | — |
| `prev_hash` | `bytea(32)` | hash dòng trước | `prev_hash` |
| `hash` | `bytea(32) not null` | hash dòng này | `hash` |

`before/after` giữ nguyên cho mutation API; không chứa `system_prompt` (hash + length) và không bao giờ giá trị `credential_ref` (DATABASE_DESIGN dòng 1256).

Chỉ mục: `(organization_id, seq)` unique; `(task_id, created_at)`; `(category, created_at)`; `(correlation_id)`; `(decision) WHERE decision IN ('deny','ask_denied')`. Partition theo `created_at` (tháng) — unique index phải gồm cột partition: dùng `(organization_id, seq, created_at)` hoặc chỉ mục toàn cục không unique + ràng buộc ở chain writer [Giả định: kiểm tra giới hạn PostgreSQL partitioned unique index].

### 4.4 Quyền DB (khớp DATABASE_DESIGN §ledger)

| Role | Quyền trên `audit_logs` |
| --- | --- |
| `aiw_app` | `INSERT`, `SELECT` (không UPDATE/DELETE/TRUNCATE) |
| `aiw_audit_writer` (chain writer) | `INSERT` + `USAGE` sequence; chỉ chain writer ghi |
| `aiw_retention` | `DROP/DETACH PARTITION` qua hàm `SECURITY DEFINER` có kiểm tra checkpoint đã export |
| `aiw_audit_verify` | `SELECT` only |
| Superuser DB | ngoài mô hình đe doạ trong vòng bảo vệ (R-A1); phát hiện qua hash chain |

Trigger `reject_mutation` (BEFORE UPDATE OR DELETE) giữ nguyên [Đã xác minh dòng 1172]; thêm `REVOKE TRUNCATE`, và event trigger cảnh báo `DROP TABLE audit_logs`.

## 5. Đường ghi

### 5.1 Trước và sau (intent/outcome)

```
hook nhận tool call
  → AuditWriter.append({phase:'intent', correlationId, action:'tool.pre', decision?, input_hash,...})   ← phải THÀNH CÔNG
  → engine quyết định
  → AuditWriter.append({phase:'intent', action:'policy.deny'|'exec.rewrite'|'approval.requested'...})
  → trả quyết định cho CLI (chỉ sau khi dòng intent đã commit/spool)
  → thực thi
  → AuditWriter.append({phase:'outcome', correlationId, exit_code, duration_ms})
```
Quy tắc cứng: **không có intent đã ghi thì không trả `allow`/`rewrite`**. Thất bại ghi outcome ⇒ cảnh báo `audit.outcome_missing`, tác vụ không dừng nhưng đánh dấu và quét "intent mồ côi" (không có outcome sau 2×timeout) bằng job 5 phút.

### 5.2 DB down ⇒ spool

- Spool NDJSON: `%LOCALAPPDATA%\aiw\audit\spool\<epoch>-<seq-range>.ndjson`, ACL chỉ user, `fsync` mỗi bản ghi, mỗi dòng đã mang `prev_hash/hash` (chuỗi tiếp nối tại writer, không phụ thuộc DB).
- Trạng thái `audit.degraded`: các hành động `ask`/`rewrite`/`allow` thuộc risk tag cao (`destructive_*`, `network_egress`, `prod_deploy`, `merge_main`) **bị deny** khi cả DB lẫn spool không ghi được (fail-closed); hành động `host_safe` được cho phép tối đa 60 s rồi pause.
- Replay: khi DB trở lại, `aiw audit replay` chèn theo thứ tự `seq`, kiểm tra hash chain; trùng lặp bị loại bởi unique `(organization_id, seq)`; ghi `audit.spool_replay`.
- Spool vượt 100 MB hoặc 15 phút ⇒ cảnh báo critical + pause scheduler.

### 5.3 Chain writer duy nhất và thông lượng

18 agent ⇒ ước lượng 5–30 audit/s cao điểm [Giả định]. Một **AuditWriter** (single-writer process/queue) tuần tự hoá per-org: nhận qua hàng đợi in-memory có giới hạn, batch ≤ 50 dòng/20 ms trong một transaction (`SELECT pg_advisory_xact_lock(hashtext(org))` + tính hash tuần tự). Nếu backend nhiều instance (server mode): leader election bằng advisory lock; MVP desktop 1 instance.

### 5.4 Chống phình log

Lệnh `host_safe` (`git status`, `ls`, `rg`…) **không** tạo dòng audit riêng: gom thành `tool.summary` mỗi 60 s theo (agent, tool) với count và hash của danh sách; mọi quyết định `deny/ask/rewrite` và mọi `Edit/Write` ngoài `writable` luôn ghi dòng riêng. Chunk `agent_events` không vào audit. Mục tiêu < 200 MB/tháng cho 18 agent [Giả định].

## 6. Truy cập log

- UI "Audit Timeline" per task: `agent muốn X → rule Y → user approved lúc Z → chạy trong container C → exit 0` (SECURITY §8), dựng bằng `correlation_id`.
- Quyền xem: Owner toàn bộ; Project Owner theo project (USER_ROLES §3.3); Viewer không.
- Agent **không** có API đọc/ghi audit; audit không nằm trong ngữ cảnh prompt (tránh injection qua log).
- Hiển thị luôn dùng bản đã redact; không có "show raw" từ audit.

## 7. Tamper-evidence: hash chain

### 7.1 Thiết kế

```
hash_n = SHA-256( prev_hash_n || canonical_json(row_n without hash, prev_hash) )
prev_hash_1 (mỗi epoch) = SHA-256("AIW-AUDIT-GENESIS" || org_id || epoch)
```
- `canonical_json`: RFC 8785 (JCS), các trường theo thứ tự cố định, thời gian ISO-8601 UTC micro giây, `null` tường minh, **sau khi** redaction (hash dựa trên nội dung đã redact để verify không cần secret).
- Chuỗi **per-organization** (không per-partition) để partition drop không phá nối tiếp.
- **Checkpoint:** mỗi 1 000 dòng hoặc 10 phút, ghi dòng `audit.checkpoint` chứa `seq`, `hash` và `HMAC-SHA256(hmacKey_epoch, seq || hash || ts)`; HMAC key lưu Credential Manager (C-11 trong CREDENTIAL_MANAGEMENT) — kẻ sửa DB không có key thì không tạo lại checkpoint hợp lệ.
- **Neo ngoài DB (anchoring):** ghi checkpoint cuối ngày vào (a) file `%LOCALAPPDATA%\aiw\audit\anchors\YYYY-MM-DD.json` (ACL), (b) tuỳ chọn gửi email/Git commit riêng/`git notes` vào repo nội bộ để lưu ngoài tầm DB [Đề xuất; Cần người dùng phê duyệt]. Không anchor thì kẻ có quyền DB cắt đuôi + tính lại được cả chuỗi.
- **Rotation key (epoch):** đổi HMAC key ⇒ dòng `audit.epoch_change` ký bằng key cũ, epoch+1, genesis mới tham chiếu hash cuối epoch cũ.

### 7.2 Verify — `aiw audit verify`

1. Đọc theo `seq` (stream, không nạp cả bảng), tính lại `hash`, so `prev_hash`; phát hiện gap/trùng/sửa.
2. Kiểm tra HMAC mọi checkpoint với key của epoch tương ứng.
3. Đối chiếu anchor ngoài DB (nếu có) và `approvals`/`agent_events` bằng `correlation_id` (tỉ lệ intent không outcome).
4. Kết quả: `OK`/`FAIL(seq, lý do)` ghi `audit.verify`; FAIL ⇒ alert critical + chặn approval mới tới khi user ack (tuỳ chọn cấu hình).
5. Lịch: lúc khởi động (đoạn đuôi 10 000 dòng), mỗi giờ (từ checkpoint cuối), hàng ngày (toàn bộ partition tháng hiện tại), hàng tuần (toàn bộ).

### 7.3 Giới hạn trung thực

Hash chain **phát hiện** chứ không **ngăn** sửa; kẻ có quyền superuser + key HMAC (cùng user OS) có thể dựng lại chuỗi — do đó anchor ngoài + chấp nhận R-A1. Không bảo vệ khỏi việc không ghi sự kiện (chỉ khỏi sửa sự kiện đã ghi); intent-trước-quyết định giảm thiểu.

## 8. Retention

Hợp nhất lệch (SECURITY §8: 180 ngày audit; DEVSECOPS §9: 365 ngày online + 2 năm export):

| Dữ liệu | Online | Lưu trữ lạnh | Xoá | Trạng thái |
| --- | --- | --- | --- | --- |
| `audit_logs` | **365 ngày** (partition tháng) | export JSONL.age + chữ ký HMAC thêm 2 năm | sau đó | Q-06 (OWR-2 2026-10-30): audit online 365 ngày, signed export 2 năm; SECURITY §8 đã sửa từ 180 |
| `audit.checkpoint` + anchors | vĩnh viễn (nhỏ) | — | không xoá | Đề xuất |
| Transcript đầy đủ (đã redact) | 30 ngày | — | tự xoá | SECURITY §8 |
| `tool_result` > 64 KB | object store local theo hash | — | purge theo task + 30 ngày | SECURITY §8 |
| `agent_events` | theo DB (partition) | — | drop partition | DATABASE_DESIGN |
| Spool NDJSON | đến khi replay + 7 ngày | — | xoá | Đề xuất |

Purge: job `aiw_retention` chỉ `DETACH PARTITION` khi partition đã **export + verify + ghi anchor**; ghi `audit.purge{partition, rows, firstSeq, lastSeq, lastHash}`; chain vẫn kiểm tra được vì dòng checkpoint đầu partition kế tiếp mang `prev_hash` tham chiếu hash cuối partition đã drop (giữ bản `lastHash` trong bảng nhỏ `audit_chain_anchors`). Legal hold: cờ `hold=true` chặn purge (đề xuất, MVP không dùng).

## 9. Redaction

- Mọi trường văn bản (`target`, `meta`, `before`, `after`, `user_agent`) đi qua `Redactor` duy nhất (CREDENTIAL_MANAGEMENT §8) **trước khi tính hash** và trước insert.
- Không ghi: giá trị secret, `credential_ref`, prompt, nội dung file, stdout; chỉ `sha256`/length/loại.
- Command line: lưu bản đã redact **và** `input_hash` của bản gốc chuẩn hoá (cho TOCTOU) — hash không thể đảo ngược nhưng với lệnh ngắn có thể brute-force; dùng `HMAC(auditKey, input)` thay SHA-256 trần cho `input_hash` khi lệnh < 32 ký tự và có nguy cơ chứa secret [Đề xuất].
- IP: `127.0.0.1` ở MVP; server mode lưu IP (cân nhắc quyền riêng tư; có thể truncate /24).
- Email/username: lưu `userId`, không email đầy đủ.
- Kiểm thử: quét toàn bảng bằng corpus secret sau test e2e ⇒ 0 match (AU-06).

## 10. Export và cảnh báo

### 10.1 Export

| Lệnh | Mô tả |
| --- | --- |
| `aiw audit export --from --to --task --category --format jsonl|csv` | Xuất lọc theo điều kiện; luôn dữ liệu đã redact |
| Bundle ký | `bundle/` = `rows.jsonl`, `manifest.json` (range seq, hash đầu/cuối, số dòng, epoch), `manifest.sig` (HMAC-SHA256, key C-11), kèm checkpoint trong range; mã hoá `age` cho lưu trữ lạnh |
| `aiw audit verify-bundle <file>` | Kiểm chữ ký + chuỗi hash trong bundle độc lập với DB (cần HMAC key của epoch) |
| Quyền | Chỉ Owner; export được ghi `audit.export` (ai, range, số dòng) |
| Định dạng chuẩn | JSONL một dòng một sự kiện, trường theo §4.3 + `schema_version`; tương thích SIEM (ECS-like mapping tuỳ chọn) |

### 10.2 Alerting

Kênh: UI banner + system tray notification + (tuỳ chọn) webhook/email. Alert không bị tắt cho mức critical.

| ID | Điều kiện | Mức | Hành động tự động |
| --- | --- | --- | --- |
| AL-01 | `audit.chain_fail` | critical | khoá approval mới, yêu cầu ack; gợi ý emergency stop |
| AL-02 | `policy_deny` với `credential_access`/`self_modify` | critical | đánh dấu agent suspicious; ≥ 3 lần/giờ ⇒ auto-pause agent |
| AL-03 | `approval.toctou_detected` | critical | hủy approval, pause agent |
| AL-04 | Spool > 100 MB hoặc DB down > 15 phút | critical | pause scheduler |
| AL-05 | `secret.compromise`/redaction hit trên diff/log | critical | quy trình DEVSECOPS §8 |
| AL-06 | ≥ 5 `ask`/phút từ một agent (approval spam) | warning | auto-pause agent (PM-12) |
| AL-07 | `container.inspect_fail` hoặc container `aiw=1` lạ | critical | không exec; kill container lạ |
| AL-08 | `egress.request` tới domain ngoài allowlist/khối lượng bytes > ngưỡng | warning→critical | ngắt network container |
| AL-09 | `file.write_outside_manifest` lặp ≥ 3 lần/task | warning | task `blocked` |
| AL-10 | `agent.spawn` mà hook không verified (`SessionStart` không thấy) | critical | kill agent (R8) |
| AL-11 | Emergency stop kích hoạt | critical | thông báo; báo cáo `killed_clean/kill_incomplete` |
| AL-12 | Elevated process phát hiện / app chạy admin | critical | từ chối khởi động |
| AL-13 | Rơi sự kiện audit (queue đầy) | critical | backpressure: chặn hành động nguy cơ |

## 11. Security test cases

| ID | Test | Kỳ vọng |
| --- | --- | --- |
| AU-01 | Hành động `allow` không có dòng intent | engine không trả allow (test bằng mock writer lỗi) |
| AU-02 | Tắt Postgres giữa chừng | ghi spool; replay đúng thứ tự; chain verify OK |
| AU-03 | `UPDATE audit_logs` / `DELETE` bằng role app | bị từ chối (trigger + quyền) |
| AU-04 | Sửa một dòng bằng superuser rồi `aiw audit verify` | FAIL tại đúng `seq` |
| AU-05 | Approve qua UI | dòng `approval.decided` có `actor_kind=user`, `request_id`, `ip` |
| AU-06 | E2E với secret giả trong lệnh/output | quét `audit_logs` toàn bảng: 0 match |
| AU-07 | Cắt đuôi chuỗi (xoá 100 dòng cuối) + anchor | verify phát hiện (anchor/ checkpoint HMAC) |
| AU-08 | Hai chain writer đồng thời (race) | chỉ một thắng; không gap seq |
| AU-09 | 18 agent × 30 audit/s trong 5 phút | không rơi, p99 ghi < 50 ms, backpressure đúng |
| AU-10 | Partition purge | chỉ khi đã export+verify; chain vẫn verify sau purge |
| AU-11 | Đổi HMAC key (epoch) | verify liên epoch OK; key cũ cần để verify epoch cũ |
| AU-12 | Verify bundle export bằng máy khác | khớp; sửa 1 byte ⇒ FAIL |
| AU-13 | Mỗi alert AL-01..AL-13 có trigger test | đúng hành động tự động |
| AU-14 | Agent cố ghi/đọc audit qua API | 403 / không có endpoint |
| AU-15 | `host_safe` flood 10 000 lệnh | gom `tool.summary`, bảng không phình |
| AU-16 | Khởi động với elevated | audit `AL-12` và từ chối |

Gate CI: PR chạm `audit/`, `redaction/`, migration `0006*` cần AU-01..AU-16 pass và coverage ≥ 90 %.

## 12. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
| --- | --- | --- |
| Tên chuẩn `audit_logs`; bổ sung cột thay vì bảng mới | Khớp migration hiện có, ít thay đổi | Đề xuất |
| Chuỗi hash per-organization, checkpoint HMAC + anchor ngoài DB | Phát hiện cắt đuôi, partition drop không phá chuỗi | Đề xuất |
| Single chain writer, batch trong transaction | Thứ tự hash tất định với 18 agent | Đề xuất |
| Intent trước, outcome sau; fail-closed khi không ghi được | SECURITY §8 | Đề xuất |
| Retention 365 ngày online (thay 180) | Thống nhất DEVSECOPS; đủ điều tra | Q-06 (OWR-2 2026-10-30): Owner chốt 365 ngày online + signed export 2 năm |
| `host_safe` gom `tool.summary` | Tránh phình log | Đề xuất |
| Hash trên dữ liệu đã redact | Verify không cần secret | Đề xuất |
| `input_hash` dùng HMAC khi lệnh ngắn | Chống brute-force từ hash | Đề xuất |

## 13. Dependencies

- `DATABASE_DESIGN.md`: `audit_logs`, `agent_events`, `approvals`, partition + `reject_mutation`, migration 0006.
- [`CREDENTIAL_MANAGEMENT.md`](CREDENTIAL_MANAGEMENT.md) (Redactor, HMAC key C-11), [`PERMISSION_MODEL.md`](PERMISSION_MODEL.md) (rule_id, approval), [`SANDBOX_DESIGN.md`](SANDBOX_DESIGN.md) (container events, emergency stop), [`THREAT_MODEL.md`](THREAT_MODEL.md) (TM-R-*, TM-T-05).
- `AuditInterceptor` (NestJS), `@napi-rs/keyring`, `age`, RFC 8785 canonicalizer.

## 14. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| R-A1: superuser DB + key HMAC cùng user OS dựng lại chuỗi | Trung bình | Anchor ngoài DB; chấp nhận |
| Single writer là nút thắt/SPOF | Trung bình | batch, spool, backpressure, test AU-09 |
| Fail-closed khi audit hỏng làm dừng công việc | Trung bình | spool; cho `host_safe` 60 s; rõ ràng trên UI |
| Phình dữ liệu | Thấp | gom summary, partition, retention |
| Canonical JSON khác biệt giữa phiên bản ⇒ verify sai | Trung bình | `schema_version`, test vector cố định, khoá canonicalizer |
| Quyền riêng tư (IP/user) ở server mode | Thấp | truncate/hash |

## 15. Vấn đề chưa giải quyết (kèm inconsistency)

1. **Tên bảng:** `audit_logs` (DATABASE_DESIGN) vs `audit_events` (SECURITY, DEVSECOPS, INFRA).
2. **Schema:** DATABASE_DESIGN dùng `before/after`, `entity_type/entity_id`, `uuid` PK; SECURITY dùng `bigserial`, `category`, `input_hash`, `hash` — cần migration §4.3.
3. **Retention:** 180 (SECURITY) vs 365 (DEVSECOPS). [Đã chốt theo Q-06, OWR-2: 365 ngày online + signed export 2 năm; SECURITY §8 đã sửa.]
4. **Phạm vi:** DATABASE_DESIGN: agent actions không vào `audit_logs` mà `agent_events`; SECURITY §8: tool_call/tool_result vào audit. Đề xuất tách như §2/§5.4.
5. DEVSECOPS §9 nói hash chain "tuỳ chọn sau MVP"; SECURITY §8 coi là yêu cầu — đề xuất bật từ M0 (SECURITY §11) vì chi phí thấp.
6. Giá trị enum `audit_severity` chưa xem đầy đủ (Giả định).
7. Anchor ngoài DB dùng kênh nào (file/email/git notes)?
8. Bảo vệ khóa HMAC khỏi user cùng máy: không khả thi hoàn toàn; chấp nhận.
9. Partitioned unique index với `seq` (kiểm chứng PostgreSQL 16).
10. Trách nhiệm đồng bộ `correlation_id` giữa `agent_events`, `approvals`, `audit_logs`.

## 16. Acceptance Criteria

- [ ] Migration §4.3 áp dụng được trên schema hiện có; `reject_mutation` vẫn bắt UPDATE/DELETE; role app không có quyền ghi ngoài INSERT.
- [ ] Chain writer: không gap `seq`, không race (AU-08); batch p99 < 50 ms ở 18 agent × 30 audit/s (AU-09).
- [ ] `aiw audit verify` phát hiện sửa, xoá giữa, cắt đuôi (AU-04, AU-07); verify bundle ngoài DB (AU-12).
- [ ] Spool + replay: DB down 15 phút không mất dòng nào, fail-closed cho hành động rủi ro cao (AU-02).
- [ ] 0 secret trong `audit_logs` sau e2e (AU-06).
- [ ] Alert AL-01..AL-13 đều có test và hành động tự động đúng (AU-13).
- [ ] Retention/purge đúng quy trình export→verify→anchor→detach (AU-10); epoch rotation (AU-11).
- [ ] Timeline UI per task dựng được từ `correlation_id` cho một kịch bản "ask → approve → rewrite → exec → exit 0".
- [ ] Đóng 5 điểm lệch §15 (1–5) với chủ sở hữu tài liệu liên quan trước M1.
