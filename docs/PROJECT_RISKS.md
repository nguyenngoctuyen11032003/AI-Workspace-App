# PROJECT RISKS — Sổ đăng ký rủi ro dự án AI WORKSPACE

Trạng thái: DRAFT (Phase 0). Ngày: 2026-10-30. Tác giả: Delivery/Engineering Lead.
Ký hiệu: [Đã xác minh] / [Đề xuất] / [Giả định] / [Cần người dùng phê duyệt].
Liên quan: `SECURITY.md` §10 (R1-R10 bảo mật), `research/MULTI_AGENT_SCALABILITY.md`, `research/LICENSE_AND_REUSE.md`, `architecture/RESOURCE_MANAGEMENT.md`, `development/ENVIRONMENT_SETUP.md`, `IMPLEMENTATION_ROADMAP.md`, `../DECISION_LOG.md`.

---

## 1. Mục đích

Một sổ đăng ký rủi ro duy nhất ở cấp dự án: nhận diện, đánh giá (likelihood × impact), giảm thiểu, gán vai trò chịu trách nhiệm, đặt trigger để biết khi nào rủi ro xảy ra, và theo dõi trạng thái xuyên 8 phase. Sổ này hợp nhất rủi ro rải rác trong từng tài liệu thiết kế và đánh dấu rủi ro nào cần Owner ký chấp nhận.

## 2. Phạm vi

- Trong: rủi ro kỹ thuật, bảo mật, pháp lý/giấy phép, vận hành, chi phí, tiến độ và phạm vi của dự án AI WORKSPACE.
- Ngoài: rủi ro của project do nền tảng sinh ra cho người dùng cuối; rủi ro kinh doanh/thị trường.
- Rủi ro bảo mật chi tiết (R1-R10 của `SECURITY.md`) được **tham chiếu**, không nhân đôi nội dung; sổ này gán ID cấp dự án (`R-xxx`) và liên kết.

## 3. Nội dung chi tiết

### 3.1 Phương pháp [Đề xuất]

**Thang điểm.** Likelihood (L) và Impact (I) đều 1-5. Điểm = L × I.

| Mức | L (khả năng xảy ra) | I (tác động) |
|---|---|---|
| 1 | Hiếm (< 10%) | Không đáng kể, xử lý trong ngày |
| 2 | Thấp (10-30%) | Trễ ≤ 1 tuần hoặc sửa nhỏ |
| 3 | Trung bình (30-50%) | Trễ 1-3 tuần, một tính năng chính bị cắt giảm |
| 4 | Cao (50-80%) | Trễ > 3 tuần, hoặc một AC nhóm không đạt |
| 5 | Gần như chắc chắn (> 80%) | Dự án không thể đạt mục tiêu chính hoặc có sự cố bảo mật/pháp lý nghiêm trọng |

Mức độ: 1-6 Thấp · 8-12 Trung bình · 15-25 Cao. Xác suất L chỉ là ước tính chủ quan ở thời điểm 2026-10-30 [Giả định], sẽ hiệu chỉnh sau mỗi phase.

**Trạng thái:** `Open` (chưa xử lý), `Mitigating` (đang giảm thiểu), `Accepted` (Owner chấp nhận bằng văn bản), `Closed` (không còn), `Occurred` (đã xảy ra, đang xử lý hậu quả).

**Vai trò chịu trách nhiệm** dùng 18 role seed: `pm, ba, architect, uiux, frontend, backend, db_infra, qa, security, reviewer, research, integration, performance, devsecops, docs, ux_research, test_automation, release`. "Owner" = chủ sở hữu sản phẩm (người dùng), chỉ dùng cho quyết định chấp nhận.

### 3.2 Xếp hạng theo điểm (L × I)

| Mức | Rủi ro (điểm) |
|---|---|
| Cao (15-25) | R-QUOTA-RATELIMIT (20), R-COST-RUNAWAY (20), R-TOS-QUOTA (16), R-WIN-SANDBOX (16), R-CRED-LEAK (15) |
| Trung bình (8-12) | R-SCOPE-SIZE (12), R-RAM-16GB (12), R-ENGINE-COMPLEXITY (12), R-CONTRACT-DRIFT (12), R-PROMPT-INJ (12), R-AGPL-CONTAM (10), R-CODEX-MISSING (9), R-DOCKER-DOWN (9), R-MERGE-CONFLICT (9), R-AGENT-ISOLATION (8) |
| Thấp (1-6) | R-ASSET-LICENSE (6), R-DEP-SCRIPTS (6), R-APPROVAL-LAG (6), R-NO-GIT (6), R-OFFICE-PERF (4) |

