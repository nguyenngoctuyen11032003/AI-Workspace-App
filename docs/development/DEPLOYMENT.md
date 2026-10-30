# Deployment – Running AI WORKSPACE (Desktop mode & Self-hosted server mode)

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Liên quan: `docs/architecture/INFRASTRUCTURE_DESIGN.md`, `docs/development/CI_CD.md`, `docs/development/ENVIRONMENT_SETUP.md`, `docs/operations/DEVSECOPS_GUIDE.md`, `docs/security/SECURITY.md` (A09), `docs/architecture/DATABASE_DESIGN.md` (A08).
>
> Nhãn: **[Đã xác minh]** / **[Đề xuất]** / **[Giả định]** / **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Mô tả cách **chính sản phẩm AI WORKSPACE** được cài đặt và vận hành ở hai chế độ: (a) **Desktop mode** – một người dùng trên máy cá nhân (Windows 11 trước, Tauri shell sau), (b) **Server mode** – self-hosted bằng Docker Compose cho nhóm nhỏ. Tài liệu cũng nêu rõ ràng ràng buộc "agent CLI phải chạy nơi có login của người dùng" và tuyên bố **không bao giờ tự động deploy project sinh ra lên production**.

## 2. Phạm vi

Topology từng chế độ, cấu hình môi trường, migration khi deploy, rollback, health check, logs, checklist deploy. Không bao gồm CI (CI_CD.md), thiết kế sandbox (INFRASTRUCTURE_DESIGN.md), chính sách secrets chi tiết (DEVSECOPS_GUIDE.md).

## 3. Chế độ Desktop (single-user, local) – MVP

### 3.1 Thành phần

```
Người dùng (Windows 11)
├── aiw backend (Node 24, NestJS :3000, BullMQ worker trong cùng process hoặc child process)
├── aiw web (Vite build static, phục vụ bởi backend tại http://127.0.0.1:3000) — sau: Tauri 2 WebView
├── Docker Desktop (WSL2): postgres, redis (compose profile "desktop"), sandbox containers
├── %USERPROFILE%\.claude  (login Claude Code) ; %USERPROFILE%\.codex (login Codex)
└── D:\aiw  (WORKSPACE_ROOT)
```

### 3.2 Cài đặt MVP (trước khi có Tauri installer) [Đề xuất]

1. Cài prerequisites theo ENVIRONMENT_SETUP.md §3–4.1 (Docker Desktop, Node 24, Git, Claude Code; Codex tuỳ chọn).
2. Tải `aiw-vX.Y.Z-win.zip` từ GitHub Release (chứa `api-dist`, `web-dist`, `infra/compose`, `scripts/aiw.ps1`).
3. `scripts\aiw.ps1 install`: tạo `%LOCALAPPDATA%\aiw\` (config, logs), `D:\aiw` (hỏi người dùng), sinh `.env` từ template với `SESSION_SECRET` ngẫu nhiên, chạy `docker compose --profile desktop up -d`, chạy migration, đăng ký Scheduled Task/`winsw` để backend chạy nền khi login (tuỳ chọn).
4. `scripts\aiw.ps1 start` → mở `http://127.0.0.1:3000`.
5. Lần đầu mở UI: wizard kiểm tra `claude auth status`, Docker daemon, disk; hướng dẫn `claude login` nếu thiếu.

Sau này: Tauri 2 shell đóng gói web + sidecar backend (`tauri.conf.json` `externalBin`), installer `.msi`; Docker Desktop và CLI vẫn là dependency ngoài (không bundle).

### 3.3 Đặc điểm bảo mật desktop mode

- Backend bind **chỉ `127.0.0.1`**; không có auth đa người dùng nhưng có session token local (chống CSRF từ web khác, A09).
- Secrets (API keys nếu người dùng nhập) lưu trong Windows Credential Manager qua `keytar`-tương đương (`@napi-rs/keyring`) – DEVSECOPS_GUIDE §3.
- DB Postgres chỉ bind localhost; mật khẩu sinh ngẫu nhiên lúc install, lưu `.env` trong `%LOCALAPPDATA%\aiw` (ACL user-only).

## 4. Chế độ Server (self-hosted, Docker Compose)

### 4.1 Topology

```
Linux host (Ubuntu 24.04, Docker Engine 27+, Compose v2)
├── caddy        :443  TLS tự động, reverse proxy -> web/api, /preview/* -> preview containers
├── web          nginx:alpine phục vụ apps/web/dist
├── api          ghcr.io/<org>/aiw-api:vX.Y.Z   (NestJS)
├── worker       ghcr.io/<org>/aiw-worker:vX.Y.Z (BullMQ: agent spawn, sandbox, git)
│     volumes: aiw_workspace:/data/aiw, claude_home:/home/aiw/.claude:ro, codex_home:/home/aiw/.codex:ro
├── docker-socket-proxy  (Tecnativa) — chỉ worker gọi; API giới hạn containers/images/exec
├── postgres:16  volume aiw_pg
├── redis:7      volume aiw_redis (AOF)
├── pgbackup     cron pg_dump -> aiw_backups (+ rclone offsite tuỳ chọn)
└── [profile observability] otel-collector, prometheus, grafana, loki
```

