# PERMISSION MODEL

> Security Architect · Phase 0 (documentation only) · 2026-10-30 · Trạng thái: PROPOSAL v1
> Hợp nhất và cụ thể hoá: `docs/product/USER_ROLES_AND_PERMISSIONS.md` (BA, vai trò/level/POL-xx), [`../SECURITY.md`](../SECURITY.md) §3, §5, §6 (denylist, engine, manifest), `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md` §6–7 (manifest), `docs/DATABASE_DESIGN.md` §4.9 (approval_policies, approvals).
> Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Định nghĩa một mô hình quyền duy nhất, thực thi được bằng code, gồm: vai trò người dùng, mức quyền agent, schema chính sách tool/path/network/command (JSON/YAML), thứ tự đánh giá, approval policy, enforcement file ownership manifest, escalation và bộ policy mặc định. Giải quyết các điểm lệch giữa tài liệu BA, security và database (xem §12).

## 2. Phạm vi

- Trong: human role (MVP single-user + sẵn data model multi-user), agent permission level L0–L5, policy schema, engine, approval, manifest, escalation, default set.
- Ngoài: xác thực multi-user/SSO, UI Approval Center (frontend), cơ chế thực thi sandbox (SANDBOX_DESIGN), lưu audit (AUDIT_LOGGING).
- Nguyên tắc tối cao: **policy layer ≠ boundary** (SECURITY §0). Quyền ở đây quyết định *cái gì được đề nghị cho phép*; boundary thật do container (SANDBOX_DESIGN).

## 3. Human roles

| Vai trò | MVP | Phạm vi | Ghi chú |
| --- | --- | --- | --- |
| Owner / Director | Có (người duy nhất, không login app) | Toàn workspace | Toàn quyền, kể cả policy/security/resume sau emergency stop |
| Project Owner | Không (data model sẵn: `users.role`) | Theo project | Duyệt BA/UI/Acceptance project mình, pause/resume agent |
| Viewer | Không | Workspace/project | Chỉ xem |

Ma trận quyền chi tiết giữ nguyên ở `USER_ROLES_AND_PERMISSIONS.md` §3.3 (AP-ROLE-01), không lặp lại. Bổ sung từ góc security:

1. **Quyền không thể uỷ thác cho agent:** thay đổi policy, security settings, approval `prod_deploy`/`merge_main`/`data_deletion`, resume sau emergency stop, nhập/xoá secret, đổi permission level của agent.
2. **Single-user:** không có đăng nhập app nên "danh tính người duyệt" = bearer token UI hợp lệ + Origin hợp lệ; audit ghi `actor=user:owner`, `request_id`, `ip=127.0.0.1`. Hệ quả: không có tách biệt nhiệm vụ (separation of duties) — chấp nhận ở MVP [Đề xuất].
3. **Emergency stop:** mọi role (khi có multi-user) được kích hoạt; chỉ Owner resume (USER_ROLES §3.3 đã đề xuất).
4. Thay đổi policy runtime bị giới hạn: **không thể** hạ/loại rule priority 10–20 (denylist, self_modify, credential_access) khi app đang chạy; chỉ qua file config được commit + restart (SECURITY §5.2).

## 4. Agent permission levels

### 4.1 Hợp nhất L0–L5 với enum database

`USER_ROLES_AND_PERMISSIONS.md` §4.2 đề xuất L0–L5; `DATABASE_DESIGN.md` enum `permission_level` = `read_only | standard | elevated | full` (dòng 170, 343). Đề xuất ánh xạ (không đổi enum, thêm cột/field chi tiết):

