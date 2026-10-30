# PHASE 0 TECHNICAL VALIDATION REPORT

> Ngày: 2026-10-30. Vai trò: Technical Validation Engineer. Phạm vi: chỉ tài liệu, không code ứng dụng, không bắt đầu Phase 1.
> **Trạng thái: KHÔNG có mục nào APPROVED.** Mọi kết luận về Codex, Docker sandbox và concurrency 18 vẫn là CHƯA CHỨNG MINH. Quyết định phê duyệt thuộc về owner.
> Nhãn Evidence: **Verified now** = lệnh read-only chạy trong phiên này, có output; **Not verifiable now** = không thể kiểm chứng nếu không thực hiện hành động bị cấm (cài đặt, bật Docker, chạy prompt model).

## 0. Giới hạn của phiên kiểm tra

Không cài đặt gì, không bật Docker Desktop/daemon, không chạy prompt `claude`/`codex`, không đọc file credential, không đổi cài đặt hệ thống. Chỉ chạy các lệnh trạng thái read-only. Email/tên tổ chức đã được lược bỏ.

## 1. Kết quả kiểm tra read-only

| # | Lệnh | Output (rút gọn, đã redact) | Evidence |
|---|---|---|---|
| 1 | `claude --version` | `2.1.312 (Claude Code)` | Verified now |
| 2 | `claude auth status` (chỉ lọc 4 trường) | `loggedIn: true`, `authMethod: claude.ai`, `apiProvider: firstParty`, `subscriptionType: team` | Verified now |
| 3 | `git --version` | `git version 2.56.0.windows.1` | Verified now |
| 4 | `node -v` | `v24.19.0` | Verified now |
| 5 | `pnpm -v` | `10.34.6` | Verified now |
| 6 | `gh --version` | `gh version 2.102.0 (2026-09-30)` | Verified now |
| 7 | `docker version` | Client OK: `Version 29.8.1`, API 1.56, `Context: desktop-linux`. Server: lỗi `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine; check if the path is correct and if the daemon is running: open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.` | Verified now (client có, **daemon KHÔNG chạy**) |
| 8 | `docker info` | Phần Client liệt kê plugin (buildx v0.37.1, compose, scout v1.24.0, ai, agent). Phần `Server:` cùng lỗi pipe như trên. | Verified now (daemon unreachable) |
| 9 | CPU (`Get-CimInstance Win32_Processor`) | Intel Core i5-11400H @ 2.70GHz, 6 cores, 12 logical processors | Verified now |
| 10 | RAM (`Win32_ComputerSystem`) | Tổng 15.77 GB; free tại thời điểm kiểm tra: 3.91 GB (`Win32_OperatingSystem.FreePhysicalMemory`) | Verified now |
| 11 | Disk D: (`Get-PSDrive D`) | Used 13.71 GB, Free 116.21 GB (~108 GiB) | Verified now |
| 12 | `where codex` | `INFO: Could not find files for the given pattern(s).` Codex **không có** trên PATH | Verified now |
| 13 | Tìm codex.exe trong VS Code extension | `C:\Users\TUYEN\.vscode\extensions\openai.chatgpt-26.51021.22027-win32-x64\bin\windows-x86_64\codex.exe` | Verified now |
| 14 | `codex.exe --version` | `codex-cli 0.164.0-alpha.11.1` | Verified now |
| 15 | `codex.exe login status` | `Logged in using ChatGPT` (không đọc auth.json) | Verified now |
| 16 | `herdr --version` | `herdr 0.9.5` (path: `C:\Users\TUYEN\.herdr\packages\standalone\releases\0.9.5-x86_64-pc-windows-msvc\herdr.exe`) | Verified now |

### 1.1 Tóm tắt trạng thái

| Hạng mục | Trạng thái |
|---|---|
| Claude Code CLI | Có, 2.1.312, đã login (claude.ai, team). Khớp tài liệu research. |
| Codex CLI trên PATH | **Chưa có**. Cài `@openai/codex` là OWNER-GATED. |
| Codex bundled (VS Code ext) | Có, `0.164.0-alpha.11.1`, đã login ChatGPT. Là bản alpha, không phải bản stable; npm latest theo tài liệu là 0.166.0 (số này chưa được kiểm lại ở đây: Not verifiable now). |
| Docker | Client có; **daemon không chạy** (npipe không tồn tại). Mọi test container là Not verifiable now. |
| Máy | 6C/12T, 15.77 GB RAM, free RAM đang chỉ 3.91 GB (thấp hơn 6.4 GB trong tài liệu đo): ảnh hưởng trực tiếp đến mọi ramp concurrency. |
| herdr 0.9.5 | Có. Theo ADR-0020 chỉ là design reference. |

### 1.2 Không verify được trong phiên này

