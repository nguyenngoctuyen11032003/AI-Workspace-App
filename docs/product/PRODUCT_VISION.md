# PRODUCT VISION — AI WORKSPACE

**Tagline:** "AI WORKSPACE — Your Virtual AI Software Company"
**Phiên bản tài liệu:** 0.1 (Phase 0 — Documentation only)
**Tác giả:** Agent 12 — Business Analyst
**Trạng thái:** Draft — chờ Owner phê duyệt
**Ngày:** 2026-10-30

> Quy ước: Nội dung đánh dấu **[Owner]** là yêu cầu gốc do Owner đưa ra (source of truth). Nội dung đánh dấu **[Giả định]** là giả định của BA, chưa được xác nhận. Nội dung đánh dấu **[Đề xuất]** là đề xuất của BA, cần Owner quyết định.

---

## 1. Mục đích tài liệu

Tài liệu này xác lập tầm nhìn sản phẩm (product vision) cho AI WORKSPACE, làm "kim chỉ nam" cho toàn bộ 18 agent trong team Phase 0 và các phase sau. Nó trả lời các câu hỏi:

- Sản phẩm này là gì và **không phải** là gì?
- Ai là người dùng và họ nhận được giá trị gì?
- Nguyên tắc bất biến nào định hướng mọi quyết định thiết kế?
- Thành công được đo lường như thế nào?

Tài liệu không mô tả chi tiết chức năng (xem `PRD.md`, `FUNCTIONAL_REQUIREMENTS.md`) hay kiến trúc (xem `docs/architecture/`).

## 2. Phạm vi

**Trong phạm vi:**
- Tuyên bố tầm nhìn, vấn đề cần giải quyết, giá trị cốt lõi.
- Persona người dùng và kịch bản giá trị (value scenarios).
- Nguyên tắc sản phẩm (product principles) không thể thương lượng.
- Phân định MVP vs tương lai ở mức cao.
- Chỉ số thành công (success metrics) ở mức đề xuất.

**Ngoài phạm vi:**
- Yêu cầu chức năng chi tiết, NFR, kiến trúc, lựa chọn công nghệ.
- Kế hoạch kinh doanh, giá bán, go-to-market (Owner chưa cung cấp thông tin).

## 3. Tuyên bố tầm nhìn (Vision Statement)

**[Owner]** AI WORKSPACE là một nền tảng quản lý và điều phối (orchestrate) nhiều AI agent, mô phỏng một công ty phần mềm thực thụ. Người dùng nhập một yêu cầu bằng ngôn ngữ tự nhiên; hệ thống phân tích yêu cầu, thành lập đội agent, phân công công việc, thiết kế, lập trình, kiểm thử, rà soát bảo mật và triển khai. Toàn bộ agent xuất hiện dưới dạng nhân vật chibi trong một văn phòng 2D pixel-art, nơi người dùng quan sát, tương tác và điều khiển theo thời gian thực.

**[Owner]** Người dùng giữ vai trò **OWNER / PRODUCT OWNER / DIRECTOR** của công ty ảo, có quyền nghiệm thu cuối cùng (final acceptance).

**[Owner]** AI WORKSPACE **không phải** là chatbot và **không phải** chỉ là dashboard. Các agent thực sự thực thi công việc (sinh code thật, chạy test thật, tạo branch git thật, build thật).

### 3.1 Một câu tóm tắt

> "Biến một người thành giám đốc của cả một công ty phần mềm AI — nơi mỗi yêu cầu được một đội ngũ agent chuyên môn hóa tiếp nhận, thực thi và bàn giao, còn người dùng chỉ cần quan sát, phê duyệt và nghiệm thu."

## 4. Vấn đề cần giải quyết

**[Giả định]** Các vấn đề dưới đây được BA suy ra từ yêu cầu của Owner; Owner chưa phát biểu trực tiếp "pain points".

