# SANDBOX DESIGN — Tier 1 & Tier 2

> Security Architect · Phase 0 (documentation only) · 2026-10-30 · Trạng thái: PROPOSAL v1
> Mở rộng [`../SECURITY.md`](../SECURITY.md) §2, §3.5, §6; hợp nhất với `docs/architecture/INFRASTRUCTURE_DESIGN.md` §6 (Sandbox Executor). Không lặp lại lập luận Windows-vs-WSL2 đã có ở SECURITY §2.2.
> Nhãn: **[Đã xác minh]** · **[Đề xuất]** · **[Giả định]** · **[Cần người dùng phê duyệt]**.
>
> **Lưu ý quan trọng:** Docker daemon **không chạy** lúc khảo sát (named pipe `dockerDesktopLinuxEngine` không tồn tại; Docker client 29.8.1, Compose 5.5.1) [Đã xác minh]. Mọi thông số về cgroup v2, seccomp profile mặc định, kernel WSL2, hành vi `--read-only`/`--tmpfs`, hiệu năng bind mount, ECI đều là **[Giả định]** cho đến khi chạy `docker info`, `docker inspect` thật (xem §11 checklist).

## 1. Mục đích

Đặc tả đủ chi tiết để Phase 1 hiện thực `packages/sandbox` và hook rewrite mà không phải đoán: tham số container chính xác, image policy, proxy allowlist, mount, giới hạn tài nguyên, command rewriting, kill process tree trên Windows, trình tự emergency stop, ranh giới những gì permission CLI không enforce được, và checklist xác minh.

## 2. Phạm vi

- Trong: Tier 1 (CLI trên host + thực thi trong container), Tier 2 (toàn bộ agent runtime trong container), egress proxy, volume, limits, rewrite, kill, emergency stop, verification.
- Ngoài: chọn đồng bộ workspace cuối cùng (host-truth vs volume — thuộc Agent Runtime team, SECURITY §2.3 mục 6), hạ tầng preview (INFRASTRUCTURE_DESIGN §7), Codex sandbox (R10).

## 3. Mô hình và thuật ngữ

| Thuật ngữ | Nghĩa |
| --- | --- |
| Policy layer | permission rules + PreToolUse hook của CLI; có thể bị bypass khi bug/obfuscation |
| Boundary | VM WSL2 + container config; chỉ Docker mới là boundary thật trên Windows |
| `aiw-exec` | wrapper do app cung cấp, nhận `<project> -- <argv>` và gọi Docker Engine API (không dùng shell) |
| Exec container | container Tier 1 chạy code sinh ra |
| Agent container | container Tier 2 chạy cả CLI |

**Khác biệt cần hoà giải với INFRASTRUCTURE_DESIGN:** SECURITY.md dùng 1 container sống lâu per project (`aiw-<projectId>`, `docker exec`), INFRA §6.1 dùng ephemeral container per job. Đề xuất hợp nhất: **container per project cho phase dev/build/test xen kẽ nhiều lệnh (warm), bị recreate khi project đổi trust level, mỗi 4 giờ, hoặc sau mọi job `install`**; container `preview` và `custom` Dockerfile luôn ephemeral. [Đề xuất, Cần người dùng phê duyệt]

## 4. Tier 1 — Exec container

### 4.1 Tham số `docker run` (CLI tham chiếu)

```bash
docker run -d --name aiw-<projectId> \
  --label aiw=1 --label aiw.project=<projectId> --label aiw.tier=1 --label aiw.policy=sha256:<hash> \
  --user 1000:1000 \
  --read-only \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --security-opt seccomp=default \
  --pids-limit 512 \
  --memory 4g --memory-swap 4g --memory-reservation 2g \
  --cpus 2 \
  --ulimit nofile=4096:8192 --ulimit nproc=512 --ulimit core=0 \
  --tmpfs /tmp:rw,noexec,nosuid,nodev,size=512m \
  --tmpfs /home/agent/.cache:rw,nosuid,nodev,size=1g \
  --mount type=volume,src=aiw-<projectId>-ws,dst=/workspace \
  --mount type=volume,src=aiw-<projectId>-nm,dst=/workspace/node_modules \
  --network none \
  --dns 0.0.0.0 \
  --init --stop-timeout 10 \
  --log-driver local --log-opt max-size=10m --log-opt max-file=3 \
  --workdir /workspace \
  aiw-sandbox-node@sha256:<digest> sleep infinity
```

