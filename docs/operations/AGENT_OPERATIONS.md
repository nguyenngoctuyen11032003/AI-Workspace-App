# AGENT_OPERATIONS.md — Runbook vận hành agent

Tác giả: Agent 13 — Phase 0, chỉ tài liệu. Ngày: 2026-10-30. Trạng thái: DRAFT.
Liên quan: `docs/architecture/AGENT_SCHEDULER.md`, `docs/architecture/RESOURCE_MANAGEMENT.md`, `docs/architecture/AGENT_COLLABORATION.md`, `docs/research/MULTI_AGENT_SCALABILITY.md`. Workflow engine: xem `ORCHESTRATION_WORKFLOW_RESEARCH.md`.

Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].

---

## 1. Mục đích

Hướng dẫn operator (owner hoặc người được ủy quyền) vận hành hàng ngày: khởi động/dừng agent, đọc dashboard, các thao tác điều khiển, xử lý sự cố (agent treo, provider outage, rate limit, chi phí tăng bất thường), quy trình dừng khẩn cấp, và báo cáo hiệu năng theo agent. Runbook mô tả hành vi **mong muốn** của Phase 1; các lệnh thủ công bên dưới dùng được ngay trên Windows hiện tại.

## 2. Phạm vi

- Trong: thao tác operator qua UI/API và fallback bằng PowerShell; incident playbooks; checklist; báo cáo.
- Ngoài: cài đặt môi trường (A14/A05), bảo mật credential (A09), thiết kế UI chi tiết (A11).

## 3. Khái niệm nhanh

- **Agent** = `AgentInstance` (nhân viên AI). **Execution** = một lần agent chạy một task = một CLI subprocess (`claude.exe`; `codex` hoãn sau MVP, OWR-3 A.8).
- **Executing** = SPAWNING/RUNNING/STOPPING — chỉ những execution này chiếm slot `max_concurrent_agents` (default 5, max 18). [OWR-3 A.2/D: 5 là giới hạn scheduler nội bộ, không phải số provider cho phép; không tự tăng lên 10/18. OWR-3 A.15: `WAITING_APPROVAL`/`WAITING_HUMAN` không tính vào 5 active workers khi worker đã giải phóng và state đã persist; quota provider vẫn áp dụng.]
- Nguồn sự thật: Postgres. Redis chỉ là cache/semaphore, xóa được.

## 4. Khởi động và dừng [Đề xuất]

### 4.1 Khởi động hệ thống

1. `docker compose up -d postgres redis` (A07/A08 chốt compose).
2. Kiểm CLI: `claude --version` (kỳ vọng 2.1.312+ [Đã xác minh hiện tại]), `claude auth status` → `loggedIn: true`. Nếu `false` → đăng nhập bằng `claude` tương tác trước, **không** chạy agent.
3. Khởi động API: `pnpm --filter api start` (tên script do Phase 1). Log boot phải có dòng `SchedulerRecovered {adopted, killed, reallocated}` — nếu `killed > 0` xem §8.1.
4. Dashboard: kiểm `Resource` widget: `executing 0/5`, `provider pool claude-code: OK`, `free RAM`.

### 4.2 Bật/tắt agent

- **Enable agent**: Organization → Agent → Enable. Agent chỉ "sẵn sàng nhận task", chưa tạo process. Không tốn tài nguyên.
- **Disable agent**: không nhận task mới; execution đang chạy được chạy hết (hoặc chọn "Disable & stop now" → cancel execution).
- **Tạo instance mới từ template**: xem `AGENT_ORGANIZATION.md` §7; khuyến nghị với dự án nhỏ chỉ enable 5–7 agent.

### 4.3 Dừng hệ thống an toàn

1. Dashboard → **Pause scheduling** (global.paused=true): không spawn mới.
2. Chờ execution đang chạy kết thúc (xem cột runtime) hoặc **Stop** từng cái.
3. Khi `executing = 0` → tắt API. Nếu tắt API khi còn execution: process CLI **vẫn sống** (không có Job Object ở Phase 1) → phải dọn thủ công (§7) hoặc tin vào recovery khi bật lại (mặc định kill + reallocate).

## 5. Dashboard — trường hiển thị cho mỗi agent [Đề xuất]