| Level (BA) | Enum DB | Ý nghĩa an ninh | Tool mặc định | Container exec | Ghi |
| --- | --- | --- | --- | --- | --- |
| L0 Read-only | `read_only` | Chỉ quan sát | Read, Grep, Glob | Không | Không |
| L1 Write-own | `standard` | Ghi trong `writable` | + Edit/Write | Chỉ lệnh safe allowlist (lint/test unit) | manifest `writable` |
| L2 Developer | `standard` + `profile=developer` | Branch/commit worktree, build/test | + Bash (rewrite), git local | Có | manifest |
| L3 Lead/Reviewer | `elevated` | Review, cấp quyền tạm thời trong phạm vi task | L2 + đọc rộng | Có | manifest + grant |
| L4 Operator | `elevated` + `profile=operator` | Build artifact, preview non-prod | L3 + preview | Có | + preview |
| L5 PM/Orchestrator | `elevated` + `profile=orchestrator` | Tạo task/assign; **không chạy code** (OI-ROLE-02) | Tool app-level, không Bash | Không | Không |
| (không dùng) | `full` | Không bao giờ cấp cho agent | — | — | Enum tồn tại nhưng policy engine từ chối `full` [Đề xuất: bỏ khỏi enum hoặc khoá bằng CHECK] |

Cần mở rộng `agents.permission_level` thêm `permission_profile text` hoặc lưu L0–L5 vào `agents.config` [Cần người dùng phê duyệt].

### 4.2 Giao của các nguồn quyền

Quyền hiệu lực của một tool call = **giao** của:

```
effective = agentLevel ∩ agentBuilder.tools ∩ skill.allowedTools ∩ task.manifest ∩ project.trustPolicy ∩ globalDenylist
```
(Agent Builder `tools` và Skill `allowed tools`: FR-067). Một tập rỗng ⇒ deny. Không có phép hợp (union) ngoại trừ **grant tạm thời** (§8) có TTL.

### 4.3 Luật bất biến (immutables)

8 luật trong `USER_ROLES_AND_PERMISSIONS.md` §4.5 được hiện thực thành rule không thể tắt (priority 10–20). Bổ sung bất biến kỹ thuật: (9) không `bypassPermissions`/`--dangerously-skip-permissions` trong Tier 1; (10) không spawn CLI lồng nhau; (11) agent không thay đổi `permission level`, policy, manifest của chính mình; (12) không gọi Docker API.

## 5. Policy schema (tool / path / network / command)