- Codex: tạo session, resume, multi-turn, streaming, đồng thời, giới hạn thật, mã lỗi. Not verifiable now (cần chạy prompt model; bundled binary chỉ xác nhận version + login).
- Docker: isolation, giới hạn tài nguyên, network, credential isolation, kill. Not verifiable now (daemon tắt).
- Claude: mọi số đo runtime (RAM, wall time, cost, 5 song song). Không chạy lại (cấm chạy prompt model).
- Concurrency 18: chưa từng được chạy bởi bất kỳ ai.

## 2. Audit các claim "Đã xác minh" (live test)

Quy ước: **Re-run by QA/validator = No** cho mọi dòng. **Status = Reported_by_agent_not_reproduced** (do research agent báo cáo, chưa tái hiện độc lập). Phần "source evidence" là bằng chứng mà tài liệu tự trích, không phải bằng chứng đã kiểm trong phiên này.

Lưu ý: `RESOURCE_MANAGEMENT.md` nằm ở `docs/architecture/RESOURCE_MANAGEMENT.md`, không phải `docs/research/`.

### 2.1 `docs/research/CLAUDE_CODE_INTEGRATION.md`

| Claim | Con số trong tài liệu | Source evidence tự trích | Re-run | Status |
|---|---|---|---|---|
| Chạy 1 lần `claude -p --output-format json` | wall `real 0m6.091s`, `duration_ms` 1720, `total_cost_usd` 0.375653, `session_id d79c6dbb-...` (rút gọn), model fable + haiku, `costBasis: list` | Block output `time`/JSON dán trong §3.1 | No | Reported_by_agent_not_reproduced |
| 5 process `claude -p` song song, cùng login | `WALL=7s`, 5 session_id (rút gọn), `duration_ms` 1118/2219/1054/1102/1029, không `api_error_status`, model `haiku`, `--max-turns 1` | Block output §3.8 | No | Reported_by_agent_not_reproduced |
| `rate_limit_event` | utilization `five_hour 0.15`, `seven_day 0.74`, `status allowed_warning` ngưỡng 0.75; `resetsAt 1793342400`, `rateLimitType five_hour` | Dòng NDJSON dán trong §3.2 và §6 | No | Reported_by_agent_not_reproduced. Lưu ý: 0.74 < 0.75 nhưng ghi `allowed_warning`: số liệu cần đối chiếu lại (có thể đo ở thời điểm khác nhau). |
| Startup ~3-5 s/process | "~4s" | Suy ra từ timing | No | Reported_by_agent_not_reproduced |
| `--resume`, stream-json multi-turn, `--json-schema`, `canUseTool` deny | OK | Output dán trong các mục §3.x/§4 | No | Reported_by_agent_not_reproduced |
| Quota exhaustion ("You've hit your session limit", `api_retry`) | mô tả hành vi | Dẫn docs/community, không phải lần chạy thật | No | Not verifiable now (chưa từng gặp) |
| "Đã verify: không có lock toàn cục" | chỉ suy ra từ 5 process ngắn | Gián tiếp | No | Reported_by_agent_not_reproduced; phạm vi chỉ 5 process nhẹ |

### 2.2 `docs/research/CODEX_INTEGRATION.md`

| Claim | Con số | Source evidence tự trích | Re-run | Status |
|---|---|---|---|---|
| Binary bundled version | `codex-cli 0.164.0-alpha.11.1` | Lệnh `--version` | **Có, tái hiện trong báo cáo này (mục 1, dòng 14)** | Verified now (chỉ riêng claim version) |
| Codex không trên PATH | KHÔNG có | `Get-Command codex` | **Có tái hiện (dòng 12)** | Verified now |
| Cú pháp/flags `exec`, `app-server` "đã kiểm chứng trên binary 0.164" | help output | `codex exec --help` | No | Reported_by_agent_not_reproduced. Đây chỉ là help text, **không phải test chạy session**. |
| Wall time / RAM / cost / chạy song song Codex | **Không có số đo nào**. Tài liệu không chạy session Codex. Con số quota `used_percent 14.0, window 43200` lấy từ rollout file có sẵn của VS Code (cli_version 0.162), không phải test của agent | Rollout `.jsonl` cũ | No | Reported_by_agent_not_reproduced; thực chất là quan sát log có sẵn |
| "Nhiều thread song song, không giới hạn" | từ docs + cộng đồng ("~2 agent hợp lý cho Plus") | Link blog cộng đồng | No | Not verifiable now |
| Sandbox Windows `elevated` đã setup, MXC đủ điều kiện (build 26300) | suy từ config và `~/.codex/.sandbox/` | Đọc config | No | Reported_by_agent_not_reproduced |

Kết luận: toàn bộ hành vi runtime của Codex (session, resume, streaming, concurrency, lỗi) **chưa được thử**. Không claim nào được coi là đã xác minh.

### 2.3 `docs/research/MULTI_AGENT_SCALABILITY.md`

