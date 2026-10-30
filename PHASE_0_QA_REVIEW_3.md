# PHASE_0_QA_REVIEW_3 — QA độc lập lần ba, Phase 0 AI WORKSPACE

Ngày: 2026-10-30. Reviewer: QA & Technical Reviewer độc lập (không viết/sửa tài liệu nào được rà). Phạm vi do Owner cho phép (OWR-3 mục E): tài liệu, traceability, AC và các thay đổi đã cập nhật. Chỉ chạy lệnh đọc (grep, wc, find, node script đọc file, `git status/diff/log`). Không cài, không bật Docker, không chạy prompt model, không đọc `~/.claude/.credentials.json` hay `~/.codex/auth.json`. File duy nhất tôi tạo: file này.

## 0. Verdict điều hành

**Verdict: PASS WITH FIXES.** Không có P0. Đếm: **P0 = 0, P1 = 1, P2 = 9, P3 = 8.**

Điểm mạnh có bằng chứng: traceability 199/199 FR, 0 FR Must thiếu AC, 0 AC được trích mà chưa định nghĩa, 0 va chạm ID; 15 FR AC_PROPOSED_OWNER_DECIDED và FR-078 WONT_MVP khớp OWR-3 mục C và A.3-A.6; 16 event gốc chép nguyên văn; không VERIFIED nào; Docker/Conductor đều ghi chưa chạy; không có secret; không có mã ứng dụng; 5 repo tham chiếu sạch.

Điểm yếu chính: một mâu thuẫn thiết kế credential/keychain với FR-088, FR-098 và AC-640 (P1); các tài liệu còn tên event UPPER_SNAKE hoặc phân đoạn có `_` không có chỉ dẫn; roadmap còn mục tiêu nâng 10/18; các file trạng thái cấp gốc chưa cập nhật theo OWR-3.

| # | Hạng mục | Verdict |
|---|---|---|
| 1 | Tài liệu bắt buộc | PASS |
| 2 | FR coverage và AC integrity | PASS (3 điểm P3) |
| 3 | Quyết định OWR-3 so với tài liệu | PARTIAL (xem §3) |
| 4 | Trung thực và từ vựng trạng thái | PASS WITH FIXES |
| 5 | Xung đột mới X-13, X-14; concurrency | PARTIAL |
| 6 | Bảo mật, credential, AGPL, mock | PARTIAL (P1 credential) |
| 7 | Git | PASS (có ghi chú) |
| 8 | Tuân thủ quy trình | PASS |

## 1. Tài liệu bắt buộc: PASS

Lệnh: vòng `for f in $(find docs -name '*.md')` với `wc -l`, in dòng nếu < 60; `find docs -empty`; `wc -l` các file gốc và PHASE_0_*.
- Kết quả: 110 file `docs/**/*.md`; `find docs -empty` rỗng; chỉ 1 file < 60 dòng: `docs/adr/ADR-0021-dynamic-skill-overlay.md` = 59 dòng (không sửa trong đợt này; chấp nhận được, ghi P3).
- Deliverable: FINAL_REPORT 99, QA_REVIEW_2 147, ACCEPTANCE_CRITERIA_TRACEABILITY 630, OWNER_DECISION_REGISTER 631, TECHNICAL_VALIDATION 317, SECURITY_SIGNOFF 249, REMAINING_RISKS 141 dòng. `docs/poc/` 3 file (64, 163, 185). `docs/owner-responses/` OWR-2 (87), OWR-3 (96). ADR-0001..0021: `ls docs/adr/ADR-00*.md | wc -l` = 21. Thêm `DOC_SYNC_LOG_OWR3_A.md` (85) và `_B.md` (67).
- Tracking gốc tồn tại: AGENTS 66, AGENT_WORK_LOG 65, DECISION_LOG 65, IMPLEMENTATION_LOG 27, KNOWN_ISSUES 25, NEXT_SESSION 46, PROJECT_STATUS 29, TASK_BACKLOG 60, WORK_PLAN 54 (nhưng nội dung chưa cập nhật OWR-3, xem F3-07).

## 2. FR coverage và AC integrity: PASS

Công cụ: node script đọc file (scratchpad, không nằm trong repo).

