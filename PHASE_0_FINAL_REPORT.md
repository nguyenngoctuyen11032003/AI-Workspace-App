# PHASE_0_FINAL_REPORT.md — AI WORKSPACE (cập nhật sau OWR-2)

Ngày: 2026-10-30. Viết cho: Owner. Nguồn chỉ thị: `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`.

**Phase 0 vẫn MỞ. Phase 1 chưa được Owner cho phép. Không mục nào được đánh dấu VERIFIED hay Completed.** Hoàn thành các đầu việc kỹ thuật không đồng nghĩa với phê duyệt. Lịch sử các lần phản hồi trước (OWR-1) nằm trong `PHASE_0_OWNER_DECISION_REGISTER.md` mục 8.

## A. Decision Register
Nguồn: `PHASE_0_OWNER_DECISION_REGISTER.md` mục 9 đến 12, `DECISION_LOG.md`, `docs/adr/`.

**Đã cập nhật:** Q-02 đến Q-20, D-003, D-005 đến D-021.

| Trạng thái | Quyết định |
|---|---|
| APPROVED (16) | D-001 (React 19 + Vite, NestJS, pnpm monorepo, Q-16), D-002, D-004 (Claude Agent SDK, Q-17), D-005 (hoãn Codex), D-006 (hoãn OpenHands), D-008, D-009, D-010, D-012, D-013, D-014, D-015, D-016, D-017, D-018, D-020 |
| APPROVED_CONDITIONAL (5) | D-003 (chỉ định hướng, chờ PoC Conductor), D-007 (sandbox hai tầng), D-011 (Phaser qua `OfficeRenderer`, benchmark trước khi khóa), D-019 (chỉ Phase 3 sau Security Review), D-021 (chờ OWN-AC-1..3) |
| VERIFIED | không có |
| PENDING_OWNER_CLARIFICATION (9 mục) | danh sách event gốc D-014 (mục 8.4 Prompt 1 chưa nằm trong repo); Q-07 điều khoản subscription; FR-122, FR-181, FR-182, FR-183; tile 16 hay 32; Owner xác nhận đã biết D-005 làm đổi nhóm chấp nhận MVP B; Q-15 có cho MIT asset không |

Mức bằng chứng của các ADR đã APPROVED vẫn **Partial hoặc Unverified** (xem `docs/adr/README.md`). ADR-0002 nâng lên Partial (PGlite) nhờ PoC. ADR mới: ADR-0021 cho D-021.

**Xung đột giữa OWR-2 và tài liệu gốc (12 mục X-1..X-12, ghi rõ, không ghi đè im lặng):**
1. Locale mặc định: OWR-2 là tiếng Việt, nhiều tài liệu thiết kế ghi tiếng Anh. Đã đồng bộ nhưng cần Owner biết.
2. D-005 hoãn Codex làm đổi nhóm chấp nhận MVP B của yêu cầu gốc (FR-077, AC-022). Cần Owner xác nhận đã biết.
3. Enum `permission_level` có `full`, Q-03 không hỗ trợ. Đã bỏ khỏi DB, còn lệch ở L0..L4 và L0..L5 giữa hai tài liệu.
4. Event naming UPPER_SNAKE trong tài liệu, Q-19 dùng dotted lowercase cho DB. Còn các đoạn tên có dấu gạch dưới.
5. Chính sách asset Q-15 chặt hơn đề xuất trước (CC0 mặc định). Ảnh hưởng Pixel Agents (MIT, nguồn gốc nghệ thuật chưa rõ).
6. Tile size 16 hay 32: OWR-2 không quyết định.
7. Timeout approval: nhiều giá trị, đã thống nhất 540 giây, còn cột `expires_at` mang hai nghĩa.
8. Retention audit 180 ngày trong SECURITY đã đổi thành 365 ngày.
9. OpenHands và Tier 2: D-006 hoãn OpenHands sau Phase 4, nhưng ADR-0007 Tier 2 dựa vào OpenHands.
10. Phase 2 chạy provider thật so với yêu cầu chỉ mock/sandbox khi cổng an ninh chưa đạt.
11. Concurrency theo provider (cap Claude 5, Codex 3) so với Q-07 không xác nhận quota.
12. Bốn FR: đề xuất của Owner chỉ là AC đề xuất, chưa phải nội dung FR gốc.