Ghi chú:
- `--tmpfs /tmp ... noexec` có thể làm hỏng toolchain ghi binary vào `/tmp` (esbuild, swc): cần test; phương án dự phòng `TMPDIR=/workspace/.tmp`. [Giả định]
- `--dns 0.0.0.0` chỉ có nghĩa khi có network; với `--network none` giữ để tránh resolver lạ khi sau này `docker network connect`. Phase install dùng mục 4.4.
- `--security-opt seccomp=default` là profile mặc định của Docker; không bao giờ `seccomp=unconfined`, `apparmor=unconfined`. Docker Desktop (WSL2) có thể không enforce AppArmor [Giả định].
- Không dùng `--privileged`, `--pid=host`, `--ipc=host`, `--userns=host`, `--network host`, `--cap-add`, `--device`, `--volume /var/run/docker.sock`, `//./pipe/docker_engine`, bind mount `C:\Users\*`.

### 4.2 Cấu hình dockerode tương đương

```ts
// packages/sandbox/src/create-container.ts  (sketch, không phải code chạy)
const spec: Docker.ContainerCreateOptions = {
  name: `aiw-${projectId}`,
  Image: `aiw-sandbox-node@sha256:${digest}`,
  User: '1000:1000',
  WorkingDir: '/workspace',
  Cmd: ['sleep', 'infinity'],
  Labels: { aiw: '1', 'aiw.project': projectId, 'aiw.tier': '1', 'aiw.policy': policyHash },
  Env: [],                                   // rỗng; env từng lệnh đi qua exec (mục 4.7)
  NetworkDisabled: false,                    // dùng NetworkMode 'none' bên dưới
  HostConfig: {
    NetworkMode: 'none',
    ReadonlyRootfs: true,
    CapDrop: ['ALL'],
    SecurityOpt: ['no-new-privileges:true'],
    Privileged: false,
    PidsLimit: 512,
    Memory: 4 * 1024 ** 3,
    MemorySwap: 4 * 1024 ** 3,
    MemoryReservation: 2 * 1024 ** 3,
    NanoCpus: 2_000_000_000,
    Init: true,
    AutoRemove: false,
    Ulimits: [{ Name: 'nofile', Soft: 4096, Hard: 8192 }, { Name: 'nproc', Soft: 512, Hard: 512 }, { Name: 'core', Soft: 0, Hard: 0 }],
    Tmpfs: { '/tmp': 'rw,noexec,nosuid,nodev,size=512m', '/home/agent/.cache': 'rw,nosuid,nodev,size=1g' },
    Mounts: [
      { Type: 'volume', Source: `aiw-${projectId}-ws`, Target: '/workspace' },
      { Type: 'volume', Source: `aiw-${projectId}-nm`, Target: '/workspace/node_modules' },
    ],
    LogConfig: { Type: 'local', Config: { 'max-size': '10m', 'max-file': '3' } },
    StopTimeout: 10,
    // KHÔNG đặt: Binds, Devices, PidMode, IpcMode, UsernsMode, CapAdd, ExtraHosts
  },
};
```

`createContainer` đi qua hàm `assertSafeSpec(spec)` (deny-by-default) từ chối mọi khoá không nằm trong allowlist ở trên; cùng hàm đó chạy lại trên kết quả `container.inspect()` trước mỗi `exec` (fail-closed nếu lệch). Xem checklist SB-01..SB-07.

### 4.3 Image policy