| Claim | Con số | Source evidence tự trích | Re-run | Status |
|---|---|---|---|---|
| Máy | 15.77 GB RAM, **6.4 GB free lúc đo**, C: 161 GB, D: 108 GB free | Lệnh hệ thống | Một phần (RAM tổng, disk D khớp; free RAM nay 3.91 GB, khác) | Verified now (phần cấu hình tĩnh); Reported_by_agent_not_reproduced (phần free-at-measure) |
| 1 process | wall 5 975 ms; peak RSS 235.2 MB (20 mẫu); CPU time 3.875 s; 0 child; cost 0.3735833; session `805a9d1e-...` | Sampling `Get-Process` mỗi 200 ms | No | Reported_by_agent_not_reproduced |
| 3 process đồng thời | wall 8 172 ms (so với 17.9 s tuần tự, speedup 2.2x); RSS 229.7/230.6/227.5 MB, tổng 685.6 MB; CPU hệ thống đỉnh 79%; cost ~0.3736 mỗi process; 3 session_id | Bảng §4.3 | No | Reported_by_agent_not_reproduced |
| Ngoại suy 5 (1.2 GB nhẹ / 4.5 GB nặng), 18 (4.1 GB nhẹ / 9-16 GB nặng) | 18 x 230 MB ≈ 4.1 GB | Phép nhân từ mẫu 230 MB | No | Not verifiable now; đây là **ngoại suy/giả định** (task thật 500-900 MB là "chưa đo") |
| "18 CLI đồng thời khả thi về OS" | kết luận | Ngoại suy | No | **Không được hỗ trợ bởi bằng chứng**; chưa chạy N>3 |

### 2.4 `docs/architecture/RESOURCE_MANAGEMENT.md`

| Claim | Con số | Source evidence | Re-run | Status |
|---|---|---|---|---|
| §3 "Số đo thực nghiệm [Đã xác minh]" | trùng MULTI_AGENT_SCALABILITY: 5 975 ms, 235.2 MB, 3.875 s CPU, 3 process 8 172 ms, 229.7/230.6/227.5 MB, tổng 685.6 MB | Cùng một lần đo (không độc lập) | No | Reported_by_agent_not_reproduced (không phải nguồn độc lập thứ hai) |
| Bảng 1 (0.23 GB, spawn 0.37) và 3 (0.69 GB, 1.12) agent | [Đã xác minh] | Cùng lần đo | No | Reported_by_agent_not_reproduced |
| Hàng 5/10/18 agent (RAM, CPU, cost 5-90 USD/giờ) | ngoại suy | Giả định A1-A4 do tài liệu tự ghi "chưa đo" | No | Not verifiable now |
| Quota Pro/Max/Team 5h + weekly | [Giả định] | Không tra cứu | No | Not verifiable now |

Nhận xét chung: số liệu Claude trong 3 tài liệu đến từ **một nguồn đo duy nhất** (các agent research cùng phiên). Không có số liệu độc lập. Cost "list" là ước tính client-side, không phải số tiền bị tính phí với subscription.

## 3. Test plans

Quy ước chung:
- Mọi lệnh chạy từ PowerShell trong thư mục scratch riêng, KHÔNG chạy trong repo chính.
- Mọi run lưu log vào `validation-evidence/<TEST-ID>/<timestamp>/` (stdout, stderr, JSONL, `Get-Process` samples, `docker inspect`), kèm hash SHA256 của log.
- Không bao giờ in/commit nội dung `~/.codex/auth.json` hay `~/.claude/.credentials.json`.
- Người chạy ghi lại: version, giờ bắt đầu, % quota trước/sau.
- ToS: chỉ dùng chính tài khoản owner, mỗi tài khoản một người dùng thật; không chia sẻ/rotate credential giữa nhiều người; không dùng login subscription để phục vụ bên thứ ba; kiểm tra điều khoản hiện hành của Anthropic/OpenAI về automation bằng subscription trước khi lên >5 song song (VAL-CONC-08).

### 3a. Codex (VAL-CODEX-xx)

Preconditions chung: owner đã phê duyệt cài Codex (OWNER-GATED), đã login ChatGPT, quota 5h còn > 50% (đọc bằng `codex login status` + `account/rateLimits/read`), thư mục test `D:\Code\Tuyen\validation-scratch\codex` rỗng (git init riêng).

| ID | Mô tả | Owner gate | Quota/cost | Phải pass ở |
|---|---|---|---|---|
| VAL-CODEX-01 | Cài CLI | **Yes** | 0 | Trước Phase 1 (nếu ADR-0005 giữ Codex) |
| VAL-CODEX-02 | Auth status | No (sau 01 hoặc dùng bundled) | 0 | Phase 1 |
| VAL-CODEX-03 | Session create (exec) | Yes (chạy prompt model) | rất thấp (1 turn) | Phase 1 |
| VAL-CODEX-04 | Resume | Yes | thấp | Phase 1 |
| VAL-CODEX-05 | Multi-turn (app-server) | Yes | thấp | Phase 1 |
| VAL-CODEX-06 | Streaming `exec --json` | Yes | thấp | Phase 1 |
| VAL-CODEX-07 | Streaming `app-server` | Yes | thấp | Phase 1 |
| VAL-CODEX-08 | Multi-process behavior 2 -> 3 | Yes | trung bình | Phase 2 (trước tăng concurrency) |
| VAL-CODEX-09 | Ghi limits và lỗi thật | Yes | trung bình | Phase 2 |