| # | Vấn đề hiện tại | Cách AI WORKSPACE giải quyết |
|---|---|---|
| P1 | Sử dụng AI coding (Claude Code, Codex...) là tương tác 1-1 với một agent duy nhất; người dùng phải tự chia nhỏ việc, tự ghép kết quả, tự review. | Hệ thống tự thành lập team, phân rã công việc, điều phối nhiều agent song song, tổng hợp và review theo quy trình. |
| P2 | Không có cái nhìn tổng thể "ai đang làm gì, tốn bao nhiêu, ở giai đoạn nào". | Virtual Office 2D + Dashboard + Activity Center hiển thị trạng thái thật từ event system. |
| P3 | Kiến thức/cách làm việc dạy cho AI bị mất sau mỗi phiên. | Skill Library + Teach Agent flow lưu kiến thức có thể review/sửa/xóa, gán cho agent theo task. |
| P4 | Thiếu quy trình chuẩn (BA → Design → Dev → QA → Security → Deploy → Acceptance) khi dùng AI. | Workflow engine bền vững (durable) với template "FULL SOFTWARE DEVELOPMENT LIFECYCLE" 8 giai đoạn và human approval gate. |
| P5 | Rủi ro bảo mật khi để AI tự chạy lệnh trên máy. | Sandbox Docker, allow/deny list, credential isolation, audit log, emergency stop. |
| P6 | Mất trạng thái khi app/máy khởi động lại giữa chừng. | Checkpoint, retry, recovery — không mất state khi restart. |

## 5. Người dùng mục tiêu (Personas)

**[Owner]** Chế độ chính là **single-user desktop**: một người dùng duy nhất là Owner/Director. Multi-user là định hướng tương lai (xem `USER_ROLES_AND_PERMISSIONS.md`).

### Persona 1 — "Giám đốc kỹ thuật một người" (Primary)
- Developer/founder/tech lead có kinh nghiệm, đã có tài khoản và CLI của Claude Code và/hoặc Codex đăng nhập sẵn trên máy.
- Muốn giao cả một dự án hoặc một feature lớn cho "công ty AI" và chỉ can thiệp ở các điểm phê duyệt.
- Giá trị: tốc độ, tầm nhìn toàn cảnh, kiểm soát chi phí token, tin tưởng vào quy trình QA/Security.

### Persona 2 — "Product Owner không chuyên code" **[Giả định]**
- Hiểu sản phẩm, viết yêu cầu tốt nhưng không đọc code sâu.
- Dùng Product Preview, QA & Security Report, screenshot markup để nghiệm thu.
- Giá trị: nhận sản phẩm chạy được mà không cần đọc diff.

### Persona 3 — "Viewer / Stakeholder" **[Giả định — tương lai multi-user]**
- Chỉ quan sát Virtual Office và dashboard, không có quyền thay đổi.

## 6. Giá trị cốt lõi (Value Proposition)

1. **Một yêu cầu → một đội ngũ.** Từ prompt tự nhiên đến team agent có vai trò rõ ràng (18 vai trò mặc định) trong 12 phòng ban mặc định.
2. **Công việc thật, không mô phỏng.** Code, test, branch, build là thật; Virtual Office chỉ là lớp trực quan hóa của sự kiện thật.
3. **Con người là người quyết định.** Mọi giai đoạn quan trọng có approval gate; không auto-deploy production; Owner nghiệm thu cuối.
4. **Tận dụng những gì người dùng đã có.** Dùng lại CLI login của Claude Code/Codex; không bắt buộc API key.
5. **Minh bạch chi phí và chất lượng.** Dashboard chi phí, token, performance, lỗi — dữ liệu thật, không giả lập.
6. **An toàn mặc định.** Sandbox, giới hạn tài nguyên, cô lập credential, emergency stop.
7. **Kiến thức tích lũy có kiểm soát.** Skill/knowledge được dạy, lưu, phiên bản hóa, và luôn có thể review/sửa/xóa.

## 7. Nguyên tắc sản phẩm (Product Principles) — không thương lượng