| Quy tắc | Chi tiết |
| --- | --- |
| Nguồn | Chỉ image `aiw-sandbox-<stack>` do platform build từ `infra/images/<stack>/Dockerfile` (INFRA §6.3); agent không được `docker pull/build` (denylist §3.1) |
| Pin | Tham chiếu bằng digest `@sha256:`; bản đồ `stack → digest` nằm trong file policy ký bởi app, không trong worktree |
| Người dùng | `USER 1000:1000`, không có `sudo`, không `setuid` binary (loại `chmod u-s` trong Dockerfile, `find / -perm -4000` phải rỗng) |
| Nội dung | multi-stage, không chứa compiler thừa, không chứa `curl/wget/nc/ssh/docker` nếu toolchain không cần; có `git`, `ca-certificates`, shell tối thiểu |
| Scan | Trivy HIGH/CRITICAL ⇒ fail CI; rebuild hằng tuần; SBOM đính kèm (DEVSECOPS §6) |
| `custom` Dockerfile của project | Chỉ build ở cấp host sau approval; sau build chạy với cùng bộ limit; compose file được parse tĩnh, từ chối `privileged`, `network_mode: host`, bind ngoài worktree, `cap_add`, `devices` |
| Registry | Không cho chạy image tag `latest`; không `--pull always` từ registry công cộng lúc runtime |

### 4.4 Network: proxy allowlist

Mặc định `--network none`. Chỉ phase `install` (và job được policy gắn `network: egress-allowlist`) nối container vào network `aiw-egress`.

```
aiw-egress  (docker network create --internal --driver bridge aiw-egress)
  └─ aiw-proxy (Squid hoặc tinyproxy; có thêm network aiw-out có route ra ngoài)
exec container: docker network connect aiw-egress  →  chỉ route tới aiw-proxy
env: HTTP_PROXY=HTTPS_PROXY=http://aiw-proxy:3128, npm_config_proxy, PIP_PROXY, NO_PROXY=localhost
```

| Domain allowlist (match theo CONNECT hostname, SNI không giải mã TLS) | Dùng cho | Phase |
| --- | --- | --- |
| `registry.npmjs.org`, `registry.yarnpkg.com` | npm/pnpm/yarn | install |
| `pypi.org`, `files.pythonhosted.org` | pip/uv | install |
| `crates.io`, `static.crates.io`, `index.crates.io` | cargo | install |
| `proxy.golang.org`, `sum.golang.org` | go | install |
| `repo.maven.apache.org` | maven | install |
| `github.com`, `objects.githubusercontent.com`, `codeload.github.com` | git dependency, tarball | install (cần approval nếu untrusted) |

Quy tắc: (1) chỉ cổng 443, method `CONNECT`; (2) cấm IP literal, cấm `.internal`, `169.254.0.0/16`, `host.docker.internal`, RFC1918; (3) log mọi request (domain, bytes, project) vào audit category `container`; (4) DNS do proxy phân giải, container không có resolver riêng; (5) sau install, `docker network disconnect aiw-egress` rồi `connect none` — không để container giữ network khi chạy test; (6) giới hạn băng thông 20 MB/s, tổng 2 GB/job, timeout 10 phút; (7) cấm response lớn hơn 500 MB.

Hạn chế đã biết: proxy theo hostname không chặn **domain fronting** hoặc upload dữ liệu lên `github.com`/registry hợp lệ (exfil tới repo công khai). Giảm thiểu: chặn `POST/PUT` tới registry trừ `npm publish` (đã deny), kích thước request body ≤ 64 KB trừ khi tải dependency; residual **R6**.

### 4.5 Volume mount

