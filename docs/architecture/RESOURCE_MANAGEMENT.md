# RESOURCE_MANAGEMENT.md — Giới hạn đồng thời, ngân sách, giám sát process và quota provider

Tác giả: Agent 13 — Phase 0, chỉ tài liệu. Ngày: 2026-10-30. Trạng thái: DRAFT.
Liên quan: `AGENT_SCHEDULER.md` (ai dùng slot), `AGENT_ORGANIZATION.md` (identity/credential), `docs/research/MULTI_AGENT_SCALABILITY.md` (thực nghiệm chi tiết), `docs/operations/AGENT_OPERATIONS.md` (runbook). Workflow engine: xem `ORCHESTRATION_WORKFLOW_RESEARCH.md`.

Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].

---

## 1. Mục đích

Định nghĩa **Resource Manager**: các tầng giới hạn đồng thời (global / per-provider / per-project / per-agent), ngân sách token & chi phí với hard stop, giám sát process con trên Windows (pid tree, CPU/RAM sampling, emergency kill), timeout, xử lý quota/rate-limit của provider, chế độ suy giảm (degraded), và schema cấu hình. Kèm số đo thực tế và bảng ngoại suy 5/10/18 agent.

## 2. Phạm vi

- Trong: limits, budgets, process monitor, timeouts, provider quota handling, degraded modes, config schema, measurements & extrapolation.
- Ngoài: thuật toán chọn agent (`AGENT_SCHEDULER.md`), sandbox/permission (A09), cách spawn CLI cụ thể (A05/A06), billing UI (A11).

## 3. Số đo thực nghiệm [Tài liệu tự báo cáo, chưa tái hiện] — 2026-10-30

Máy: Acer Nitro AN515-57, Intel Core i5-11400H (6 core / 12 thread, 2.7 GHz), 15.77 GB RAM (6.4 GB free lúc đo), Windows 11 Pro for Workstations 10.0.26300, Node v24.19.0, Claude Code 2.1.312 (`claude.exe` là PE32+ native executable 298 MB — không phải node script; header `MZ`, `file` báo `PE32+ executable, x86-64, console`). Đăng nhập: `authMethod: claude.ai`, `subscriptionType: team`, `apiProvider: firstParty`.

Lệnh: `claude -p "Reply with exactly OK" --output-format json --max-turns 1` trong thư mục scratchpad trống (không có CLAUDE.md dự án).

| Phép đo | Kết quả |
|---|---|
| 1 process — wall time | **5 975 ms** (`duration_api_ms` 3 393, `ttft_ms` 2 009) |
| 1 process — peak RSS (sampling 200 ms, 20 mẫu) | **235.2 MB**; không có child process (tree = chính nó) |
| 1 process — CPU time | 3.875 s CPU (≈ 65% của wall, phần lớn là khởi động/nạp prompt) |
| 1 process — usage | input 2, cache_creation 18 287, cache_read 29 992, output 4; `total_cost_usd` 0.3736 (list price) |
| 3 process đồng thời — tổng wall | **8 172 ms** (so với 3 × 5.98 = 17.9 s tuần tự → scale gần tuyến tính) |
| 3 process — peak RSS từng process | 229.7 / 230.6 / 227.5 MB; tổng đỉnh **685.6 MB** |
| 3 process — CPU hệ thống | mẫu `LoadPercentage`: 79, 51, 42, 26, 22, 13 % (đỉnh lúc cả 3 khởi động) |
| 3 process — lỗi | **Không** rate-limit, không `api_error_status`, `is_error=false` cả 3, exit 0 |
| session_id | `9cf06315-39da-413c-8580-9309e49c07ae`, `c0b4af17-3041-42e9-8745-198b4befda54`, `35f70b04-4b35-4e13-9390-d5671db7235a` (và đơn: `805a9d1e-f2af-4c7d-949e-5aa86dcc0eac`) |
| cost mỗi process | 0.37364 / 0.37364 / 0.37366 USD (list) — tổng 4 lần chạy ≈ **1.49 USD list-price** |

