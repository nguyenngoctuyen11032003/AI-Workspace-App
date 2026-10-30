# Infrastructure Design – Topology, Sandbox Executor, Preview, Observability

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Liên quan: `docs/security/SECURITY.md` (A09), `docs/architecture/DATABASE_DESIGN.md` (A08), `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md`, `docs/development/ENVIRONMENT_SETUP.md`, `docs/development/DEPLOYMENT.md`, `docs/operations/DEVSECOPS_GUIDE.md`.
>
> Nhãn: **[Đã xác minh]** / **[Đề xuất]** / **[Giả định]** / **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Mô tả hạ tầng chạy AI WORKSPACE ở ba môi trường (dev / staging / prod-self-hosted), và đặc biệt là **Sandbox Executor** – thành phần build/test/preview các project do AI agent sinh ra bên trong Docker container với giới hạn tài nguyên và cách ly mạng. Tài liệu này là nguồn tham chiếu cho Phase 1 khi hiện thực `packages/sandbox` và `infra/compose`.

## 2. Phạm vi

**Trong:** topology, Docker Compose cho local dev, sandbox executor (lifecycle, image, mount, resource limits, network, timeouts, output capture), preview deployment, tránh Docker-in-Docker, logging/monitoring, backup/restore Postgres, ghi chú Windows.

**Ngoài:** threat model đầy đủ (A09), schema DB (A08), CI pipeline (CI_CD.md), hướng dẫn cài đặt từng bước (ENVIRONMENT_SETUP.md).

## 3. Môi trường đã xác minh trên máy dev

| Hạng mục | Kết quả | Trạng thái |
|---|---|---|
| OS | Windows 11 Pro for Workstations 10.0.26300 | [Đã xác minh] |
| Docker client | 29.8.1; Docker Compose v5.5.1 | [Đã xác minh] |
| Docker daemon | **không chạy** lúc kiểm tra (`dockerDesktopLinuxEngine` pipe không tồn tại) → chưa xác minh được cgroup version, kernel, ECI | [Đã xác minh là chưa chạy] |
| Docker context | `desktop-linux` (mặc định), `default` (npipe docker_engine) | [Đã xác minh] |
| WSL | WSL 2, distro `docker-desktop` (Stopped) – không có distro Ubuntu riêng | [Đã xác minh] |
| Node / npm / pnpm / corepack | 24.19.0 / 11.17.0 / 10.34.6 / 0.35.0 | [Đã xác minh] |
| trivy, gitleaks | chưa cài | [Đã xác minh] |
| Disk | D: 109 GB trống; C: 162 GB trống | [Đã xác minh] |

**Giả định cần xác nhận khi Docker Desktop chạy:** kernel WSL2 ≥ 6.6, cgroup v2 (Docker Desktop mặc định cgroup v2 từ 4.x), Enhanced Container Isolation (ECI) **không** khả dụng trên WSL2 backend theo tài liệu Docker (chỉ Hyper-V backend, và cần Docker Business). Rootless Docker không áp dụng cho Docker Desktop trên Windows: daemon đã chạy trong VM WSL2 với user `root` của VM, nhưng VM là ranh giới cách ly với host Windows. [Giả định dựa trên tài liệu Docker; cần xác minh bằng `docker info` khi daemon chạy]

## 4. Topology

### 4.1 Dev (máy developer, Windows 11)

```
Windows host
├── Node 24 (backend NestJS :3000, Vite dev :5173, BullMQ workers)  ← chạy native để spawn claude/codex CLI
├── Docker Desktop (WSL2 VM)
│   ├── postgres:16-alpine   :5432   (volume aiw_pg)
│   ├── redis:7-alpine       :6379   (volume aiw_redis)
│   ├── [tuỳ chọn] mailpit, pgadmin, otel-collector, grafana/loki
│   └── sandbox containers (ephemeral, do backend tạo qua Docker Engine API)
└── D:\aiw  (WORKSPACE_ROOT: repo.git, worktrees, artifacts)
```

Backend chạy **native trên Windows** chứ không trong container, vì nó phải spawn `claude`/`codex` với login của người dùng (`%USERPROFILE%\.claude`) – xem DEPLOYMENT.md §4.

### 4.2 Staging (self-hosted Linux VM, tuỳ chọn)

