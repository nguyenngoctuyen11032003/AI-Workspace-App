# PHASE_0_REMAINING_RISKS.md — Sổ rủi ro còn mở sau Phase 0

> Vai trò: Security Reviewer · Ngày: 2026-10-30 · Chỉ tài liệu. **Trạng thái: DRAFT — không có rủi ro nào ở trạng thái Accepted/Closed**; chỉ Owner được đặt `Accepted` (PROJECT_RISKS PR-3). Reviewer không phê duyệt gì.
> Hợp nhất từ: `docs/PROJECT_RISKS.md` (ID `R-*`), `docs/SECURITY.md` §10 (R1–R10), `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` (G-xx, RR-xx), `docs/QA_REVIEW_PHASE0.md`, `docs/adr/CONFLICTS_TO_RESOLVE.md`, `KNOWN_ISSUES.md`, `docs/research/HERDR_ANALYSIS.md`, `docs/research/LICENSE_AND_REUSE.md`.
> Thang: L = likelihood, I = impact (H/M/L, quy đổi từ thang 1–5 của PROJECT_RISKS khi có; còn lại là ước lượng chủ quan của Reviewer [Giả định]). "Blocks Phase 1" = chặn **Phase 1 Foundation** (repo, compose, contract/DB freeze, API skeleton). Mục "no — gate P2-live" nghĩa là không chặn Phase 1 nhưng phải đóng trước khi agent thật có tool chạy (Phase 2). Owner role dùng 18 role seed + `Owner`.
> Dữ kiện máy (read-only, 2026-10-30): Docker daemon không chạy, RAM 15.77 GB (trống 4.18 GB), Codex CLI không trên PATH, `.git` đã tồn tại với remote GitHub, `gitleaks`/`trivy` chưa cài. Chi tiết: checklist mục 4.