## B. FR Acceptance Criteria
Nguồn: `PHASE_0_ACCEPTANCE_CRITERIA_TRACEABILITY.md` (mục "Cập nhật OWR-2"), `docs/product/traceability_parts/AC_ADDENDUM_PART4_OWR2.md`.
- Đối chiếu lại văn bản FR gốc trước khi viết. Các AC mới (AC-601..614, AC-621..623) đều **PROPOSED**, không APPROVED.
- Tổng vẫn 199 FR: 109 AC_DEFINED, 74 AC_ADDED_PHASE0, **4 AC_PROPOSED_PENDING_CLARIFICATION** (FR-122, 181, 182, 183), **12 NEEDS_OWNER_CLARIFICATION**.
- **FR-122 Workflow Deliverables:** có AC đề xuất. Còn hỏi Owner: artifact nào bắt buộc; có export/download không hay chỉ lưu và truy xuất trong workspace.
- **FR-181 Incident Dashboard:** có AC đề xuất. Còn hỏi: approval chờ lâu là incident hay chỉ operational alert; quy tắc gom incident lặp lại (đề xuất, chưa duyệt).
- **FR-182 Progress:** công thức theo trọng số, trọng số bằng nhau cho MVP. Còn hỏi: xử lý task `cancelled`; chấp nhận equal-weight không; định nghĩa baseline.
- **FR-183 Performance:** không đồng nhất với Progress. Còn hỏi: nhiều metric hay một KPI tổng hợp; nếu KPI thì cần công thức và trọng số được duyệt.
- OWN-AC-1..3 của D-021 ghi thành AC-621..623, gắn FR-293..298 (đề xuất, chờ duyệt). Các FR-109, FR-113, FR-164 và các FR Codex/OpenHands đã chỉnh theo Q-09, Q-11, Q-14, D-005, D-006, có ghi nguồn OWR-2.

## C. Documentation Changes
- **Ghi nhận quyết định:** register, DECISION_LOG, ADR-0001..0021, CONFLICTS_TO_RESOLVE (cập nhật OWR-2, CF-056..062 tạm).
- **Đồng bộ nhóm A (bảo mật, provider, git, roadmap):** 44 thay đổi, `docs/DOC_SYNC_LOG_OWR2_A.md`. Gồm SECURITY mục 5 (SDK `canUseTool` cộng chính sách PreToolUse), retention 365 ngày, thống nhất timeout approval, phụ lục roadmap theo OWR-2 mục 8.
- **Đồng bộ nhóm B (DB, event, workflow, thiết kế, office, skill):** 30 thay đổi, `docs/DOC_SYNC_LOG_OWR2_B.md`. Gồm event dotted, `event_outbox` (đề xuất, có cờ), bỏ `full` và thêm `permission_profile`, `OfficeRenderer`, locale Việt, asset CC0, Q-16 APPROVED.
- **Báo cáo:** `PHASE_0_TECHNICAL_VALIDATION_REPORT.md` (phụ lục PoC), `PHASE_0_REMAINING_RISKS.md`, `PHASE_0_SECURITY_SIGNOFF_CHECKLIST.md` (các ô ký vẫn để trống cho Owner), `AGENTS.md`, `NEXT_SESSION.md`, `PROJECT_STATUS.md`, `KNOWN_ISSUES.md`.
- **Chưa đồng bộ được và lý do:** xung đột X-1..X-12 còn mở; EVENT_SYSTEM vẫn liệt kê tên UPPER_SNAKE (ghi chú là tên tương thích); `WORKSPACE_AND_GIT_ISOLATION` mục 7.1 còn thiết kế CLI; `AGENT_ORGANIZATION` L0..L4 lệch L0..L5; `expires_at` hai nghĩa; CF-048 (WAITING_APPROVAL có giữ slot đồng thời không). Không sửa im lặng, đã để dấu hiệu tại chỗ.