| Kiểm tra | Kết quả |
|---|---|
| FR có hàng định nghĩa trong FUNCTIONAL_REQUIREMENTS | 199 (mục 1 đến 8) + 6 FR đề xuất FR-293..298 ở mục 3.13, được ghi rõ "không tính" ở dòng 297 (mục 4). Script thô báo 205 vì gồm 6 FR đề xuất; 8 "trùng" (FR-077/078/113/122/181/182/183/231) chỉ là các hàng tóm tắt ở mục 10 (dòng 372-388), không phải định nghĩa trùng |
| Bảng truy vết (PHASE_0_ACCEPTANCE_CRITERIA_TRACEABILITY.md) | 199 hàng; missing (ngoài 6 FR đề xuất) = 0; extra = 0; duplicate = 0 |
| Đếm trạng thái | AC_DEFINED 109 + AC_ADDED_PHASE0 74 + AC_PROPOSED_OWNER_DECIDED 15 + WONT_MVP 1 = **199**, khớp mục 1 của file |
| FR Must không có AC | 0 (chỉ FR-078, mức Could, không AC, đúng WONT_MVP) |
| AC được trích nhưng chưa định nghĩa | 0 (212 mã trích, 220 mã định nghĩa, 0 định nghĩa trùng) |
| Va chạm ID | dải 001-199: 104; 200: 19; 300: 20; 400: 33; 600: 44 (AC-601..614, 621..623, 631..657); không trùng |
| 15 FR AC_PROPOSED_OWNER_DECIDED | FR-006, 009, 038, 088, 098, 122, 153, 181, 182, 183, 186, 200, 283, 284, 292 = đúng 11 FR mục C (không tính FR-078) cộng 4 FR A.3-A.6 |
| WONT_MVP | chỉ FR-078, hàng "(không có, WON'T DO IN MVP)", giữ trong truy vết, đúng OWR-3 C |
| PROPOSED thành APPROVED | không có: grep `APPROVED` trong `docs/product` và file truy vết, loại phủ định, chỉ còn tên event `HUMAN_APPROVED` và `APPROVED_CONDITIONAL`; mọi AC mới ghi PROPOSED |

Lấy mẫu 28 AC mới hoặc cập nhật (AC-601..614, AC-631..657), đối chiếu OWR-3:
- Công thức FR-186: AC-646 (6/(6+2) = 75%), AC-647 (running, blocked, cancelled không tính), AC-648 (mẫu số 0 hiển thị N/A): khớp, đo được.
- FR-009: AC-634 (token theo dõi riêng), AC-635 (80%, đúng một cảnh báo, ranh giới 79 -> 80 -> 81), AC-636 (dừng cấp task và gọi model mới, chờ Owner): khớp.
- FR-181: AC-605 (approval chờ lâu không là incident), AC-607 (grouping 15 phút, `occurrence_count`, ranh giới 14:59/15:01): khớp.
- FR-182: AC-608..611 (baseline, cancelled loại, equal-weight, 100 x 4/8 = 50%); FR-183: AC-612 (không KPI tổng hợp); FR-122: AC-601..604; FR-200: AC-649..651 (`remind_at` 24 giờ, `expires_at` 15 phút, trường riêng); FR-153: AC-644..645; FR-006: AC-631..633 (ràng buộc DB, 50 cấp, chặn vòng); FR-283: AC-652..653; FR-284: AC-654..655; FR-292: AC-656..657.
- Đánh giá testability: 26/28 đủ Given/When/Then và có con số hoặc điều kiện đo được. Hai AC có điều chỉnh của BA, đã tự gắn nhãn: AC-607 (cửa sổ đo từ `first_seen`, "diễn giải của BA") và AC-651 (hết hạn không ngầm là duyệt, "BA diễn giải"). AC-657 chưa chạy trong Phase 0 (ghi rõ). Điểm nhỏ: AC-636 viết "chạm hoặc vượt 100%" trong khi Owner viết "vượt 100%" (P3, F3-14).

## 3. Quyết định OWR-3 so với tài liệu

