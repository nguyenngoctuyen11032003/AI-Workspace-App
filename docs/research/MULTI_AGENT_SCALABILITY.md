# MULTI_AGENT_SCALABILITY.md — Môi trường này chạy được 18 CLI agent đồng thời không?

Tác giả: Agent 13 — Phase 0 (research + đo thực tế). Ngày: 2026-10-30. Trạng thái: DRAFT.
Liên quan: `docs/architecture/RESOURCE_MANAGEMENT.md` (áp dụng số đo), `docs/architecture/AGENT_SCHEDULER.md`, `docs/operations/AGENT_OPERATIONS.md`. Workflow engine: xem `ORCHESTRATION_WORKFLOW_RESEARCH.md`.

Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].

---

## 1. Mục đích

Trả lời bằng bằng chứng câu hỏi của owner: *máy hiện tại (Windows 11, 16 GB, i5-11400H) với Claude Code CLI 2.1.312 có thể chạy 18 agent CLI đồng thời không?* Đưa ra mô hình chi phí, giới hạn OS và provider, khuyến nghị mặc định (5) và điều kiện để nâng lên 18, và test plan để chứng minh 18-concurrency bằng session thật, log và task id như owner yêu cầu.

## 2. Phạm vi

- Trong: thực nghiệm 1 và 3 process, tài nguyên máy, giới hạn OS, giới hạn provider (ở mức biết được), cost model, khuyến nghị, rủi ro, test plan.
- Ngoài: thiết kế scheduler/resource manager (đã có file riêng), so sánh workflow engine (A07), đánh giá Codex (A06, CLI chưa cài).

## 3. Phương pháp [Đã xác minh]

- Thư mục chạy: `C:\Users\TUYEN\AppData\Local\Temp\claude\d--Code-Tuyen-Agent-management\9dde48b7-c365-4f17-9f11-5331b00699a6\scratchpad\a13` (và `c1`, `c2`, `c3` cho 3 process). Output JSON lưu tại `single.json`, `concurrent_1.json`, `concurrent_2.json`, `concurrent_3.json` trong thư mục đó.
- Lệnh: `claude -p "Reply with exactly OK" --output-format json --max-turns 1`.
- Đo bằng PowerShell `System.Diagnostics.Process` (Stopwatch cho wall time; `WorkingSet64` sampling 200 ms cho RSS; `Win32_Process` để tìm child; `Win32_Processor.LoadPercentage` cho CPU hệ thống).
- Giới hạn: chỉ chạy tối đa 3 process đồng thời để không đốt quota của user; mọi con số > 3 là ngoại suy có ghi giả định.
- Binary: `C:\Users\TUYEN\.local\bin\claude.exe`, 298 690 208 byte, PE32+ x86-64 console (native executable, không phải node script) — nên RSS không phụ thuộc Node runtime của hệ thống.

## 4. Kết quả đo [Đã xác minh]

### 4.1 Máy

| Thành phần | Giá trị |
|---|---|
| Model | Acer Nitro AN515-57 (laptop) |
| CPU | Intel Core i5-11400H, 6 cores / 12 logical, 2.69 GHz base |
| RAM | 15.77 GB tổng; **6.4 GB free** tại thời điểm đo (đang mở IDE + session Claude Code chính + 10+ agent nghiên cứu) |
| Disk | C: 161 GB free; D: 108 GB free |
| OS | Windows 11 Pro for Workstations 10.0.26300 |
| Node | v24.19.0; Docker Desktop có |
| CLI | Claude Code 2.1.312, logged in (claude.ai OAuth, org "INTERNATIONAL CYBER SECURITY - ICS. JCS", `subscriptionType: team`) |

### 4.2 Một process

| Chỉ số | Giá trị |
|---|---|
| Wall time | 5 975 ms |
| `duration_api_ms` / `ttft_ms` / `duration_ms` (CLI tự báo) | 3 393 / 2 009 / 2 043 ms |
| Peak RSS | 235.2 MB (20 mẫu); `PeakWorkingSet64` 235.2 MB |
| Child process | 0 |
| CPU time | 3.875 s |
| Exit code | 0; `is_error: false`; `result: "OK"` |
| session_id | `805a9d1e-f2af-4c7d-949e-5aa86dcc0eac` |
| usage | input 2, cache_creation 18 287, cache_read 29 992, output 4; `service_tier: standard` |
| total_cost_usd | 0.3735833 (`costBasis: list`) — model `claude-fable-5-1` 0.373458 + `claude-haiku-5-5` 0.0001253 |