Lưu ý đọc điểm:
- R-AGPL-CONTAM (tác động 5) và R-AGENT-ISOLATION (tác động 4) có điểm không cao nhưng hậu quả nghiêm trọng nên được theo dõi như rủi ro trung bình trở lên.
- Rủi ro điểm ≥ 15 bắt buộc có kế hoạch ứng phó (contingency) bằng văn bản ở 3.4.
- Điểm là ước tính chủ quan [Giả định] tại 2026-10-30, hiệu chỉnh lại sau mỗi phase.

### 3.3 Sổ đăng ký rủi ro

Cột: ID · Rủi ro · L · I · Điểm · Giảm thiểu · Owner role · Trigger · Trạng thái. Chi tiết bổ sung ở 3.4.

| ID | Rủi ro | L | I | Điểm | Giảm thiểu | Owner role | Trigger (dấu hiệu) | Trạng thái |
|---|---|---|---|---|---|---|---|---|
| R-TOS-QUOTA | **Điều khoản dịch vụ / quota của provider** có thể không cho phép hoặc giới hạn nhiều phiên CLI song song dùng chung một tài khoản subscription (claude.ai `team`; ChatGPT cho Codex). Quota 5 giờ/tuần dùng chung cho mọi agent | 4 | 4 | 16 | Xác minh điều khoản bằng văn bản (EXT-7); mặc định concurrency 5 (Q-07/OWR-2: giới hạn của ứng dụng, không xác nhận subscription cho phép song song, chưa xác minh ToS); không tự tăng lên 10/18 khi chưa xác minh điều khoản provider (trước đây: tăng 5→10→18 khi có bằng chứng); scheduler tính quota theo credential; chế độ suy giảm (`RESOURCE_MANAGEMENT.md` §9); cân nhắc nhiều credential/API key (FR-098, OI-06) | pm + performance | Thông báo "usage limit", 429, email cảnh báo từ provider, tài khoản bị hạn chế | Open |
| R-QUOTA-RATELIMIT | Quota cạn hoặc rate-limit giữa chừng test 18 hoặc SDLC thật, mất bằng chứng và đứng việc | 5 | 4 | 20 | Chạy benchmark đầu cửa sổ quota; dừng ngay khi `RATE_LIMIT`; ghi thời điểm reset; T1-T4 theo từng bước; checkpoint để resume (FR-289); hàng đợi và backoff | performance | Lỗi `RATE_LIMITED`/`usage limit` trong adapter; `scheduler_event` THROTTLE tăng | Open |
| R-WIN-SANDBOX | **Khoảng trống sandbox trên Windows**: Claude Code native Windows chạy lệnh không có OS sandbox; file tool chạy quyền user; chính sách CLI là lớp chính sách, không phải ranh giới bảo mật (D-007, `SECURITY.md` R1, R4, R8) | 4 | 4 | 16 | Mô hình 2 tầng: hook chặn/viết lại lệnh, build/test/run trong Docker, Tier 2 container hóa toàn bộ cho project không tin cậy; post-hoc diff validation là cổng cuối; Owner ký R1; không chạy với quyền admin (admin guard); cân nhắc WSL2/Tier 2 làm mặc định cho project untrusted | security | Phát hiện ghi ngoài worktree; hook không chạy (`--bare` hoặc lỗi settings); audit thấy lệnh ngoài policy | Open — cần EXT-5 |
| R-AGPL-CONTAM | **Nhiễm AGPL-3.0**: sao chép mã, prompt, layout hoặc asset từ My Virtual Office vào sản phẩm sẽ buộc phát hành toàn bộ mã nguồn theo AGPL | 2 | 5 | 10 | D-012: chỉ tham khảo ý tưởng; cấm copy; asset chỉ CC0/MIT/OFL; `assets-manifest.json` + CI; `THIRD_PARTY_NOTICES.md`; license-checker chặn AGPL/GPL/SSPL ở dependency runtime; review giấy phép cho mọi PR chạm office/asset; agent viết Office dùng `PIXEL_AGENTS_ANALYSIS.md` (MIT) làm nguồn, không đọc mã MVO khi viết | reviewer + research | Diff chứa đoạn trùng với `../my-virtual-office`; dependency mới có giấy phép AGPL; asset không có dòng manifest | Open |
| R-CODEX-MISSING | **Codex CLI chưa cài** [Đã xác minh]; toàn bộ thiết kế Codex dựa trên tài liệu, `mxc` sandbox chưa kiểm chứng trên Windows build 26300 (`SECURITY.md` R10); AC-022 không thể kiểm | 3 | 3 | 9 | EXT-2 (Owner cài + login); adapter dựa fixture đã ghi từ tài liệu và gắn `UNVERIFIED`; không đánh dấu đạt AC-022 khi chưa chạy thật; Claude là provider chính của MVP; fallback `codex exec --json` | integration | Đến Phase 2 vẫn `codex: command not found`; test P-D bị `BLOCKED` | Open — cần EXT-2 |
| R-DOCKER-DOWN | **Docker daemon chưa chạy** lúc kiểm tra (Docker Desktop 29.8.1 đã cài); cgroup, ECI, rootless chưa xác minh; mọi task sandbox (AC-101, 102, 105, 106, 113) bị chặn | 3 | 3 | 9 | EXT-3; `pnpm doctor` báo rõ; test container báo `SKIPPED-NO-DOCKER`, không pass giả; AC-114 định nghĩa hành vi khi không có Docker (EXT-8); WSL2 backend đã bật theo `ENVIRONMENT_SETUP.md` | devsecops | `docker info` lỗi; CI job container skip; Docker Desktop không tự khởi động | Open — cần EXT-3 |
| R-ASSET-LICENSE | **Giấy phép asset** (sprite pixel-art, âm thanh, icon, font) không rõ hoặc không tương thích; asset từ repo tham khảo có giấy phép riêng khác với mã | 2 | 3 | 6 | Chỉ CC0/MIT/OFL; manifest bắt buộc (nguồn, tác giả, giấy phép, URL, ngày); CI chặn asset thiếu manifest; asset tạo mới do agent `uiux` thì ghi rõ nguồn gốc; không dùng asset của Pixel Agents/MVO trừ khi giấy phép xác nhận rõ | uiux + reviewer | Asset không có dòng manifest; giấy phép "không rõ"; khiếu nại | Open |
| R-SCOPE-SIZE | **Quy mô phạm vi**: 192 FR (167 MVP), 100 AC, 22 màn hình, 18 vai trò, 44 bảng, 3 provider, Office 2D, workflow engine tự xây — rủi ro không hoàn thành hoặc chất lượng thấp | 4 | 3 | 12 | Phase hoá 0-7; FR P2 (24) hoãn; kích hoạt 5-7 agent mặc định (`AGENT_ORGANIZATION.md`); thứ tự "vertical slice" sớm; Owner duyệt ưu tiên MoSCoW; cắt giảm theo thứ tự: OpenHands/Tauri → Workflow Builder đầy đủ → layout editor Office → màn hình phụ; mỗi phase có exit criteria và demo | pm | Trễ > 30% so với ước lượng phase; số AC `UNVERIFIED` tăng; backlog phát sinh | Open |
| R-COST-RUNAWAY | **Chi phí/quota vượt kiểm soát**: spawn tax ≈ 48k token/spawn (≈ 0,37 USD list) [Đã xác minh]; 18 agent liên tục ≈ 250-540 USD list/ngày 8 giờ [Giả định]; vòng lặp agent không dừng, retry bão | 5 | 4 | 20 | Budget cứng theo agent/project/ngày (FR-009, FR-185); `max_turns` và timeout; dừng khi vượt ngân sách; giảm spawn tax (config dir tối giản, P2-14); giới hạn retry/backoff + jitter; cảnh báo ở 50/80/100%; Emergency Stop; `UsageValue` hiển thị nhãn ước tính; Owner duyệt trước khi chạy benchmark lớn | performance + backend | Chi phí/giờ vượt ngưỡng; một task > N turn; retry count tăng; quota 5 giờ cạn sớm | Open |
| R-PROMPT-INJ | **Prompt injection**: agent đọc nội dung không tin cậy (file repo, web, issue, output tool) bị chỉ thị độc hại, dẫn tới hành động nguy hiểm hoặc rò dữ liệu (`SECURITY.md` R3) — không thể chặn triệt để | 3 | 4 | 12 | Chính sách chặn **hành động** (deny/ask list, ownership, sandbox, egress deny mặc định) thay vì tin vào mô hình; tách chỉ dẫn và dữ liệu không tin cậy trong prompt; approval cho lệnh/chi tiêu/gọi ngoài; không cấp credential cho agent; review diff; corpus test injection; Owner ký R3 | security | Hành động bị `policy_deny` bất thường tăng; agent cố truy cập URL/lệnh lạ; nội dung file chứa chỉ thị kiểu "ignore previous" | Open — cần EXT-5 |
| R-CRED-LEAK | Rò rỉ credential: token OAuth của CLI nằm trong file người dùng đọc được bởi tiến trình cùng user (kiểm soát bằng ACL) (`SECURITY.md` R2), transcript CLI trên host (R9), log/DB/payload FE chứa secret | 3 | 5 | 15 | D-008: không đọc file credential; secret khác ở OS keychain; redaction middleware (corpus 200); response DTO không chứa secret; không mount credential vào container; gitleaks + trufflehog; quét DB/log/payload (AC-029, 030, 113); Semgrep cấm đọc `.credentials.json`/`auth.json`; Owner ký R2 | security | gitleaks báo; payload FE khớp regex secret; file lạ trong repo | Open — cần EXT-5 |
| R-RAM-16GB | **RAM/CPU của máy tham chiếu** (i5-11400H, 16 GB, ~6,4 GB free lúc đo): 18 process nặng cần 9-16 GB; khởi động đồng loạt bão hòa CPU [Đã xác minh số đo] | 4 | 3 | 12 | Host guard `min_free_ram_mb`; stagger spawn ≤ 3/2 s; giới hạn build đồng thời (đề xuất 3); mặc định 5; nâng RAM ≥ 32 GB hoặc chia máy; đóng IDE phụ khi N > 10; ghi cấu hình máy vào mọi báo cáo | performance | RAM free < ngưỡng; swap tăng; spawn chậm > 10 s | Open |
| R-ENGINE-COMPLEXITY | **Tự xây workflow engine** ("Conductor", D-003): độ phức tạp trạng thái bền, DAG, semaphore, recovery; nguy cơ lỗi tinh vi (double-spawn, deadlock, mất checkpoint) | 3 | 4 | 12 | Bám thiết kế `AGENT_SCHEDULER.md`/`RESOURCE_MANAGEMENT.md`; property test state machine; chaos test sớm (P3-13); semaphore rebuild từ Postgres; idempotency key (FR-291); giữ phạm vi node ở 8 loại; đánh giá lại phương án BullMQ Flows/Temporal nếu P3 trễ > 50% | architect + backend | Test chaos lặp lại thất bại; `COUNT(RUNNING)` ≠ semaphore; task treo | Open |
| R-CONTRACT-DRIFT | **Lệch contract/tài liệu**: nhiều agent song song, các bộ trạng thái khác nhau (FR-024 12 trạng thái vs SCREEN_SPEC vs `StatusBadge` 8 tone; `task_status` HOA/thường), Phaser vs PixiJS | 4 | 3 | 12 | Contract freeze P1-04; `toTone()` một nguồn; quy trình Contract change; contract test/snapshot; bảng mâu thuẫn `IMPLEMENTATION_ROADMAP.md` §3.14 | architect | PR sửa contract không có mục "Contract change"; FE/BE test contract fail | Open |
| R-AGENT-ISOLATION | **Cách ly agent chỉ ở mức logical** (cùng user OS): agent A có thể chạm worktree B qua đường dẫn tuyệt đối nếu lọt policy (`SECURITY.md` R4) | 2 | 4 | 8 | Manifest ownership + hook + diff validation; Docker cho lệnh chạy; deny đường dẫn ngoài workspace; test AC-107; Tier 2 cho project nhạy cảm | security + backend | Diff có file ngoài `owns`; audit thấy truy cập chéo worktree | Open |
| R-DEP-SCRIPTS | **Cài dependency chạy mã bên thứ ba** (postinstall) trong container có network registry (`SECURITY.md` R7); chuỗi cung ứng cho chính repo nền tảng | 2 | 3 | 6 | `--ignore-scripts` mặc định; lockfile-lint; `npm audit`, osv-scanner, dependency-review; pin version; `deps-request` thay vì agent tự thêm | devsecops | Cảnh báo audit high/critical; install script mới xuất hiện | Open |
| R-MERGE-CONFLICT | **Xung đột merge Stage 5** khi FE/BE/DB song song sửa file chồng lấn (OI-FR-08); xung đột hot files (lockfile, migration, contract) khi phát triển nền tảng | 3 | 3 | 9 | Manifest ownership không chồng lấn trong batch; thứ tự merge theo dependency; hot files một chủ; agent không tự giải conflict ngoài vùng `owns` | backend + architect | Rebase conflict thường xuyên; PR bị chặn vì conflict | Open |
| R-OFFICE-PERF | Virtual Office giật khi 18 nhân vật (AC-087, 095 ≥ 30 FPS) trên laptop tích hợp | 2 | 2 | 4 | Tối ưu sprite atlas, giảm FPS khi tab ẩn, reduced-motion, tách logic FSM; đo trên máy tham chiếu | frontend + performance | FPS < 30 trong sampler | Open |
| R-APPROVAL-LAG | **Owner chậm phê duyệt** quyết định (16 quyết định Chờ duyệt, nhiều mục cần ký) làm chặn Phase 1 và 4 | 2 | 3 | 6 | Gom quyết định vào bảng duyệt một lần; cho phép bắt đầu với D-001, D-002 duyệt trước; ghi nhận giả định; cổng EXT rõ ràng | pm | Quá 1 tuần chưa có phản hồi EXT-1; task `BLOCKED(EXT-x)` | Open |
| R-NO-GIT | Thư mục chưa là git repo, chưa có remote [Đã xác minh]; không dùng được worktree/CI | 3 | 2 | 6 | P1-01, EXT-6 làm đầu tiên | devsecops | Phase 1 bắt đầu mà chưa có repo | Open |