| # | Nguyên tắc | Nguồn |
|---|---|---|
| PP-01 | **Event là nguồn sự thật duy nhất** cho UI và Virtual Office. Nhân vật không di chuyển/animation nếu không có event thật tương ứng. | [Owner] |
| PP-02 | **Không có dữ liệu giả** trên dashboard. Mọi metric đến từ dữ liệu thật của hệ thống. | [Owner] |
| PP-03 | **Human-in-the-loop theo chính sách.** Approval policy có thể cấu hình, nhưng mặc định bật ở BA, UI/UX, Acceptance; không auto-deploy production. | [Owner] |
| PP-04 | **Reject tạo revision task có mục tiêu, không chạy lại toàn bộ.** Revision được đánh phiên bản. | [Owner] |
| PP-05 | **Agent không sửa code của agent khác nếu không được phép.** File ownership theo task. | [Owner] |
| PP-06 | **Không bao giờ hỏi mật khẩu, không trích xuất token, không lưu OAuth plaintext, không gửi credential lên frontend/log, không chia sẻ credential giữa agent.** | [Owner] |
| PP-07 | **Không mất state khi restart.** Mọi workflow/task có checkpoint và có thể recover. | [Owner] |
| PP-08 | **Kiến thức dạy cho agent không tự động persist** — phải qua review và lưu có chủ đích. | [Owner] |
| PP-09 | **Chỉ load skill liên quan** cho từng task (không nhồi toàn bộ library vào context). | [Owner] |
| PP-10 | **Premium UI** — 22 màn hình chuẩn, trải nghiệm cao cấp. | [Owner] |

## 8. Bức tranh sản phẩm (Product Landscape) — 12 nhóm năng lực

**[Owner]** đã liệt kê 12 nhóm năng lực; BA tóm tắt lại để đối chiếu với PRD:

1. AI Company Management — multi-company/workspace, 12 phòng ban mặc định, org chart drag-drop, trưởng phòng, cấp quản lý, ngân sách, hiệu suất phòng ban.
2. Dynamic Agent Management — Agent Builder, lifecycle 12 trạng thái, Agent Profile, cộng tác giữa agent.
3. Skill Management — Skill Library, custom skill theo chuẩn Agent Skills (`skills/<name>/SKILL.md`), Teach Agent flow, gán skill many-to-many.
4. AI Provider Hub — adapter Claude Code, Codex (CLI-based), OpenHands, generic CLI; 13 standard operations; luật bảo mật xác thực; provider dashboard.
5. Orchestration & Workflow — durable engine, task, AUTO/MANUAL assignment, Workflow Builder node-based, event system, template SDLC 8 giai đoạn.
6. Virtual Office 2D — 14 phòng, zoom/pan/floors/edit layout, hệ thống nhân vật 13 animation, speech bubble, click-to-interact.
7. Dashboards — overview, project, agent, workflow, cost, quality, system.
8. Approval Center — chính sách phê duyệt cấu hình được.
9. Security & Sandbox — workspace per project, Docker sandbox, limits, allow/deny, credential isolation, audit, emergency stop.
10. Multi-agent Orchestration — concurrency mặc định 5, tối đa 18; 18 vai trò mặc định; worktree per agent; file ownership; 10 task status; message bus; resource manager; monitoring.
11. Premium UI — 22 màn hình.
12. Reliability — checkpoint, retry, recovery, không mất state.

## 9. MVP vs. Tương lai (mức cao)

**[Owner]** đã định nghĩa tiêu chí nghiệm thu MVP nhóm A–G (Agent, Provider, Workflow, Skill, Virtual Office, Security, Reliability). Chi tiết tại `ACCEPTANCE_CRITERIA.md`.

**[Đề xuất]** Phân tầng:

| Tầng | Nội dung |
|---|---|
| **MVP (Phase 1)** | Single-company, single-user; Claude Code + Codex adapter; Agent Builder + lifecycle; Skill Library + Teach Agent cơ bản; Workflow engine durable + template SDLC 8 giai đoạn; Task Board; Approval Center; Virtual Office 2D với 14 phòng và animation theo event; Docker sandbox + emergency stop; checkpoint/recovery; các dashboard cốt lõi. |
| **Phase 2** | OpenHands + generic CLI adapter; Workflow Builder node-based đầy đủ với dry-run; multi-company/workspace; org chart drag-drop nâng cao; edit layout/furniture văn phòng; cost management nâng cao; 22 màn hình hoàn chỉnh premium. |
| **Phase 3 (Tương lai)** | Multi-user với role Owner/Project Owner/Viewer; floors nhiều tầng; marketplace skill; tích hợp thêm provider. |

> Việc Owner yêu cầu "22 màn hình premium" và "tiêu chí MVP A–G" cùng lúc tạo ra câu hỏi: 22 màn hình có thuộc MVP không? Xem mục 13.