- 1 VM Ubuntu 24.04 (4 vCPU / 16 GB), Docker Engine + Compose.
- `docker compose -f compose.yml -f compose.staging.yml up -d`: `api`, `web` (nginx static), `worker`, `postgres`, `redis`, `caddy` (TLS).
- Agent CLI chạy trong container `worker` với volume `claude-home` đã login bằng `claude login` một lần (xem DEPLOYMENT.md).
- Sandbox: backend dùng Docker socket **chỉ từ `worker`**, qua **docker-socket-proxy** (Tecnativa) giới hạn API (`CONTAINERS=1 IMAGES=1 EXEC=1`, cấm `volumes`, `networks` create, `system`). [Đề xuất]

### 4.3 Prod (self-hosted, multi-user)

Giống staging + Postgres managed/backup offsite, Redis persistence AOF, reverse proxy có auth, giới hạn worker sandbox theo node. Chưa có kế hoạch SaaS multi-tenant trong Phase 0. Mọi chế độ đều **không** tự deploy project sinh ra lên production (DEPLOYMENT.md §9).

## 5. Docker Compose cho local dev [Đề xuất]

`infra/compose/compose.yml` (rút gọn):

```yaml
name: aiw
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: aiw
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-aiw_dev_only}
      POSTGRES_DB: aiw
    ports: ["127.0.0.1:5432:5432"]
    volumes: ["aiw_pg:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U aiw -d aiw"], interval: 5s, timeout: 3s, retries: 20 }
    command: ["postgres", "-c", "log_min_duration_statement=500", "-c", "shared_buffers=256MB"]
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "512mb", "--maxmemory-policy", "noeviction"]
    ports: ["127.0.0.1:6379:6379"]
    volumes: ["aiw_redis:/data"]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 3s, retries: 20 }
volumes:
  aiw_pg: {}
  aiw_redis: {}
```

Ghi chú: bind port vào `127.0.0.1` để không lộ ra LAN; `noeviction` vì BullMQ yêu cầu Redis không eviction job key; profile `tools` thêm `pgadmin`/`mailpit`.

## 6. Sandbox Executor

### 6.1 Vai trò

Chạy các job `build`, `test`, `lint`, `typecheck`, `preview` cho project sinh ra, theo yêu cầu của Orchestrator hoặc của Worktree Manager (validate trước merge). Mỗi job là một **ephemeral container**, không bao giờ reuse container giữa các task.

### 6.2 Lifecycle

```
Orchestrator ──enqueue(SandboxJob)──▶ BullMQ queue "sandbox"
   worker: 1) chọn image theo stack  2) tạo container (Docker Engine API, dockerode)
           3) bind mount worktree  4) start, stream logs → artifacts + WebSocket
           5) chờ exit / timeout  6) thu thập report (junit, coverage)  7) rm container
```

```ts
export interface SandboxJob {
  jobId: string; taskId: string; worktreePath: string;
  stack: 'node' | 'python' | 'java' | 'custom';
  kind: 'build' | 'test' | 'lint' | 'typecheck' | 'preview' | 'custom';
  command: string[];                       // argv, không qua shell
  env?: Record<string, string>;            // chỉ biến whitelist
  limits: { cpus: number; memoryMb: number; pidsLimit: number; timeoutSec: number; diskMb?: number };
  network: 'none' | 'egress-allowlist';
  mountMode: 'rw' | 'ro';                  // test/lint có thể ro + tmpfs cho output
}
export interface SandboxResult { exitCode: number | null; timedOut: boolean; oomKilled: boolean; durationMs: number; logPath: string; reports: string[]; }
```

### 6.3 Image selection per stack [Đề xuất]

| Stack | Base image (pin digest trong `infra/images/`) | Ghi chú |
|---|---|---|
| node | `node:24-bookworm-slim` + pnpm/npm/yarn qua corepack, git, ca-certificates | user `node` (uid 1000) |
| python | `python:3.12-slim` + uv/pip, build-essential tối thiểu | user `app` (uid 1000) |
| java | `eclipse-temurin:21-jdk` + maven/gradle wrapper | user `app` |
| custom | Dockerfile trong repo project, build **chỉ khi người dùng phê duyệt**, chạy với `--network none` khi build xong | [Cần người dùng phê duyệt] |

Images được build bởi platform (`infra/images/<stack>/Dockerfile`), scan Trivy trong CI, tag theo `aiw-sandbox-<stack>:<version>`; không pull image tuỳ ý theo yêu cầu của agent.

### 6.4 Mount & filesystem