### 3.4 Chi tiết các rủi ro trọng yếu

**R-TOS-QUOTA / R-QUOTA-RATELIMIT.**
- Bằng chứng hiện có [Đã xác minh]: tài khoản claude.ai `subscriptionType: team`, mọi agent dùng chung một credential nên một quota chung; thử 3 process đồng thời không thấy rate-limit; chưa thử > 3 (`MULTI_AGENT_SCALABILITY.md` §4, §6).
- [Giả định] Cửa sổ 5 giờ + giới hạn tuần; không có API đọc quota còn lại; có thể có giới hạn concurrent request chỉ lộ ra ở N lớn.
- Hậu quả nếu xảy ra: giảm N hiệu dụng, thời gian chạy dài, hoặc cần API key/nhiều credential (FR-098 chưa chốt, OI-06). Tác động lên AC-064 (chỉ kiểm slot) và exit criterion 18-concurrency.
- Kế hoạch ứng phó: kịch bản suy giảm (`RESOURCE_MANAGEMENT.md` §9) → giảm N tự động; hàng đợi; thông báo Owner; không tự đổi credential.

**R-WIN-SANDBOX.**
- Quyết định D-007 dung hòa mâu thuẫn "không chạy code AI trên host" bằng Tier 1 + Docker cho lệnh build/test/run; nhưng CLI vẫn chạy trên host. Đây là rủi ro được thừa nhận, không phải rủi ro đã loại bỏ.
- Điều kiện chấp nhận: Owner ký R1, R2, R3, R5, R6 (`SECURITY.md` §10). Nếu Owner không chấp nhận, phương án thay thế: Tier 2 (container hóa toàn bộ runtime, D-006 OpenHands) làm mặc định — đổi kiến trúc, ảnh hưởng Phase 4-5 và AC-101.
- Rủi ro phụ thuộc Docker (R-DOCKER-DOWN): nếu không có Docker, toàn bộ phòng thủ lệnh thực thi yếu đi (AC-114).