| Trường | Nguồn | Ghi chú |
|---|---|---|
| Name, avatar | `AgentInstance.displayName` | |
| Role / Department | template | |
| Provider / Model | `providerBinding` + `modelUsage` thực tế từ output JSON (ví dụ `claude-fable-5-1`) [Đã xác minh định dạng] | Hiển thị cả model thật sự dùng, không chỉ cấu hình |
| Current task | `task.id`, title, link | |
| State | execution state + task state | màu: RUNNING xanh, PAUSED/WAITING_APPROVAL vàng, LOST/FAILED đỏ |
| Evidence-based progress | % = checklist AC đã có bằng chứng (file tồn tại, test pass, review PASS) — **không** dùng tự khai của agent | Nguồn: `task_evidence` do orchestrator kiểm |
| Runtime | `now − startedAt`; cảnh báo khi > 80% `task_timeout` | |
| Last log | 1 dòng stream-json gần nhất (tool call/message) + thời điểm; vàng nếu > 5 phút (stale soft) | |
| Tokens / Cost (list) | ledger execution hiện tại + hôm nay | ghi rõ "giá niêm yết" |
| PID / RSS / CPU | process monitor | tree RSS nếu có child |
| Queue position (nếu QUEUED) | scheduler | kèm `queuedReason` ("chờ slot global 5/5", "provider rate-limited tới 14:30") |
| Inbox unread | message bus | |
| Attempt | `task.attempt/maxRetries` | |

Widget toàn cục: `executing N/max`, free RAM, CPU, provider pool state + countdown reset, budget hôm nay (org/project), số task READY/QUEUED/BLOCKED, cảnh báo fairness starvation.

## 6. Thao tác operator [Đề xuất]

| Thao tác | Hiệu ứng | Ghi chú / fallback thủ công |
|---|---|---|
| **Pause** | Dừng process (cancel mềm), lưu `session_id`, state PAUSED; không chiếm slot | Resume bằng `--resume` [Giả định A05]; nếu không hỗ trợ → resume = chạy lại từ đầu với context snapshot |
| **Resume** | Spawn lại khi có slot | |
| **Stop** | SIGTERM → 10 s → `taskkill /PID <pid> /T /F`; task CANCELLED, không retry | Thủ công: `taskkill /PID <pid> /T /F` |
| **Retry** | task về READY, attempt+1 (dù đã FAILED vĩnh viễn — operator override) | Có tùy chọn "retry với agent khác" |
| **Reassign** | Chọn agent đích; scheduler kiểm hard constraints (skills/permission/workspace); ghi REASSIGN | Nếu execution đang chạy → stop trước |
| **View logs** | Mở `logs/<executionId>.jsonl` (stream-json) + stderr; filter theo tool | Thủ công: `Get-Content logs\<id>.jsonl -Tail 50 -Wait` |
| **View changes** | `git diff` trong worktree của task so với base; danh sách file + ownership check | Thủ công: `git -C <worktree> status && git diff --stat` |
| **Open workspace** | Mở thư mục worktree trong Explorer/VS Code | `code <worktree>` |
| **Request review** | Tạo `REQUEST_REVIEW` cho role reviewer với diffRef | Xem `AGENT_COLLABORATION.md` §4 |
| **Approve / Deny** | Giải quyết `APPROVAL_REQUEST` (L4 command, chuyển provider, quyết định họp) | Mọi approve có audit userId. [OWR-3 A.13: "L4 command" là nhãn cũ, cần làm rõ không nhầm với permission level L4 Operator trong L0-L5; xem AGENT_COLLABORATION §4] |
| **Adjust limits** | Đổi `max_concurrent_agents` (1–18), budget | Hiệu lực tick kế tiếp; giảm không giết execution |
| **Send message to agent** | `AGENT_MESSAGE` từ user; giao ở turn kế | Không phải chat realtime |

## 7. Dừng khẩn cấp (Emergency stop) [Đề xuất]

Khi: chi phí tăng bất thường, agent ghi ngoài workspace, provider báo lạm dụng, máy hết RAM.

1. Dashboard → **STOP ALL** (hoặc `POST /ops/emergency-stop`). Hệ thống: `global.paused=true` → với mọi execution: `taskkill /PID <pid> /T /F` → đánh dấu `KILLED(USER_CANCEL)` → release semaphore → ghi `ops_event(EMERGENCY_STOP, userId)`.
2. Nếu API không phản hồi, chạy thủ công (PowerShell, với quyền user hiện tại):
   ```powershell
   # Liệt kê mọi claude/codex process và cây con
   Get-CimInstance Win32_Process | Where-Object { $_.Name -in 'claude.exe','codex.exe','node.exe' } |
     Select-Object ProcessId, ParentProcessId, Name, CommandLine | Format-Table -Wrap
   # Diệt theo cây (lặp cho từng PID gốc của claude.exe do app spawn — KHÔNG giết session Claude Code tương tác của bạn)
   taskkill /PID <pid> /T /F
   ```
   Phân biệt process của app với session cá nhân: app spawn với env `AIW_EXECUTION_ID` và `cwd` trong thư mục workspace của app → kiểm `CommandLine`/`cwd` trước khi kill. [Đề xuất: app thêm `--name`/env tag để nhận diện; A05 xác nhận cách.]