**Runbook VAL-CODEX-01: Cài CLI (OWNER-GATED)**
1. Precondition: owner viết "được phép cài @openai/codex" vào DECISION_LOG. Chốt phiên bản (npm latest tài liệu ghi 0.166.0, phải kiểm lại bằng `npm view @openai/codex version`, read-only).
2. Lệnh (chỉ sau phê duyệt): `npm install -g @openai/codex@<version>` hoặc dùng `npx @openai/codex@<version>` (không cài global, ưu tiên). 
3. Kiểm: `codex --version`; `where codex`.
4. Pass: version in ra khớp phiên bản chốt; exit 0. Fail: lỗi npm, version khác.
5. Evidence: output cài đặt, `npm ls -g @openai/codex`.
6. Rollback: `npm uninstall -g @openai/codex`; xác nhận `where codex` rỗng; không đụng `~/.codex`.

**Runbook VAL-CODEX-02: Auth status**
1. `codex login status` (không dùng `codex login` mới).
2. Expected: `Logged in using ChatGPT`. Pass nếu khớp và không có API key env (`$env:OPENAI_API_KEY` rỗng; chỉ kiểm có/không, không in giá trị).
3. Fail: "Not logged in" -> dừng, yêu cầu owner login thủ công.
4. Evidence: output đã redact. Rollback: không cần.

**Runbook VAL-CODEX-03: Session create**
1. Precondition: 01/02 pass, thư mục scratch git repo rỗng.
2. `codex exec --json --skip-git-repo-check -C <scratch> "Reply with exactly: PONG"` (flags phải đối chiếu với `codex exec --help` của đúng version, vì tài liệu ghi flags thay đổi giữa bản).
3. Expected: JSONL có thread/session id, event hoàn tất, exit code 0, nội dung `PONG`.
4. Pass: có `thread_id` hoặc tương đương + exit 0. Fail: exit != 0, không có id, hoặc cần API key.
5. Evidence: JSONL đầy đủ, exit code, rollout file `~/.codex/sessions/...` (chỉ tên file + `session_meta`, không dán nội dung nhạy cảm), %quota trước/sau.
6. Quota: 1 turn model nhỏ. Dùng model nhỏ nhất khả dụng, `--sandbox read-only`.
7. Rollback: xóa thư mục scratch; session file giữ lại làm bằng chứng hoặc xóa theo quyết định owner.

**Runbook VAL-CODEX-04: Resume**
1. Từ 03 lấy `<thread_id>`. Run 1: "Remember secret word MANGO. Reply OK". Run 2: `codex exec resume <thread_id> "What was the secret word? Reply only the word"`.
2. Pass: trả `MANGO`, cùng thread id. Fail: không nhớ, tạo thread mới, hoặc lỗi resume trên Windows.
3. Evidence: 2 JSONL + id. Rollback: như 03.

**Runbook VAL-CODEX-05: Multi-turn (app-server)**
1. `codex app-server` qua stdio (JSON-RPC lines). Gửi `initialize`, `thread/start`, `turn/start` x3 trên cùng thread (ghi tên method chính xác từ bindings `codex app-server generate-ts` của đúng version).
2. Pass: 3 turn hoàn tất tuần tự, `turnId` khác nhau, ngữ cảnh giữ. Fail: treo, mất thread, lỗi schema.
3. Evidence: toàn bộ frame JSON-RPC vào/ra. Rollback: đóng stdin, đảm bảo process thoát (kiểm `Get-Process codex*`).

**Runbook VAL-CODEX-06 và 07: Streaming**
1. 06: `codex exec --json` với prompt yêu cầu câu trả lời dài ~300 từ; ghi timestamp mỗi dòng JSONL.
2. 07: app-server, bắt các notification delta (agent message delta) với timestamp.
3. Pass: >= 5 sự kiện delta tăng dần trước sự kiện completion, độ trễ dòng đầu < 15 s. Fail: chỉ có 1 khối cuối.
4. Evidence: file log có timestamp. Quota thấp.

**Runbook VAL-CODEX-08: Multi-process**
1. Chạy N = 2 rồi 3 `codex exec` đồng thời ở 3 thư mục riêng, cùng login. Giữa các bậc nghỉ >= 2 phút.
2. Mẫu `Get-Process codex*` (RSS, CPU) mỗi 1 s.
3. Pass: tất cả exit 0, session id khác nhau, không lỗi khóa file (`thread-writer-locks`), không `-32001`/429. Fail: bất kỳ lỗi nào (ghi lại, đó là giới hạn thật).
4. Quota: dừng nếu `used_percent` tăng > 5 điểm/bậc hoặc xuất hiện rate-limit.
5. Rollback: kill process còn sót (`taskkill /T /PID`), xóa scratch.