| Mount | Nguồn | Đích | Chế độ | Ghi chú |
| --- | --- | --- | --- | --- |
| Workspace | named volume `aiw-<pid>-ws` hoặc đồng bộ một chiều host→volume (SECURITY §2.3 mục 6) | `/workspace` | rw | Không bind `D:\` trực tiếp ở mặc định (9P chậm, junction không resolve — INFRA §6.4) |
| deps | volume `aiw-<pid>-nm` | `/workspace/node_modules` | rw | xoá khi cleanup task |
| cache | tmpfs hoặc volume `aiw-<pid>-cache` | `/home/agent/.cache` | rw, **không** chia sẻ giữa project | chống cache poisoning (AB-05) |
| `.git` | không mount; mask bằng tmpfs rỗng | `/workspace/.git` | — | job cần git log dùng `ro` có chủ đích |
| `.env*` thật | không bao giờ | — | — | chỉ `.env.test` giá trị giả |

Nếu Runtime team chọn bind mount host: nguồn phải đi qua `canonicalizeAndAssertUnder(WORKSPACE_ROOT/worktrees/<id>)`, từ chối reparse point (`fsutil reparsepoint query`), và mount `ro` cho lint/test (INFRA §6.4). Không bao giờ mount `%USERPROFILE%`, `%APPDATA%`, `~/.claude`, `~/.codex`, `~/.ssh`, `~/.aws`.

### 4.6 Giới hạn tài nguyên

| Tham số | build | test | install | preview |
| --- | --- | --- | --- | --- |
| `--cpus` | 2 | 2 | 1 | 1 |
| `--memory` = `--memory-swap` | 4g | 3g | 2g | 1g |
| `--pids-limit` | 512 | 512 | 256 | 256 |
| timeout | 15 phút | 20 phút | 10 phút | TTL 2h |
| stdout tối đa | 20 MB (cắt đuôi) | 20 MB | 5 MB | 5 MB |

Toàn cục: sandbox đồng thời ≤ 3 (dev box) [INFRA §6.5]; với 18 agent cần tăng theo RAM thực của VM — `.wslconfig` (`memory`, `processors`, `swap=0`) là giới hạn duy nhất cho tổng VM [Đã xác minh theo docs, chưa đo]. Giám sát: `docker stats` mỗi 5 s; RAM > 90 % trong 60 s ⇒ `docker kill`; OOM đọc `State.OOMKilled`. Giám sát disk: `du` volume mỗi 5 phút, vượt quota 5 GB ⇒ kill job [Giả định: không có `--storage-opt size`].

### 4.7 Command rewriting qua PreToolUse `updatedInput`

Luồng: Claude gọi tool `Bash` → CLI gửi HTTP POST tới hook listener (cổng `portH`, header `Authorization: Bearer <AIW_HOOK_TOKEN>`) → policy engine phân loại → trả JSON.

Request (rút gọn):
```json
{ "session_id": "s-7f3", "hook_event_name": "PreToolUse", "tool_name": "Bash",
  "tool_input": { "command": "npm test -- --runInBand", "description": "run tests" },
  "cwd": "D:\\aiw\\worktrees\\4K7QX2MZ" }
```
Response khi rewrite:
```json
{ "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "rule 60: executes-code -> container aiw-p1",
    "updatedInput": { "command": "aiw-exec p1 --task T-123 --hash 9f2c.. -- npm test -- --runInBand",
                      "description": "run tests (sandboxed)" } } }