## D. PoC Results
Chỉ ba nhóm được cấp phép chạy. Không chạy 27 runbook, không cài Codex, không đổi cấu hình WSL hay Docker.
| PoC | Kết quả | Bằng chứng |
|---|---|---|
| Drizzle | 9 PASS, 1 FAIL, 1 BLOCKED. Engine PGlite 0.5.13 (báo PostgreSQL 18.6), không PG16 | `docs/poc/POC_DRIZZLE.md`; tôi xác nhận `npm view drizzle-orm@0.45.4` tồn tại và là `latest` |
| Dynamic Skill/Resume | SK-01 PASS, SK-02 PASS, SK-03 PASS (cơ chế, có điều kiện), SK-04 **SIMULATED**, SK-05 PASS (mức guard/CLI). SDK NOT TESTED | `docs/poc/POC_DYNAMIC_SKILL.md`; 8 lần gọi model, 0,006283 USD |
| Docker sandbox | **BLOCKED** (9 mục VAL-DOCKER chưa chạy) | `docs/poc/POC_DOCKER_SANDBOX.md`; daemon không kết nối được lúc 13:39 |
| Conductor (Q-20) | **NOT RUN** (ngoài phạm vi được cấp phép) | không |

Chi tiết quan trọng:
- **Drizzle FAIL duy nhất:** `drizzle-kit generate` và `check` không phát hiện thay đổi thủ công trong DB; `pull` có. Giới hạn: unique trên bảng phân vùng phải chứa khóa phân vùng, nên unique theo `sequence` phải do ứng dụng bảo đảm. Chưa thử `postgres.js`. Một PostgreSQL 17 cục bộ đang chạy cần thông tin đăng nhập, không dùng.
- **SK-03:** `--resume` với file mới trả marker cũ (snapshot dùng lại). `--system-prompt-snapshot off` trả marker mới. Hash-check cộng session mới trả marker mới kèm audit. Chi phí làm mới cao hơn khoảng 47% trong một phép đo, chưa đủ mẫu.
- **SK-04:** agent thật bị chặn quyền ghi hai lần, nên phần ghi tệp là script. Giới hạn theo Owner: cùng OS user nên guard chỉ biết thay đổi, thời điểm, phiên thực thi, **không xác định ai sửa**.
- **Docker:** RAM trống 4,04 GB trên 15,77 GB, có thể không đủ khi bật Docker Desktop.
- **Độ độc lập:** tôi chỉ kiểm lại một phần (`npm view`, sự tồn tại của file đầu ra). Tôi **chưa tái chạy** các PoC.

## E. Git Status
Kiểm tra lúc bắt đầu đợt này (đã bỏ qua thông tin cũ):
- `git status --short`, `git status --porcelain=v1`: trống (0 file). `git diff --check`: exit 0. HEAD `32e7155` trên `main`, do Owner commit.
- Sau các thay đổi của đợt này: **xem mục Git trong báo cáo trò chuyện**. Số liệu thật lấy từ `git status` lúc kết thúc, không suy từ trạng thái cũ.
- Quét bí mật trên file đã track: không thấy khóa, token hay private key (chỉ một dòng tài liệu mô tả mẫu quét).
- Không push. Commit hash: ghi trong tin nhắn cuối sau khi commit.