## 1. Security (SEC)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| SEC-01 | R1 (R-WIN-SANDBOX): CLI/file tools chạy trên host không OS sandbox; lệnh lọt policy có toàn quyền user trên `C:\`, `D:\` | H (4) | H (4) | Tier 1: policy + hook + rewrite vào container; admin guard; diff gate (SECURITY §2.3) | Owner ký R1 có điều kiện; RV-1 (tắt host Bash) hoặc argument-level allowlist; Tier 2 cho untrusted; PoC hook thật | security | no — gate P2-live | Open — cần Owner ký (EXT-5) |
| SEC-02 | R2 (R-CRED-LEAK): OAuth login do CLI giữ, tiến trình cùng user đọc được | M (3) | H (5) | D-008; Read-deny; không mount home; redaction | Owner ký R2; BitLocker xác nhận; test "không code đọc cred"; sửa mô tả sang ACL (QA P2-05) | security | no | Open |
| SEC-03 | R3 (R-PROMPT-INJ): prompt injection không loại bỏ được | M (3) | H (4) | Chặn hành động; manifest; diff validation; human review merge | `merge_main/prod_deploy/data_deletion` immutable (G-04); corpus PI ≥ 50 chạy thật; Owner ký R3 | security | no | Open |
| SEC-04 | R4: cách ly agent chỉ logical (cùng user OS) | L (2) | H (4) | Worktree riêng, manifest, hook path check, `validateDiff` | `validateDiff` trong container có test; Owner ký R4 | security + backend | no | Open |
| SEC-05 | R5: Docker Desktop = root của VM; **boundary chưa xác minh** vì daemon down | M (3) | H (5) | `assertSafeSpec`, label, fail-closed (SANDBOX §9) | Bật daemon; chạy SB-00..SB-17, EG-01..03, CH-05; ghi `VERIFICATION_LOG.md`; Owner ký R5 | devsecops | no — gate P2-live | Open — cần EXT-3 |
| SEC-06 | R6: egress proxy theo hostname, không phải firewall kernel; domain fronting | M (3) | M (3) | `--network none` mặc định; proxy chỉ phase install | EG-01..03 pass; Owner ký R6 | devsecops | no | Open |
| SEC-07 | R7 (R-DEP-SCRIPTS): install chạy mã bên thứ ba | L (2) | M (3) | `--ignore-scripts`, tuổi package, audit | Test SC-01; lockfile-lint/osv trong CI | devsecops | no | Open |
| SEC-08 | R8: hook/`--settings` là policy runtime; **approval path chưa chứng minh** trên Windows (HTTP hook vs SDK `canUseTool`, G-01) | H (4) | H (4) | Thiết kế hai tầng (PROVIDER §17); 1 probe `canUseTool` doc-reported | PoC SDK 0.3.314 trên Windows (deny/rewrite/hook lỗi ⇒ deny/`input_hash`); sửa SECURITY §5; DECISION_LOG một quy tắc timeout | security + integration | no — gate P2-live (schema approvals: yes) | Open |
| SEC-09 | R9: transcript CLI trên host ngoài redaction | M (3) | M (3) | Khuyến nghị `cleanupPeriodDays`; không đọc | Owner đặt retention; cảnh báo prompt-secret trong UI | security | no | Open |
| SEC-10 | Token UI/hook ghi `server.json`; quyền duyệt lấy được bởi tiến trình cùng user (G-09) | H (4) | H (5) | ACL file (không đủ) | Token không ghi đĩa; handoff in-memory/`invoke`; nonce UI cho class cao; Origin cho web MVP; API-01..05 | security + backend | **yes** (API skeleton ở P1) | Open |
| SEC-11 | Git của orchestrator chạy trên repo agent ghi được thực thi cấu hình/hook độc (G-10) | M (3) | H (5) | Denylist `git config --global` (chưa đủ) | Hardened git invocation, deny `git config`/`-c`, merge trên bare repo agent không ghi, test chứng minh | security + backend | no — gate P2-live | Open — **chưa kiểm chứng**, là phân tích của Reviewer |
| SEC-12 | Phân loại lệnh host thiếu PowerShell/cmd/`wsl`/`docker`; allowlist theo tên lệnh (`rg --pre`, `find -exec`) (G-11) | M (3) | H (5) | Denylist regex | Argument-level allowlist / tắt host Bash; deny `wsl*/docker*/podman*`; corpus CP ≥ 300 | security | no — gate P2-live | Open |
| SEC-13 | `merge_main`/`data_deletion` có thể tắt; POL-11/12 tuỳ chọn (G-04, CF-017) | M (3) | H (5) | R030 trong PERMISSION_MODEL | Immutable policy + seed + test PM-10/PM-13 | security + architect | **yes** (seed policy/contract P1-04) | Open — cần Owner |
| SEC-14 | Roadmap cho agent live (P2/P3) trước policy/API auth/audit (P4) (G-01b) | H (4) | H (5) | Roadmap có EXT-5 ở P4 exit | Sửa roadmap: M0 tối thiểu là điều kiện P2-live; Phase 2 chat tool-off/thư mục hy sinh | pm + security | **yes** (ảnh hưởng scope P1: API auth) | Open |
| SEC-15 | Emergency stop chưa kiểm với SDK-spawned `claude.exe`, Job Object, build 26300 (G-20) | M (3) | H (4) | Thiết kế SANDBOX §7–8 | Launcher + Job Object PoC; SB-15/CH-05 < 5 s; STOP chạy khi Docker down | backend + devsecops | no — gate P2-live | Open |
| SEC-16 | Mật khẩu mặc định Postgres `aiw_dev_only`, Redis không mật khẩu (G-14) | M (3) | H (4) | Bind `127.0.0.1` | Mật khẩu ngẫu nhiên keychain; Redis ACL; role DB tách | db_infra | **yes** (P1-20) | Open |
| SEC-17 | Doc trái D-008: `claude-home` volume, mount `~/.claude` ro (G-12) | M (3) | H (4) | SANDBOX §5 cấm | Sửa INFRASTRUCTURE §4.2, WORKSPACE §7.2, DEPLOYMENT §4 | docs + devsecops | **yes** (doc, trước khi implementer đọc) | Open |
| SEC-18 | Settings người dùng (`skipDangerousModePermissionPrompt: true`, `autoMode`) và repo `.claude/settings.json`/`.mcp.json` ảnh hưởng agent (G-06/G-07) | M (3) | H (4) | `--settings` precedence (docs) | `settingSources: []`, settings tường minh, `strictMcpConfig`, snapshot test, PoC login (CF-047) | integration + security | no — gate P2-live | Open |
| SEC-19 | Audit lệch: `audit_logs` vs `audit_events`, 180 vs 365 ngày, hash chain optional (G-15) | M (3) | M (4) | AUDIT_LOGGING chính tắc | Chốt trong DECISION_LOG; migration §4.3 trước freeze; sửa SECURITY/DEVSECOPS | backend + security | **yes** (schema P1-04) | Open |
| SEC-20 | Agent-to-agent/memory không có quy tắc untrusted (G-13) | M (3) | M (4) | Orchestrator là người đưa thư duy nhất | Spec `<untrusted>`, Redactor, duyệt memory, MCP allowlist; test PI-09/PI-10 | architect + security | no — gate Phase 5 | Open |
| SEC-21 | Container warm dùng chung giữa agent; warm vs ephemeral chưa ADR (G-03, CF-016) | M (3) | M (3) | Recreate sau install/4 h | ADR; ephemeral cho untrusted | architect + devsecops | no | Open |
| SEC-22 | Remote GitHub đã có, visibility chưa biết; chưa gitleaks/trivy; chưa `.gitignore` (G-21) | M (3) | M (3) | Không (Reviewer phát hiện 2026-10-30) | Owner xác nhận private; `.gitignore`; gitleaks pre-commit + CI; cập nhật KNOWN_ISSUES #12 | devsecops | **yes** (P1-01) | Open — cần Owner |
| SEC-23 | R10 / Codex sandbox (`mxc`/`elevated`) chưa kiểm; `~/.codex/config.toml` đang `sandbox="elevated"` + trust đường dẫn | M (3) | M (3) | Adapter dựa fixture, gắn UNVERIFIED | Cài Codex (EXT-2); kiểm `mxc` trên 26300; config do app sinh; Codex tắt mặc định tới lúc đó | integration | no | Open |
| SEC-24 | Approval fatigue với 5–18 agent (AB-01) | M (3) | M (3) | Rate limit 5 ask/phút, gom nhóm, scope hẹp | Test PM-12 với 18 agent giả lập | security | no | Open |