Nhận xét quan trọng:
- `total_cost_usd` có `costBasis: "list"` — là **ước tính theo giá niêm yết API**, không phải tiền thật trừ vào subscription `team`. Với subscription, "chi phí" thật là **quota theo cửa sổ thời gian** [Giả định về cơ chế, xem §8].
- 0.37 USD cho một câu trả lời "OK" là do **~48k token prompt** (18k cache write + 30k cache read): system prompt + toàn bộ skills/MCP/plugins đang cài của user được nạp vào mỗi process. Đây là chi phí cố định mỗi spawn ("spawn tax") và là biến số lớn nhất của cost model (xem §4.3).
- Main model là `claude-fable-5-1` (context 1M, max output 64k), model phụ `claude-haiku-5-5`, `fast_mode_state: off` (`extra_usage_disabled`).

## 4. Các tầng giới hạn đồng thời [Đề xuất]

### 4.1 Thứ tự kiểm (tất cả phải pass mới spawn)

```
global.max_concurrent_agents (default 5, max 18)
  └─ provider_pool[credentialId].max_concurrent (default: claude-code 5, codex 3; D-010 APPROVED 2026-10-30)
       └─ project.max_concurrent (default = global; fair share xem AGENT_SCHEDULER §9)
            └─ agent.max_concurrent_executions (default 1)
                 └─ host guard: free RAM ≥ min_free_ram_mb (2048), CPU 1-min avg ≤ 90%
```

"Đang executing" = `AgentExecution.state ∈ {SPAWNING, RUNNING, STOPPING}`. `PAUSED` và `WAITING_APPROVAL` **không** chiếm slot (process đã dừng) nhưng vẫn chiếm workspace lock.

### 4.2 Semaphore trong Redis (rebuild được từ Postgres)

```ts
// Lua script atomic: acquire tất cả tầng hoặc không gì cả
// KEYS: sem:global, sem:provider:<cred>, sem:project:<p>, sem:agent:<a>
// ARGV: limits tương ứng, leaseToken, ttl
// Trả về 1 nếu acquire; nếu fail trả về tên tầng bị nghẽn để scheduler ghi queuedReason
```

Mỗi acquire tạo `lease:exec:<id>` TTL 60 s; release khi execution kết thúc hoặc lease hết hạn (reaper gom rác mỗi 30 s). Boot: `AGENT_SCHEDULER.md` §11.

### 4.3 Giảm "spawn tax" [Đề xuất] — [Cần người dùng phê duyệt]

Spawn subprocess với môi trường tối giản để không nạp skills/plugins cá nhân của user: `CLAUDE_CONFIG_DIR=<app-managed dir>` chứa `settings.json` riêng (chỉ skills của role, MCP cần thiết), hoặc dùng flag settings/`--strict-mcp-config` nếu A05 xác nhận tồn tại trong 2.1.312 [Giả định]. Kỳ vọng giảm prompt cố định từ ~48k xuống < 15k token/spawn. **Cần đo lại** sau khi A05 chốt cách spawn. Lưu ý: config dir riêng có thể tách khỏi OAuth của user → cần A05 xác nhận credential vẫn dùng được (hoặc copy `.credentials.json`).

## 5. Ngân sách token và chi phí [Đề xuất]

| Cấp | Trường | Mặc định | Hành vi khi chạm |
|---|---|---|---|
| Task | `budgetUsd`, `maxTokens` | theo complexity: 1→0.5, 2→1, 3→3, 4→8, 5→20 USD (list) | **hard stop**: kill process, `FAILED(BUDGET_EXCEEDED)`, không retry |
| Agent (instance) | `dailyBudgetUsd` | 10 USD | agent không nhận task mới tới 00:00 (timezone org); task đang chạy được chạy hết budget task |
| Project | `budgetUsd` (tổng), `dailyBudgetUsd` | không giới hạn / 50 USD | project `paused_by_budget`; mọi task về QUEUED; notify owner |
| Org/global | `dailyBudgetUsd` | 100 USD | như project, cấp toàn cục |