**R-AGPL-CONTAM.**
- Xác suất thấp nhưng hậu quả nghiêm trọng. My Virtual Office là AGPL-3.0; kể cả trích prompt, layout, asset cũng rủi ro. Nguồn mã được phép: ChatDev (Apache-2.0), Pixel Agents và OpenHands (MIT) có attribution (D-012).
- Kiểm soát chủ động: agent làm Office được chỉ dẫn không đọc mã MVO; review giấy phép trước merge; CI chặn license không hợp lệ; ghi attribution ở `THIRD_PARTY_NOTICES.md`.

**R-CODEX-MISSING.**
- [D-005/Q-10, OWR-2 2026-10-30: Codex hoãn khỏi MVP; không cài Codex CLI trong phạm vi hiện tại; rủi ro này giảm thành rủi ro hoãn, không chặn MVP.]
- Hiện: Codex CLI chưa cài; đã đăng nhập ChatGPT qua extension VS Code, không đồng nghĩa CLI sẵn sàng [Đã xác minh theo `DECISION_LOG.md` D-005].
- Hệ quả: AC-022, P-D test, thiết kế sandbox Codex (`mxc`) `UNVERIFIED`. Không làm chặn MVP nếu Owner chấp nhận Claude-only làm bằng chứng chính.

**R-DOCKER-DOWN.**
- Docker 29.8.1 đã cài nhưng daemon không chạy khi kiểm tra [Đã xác minh]. Cho tới khi chạy được, không xác minh được cgroup, ECI, rootless, egress proxy, giới hạn tài nguyên. CI Linux có Docker nên vẫn kiểm được cấu hình container ở CI, nhưng hành vi trên Windows/Docker Desktop của máy Owner thì chưa.