```
Response khi deny: `permissionDecision: "deny"` + lý do ngắn (không lộ regex). Response khi cần approval: `"ask"` — backend giữ kết nối chờ quyết định (timeout 540 s, SECURITY §5.3).

Quy tắc rewrite:
1. Normalize + tách segment (SECURITY §3.1). Mỗi segment được phân loại độc lập: `host-safe`, `workspace-mutating`, `executes-code`, `dangerous`.
2. Nếu **bất kỳ** segment là `dangerous` ⇒ `deny` cả lệnh. Nếu hỗn hợp `host-safe` + `executes-code` (ví dụ `git diff && npm test`) ⇒ **từ chối ghép**, trả lý do "tách thành hai lệnh" (tránh quote lồng nhau khi bọc).
3. Lệnh bọc **không dùng shell trên host**: `aiw-exec` nhận argv đã parse (`shell-quote`/PowerShell tokenizer) và gửi tới Docker exec dạng mảng `Cmd: ['sh','-lc', <string>]` chỉ khi segment gốc có pipe/redirect hợp lệ trong container.
4. `--hash` = SHA-256 của `tool_input` gốc đã chuẩn hoá; `aiw-exec` yêu cầu backend xác nhận hash khớp quyết định đã duyệt (TOCTOU, PERMISSION_MODEL §6.3).
5. Interpreter-by-default: `node`, `python`, `pwsh -c`, `bash -c`, `npx`, `deno`, `ruby`, `perl` luôn vào container, chấp nhận false positive (`node --version`).
6. Biến môi trường trong lệnh (`KEY=val cmd`) bị loại và thay bằng danh sách allowlist của job.
7. Hook **không** được tin để rewrite đường dẫn: `aiw-exec` tự ánh xạ `cwd` host → `/workspace` bằng bảng của project, từ chối cwd ngoài worktree.
8. Fail-closed: hook timeout, backend lỗi, JSON parse lỗi ⇒ CLI nhận exit 2/deny (kiểm tra bằng test); không có đường "fail-open".
9. `aiw-exec` trả stdout/stderr + exit code nguyên vẹn; redaction áp dụng ở bước ingest chứ không ở `aiw-exec` (tránh làm hỏng output máy đọc).

Giới hạn đã biết: hook chỉ thấy lệnh Bash/PowerShell; **không** thấy hành vi bên trong `Edit/Write` (đã có path check) hay MCP server chạy trên host (R8, SECURITY §2.3). MCP: chỉ `--strict-mcp-config` với server do app cung cấp.

## 5. Tier 2 — Agent container (opt-in cho `trust_level = untrusted`)

| Hạng mục | Thiết kế |
| --- | --- |
| Image | `aiw-agent-runtime@sha256:…` cài Claude Code (hoặc Codex) bằng package cố định version; `USER 1000`; không `sudo` |
| Flags | Như §4.1 + `--read-only` với `/home/agent` tmpfs 256m cho state CLI; `--memory 6g --cpus 3 --pids-limit 768` (CLI + toolchain) |
| Credential | `ANTHROPIC_API_KEY` riêng cho Tier 2 (hoặc `claude setup-token`), inject bằng Docker secret/file tmpfs `/run/secrets/anthropic` mode 0400, **không** qua `Env` của container (hiện trong `docker inspect`); tuyệt đối không copy `.credentials.json` OAuth [Đề xuất] |
| Network | Chỉ tới `api.anthropic.com` (và `api.openai.com` nếu Codex) qua proxy allowlist riêng `aiw-egress-llm`; hook về backend qua `host.docker.internal:portH` chỉ cho phép đúng IP/port đó (iptables sidecar) |
| Sandbox nội bộ | bật `sandbox.enabled: true` của Claude Code trong Linux; có thể cần `enableWeakerNestedSandbox` [Giả định theo docs, chưa test] |
| Hook token | token riêng per-container, hết hạn khi container dừng |
| Workspace | volume riêng per task (`aiw-t-<taskId>`), clone từ bare repo bằng `git clone --local`; kết quả đưa ra host chỉ bằng `git bundle`/patch qua Lớp 3 diff validation, không bind mount ghi vào host |
| Dùng khi | repo lạ, task "cài và chạy package X", `trust_level = untrusted`, mọi task `network != none` không do user viết |
| Chi phí | Dùng API credits, không dùng subscription (SECURITY §2.4) [Cần người dùng phê duyệt] |

Tier 2 so với Tier 1: loại bỏ R1/R2 cho các task untrusted (không có OAuth, không có FS host), nhưng không loại R5 (Docker daemon) và R6.

## 6. Những gì permission CLI **không** enforce được

(Chi tiết nền tảng ở SECURITY §2.2; đây là danh sách thực hành để quyết định đặt control ở đâu.)

| # | Hạn chế | Hệ quả | Control bù |
| --- | --- | --- | --- |
| L-01 | Native Windows: không sandbox cho Bash/PowerShell [Đã xác minh docs] | Lệnh được phép chạy với toàn quyền user | rewrite vào container, denylist |
| L-02 | `deny Bash(rm *)` không chặn `/bin/rm`, `bash -c`, `node -e`, script tự xoá file | bypass trivial nếu chỉ dùng rule | hook normalizer + interpreter-by-default |
| L-03 | `Read` deny không bao phủ chương trình tự mở file | đọc credential qua `node script.js` | chạy trong container; không mount home |
| L-04 | Hook không chạy với `--bare`/`disableAllHooks`/lỗi settings | mất policy im lặng | verify `SessionStart` hook qua `--include-hook-events` trước prompt đầu; thiếu ⇒ kill (R8) |
| L-05 | MCP server, LSP, hook script chạy trên host | code ngoài policy | `--strict-mcp-config`, chỉ server app cung cấp |
| L-06 | File tools chạy quyền user trên host | ghi tuỳ ý nếu path check lọt | hook path check + realpath + reparse point |
| L-07 | Symlink/junction/8.3 name/UNC/ADS (`file.txt:stream`) | thoát worktree | cấm reparse point, UNC, `:` trong path ngoài ổ đĩa, 8.3 name (so sánh bằng `GetLongPathName`) |
| L-08 | Không cô lập giữa agent (cùng user OS) | R4 | worktree riêng, manifest, diff validation |
| L-09 | Không giới hạn CPU/RAM host cho tiến trình CLI | DoS | Job Object limit (§8), semaphore |
| L-10 | `settings.json` user (có `skipDangerousModePermissionPrompt: true`, `autoMode`) [Đã xác minh] | có thể ảnh hưởng agent | `--settings` precedence cao hơn user; `disableBypassPermissionsMode`; test snapshot |
| L-11 | Codex `unelevated` không deny-read, network yếu | Codex yếu hơn Claude Tier 1 | dùng `elevated`/`mxc` sau khi verify (R10) |

## 7. Windows process-tree kill

Mục tiêu: dừng mọi tiến trình con của CLI, kể cả đã detach, kể cả khi cha đã chết.

1. **Job Object (chính):** tạo trước khi spawn: `CreateJobObject`; `JOBOBJECT_EXTENDED_LIMIT_INFORMATION.LimitFlags = KILL_ON_JOB_CLOSE | ACTIVE_PROCESS | PROCESS_MEMORY`; `ActiveProcessLimit = 64` (chặn fork bomb host), `ProcessMemoryLimit = 4 GB`; **không** đặt `BREAKAWAY_OK`/`SILENT_BREAKAWAY_OK`. Spawn tiến trình ở trạng thái `CREATE_SUSPENDED`, `AssignProcessToJobObject`, rồi `ResumeThread` (tránh race con thoát trước khi gán). Hiện thực bằng `aiw-job.exe` nhỏ (C#/Rust) hoặc `koffi` FFI [Đề xuất]; test trên build 26300.
2. **Kill:** `TerminateJobObject(job, 1)`; ghi audit `agent.kill` kèm `pids` đã liệt kê trước khi kill (`QueryInformationJobObject(BasicProcessIdList)`).
3. **Fallback:** `taskkill /PID <pid> /T /F`; sau đó quét `Get-CimInstance Win32_Process -Filter "ParentProcessId=<pid>"` đệ quy theo cây đã chụp trước kill (vì `/T` bỏ sót con mồ côi); so sánh `CreationDate` để không giết PID tái dùng.
4. **Xác nhận:** sau 3 s không còn tiến trình nào trong job và không còn `Win32_Process` có `ExecutablePath` dưới `WORKSPACE_ROOT`/cwd worktree ⇒ `killed_clean`; nếu còn ⇒ cảnh báo `kill_incomplete` mức critical.
5. Tiến trình con chạy trong container: dừng bằng `docker kill` (không phụ thuộc Job Object).

## 8. Trình tự emergency stop

| Bước | Hành động | Thời hạn |
| --- | --- | --- |
| 0 | Nhận lệnh (UI "STOP ALL", API `/emergency-stop`, hotkey, hoặc auto-trigger: audit chain fail, RAM > 95 %, egress bất thường). Không cần approval | t=0 |
| 1 | Đặt cờ `system.state=stopping` trong Redis + memory: scheduler ngừng giao task; hook listener trả `deny` cho mọi request mới ("emergency stop") | < 100 ms |
| 2 | Ghi audit `emergency_stop` **trước** (intent) với actor, lý do; nếu DB down ghi spool NDJSON | < 200 ms |
| 3 | `docker network disconnect` mọi `aiw-egress`/`aiw-preview` (nghi exfil) | < 1 s |
| 4 | `TerminateJobObject` cho tất cả job CLI (song song) | < 1 s |
| 5 | `docker kill $(docker ps -q --filter label=aiw=1)` (song song, bỏ `docker rm` để giữ forensic) | < 3 s [Giả định: SECURITY §9 chaos test] |
| 6 | Revoke mọi cached approval, hook token, API session; xoá file secret tạm `%LOCALAPPDATA%\aiw\run\*` | < 1 s |
| 7 | Quét orphan (`Win32_Process` dưới WORKSPACE_ROOT), `docker ps` kiểm tra rỗng; ghi audit outcome `killed_clean`/`kill_incomplete` | < 5 s |
| 8 | Trạng thái task → `paused`; worktree **giữ nguyên** (để review); UI hiện báo cáo stop | — |
| 9 | Resume: chỉ Owner, qua UI, yêu cầu xem báo cáo; hook token mới, container tạo lại | thủ công |

Emergency stop không gọi bất kỳ đường nào có thể bị agent chặn (không qua hook, không qua CLI); chạy trong tiến trình orchestrator, có watchdog thứ hai (`aiw-job.exe`) tự giết job nếu orchestrator mất heartbeat 15 s.

## 9. Fail-closed & trạng thái Docker

- Docker daemon không chạy ⇒ lệnh `executes-code` bị `deny` với thông báo "Start Docker Desktop"; agent vẫn đọc/ghi tài liệu được (SECURITY R5, INFRA §13).
- `inspect` không khớp `assertSafeSpec` ⇒ không `exec`, ghi audit `policy_deny`, cảnh báo UI.
- Image digest không có trong policy ⇒ từ chối tạo container.
- Container label `aiw=1` lạ (không do app tạo) ⇒ cảnh báo, không `exec`.

## 10. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
| --- | --- | --- |
| Container per project (warm) cho dev/test, ephemeral cho preview/custom | Cân bằng độ trễ (18 agent ≈ ít project) và cách ly | Đề xuất; cần hoà giải INFRA §6.1 |
| `--network none` + proxy allowlist chỉ cho install | Giảm exfil; chấp nhận R6 | Đề xuất |
| Digest pinning + `assertSafeSpec` deny-by-default | Chống drift cấu hình và chỉnh tay | Đề xuất |
| Rewrite fail-closed, từ chối lệnh ghép host+container | Tránh quote injection khi bọc | Đề xuất |
| Job Object + watchdog ngoài | `taskkill /T` không đủ | Đề xuất; cần test build 26300 |
| Tier 2 dùng API key riêng, không OAuth | Không đưa credential subscription vào container | Đề xuất; Cần người dùng phê duyệt (chi phí) |
| Không Docker-in-Docker, không docker.sock | DinD ⇒ privileged | Đề xuất (khớp INFRA §6.8) |

## 11. Checklist xác minh (chạy khi Docker Desktop chạy; trước M1)

| ID | Kiểm tra | Cách làm | Kỳ vọng | Trạng thái hiện tại |
| --- | --- | --- | --- | --- |
| SB-00 | Daemon & cgroup | `docker info --format '{{json .}}'` | `CgroupVersion=2`, kernel ≥ 6.6, `SecurityOptions` chứa `seccomp` | Giả định |
| SB-01 | Không privileged | `docker inspect … .HostConfig.Privileged` | `false` | Giả định |
| SB-02 | cap-drop | `.HostConfig.CapDrop` | chứa `ALL`, `CapAdd` rỗng | Giả định |
| SB-03 | no-new-privileges | `.HostConfig.SecurityOpt` | chứa `no-new-privileges:true` | Giả định |
| SB-04 | rootfs read-only | `touch /x` trong container | lỗi `Read-only file system` | Giả định |
| SB-05 | user | `id -u` | `1000` | Giả định |
| SB-06 | không docker.sock | `.Mounts`, `ls /var/run/docker.sock` | không có | Giả định |
| SB-07 | không mount host nhạy cảm | `.Mounts[].Source` | không chứa `C:\Users`, `.claude`, `.ssh` | Giả định |
| SB-08 | pids limit | fork bomb `:(){ :\|:& };:` | dừng ở 512, host không treo | Giả định |
| SB-09 | disk/tmpfs | ghi 1 GB vào `/tmp` | lỗi `No space left` ở 512 MB | Giả định |
| SB-10 | cô lập project | container P1 đọc volume P2 | không thấy | Giả định |
| SB-11 | memory/OOM | cấp phát 5 GB với limit 4g | `OOMKilled=true`, host không swap | Giả định |
| SB-12 | settings tamper | agent ghi `.claude/settings.json` | deny (hook) + Lớp 3 reject | Giả định |
| SB-13 | reparse point | tạo junction trong worktree trỏ `C:\Users` | hook deny, `fsutil` phát hiện | Giả định |
| SB-14 | UNC | `Read \\\\127.0.0.1\\c$\\x` | deny, không phát sinh kết nối SMB | Giả định |
| SB-15 | Job Object | spawn `start /b ping -t`, kill job | không còn tiến trình < 3 s | Giả định |
| SB-16 | admin guard | chạy app elevated | từ chối khởi động | Giả định |
| EG-01 | egress none | `curl https://example.com` (phase test) | fail | Giả định |
| EG-02 | egress install | `npm i left-pad` qua proxy; `curl example.com` | npm OK; example.com bị chặn 403 | Giả định |
| EG-03 | DNS/raw socket | `nslookup 8.8.8.8`, `nc 1.1.1.1 443` | fail | Giả định; R6 |
| CH-05 | emergency stop | giết 18 job + container | `killed_clean` < 5 s | Giả định |
| SB-17 | seccomp mặc định | `docker inspect … SecurityOpt` không có `unconfined`; `unshare -U` trong container | bị chặn | Giả định |