**Runbook VAL-CODEX-09: Limits & errors**
1. Ghi lại: `account/rateLimits/read` (app-server) trước/sau mỗi bậc; thử cố ý prompt lỗi (model không tồn tại, cwd không tồn tại, sandbox deny) để thu mã lỗi.
2. Không cố ý đốt quota đến cạn. Nếu gặp limit tự nhiên: dừng, ghi `resets_at`.
3. Pass: có bảng exit code/lỗi được phân loại thực tế (tài liệu hiện ghi "cần đo sau khi cài"). Fail: lỗi không phân loại được.

### 3b. Docker sandbox (VAL-DOCKER-xx)

Preconditions chung: owner phê duyệt bật Docker Desktop (OWNER-GATED: sẽ dùng RAM, WSL2 backend, có thể làm giảm hiệu năng máy 15.77 GB). Image được pin theo digest. Không bao giờ mount `C:\Users\TUYEN` hay `D:\Code` nguyên thư mục; chỉ mount thư mục scratch.

| ID | Mô tả | Owner gate | Quota/cost | Phải pass ở |
|---|---|---|---|---|
| VAL-DOCKER-01 | Start daemon | **Yes** | 0 (RAM ~2-4 GB host) | Trước Phase 2 (sandbox tier 2, ADR-0007) |
| VAL-DOCKER-02 | Container isolation | Yes (sau 01) | 0 | Phase 2 |
| VAL-DOCKER-03 | Filesystem restriction | Yes | 0 | Phase 2 |
| VAL-DOCKER-04 | Network restriction | Yes | 0 | Phase 2 |
| VAL-DOCKER-05 | Resource limits | Yes | 0 | Phase 2 |
| VAL-DOCKER-06 | Credential isolation | Yes | 0 | Phase 2 |
| VAL-DOCKER-07 | Docker socket không lộ | Yes | 0 | Phase 2 |
| VAL-DOCKER-08 | Privileged/caps qua inspect | Yes | 0 | Phase 2 |
| VAL-DOCKER-09 | Emergency stop | Yes | 0 | Phase 2 |

**VAL-DOCKER-01: Start daemon (OWNER-GATED)**
1. Owner tự mở Docker Desktop (hoặc cho phép agent làm). Chờ engine sẵn sàng.
2. `docker version` và `docker info`. Pass: có phần `Server:`, `OSType: linux`, không lỗi npipe. Fail: lỗi như hiện tại.
3. Ghi RSS của Docker Desktop/vmmem trước và sau (`Get-Process`), free RAM trước và sau.
4. Rollback: `Quit Docker Desktop`; `wsl --shutdown` chỉ khi owner đồng ý; xác nhận free RAM hồi phục.

**VAL-DOCKER-02 đến 08: Cấu hình chuẩn để test**
```
docker run -d --name aiw-val --read-only --tmpfs /tmp:rw,size=64m \
  --cap-drop ALL --security-opt no-new-privileges \
  --cpus 1 --memory 512m --memory-swap 512m --pids-limit 128 \
  --network none --user 10001:10001 \
  -v D:\Code\Tuyen\validation-scratch\ws:/workspace:rw \
  --workdir /workspace <image@sha256:...> sleep 3600
```
- **02 isolation:** trong container `id`, `ps -ef` (chỉ thấy PID container), `hostname`; Pass: không thấy process host. Fail: thấy process host (có `--pid=host`).
- **03 filesystem:** ghi `/etc/x` (phải fail, read-only), ghi `/tmp/x` (ok), ghi `/workspace/x` (ok, hiện trên host), thử đọc `/host`, `C:\`, `/mnt/c` (phải không tồn tại). Pass: chỉ `/workspace` và tmpfs ghi được; không đường dẫn host nào ngoài mount.
- **04 network:** với `--network none`: `getent hosts example.com` và `wget -T3 https://example.com` phải fail. Sau đó chạy biến thể có egress allowlist (proxy/ network riêng) và xác nhận chỉ host cho phép truy cập được; cổng host (`host.docker.internal`) phải bị chặn. Pass/Fail theo từng điểm.
- **05 resource limits:** `docker inspect aiw-val --format "{{.HostConfig.NanoCpus}} {{.HostConfig.Memory}} {{.HostConfig.PidsLimit}}"` -> `1000000000 536870912 128`. Stress: process tiêu RAM > 512 MB phải bị OOM-kill (exit 137, `State.OOMKilled=true`); fork bomb có kiểm soát (`for` tới 200 child) phải bị chặn tại 128 pids; vòng lặp CPU không vượt ~100% 1 core (`docker stats --no-stream`). Chạy stress chỉ trong container, theo dõi free RAM host, dừng nếu free < 1 GB.
- **06 credential isolation:** `docker inspect aiw-val --format "{{json .Mounts}}"` không chứa `.claude`, `.codex`, `.ssh`, `.config`, `.aws`, home user; `docker inspect --format "{{.Config.Env}}"` và `docker exec aiw-val env` không chứa token/key (kiểm bằng pattern `KEY|TOKEN|SECRET|PASSWORD|AUTH`, chỉ in tên biến, không in giá trị). Pass: 0 khớp mount nhạy cảm, 0 biến nhạy cảm. Ghi chú thiết kế (ADR-0008): credential do CLI tự quản; nếu cần login trong container thì dùng cơ chế credential riêng của sandbox, không mount home host (quyết định này cần owner, **chưa verify**).
- **07 Docker socket:** `docker inspect` mounts không chứa `/var/run/docker.sock` hoặc `//./pipe/docker_engine`; trong container `ls -l /var/run/docker.sock` phải not found; `docker` binary không có trong image; không có `DOCKER_HOST` env. Pass: cả 4 đều đúng.
- **08 privileged/caps:** `docker inspect aiw-val --format "{{.HostConfig.Privileged}} {{.HostConfig.CapAdd}} {{.HostConfig.CapDrop}} {{.HostConfig.SecurityOpt}} {{.HostConfig.PidMode}} {{.HostConfig.NetworkMode}} {{.HostConfig.ReadonlyRootfs}} {{.Config.User}}"` -> `false [] [ALL] [no-new-privileges] "" none true 10001:10001`. Trong container `cat /proc/self/status | findstr Cap` (`CapEff` = 0). Pass: khớp toàn bộ. Fail: bất kỳ `Privileged=true`, `CapAdd` khác rỗng, `PidMode=host`, `NetworkMode=host`, user root.

