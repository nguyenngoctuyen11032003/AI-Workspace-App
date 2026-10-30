# DevSecOps Guide – Secrets, Hardening, Dependencies, SBOM, Incident Response

> **Tác giả:** Agent 14 (DevSecOps & Infrastructure) · **Phase:** 0 · **Trạng thái:** Draft v0.1 · **Ngày:** 2026-10-30
>
> Liên quan: `docs/SECURITY.md` (A09 – threat model, auth, trust boundaries: tài liệu này **không** lặp lại), `docs/DATABASE_DESIGN.md` (A08 – bảng `secrets`, `audit_logs`), `docs/architecture/INFRASTRUCTURE_DESIGN.md`, `docs/development/CI_CD.md`, `docs/development/DEPLOYMENT.md`.
>
> Nhãn: **[Đã xác minh]** / **[Đề xuất]** / **[Giả định]** / **[Cần người dùng phê duyệt]**.

## 1. Mục đích

Quy định vận hành bảo mật hàng ngày cho đội phát triển và người vận hành AI WORKSPACE: quản lý secrets, hardening container, cập nhật dependency, SBOM, triage lỗ hổng, ứng phó sự cố lộ khoá, và lưu giữ audit log. Threat model và kiến trúc bảo mật tổng thể do A09 sở hữu; tài liệu này là "sổ tay thao tác".

## 2. Phạm vi

Platform AI WORKSPACE (code, image, hạ tầng của chính nó) và các cơ chế platform dùng để bảo vệ project sinh ra (sandbox, gitleaks trên diff). Không bao gồm bảo mật ứng dụng nghiệp vụ của project sinh ra (trách nhiệm người dùng), hay chính sách pháp lý/compliance.

## 3. Secrets management

### 3.1 Phân loại

| Loại | Ví dụ | Nơi lưu | Ai được đọc |
|---|---|---|---|
| Credential CLI của người dùng | login Claude Code, Codex | thư mục home do CLI quản lý (`~/.claude`, `~/.codex`) | chỉ tiến trình CLI; platform **không** đọc |
| API key tuỳ chọn | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, GitHub token cho `gh` | **OS keychain** (Windows Credential Manager / macOS Keychain / libsecret) qua `@napi-rs/keyring`; server mode: file mã hoá `age` với master key từ systemd credential hoặc Vault (sau) | backend process khi spawn agent (inject env cho tiến trình con, không log) |
| Secrets hạ tầng | `POSTGRES_PASSWORD`, `SESSION_SECRET`, `REDIS_PASSWORD` | `.env` (dev/desktop, ACL user-only) ; server: `.env` chmod 600 hoặc Docker secrets | api/worker |
| Secrets của project sinh ra | `.env` của app khách | **không** do platform quản lý ở MVP; manifest `forbidden: .env*`; sandbox không mount | không ai |
| CI secrets | Tauri signing key, cosign | GitHub Actions secrets / OIDC | workflow release |

### 3.2 Quy tắc bắt buộc

1. **Không bao giờ** lưu secret plaintext trong Postgres. Bảng `secrets` (A08) chỉ lưu `name`, `backend`, `reference` (keychain id) hoặc `ciphertext` (AES-256-GCM, key từ keychain/master key), `created_at`, `last_used_at`.
2. Biến môi trường chứa secret chỉ hợp lệ ở **dev** (`SECRETS_BACKEND=env`); app log cảnh báo khi `APP_MODE=server` mà dùng `env` backend.
3. Secret không bao giờ được truyền vào: prompt của agent, log (pino redaction), artifact sandbox, response API (chỉ trả `****last4`).
4. Tiến trình agent nhận env **whitelist** (WORKSPACE doc §7.2); sandbox nhận env rỗng ngoài `PATH`, `HOME`, `CI=1`, biến do job khai báo và đã được kiểm tra không khớp pattern secret.
5. Xoay secret hạ tầng ít nhất 12 tháng hoặc ngay khi nghi lộ; `SESSION_SECRET` xoay làm logout mọi phiên – chấp nhận.
6. CLI quản lý: `pnpm aiw secrets set <name>` (đọc từ stdin/prompt ẩn, không nhận qua argv để tránh shell history), `secrets list` (chỉ tên), `secrets rm`.

### 3.3 Ngăn secret vào git