| Mục | Kết quả | Bằng chứng |
|---|---|---|
| A.1 16 event nguyên văn, không thêm/xóa/đổi tên | PASS | `EVENT_SYSTEM.md:17-45` bảng §0: 16 dòng, đúng thứ tự và tên của phụ lục OWR-3 |
| A.1 ánh xạ đánh dấu pending P-16 | PASS | `EVENT_SYSTEM.md:13` và `:19` "đề xuất cơ học, PENDING_OWNER_CLARIFICATION (P-16)"; Register 13.3 |
| A.2/D concurrency 5 | PASS có điểm trừ | Nhiều nơi ghi "5 là giới hạn scheduler nội bộ, không phải provider bảo đảm" (ADR-0010 §14, PRD:151, RESOURCE_MANAGEMENT:48,190). Điểm trừ: xem §5 và F3-04 |
| A.3-A.6 | PASS | FR-122/181/182/183 dòng 142, 181-183 và AC-601..614 |
| A.7 tile 16 và display scale 2 | PASS có điểm trừ | `DATABASE_DESIGN.md:1035-1036` `tile_size` 16, `display_scale` 2 (đề xuất cột); `SCREEN_SPECIFICATION.md:142`, `:126`. Còn văn bản lỗi thời có banner phủ: `ADR-0011:68,117` ("Chưa quyết định 16 hay 32"; mục §14 dòng 119 ghi đè) và `CONFLICTS_TO_RESOLVE.md:509` (F3-13) |
| A.8 Codex hoãn, giữ AgentAdapter | PARTIAL | FR-077:94, PRD:88, AGENT_ARCHITECTURE:9. Không nhất quán về AC-022: `FUNCTIONAL_REQUIREMENTS.md:94` ghi "Owner-acknowledged", còn `ACCEPTANCE_CRITERIA.md:202` vẫn "PENDING_OWNER_CLARIFICATION" (F3-08). Chính OWR-3 A.8 không nêu AC-022 |
| A.9 asset MIT | PASS | `ADR-0012:125`, `LICENSE_AND_REUSE.md:120`, `REUSE_DECISIONS.md:6`: chỉ dùng khi xác minh nguồn gốc, license, NOTICE; CC0 mặc định. `VIRTUAL_OFFICE.md:269` vẫn viết "chỉ asset CC0", chặt hơn Owner, không mâu thuẫn (P3) |
| A.10 sandbox | PARTIAL | Ghi nhận ở ADR-0007:121-125, SECURITY §0.6, SANDBOX_DESIGN:7; xem §5 |
| A.11 Phase 2 mock hoặc sandbox | PASS | `IMPLEMENTATION_ROADMAP.md:174,573-574,589` |
| A.12 `remind_at` và `expires_at` | PASS | FR-106, FR-200, AC-649..651; grep: `remind_at` có trong 20 file |
| A.13 L0-L5, không `full` | PARTIAL | `DATABASE_DESIGN.md:170,205` enum không `full`; `PERMISSION_MODEL.md:36,46,327,359` vẫn mô tả enum có `full` ("đề xuất bỏ khỏi enum hoặc CHECK"), có banner dòng 50-52 ghi đè (F3-09). P-17 (ý nghĩa L5) mở, được ghi nhận |
| A.14 9 UI node types | PASS có điểm trừ | FR-113:133, AC-306, WORKFLOW_ENGINE:10. Còn nhắc "11 engine kind": `FUNCTIONAL_REQUIREMENTS.md:368`, `AC_ADDENDUM_PART2.md:90`, `ADR-0003:117` (lịch sử), và `WORKFLOW_ENGINE.md:56` vẫn "8 loại" (P3, F3-15) |
| A.15 WAITING_APPROVAL | PASS | `MULTI_AGENT_ORCHESTRATION.md:69`, `RESOURCE_MANAGEMENT.md:55`, `ADR-0010:§14`, FR-231 ghi chú dòng 223 |
| A.16 dotted-only | PARTIAL | `EVENT_SYSTEM.md:9,208` chốt; `§5.7` đề xuất tên dotted cho toàn bộ. Còn sót: xem F3-02, F3-03 |
| B X-1 locale tiếng Việt | PASS | `UI_UX_GUIDELINES.md:102`, FRONTEND_ARCHITECTURE:164, AC-654. `ADR-0015:1` tiêu đề còn "mặc định tiếng Anh" nhưng có mục đính chính dòng 113-121 (P3) |
| B X-3 bỏ `full` | PARTIAL | xem A.13 |
| B X-4 dotted | PARTIAL | xem A.16 |
| B X-7 timeout 540 s | PASS có điểm trừ | `PROVIDER_INTEGRATION.md:416`, `PERMISSION_MODEL.md:317`, ADR-0003:117. `CLAUDE_CODE_INTEGRATION.md:516` ví dụ mã `timeoutMs: 30 * 60_000` (research, P3) |
| B X-8 audit 365 ngày | PASS | `SECURITY.md:523`, `DATABASE_DESIGN.md:1510`. Còn 4 chỗ `audit_events` (xem bảng tồn đọng) |
| D provider terms | PASS | ADR-0010 §14, IMPLEMENTATION_ROADMAP:591; ghi rõ là lời Owner, chưa kiểm chứng độc lập |
| MVP nhóm B | PARTIAL | xem A.8 |

## 4. Trung thực và từ vựng trạng thái: PASS WITH FIXES