## F. Remaining Risks
Sổ đầy đủ: `PHASE_0_REMAINING_RISKS.md` (có mục "Cập nhật sau OWR-2", RR-OWR2-01..10).
- **Bảo mật:** Windows không có sandbox cấp hệ điều hành cho CLI; cùng OS user nên không phân biệt Owner và agent; SK-04 chưa chứng minh trên agent thật.
- **Điều khoản provider:** Q-07 chưa xác minh. Concurrency 5 chỉ là giới hạn của ứng dụng. Không tăng lên 10 hoặc 18.
- **Sandbox:** Docker chưa kiểm chứng; RAM thấp; Tier 2 phụ thuộc OpenHands bị hoãn.
- **Quyết định thiếu dữ liệu:** Conductor chưa PoC; PG16 chưa kiểm; danh sách event gốc chưa có; 4 FR cộng 12 FR chưa viết được AC.
- **Phạm vi:** D-005 đổi nhóm chấp nhận MVP B.
- **Bằng chứng:** các số đo cũ về 18 agent và 5 tiến trình song song do một agent đo một lần, chưa tái hiện. Chưa có kiểm thử 18 agent đồng thời.

## G. Phase 0 Exit Criteria
| # | Requirement | Expected result | Evidence | Status | Remaining action | Owner approval required |
|---|---|---|---|---|---|---|
| 1 | Không còn P0 chưa xử lý | 0 P0 | QA lần hai: 0 P0 (`PHASE_0_QA_REVIEW_2.md`). QA lần ba **chưa chạy** sau đợt OWR-2 | CHƯA ĐẠT (cần QA lại sau thay đổi mới) | Chạy QA độc lập lần ba | Không |
| 2 | AC hoàn thiện và truy vết | 100% FR có AC hoặc cờ rõ | 199/199 có dòng truy vết; 183 có AC (109 + 74), 4 đề xuất chờ làm rõ, 12 chờ Owner | CHƯA ĐẠT | Owner trả lời; viết AC cho 12 FR còn lại | Có |
| 3 | Tài liệu đã đồng bộ | Không giá trị trái ngược giữa nguồn và chính tắc | 74 thay đổi OWR-2 + các đợt trước; 12 xung đột còn mở | CHƯA ĐẠT | Owner quyết X-1..X-12, đồng bộ phần còn lại | Có |
| 4 | Mâu thuẫn xử lý hoặc ghi chính xác | Mỗi CF có trạng thái rõ | `docs/adr/CONFLICTS_TO_RESOLVE.md` (cập nhật OWR-2) | ĐẠT MỘT PHẦN | Đóng các CF còn STILL_OPEN | Có |
| 5 | ADR có trạng thái minh bạch | PENDING, APPROVED hoặc VERIFIED đúng | 21 ADR; không VERIFIED | ĐẠT (về trạng thái) | Nâng VERIFIED khi có bằng chứng | Không |
| 6 | PoC được cấp phép đã chạy | Drizzle, Skill/Resume, Docker | Drizzle và Skill chạy (Partial); Docker BLOCKED | CHƯA ĐẠT | Owner mở Docker; lặp Drizzle trên PG16; SK-04 agent thật; test qua SDK | Có (mở Docker) |
| 7 | QA độc lập lần hai/ba đạt | PASS không có P0 | Lần hai PASS WITH FIXES (trước OWR-2) | CHƯA ĐẠT | QA lần ba | Không |
| 8 | Rủi ro chưa giải quyết được trình bày | Sổ rủi ro đầy đủ | `PHASE_0_REMAINING_RISKS.md` | ĐẠT | Cập nhật khi có thay đổi | Có (chấp nhận) |
| 9 | Owner đủ thông tin để quyết định | Sổ quyết định và báo cáo này | Mục A đến F | ĐẠT | Owner trả lời | Có |
| 10 | Cổng an ninh trước agent chạy thật | Authn, authz, permission, credential isolation, audit, sandbox, emergency stop | Chưa có mã; đặc tả trong SECURITY; sandbox chưa kiểm | CHƯA ĐẠT (đúng, vì chưa có mã) | Làm ở đầu Phase 2 theo OWR-2 | Có |
| 11 | Owner cho phép bắt đầu Phase 1 | Xác nhận rõ ràng | Không có | **CHƯA** | Owner xác nhận | **Có** |

Phase 0 **không** được tự tuyên bố PASS. Tôi dừng tại đây và chờ Owner.