### 4.2 Ràng buộc quan trọng: agent CLI chạy ở đâu?

`claude` và `codex` xác thực bằng **login của người dùng** (OAuth/subscription hoặc API key của họ), lưu trong thư mục home của user chạy CLI. Vì vậy:

- Ở **desktop mode**, backend chạy cùng user Windows → CLI dùng login sẵn có; không cần cấu hình thêm. Đây là lý do backend chạy native chứ không trong container (INFRASTRUCTURE_DESIGN §4.1).
- Ở **server mode**, CLI chạy trong container `worker` dưới user `aiw`. Người vận hành phải login **một lần** bên trong container (`docker compose exec worker claude login` – flow thiết bị/browser; tương tự `codex login`), credential lưu vào volume `claude_home`/`codex_home`. Mọi task trên server dùng **cùng một login** → hoá đơn và rate limit dồn về tài khoản đó; không thể phân tách theo người dùng cuối ở MVP. [Giả định về luồng login không cần GUI – cần xác minh với phiên bản CLI; nếu bắt buộc browser cục bộ, login trên máy có GUI rồi copy thư mục credential vào volume, hoặc dùng API key qua biến môi trường do secrets manager cấp]
- Điều khoản dịch vụ của Anthropic/OpenAI về việc dùng subscription login cho nhiều người/automation phải được người dùng tự xác nhận. [Cần người dùng phê duyệt]
- Thay thế: `ANTHROPIC_API_KEY`/`OPENAI_API_KEY` inject vào worker từ secrets store (DEVSECOPS_GUIDE §3) – tách billing rõ hơn, nhưng là API billing.
- Worker cần egress tới `api.anthropic.com`, `api.openai.com` (và endpoint login); sandbox containers thì **không** (INFRASTRUCTURE_DESIGN §6.6).

### 4.3 Cài đặt server mode [Đề xuất]

```bash
mkdir -p /opt/aiw && cd /opt/aiw
curl -L -o compose-bundle.tgz https://github.com/<org>/ai-workspace-app/releases/download/vX.Y.Z/compose-bundle.tgz
tar xzf compose-bundle.tgz            # compose.yml, compose.server.yml, .env.example, Caddyfile, scripts/
cp .env.example .env && $EDITOR .env  # APP_MODE=server, APP_URL, POSTGRES_PASSWORD, SESSION_SECRET, ...
docker compose -f compose.yml -f compose.server.yml pull
docker compose -f compose.yml -f compose.server.yml up -d postgres redis
docker compose -f compose.yml -f compose.server.yml run --rm api pnpm db:migrate
docker compose -f compose.yml -f compose.server.yml up -d
docker compose exec worker claude login && docker compose exec worker claude auth status
curl -fsS https://<host>/api/ready
```

Auth người dùng ở server mode: tài khoản local + password (argon2) hoặc OIDC (sau MVP) – thuộc phạm vi A09/backend.

## 5. Cấu hình môi trường

| Biến | Desktop | Server | Ghi chú |
|---|---|---|---|
| `APP_MODE` | `desktop` | `server` | bật/tắt auth, bind address |
| `API_BIND` | `127.0.0.1` | `0.0.0.0` (sau proxy) | |
| `APP_URL` | `http://127.0.0.1:3000` | `https://aiw.example.com` | CORS/cookies |
| `DATABASE_URL`, `REDIS_URL` | localhost | service name trong compose | |
| `WORKSPACE_ROOT` | `D:\aiw` | `/data/aiw` | volume |
| `DOCKER_HOST` | mặc định npipe | `tcp://docker-socket-proxy:2375` | worker only |
| `SECRETS_BACKEND` | `keychain` | `file-encrypted` (age key từ `AIW_MASTER_KEY` do systemd/credential) hoặc Vault sau | DEVSECOPS_GUIDE |
| `SESSION_SECRET` | sinh lúc install | bắt buộc đặt | ≥ 32 bytes |
| `LOG_LEVEL`, `LOG_PRETTY` | info/true | info/false | |
| `OTEL_ENABLED` | false | tuỳ | |

Nguyên tắc: cấu hình qua env (12-factor), validate bằng zod khi khởi động, không có secret trong image, không có secret trong DB plaintext (A08 lưu tham chiếu + ciphertext).

## 6. Migration khi deploy