- VERIFIED: grep `\bVERIFIED\b` toàn repo, loại các câu phủ định: chỉ còn nhãn kế hoạch kiểm thử (`TESTING.md:85-86` "VERIFIED-MOCK", `IMPLEMENTATION_ROADMAP.md:191,202` "VERIFIED-REAL" cho Phase sau), và các câu "không có mục nào VERIFIED". Không có tuyên bố VERIFIED nào đã đạt. PASS.
- Phase 0 hoàn tất: grep không thấy "Phase 0 Completed". Ngoại lệ gần nhất: `WORK_PLAN.md:3` "Phase 0 nội dung hoàn tất, chờ chủ sở hữu duyệt" (chưa cập nhật từ trước OWR-3; không phải "Completed" nhưng dễ bị đọc sai, F3-07). Phase 1 bắt đầu: không có (`ADR-0001:102` cấm).
- D-003, D-007, D-011, D-019, D-021: Register dòng 31, 35, 39, 47, 49 đều APPROVED_CONDITIONAL; `adr/README.md` bảng khớp; mục OWR-3 của README giữ nguyên các điều kiện.
- Mức bằng chứng ADR: README bảng: Verified 0; ADR-0002 "Partial (PGlite, chưa PG16)", ADR-0021 "Partial (script và CLI, chưa SDK)"; ADR-0003, 0007, 0011 Unverified. Không bị thổi phồng. **Lỗi:** `adr/README.md:16` và `:61` vẫn ghi "chưa có PoC report (không giả định `docs/poc/` tồn tại)" trong khi `docs/poc/` có 3 báo cáo thật (F3-06).
- PoC khớp Register: POC_DRIZZLE.md:23 "9 PASS, 1 FAIL, 1 BLOCKED" và dòng 3, 57 "PGlite, báo PostgreSQL 18.6, không phải 16, NOT RUN cho PG16"; POC_DYNAMIC_SKILL.md:15,131 "SK-04 SIMULATED"; POC_DOCKER_SANDBOX.md:3 "BLOCKED / NOT AUTHORIZED"; ADR-0003:§14 và README "PoC Conductor NOT RUN". Khớp.

## 5. Xung đột mới X-13, X-14 và concurrency

X-13 (A.10 so với Tier 1 D-007):
- Register mục 13.3 (dòng 565-583): ghi đủ 4 câu hỏi (i) Edit/Write host trong worktree, (ii) tiến trình runtime SDK trên host, (iii) runtime sandbox khi OpenHands hoãn, (iv) Docker thành điều kiện cứng trong khi chưa được phép. D-007 giữ APPROVED_CONDITIONAL (dòng 35, 482, 453). Nhất quán với ADR-0007:121-125, SECURITY.md:16,28-45, SANDBOX_DESIGN:7,205,268, POC_DOCKER_SANDBOX.md:7, REMAINING_RISKS và SIGNOFF (mỗi file có 1 lần nhắc X-13).
- Thiết kế Tier 1 gốc không bị viết lại như đã quyết: ADR-0007:125 "Đề xuất gốc không được viết lại"; SECURITY.md:39 "giữ nguyên làm lịch sử".
- Điểm trừ (P2, F3-05): `SECURITY.md:32-37` đưa "Quy tắc thiết kế mới (viết lại theo A.10)" 1 đến 5 (tắt shell tích hợp trên host, mọi thực thi chỉ qua sandbox execution tool) và tuyên bố các mục §2.2..§2.4, §3, §5, R1, R4 "bị chặn bởi quy tắc 1 đến 4 ... cho đến khi Owner chốt", đồng thời nói "Không tự quyết định". Quy tắc 1 phụ thuộc câu hỏi (i) và (ii) chưa được Owner trả lời; `PROVIDER_INTEGRATION.md:204` đã bake "Không bật shell tích hợp". Đây là diễn giải chặt của agent đặt lên trên D-007 đã duyệt có điều kiện. Không "âm thầm" (có nhãn), nhưng nên ghi rõ là "giả định an toàn tạm thời".

X-14 (thiếu bảng ánh xạ): Register 13.3 (dòng 585-588) + P-16; CONFLICTS (CF-064 tạm); adr/README:... ADR-0014 bảng; `EVENT_SYSTEM.md:13` ghi P-16. EVENT_SYSTEM không dẫn mã X-14 (chỉ P-16, P3, F3-16). ADR-0007, SECURITY, SANDBOX_DESIGN không liên quan X-14 (đúng).

Số event gốc thiếu trong danh mục đề xuất: lệnh `grep -nE "^\| \`?<tên>\`? \|"` cho 16 tên: chỉ AGENT_CREATED (dòng 114), TASK_CREATED (130), TASK_BLOCKED (138), TASK_COMPLETED (141), HUMAN_APPROVED (173), HUMAN_REJECTED (174) có hàng; 10 tên (AGENT_ASSIGNED, AGENT_STARTED, AGENT_THINKING, AGENT_TOOL_STARTED, AGENT_TOOL_COMPLETED, AGENT_WAITING_APPROVAL, AGENT_COMPLETED, AGENT_FAILED, WORKFLOW_STAGE_CHANGED, HUMAN_APPROVAL_REQUIRED) không có hàng. **Khẳng định "10/16 thiếu" của tài liệu là đúng** và được ghi ở `EVENT_SYSTEM.md` §0 ghi chú.