- Bind mount `worktreePath` → `/work` (`rw` cho build/preview, `ro` cho lint/test có thể dùng kèm `tmpfs /work/.cache`).
- `node_modules`: trên Windows, pnpm tạo junction/hardlink không resolve được qua bind mount WSL2 → sandbox dùng **named volume theo task** `aiw-nm-<taskId>` mount vào `/work/node_modules` (ẩn thư mục host), xoá khi task cleanup. Ưu điểm: I/O nhanh hơn nhiều so với NTFS qua 9p.
- Không mount `.git` khi không cần (`--mount type=bind,src=<wt>/.git,dst=/work/.git,ro` chỉ cho job cần git log; mặc định mask bằng tmpfs rỗng).
- `--read-only` rootfs + `--tmpfs /tmp:rw,noexec,nosuid,size=512m`; thư mục ghi được chỉ `/work` và `/tmp`.
- Không mount `/var/run/docker.sock`, không `--privileged`, không `--pid=host`, không `--cap-add`; `--cap-drop ALL`, `--security-opt no-new-privileges`, seccomp default của Docker.

### 6.5 Resource limits & timeouts (mặc định) [Đề xuất]

| Tham số | build | test | preview |
|---|---|---|---|
| `--cpus` | 2 | 2 | 1 |
| `--memory` / `--memory-swap` | 4g / 4g | 3g / 3g | 1g / 1g |
| `--pids-limit` | 512 | 512 | 256 |
| timeout | 15 phút | 20 phút | TTL 2h idle |
| disk (volume quota) | 5 GB | 5 GB | 2 GB |
| ulimit nofile | 65536 | 65536 | 16384 |

Timeout: worker gọi `container.stop({ t: 10 })` rồi `kill`; ghi `timedOut=true`. OOM đọc từ `inspect.State.OOMKilled`. Tổng sandbox đồng thời trên dev box: 3 (cấu hình `SANDBOX_MAX_CONCURRENCY`) – Docker Desktop VM mặc định chia sẻ CPU/RAM theo `.wslconfig` (khuyến nghị `memory=12GB`, `processors=6` trên máy 32 GB; cần người dùng chỉnh) [Cần người dùng phê duyệt].

### 6.6 Network modes

- **`none` (mặc định):** không có network. Mọi dependency phải được cài trước trong bước `install` riêng hoặc có trong image.
- **`egress-allowlist`:** dùng cho job `install`. Container nối vào network nội bộ `aiw-egress` chỉ có một container `proxy` (Squid/tinyproxy) với allowlist domain: `registry.npmjs.org`, `registry.yarnpkg.com`, `pypi.org`, `files.pythonhosted.org`, `repo.maven.apache.org`, `github.com` (git over https). Container job được set `HTTP_PROXY/HTTPS_PROXY/npm_config_proxy`, và network `aiw-egress` là `internal: true` nên không có đường ra trực tiếp. Tốt hơn: cache registry nội bộ (Verdaccio) sau MVP. [Đề xuất]
- Preview container: network riêng `aiw-preview`, chỉ publish port lên `127.0.0.1`.
- Chạy agent CLI trong container (hardened mode, sau MVP): cần egress tới `api.anthropic.com`/`api.openai.com` – dùng cùng cơ chế allowlist.

### 6.7 Output capture

- stdout/stderr stream qua Docker attach → file `artifacts/<taskId>/<jobId>.log` (giới hạn 20 MB, cắt đuôi + ghi chú), đồng thời push qua WebSocket/SSE tới UI (throttle 20 msg/s).
- Report: job test được yêu cầu ghi `/work/.aiw-out/junit.xml`, `coverage/`; worker copy ra `artifacts/` rồi parse (junit → DB bảng `test_results` – A08).
- Log lưu DB chỉ metadata (path, size, exit code); nội dung ở filesystem/object storage.
- Mọi output coi là **untrusted** (có thể chứa prompt injection) – UI render dạng text, không markdown-execute; khi feed lại cho agent phải bọc trong delimiter rõ ràng (A09).

### 6.8 Tránh Docker-in-Docker

- Backend/worker gọi Docker Engine API của host (named pipe trên Windows, socket trên Linux qua docker-socket-proxy). Sandbox container **không** có docker socket, không chạy dockerd.
- Nếu project sinh ra có `Dockerfile`/`compose.yml`, platform chỉ build/run nó **ở cấp host** sau khi người dùng phê duyệt, với cùng bộ giới hạn ở §6.5, không cho phép `privileged`, bind mount ngoài worktree, hay `network_mode: host`. Compose file được parse và kiểm tra tĩnh trước khi chạy.
- Testcontainers trong project sinh ra: không hỗ trợ ở MVP (yêu cầu docker socket) → ghi nhận là limitation.

## 7. Preview deployment của app sinh ra