- Migration là **forward-only** và phải **backward-compatible một phiên bản** (expand/contract): thêm cột nullable → deploy code → backfill → ràng buộc NOT NULL ở release sau. Lý do: cho phép rollback code mà không rollback DB.
- Chạy migration **trước** khi start phiên bản mới của api/worker (`run --rm api pnpm db:migrate`), dưới advisory lock (công cụ migration của A08 xử lý) để tránh chạy trùng.
- Desktop mode: `aiw.ps1 update` tự backup (`pg_dump`) rồi migrate; nếu migrate lỗi → restore và giữ phiên bản cũ.
- Release notes ghi rõ "Requires migration" và "Destructive migration" (nếu có, phải có bước xác nhận thủ công).

## 7. Rollback

| Tình huống | Cách làm |
|---|---|
| Code lỗi, DB tương thích | `docker compose ... up -d` với tag phiên bản trước (`AIW_VERSION=vX.Y.(Z-1)` trong `.env`); desktop: `aiw.ps1 rollback` dùng bản zip trước đó trong `%LOCALAPPDATA%\aiw\releases\` (giữ 2 bản) |
| Migration lỗi giữa chừng | Công cụ migration chạy trong transaction (Postgres DDL transactional) → tự rollback; sửa và chạy lại |
| Dữ liệu hỏng | `pg_restore` từ backup gần nhất (INFRASTRUCTURE_DESIGN §9); mất dữ liệu kể từ backup → cảnh báo người dùng |
| Worktree/workspace hỏng | Tái tạo worktree từ branch trong `repo.git`; `repo.git` khôi phục từ `git bundle` backup |
| Redis mất | Job đang chạy fail → orchestrator đánh dấu task `interrupted`, người dùng re-run; không mất dữ liệu nghiệp vụ |

Quy trình: ghi `audit_events` loại `deployment` với version trước/sau; rollback cũng là một deployment event.

## 8. Health checks & logs

- `GET /health`: process alive (200).
- `GET /ready`: Postgres, Redis, Docker API, `WORKSPACE_ROOT` writable + free space, `claude --version` (và `auth status` nếu rẻ), phiên bản migration khớp. Trả 503 + JSON chi tiết từng check. Compose `healthcheck` dùng `/ready` cho api/worker; Caddy chỉ route khi healthy.
- Logs: pino JSON ra stdout → Docker logging driver (`json-file`, `max-size=50m`, `max-file=5`) hoặc Loki (profile observability). Desktop: file `%LOCALAPPDATA%\aiw\logs\aiw-YYYY-MM-DD.log` xoay vòng 14 ngày.
- Audit log trong DB (A08) – retention DEVSECOPS_GUIDE §9. Log sandbox/agent nằm ở `artifacts/<taskId>/`, retention theo project quota.
- Alert tối thiểu ở server mode: `/ready` fail > 2 phút, disk `WORKSPACE_ROOT` < 10%, backup thất bại (healthchecks.io ping hoặc email) [Đề xuất].

## 9. Chính sách: không tự động deploy project sinh ra

**Tuyên bố:** AI WORKSPACE **không bao giờ** tự động deploy, publish, hay push code/artefact của project do agent sinh ra lên bất kỳ môi trường production, registry công khai, hay hạ tầng của người dùng mà không có phê duyệt tường minh của con người cho **từng lần** thực hiện.

Cụ thể:

1. Agent không có quyền `git push` (WORKSPACE doc §7); platform chỉ push/tạo PR khi người dùng bấm xác nhận, và chỉ tới remote đã cấu hình.
2. Preview (INFRASTRUCTURE_DESIGN §7) chỉ chạy container cục bộ bind `127.0.0.1` (hoặc sau Caddy có auth ở server mode); không có tích hợp Vercel/Netlify/Fly/AWS ở MVP.
3. Nếu sau này thêm "Deploy to X", tính năng phải: yêu cầu xác nhận có mô tả đích + diff, lưu `audit_events`, có kill-switch cấu hình (`FEATURE_EXTERNAL_DEPLOY=false` mặc định), và không bao giờ được kích hoạt bởi agent qua tool call.
4. Secrets deploy của người dùng (cloud tokens) không bao giờ được truyền vào tiến trình agent hay sandbox.

## 10. Deployment checklist

**Trước khi deploy (server mode)**
- [ ] Release `vX.Y.Z` đã qua CI (`ci-ok`, `security`, image scan) – CI_CD.md.
- [ ] Đọc release notes: migration? destructive? biến env mới?
- [ ] Backup Postgres (`pg_dump -Fc`) và `git bundle` workspace; kiểm tra file backup mở được.
- [ ] `.env` cập nhật biến mới; secrets lấy từ secrets store, không dán vào shell history.
- [ ] Thông báo người dùng (nếu có nhiều) về downtime dự kiến (< 5 phút).

**Deploy**
- [ ] `docker compose pull` đúng tag; `docker image inspect` xác nhận digest khớp release (cosign verify nếu bật).
- [ ] Chạy migration (`run --rm api pnpm db:migrate`), kiểm tra log không lỗi.
- [ ] `up -d`; chờ `/ready` 200 trong ≤ 2 phút.
- [ ] Smoke test: đăng nhập UI, tạo task mock (`AGENT_MODE=mock`) hoặc chạy task thật nhỏ; preview container khởi động/stop được.
- [ ] `claude auth status` trong worker OK.

**Sau deploy**
- [ ] Ghi `audit_events` deployment (tự động) và cập nhật `CHANGELOG`/ops log.
- [ ] Theo dõi log lỗi 30 phút; kiểm tra queue BullMQ không tồn job stuck.
- [ ] Xoá image cũ hơn 2 phiên bản (`docker image prune` có chọn lọc).

**Desktop mode (`aiw.ps1 update`)**: tự động hoá các bước backup → tải → migrate → start → `/ready`; thất bại ở bất kỳ bước nào → rollback và hiển thị hướng dẫn.

## 11. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| Desktop mode là mục tiêu MVP; server mode là compose đơn giản | Người dùng chính là 1 dev trên Windows; tránh phức tạp multi-tenant | [Đề xuất] |
| Backend native trong desktop mode | Dùng login CLI của user, không mount credential vào container | [Đề xuất] |
| Server mode: CLI login trong volume worker, 1 login dùng chung | Cách duy nhất không cần API key; chấp nhận hạn chế billing | [Cần người dùng phê duyệt] |
| Expand/contract migration | Cho phép rollback code độc lập | [Đề xuất] |
| Không auto-deploy project sinh ra | An toàn, trách nhiệm con người (A09) | [Đề xuất – nguyên tắc cứng] |
| Caddy cho TLS | Tự động Let's Encrypt, cấu hình ngắn | [Đề xuất] |

## 12. Dependencies

- A08: công cụ migration, bảng `audit_events`, chiến lược expand/contract trong schema.
- A09: auth server mode, session token desktop mode, điều khoản dùng CLI login.
- CI_CD.md: artifact `compose-bundle.tgz`, image tags, cosign.
- INFRASTRUCTURE_DESIGN.md: compose, docker-socket-proxy, backup.
- Tauri team (sau): sidecar backend, auto-updater (Tauri updater với signing key – CI_CD §4.4).

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| CLI login hết hạn trên server → mọi task fail | `/ready` kiểm tra `auth status`; alert; hướng dẫn re-login |
| Người dùng chạy desktop mode nhưng Docker Desktop tắt | UI wizard + readiness; retry queue |
| Migration destructive mất dữ liệu | Backup bắt buộc trước migrate; cờ xác nhận thủ công |
| Cổng 3000 bị chiếm | Cấu hình `API_PORT`; installer tự chọn port trống |
| Server mode lộ ra internet không có auth | `APP_MODE=server` bắt buộc auth; Caddy mặc định chỉ HTTPS; checklist |
| Tauri updater bị giả mạo | Ký update bằng khoá Tauri, khoá riêng chỉ trong GitHub secrets |

## 14. Vấn đề chưa giải quyết

1. Luồng `claude login`/`codex login` không có browser trong container có hoạt động không (device-code flow?) – cần xác minh.
2. Server mode có hỗ trợ nhiều login CLI theo người dùng (per-user volume) không – ngoài MVP.
3. Chạy backend desktop mode như Windows service (winsw/NSSM) hay chỉ khi mở app.
4. Tauri sidecar: bundle Node runtime (pkg/SEA) hay yêu cầu Node cài sẵn.
5. Chính sách giữ bao nhiêu bản release cục bộ để rollback.

## 15. Acceptance Criteria

- [ ] Desktop: `aiw.ps1 install && aiw.ps1 start` trên máy đã có prerequisites → UI mở, `/ready` 200, wizard phát hiện thiếu `claude login`.
- [ ] `aiw.ps1 update` từ vN → vN+1 thành công với migration; giả lập lỗi migration → tự rollback về vN với dữ liệu nguyên vẹn.
- [ ] Server: compose bundle từ release lên được trên VM Ubuntu sạch trong ≤ 30 phút; `claude auth status` trong worker OK; sandbox job chạy qua docker-socket-proxy.
- [ ] `/ready` trả 503 với lý do khi tắt Redis/Postgres/Docker/hết hạn login.
- [ ] Không tồn tại code path nào push/deploy ra ngoài mà không qua xác nhận UI; kiểm tra bằng test + review A09.
- [ ] Checklist §10 được dùng và lưu kết quả cho mỗi lần deploy.