Concurrency 18: ADR-0010 §14 ("Không tăng lên 10 hoặc 18"), FR-231 ghi chú, PRD:151, RESOURCE_MANAGEMENT:190 ("không có nghĩa 18 agent đã được kiểm thử") làm rõ 5 là mặc định. Tuy nhiên không có chỗ nào nói thẳng "18 chỉ là trần cấu hình, không bật", và roadmap còn kế hoạch tăng: `IMPLEMENTATION_ROADMAP.md:215` ("tăng đồng thời 5 -> 10 bằng session thật", không banner), `:232` P3-12 ("rồi 10"), `:367` Phase 7 mục tiêu "Chứng minh bằng session thật rằng 18 agent chạy đồng thời" (không banner), `:530` ("Tăng dần 5 -> 10 -> 18"), `TESTING.md:126`. Một số dòng có điều kiện (IR-2:507, P2-13:194, :575, :591); các dòng còn lại chưa (F3-04). Không tài liệu nào tuyên bố đã test 18 (grep không có kết quả).

## 6. Bảo mật, credential, AGPL, mock

- Đọc credential: grep `\.credentials\.json|auth\.json` toàn repo (ngoài QA/owner-responses): mọi kết quả đều là cấm hoặc mô tả rủi ro (ví dụ ADR-0008:44,114, PROVIDER_INTEGRATION:326,424, DEVELOPMENT_GUIDE:143, CODING_STANDARDS:145). Không có chỉ dẫn đọc. PASS.
- Secret trong PoC và diff: regex `sk-[A-Za-z0-9_-]{16,}|Bearer ...|BEGIN .*PRIVATE KEY|AKIA...|ghp_...|xox..|JWT|AIza...|(password|secret|token|api_key)=...` chạy trên `docs/poc/*.md` = 0 kết quả; trên các dòng thêm vào của `git diff -U0` = 0; trên 3 file untracked = 0; toàn bộ `*.md` chỉ ra 3 kết quả vô hại (`SESSION_SECRET=change-me-dev-only` ở ENVIRONMENT_SETUP:165, mô tả rule gitleaks ở DEVSECOPS_GUIDE:41, một chuỗi trong research). PASS.
- Lưu token dạng plaintext: không thấy; `DATABASE_DESIGN.md:321` `credentialRef` chỉ là khóa. Xem F3-01 cho vấn đề khác.
- **F3-01 (P1)**: OWR-3 C: FR-088 "không lưu hoặc tự ý đọc credential trong ứng dụng", FR-098 "không có API key fallback". AC-640 (`AC_ADDENDUM_PART4_OWR2.md:92`) yêu cầu "0 lời gọi ghi vào OS credential manager hoặc keyring" và AC-642 "0 trường nhập API key". Nhưng tài liệu thiết kế vẫn có: `DATABASE_DESIGN.md:321,1303` (credential_ref và "Secrets (API key, git token) ... Windows Credential Manager"), `DEPLOYMENT.md:43,101` ("API keys nếu người dùng nhập ... Windows Credential Manager", `SECRETS_BACKEND=keychain`), `ENVIRONMENT_SETUP.md:162-166` ("Nếu bắt buộc dùng API key, lưu vào OS keychain qua `pnpm aiw secrets set`"), `ADR-0008:1,44` ("secret khác vào OS keychain"), `DATABASE_DESIGN.md:174-176` enum `provider_kind` giữ `anthropic_api, openai, gemini...` "cho phương án API key". Thiết kế hiện tại sẽ làm AC-640/AC-642 fail ngay từ đầu. Chưa có tài liệu nào phân định "secret không phải credential của provider" (ví dụ git token) so với FR-088.
- Hardcoded agent count trong thiết kế mức mã: `grep` không thấy `if agents == 18`; số 18 chỉ xuất hiện là trần cấu hình hoặc seed (D-016 18 vai trò, ADR-0016) hợp lệ.
- Auto deploy production: FR-126:146, DEPLOYMENT, CODING_STANDARDS:152 đều cấm. PASS.
- my-virtual-office (AGPL): ADR-0012:48 lint chặn import; không có chỉ dẫn sao chép. PASS.
- Mock: dùng FakeProvider chỉ trong test (AC-636, AC-643, TESTING.md); Roadmap Phase 2 "mock hoặc sandbox" là chủ trương Owner A.11. `provider_kind` còn `mock` (`DATABASE_DESIGN.md:174`, ghi "chỉ CI/test"; tồn đọng F-10 của vòng 2). Lưu ý: một số AC (AC-651..657) phụ thuộc đồng hồ giả và môi trường test, chấp nhận được.