**VAL-DOCKER-09: Emergency stop**
1. Chạy agent giả (vòng lặp) trong container; bên host tiến trình điều khiển giữ PID.
2. Dừng: `docker kill aiw-val` (hoặc `docker stop -t 10` rồi kill); `docker rm -f aiw-val`; `taskkill /PID <host-controller-pid> /T /F` cho cây process host.
3. Pass: container biến mất khỏi `docker ps -a` trong <= 10 s, không còn process con host (`Get-CimInstance Win32_Process` theo ParentProcessId), workspace không bị khóa. Fail: container zombie hoặc process mồ côi.
4. Evidence: thời điểm lệnh, `docker ps -a`, danh sách process trước/sau.
5. Rollback chung: `docker rm -f aiw-val`, `docker network prune`/`docker volume prune` chỉ cho tài nguyên có label `aiw-val`, xóa scratch, thoát Docker Desktop nếu owner muốn.

Caution: các lệnh stress chỉ chạy trong container có limit; không bao giờ `docker system prune -a` (xóa tài nguyên ngoài phạm vi).

### 3c. Concurrency (VAL-CONC-xx)

Phân biệt bắt buộc: **agents created** (bản ghi/cấu hình, không tốn tài nguyên) khác **agents executing** (có process CLI đang chạy một turn). `max_concurrent_agents` chỉ đếm executing (RESOURCE_MANAGEMENT §Schema). Mọi báo cáo phải ghi cả hai con số riêng; "18 agents" tạo ra không phải "18 concurrency".

| ID | Mô tả | Owner gate | Quota/cost | Phải pass ở |
|---|---|---|---|---|
| VAL-CONC-01 | Created vs executing counter | No (không gọi model, dùng stub) | 0 | Phase 1 |
| VAL-CONC-02 | Queue và semaphore | No (stub) | 0 | Phase 1 |
| VAL-CONC-03 | Retry/backoff | No (stub lỗi giả lập) | 0 | Phase 1 |
| VAL-CONC-04 | Cancellation | Yes khi dùng model thật | thấp | Phase 1 |
| VAL-CONC-05 | Ramp N=1 | Yes (model thật) | thấp | Phase 1 |
| VAL-CONC-06 | Ramp N=3 | Yes | thấp-trung bình | Phase 1 |
| VAL-CONC-07 | Ramp N=5 | Yes | trung bình | Phase 1 (mặc định 5 chỉ được dùng khi pass) |
| VAL-CONC-08 | Ramp N=10 | Yes + review ToS | cao | Phase 2 |
| VAL-CONC-09 | Ramp N=18 | Yes + review ToS | rất cao | Phase 3 hoặc muộn hơn; chỉ khi 08 pass |

Cổng khởi chạy mỗi bậc (đều phải thỏa): free RAM >= 1 GB x N + 2 GB dự phòng (hôm nay chỉ 3.91 GB free: **N=5 nhẹ cũng chưa đủ điều kiện nếu không đóng bớt ứng dụng**); CPU 1-phút <= 70%; quota 5h còn >= 60% và 7 ngày <= 50%; bậc trước pass 2 lần liên tiếp.

**VAL-CONC-01 đến 03 (không tốn quota, dùng stub CLI giả lập)**
1. Tạo 18 agent record, 0 execution: `executing = 0`, `created = 18`. Pass: counter đúng.
2. Submit 30 task vào queue với `max_concurrent_agents=5`, stub sleep 5 s. Pass: đồng thời đo được không bao giờ > 5 (sample mỗi 100 ms), thứ tự FIFO/ưu tiên đúng, semaphore trả lại slot khi stub crash.
3. Retry: stub trả lỗi `rate_limit` 2 lần rồi thành công. Pass: backoff tăng (có jitter), tối đa `max_retries`, không retry lỗi không retryable, không đếm 1 task 2 lần.
4. Mọi test ghi task ID, execution ID, log có timestamp.