Cơ chế đo: (1) với `--output-format json` cost chỉ biết **sau khi** process kết thúc → chỉ hard-stop được ở ranh giới turn; (2) với `stream-json` (A05) nhận usage theo từng message → kiểm mỗi event, cho phép kill giữa chừng; (3) ước lượng trước spawn: `estimatedCost = spawnTax + complexity × avgCostPerComplexity[agent]`, so với budget còn lại (hard reject trong scoring).

Ledger: bảng `resource_ledger(executionId, ts, tokensIn, tokensOut, cacheRead, cacheWrite, costUsdList, source)` append-only; tổng hợp per task/agent/project/day bằng materialized view (A08).

Lưu ý: vì subscription `team` không tính tiền theo token, **budget USD là proxy** cho quota. Đề xuất bổ sung `budgetTokens` song song và cho owner chọn đơn vị hiển thị. [Cần người dùng phê duyệt]

## 6. Giám sát process (Windows) [Đề xuất]

- Spawn qua `child_process.spawn` với `windowsHide: true`, stdout/stderr pipe, `env` tối giản, `cwd` = workspace của task.
- **Pid tree**: mỗi 5 s chạy `Get-CimInstance Win32_Process -Filter "ParentProcessId=<pid>"` đệ quy (hoặc `wmic process where (ParentProcessId=…)`), lưu danh sách pid con. Thực nghiệm: `claude -p` đơn giản **không** tạo child; khi agent chạy Bash/test sẽ có child (node, git, pwsh) — phải theo dõi để kill đúng.
- **CPU/RAM sampling**: `Get-Process -Id` → `WorkingSet64`, `TotalProcessorTime` delta/interval; lưu 1 mẫu/5 s vào ring buffer 1 giờ, cứ 1 phút ghi 1 dòng vào `execution_metric` (A08). Chi phí sampling 18 process: không đáng kể (<1% CPU, đo bằng vòng lặp 200 ms trong thực nghiệm).
- **Ngưỡng**: RSS execution > `max_rss_mb_per_execution` (default 2048) hoặc tree > 4 GB → cảnh báo; > 2× → kill (`failureClass=CRASH`, retry 1 lần).
- **Emergency stop**: ưu tiên (1) đóng stdin + `process.kill('SIGTERM')` (trên Windows node gửi TerminateProcess → chết ngay, không graceful); (2) `taskkill /PID <pid> /T /F` để diệt cả cây; (3) Windows Job Object (gói `node-windows-job`/native addon hoặc `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) để OS tự diệt cây khi app chết — [Giả định] chưa kiểm thử addon nào trên Node 24; Phase 1 dùng (1)+(2), Job Object là nâng cao.
- Toàn cục: nút "STOP ALL" trên dashboard = lặp (2) cho mọi execution + đặt `global.paused=true` để scheduler không spawn mới.

## 7. Timeouts [Đề xuất]

| Timeout | Default | Ghi chú |
|---|---|---|
| `spawn_timeout` | 60 s | không nhận được event init/first output → FAILED(CRASH) |
| `task_timeout` | theo complexity: 15/30/60/120/240 phút | hard kill, retry nếu attempt < maxRetries |
| `silence_timeout` (stale hard) | 20 phút | không có stdout event/file change — xem `AGENT_SCHEDULER.md` §10 |
| `graceful_stop_timeout` | 10 s | sau SIGTERM trước khi taskkill /F |
| `approval_timeout` | 24 h | WAITING_APPROVAL quá hạn → task về QUEUED và thông báo |
| `tool_command_timeout` | 10 phút | truyền vào CLI (A05: cấu hình Bash timeout qua settings) |

## 8. Quota và rate-limit của provider

### 8.1 Claude (tài khoản hiện tại: claude.ai, `subscriptionType: team`) [Đã xác minh phần đăng nhập]

- [Giả định — chưa xác minh bằng tài liệu trong phiên này; A13 không chạy WebSearch để tránh phụ thuộc kết quả ngoài] Các gói Pro/Max/Team dùng **cửa sổ quota 5 giờ** cộng thêm **giới hạn tuần**; quota tính theo lượng token/tin nhắn tiêu thụ, chia sẻ giữa claude.ai web và Claude Code; gói Max cao hơn Pro ~5× hoặc ~20×. Khi hết quota, CLI trả lỗi dạng "usage limit reached, resets at HH:MM" và `is_error=true`. **Owner cần xác nhận gói cụ thể của tài khoản team và có bật "extra usage" không** (thực nghiệm cho thấy `fast_mode_disabled_reason: extra_usage_disabled`). Nguồn đọc quota: `rate_limit_event.unifiedWindows` (PROVIDER_INTEGRATION §7) [Tài liệu, chưa chạy live].
- Hệ quả thiết kế: 18 agent dùng chung một quota → tổng throughput bị chặn bởi quota, không phải bởi số process. Tăng concurrency chỉ làm **cạn quota nhanh hơn**, không tăng tổng công việc/ngày.
- Rate-limit API (HTTP 429, `retry-after`) với `--output-format json` chỉ xuất hiện sau khi process kết thúc; với `stream-json` có thể bắt sớm hơn (A05 xác nhận).

### 8.2 Codex (ChatGPT plan) [Giả định — CLI chưa cài] Đường đọc quota: `account/rateLimits/read` (bindings app-server 0.164) [Tài liệu, chưa chạy live].

- Plus/Pro/Team có giới hạn số "message" Codex theo cửa sổ 5 giờ và theo tuần; Pro cao hơn Plus nhiều lần. Chi tiết chờ A06 + owner cài CLI và `codex login`.

### 8.3 Phát hiện và phản ứng [Đề xuất]

```ts
function classify(exitCode: number, json?: CliResult, stderr?: string): FailureClass {
  if (json?.is_error && /usage limit|rate limit|429|overloaded/i.test(json.result ?? '')) return 'RATE_LIMIT';
  if (/not logged in|authentication|401|invalid api key/i.test(stderr ?? '')) return 'AUTH';
  if (json?.api_error_status === 429 || json?.api_error_status === 529) return 'RATE_LIMIT';
  if (json?.api_error_status === 401 || json?.api_error_status === 403) return 'AUTH';
  if (exitCode !== 0) return 'CRASH';
  return 'UNKNOWN';
}
```

Khi `RATE_LIMIT`: `provider_pool[cred].state = RATE_LIMITED(until)` với `until` = reset time nếu parse được, nếu không = now + 15 phút (tăng gấp đôi mỗi lần liên tiếp, cap 5 giờ). Mọi task trên pool đó về QUEUED(reason=`PROVIDER_RATE_LIMITED`). **Không** tự chuyển sang credential/provider khác.

## 9. Chế độ suy giảm (degraded modes) [Đề xuất] — [Cần người dùng phê duyệt]

| Tình huống | Hành vi mặc định | Tùy chọn cần approval |
|---|---|---|
| Hết slot global | Queue, hiển thị vị trí hàng đợi | Tăng limit tạm thời (operator, có audit) |
| Provider rate-limited | Queue + countdown tới reset | Chuyển provider (Claude→Codex) **chỉ khi user bấm đồng ý** trên approval request; có thể bật "auto-switch" per project nhưng mặc định tắt |
| Provider DOWN (AUTH/outage) | Mọi task của pool về BLOCKED; notify | Như trên |
| Host thiếu RAM/CPU | Giảm `effective_concurrency` xuống mức thỏa guard (tối thiểu 1) | Tắt host guard (không khuyến nghị) |
| Budget project cạn | Pause project | Tăng budget |
| Task quá budget | Kill + FAILED | Owner nâng budget và retry thủ công |

Lý do không auto-switch: hai provider khác model, khác thói quen tool → kết quả không nhất quán; và dùng tài khoản/quota khác là quyết định tiền bạc của owner.

## 10. Schema cấu hình [Đề xuất]

```ts
// config/resources.schema.ts (zod) — giá trị mặc định
export const ResourceConfig = z.object({
  max_concurrent_agents: z.number().int().min(1).max(18).default(5),   // đếm executing, không đếm created
  max_concurrent_tasks: z.number().int().min(1).default(5),             // ≤ max_concurrent_agents × max agent.executions
  max_retries: z.number().int().min(0).max(10).default(3),
  task_timeout_minutes: z.record(z.enum(['1','2','3','4','5']), z.number()).default({1:15,2:30,3:60,4:120,5:240}),
  silence_timeout_minutes: z.number().default(20),
  graceful_stop_timeout_seconds: z.number().default(10),
  min_free_ram_mb: z.number().default(2048),
  max_cpu_load_percent: z.number().default(90),
  max_rss_mb_per_execution: z.number().default(2048),
  per_agent_limits: z.object({
    max_concurrent_executions: z.number().int().default(1),
    daily_budget_usd: z.number().default(10),
  }),
  per_provider_limits: z.record(z.string(), z.object({          // key: credentialId hoặc provider code
    max_concurrent: z.number().int().default(5),
    rate_limit_cooldown_minutes: z.number().default(15),
    auto_switch_allowed: z.boolean().default(false),
  })),
  per_project_defaults: z.object({ weight: z.number().default(1), max_concurrent: z.number().int().optional(), daily_budget_usd: z.number().default(50) }),
  global_daily_budget_usd: z.number().default(100),
  budget_unit: z.enum(['usd_list','tokens']).default('usd_list'),
});
```

Nguồn: file `config/resources.json` (override bằng env `AIW_RESOURCES_*`) + bảng `org_setting` cho thay đổi runtime từ UI (ưu tiên: DB > env > file > default). Thay đổi `max_concurrent_agents` có hiệu lực ở tick kế tiếp, không giết execution đang chạy nếu giảm.

## 11. Bảng ngoại suy 5 / 10 / 18 agent

Giả định (ghi rõ để owner kiểm): (A1) mỗi process ổn định ở 230 MB khi idle/ngắn; khi làm task thật với context lớn + child process (test runner, dev server) ước **500–900 MB**/agent [Giả định, chưa đo]; (A2) CPU: khởi động chiếm ~3.8 s CPU/process, khi chờ API gần 0%; task thật chạy build/test có thể chiếm 1–2 core; (A3) cost: spawn tax 0.37 USD list hiện tại, giảm xuống ~0.12 nếu tối giản config (§4.3); task thật trung bình 1–3 USD list/giờ/agent với cache [Giả định]; (A4) quota subscription là chặn trên thực tế, chưa biết con số.

| Số agent executing | RAM ước tính (nhẹ / nặng) | CPU | Cost list/giờ (nhẹ / nặng) | Khả thi trên máy này (16 GB, 6C/12T)? |
|---|---|---|---|---|
| 1 (đo) | 0.23 GB | ~0% idle, 65% 1 core lúc khởi động | spawn 0.37 | Có [Tài liệu tự báo cáo, chưa tái hiện] |
| 3 (đo) | 0.69 GB đỉnh | đỉnh 79% toàn hệ thống trong ~1 s | 1.12 cho 3 spawn | Có, không lỗi rate-limit [Tài liệu tự báo cáo, chưa tái hiện] |
| 5 | 1.2 / 4.5 GB | thoải mái; build đồng thời có thể bão hòa 12 thread | 5–15 / 10–25 USD | **Có — khuyến nghị mặc định** |
| 10 | 2.3 / 9 GB | 10 build/test đồng thời sẽ chậm ×2–3 | 10–30 / 20–50 USD | Có với free RAM ≥ 10 GB (hiện chỉ 6.4 GB free → phải đóng app khác); quota là chặn thực tế |
| 18 | 4.1 / 16 GB | bão hòa CPU; latency tool call tăng mạnh | 18–54 / 36–90 USD | **Nhẹ: có; nặng: không** trên 16 GB RAM. Cần ≥ 32 GB hoặc chia agent sang máy/container khác; quota gần như chắc chắn cạn trước |

Kết luận: về OS/process, 18 CLI đồng thời **khả thi** (18 × 230 MB ≈ 4 GB); nút thắt thật là (1) quota subscription dùng chung, (2) RAM khi task nặng, (3) chất lượng (xung đột workspace, review không kịp). Chi tiết: `docs/research/MULTI_AGENT_SCALABILITY.md`.

## 12. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Default 5, max 18, đếm executing | Owner yêu cầu; 5 là mức đo được an toàn về RAM/CPU và hợp với quota chưa rõ | [Đề xuất] trên số đo [Đã xác minh] |
| Semaphore Redis + ledger Postgres | Redis mất được; Postgres là sự thật | [Đề xuất] |
| Hard stop budget ở cấp task | Tránh runaway cost; task là đơn vị nhỏ nhất có thể retry | [Đề xuất] |
| Không auto-switch provider | Quyết định tiền + nhất quán kết quả thuộc owner | [Đề xuất] — [Cần người dùng phê duyệt] |
| taskkill /T /F là cơ chế kill chuẩn Phase 1; Job Object để sau | Chưa kiểm thử addon; taskkill có sẵn | [Đề xuất] |

## 13. Dependencies

A05 (stream-json usage events, flags `--settings`/`--strict-mcp-config`/`CLAUDE_CONFIG_DIR`, thông báo rate-limit thực tế), A06 (Codex limits), A07 (engine đọc QUEUED reason), A08 (ledger, metric tables), A09 (credential storage), A11 (hiển thị budget/queue).

## 14. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Quota 5 giờ cạn giữa sprint → 18 agent đứng im | Dashboard hiển thị reset time; scheduler ưu tiên task critical path khi quota còn < 20% [Giả định về khả năng đọc quota — chưa có API] |
| `total_cost_usd` list price gây hiểu nhầm là tiền thật | UI ghi rõ "ước tính theo giá API niêm yết"; cho chọn đơn vị token |
| Spawn tax 48k token × 18 agent × nhiều lần retry | Tối giản config dir (§4.3); gộp task nhỏ; `max_retries` thấp |
| Kill không sạch cây process → port/dev server treo | Pid tree tracking + taskkill /T; test E2E cho emergency stop |
| Free RAM thực tế chỉ 6.4 GB | Host guard `min_free_ram_mb`; khuyến nghị owner nâng RAM lên 32 GB nếu muốn 18 |

## 15. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Gói Claude của tài khoản team là gì, có bật extra usage không, có chấp nhận dùng API key (pay-as-you-go) cho một số agent không?
2. Có API/CLI nào đọc quota còn lại của subscription không? (chưa thấy trong output JSON; A05 kiểm `/usage` hoặc endpoint). (đã có đường đọc: xem §8.1, 8.2)
3. Đo RAM/CPU của task thật (build + test) — chưa có số, cần Phase 1 benchmark (test plan trong `MULTI_AGENT_SCALABILITY.md` §9).
4. Job Object trên Node 24: chọn addon nào, hay viết PowerShell helper.
5. Codex limits — chờ cài CLI.

## 16. Acceptance Criteria

- [ ] Config schema zod với mặc định như §10; đổi `max_concurrent_agents` từ UI có hiệu lực trong ≤ 2 s (test).
- [ ] Acquire/release atomic: 50 request đồng thời với limit 5 → đúng 5 thành công (test Lua script).
- [ ] Redis flush giữa run → rebuild đúng, không vượt limit (test).
- [ ] Budget task hard stop: giả lập usage vượt → process bị kill ≤ 5 s sau event, `FAILED(BUDGET_EXCEEDED)`.
- [ ] Emergency stop: 5 execution có child (giả lập `node -e setInterval`) → sau 15 s không còn process nào trong cây.
- [ ] Rate-limit giả lập (mock adapter trả 429) → pool RATE_LIMITED, không spawn mới trên pool, task QUEUED với reason; không auto-switch.
- [ ] Ledger ghi đúng usage từ JSON output (so với số liệu thực nghiệm §3 làm fixture).