**R-COST-RUNAWAY.**
- Số liệu: spawn tax 18.287 cache write + 29.992 cache read token mỗi lần spawn (≈ 0,37 USD giá niêm yết) [Đã xác minh]; chi phí làm việc giả định 1-3 USD/giờ/agent [Giả định]. Với subscription, tương đương quota chứ không phải hóa đơn, nhưng quota cạn là hậu quả thực.
- Kiểm soát: budget cứng + cảnh báo + Emergency Stop + giảm spawn tax là KPI trước khi scale.

**R-SCOPE-SIZE.**
- Chiến lược cắt giảm nếu trễ (theo thứ tự hy sinh, cần Owner duyệt): (1) OpenHands, generic CLI, Tauri (P2), (2) layout editor Office và Workflow Builder dạng canvas (FR-112 là Should/P2), (3) màn hình Cost/Activity chi tiết, (4) giảm số role kích hoạt mặc định. Không cắt: event thật, bằng chứng test, bảo mật cốt lõi, 18-concurrency proof (vì là yêu cầu của Owner).

**R-PROMPT-INJ.**
- Chấp nhận rằng chỉ giảm thiểu. Kiểm thử đo khả năng chặn **hành động** thay vì khả năng mô hình "không bị lừa". Ghi nhận trong audit để điều tra.