1. Người dùng bấm **Preview** trên task/project → job `preview` (stack-aware: `pnpm dev --host 0.0.0.0 --port 3000` hoặc `python -m uvicorn`...). Lệnh preview lấy từ `project.json`/`package.json` do PM agent đề xuất, người dùng xác nhận lần đầu.
2. Container: image stack, mount worktree `rw`, network `aiw-preview`, `-p 127.0.0.1:0:3000` (Docker chọn port tự do), label `aiw.task=<id>`.
3. Health check: worker poll `GET http://127.0.0.1:<port>/` (hoặc path cấu hình) tối đa 90s; thành công → DB lưu `preview_url`, UI hiển thị link + nút Stop; thất bại → log + trạng thái `preview_failed`.
4. TTL idle 2h (không có request qua reverse proxy nhẹ của platform, hoặc đơn giản là 2h tuyệt đối ở MVP) → auto stop. Tối đa 3 preview đồng thời.
5. Preview chỉ bind `127.0.0.1`; chế độ self-hosted expose qua Caddy với path `/preview/<taskId>/` và auth session của platform. [Đề xuất]
6. Mọi preview đều mang banner "Generated app – not production" trong UI platform; **không** có nút deploy ra ngoài.

## 8. Monitoring & logging

- **Logs:** pino (NestJS `nestjs-pino`), JSON, trường bắt buộc: `ts, level, service, reqId, taskId, agentSlug, jobId`. Dev: `pino-pretty`. Redaction paths: `req.headers.authorization`, `*.apiKey`, `*.token`, `env.*_KEY`.
- **Audit log** riêng (bảng `audit_events` – A08): spawn agent, hook deny, diff validation fail/override, merge, preview start/stop, secret access. Retention: xem DEVSECOPS_GUIDE §9.
- **Metrics:** `@opentelemetry/sdk-node` tuỳ chọn (`OTEL_ENABLED=false` mặc định); exporter OTLP → `otel-collector` → Prometheus/Grafana trong profile `observability` của compose. Metrics tối thiểu: queue depth BullMQ, job duration theo kind, sandbox OOM/timeout count, worktree count, disk usage.
- **Health:** `GET /health` (liveness), `GET /ready` (Postgres, Redis, Docker daemon reachable, `WORKSPACE_ROOT` writable, `claude --version`).
- **BullMQ dashboard:** `@bull-board` ở `/admin/queues` chỉ dev/self-hosted với auth.

## 9. Backup & restore Postgres

- Dev: script `pnpm db:backup` → `docker exec aiw-postgres pg_dump -U aiw -Fc aiw > backups/aiw-<ts>.dump`; `pnpm db:restore <file>` → `pg_restore --clean --if-exists`.
- Self-hosted: container `pgbackup` (cron hàng ngày 02:00) ghi `pg_dump -Fc` vào volume `aiw_backups`, giữ 7 daily + 4 weekly; đồng bộ offsite (rclone → S3/B2) tuỳ cấu hình. Backup được mã hoá bằng `age` trước khi rời máy. [Đề xuất]
- Redis: AOF đủ cho queue; không backup riêng (job có thể re-enqueue).
- Workspace data (`repo.git`): backup bằng `git bundle --all` hàng ngày vào `aiw_backups`; worktrees không backup (tái tạo từ branch).
- Kiểm tra restore mỗi tháng (xem DEVSECOPS_GUIDE checklist).

## 10. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Backend chạy native trên Windows ở dev/desktop mode | Phải spawn CLI với login user; tránh mount `~/.claude` vào container ở MVP | [Đề xuất] |
| Ephemeral container per job, Docker Engine API qua dockerode | Cách ly đơn giản, không state tồn đọng; dockerode phổ biến, typed | [Đề xuất] |
| `--network none` mặc định, allowlist proxy cho install | Chặn exfiltration từ code sinh ra; vẫn cài được dependency | [Đề xuất] |
| Named volume cho `node_modules` thay vì bind mount | Junction NTFS không resolve trong WSL2; hiệu năng | [Đề xuất] |
| Không Docker-in-Docker | DinD cần privileged → phá cách ly | [Đề xuất] |
| ECI/rootless không dùng | ECI không hỗ trợ WSL2 backend, cần Docker Business; rootless không áp dụng với Docker Desktop | [Giả định – cần xác minh khi daemon chạy] |
| OpenTelemetry tuỳ chọn | Tránh phức tạp MVP; pino là bắt buộc | [Đề xuất] |

## 11. Dependencies

- A08: bảng `sandbox_jobs`, `test_results`, `previews`, `audit_events`.
- A09: threat model cho sandbox escape, prompt injection từ output, egress.
- WORKSPACE_AND_GIT_ISOLATION.md: worktree path, cleanup hook để xoá volume `aiw-nm-<taskId>`.
- CI_CD.md: build + scan sandbox images.
- Docker Desktop phải chạy khi backend khởi động (readiness check).