### 4.3 Ba process đồng thời

| Chỉ số | P1 | P2 | P3 |
|---|---|---|---|
| session_id | `9cf06315-39da-413c-8580-9309e49c07ae` | `c0b4af17-3041-42e9-8745-198b4befda54` | `35f70b04-4b35-4e13-9390-d5671db7235a` |
| Peak RSS (MB) | 229.7 | 230.6 | 227.5 |
| CPU time (s) | 3.81 | 3.50 | 3.56 |
| `duration_ms` / `duration_api_ms` | 1 818 / 2 853 | 3 415 / 3 979 | 1 597 / 2 203 |
| cost (list) | 0.3736433 | 0.3736433 | 0.3736633 |
| usage | 2 / 18 290 / 29 992 / 4 | 2 / 18 290 / 29 992 / 4 | 2 / 18 291 / 29 992 / 4 |
| lỗi | không | không | không |

Tổng: wall **8 172 ms** cho cả 3 (vs 17.9 s nếu tuần tự → speedup 2.2×); tổng RSS đỉnh **685.6 MB**; CPU hệ thống đỉnh **79%** trong giây đầu (3 process khởi động cùng lúc), rồi 51 → 42 → 26 → 22 → 13%. **Không** có rate-limit, 429, hay `api_error_status`.

### 4.4 Diễn giải

1. **Khởi động là phần tốn CPU nhất** (~3.5–3.9 s CPU/process, tương đương một core bận gần hết trong lúc nạp). Spawn 18 process cùng một lúc sẽ bão hòa 12 thread trong ~6–10 s → scheduler nên **stagger spawn** (ví dụ tối đa 3 spawn/2 s).
2. **RAM ổn định ~230 MB/process** cho session ngắn, không có child. Task thật (đọc nhiều file, context 100k+ token, chạy test) sẽ cao hơn — chưa đo.
3. **Chi phí cố định/spawn ~48k token** (18k cache write + 30k cache read): system prompt + skills/plugins/MCP của môi trường user. Đây là biến lớn nhất của cost model và **có thể giảm** bằng config dir tối giản (RESOURCE_MANAGEMENT §4.3) — cần đo lại sau khi A05 chốt cách spawn.
4. Cache đọc 29 992 token giống hệt ở cả 4 lần → prompt cache của provider hoạt động xuyên process; 3 process đồng thời không làm hỏng cache.
5. `total_cost_usd` là **giá niêm yết API**, không phải tiền trừ subscription. Với gói team, chi phí thật là quota.

## 5. Giới hạn OS/host [Đề xuất dựa trên số đo]

| Yếu tố | 18 process nhẹ | 18 process nặng (task thật) | Ghi chú |
|---|---|---|---|
| RAM | 18 × 230 MB ≈ 4.1 GB — **vừa** 6.4 GB free | 18 × 0.5–0.9 GB ≈ 9–16 GB — **không vừa** 16 GB tổng khi còn IDE/Docker | Host guard `min_free_ram_mb` bắt buộc |
| CPU | khởi động đồng loạt bão hòa; sau đó phần lớn chờ API | build/test song song của 18 agent sẽ chậm ×2–3 | stagger spawn; giới hạn `max_concurrent_builds` (đề xuất 3) |
| Process/handle | Windows không giới hạn đáng kể ở quy mô 18 process + vài trăm child | như vậy | — |
| Disk I/O | 18 worktree × repo size; git worktree rẻ | test runner ghi nhiều | SSD hiện tại đủ |
| Network | 18 kết nối HTTPS streaming | như vậy | không đáng kể |
| Docker (nếu sandbox) | 18 container × overhead ~50–100 MB + CLI bên trong | cộng dồn | Phase 1 chạy process trực tiếp; container là tùy chọn A09 |

Kết luận OS: **chạy 18 process CLI đồng thời là khả thi về mặt OS** trên máy này cho tác vụ nhẹ–trung bình; với tác vụ nặng cần **≥ 32 GB RAM** hoặc chia bớt sang máy/container khác.

## 6. Giới hạn provider