3. Sau khi dừng: kiểm `Get-Process claude -ErrorAction SilentlyContinue` trống; kiểm `git status` các worktree; đọc ledger để biết tổng cost; viết incident note (§8.5).
4. Khởi động lại: bật API → recovery (§4.1) → rà task FAILED/KILLED → quyết định retry từng task (không bật lại tự động).

## 8. Xử lý sự cố [Đề xuất]

### 8.1 Agent treo (stuck)

Dấu hiệu: `Last log` vàng > 5 phút, process sống, CPU ~0, không file change.
1. Mở logs: xem tool call cuối (thường là Bash chờ input, dev server không thoát, test treo).
2. Nếu tool call treo: Stop → kiểm child còn sống không (`Get-CimInstance Win32_Process -Filter "ParentProcessId=<pid>"`) → kill cây.
3. Retry với ghi chú vào task context: "lần trước treo ở lệnh X; dùng timeout/`--run-in-background`".
4. Nếu lặp lại ≥ 2 lần cùng lệnh → BLOCKED, giao PM/user xem lại task.
Hệ thống tự làm bước 2–3 khi `silence_timeout` (20 phút) — operator can thiệp sớm hơn nếu thấy.

### 8.2 Provider outage / AUTH

Dấu hiệu: nhiều execution FAILED(AUTH) hoặc `api_error_status` 5xx; pool `DOWN`.
1. Kiểm `claude auth status`; nếu `loggedIn: false` → đăng nhập lại tương tác; pool tự về OK sau health-check (chạy `claude -p "OK" --max-turns 1` nội bộ, chi phí ~1 spawn).
2. Nếu đăng nhập OK nhưng API lỗi → chờ; task về BLOCKED(provider); **không** tự chuyển provider (Q-08/OWR-2: không auto-failover; thông báo Owner và chờ quyết định) — Codex hoãn khỏi MVP (D-005); nếu sau này có và Owner muốn chuyển, bấm Approve trên approval request từng project.
3. Khi pool OK → task tự quay lại READY; không cần retry tay.

### 8.3 Rate limit / hết quota

Dấu hiệu: `is_error=true` với "usage limit"/429; pool `RATE_LIMITED(until)`.
1. Đọc `until` trên dashboard (parse từ thông báo CLI nếu có; nếu không, cooldown 15 phút tăng dần).
2. Không bấm retry hàng loạt (tốn thêm lần spawn thất bại). Ưu tiên: Pause scheduling, chờ reset.
3. Nếu lặp lại hàng ngày → giảm `max_concurrent_agents`, gộp task nhỏ, giảm spawn tax (RESOURCE_MANAGEMENT §4.3), hoặc owner nâng gói/thêm credential.
4. Ghi vào báo cáo tuần: số lần rate-limit, giờ xảy ra, N executing lúc đó — dữ liệu để quyết định scale (MULTI_AGENT_SCALABILITY §8).

### 8.4 Runaway cost

Dấu hiệu: cost/giờ project vượt 2× trung bình 7 ngày; một execution > budget task; retry vòng lặp.
1. Hệ thống tự: hard stop task vượt budget; pause project khi cạn daily budget.
2. Operator: mở ledger theo execution → tìm execution/agent tốn nhất → View logs → thường là vòng lặp tool (đọc file lặp, test fail-retry) → Stop, giảm `max_retries` cho task, thêm hướng dẫn vào task.
3. Nếu do spawn tax (nhiều task nhỏ) → yêu cầu PM gộp task; kiểm config tối giản có đang áp dụng.
4. Nếu nghi provider tính sai → đối chiếu `usage` trong JSON output với ledger (fixture thực nghiệm: 2/18 287/29 992/4 → 0.3736 USD list [Đã xác minh]).

### 8.5 Incident note (bắt buộc sau emergency stop hoặc sự cố > 30 phút)

`docs/operations/incidents/INC-<yyyymmdd>-<n>.md`: thời gian, dấu hiệu, execution/task id liên quan, hành động, chi phí phát sinh, nguyên nhân gốc, thay đổi cấu hình/quy trình. PM agent có thể soạn draft từ `ops_event` + ledger; operator duyệt.

## 9. Checklist hàng ngày [Đề xuất]