## 12. Ghi chú Windows

- Docker Desktop WSL2 backend: bind mount từ NTFS (`D:\aiw\...`) đi qua 9p → chậm với hàng chục nghìn file nhỏ. Giải pháp theo thứ tự: (1) named volume cho `node_modules`/cache; (2) sau MVP: đặt `WORKSPACE_ROOT` trong WSL2 distro (`\\wsl$\...`) và backend truy cập qua `\\wsl.localhost` – đổi lại git trên Windows chậm hơn; (3) chạy toàn bộ backend trong WSL2 (cần `claude` login trong WSL2).
- `.wslconfig` giới hạn VM (`memory`, `processors`, `swap=0`) là cách duy nhất giới hạn tổng tài nguyên; `--cpus/--memory` chỉ giới hạn từng container trong VM.
- Named pipe `//./pipe/dockerDesktopLinuxEngine` là endpoint; dockerode hỗ trợ `socketPath` dạng npipe.
- Line endings: script chạy trong container phải LF (`.gitattributes` – xem WORKSPACE doc §11).
- Port publish `127.0.0.1:<port>` trên Docker Desktop vẫn đi qua `vpnkit`/WSL relay – hoạt động, nhưng `host.docker.internal` cần để container gọi ngược backend (preview không cần).
- Windows Defender: loại trừ `D:\aiw` và `%LOCALAPPDATA%\Docker\wsl` khỏi realtime scan để tránh khoá file [Cần người dùng phê duyệt].

## 13. Rủi ro

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Code sinh ra khai thác kernel WSL2 (shared kernel giữa container) | Trung bình | Không privileged, cap-drop ALL, seccomp, no-new-privileges; Hyper-V backend + ECI cho người dùng Docker Business (tuỳ chọn) |
| Docker daemon không chạy → toàn bộ job fail | Cao (dev) | Readiness check + hướng dẫn UI "Start Docker Desktop"; retry queue với backoff |
| Bind mount chậm khiến test timeout | Trung bình | Named volume cho deps; timeout cấu hình theo stack |
| Preview container giữ port/CPU lâu | Thấp | TTL + max 3 |
| Allowlist proxy bị vòng qua bằng DNS/IP trực tiếp | Trung bình | Network `internal: true`, chỉ proxy có route ra ngoài |
| Image base có CVE | Trung bình | Trivy trong CI, rebuild hàng tuần (DEVSECOPS_GUIDE) |

## 14. Vấn đề chưa giải quyết

1. Xác minh `docker info` (cgroup v2, kernel, security options) khi Docker Desktop chạy – cập nhật §3.
2. Có đầu tư Verdaccio/pip cache nội bộ ngay MVP không, hay chỉ proxy allowlist?
3. Cơ chế disk quota cho named volume (Docker Desktop không hỗ trợ `--storage-opt size` trên overlay2 mặc định) → tạm dùng kiểm tra `du` định kỳ.
4. Lựa chọn dockerode vs gọi `docker` CLI qua execa (dockerode: typed, stream tốt; CLI: ít phụ thuộc). Đề xuất dockerode.
5. Preview cho app cần DB riêng (Postgres sinh ra bởi project) – có cho phép compose con hay không.

## 15. Acceptance Criteria

- [ ] `docker compose up -d` trong `infra/compose` khởi động Postgres+Redis healthy trong < 60s trên máy dev; port chỉ bind 127.0.0.1.
- [ ] Sandbox job `node/test` chạy trên worktree mẫu, trả `exitCode`, junit parse được, log lưu `artifacts/`.
- [ ] Job với `network: none` không thể `curl https://example.com` (test tự động); job `egress-allowlist` cài được `npm i left-pad` nhưng không truy cập `example.com`.
- [ ] Container bị kill đúng khi vượt timeout; `memory` 256m với script cấp phát 1 GB → `oomKilled=true`.
- [ ] `docker inspect` container sandbox: `Privileged=false`, `CapDrop=[ALL]`, `ReadonlyRootfs=true`, không có mount docker.sock, user != root.
- [ ] Preview: URL `http://127.0.0.1:<port>` hiển thị trong UI sau health check; stop sau TTL.
- [ ] `/ready` trả 503 khi Docker daemon tắt, kèm message rõ ràng.
- [ ] `pnpm db:backup` + `pnpm db:restore` khôi phục được dữ liệu trên DB rỗng.