### 3.5 Ánh xạ R1-R10 của `SECURITY.md` sang sổ này

| `SECURITY.md` | Nội dung (rút gọn) | ID dự án |
|---|---|---|
| R1 | Tier 1 không có OS-level sandbox cho CLI | R-WIN-SANDBOX |
| R2 | Credential OAuth file đọc được bởi tiến trình cùng user (ACL) do CLI quản lý | R-CRED-LEAK |
| R3 | Prompt injection không chặn triệt để | R-PROMPT-INJ |
| R4 | Cách ly giữa agent mức logical | R-AGENT-ISOLATION |
| R5 | Docker Desktop daemon = root của VM, mount được toàn ổ đĩa | R-WIN-SANDBOX (phụ), R-DOCKER-DOWN |
| R6 | Egress container dựa proxy allowlist, không phải firewall kernel | R-WIN-SANDBOX (phụ) |
| R7 | Cài dependency chạy mã bên thứ ba | R-DEP-SCRIPTS |
| R8 | Hook/`--settings` là policy runtime của CLI; `--bare` làm mất hook | R-WIN-SANDBOX (phụ) |
| R9 | Transcript/history CLI trên host ngoài redaction | R-CRED-LEAK (phụ) |
| R10 | Codex chưa cài, `mxc` chưa kiểm chứng | R-CODEX-MISSING |

Top 5 cần Owner ký: R1, R2, R3, R5, R6 (`SECURITY.md` §10) → cổng EXT-5 của roadmap.

### 3.6 Rủi ro theo phase và hành động cần làm trước