- Tài khoản hiện tại: claude.ai OAuth, `subscriptionType: team` [Đã xác minh]. Mọi agent dùng chung một credential → **một quota chung**.
- [Giả định — cần owner xác nhận gói và chính sách; A13 không xác minh bằng tài liệu trong phiên này] Gói subscription của Claude áp dụng giới hạn theo cửa sổ 5 giờ và giới hạn tuần, tính theo mức sử dụng token; khi chạm, CLI báo lỗi "usage limit" và thời điểm reset. Không có API công khai để đọc quota còn lại (chưa thấy trong output JSON).
- Thực nghiệm 3 process đồng thời **không** thấy giới hạn số session đồng thời. Có thể tồn tại giới hạn concurrent request ở phía API cho subscription [Giả định] — chỉ biết khi chạy test 18 (§9).
- Codex: CLI chưa cài; giới hạn ChatGPT plan [Giả định] theo cửa sổ 5 giờ + tuần. Chờ A06.
- Hệ quả: **tổng công việc/ngày bị chặn bởi quota, không phải bởi số agent.** 18 agent chỉ có ý nghĩa nếu quota đủ (gói Max cao/nhiều credential/API key).

## 7. Cost model [Đề xuất]

```
cost_per_execution ≈ spawn_tax + Σ_turns (input_new × p_in + cache_read × p_cache_read + cache_write × p_cache_write + output × p_out)
spawn_tax(hiện tại) ≈ 0.37 USD list   (18k cache write + 30k cache read)
spawn_tax(tối giản, mục tiêu) ≈ 0.10–0.15 USD list   [Giả định]
```

Ước tính theo giờ cho một agent làm task thật (complexity 3, ~10 turn, context tăng dần tới 150k, cache tốt) ≈ **1–3 USD list/giờ** [Giả định — chưa đo task thật]. Tổng cho N agent chạy liên tục:

| N | Spawn tax/giờ (giả định 2 execution/agent/giờ) | Chi phí làm việc/giờ | Tổng list/giờ | Tổng list/ngày 8h |
|---|---|---|---|---|
| 5 | 3.7 | 5–15 | ≈ 9–19 | 70–150 |
| 10 | 7.4 | 10–30 | ≈ 17–37 | 140–300 |
| 18 | 13.4 | 18–54 | ≈ 31–67 | 250–540 |

Với subscription, các con số này là **tương đương quota** chứ không phải hóa đơn; chúng cho thấy 18 agent chạy liên tục gần như chắc chắn vượt quota 5 giờ của bất kỳ gói cá nhân nào [Giả định về mức quota].

## 8. Khuyến nghị [Đề xuất]

**Mặc định `max_concurrent_agents = 5`** vì:
1. Đã đo 3 đồng thời an toàn; 5 ≈ 1.2 GB RAM nhẹ / ≤ 4.5 GB nặng — vừa với 6.4 GB free hiện tại.
2. CPU khởi động 5 process ≈ 20 s CPU, chia trên 12 thread → không nghẽn.
3. Quota chưa rõ; 5 là mức cho phép quan sát tốc độ cạn quota trước khi tăng.
4. Chất lượng cộng tác: với 1 reviewer và 1 PM, > 5 dev song song tạo backlog review không kịp.

**Điều kiện để nâng lên 10 → 18** (từng bước, mỗi bước chạy test §9):
- RAM free ≥ 1 GB × N (nhẹ) hoặc ≥ 32 GB tổng (nặng); host guard bật.
- Quota: chạy 1 ngày ở N hiện tại **không** chạm rate-limit; hoặc có ≥ 2 credential pool/API key.
- Spawn tax đã giảm (< 20k token/spawn) — nếu không, cost ×18 không hợp lý.
- Workspace: mỗi agent có worktree riêng; ownership pattern không chồng lấn trong batch đang chạy.
- Review throughput: ≥ 2 reviewer instance khi ≥ 10 dev agent.
- Stagger spawn ≤ 3 process / 2 s.

## 9. Test plan chứng minh 18-concurrency bằng session thật [Đề xuất] — [Cần người dùng phê duyệt trước khi chạy vì tốn quota]

Mục tiêu: bằng chứng gồm **18 session_id thật, 18 task id, log, số đo RAM/CPU, không rate-limit** — đúng yêu cầu owner.