**VAL-CONC-04 Cancellation (thật)**
1. Bắt đầu 1 task dài (prompt "đếm từ 1 đến 200, mỗi số 1 dòng", `--max-turns 1`).
2. Hủy: gửi interrupt trước (graceful), sau 5 s mới `close()`, sau cùng kill cây process. Với Claude: ghi nhận exit 143 khi SIGTERM; với Codex: ghi nhận kết quả turn interrupt.
3. Pass: process thoát <= 10 s, trạng thái task `cancelled`, không process mồ côi, slot semaphore giải phóng. Fail: zombie, slot kẹt.

**Ramp plan (VAL-CONC-05 đến 09)**

| Bậc | N | Loại tải | Model | Cost cap (list, mỗi bậc) | Quota cap | Stop condition |
|---|---|---|---|---|---|---|
| 1 | 1 | PONG ngắn | haiku/nhỏ nhất, `--max-turns 1` | 1 USD | <= +1% window 5h | bất kỳ lỗi |
| 2 | 3 | PONG ngắn, stagger 3 spawn/2 s | như trên | 3 USD | <= +3% | bất kỳ lỗi, hoặc free RAM < 1.5 GB |
| 3 | 5 | tác vụ nhẹ 1-2 turn | như trên | 8 USD | <= +8% | rate-limit event, `status != allowed`, free RAM < 2 GB |
| 4 | 10 | tác vụ nhẹ | như trên, **chỉ sau review ToS** | 20 USD | <= +15% | như trên + CPU 1-phút > 90% |
| 5 | 18 | tác vụ nhẹ, 2 lần, stagger | như trên | 40 USD | <= +25%, 7 ngày không vượt 0.9 | như trên + bất kỳ OOM/crash |

(Cost cap tính theo `total_cost_usd` list, vì với subscription đây chỉ là chỉ báo tiêu thụ tương đối; **cap quota theo utilization mới là cap thật**. Các cap USD là đề xuất của validator, cần owner xác nhận.)

Dừng toàn bộ ramp ngay khi: `rate_limit_event.status != allowed` (hoặc tương đương Codex `-32001`/429); 7-day utilization > 0.9; free RAM < 1 GB; CPU > 95% trong 30 s; bất kỳ process ngoài phạm vi test bị ảnh hưởng (ví dụ IDE đóng băng); owner yêu cầu. Khi dừng: cancel tất cả execution (theo 04), ghi `resetsAt`, không thử lại trong cùng cửa sổ 5h.

Chạy đầu cửa sổ 5h (quota đầy); nghỉ >= 5 phút giữa các bậc; mỗi bậc lặp 2 lần.

**Evidence bắt buộc mỗi bậc:**
- Danh sách task ID + execution ID + `session_id`/`thread_id` thật (không rút gọn trong file evidence gốc).
- Log stdout/stderr/JSONL từng process, kèm timestamp.
- CSV mẫu `Get-Process` mỗi 1 s (RSS, CPU), free RAM host, số executing đồng thời đo độc lập.
- `rate_limit` trước/sau (Claude `rate_limit_event`; Codex `account/rateLimits`).
- Exit code từng process, số lỗi, số retry.
- Người thực hiện, thời điểm, phiên bản CLI, SHA256 của log.

**Tuyên bố bắt buộc:** *Concurrency 18 KHÔNG được claim (trong README, ADR-0010, báo cáo, UI hay giao tiếp với owner) cho đến khi bằng chứng ở bậc 18 tồn tại, đầy đủ (task ID, session ID, log, mẫu tài nguyên, quota), và đã được người thứ hai xem xét. Hiện tại bằng chứng cao nhất là 3 process đồng thời (báo cáo bởi agent, chưa tái hiện); mức mặc định 5 và mức tối đa 18 trong ADR-0010 là **đề xuất chưa kiểm chứng**.*

Rollback chung: cancel toàn bộ execution, kill cây process còn lại, xóa thư mục scratch, đặt `max_concurrent_agents` về 1, ghi vào KNOWN_ISSUES nếu có sự cố.

## 4. Các việc bị chặn bởi hành động của owner