- [ ] Sáng: `claude auth status` OK; free RAM ≥ 4 GB; pool OK; không execution LOST qua đêm.
- [ ] Rà QUEUED có `queuedReason` bất thường (NO_ELIGIBLE_AGENT → thiếu role/skill).
- [ ] Rà approval đang chờ (`WAITING_HUMAN`): chờ tối đa 540 s rồi lưu và chuyển `WAITING_HUMAN` không giữ worker; nhắc sau 24 h; `prod_deploy`/`data_deletion`/`merge_main` có approval riêng hết hiệu lực sau 15 phút (Q-05/OWR-2). OWR-3 A.12: `remind_at` = nhắc sau 24 giờ (không hết hạn); `expires_at` = 15 phút chỉ cho approval hành động quan trọng (prod deploy, data deletion, merge main). X-7: timeout chờ 540 s.
- [ ] Cost hôm qua vs budget; rate-limit events.
- [ ] Tối: Pause scheduling nếu không muốn agent chạy qua đêm (quota 5 giờ reset tốt hơn khi dùng tập trung [Giả định]).

## 10. Báo cáo hiệu năng theo agent [Đề xuất]

Tuần/sprint, tự sinh từ DB (PM agent viết phần nhận xét):

| Chỉ số | Định nghĩa | Nguồn |
|---|---|---|
| Tasks done / failed / cancelled | theo state cuối, theo agent | task, execution |
| Success rate | done / (done + failed vĩnh viễn) | — |
| First-pass review rate | REVIEW PASS ở lần review đầu / tổng review | message bus |
| Avg runtime theo complexity | `endedAt − startedAt` nhóm theo complexity | execution |
| Tokens & cost (list) theo task, theo giờ | ledger | — |
| Spawn tax share | (cache_write + cache_read đầu turn) / tổng token | ledger |
| Stale/timeout count | failureClass STALE/TIMEOUT | execution |
| Rate-limit count (theo pool) | failureClass RATE_LIMIT | — |
| Ownership violations | hook denials | audit (A09) |
| Reassign count | scheduler_event REASSIGN | — |

Báo cáo lưu `docs/performance/AGENT_REPORT_<sprint>.md`; dashboard có view tương đương. Các chỉ số này cũng feed vào `agent.stats` dùng trong scoring (`AGENT_SCHEDULER.md` §5.3).

## 11. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Progress dựa trên evidence, không tự khai | Quy tắc WORK_PLAN §3 "không COMPLETED khi chưa có bằng chứng" | [Đề xuất] |
| STOP ALL = taskkill /T /F toàn bộ | Đơn giản, có sẵn trên Windows; Job Object để sau | [Đề xuất] |
| Không auto-switch provider, không auto-retry khi rate-limit hàng loạt | Tiền/quota là quyết định của owner | Q-08 (OWR-2 2026-10-30): không auto-failover khi hết quota, thông báo Owner và chờ; phần "không auto-retry hàng loạt" vẫn [Đề xuất] |
| Incident note bắt buộc | Tạo dữ liệu cho quyết định scale | [Đề xuất] |

## 12. Dependencies

A05 (`--resume`, nhận diện process, stream-json), A07 (trạng thái workflow hiển thị), A08 (ledger/ops_event), A09 (audit, hook), A11 (dashboard UI), A14 (compose, script start/stop).

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Operator kill nhầm session Claude Code cá nhân khi dọn thủ công | Tag process của app (env/cwd); hướng dẫn kiểm CommandLine trước khi kill |
| Tắt API khi còn execution → process mồ côi | Quy trình §4.3; recovery kill mồ côi; Job Object ở phase sau |
| Dashboard "progress" bị hiểu là tự khai | Nhãn rõ "evidence-based"; tooltip liệt kê evidence |
| Operator retry hàng loạt khi rate-limit | Nút retry-all bị khóa khi pool RATE_LIMITED |

## 14. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Ai là operator ngoài owner? Có cần RBAC operator vs viewer ở Phase 1?
2. Pause/resume thật của CLI (A05) quyết định thao tác Pause có giữ tiến độ hay không.
3. Cách tag process của app để phân biệt với session cá nhân (A05).
4. Thông báo (email/Slack/desktop) cho incident — chưa chọn kênh.
5. Có chạy agent qua đêm không (ảnh hưởng quota và giám sát).

## 15. Acceptance Criteria

- [ ] Mọi thao tác §6 có API endpoint + nút UI + audit log (userId, agentId, executionId).
- [ ] STOP ALL: với 5 execution giả lập có child, sau 15 s không còn process trong cây; semaphore = 0; `ops_event` ghi.
- [ ] Dashboard hiển thị đủ 14 trường §5; `Last log` chuyển vàng sau 5 phút không event (test clock mock).
- [ ] Playbook 8.1–8.4 được diễn tập với mock adapter (stuck, AUTH, 429, budget) và kết quả khớp mô tả.
- [ ] Báo cáo §10 sinh được từ dữ liệu test, số liệu khớp ledger.
- [ ] Incident note template tồn tại tại `docs/operations/incidents/TEMPLATE.md`.