## 2. Provider ToS / quota (PROV)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| PROV-01 | R-TOS-QUOTA: điều khoản cho nhiều phiên CLI song song dùng chung một tài khoản `team` chưa xác minh (CF-044) | H (4) | H (4) | Mặc định concurrency 5; tăng chỉ khi có bằng chứng | Owner đọc/xác nhận điều khoản bằng văn bản (EXT-7); ghi gói/seat | pm + Owner | no (chặn P7, N>5) | Open |
| PROV-02 | Dùng đăng nhập OAuth subscription qua Agent SDK/app bên thứ ba có thể không nằm trong điều khoản cho phép [Giả định, chưa nghiên cứu trong Phase 0] | M (3) | H (5) | D-004 kế thừa login CLI (đã chạy 1 lần) | Xác minh điều khoản Anthropic/OpenAI cho dùng SDK/CLI headless tự động bằng subscription; nếu không: API key (đổi chi phí) | research + Owner | no (gate P2-live) | Open |
| PROV-03 | R-QUOTA-RATELIMIT: hạn mức 5 giờ đã làm 3 agent dừng giữa chừng (KNOWN_ISSUES #4) | H (5) | H (4) | Benchmark đầu cửa sổ; backoff; checkpoint | Số đo T1–T4 (EXT-4); gauge từ `rate_limit_event`; chế độ suy giảm | performance | no | Open |
| PROV-04 | R-CODEX-MISSING: Codex CLI chưa cài; AC-022 không kiểm được | M (3) | M (3) | Claude là provider chính; UNVERIFIED | EXT-2 + live smoke | integration | no | Open — cần EXT-2 |
| PROV-05 | Trôi phiên bản SDK 0.3.x / app-server experimental / CLI 2.1.312 | H (4) | M (3) | Pin, canary đêm, mapper ignore | Contract test + canary xanh qua 2 lần nâng version | integration | no | Open |
| PROV-06 | OpenHands: ACP vs D-008; native cần LLM API key (CF-024) | L (2) | M (3) | ACP tắt, chỉ native + keychain | Owner duyệt chi phí API key hoặc loại OpenHands khỏi MVP | integration | no | Open |

## 3. Technical feasibility (TECH)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| TECH-01 | R-RAM-16GB: 15.77 GB tổng, 4.18 GB trống; khuyến nghị `.wslconfig 12GB` sai máy (G-05, QA P1-12) | H (4) | M (3) | Mặc định 5 agent; host guard (thiết kế) | Tham số hoá `.wslconfig` (6–8 GB), số đo thật, hoặc nâng RAM/chia máy | performance | no | Open |
| TECH-02 | R-DOCKER-DOWN: daemon không chạy; ECI không có với WSL2 (Giả định); bind mount 9P chậm | H (4) | M (3) | `pnpm doctor`; `/ready` 503; test `SKIPPED-NO-DOCKER` | Bật Docker Desktop; chạy `docker info`; quyết định EXT-8 | devsecops | **yes** (P1-20 integration; DB cần Docker) | Open — cần EXT-3 |
| TECH-03 | 18 đồng thời chưa chứng minh; chỉ số đo 3–5 process | M (3) | H (4) | Mặc định 5, nâng 5→10→18 theo bằng chứng | P7-01 T3–T7 | performance + qa | no | Open |
| TECH-04 | R-ENGINE-COMPLEXITY: tự xây Conductor (D-003) | M (3) | H (4) | Property/chaos test; đánh giá lại nếu trễ | Chaos P3-13; crash recovery 20/20 (AC-127) | architect + backend | no | Open |
| TECH-05 | Drizzle + bảng phân vùng + FK vòng viết tay; nhãn "v1.x" thực tế beta (QA concern 6, P1-08) | M (3) | M (3) | Migration tay có kế hoạch | Spike partition/migration 1 ngày trước freeze; ghi phiên bản thực | db_infra | **yes** (P1 DB) | Open |
| TECH-06 | Quoting/đường dẫn có khoảng trắng (`Agent management`) cho `aiw-exec`/hook trên PowerShell (QA concern 10) | H (4) | M (3) | Chưa có | Test quoting trên PowerShell + `cmd` với path có space | backend | no | Open |
| TECH-07 | Job Object/`aiw-job.exe` vs `koffi` trên build 26300 chưa thử | M (3) | M (4) | Thiết kế SANDBOX §7 | PoC (xem SEC-15) | backend | no | Open |
| TECH-08 | Phaser 3.90 vs 4.x, tile 16/32, FPS 30 với 18 nhân vật (R-OFFICE-PERF, CF-018..020) | L (2) | L (2) | Tối ưu atlas; reduced-motion | Đo trên máy tham chiếu | frontend | no | Open |
| TECH-09 | Skill/memory nạp khác nhau hai CLI (`--add-dir` vs junction `.agents/skills`) chưa thử Codex | M (3) | L (2) | SKILL_SYSTEM | Test khi có Codex | integration | no | Open |

## 4. Scope (SCOPE)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| SCOPE-01 | R-SCOPE-SIZE: 192 FR, 100 AC, 22 màn hình, 44 bảng, 3 provider | H (4) | M (3) | Phase hoá, thứ tự hy sinh định sẵn | Owner duyệt MoSCoW + thứ tự hy sinh (PR-4) | pm | no | Open |
| SCOPE-02 | 55 mâu thuẫn (37 cần Owner duyệt) chưa hoà giải trong tài liệu nguồn; contract drift (R-CONTRACT-DRIFT) | H (4) | M (3) | CONFLICTS_TO_RESOLVE đề xuất xử lý | Owner duyệt CF; vá tài liệu nguồn; contract freeze 0.1 | architect | **yes** (P1-04 freeze) | Open |
| SCOPE-03 | Chưa quyết định nào được duyệt (DECISION_LOG: tất cả "Chờ duyệt"); R-APPROVAL-LAG; bus factor = 1 | H (4) | M (3) | Gom quyết định một bảng; D-001/D-002 duyệt trước | EXT-1 | Owner | **yes** (EXT-1) | Open |
| SCOPE-04 | QA P0-1: 50/199 FR (31 Must/MVP) không có AC; 30 mã AC chưa định nghĩa | H (5) | M (3) | Ghi trong KNOWN_ISSUES #6 | BA bổ sung AC + bảng truy vết | ba | **yes** (trước exit P1, AC nghiệm thu) | Open |
| SCOPE-05 | R-MERGE-CONFLICT khi nhiều agent sửa file chồng lấn | M (3) | M (3) | Manifest không chồng lấn; merge mutex | Test GIT-02/03 | backend + architect | no | Open |

## 5. Licensing (LIC)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| LIC-01 | R-AGPL-CONTAM: nhiễm AGPL-3.0 từ My Virtual Office | L (2) | H (5) | D-012: chỉ tham khảo ý tưởng; clean-room; manifest asset; CI license | ADR clean-room (QA P1-10), lint chặn path MVO, review giấy phép mỗi PR Office | reviewer + research | no | Open |
| LIC-02 | **herdr `vendor/` chưa kiểm giấy phép** (HERDR_ANALYSIS §126, §137; KNOWN_ISSUES #11). Lưu ý: không có thư mục `vendor/` trong repo AI-Workspace-App (Đã xác minh (lệnh)); rủi ro nằm ở repo herdr tham chiếu | M (3) | M (3) | Quy tắc "không sao chép phần chưa kiểm"; ADR-0020 chỉ tham chiếu thiết kế | Kiểm giấy phép từng thành phần `vendor/` của herdr, ghi vào REUSE_DECISIONS; nếu không rõ ⇒ cấm dùng | reviewer + research | no (chỉ cần trước khi dùng bất kỳ file herdr nào) | Open |
| LIC-03 | **herdr `assets/sounds` / âm thanh thông báo chưa kiểm giấy phép** (HERDR_ANALYSIS §120, §137; HERDR_DESIGN_INTEGRATION tham chiếu `src/sound.rs`); không thấy file âm thanh trong repo này (find ≤ 3 cấp) | M (3) | M (3) | Đánh dấu "Tham khảo", không dùng file | Chọn âm thanh CC0/MIT/OFL có manifest, hoặc bỏ tính năng âm thanh | uiux + reviewer | no | Open |
| LIC-04 | **JIK-A-4 / MetroCity sprite: trạng thái giấy phép mâu thuẫn** (CF-025): PIXEL_AGENTS_ANALYSIS §0/§5.3 ghi chưa xác minh; LICENSE_AND_REUSE §1.3/§6.1 ghi CC0 "đã xác minh" bằng web search, **không lưu bằng chứng**; QA chỉ xác nhận README Pixel Agents:48 có credit, không xác nhận giấy phép. Id commit Pixel Agents `add749a` không kiểm được | M (3) | M (3) | Quy tắc: asset chỉ CC0/MIT/OFL + manifest | Coi là **chưa xác minh** tới khi chụp trang license itch.io (CC0) vào PR; lấy từ nguồn gốc, không qua bản Pixel Agents; sửa hai tài liệu cho khớp | reviewer + uiux | no (chặn Phase 6) | Open |
| LIC-05 | CC0-only (yêu cầu) vs D-012/LICENSE_AND_REUSE cho phép MIT/OFL/CC-BY (CF-026) | M (3) | L (2) | THIRD_PARTY_NOTICES (kế hoạch) | Owner chọn chính sách asset | Owner | no | Open — cần Owner |
| LIC-06 | R-ASSET-LICENSE: asset không rõ nguồn/giấy phép nói chung | L (2) | M (3) | Manifest bắt buộc, CI chặn | `assets-manifest.json` + CI | uiux + reviewer | no | Open |
| LIC-07 | Giấy phép của chính dự án chưa chọn; `THIRD_PARTY_NOTICES.md`, `CONTRIBUTING.md` chưa tồn tại; herdr/ChatDev Apache-2.0 cần NOTICE/change notice; không dùng tên/logo OpenHands | M (3) | M (3) | REUSE_DECISIONS nêu attribution | Owner chọn LICENSE (P1-01); tạo NOTICE; license-checker trong CI | docs + reviewer | **yes** (P1-01 cần LICENSE) | Open — cần Owner |

## 6. Cost (COST)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| COST-01 | R-COST-RUNAWAY: spawn tax ≈ 48k token/spawn (≈ 0,37 USD list); 18 agent ≈ 250–540 USD list/ngày [Giả định]; vòng lặp/retry | H (5) | H (4) | Budget cứng, `max_turns`, Emergency Stop, cảnh báo 50/80/100% (thiết kế) | Triển khai + test budget; giảm spawn tax; hạn mức chi tiêu cấp tài khoản | performance + backend | no | Open |
| COST-02 | Tier 2 dùng API credits (không subscription) chưa có ngân sách | M (3) | M (3) | Opt-in cho untrusted | Owner duyệt trần chi phí Tier 2 | Owner | no | Open — cần Owner |
| COST-03 | Benchmark T1–T4 ≈ 89 spawn ≈ 33 USD list cần duyệt (EXT-4) | M (3) | L (2) | Chưa chạy | Owner duyệt EXT-4 | Owner | no | Open |
| COST-04 | Docker Business cho ECI không áp dụng WSL2; không có ngân sách thay thế [Giả định] | L (2) | L (2) | Không dùng ECI | Ghi rõ trong ADR-0007 | devsecops | no | Open |

## 7. Documentation (DOC)

| ID | Risk | L | I | Current mitigation | What would close it | Owner role | Blocks Phase 1 | Status |
|---|---|---|---|---|---|---|---|---|
| DOC-01 | Tài liệu nguồn chưa vá theo giá trị chính tắc; người triển khai có thể đọc giá trị cũ (QA §6 nhóm) | H (4) | M (3) | CONFLICTS_TO_RESOLVE | Vá nguồn sau khi Owner duyệt CF | docs + architect | **yes** (contract-related) | Open |
| DOC-02 | Nhãn "Đã xác minh" dùng cho "khớp doc khác" (KNOWN_ISSUES #9, QA P2-06) | M (3) | M (3) | — | Dùng nhãn "Đã xác minh (lệnh)" / "(doc)" như checklist | docs | no | Open |
| DOC-03 | Trạng thái lỗi thời: KNOWN_ISSUES #12/R-NO-GIT nói chưa có git, thực tế có `.git` + remote (Đã xác minh (lệnh)) | H (5) | L (2) | — | Cập nhật KNOWN_ISSUES, PROJECT_RISKS, PROJECT_STATUS | pm + docs | no | Open |
| DOC-04 | Liên kết hỏng (`docs/security/SECURITY.md`, `docs/architecture/DATABASE_DESIGN.md`, `docs/DECISION_LOG.md`); `VERIFICATION_LOG.md` chưa tồn tại (QA P2-03) | H (5) | L (2) | — | Sửa đường dẫn; tạo `VERIFICATION_LOG.md` khi có bằng chứng | docs | no | Open |
| DOC-05 | SECURITY.md chưa đồng bộ AUDIT_LOGGING/PROVIDER (hook HTTP, `audit_events`, 180 ngày, "617 byte/644") (QA P1-01, P1-02, P1-05, P2-05) | H (4) | M (3) | Doc con đã là chính tắc | Viết lại SECURITY §5, §8; bỏ số byte/mode | security | **yes** (trước schema freeze) | Open |
| DOC-06 | Prisma còn trong `AGENT_ORGANIZATION.md`, SECURITY, PERMISSION_MODEL (manifest ownership path `prisma/**`) — diff gate không bảo vệ thư mục Drizzle (QA P1-06) | H (4) | M (3) | — | Đổi sang path Drizzle thực | architect | **yes** (manifest mẫu P1) | Open |
| DOC-07 | Event naming UPPER_SNAKE vs dotted; D-014 dẫn nguồn không có trong `docs/` (QA P1-13) | H (4) | M (3) | CF-003 đề xuất | Chép đặc tả vào `docs/`; chốt quy ước | architect | **yes** (P1-04) | Open |

## 8. Tổng hợp

- Tổng số dòng: SEC 24, PROV 6, TECH 9, SCOPE 5, LIC 7, COST 4, DOC 7 = **62**.
- Rủi ro đánh dấu **yes** cho Phase 1: SEC-10, SEC-13, SEC-14, SEC-16, SEC-17, SEC-19, SEC-22, TECH-02, TECH-05, SCOPE-02, SCOPE-03, SCOPE-04, LIC-07, DOC-01, DOC-05, DOC-06, DOC-07 (17 mục). Các mục còn lại không chặn Phase 1 nhưng nhiều mục chặn **P2-live** (xem cột).
- Không có mục nào ở `Accepted`/`Closed`. Mọi chấp nhận thuộc Owner (ký trong `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` mục 2 và 5).
- Phase 1 **chưa** được bắt đầu theo yêu cầu.