Mọi mục chuyển từ "Giả định" sang "Đã xác minh" phải ghi lệnh, output và ngày vào `docs/security/VERIFICATION_LOG.md` (file do Phase 1 tạo khi có daemon).

## 12. Dependencies

- `docs/architecture/INFRASTRUCTURE_DESIGN.md` §6, §12; `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md` §7, §11.
- [`PERMISSION_MODEL.md`](PERMISSION_MODEL.md) (rule 60 rewrite, normalizer), [`AUDIT_LOGGING.md`](AUDIT_LOGGING.md) (category `container`, `emergency_stop`), [`CREDENTIAL_MANAGEMENT.md`](CREDENTIAL_MANAGEMENT.md) §7 (Tier 2 secret).
- Docker Desktop (WSL2), dockerode, Squid/tinyproxy image, `aiw-job.exe`.

## 13. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
| --- | --- | --- |
| R1: lệnh lọt hook chạy unsandboxed trên host | Cao | interpreter-by-default, fuzz, chấp nhận hoặc Tier 2/WSL2 |
| R5: Docker Desktop daemon = root VM | Cao | `assertSafeSpec`, label, không expose API |
| R6: proxy allowlist ≠ firewall kernel | Trung bình | `none` mặc định; test EG-03 |
| `noexec` tmpfs phá toolchain | Thấp | fallback TMPDIR |
| Container warm giữ trạng thái bị nhiễm | Trung bình | recreate sau install/4h/đổi trust |
| Quota volume không enforce được | Trung bình | `du` định kỳ + kill |
| Hook latency làm chậm mỗi tool call | Thấp | HTTP loopback < 20 ms, cache phân loại |