| # | Hành động owner | Chặn test nào | Rủi ro/Chi phí cho host |
|---|---|---|---|
| O1 | Cho phép cài Codex CLI (`@openai/codex`), hoặc xác nhận dùng bản bundled 0.164.0-alpha.11.1 | VAL-CODEX-01 (và gián tiếp 03-09 nếu cần bản stable) | Thay đổi PATH/npm global |
| O2 | Cho phép chạy prompt model thật bằng Claude/Codex (tiêu quota) | VAL-CODEX-03 đến 09, VAL-CONC-04 đến 09 | Tiêu quota subscription dùng chung với chat |
| O3 | Bật Docker Desktop/daemon | VAL-DOCKER-01 đến 09 | ~2-4 GB RAM; free RAM hiện chỉ 3.91 GB |
| O4 | Xác nhận trần quota/cost mỗi bậc ramp (đề xuất ở 3c) | VAL-CONC-05 đến 09 | Quota 5h và 7 ngày |
| O5 | Review ToS Anthropic/OpenAI cho automation đa tiến trình bằng subscription login | VAL-CONC-08, 09 | Rủi ro tài khoản bị hạn chế |
| O6 | Quyết định cách cấp credential cho container sandbox (không mount home host) | VAL-DOCKER-06, thiết kế ADR-0008/0007 | Rủi ro lộ credential |
| O7 | Giải phóng RAM (đóng IDE/ứng dụng) hoặc nâng RAM >= 32 GB trước ramp >= 10 | VAL-CONC-08, 09 | Free RAM hiện 3.91 GB của 15.77 GB |
| O8 | Chọn người thứ hai tái hiện/duyệt bằng chứng (QA độc lập) | Mọi mục để chuyển khỏi Reported_by_agent_not_reproduced | Quy trình |
| O9 | Phê duyệt cuối cùng cho bất kỳ mục nào | Tất cả | Chỉ owner quyết định; báo cáo này **không** đánh dấu APPROVED |

## 5. Kết luận

- Đã xác minh nay: các phiên bản công cụ, trạng thái login Claude (team) và Codex (ChatGPT), phần cứng, Docker client có nhưng daemon tắt, Codex chưa trên PATH (chỉ có bản bundled alpha), herdr 0.9.5.
- Chưa tái hiện: toàn bộ số đo runtime trong 4 tài liệu (một nguồn đo duy nhất, do agent báo cáo).
- Chưa kiểm chứng: Codex runtime, Docker sandbox, concurrency > 3.
- Trạng thái: **không APPROVED**; các runbook trên là điều kiện để chuyển trạng thái, đợi hành động owner ở mục 4.

## Phụ lục: kết quả PoC được cấp phép (OWR-2 mục 5, ngày 2026-10-30)
Chỉ ba nhóm được cấp phép. Không chạy 27 runbook, không cài Codex, không bật Docker.
| PoC | Báo cáo | PASS | FAIL | BLOCKED | NOT RUN / SIMULATED | Mức bằng chứng |
|---|---|---|---|---|---|---|
| Drizzle (OWR-2 5.4) | `docs/poc/POC_DRIZZLE.md` | 9 | 1 | 1 | 0 | Partial: PGlite 0.5.13 (báo PostgreSQL 18.6), không phải PG16 |
| Dynamic Skill/Resume SK-01..05 (OWR-2 5.5) | `docs/poc/POC_DYNAMIC_SKILL.md` | SK-01, SK-02, SK-03 (có điều kiện), SK-05 (mức guard/CLI) | 0 | 0 | SK-04 SIMULATED (agent thật bị chặn quyền hai lần); SDK NOT TESTED | Partial |
| Docker sandbox (OWR-2 5.3) | `docs/poc/POC_DOCKER_SANDBOX.md` | 0 | 0 | 9 (VAL-DOCKER-01..09) | 0 | Không có. Daemon chưa chạy, chờ Owner tự mở Docker Desktop |
| Conductor PoC (Q-20) | không | 0 | 0 | 0 | NOT RUN | Không nằm trong phạm vi được cấp phép; D-003 giữ APPROVED_CONDITIONAL |

Chi tiết đáng chú ý:
- **Drizzle:** pin `drizzle-orm@0.45.4`, `drizzle-kit@0.31.11` (tôi kiểm lại bằng `npm view`: 0.45.4 là `latest`). FAIL duy nhất: `generate` và `check` không phát hiện thay đổi thủ công trong DB, `pull` có. Giới hạn: unique index trên bảng phân vùng phải chứa khóa phân vùng, nên unique theo `(aggregate_type, aggregate_id, sequence)` phải do ứng dụng bảo đảm. Chưa thử `postgres.js`.
- **Skill/Resume:** `--resume` với file mới trả marker cũ (snapshot được dùng lại). `--system-prompt-snapshot off` trả marker mới. Hash-check cộng session mới trả marker mới và ghi audit. 8 lần gọi model (haiku), tổng 0,006283 USD. SK-04: guard (audit, pause, quarantine) chỉ được thử bằng ghi tệp qua script, chưa bằng agent thật. Hạn chế theo Owner: cùng OS user nên guard chỉ biết thay đổi, thời điểm, phiên thực thi, không biết ai sửa.
- **Docker:** `docker version` cho client 29.8.1 nhưng không kết nối được daemon (pipe `dockerDesktopLinuxEngine` không tồn tại), kiểm tra lúc 13:39. RAM trống 4,04 GB trên 15,77 GB.
- Các PoC kiểm chứng độc lập một phần: tôi chạy lại `npm view` cho phiên bản Drizzle và xác nhận các file đầu ra tồn tại trong thư mục scratch, nhưng **chưa tái chạy** PoC.