| Phase | Rủi ro chủ đạo | Hành động trước khi vào phase |
|---|---|---|
| 0 → 1 | R-APPROVAL-LAG, R-NO-GIT, R-DOCKER-DOWN, R-CONTRACT-DRIFT | EXT-1, EXT-3, EXT-6; chốt bảng trạng thái |
| 1 → 2 | R-TOS-QUOTA, R-COST-RUNAWAY, R-CODEX-MISSING | Xác nhận gói/điều khoản; EXT-2, EXT-4 |
| 2 → 3 | R-QUOTA-RATELIMIT, R-RAM-16GB, R-ENGINE-COMPLEXITY | Có số đo T1-T3; host guard thiết kế xong |
| 3 → 4 | R-WIN-SANDBOX, R-PROMPT-INJ, R-DOCKER-DOWN | EXT-3 xong; kế hoạch ký R1-R6 |
| 4 → 5 | R-MERGE-CONFLICT, R-AGENT-ISOLATION, R-CRED-LEAK | EXT-5 ký; security test xanh |
| 5 → 6 | R-SCOPE-SIZE, R-ASSET-LICENSE, R-AGPL-CONTAM | Manifest asset; review giấy phép |
| 6 → 7 | R-OFFICE-PERF, R-COST-RUNAWAY, R-QUOTA-RATELIMIT | Ngân sách quota T3-T6; EXT-7 |

### 3.7 Quy trình quản lý rủi ro [Đề xuất]

1. `pm` rà soát sổ này **cuối mỗi phase** và khi có sự cố; cập nhật L, I, trạng thái.
2. Mỗi PR có thể gắn nhãn `risk:<ID>` khi thay đổi giảm/tăng rủi ro đó.
3. Trigger kích hoạt thì tạo issue ưu tiên cao với owner role xử lý trong ≤ 1 ngày làm việc; rủi ro điểm ≥ 15 phải có kế hoạch ứng phó (contingency) bằng văn bản.
4. `Accepted` chỉ do Owner đặt, kèm ngày và điều kiện (ví dụ R1-R6).
5. `KNOWN_ISSUES.md` ghi sự cố đã xảy ra (`Occurred`) và liên kết ID rủi ro.
6. Mọi thay đổi L, I ghi vào `IMPLEMENTATION_LOG.md` một dòng.

### 3.8 Rủi ro mới có thể phát sinh (theo dõi, chưa đăng ký) [Giả định]

- Thay đổi chính sách hoặc CLI của provider làm vỡ adapter (CLI 2.1.312 hôm nay; version khác có thể thay đổi stream-json): giảm bằng pin version, golden fixture có version, nightly P-C.
- Phụ thuộc vào `@anthropic-ai/claude-agent-sdk` (D-004) thay đổi API.
- Windows path length/Job Object/junction khác biệt giữa các máy.
- Quá tải review khi nhiều agent: hàng đợi `reviewer` (cần ≥ 2 instance khi ≥ 10 dev agent, `MULTI_AGENT_SCALABILITY.md` §8).
- Phụ thuộc một Owner duy nhất (bus factor = 1).

### 3.9 Chỉ báo sớm cần đo trong suốt dự án [Đề xuất]

| Chỉ báo | Rủi ro liên quan | Ngưỡng cảnh báo | Nguồn đo |
|---|---|---|---|
| Số lần `RATE_LIMITED` mỗi giờ | R-TOS-QUOTA, R-QUOTA-RATELIMIT | ≥ 1 trong cửa sổ test 18 | log adapter, `scheduler_event` |
| Token/spawn trung bình | R-COST-RUNAWAY | > 20k sau P2-14 | bảng usage |
| RAM free tại thời điểm spawn | R-RAM-16GB | < ngưỡng host guard | process monitor |
| Số `policy_deny` bất thường trên mỗi task | R-PROMPT-INJ, R-WIN-SANDBOX | tăng > 3× so với trung bình | audit log |
| Số AC `UNVERIFIED` còn lại | R-CODEX-MISSING, R-DOCKER-DOWN, R-SCOPE-SIZE | không giảm sau 2 phase | bảng truy vết AC |
| Số PR vi phạm lint F-01…F-20 | R-CONTRACT-DRIFT | > 0 trên `main` | CI |

## 4. Quyết định & lý do