## 14. Vấn đề chưa giải quyết

1. Daemon chưa chạy ⇒ mọi mục §11 còn "Giả định".
2. Worktree host-truth vs volume-truth chưa chốt (Runtime team); ảnh hưởng §4.5.
3. Hợp nhất mô hình container warm (SECURITY) vs ephemeral (INFRA) cần ADR.
4. Hiệu năng/độ ổn định của `aiw-job.exe` vs `koffi` trên Windows build 26300.
5. Squid vs tinyproxy; có cần Verdaccio/pip cache nội bộ (INFRA §14.2).
6. Tier 2: `enableWeakerNestedSandbox` có thực sự cần không; cách nhận hook trên `host.docker.internal` an toàn.
7. Chiến lược cho Codex (R10).

## 15. Acceptance Criteria

- [ ] `assertSafeSpec` có unit test từ chối mọi khoá cấm (Privileged, CapAdd, Binds, PidMode, NetworkMode=host…) và chạy lại trên `inspect` trước mỗi exec.
- [ ] Toàn bộ SB-00..SB-17, EG-01..03, CH-05 chạy tự động trên máy có daemon và pass; kết quả ghi vào VERIFICATION_LOG.
- [ ] Hook rewrite: 100 % lệnh `executes-code` trong corpus (≥ 300) được bọc `aiw-exec`; lệnh ghép host+container bị từ chối; hook lỗi ⇒ deny (fail-closed).
- [ ] `docker inspect` mọi container label `aiw=1` thoả policy as code (conftest) trong CI.
- [ ] Emergency stop: 18 agent giả lập + container + tiến trình `start /b` dừng hoàn toàn < 5 s, audit đủ intent + outcome.
- [ ] Tier 2: `docker inspect` không chứa API key trong `Env`; `curl` tới domain ngoài allowlist fail; ghi file ngoài volume không ảnh hưởng host.
- [ ] Docker tắt ⇒ `executes-code` bị deny với thông báo rõ; `/ready` trả 503.