| Bước | Nội dung | Evidence thu |
|---|---|---|
| T0 | Chuẩn bị: config dir tối giản (A05), 18 thư mục workspace `bench/ws-01..18`, script PowerShell `bench/run-concurrency.ps1` nhận tham số N, prompt nhẹ ("Reply OK") và prompt trung bình ("đọc file X, tóm tắt 5 dòng") | script + commit |
| T1 | N=5 nhẹ, 3 lần | 15 JSON (session_id, cost, usage), bảng RSS/CPU, wall time |
| T2 | N=10 nhẹ, 2 lần | như trên |
| T3 | N=18 nhẹ, 2 lần, stagger 3/2s | như trên + sample `Get-Process` mỗi 1 s suốt run |
| T4 | N=18 trung bình (prompt đọc file, 3 turn) | như trên + so sánh `duration_api_ms` để phát hiện throttling |
| T5 | Chạy qua Agent Scheduler thật (Phase 1): 18 task READY, limit 18 → ghi `task.id`, `execution.id`, `providerSessionId`, `scheduler_event`, log file | DB export + log |
| T6 | Chaos: kill ngẫu nhiên 3 process giữa T5 → kiểm reallocation, semaphore đúng | scheduler_event REASSIGN, không spawn đôi |
| T7 | Báo cáo `docs/performance/CONCURRENCY_BENCH_<date>.md` với bảng kết quả, link JSON, pass/fail theo AC §13 | tài liệu |

Dự toán quota: T1–T4 ≈ 5×3 + 10×2 + 18×2 + 18 = 89 spawn nhẹ ≈ 33 USD list ở spawn tax hiện tại, ≈ 11 USD nếu tối giản. **Owner quyết định có chạy hay không và khi nào** (nên chạy đầu cửa sổ quota).

## 10. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Chỉ đo tới 3 đồng thời trong Phase 0 | Bảo vệ quota của user theo chỉ thị | [Đã xác minh] |
| Default 5, max 18 | Số đo + quota chưa rõ + review throughput | [Đề xuất] |
| 18 chỉ bật sau test §9 từng bước | Owner yêu cầu bằng chứng session thật | [Đề xuất] |
| Spawn tax là KPI phải giảm trước khi scale | 48k token/spawn là chi phí lớn nhất đo được | [Đã xác minh] |

## 11. Dependencies

A05 (cách spawn tối giản, stream-json, thông điệp rate-limit thật), A06 (Codex limits), A07 (engine để chạy T5), A08 (bảng ghi evidence), A09 (sandbox ảnh hưởng RAM nếu container), owner (gói subscription, phê duyệt chạy test).

## 12. Rủi ro

| Rủi ro | Ảnh hưởng | Giảm thiểu |
|---|---|---|
| Quota cạn giữa test 18 | Mất bằng chứng, đứng việc | Chạy đầu cửa sổ; dừng ngay khi thấy `RATE_LIMIT`; ghi reset time |
| Số đo task nhẹ không đại diện task nặng | RAM ước tính sai | T4 + benchmark task thật Phase 1 |
| Free RAM 6.4 GB biến động theo app khác | Host guard kích hoạt bất ngờ | Khuyến nghị đóng IDE phụ khi chạy > 10; nâng RAM |
| Provider thay đổi chính sách concurrency | Kết quả hôm nay không còn đúng | Lưu version CLI + ngày đo trong mọi báo cáo |
| PID reuse / zombie khi kill 18 process | Semaphore sai | Rebuild từ Postgres (scheduler §11) |

## 13. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Gói subscription chính xác (team: seat loại nào? có extra usage?) và có sẵn sàng dùng API key cho benchmark không?
2. [Cần người dùng phê duyệt] Cho phép chạy test §9 (≈ 89 spawn)? Thời điểm?
3. Chưa đo RAM/CPU cho task thật (build/test) — Phase 1.
4. Có giới hạn concurrent request ẩn ở phía API cho OAuth subscription không — chỉ biết khi chạy N=18.
5. Codex hoàn toàn chưa đo.

## 14. Acceptance Criteria (cho kết luận "18 khả thi")

- [ ] T3: 18 process, 2 lần, 0 lỗi, 0 rate-limit, 18 session_id khác nhau mỗi lần, tổng RSS đỉnh < 70% RAM free tại thời điểm chạy, wall time < 3× wall time của N=1.
- [ ] T4: `duration_api_ms` trung bình N=18 không vượt 2× N=1 (không throttling rõ rệt).
- [ ] T5: 18 `task.id` ↔ 18 `execution.id` ↔ 18 `providerSessionId` trong DB và log file khớp nhau.
- [ ] T6: sau chaos, `COUNT(RUNNING)` = giá trị semaphore; mọi task bị kill được reallocate với attempt+1.
- [ ] Báo cáo T7 có đủ bảng, link evidence, ngày, version CLI, gói subscription.
- [ ] Nếu bất kỳ AC nào fail → max hiệu dụng = N lớn nhất pass, ghi vào `RESOURCE_MANAGEMENT.md` §10 default.