## 7. Git

- `git status --short`: 73 dòng = 70 ` M` (tracked, đã sửa) + 3 `??` (untracked). Branch: `docs/phase0-owner-decisions`. Last 3 commits: `ffa065e docs(phase0): update validation, security gates, and acceptance criteria`; `f08240c docs(phase0): record owner decisions and architecture ADRs`; `32e7155 Update documentation: ...`. Không **sạch**: các thay đổi OWR-3 chưa commit.
- `git diff --check`: exit 0 (có cảnh báo "LF will be replaced by CRLF" cho 60+ file; chỉ là cảnh báo line ending, không phải lỗi whitespace).
- Untracked: `docs/DOC_SYNC_LOG_OWR3_A.md` (14.270 B), `docs/DOC_SYNC_LOG_OWR3_B.md` (12.987 B), `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (8.605 B). Không có secret, build artifact hay file lớn. `git status --ignored`: không có mục `!!`.
- Hai commit trước theo OWR-3 F, chưa push (không kiểm tra remote). Không tạo commit nào trong lần rà này.
- Quét secret trên diff tracked + untracked: xem §6, 0 kết quả.

## 8. Tuân thủ quy trình: PASS

- `ls apps packages src package.json node_modules`: tất cả "No such file". `find . -path ./.git -prune -o -type f ! -name '*.md' -print`: rỗng. Không có mã ứng dụng.
- `git -C <repo> status --porcelain | wc -l` cho ChatDev, pixel-agents, my-virtual-office, OpenHands, herdr: 0, 0, 0, 0, 0.
- Không tài liệu nào tuyên bố đã test 18 agent đồng thời. Docker chưa bật (POC_DOCKER_SANDBOX.md, TECHNICAL_VALIDATION). Không cài Codex.

## 9. Bảng phát hiện

| ID | Sev | Phát hiện | Bằng chứng (lệnh -> kết quả) | File:dòng | Đề xuất sửa |
|---|---|---|---|---|---|
| F3-01 | P1 | Thiết kế vẫn lưu secret/API key vào OS keychain và giữ kind API, mâu thuẫn FR-088, FR-098, AC-640, AC-642 (OWR-3 C) | `grep -rnE "Credential Manager\|keychain\|keytar"` -> DEPLOYMENT:43,101; ENVIRONMENT_SETUP:162-166; DATABASE_DESIGN:321,1303; ADR-0008:1,44; enum provider_kind dòng 174-176 | docs/development/DEPLOYMENT.md:43; docs/development/ENVIRONMENT_SETUP.md:162; docs/DATABASE_DESIGN.md:174,1303; docs/adr/ADR-0008-credentials-managed-by-cli.md:44; docs/product/traceability_parts/AC_ADDENDUM_PART4_OWR2.md:92 | Thêm banner OWR-3 C vào các file này; hỏi Owner xem secret không phải credential provider (git token...) có được lưu không; nếu không, đổi AC-640 hoặc bỏ kind API |
| F3-02 | P2 | Còn tên event có phân đoạn `_` hoặc UPPER_SNAKE xuất hiện như tên dùng thực tế mà không có chỉ dẫn dotted-canonical | `grep -rnE "\.[a-z]+_[a-z_]+"` -> PROVIDER_INTEGRATION:123,130,260,331 (`session.state_changed`, `rate_limit.updated`, `provider.status_changed`); grep tên UPPER: 18 file, 8 file không có chữ dotted/toLegacyName (AGENT_ARCHITECTURE, MULTI_AGENT_ORCHESTRATION, WORKFLOW_ENGINE, PRD, SRS, ACCEPTANCE_CRITERIA, CODING_STANDARDS, ORCHESTRATION_WORKFLOW_RESEARCH) | docs/PROVIDER_INTEGRATION.md:123,130; docs/product/ACCEPTANCE_CRITERIA.md:29,31,45; docs/AGENT_ARCHITECTURE.md:219 | Đổi tên hoặc thêm banner "tên tương thích, canonical xem EVENT_SYSTEM §0, §5.7"; sửa `rate_limit`, `state_changed` thành dạng không có `_` hoặc ghi rõ là lớp adapter |
| F3-03 | P2 | FR-117 gán nhãn `[Owner]` cho dải "AGENT_CREATED…HUMAN_REJECTED" trong khi 16 event gốc không như vậy (có EMERGENCY_STOP, PROVIDER_*, TASK_STARTED...) | `sed -n 137p FUNCTIONAL_REQUIREMENTS.md` | docs/product/FUNCTIONAL_REQUIREMENTS.md:137 | Ghi "6 trong 16 event gốc; phần còn lại [Đề xuất]" |
| F3-04 | P2 | Roadmap còn mục tiêu nâng concurrency lên 10/18 không kèm điều kiện; không có câu "18 chỉ là trần cấu hình, không bật" | `grep -nE "5 -> 10\|18 agent"` -> IMPLEMENTATION_ROADMAP:215, 232, 367, 530; TESTING:126 | docs/IMPLEMENTATION_ROADMAP.md:215,367,530; docs/development/TESTING.md:126 | Gắn nhãn OWR-3 A.2/D (không tăng nếu chưa xác minh điều khoản và usage); đổi mục tiêu Phase 7 thành "chỉ khi Owner cho phép" |
| F3-05 | P2 | SECURITY §0.6 và PROVIDER_INTEGRATION:204 đặt quy tắc tắt shell trên host (giả định của agent) cao hơn D-007 trong khi câu hỏi (i)/(ii) chưa được trả lời | đọc SECURITY.md:28-45; PROVIDER_INTEGRATION:204 | docs/SECURITY.md:32-39; docs/PROVIDER_INTEGRATION.md:204 | Gắn nhãn "giả định an toàn tạm thời, chưa phải quyết định" cho quy tắc 1 đến 4 |
| F3-06 | P2 | `adr/README.md` nói chưa có PoC report dù `docs/poc/` có 3 báo cáo | `grep -n "chưa có PoC report" docs/adr/README.md` -> dòng 16, 61; `ls docs/poc` -> 3 file | docs/adr/README.md:16,61 | Sửa: có PoC Drizzle (Partial), Skill (Partial), Docker BLOCKED; mức bằng chứng giữ nguyên |
| F3-07 | P2 | File trạng thái cấp gốc chưa cập nhật OWR-3 và còn số cũ | `grep -c OWR-3` = 0 cho PROJECT_STATUS, NEXT_SESSION, KNOWN_ISSUES, TASK_BACKLOG, WORK_PLAN, AGENTS; FINAL_REPORT:38,70,88 "12 NEEDS_OWNER_CLARIFICATION", "HEAD 32e7155 trên main, trống"; WORK_PLAN:3 "Phase 0 nội dung hoàn tất" | PHASE_0_FINAL_REPORT.md:38,70,88; WORK_PLAN.md:3; NEXT_SESSION.md:3 | Cập nhật khi viết báo cáo mục G.8; bỏ chữ "hoàn tất" |
| F3-08 | P2 | AC-022 ghi hai trạng thái khác nhau: "Owner-acknowledged" ở FR và "PENDING" ở AC | đọc FR:94 và ACCEPTANCE_CRITERIA:202 | docs/product/FUNCTIONAL_REQUIREMENTS.md:94; docs/product/ACCEPTANCE_CRITERIA.md:202 | Thống nhất thành "A.8 xác nhận Claude-only; AC-022 chưa được Owner nêu riêng" |
| F3-09 | P2 | PERMISSION_MODEL còn mô tả enum có `full` và "4 giá trị" | `grep -n full PERMISSION_MODEL.md` -> 36, 46, 327, 359 | docs/security/PERMISSION_MODEL.md:36,46,327,359 | Sửa theo X-3, giữ banner dòng 50-52 làm nguồn |
| F3-10 | P2 | Tồn đọng vòng 2 F-06: TESTING và ROADMAP vẫn "AC-001…AC-130"; ACCEPTANCE_CRITERIA.md chưa hợp nhất AC mới | `grep -n "AC-001…AC-130"` | docs/development/TESTING.md:11; docs/IMPLEMENTATION_ROADMAP.md:5 | Ghi "AC-001..130 + AC-201..657 PROPOSED" |
| F3-11 | P3 | ADR-0021 còn 59 dòng (< 60) | `wc -l` | docs/adr/ADR-0021-dynamic-skill-overlay.md | Bổ sung nếu muốn đạt ngưỡng cũ |
| F3-12 | P3 | FR-293..298 không nằm trong bảng truy vết nhưng AC-621..623 trích chúng (đã ghi chú loại trừ) | grep FR-29[3-8] | docs/product/FUNCTIONAL_REQUIREMENTS.md:297 | Thêm hàng truy vết "đề xuất" khi Owner duyệt |
| F3-13 | P3 | Văn bản tile lỗi thời có banner phủ nhưng không đánh dấu tại chỗ | grep "tile 16 hay 32" | docs/adr/ADR-0011-phaser-virtual-office.md:68,117; docs/adr/CONFLICTS_TO_RESOLVE.md:509 | Thêm "(đã giải quyết, A.7)" |
| F3-14 | P3 | AC-636 "chạm hoặc vượt 100%" và AC-607 cửa sổ từ `first_seen` là diễn giải của BA | đọc AC | docs/product/traceability_parts/AC_ADDENDUM_PART4_OWR2.md:31,76 | Hỏi Owner "vượt" hay "đạt", cửa sổ trượt hay cố định |
| F3-15 | P3 | Còn nhắc "11 engine kind" và "8 loại node" | grep | docs/product/FUNCTIONAL_REQUIREMENTS.md:368; docs/WORKFLOW_ENGINE.md:56; docs/product/traceability_parts/AC_ADDENDUM_PART2.md:90 | Đánh dấu lịch sử |
| F3-16 | P3 | EVENT_SYSTEM không dẫn X-14 (chỉ P-16) | `grep -c X-14 docs/EVENT_SYSTEM.md` = 0 | docs/EVENT_SYSTEM.md:13 | Thêm "X-14" |
| F3-17 | P3 | ADR-0015 tiêu đề, README bảng ADR-0014/0015 còn trạng thái cũ (đã có chú thích) | đọc README:30,34 | docs/adr/ADR-0015-design-ai-accent-purple-inter.md:1; docs/adr/README.md:34 | Đổi tiêu đề |
| F3-18 | P3 | Ví dụ timeout 30 phút trong research | `sed -n 516p CLAUDE_CODE_INTEGRATION.md` | docs/research/CLAUDE_CODE_INTEGRATION.md:516 | Ghi "ví dụ, thực tế 540 s" |

## 10. So sánh với QA vòng hai (F-01..F-17)

| Mã cũ | Mô tả | Trạng thái hiện tại |
|---|---|---|
| F-01 (P1) | 4 FR Must chưa có AC | **Đã xử lý**: AC-601..614; 0 FR Must thiếu AC |
| F-02 (P1) | `audit_events` | **Còn một phần**: 4 dòng ngoài `adr/`, QA, log (từ 17); chưa liệt kê file:dòng chính xác |
| F-03 (P1) | Overlay `.ai/**` | **Còn mở**: `DYNAMIC_SKILL_INJECTION.md` không được sửa trong đợt này |
| F-04 | Nhãn "Đã xác minh" | Chưa kiểm lại từng dòng (xem Chưa kiểm chứng) |
| F-05 | Tracking sai | Một phần: KNOWN_ISSUES #12 đã sửa; nhưng xem F3-07 |
| F-06 | AC-001..130 | **Còn mở** (F3-10) |
| F-07 | DOC_SYNC_LOG | Đợt mới có OWR3_A/B; không rà chi tiết |
| F-08 | CC0 VIRTUAL_OFFICE | Chưa kiểm lại |
| F-09 | `--append-system-prompt-file` | **Còn mở**: DYNAMIC_SKILL_INJECTION:96 vẫn `[ĐX]` |
| F-10 | `mock` trong `provider_kind` | **Còn mở** (và xem F3-01) |
| F-11 | REMAINING_RISKS DOC-04 | **Còn mở** (dòng 100) |
| F-12 | `(644)` | **Còn mở**: `SECURITY.md:599` |
| F-13 | D-021 trong Register | **Đã xử lý** (2 hàng `^| D-021`) |
| F-14..F-17 | AC/định dạng/RAM | Không đổi, P3 |

## 11. Chưa kiểm chứng

- Mọi hành vi runtime: SDK `systemPrompt.append`, `--append-system-prompt-file`, resume snapshot, Docker boundary, WSL2, Job Object, emergency stop, concurrency 5/10/18, điều khoản Anthropic/OpenAI (chỉ là lời Owner, tôi không xác minh).
- Chưa mở lại từng dòng của F-04 (nhãn "Đã xác minh" trong RESOURCE_MANAGEMENT, MULTI_AGENT_SCALABILITY, BACKEND_ARCHITECTURE), F-08, F-15, F-16.
- Không đọc toàn bộ 2 file DOC_SYNC_LOG_OWR3; không đối chiếu từng mục DS với diff.
- AC mới chỉ lấy mẫu 28 trong khoảng 76 AC thêm từ OWR-2 và OWR-3; 15 FR OWNER_DECIDED chỉ đối chiếu công thức và con số chính.
- Không kiểm tra liên kết nội bộ toàn bộ; không kiểm tra remote GitHub hay trạng thái push.
- Số liệu PoC (PGlite, token, thời gian) chỉ đối chiếu nhãn PASS/FAIL/BLOCKED, không chạy lại.