## 10. Chỉ số thành công (Success Metrics) — [Đề xuất]

Owner chưa đưa ra chỉ số đo lường. BA đề xuất để Owner chọn/sửa:

| Chỉ số | Mục tiêu đề xuất | Cách đo |
|---|---|---|
| Time-to-first-preview | ≤ 60 phút cho dự án web nhỏ (≤ 10 màn hình) từ prompt đến Build & Deployment preview | Timestamp event PROJECT_CREATED → PREVIEW_READY |
| Tỷ lệ workflow hoàn thành không cần can thiệp ngoài approval gate | ≥ 70% | Số workflow COMPLETED / tổng workflow, loại trừ các can thiệp manual |
| Tỷ lệ recovery thành công sau restart | 100% | Test kịch bản kill app giữa chừng |
| Độ trễ event → Virtual Office | ≤ 500 ms (p95) | Timestamp event vs frame render |
| Chi phí token minh bạch | 100% task có token/cost ghi nhận | Audit so sánh với GetUsage của provider |
| Số sự cố credential leak | 0 | Security review + log scan |

## 11. Dependencies (phụ thuộc)

| Loại | Phụ thuộc | Ghi chú |
|---|---|---|
| Bên ngoài | Claude Code CLI, Codex CLI đã cài và login trên máy người dùng | [Owner] không yêu cầu API key nếu CLI auth hoạt động |
| Bên ngoài | Docker Desktop (hoặc tương đương) trên Windows 11 | Cần cho sandbox [Owner]; chưa rõ hành vi khi không có Docker |
| Bên ngoài | Git | Branch/worktree per agent |
| Bên ngoài | OpenHands (Phase 2) | Chưa rõ phiên bản/cách tích hợp |
| Nội bộ | `docs/architecture/` (Agent kiến trúc) — quyết định Tauri vs web, DB, event bus | Ảnh hưởng NFR |
| Nội bộ | `docs/design/` (UI/UX) — 22 màn hình, pixel-art asset | Ảnh hưởng phạm vi MVP |
| Nội bộ | `docs/security/` — sandbox model, credential storage | Ảnh hưởng FR nhóm Security |

## 12. Rủi ro (Risks)

| ID | Rủi ro | Mức | Giảm thiểu đề xuất |
|---|---|---|---|
| R-01 | Phạm vi quá lớn (12 nhóm năng lực, 22 màn hình, 18 vai trò) cho MVP | Cao | Chốt MVP theo A–G, dời phần còn lại sang Phase 2 |
| R-02 | Provider CLI thay đổi giao diện/định dạng output ngoài kiểm soát | Cao | Adapter layer cô lập, test contract định kỳ, version pinning |
| R-03 | Rate limit / quota của tài khoản CLI cá nhân không đủ cho 18 agent song song | Cao | Resource manager, concurrency mặc định 5, queue, hiển thị quota |
| R-04 | Docker sandbox vs. yêu cầu "reuse CLI login" trên host: CLI login nằm trên host, sandbox cô lập có thể không thấy credential | Cao | Cần quyết định kiến trúc (xem mục 13) |
| R-05 | Virtual Office tốn hiệu năng render khi 18 nhân vật + nhiều event | Trung bình | Giới hạn FPS, batch event, canvas/WebGL |
| R-06 | Chi phí token tăng cao do nhiều agent review lẫn nhau | Trung bình | Budget per agent/task, cảnh báo, chính sách review |
| R-07 | Durable workflow engine là thành phần phức tạp, dễ lỗi | Cao | Chọn engine đã kiểm chứng hoặc thiết kế event-sourcing rõ ràng |

## 13. Vấn đề chưa giải quyết / Mâu thuẫn phát hiện

