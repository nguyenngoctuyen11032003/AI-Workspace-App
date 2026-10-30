# Environment Setup – Windows 11 Developer Machine

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Liên quan: `docs/architecture/INFRASTRUCTURE_DESIGN.md`, `docs/architecture/WORKSPACE_AND_GIT_ISOLATION.md`, `docs/development/CI_CD.md`, `docs/operations/DEVSECOPS_GUIDE.md`, `docs/SECURITY.md` (A09).
>
> Nhãn: **[Đã xác minh]** / **[Đề xuất]** / **[Giả định]** / **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Hướng dẫn từng bước để một developer mới thiết lập máy Windows 11 chạy được AI WORKSPACE ở chế độ dev: backend NestJS + frontend Vite chạy native, Postgres/Redis trong Docker, cùng các AI coding-agent CLI (Claude Code, Codex) đã đăng nhập.

## 2. Phạm vi

Chỉ Windows 11 (máy dev hiện tại). Linux/macOS sẽ được bổ sung khi có nhu cầu (phần lớn lệnh giống nhau). Không bao gồm cài đặt production (xem DEPLOYMENT.md).

## 3. Prerequisites

| Công cụ | Phiên bản yêu cầu | Trên máy hiện tại | Trạng thái |
|---|---|---|---|
| Windows 11 | 22H2+ (WSL2) | Pro for Workstations 10.0.26300 | [Đã xác minh] |
| Docker Desktop | ≥ 4.40 (client 29.x), WSL2 backend | client 29.8.1, Compose v5.5.1 | [Đã xác minh] |
| WSL | 2 | WSL2, distro `docker-desktop` | [Đã xác minh] |
| Git for Windows | ≥ 2.45 | 2.56.0.windows.1 | [Đã xác minh] |
| GitHub CLI `gh` | ≥ 2.60 | 2.102.0 | [Đã xác minh] |
| Node.js | 24 LTS (`.nvmrc` = `24`) | 24.19.0 | [Đã xác minh] |
| npm | đi kèm Node | 11.17.0 | [Đã xác minh] |
| pnpm | 10.x (qua corepack) | 10.34.6, corepack 0.35.0 | [Đã xác minh] |
| Claude Code CLI | ≥ 2.1 | 2.1.312 | [Đã xác minh] |
| Codex CLI | mới nhất | **chưa cài** | [Đã xác minh] |
| trivy, gitleaks | mới nhất (tuỳ chọn ở dev, bắt buộc trong CI) | chưa cài | [Đã xác minh] |
| VS Code (khuyến nghị) | mới nhất | – | [Giả định] |

Yêu cầu phần cứng khuyến nghị: 8 core / 32 GB RAM / SSD ≥ 100 GB trống (sandbox + worktrees). Ổ D: hiện còn 109 GB. [Đã xác minh]

Package manager: **pnpm** (workspace monorepo) – xem quyết định §11 [Cần người dùng phê duyệt].

## 4. Cài đặt từng bước

### 4.1 Bật WSL2 và Docker Desktop

```powershell
# PowerShell (Admin)
wsl --install --no-distribution      # nếu chưa có WSL
wsl --update
wsl --status                          # Default Version: 2
```

Cài Docker Desktop (winget: `winget install Docker.DockerDesktop`). Trong Settings:

- General → **Use the WSL 2 based engine** (bật).
- Resources → WSL Integration: bật cho distro dev nếu dùng.
- Resources → Advanced (hoặc `%USERPROFILE%\.wslconfig`):

```ini
[wsl2]
memory=12GB
processors=6
swap=0
localhostForwarding=true
```

[Cần người dùng phê duyệt – giá trị tuỳ RAM máy]

Khởi động Docker Desktop và kiểm tra:

```powershell
docker info --format '{{.OSType}} {{.ServerVersion}} cgroup={{.CgroupVersion}} {{.SecurityOptions}}'
docker run --rm hello-world
```

Lưu ý: tại thời điểm viết, daemon **chưa chạy** trên máy dev nên các giá trị cgroup/kernel chưa xác minh.

### 4.2 Git cấu hình