Policy được lưu dưới dạng file YAML/JSON trong thư mục policy **ngoài worktree** (`%LOCALAPPDATA%\aiw\policy\`), có chữ ký hash lưu audit, nạp lúc start; bản DB (`approval_policies.rules jsonb`) chỉ chứa phần approval (xem §6). Schema JSON Schema `policy.v1`:

```yaml
# policy/default.policy.yaml
schemaVersion: 1
id: default
precedence: [deny, ask, rewrite, allow]      # thắng theo thứ tự này trong CÙNG priority
defaults: { decision: ask, approver: user, ttlMinutes: 30, scope: once }

tools:                                      # theo tên tool chuẩn của CLI
  allow: [Read, Grep, Glob, Edit, Write, MultiEdit, NotebookEdit, Bash]
  deny:  [WebSearch, "mcp__*"]              # MCP chỉ server do app cung cấp
  webFetch: { mode: allowlist, domains: ["docs.python.org", "developer.mozilla.org", "nodejs.org"] }

paths:                                      # glob, minimatch dot:true, sau realpath; "/" làm phân cách
  roots:
    worktree: "${WORKTREE}"
    shared_ro: "${PROJECT}/shared-ro"
  deny_read:
    - "${HOME}/.claude/**"
    - "${HOME}/.codex/**"
    - "${HOME}/.ssh/**"
    - "${HOME}/.aws/**"
    - "${HOME}/AppData/**"
    - "**/.env"
    - "**/.env.*"
    - "**/*.pem"
    - "**/*.key"
    - "${WORKTREE}/../**"                   # worktree khác
  deny_write:
    - "**/.git/**"
    - ".claude/**"
    - ".aiw/**"
    - "CLAUDE.md"
    - "AGENTS.md"
    - ".github/workflows/**"
  ask_write: ["Dockerfile", "**/*.lock", "package.json#scripts", "apps/api/src/database/schema/**"]
  reject: { uncPaths: true, reparsePoints: true, shortNames8dot3: true, alternateDataStreams: true }

network:
  host:      { default: deny }              # Bash trên host không có network; web qua WebFetch tool
  container:
    default: none
    phases:
      install: { mode: proxy, domains: [registry.npmjs.org, pypi.org, files.pythonhosted.org, github.com] }
      test:    { mode: none }
  untrustedProject: { container: none, install: ask }

commands:
  normalize: { stripWrappers: [cmd, powershell, pwsh, bash, sh, sudo, nice, time, env], resolveBasename: true, decodeBase64: true, splitOn: ["&&","||",";","|","&","\n"] }
  classes:
    host_safe:         ["git status","git log","git diff","git show","rg","ls","cat","head","tail","wc","pwd","tree"]
    workspace_mutate:  ["git add","git commit","mkdir","touch","mv","cp"]   # target phải trong worktree
    executes_code:     ["npm","pnpm","yarn","bun","node","python","pip","cargo","go","dotnet","make","pytest","jest","vitest","tsc","npx","deno"]
  deny:  ["<danh sách regex SECURITY §3.1 theo nhóm>"]                       # tham chiếu, không nhân bản
  ask:   ["git push","git merge","git rebase","npm publish","docker build","gh pr merge","drizzle-kit migrate"]
  limits: { maxToolCallsPerTask: 300, taskWallClockMin: 120, bashTimeoutSec: 120, maxStdoutMB: 50, maxFilesDeletedPerCommand: 20, maxWriteMB: 5 }

rules:                                      # xem §6; priority thấp thắng
  - { id: R010, priority: 10, match: { commandClass: dangerous },                decision: deny, rationale: "Denylist SECURITY 3.1" }
  - { id: R020, priority: 20, match: { riskTag: [self_modify, credential_access] }, decision: deny }
  - { id: R060, priority: 60, match: { commandClass: executes_code },            decision: rewrite, rewrite: { template: "aiw-exec {project} --task {task} --hash {inputHash} -- {command}" } }
  - { id: R070, priority: 70, match: { tool: [Edit, Write, MultiEdit], pathClass: writable }, decision: allow }
```

### 5.1 JSON ví dụ cho một agent cụ thể (compile từ policy + manifest + level)

```json
{
  "agentId": "backend-02", "level": "L2", "profile": "developer", "trust": "trusted",
  "effectiveTools": ["Read","Grep","Glob","Edit","Write","MultiEdit","Bash"],
  "cwd": "D:\\aiw\\worktrees\\4K7QX2MZ",
  "addDir": [],
  "claudeSettings": {
    "permissions": {
      "defaultMode": "manual",
      "deny": ["Read(//C:/Users/TUYEN/.claude/**)", "Read(./.env*)", "Edit(./.claude/**)", "Edit(./.aiw/**)", "Bash(docker *)", "Bash(curl *)"],
      "allow": ["Read", "Grep", "Glob"],
      "disableBypassPermissionsMode": "disable",
      "blockReadsOutsideWorkingDirectories": true
    },
    "hooks": { "PreToolUse": [{ "matcher": "*", "hooks": [{ "type": "http", "url": "http://127.0.0.1:${PORT_H}/hooks/pre", "headers": { "Authorization": "Bearer ${HOOK_TOKEN}" }, "timeout": 600 }] }] },
    "env": { "BASH_MAX_TIMEOUT_MS": "600000" }
  }
}
```

(`disableBypassPermissionsMode` nằm dưới `permissions` theo SECURITY §2.3; đặt đúng vị trí khi hiện thực và xác nhận trong snapshot test — Giả định về key chính xác trên 2.1.312.)

Codex tương ứng: `config.toml` sinh bởi app, `sandbox_mode="workspace-write"`, `approval_policy="on-request"` (SECURITY §2.5) — khác WORKSPACE doc §7.1 (`--ask-for-approval never`); lấy SECURITY làm chuẩn (§12).

## 6. Approval policy & thứ tự đánh giá

### 6.1 Thứ tự đánh giá (pipeline một tool call)

```
 0. Preconditions : hook token hợp lệ; session còn sống; system.state != stopping
 1. Normalize     : strip wrapper, resolve basename, split segment, decode base64, canonicalize path (realpath, reparse, UNC)
 2. Classify      : commandClass + riskTags (pattern trước, LLM-judge advisory sau)
 3. Immutables    : luật bất biến + denylist (priority 10–20)            → deny, kết thúc
 4. Capability    : effective = level ∩ tools ∩ skill ∩ manifest ∩ trust  → rỗng ⇒ deny
 5. Path/Manifest : forbidden → deny; readOnly/unmatched write → ask; writable → tiếp
 6. Network       : host network → deny; container network theo phase/trust
 7. Rules         : so khớp theo priority tăng dần; rule đầu tiên khớp thắng
                    (cùng priority: deny > ask > rewrite > allow)
 8. Approval      : nếu ask → cached decision? (agent, tool, normalizedPrefix, scope, ttl, input_hash) → có ⇒ dùng
                    chưa ⇒ tạo approvals row, giữ hook mở chờ quyết định
 9. Budget/limit  : maxToolCalls, token budget (POL-08), wall-clock → vượt ⇒ ask (POL-08) hoặc pause
10. Rewrite       : nếu decision=rewrite → updatedInput
11. Audit         : ghi intent (trước) → trả quyết định → ghi outcome (sau)
```

Quy tắc bất biến của engine: (a) lỗi bất kỳ ở bước 0–10 ⇒ **deny** (fail-closed); (b) LLM-judge chỉ có thể nâng mức nghiêm ngặt (`allow→ask`, `ask→deny`), không bao giờ hạ; (c) kết quả bước 3 không bị bước sau ghi đè; (d) một lệnh ghép chỉ được qua khi **mọi** segment qua.

### 6.2 Mô hình dữ liệu approval

Dùng bảng `approval_policies`/`approvals` hiện có (DATABASE_DESIGN §4.9) và bổ sung từ SECURITY §5.3 (mục "Lệch" §12):

| Trường SECURITY | Trường DB hiện có | Đề xuất |
| --- | --- | --- |
| `source{claude_hook,claude_control,codex,app}` | không có | thêm vào `payload.source` hoặc cột `source` |
| `input_hash` | không có | thêm cột `input_hash text not null` (SHA-256 canonical) |
| `risk_tags[]`, `rule_id` | `policy_id` | thêm `risk_tags text[]`, `rule_id text` |
| status `denied`,`revoked` | enum `pending/approved/rejected/expired/cancelled` | map `denied→rejected`, `revoked→cancelled`; thêm `revoked` nếu cần phân biệt |
| `scope{once,task,session,project}` | `decision{approve_once, approve_always}` | map: `approve_once→once`, `approve_always→task` (**không** `project` mặc định) |
| `expires_at` | `expiresAt` | khớp |
| `on_timeout` | `onTimeout default 'reject'`, `timeoutSec 3600` | SECURITY: hook giữ 540 s; chính sách hết hạn 3600 s của DB khác với hook; xem §12 |

### 6.3 TOCTOU và tính toàn vẹn quyết định

1. `input_hash = SHA-256(canonicalJSON({tool, normalizedInput, cwd, manifestHash}))`.
2. Quyết định chỉ hợp lệ cho đúng `input_hash`; cached decision keyed `(agentId, tool, normalizedCommandPrefix)` **kèm** ràng buộc tham số nhạy cảm (path nằm trong cùng thư mục cha, không có `--force`).
3. `aiw-exec` và hook xác nhận lại hash ngay trước khi thực thi; lệch ⇒ deny + audit `toctou_detected`.
4. Cached decision không bao giờ persist vào `permissions.allow` của settings user (SECURITY §5.3) và hết hạn khi task kết thúc/TTL/emergency stop.
5. Không cached decision cho rule class `deny`-adjacent: `prod_deploy`, `data_deletion`, `merge_main` luôn `scope=once`, ttl ≤ 15 phút.

### 6.4 Timeout approval — hoà giải

BA đề xuất "không timeout, nhắc 24 h" (USER_ROLES §5.5) vs SECURITY §5.3 "hook timeout 540 s ⇒ deny, task `blocked`". Đề xuất hợp nhất: hook HTTP không thể giữ vô hạn (600 s tối đa). Khi hết 540 s: hook trả `deny` với lý do "pending_human"; approval **vẫn `pending`** trong DB (không expire); task chuyển `WAITING_HUMAN` (D-003, không giữ worker). Khi user duyệt muộn, orchestrator **kích hoạt lại** hành động từ đầu (agent được nhắc retry với hash đã duyệt) thay vì giữ kết nối. `expires_at` mặc định 7 ngày; `onTimeout='reject'`. [Cần người dùng phê duyệt]

## 7. Enforcement file ownership manifest

### 7.1 Chuẩn hoá schema (hợp nhất hai định dạng)

WORKSPACE doc §6 (JSON, `readOnly`) và SECURITY §6.2 (YAML, `readonly`) lệch tên/định dạng. Chuẩn đề xuất: **JSON `schemaVersion: 1`, key `readOnly`**, bổ sung trường của SECURITY (`maxLinesChanged`, `forbidden`):

```json
{ "schemaVersion": 1, "taskId": "01JD4K7QX2MZ", "agent": {"slug": "backend-02", "cli": "claude"},
  "writable": ["apps/api/src/orders/**", "apps/api/test/orders/**"],
  "readOnly": ["apps/api/src/common/**", "packages/shared/**", "docs/**"],
  "forbidden": [".env*", "**/*.pem", "**/*.key", ".git/**", ".aiw/**", ".ai-workspace/**", ".claude/**", ".github/**", "infra/**"],
  "allowNewFilesUnder": ["apps/api/src/orders/**"], "allowDeletes": false,
  "maxChangedFiles": 60, "maxLinesChanged": 2500, "maxDiffBytes": 2000000,
  "grants": [] }
```

Vị trí: **ngoài worktree** (`<root>/manifests/<taskId>.manifest.json`, ACL user-only); bản đọc nhanh cho hook nằm trong backend memory, không ở `.ai-workspace/manifest.json` của worktree (WORKSPACE §4 đặt copy trong worktree — agent có thể sửa nếu Lớp 3 không chặn). Hash manifest lưu audit lúc tạo task.

### 7.2 Thứ tự ưu tiên path

`forbidden` > `readOnly` > `writable`; path không khớp ⇒ coi như `readOnly` (write ⇒ ask, rule R080). Glob `minimatch {dot:true}`, path chuẩn hoá `/`, so sánh sau `realpath`; hai path chỉ khác hoa/thường bị từ chối lúc tạo manifest (WORKSPACE §11).

### 7.3 Ba lớp enforcement

| Lớp | Thời điểm | Cơ chế | Phụ thuộc CLI? |
| --- | --- | --- | --- |
| 1 Hook | PreToolUse trên `Edit/Write/MultiEdit/NotebookEdit` + segment Bash có redirect/`mv/cp/sed -i/tee/Out-File/Set-Content` | `classify(path)`; `forbidden`⇒deny; ngoài `writable`⇒ask | Có (R8) |
| 2 Container | exec trong container: workspace volume; manifest được dịch thành quyền ghi? (không thể per-glob) | Chỉ giới hạn tới worktree | Không |
| 3 Diff validation (`validateDiff`) | Trước merge | `git diff --name-only` ⊆ `writable` + ngoại lệ đã duyệt; quota; secret; `.github`, Dockerfile, lockfile; binary; reparse | **Không — lớp quyết định** |

Hợp nhất Windows: nếu hook bỏ sót (bug/obfuscation), Lớp 3 phát hiện sau; vi phạm ⇒ task `needs_review`, không merge tự động, lưu `violations.json`.

### 7.4 Xung đột và đồng thời

- Scheduler từ chối gán hai task có `writable` giao nhau (mở rộng AB-02); phát hiện giao bằng so sánh glob (`picomatch` + kiểm tra tiền tố) và fallback chạy `git ls-files` expand.
- Với 18 agent: bảng `manifest_locks(path_prefix, task_id)` giữ lock tiền tố trong Redis; giải phóng khi task `merged/cancelled`.
- Grant tạm thời (§8) tạo bản **manifest overlay** có `expiresAt`, không sửa manifest gốc.

## 8. Escalation và grant tạm thời

| Tình huống | Luồng | Approver | TTL |
| --- | --- | --- | --- |
| Agent cần ghi file ngoài `writable` | hook `ask` → Approval Center: "mở rộng manifest thêm `<glob>`" | Lead agent L3 **đề xuất**; user duyệt nếu chạm `readOnly`; `forbidden` luôn user + lý do | 1 task |
| Agent cần lệnh ngoài allowlist (POL-06) | `ask` | Owner | scope `task` |
| Truy cập network ngoài allowlist (POL-07) | `ask` + cảnh báo `network_egress` | Owner | `once`, TTL 15 phút |
| Vượt budget (POL-08) | `ask` | Owner | tăng ngân sách có trần |
| Chạm `forbidden`/`credential_access`/`self_modify` | **deny**, không có đường escalation runtime | — | — |
| Agent bị chặn lặp ≥ 3 lần cùng rule | task `blocked`, thông báo PM agent + user, gợi ý sửa manifest | PM/User | — |
| Cấp quyền tạm thời cho agent khác (L3+) | overlay manifest, ghi audit `grant`, hiển thị trên UI | L3+ trong phạm vi task mình quản lý; chạm `forbidden` ⇒ user | ≤ 4 giờ |

Giới hạn escalation (chống approval fatigue AB-01): tối đa 5 `ask`/phút/agent và 30/giờ; vượt ⇒ auto-pause agent và báo "agent đang spam approval" (PM-12); nhóm các `ask` tương tự thành một thẻ.

OI-ROLE-03 (Lead L3 có "approve" thay human ở POL-11?) — đề xuất: Lead chỉ **review**, `approve merge` vào `integration/*` được tự động sau Lớp 3 + review L3 đạt; vào `main` luôn user (khớp SECURITY rule 30 và WORKSPACE §8). Khác POL-11 (tùy chọn) — xem §12.

## 9. Bộ policy mặc định (default set)

Hợp nhất `SECURITY §5.2` (rule 10–90) và `USER_ROLES §5.3` (POL-01..12):

| Priority | Rule ID | Điều kiện | Quyết định | Ánh xạ POL | Có tắt được? |
| --- | --- | --- | --- | --- | --- |
| 10 | R010 | Denylist (SECURITY §3.1) | deny | luật bất biến 6 | Không (chỉ commit config + restart) |
| 20 | R020 | `self_modify`, `credential_access` | deny | luật bất biến 2,5 | Không |
| 25 | R025 | Tool/lệnh ngoài `effective` (level/tools/skill) | deny | — | Không |
| 30 | R030 | `prod_deploy`, `merge_main`, `data_deletion` | ask, user, once, ttl 15 | POL-04, POL-11, POL-10 | `prod_deploy`: Không |
| 35 | R035 | Stage 8 Human Acceptance | ask (bắt buộc) | POL-03 | Không |
| 36 | R036 | Stage 2 BA, Stage 4 UI/UX hoàn thành | ask | POL-01, POL-02 | Có (Owner) |
| 40 | R040 | `destructive_git`, `arch_change`, `ci_change`; `dependency_add` ở project untrusted | ask, scope task | POL-10, POL-12 | Có |
| 45 | R045 | Network ngoài allowlist | ask, once | POL-07 | Có |
| 46 | R046 | Lệnh ngoài allowlist nhưng không trong denylist | ask | POL-06 | Có |
| 47 | R047 | Vượt budget token/cost | ask | POL-08 | Có |
| 48 | R048 | Đề xuất thay đổi ngoài phạm vi task | ask (Owner/PO hoặc L3) | POL-09 | Có |
| 50 | R050 | `dependency_add` ở project trusted | allow + audit + `npm audit` | POL-12 (mặc định tắt) | Có |
| 55 | R055 | Stage 3 System Design hoàn thành | ask (tuỳ chọn, mặc định tắt) | POL-05 | Có |
| 60 | R060 | `executes_code` | rewrite → container | — | Không (Tier 1) |
| 70 | R070 | Edit trong worktree & `writable` | allow | — | — |
| 80 | R080 | Edit ngoài manifest | ask (lead có thể mở rộng) | — | — |
| 90 | R090 | Mặc định | ask | — | — |

Khác biệt cần chốt: POL-11 (merge vào nhánh chính "tùy chọn, code review L3 đủ") vs R030 (`merge_main` luôn ask) vs D-009 (không tự merge vào `main`) — §12.

## 10. Test case (ID tham chiếu từ THREAT_MODEL)

| ID | Nội dung | Kỳ vọng |
| --- | --- | --- |
| PM-01 | `effective` tập rỗng (level ∩ skill) | deny |
| PM-02 | Rule đầu tiên theo priority thắng; cùng priority deny > ask | đúng thứ tự |
| PM-03 | settings sinh ra chứa `disableBypassPermissionsMode`, không chứa `bypassPermissions`; spawn con với `--dangerously-skip-permissions` | deny + audit |
| PM-04 | LLM-judge trả "safe" cho lệnh ask-class | vẫn ask |
| PM-05 | Agent ghi `.claude/settings.json` / `.aiw/**` / `CLAUDE.md` | deny (hook) + Lớp 3 reject |
| PM-06 | Agent đọc `..\worktrees\<other>\` | deny |
| PM-07 | Agent xin mở rộng manifest chạm `forbidden` | ask user, không auto |
| PM-08 | Hook timeout/JSON lỗi/backend down | deny (fail-closed) |
| PM-09 | Đổi lệnh sau khi duyệt (hash lệch) | deny `toctou_detected` |
| PM-10 | Lead agent gọi API approve | 403; approve chỉ từ UI token |
| PM-11 | Hai task `writable` giao nhau | scheduler từ chối |
| PM-12 | 6 `ask`/phút từ một agent | auto-pause |
| PM-13 | Level `full` gán cho agent | từ chối |
| CP-* | Lệnh obfuscation corpus (SECURITY §9) | deny/ask đúng; ≥ 300 case |

## 11. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
| --- | --- | --- |
| Quyền = giao của 6 nguồn; chỉ grant tạm thời mới là hợp | Tránh leo thang ngầm | Đề xuất |
| Ánh xạ L0–L5 → enum DB + profile; cấm `full` | Giữ schema, mở rộng tối thiểu | Cần người dùng phê duyệt |
| Policy file ngoài worktree, hash lưu audit | Chống TM-T-01 | Đề xuất |
| Approval pending không expire theo hook; reactivation | Hợp nhất BA (không timeout) với giới hạn hook 600 s | Cần người dùng phê duyệt |
| Manifest JSON chuẩn `readOnly`, đặt ngoài worktree | Hợp nhất hai tài liệu | Đề xuất |
| Lead agent không approve thay human | Đóng OI-ROLE-03 theo hướng an toàn | Cần người dùng phê duyệt |
| Fail-closed ở mọi lỗi engine | Tránh bypass bằng lỗi | Đề xuất |

## 12. Dependencies và inconsistency liên quan

Dependencies: `USER_ROLES_AND_PERMISSIONS.md`, `SECURITY.md`, `WORKSPACE_AND_GIT_ISOLATION.md`, `DATABASE_DESIGN.md` §4.9, D-003, D-004, D-009; [`SANDBOX_DESIGN.md`](SANDBOX_DESIGN.md), [`AUDIT_LOGGING.md`](AUDIT_LOGGING.md), [`CREDENTIAL_MANAGEMENT.md`](CREDENTIAL_MANAGEMENT.md).

Lệch cần chủ sở hữu từng doc xử lý (tôi không sửa file của người khác):
1. Enum `permission_level` (4 giá trị) vs L0–L5; `full` vô nghĩa.
2. `approvals` thiếu `input_hash`, `source`, `risk_tags`; status `denied/revoked` vs `rejected/cancelled`.
3. Timeout approval: 540 s (SECURITY) vs 3600 s `timeout_sec` DB vs "không timeout" BA.
4. Manifest: JSON `readOnly` ở WORKSPACE vs YAML `readonly` ở SECURITY; vị trí trong/ngoài worktree.
5. Chế độ spawn: WORKSPACE `acceptEdits` + `.claude/settings.json` + hook `command` vs SECURITY `manual` + `--settings` + hook `http`; Codex `never` vs `on-request`.
6. POL-11/POL-12 so với rule 30/50.

## 13. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| Quá chặt ⇒ agent BLOCKED nhiều | Trung bình | log lý do chặn, nút grant nhanh, tuning theo dữ liệu |
| Approval fatigue (AB-01) | Trung bình | rate limit, gom nhóm, scope hẹp |
| Policy engine bug ⇒ fail-open | Cao | fail-closed, ≥ 90 % coverage, property test |
| Hai nguồn policy (file vs DB) lệch | Trung bình | file là nguồn cho immutables; DB chỉ approval; hash đối chiếu |
| Single-user không có separation of duties | Thấp | chấp nhận; audit |

## 14. Vấn đề chưa giải quyết

1. Hành vi chính xác của exit 2 / `updatedInput` trong `-p` trên 2.1.312 (WORKSPACE §16.6) cần test thực.
2. Vị trí chính xác key `disableBypassPermissionsMode` trong settings (Giả định).
3. Số cấp L0–L5, PM chạy code (OI-ROLE-01/02), multi-user data model (OI-ROLE-04).
4. Có dùng `canUseTool` của Agent SDK (D-004) thay cho PreToolUse hook không; hai đường phải dùng chung một engine.
5. Cơ chế phát hiện giao glob chính xác giữa manifest.
6. Taint flag có ảnh hưởng quyết định (không chỉ hiển thị)?

## 15. Acceptance Criteria

- [ ] JSON Schema `policy.v1` + validator; policy mặc định §9 nạp được và có snapshot test.
- [ ] Engine §6.1 có test PM-01..PM-13 pass; coverage ≥ 90 % cho module policy.
- [ ] Mọi lỗi nội bộ (timeout, parse, DB down) trả deny (test chaos).
- [ ] Manifest chuẩn hoá; 3 lớp enforcement có test: ghi ngoài `writable` ⇒ ask, `forbidden` ⇒ deny, bypass hook ⇒ Lớp 3 bắt.
- [ ] Ánh xạ level↔enum được ADR hoá; `full` bị CHECK constraint.
- [ ] Approval có `input_hash`; thay đổi lệnh sau duyệt bị chặn; không cached decision cho `prod_deploy/data_deletion/merge_main`.
- [ ] Rate limit approval hoạt động với 18 agent giả lập.
- [ ] Danh sách lệch §12 được các owner xác nhận/đóng trước M1.