| ID | Vấn đề | Mô tả | Hệ quả nếu không chốt |
|---|---|---|---|
| OI-01 | **Host execution vs. Docker sandbox** | Owner yêu cầu (a) dùng lại CLI login trên máy (host), (b) agent chạy trong Docker sandbox với FS/network restriction. CLI login (Claude Code/Codex) lưu credential ở host; chạy CLI trong container cần mount credential hoặc chạy CLI ở host và chỉ sandbox phần thực thi code. | Không thể thiết kế adapter và sandbox |
| OI-02 | **18 agent song song vs. rate limit provider** | 18 agent dùng chung 1 tài khoản CLI sẽ chạm rate limit. Owner chưa nói có hỗ trợ nhiều tài khoản/provider đồng thời hay chính sách queue. | Không thể định NFR hiệu năng thực tế |
| OI-03 | **Tauri desktop vs. web app** | Owner mô tả "single-user desktop mode" và Windows 11, nhưng chưa chốt công nghệ (Tauri/Electron/web local). Ảnh hưởng tới credential manager OS, Docker access, file system. | Kiến trúc không thể khởi động |
| OI-04 | **22 màn hình premium có thuộc MVP?** | Tiêu chí MVP A–G không đề cập UI đầy đủ. | Ước lượng sai phạm vi |
| OI-05 | **Multi-company/workspace trong MVP?** | Được liệt kê ở nhóm năng lực 1 nhưng không có trong tiêu chí A–G. | Data model khác nhau |
| OI-06 | **"No API key required if CLI auth works"** | Khi CLI auth không hoạt động thì có hỗ trợ API key không, lưu ở đâu? | Ảnh hưởng luật bảo mật |
| OI-07 | **Ngôn ngữ UI** | Owner viết yêu cầu bằng tiếng Việt + thuật ngữ Anh; chưa chốt UI tiếng Việt, Anh hay song ngữ. | i18n phải quyết sớm |
| OI-08 | **Agent "Code Reviewer/Product Evaluator"** là 1 hay 2 vai trò? | Danh sách 18 vai trò ghép hai vai trò này thành một mục, nhưng danh sách phòng ban tách Code Review và Product Evaluation. | Mapping vai trò ↔ phòng ban |
| OI-09 | **OpenHands adapter**: CLI hay API? | Owner nói adapter OpenHands nhưng không nói cơ chế. | Thiết kế adapter |

## 14. Giả định (Assumptions)

- A-01 Người dùng đã cài Claude Code CLI và/hoặc Codex CLI, đã login thành công trước khi dùng AI WORKSPACE.
- A-02 Máy người dùng có Docker Desktop; nếu không có, hệ thống sẽ cảnh báo và (tùy Owner) cho phép chế độ host-execution giới hạn.
- A-03 Một "project" tương ứng một git repository (có thể do hệ thống khởi tạo).
- A-04 MVP là single-company, single-user, chạy local trên Windows 11.
- A-05 Dữ liệu lưu local (SQLite hoặc tương đương) — do Agent kiến trúc quyết định.

## 15. Cần người dùng phê duyệt

| # | Hạng mục | Lựa chọn / câu hỏi |
|---|---|---|
| AP-01 | Tuyên bố tầm nhìn mục 3 và 10 nguyên tắc mục 7 | Phê duyệt / sửa |
| AP-02 | Persona 2, 3 | Giữ / bỏ |
| AP-03 | Phân tầng MVP / Phase 2 / Phase 3 (mục 9) | Phê duyệt / điều chỉnh |
| AP-04 | Chỉ số thành công (mục 10) | Chọn giá trị mục tiêu |
| AP-05 | OI-01 Host vs sandbox | (a) CLI ở host, code chạy trong sandbox; (b) mount credential vào container; (c) khác |
| AP-06 | OI-02 Rate limit | (a) chỉ 1 tài khoản, queue; (b) nhiều tài khoản; (c) mix provider |
| AP-07 | OI-03 Tauri vs web | Chốt công nghệ shell |
| AP-08 | OI-04, OI-05 phạm vi MVP | 22 màn hình & multi-company thuộc MVP hay Phase 2 |
| AP-09 | OI-06 API key fallback | Có / không; nếu có, lưu bằng OS credential manager |
| AP-10 | OI-07 Ngôn ngữ UI | Việt / Anh / song ngữ |
| AP-11 | Giả định A-01…A-05 | Xác nhận |

---
*Tài liệu liên quan:* `PRD.md`, `SRS.md`, `FUNCTIONAL_REQUIREMENTS.md`, `NON_FUNCTIONAL_REQUIREMENTS.md`, `USER_ROLES_AND_PERMISSIONS.md`, `USER_FLOWS.md`, `ACCEPTANCE_CRITERIA.md`.