```powershell
# Admin – bật long path ở Windows và Git
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force
git config --system core.longpaths true

# Người dùng
git config --global user.name  "<Tên>"
git config --global user.email "<email>"
git config --global core.autocrlf false      # system hiện là true -> override; repo có .gitattributes eol=lf
git config --global init.defaultBranch main
git config --global pull.rebase true
gh auth login                                # chọn GitHub.com, HTTPS, browser
gh auth status
```

Lý do `core.autocrlf=false`: system config hiện là `true` [Đã xác minh]; script chạy trong container Linux cần LF, và diff giả toàn file sẽ làm hỏng validation manifest (WORKSPACE doc §11). [Cần người dùng phê duyệt vì đổi thiết lập toàn cục]

### 4.3 Node 24 + pnpm

```powershell
node -v          # v24.x
corepack enable
corepack prepare pnpm@10 --activate
pnpm -v
pnpm config set store-dir D:\pnpm-store      # store chung, giảm disk cho nhiều worktree
```

Nếu cần nhiều phiên bản Node: dùng `fnm` (`winget install Schniz.fnm`) và `.nvmrc`.

### 4.4 Clone repo và cài dependency

```powershell
git clone https://github.com/<org>/ai-workspace-app.git D:\Code\ai-workspace-app
cd D:\Code\ai-workspace-app
pnpm install --frozen-lockfile
pnpm run prepare          # cài git hooks (lefthook/husky): gitleaks + lint-staged
```

Cấu trúc monorepo dự kiến (Phase 1): `apps/api` (NestJS), `apps/web` (React/Vite), `apps/desktop` (Tauri, sau), `packages/shared`, `packages/workspace`, `packages/sandbox`, `infra/compose`, `infra/images`.

### 4.5 Postgres + Redis qua Docker Compose

```powershell
cd infra\compose
docker compose up -d
docker compose ps          # postgres, redis: healthy
```

### 4.6 Biến môi trường

Copy `.env.example` → `.env` (file `.env` nằm trong `.gitignore`, gitleaks pre-commit chặn nếu lỡ add). Nội dung `.env.example` [Đề xuất]:

```dotenv
# ---- App ----
NODE_ENV=development
APP_MODE=desktop                 # desktop | server
API_PORT=3000
WEB_PORT=5173
APP_URL=http://localhost:5173
LOG_LEVEL=debug                  # pino level
LOG_PRETTY=true

# ---- Data stores ----
DATABASE_URL=postgresql://aiw:aiw_dev_only@127.0.0.1:5432/aiw
REDIS_URL=redis://127.0.0.1:6379
POSTGRES_PASSWORD=aiw_dev_only   # chỉ dev; compose đọc biến này

# ---- Workspace ----
WORKSPACE_ROOT=D:\aiw
WORKSPACE_MAX_WORKTREES_PER_PROJECT=6
WORKSPACE_QUOTA_GB_PER_PROJECT=10

# ---- Sandbox ----
DOCKER_HOST=                      # để trống = mặc định (npipe trên Windows)
SANDBOX_MAX_CONCURRENCY=3
SANDBOX_DEFAULT_CPUS=2
SANDBOX_DEFAULT_MEMORY_MB=4096
SANDBOX_DEFAULT_TIMEOUT_SEC=900
SANDBOX_EGRESS_ALLOWLIST=registry.npmjs.org,pypi.org,files.pythonhosted.org,repo.maven.apache.org,github.com

# ---- Agent CLIs ----
CLAUDE_BIN=claude
CODEX_BIN=codex
AGENT_MAX_TURNS=50
AGENT_SESSION_TIMEOUT_SEC=3600
# KHÔNG đặt ANTHROPIC_API_KEY / OPENAI_API_KEY ở đây: CLI dùng login riêng của người dùng.
# Nếu bắt buộc dùng API key, lưu vào OS keychain qua lệnh `pnpm aiw secrets set` (xem DEVSECOPS_GUIDE).

# ---- Security ----
SESSION_SECRET=change-me-dev-only
SECRETS_BACKEND=keychain          # keychain | env (chỉ dev)

# ---- Observability (tuỳ chọn) ----
OTEL_ENABLED=false
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
```

Validation env bằng `zod` lúc khởi động; thiếu biến bắt buộc → app dừng với thông báo rõ.