- **gitleaks pre-commit** (lefthook `pre-commit: gitleaks protect --staged --redact`) trong repo platform; CI `gitleaks-action` quét toàn lịch sử PR (CI_CD §4.2).
- `.gitleaks.toml` mở rộng rule: pattern `sk-ant-`, `sk-proj-`, `ghp_`, `github_pat_`, JWT, `-----BEGIN PRIVATE KEY-----`; allowlist cho `*.example`, fixtures test có tiền tố `FAKE_`.
- Platform chạy `gitleaks detect --no-git --source <patch>` trong Lớp 3 diff validation của mỗi task (WORKSPACE doc §7.3) trước merge vào `integration/*`; phát hiện → task `needs_review`, không merge.
- `.gitignore` mẫu cho project sinh ra luôn có `.env*`, `*.pem`, `*.key`, `*.p12`.

## 4. Container hardening checklist

Áp dụng cho sandbox containers (INFRASTRUCTURE_DESIGN §6) và image của chính platform (api/worker/web).

- [ ] Image base pin digest (`node:24-bookworm-slim@sha256:...`), rebuild hàng tuần qua `images.yml`.
- [ ] Multi-stage build; image runtime không chứa compiler/pnpm store/`.git`.
- [ ] `USER` non-root (uid 1000), `WORKDIR` thuộc user đó; `HOME` ghi được chỉ ở tmpfs.
- [ ] `--cap-drop ALL`, không `--cap-add`; `--security-opt no-new-privileges:true`; seccomp mặc định (không `unconfined`).
- [ ] `--read-only` rootfs + `--tmpfs /tmp:noexec,nosuid,size=512m`; chỉ `/work` ghi được (sandbox).
- [ ] `--cpus`, `--memory`, `--memory-swap = memory`, `--pids-limit`, `ulimit nofile`.
- [ ] `--network none` mặc định; egress qua proxy allowlist trên network `internal: true`.
- [ ] Không mount `docker.sock`, `/proc`, `/sys`, `/dev` của host; không `--privileged`, `--pid=host`, `--ipc=host`, `--userns=host`.
- [ ] Không bind mount ngoài `WORKSPACE_ROOT/worktrees/<task>`; path được canonicalize và kiểm tra prefix trước khi mount (chống `..`/junction escape).
- [ ] Healthcheck + `--stop-timeout`; container có label `aiw.task`, `aiw.job`, `aiw.kind` để dọn (`docker container prune --filter label=aiw.kind`).
- [ ] Image platform (api/worker): `NODE_ENV=production`, `npm_config_ignore_scripts=true` khi install trong image, không có `curl`/`wget` trong runtime image nếu không cần.
- [ ] Trivy image scan HIGH/CRITICAL = fail CI; SBOM đính kèm.
- [ ] docker-socket-proxy ở server mode: chỉ bật `CONTAINERS`, `IMAGES`, `EXEC`, `POST=1`; tắt `VOLUMES`, `NETWORKS`, `SYSTEM`, `SWARM`, `BUILD` (build image chỉ do CI). [Đề xuất]
- [ ] Docker Desktop (dev): bật "Use WSL 2 based engine", tắt "Expose daemon on tcp://localhost:2375 without TLS"; Hyper-V backend + ECI chỉ với Docker Business [Giả định].

## 5. Dependency update policy

| Hạng mục | Chính sách [Đề xuất] |
|---|---|
| Lockfile | `pnpm-lock.yaml` commit; CI `--frozen-lockfile`; `minimum-release-age` (pnpm 10) = 3 ngày để tránh package vừa bị chiếm quyền [Cần người dùng phê duyệt] |
| Dependabot | weekly, grouped: `minor-and-patch` (auto-merge khi CI xanh và không phải prod dep "critical" như `@nestjs/*`, `pg`, `ioredis`, `dockerode`), `major` tạo PR riêng cần review |
| Ecosystem | npm, docker (base images), github-actions |
| Node runtime | theo LTS; nâng major trong vòng 3 tháng sau khi LTS active |
| Agent CLI (`claude`, `codex`) | platform ghi version tối thiểu hỗ trợ; `doctor` cảnh báo; test `test:agent-live` thủ công khi nâng |
| Base image | rebuild + scan hàng tuần kể cả không đổi Dockerfile |
| Audit | `pnpm audit --prod --audit-level=high` trong CI; allowlist CVE có hạn dùng ≤ 30 ngày trong `.audit-allowlist.json` với lý do |
| Install scripts | `pnpm` `onlyBuiltDependencies` whitelist (esbuild, @napi-rs/keyring, …) để chặn postinstall của package lạ |
| License | Dependency review chặn AGPL/GPL-3.0 trong dependencies runtime (docs/CLI dev tool OK) [Cần người dùng phê duyệt] |