| # | Quyết định | Lý do | Nhãn |
|---|---|---|---|
| PR-1 | Thang 1-5 × 1-5, ngưỡng 15 cần contingency | Đơn giản, dễ rà soát | [Đề xuất] |
| PR-2 | Hợp nhất R1-R10 bảo mật vào sổ dự án bằng ánh xạ, không nhân đôi nội dung | Tránh lệch hai nguồn | [Đề xuất] |
| PR-3 | Rủi ro `Accepted` chỉ do Owner đặt | Rủi ro bảo mật/pháp lý cần chủ sở hữu chịu trách nhiệm | [Cần người dùng phê duyệt] |
| PR-4 | Thứ tự hy sinh phạm vi định sẵn | Quyết định nhanh khi trễ tiến độ | [Cần người dùng phê duyệt] |
| PR-5 | Chấp nhận R-PROMPT-INJ chỉ giảm thiểu | Không có giải pháp triệt để (`SECURITY.md` R3) | [Đã xác minh] |

## 5. Dependencies

- `SECURITY.md` §10 (R1-R10), `LICENSE_AND_REUSE.md` (giấy phép), `MULTI_AGENT_SCALABILITY.md` (số đo), `RESOURCE_MANAGEMENT.md` (chế độ suy giảm), `ENVIRONMENT_SETUP.md` (trạng thái công cụ), `IMPLEMENTATION_ROADMAP.md` (cổng EXT, phase).
- Owner: ký chấp nhận (EXT-5), xác nhận điều khoản/gói (EXT-7), cài Codex (EXT-2), chạy Docker (EXT-3).

## 6. Rủi ro (của chính sổ này)

| Rủi ro | Giảm thiểu |
|---|---|
| Sổ không được cập nhật, thành tài liệu chết | `pm` rà soát cuối mỗi phase, tiêu chí exit có mục "cập nhật sổ rủi ro" |
| Điểm L, I chủ quan | Hiệu chỉnh bằng số đo thật sau mỗi phase; ghi nguồn |
| Quá nhiều rủi ro làm loãng ưu tiên | Chỉ rủi ro điểm ≥ 12 vào báo cáo hằng tuần |

## 7. Vấn đề chưa giải quyết

1. [Cần người dùng phê duyệt] Owner chấp nhận hay không R1, R2, R3, R5, R6; nếu không, đổi kiến trúc (Tier 2 mặc định).
2. Điều khoản chính thức của Anthropic/OpenAI về nhiều phiên CLI song song trên một subscription chưa được xác minh bằng tài liệu trong Phase 0 [Giả định]; cần Owner hoặc `research` kiểm tra điều khoản hiện hành.
3. Gói subscription chính xác (team: loại seat, extra usage) chưa biết (EXT-7).
4. Chưa có số đo RAM/CPU cho task thật (build/test) nên L của R-RAM-16GB là ước lượng.
5. Chưa biết điều kiện pháp lý về attribution/lưu trữ nếu phát hành mã nguồn (giấy phép của dự án, MIT/Apache/proprietary) — ảnh hưởng R-AGPL-CONTAM và `THIRD_PARTY_NOTICES.md`.
6. Có cần bảo hiểm/ràng buộc ngân sách cứng ở cấp tài khoản provider (hạn mức chi tiêu) để chặn R-COST-RUNAWAY hay không.
7. Chưa có kế hoạch sao lưu dữ liệu workspace cho MVP (FR-292 là P2); rủi ro mất dữ liệu chưa đăng ký.

## 8. Acceptance Criteria (cho sổ rủi ro)

- [ ] Mỗi rủi ro có ID, mô tả, L, I, điểm, giảm thiểu, owner role (một trong 18 role), trigger, trạng thái.
- [ ] Đã đăng ký đủ các rủi ro Owner yêu cầu: provider ToS/quota, khoảng trống sandbox Windows, nhiễm AGPL, Codex chưa cài, Docker daemon chưa chạy, giấy phép asset, quy mô phạm vi, chi phí vượt kiểm soát, prompt injection.
- [ ] Mọi R1-R10 của `SECURITY.md` có ánh xạ (3.5).
- [ ] Rủi ro điểm ≥ 15 (R-TOS-QUOTA, R-QUOTA-RATELIMIT, R-WIN-SANDBOX, R-COST-RUNAWAY, R-CRED-LEAK) có kế hoạch ứng phó trong 3.4.
- [ ] Sổ được rà soát và cập nhật cuối mỗi phase (kiểm bằng `IMPLEMENTATION_LOG.md`).
- [ ] Rủi ro `Accepted` có chữ ký/ngày của Owner.