### 4.7 Database migration & seed

```powershell
pnpm --filter api db:migrate      # công cụ migration do A08 quyết định (ORM theo D-002, đề xuất, chờ duyệt)
pnpm --filter api db:seed         # dữ liệu mẫu dev
```

### 4.8 Chạy ứng dụng

```powershell
pnpm dev            # pnpm -r chạy song song: api (3000), web (5173), worker
```

Mở `http://localhost:5173`. Kiểm tra `http://localhost:3000/ready` trả `200` với các check `postgres`, `redis`, `docker`, `workspaceRoot`, `claudeCli`.

## 5. Claude Code CLI

```powershell
npm i -g @anthropic-ai/claude-code      # hoặc cập nhật: claude update
claude --version                         # 2.1.312 [Đã xác minh]
claude login                             # đăng nhập bằng tài khoản Claude (browser)
claude auth status                       # hoặc: claude -p "ping" --max-turns 1
```

Kiểm tra non-interactive mode mà platform dùng:

```powershell
claude -p "Trả lời đúng một từ: OK" --output-format json --max-turns 1
```

Lưu ý: login lưu ở `%USERPROFILE%\.claude` (và Credential Manager trên Windows tuỳ phiên bản). Platform **không** đọc/ghi thư mục này; chỉ spawn `claude` với cùng user. [Giả định về vị trí lưu – cần xác minh trên bản 2.1.312]

## 6. Codex CLI

```powershell
npm i -g @openai/codex
codex --version
codex login                              # ChatGPT account hoặc API key
codex login status
codex exec --sandbox read-only --cd D:\tmp\probe "Liệt kê file trong thư mục này"
```

Trạng thái hiện tại: **chưa cài** [Đã xác minh]. Sandbox của Codex trên Windows native có thể hạn chế hơn so với macOS/Linux (seatbelt/landlock); nếu `workspace-write` không khả dụng trên Windows, platform dựa vào Lớp 3 validation (WORKSPACE doc §7.3) và/hoặc chạy Codex trong WSL2. [Giả định – cần xác minh khi cài]

## 7. Công cụ bảo mật (tuỳ chọn ở dev)

```powershell
winget install Gitleaks.Gitleaks
winget install AquaSecurity.Trivy
gitleaks version; trivy --version
gitleaks git --redact --no-banner .     # quét lịch sử repo
```

Git hook pre-commit (lefthook) gọi `gitleaks protect --staged` – cài tự động bởi `pnpm run prepare`. Nếu chưa cài gitleaks, hook sẽ cảnh báo nhưng không chặn ở dev (CI chặn). [Đề xuất]

## 8. Lệnh kiểm tra setup (`pnpm doctor`) [Đề xuất]

Script `scripts/doctor.ts` kiểm tra và in bảng PASS/FAIL:

| Kiểm tra | Lệnh | Kỳ vọng |
|---|---|---|
| Node | `node -v` | ≥ 24 |
| pnpm | `pnpm -v` | 10.x |
| Docker daemon | `docker info` | OSType=linux, không lỗi pipe |
| Compose services | `docker compose ps --format json` | postgres, redis healthy |
| Postgres | `SELECT 1` qua `DATABASE_URL` | OK |
| Redis | `PING` | PONG |
| Git long paths | `git config --get core.longpaths` + registry | true / 1 |
| Git autocrlf | `git config --get core.autocrlf` | false (repo hoặc global) |
| WORKSPACE_ROOT | tạo/xoá file thử | writable, free ≥ 20 GB |
| Claude CLI | `claude --version`, `claude auth status` | OK, logged in |
| Codex CLI | `codex --version`, `codex login status` | OK (warning nếu thiếu) |
| gh | `gh auth status` | logged in (warning nếu thiếu) |
| gitleaks/trivy | `--version` | warning nếu thiếu |