## 6. SBOM

- Sinh **CycloneDX JSON** cho npm workspace (`@cyclonedx/cyclonedx-npm` hoặc `cdxgen`) và **SPDX** cho image (Trivy/Syft) ở mỗi release (CI_CD §4.3–4.4).
- Lưu: GitHub Release asset + attach vào image bằng `cosign attach sbom`; giữ tối thiểu 24 tháng.
- Dùng SBOM để trả lời "có bị ảnh hưởng bởi CVE-X?" nhanh: `trivy sbom sbom.json` hoặc `grype sbom:sbom.json`.
- Sandbox images: SBOM theo stack giúp người dùng biết toolchain nào dùng để build project của họ.

## 7. Vulnerability triage

| Mức (CVSS/Trivy) | Thời hạn xử lý | Hành động |
|---|---|---|
| Critical (≥ 9.0) hoặc đang bị khai thác (KEV) | 48 giờ | Hotfix release, hoặc tắt tính năng/ngắt egress liên quan |
| High (7.0–8.9) | 7 ngày | Cập nhật ở release kế tiếp; allowlist tạm nếu không reachable (ghi lý do) |
| Medium | 30 ngày | Gom vào Dependabot group |
| Low | best effort | – |

Quy trình: (1) CI/Dependabot/báo cáo → issue nhãn `security` (private nếu repo public → GitHub Security Advisory); (2) xác định reachability (có dùng code path?) và exposure (sandbox? api? dev-only?); (3) quyết định fix/allowlist, ghi vào `SECURITY_LOG.md` hoặc issue; (4) verify bằng rescan; (5) nếu ảnh hưởng người dùng self-hosted → release notes "Security".

Báo cáo từ bên ngoài: `SECURITY.md` (A09) quy định kênh; thời gian phản hồi 3 ngày làm việc.

## 8. Incident response – lộ khoá / secret

**Kịch bản điển hình:** API key Anthropic/OpenAI, GitHub token, hoặc `POSTGRES_PASSWORD` xuất hiện trong commit, log, artifact, prompt agent, hoặc màn hình chia sẻ.

1. **Phát hiện & ghi nhận (0–15 phút):** mở incident ticket (nhãn `incident`), ghi thời điểm, loại secret, nơi lộ, phạm vi (repo public? log gửi đi đâu?).
2. **Thu hồi ngay (≤ 1 giờ):** revoke/rotate khoá tại nhà cung cấp (Anthropic Console, OpenAI dashboard, GitHub token settings); đổi `POSTGRES_PASSWORD`/`SESSION_SECRET` và restart; `claude logout`/`codex logout` nếu credential CLI bị lộ rồi login lại.
3. **Cập nhật hệ thống:** `pnpm aiw secrets set` khoá mới; server: cập nhật secrets store, `docker compose up -d`.
4. **Dọn vết:** nếu trong git history: `git filter-repo` (hoặc BFG) trên repo, force-push sau khi thông báo, invalidate cache GitHub (liên hệ GitHub Support nếu repo public); xoá log/artifact chứa secret (`artifacts/<task>/`), xoá Docker image layer nếu có.
5. **Đánh giá ảnh hưởng:** kiểm tra usage/billing bất thường tại nhà cung cấp trong khoảng thời gian lộ; audit log `audit_logs` cho truy cập bất thường.
6. **Khắc phục gốc:** vì sao gitleaks/redaction không bắt? Thêm rule, test hồi quy.
7. **Postmortem (≤ 5 ngày):** blameless, ghi timeline, root cause, action items, cập nhật tài liệu này.

Checklist "ai làm gì" ở MVP (solo dev): người dùng chính là incident owner; A09 SECURITY.md liệt kê liên hệ.

## 9. Audit log retention