## 9. Sự cố thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine` | Docker Desktop chưa chạy | Mở Docker Desktop, chờ "Engine running"; `docker context use desktop-linux` |
| `Filename too long` khi `pnpm install`/checkout | long paths chưa bật | §4.2 (registry + `core.longpaths`) |
| Diff toàn file `^M` / script `.sh` lỗi `\r` trong container | `core.autocrlf=true` | `git config --global core.autocrlf false`; `git add --renormalize .` |
| `EPERM: operation not permitted, symlink` | `core.symlinks=false`, user không có quyền symlink | Không dùng symlink; pnpm dùng junction – OK; bật Developer Mode nếu cần |
| Port 5432/6379 đã dùng | Postgres/Redis cài native | Đổi port trong compose + `.env`, hoặc tắt service native |
| `pnpm install` chậm/treo | Defender scan `node_modules` | Loại trừ `D:\Code`, `D:\pnpm-store`, `D:\aiw` khỏi realtime scan [Cần người dùng phê duyệt] |
| `claude -p` treo chờ login | chưa `claude login` hoặc token hết hạn | `claude login`; kiểm tra `claude auth status` |
| Bind mount vào container rỗng | path Windows không được Docker Desktop share (file sharing) | Docker Desktop → Resources → File sharing thêm `D:\aiw` (WSL2 thường tự động) |
| `WSL2 vmmem` chiếm RAM | thiếu `.wslconfig` | §4.1; `wsl --shutdown` để áp dụng |
| `git worktree add` lỗi `is already checked out` | branch đang được worktree khác dùng | `git worktree list`; `git worktree prune` |

## 10. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| pnpm + workspaces | Store chung tiết kiệm disk cho nhiều worktree, strict hoisting giúp sandbox tái tạo đúng deps, nhanh | [Cần người dùng phê duyệt] (thay thế: npm workspaces) |
| Turborepo làm task runner | Cache lint/test/build theo package, hỗ trợ `--filter` và remote cache (tuỳ chọn) | Hoãn: D-001 APPROVED chọn `pnpm -r` thuần (2026-10-30) |
| Backend chạy native, chỉ DB/Redis trong Docker | Spawn CLI với login của user; hot reload nhanh | [Đề xuất] |
| `.env` chỉ cho dev, secrets thật trong OS keychain | Tránh lộ key qua repo/log | [Đề xuất] (DEVSECOPS_GUIDE) |
| `core.autocrlf=false` global | Nhất quán LF với container | [Cần người dùng phê duyệt] |

## 11. Dependencies

- A08 quyết định công cụ migration → cập nhật §4.7.
- INFRASTRUCTURE_DESIGN.md §5 cho nội dung compose.
- DEVSECOPS_GUIDE.md cho `secrets set` và pre-commit.
- Orchestrator team cho flag spawn `claude -p` chính xác.

## 12. Rủi ro

- Thiết lập cần quyền Admin (long paths, Defender exclusion) – developer không có quyền sẽ bị chặn ở `pnpm install`.
- Codex trên Windows chưa xác minh được sandbox → có thể cần WSL2 distro riêng (Ubuntu) làm môi trường chạy agent.
- Phiên bản Claude Code cập nhật nhanh; flag CLI có thể đổi → `doctor` kiểm tra phiên bản tối thiểu và ghi vào `docs/`.

## 13. Vấn đề chưa giải quyết

1. ĐÃ CHỐT (D-001): pnpm workspaces thuần, không Turborepo.
2. Có cài distro Ubuntu WSL2 riêng để chạy agent CLI/Linux tooling không?
3. Vị trí `WORKSPACE_ROOT` (`D:\aiw`?) và `pnpm store` (`D:\pnpm-store`?).
4. Tauri prerequisites (Rust toolchain, MSVC Build Tools, WebView2) – thêm vào tài liệu này khi desktop shell bắt đầu.

## 14. Acceptance Criteria

- [ ] Developer mới theo tài liệu này đi từ máy sạch đến `pnpm dev` chạy được trong ≤ 60 phút (không tính thời gian tải).
- [ ] `pnpm doctor` PASS toàn bộ mục bắt buộc trên máy dev hiện tại.
- [ ] `.env.example` có đủ biến mà `zod` schema yêu cầu; `.env` không bao giờ được commit (gitleaks + `.gitignore`).
- [ ] `claude -p` và (khi cài) `codex exec` chạy được từ backend với user hiện tại.
- [ ] Bảng sự cố thường gặp được cập nhật sau mỗi onboarding.