| Loại dữ liệu | Nơi lưu | Retention mặc định [Đề xuất] |
|---|---|---|
| `audit_logs` (spawn agent, hook deny, validation fail/override, merge, deploy, secret set/use, login) | Postgres (A08) | 365 ngày; sau đó export JSONL mã hoá vào `aiw_backups` thêm 2 năm, rồi xoá |
| Application log (pino) | file/Loki | 14 ngày desktop; 30 ngày server |
| Agent transcript / stream-json | `artifacts/<task>/` | theo vòng đời task + 30 ngày; người dùng có thể xoá sớm |
| Sandbox log | `artifacts/<task>/` | 30 ngày hoặc đến khi project xoá |
| Backup Postgres | `aiw_backups` (+ offsite) | 7 daily, 4 weekly, 6 monthly |
| SBOM / release artifact | GitHub Release | ≥ 24 tháng |

Audit event là **append-only** (không có UPDATE/DELETE qua app role; retention job dùng role riêng), chứa `actor` (user/agent/system), `action`, `target`, `taskId`, `ip` (server mode), `result`, `metadata` (không chứa secret). Hash chain (`prev_hash`) tuỳ chọn sau MVP để phát hiện sửa đổi.

## 10. Lịch vận hành định kỳ

| Tần suất | Việc |
|---|---|
| Mỗi commit | gitleaks pre-commit, lint |
| Mỗi PR | CI đầy đủ, dependency review |
| Hàng tuần | Dependabot PR, rebuild + scan sandbox images, review `security` issues mở |
| Hàng tháng | Test restore backup Postgres + `git bundle`; review audit log bất thường; xoay token `gh` nếu dùng PAT |
| Hàng quý | Review allowlist CVE hết hạn, review quyền Docker socket proxy, cập nhật threat model với A09, diễn tập incident lộ khoá (tabletop) |
| Hàng năm | Xoay secrets hạ tầng; review retention |

## 10b. Agent runtime hardening checklist (bổ sung cho sandbox)

Áp dụng mỗi lần platform spawn `claude`/`codex` (WORKSPACE doc §7); là phần vận hành của threat model A09.

- [ ] `cwd` = worktree của task; path đã canonicalize, nằm trong `WORKSPACE_ROOT/worktrees/`.
- [ ] Env whitelist: `PATH`, `HOME`/`USERPROFILE`, `TEMP`/`TMP`, `SystemRoot`, biến CLI cần (`CLAUDE_CONFIG_DIR` nếu dùng); mọi biến khác bị loại; log tên biến bị loại ở mức debug.
- [ ] `.claude/settings.json` và hook script được ghi đè từ template platform **trước mỗi lần spawn**; checksum lưu `audit_logs`.
- [ ] Không bao giờ truyền `--dangerously-skip-permissions` ngoài container sandbox.
- [ ] `--max-turns`, timeout session (`AGENT_SESSION_TIMEOUT_SEC`), giới hạn kích thước stdout (stream-json) 50 MB; vượt → kill + task `failed`.
- [ ] stdin đóng sau khi gửi prompt (tránh CLI chờ tương tác); `GIT_TERMINAL_PROMPT=0`.
- [ ] Nội dung task brief/prompt được render từ template, dữ liệu người dùng và nội dung repo được đặt trong delimiter rõ ràng (prompt-injection mitigation – A09).
- [ ] Sau khi agent dừng: Lớp 3 diff validation + gitleaks trước bất kỳ merge nào; kết quả ghi `audit_logs`.
- [ ] Process tree bị kill hoàn toàn khi huỷ task (Windows: `taskkill /T /F` hoặc Job Object; Linux: process group).

## 10c. Checklist bảo mật trước mỗi release

- [ ] CI `security` job xanh; không có allowlist CVE quá hạn.
- [ ] Trivy image scan cho `aiw-api`, `aiw-worker`, `aiw-web`, 3 sandbox images: 0 HIGH/CRITICAL chưa vá.
- [ ] SBOM sinh và đính kèm release.
- [ ] `pnpm audit --prod` sạch ở mức high.
- [ ] Review diff của `infra/compose/**`, `infra/images/**`, `.github/workflows/**` bởi người thứ hai (hoặc checklist tự review khi solo).
- [ ] Không có thay đổi nào nới lỏng mặc định an toàn (`--network none`, cap-drop, read-only, deny rules) mà không có ADR trong `docs/adr/`.
- [ ] Release notes có mục "Security" nếu có fix CVE hoặc thay đổi hardening.
- [ ] Kiểm tra `doctor` trên máy dev với bản release: keychain hoạt động, `/ready` OK.

## 11. Quyết định & lý do

| Quyết định | Lý do | Trạng thái |
|---|---|---|
| OS keychain cho secrets desktop | Có sẵn, mã hoá theo user, không cần dịch vụ thêm | [Đề xuất] |
| Không đọc credential CLI của người dùng | Giảm bề mặt tấn công, tôn trọng ranh giới CLI | [Đề xuất] |
| gitleaks ở 3 điểm (pre-commit, CI, diff validation) | Bảo vệ cả code platform lẫn code sinh ra | [Đề xuất] |
| Trivy + `pnpm audit` + Dependabot | Chuỗi OSS đủ cho đội nhỏ | [Đề xuất] |
| pnpm `minimum-release-age` 3 ngày | Giảm rủi ro supply-chain (package vừa bị chiếm) | [Cần người dùng phê duyệt] |
| Audit retention 365 ngày online | Cân bằng dung lượng/điều tra | Q-06 (OWR-2 2026-10-30): 365 ngày online + signed export 2 năm |

## 12. Dependencies

- A08: schema `secrets`, `audit_logs` (append-only, role tách biệt), job retention.
- A09: threat model, kênh báo cáo lỗ hổng, auth.
- CI_CD.md: các job security và release SBOM.
- INFRASTRUCTURE_DESIGN.md: tham số hardening sandbox.
- ENVIRONMENT_SETUP.md: cài gitleaks/trivy, lefthook.

## 13. Rủi ro

| Rủi ro | Giảm thiểu |
|---|---|
| Keychain không khả dụng (headless Windows service, Linux không có libsecret) | Fallback `file-encrypted` với master key từ env/credential của service manager, cảnh báo trong `doctor` |
| Agent đọc được secret qua biến môi trường của tiến trình cha | Whitelist env khi spawn; test tự động kiểm tra `process.env` trong agent không chứa pattern secret |
| Allowlist CVE bị quên | Hạn dùng bắt buộc, CI fail khi quá hạn |
| Audit log tăng nhanh (stream event) | Chỉ ghi event có ý nghĩa bảo mật; transcript ở filesystem |
| Người dùng tự dán secret vào prompt | UI cảnh báo khi phát hiện pattern secret trong prompt (regex gitleaks) trước khi gửi |

## 14. Vấn đề chưa giải quyết

1. Thư viện keychain cho Node 24 (`@napi-rs/keyring` vs `keytar` đã ngừng bảo trì) – cần thử trên Windows + Tauri sidecar.
2. Server mode: Vault/SOPS/age – chọn gì cho đội nhỏ.
3. Có mã hoá `artifacts/` (transcript có thể chứa code nhạy cảm) ở rest không.
4. Chính sách license cụ thể (GPL trong dev deps?).
5. Ai là incident owner khi team > 1.

## 15. Acceptance Criteria

- [ ] `pnpm aiw secrets set/list/rm` hoạt động trên Windows (Credential Manager) và Linux (file-encrypted fallback); DB không có cột secret plaintext (kiểm tra schema A08).
- [ ] Test: spawn agent với env chứa `FAKE_SECRET=sk-ant-...` ở tiến trình cha → trong agent `process.env` không có biến đó.
- [ ] Pre-commit chặn commit chứa `sk-ant-api03-...` giả; CI gitleaks chặn PR tương tự; diff validation đánh dấu task `needs_review`.
- [ ] `docker inspect` mọi sandbox container thoả checklist §4 (test tự động trong integration suite).
- [ ] Dependabot cấu hình 3 ecosystem, grouped; `pnpm audit` CI với allowlist có hạn dùng.
- [ ] Release `v0.1.0` có SBOM CycloneDX + SPDX; `trivy sbom` chạy được trên file đó.
- [ ] Diễn tập incident lộ khoá hoàn thành trong ≤ 1 giờ theo §8, có biên bản.
- [ ] Job retention xoá `audit_logs` > 365 ngày sau khi export; role app không có quyền DELETE trên bảng này.
