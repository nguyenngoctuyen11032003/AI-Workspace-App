# OWR-2 — Phản hồi chính thức của Owner, Phase 0 Finalization & Execution Authorization

Nguồn: tin nhắn của Owner trong phiên làm việc ngày 2026-10-30, sao chép lại nguyên nội dung quyết định để làm nguồn truy vết. Mã tham chiếu: **OWR-2**. Không phải tài liệu do agent sinh ra. Mọi thay đổi trong tài liệu khác phải dẫn mã Q, D hoặc FR tương ứng kèm OWR-2.

Lưu ý quan trọng của Owner: đây là chỉ thị cập nhật hồ sơ và thực hiện công việc trong phạm vi được cấp phép. **Không hiểu rằng Phase 0 đã hoàn tất hoặc Owner đã cho phép bắt đầu Phase 1.**

## 1. Q-16 đến Q-20
- **Q-16 Frontend và monorepo:** chấp thuận React 19 + Vite, NestJS backend, TypeScript monorepo, pnpm workspace. Không chuyển sang Next.js trong MVP.
- **Q-17 Claude provider:** chấp thuận Claude Agent SDK làm adapter chính cho Claude. Không dùng raw CLI spawn làm kiến trúc tích hợp chính. CLI chỉ dùng cho tác vụ hỗ trợ, kiểm tra hoặc PoC được cấp phép. Phải phân biệt subscription authentication, API authentication và usage limits. Không mặc định rằng dùng Agent SDK đồng nghĩa với được phép chạy số session không giới hạn.
- **Q-18 Dynamic Skill Injection:** chấp thuận có điều kiện D-021 với ba tiêu chí bắt buộc.
  - **OWN-AC-1 Dynamic Skill Loading:** Owner/Admin thêm hoặc sửa file Markdown trong `.ai/skills/<agent_name>/`. Task mới tự động nhận nội dung Skill mới. Không yêu cầu restart ứng dụng hoặc recompile.
  - **OWN-AC-2 Unauthorized Overlay Modification:** nếu Agent dùng Bash hoặc công cụ khác để thay đổi file overlay ngoài phạm vi quyền được phép, hệ thống phải ghi audit log, đánh dấu thay đổi đáng ngờ, tạm dừng Agent cho đến khi Manager/Owner xác nhận, và không cho Agent tự tiếp tục trước khi có xác nhận. Lưu ý: vì Owner và Agent có thể dùng cùng Windows OS user, hệ thống không được tuyên bố xác định tuyệt đối danh tính người sửa file. Chỉ được xác định thay đổi, thời điểm, phiên thực thi và bằng chứng audit liên quan.
  - **OWN-AC-3 Resume Session Context Refresh:** khi một Claude session bị gián đoạn và được resume, hệ thống phải kiểm tra phiên bản/hash của Skill và System Prompt. Nếu nội dung đã đổi, không được tiếp tục dùng context cũ mà không kiểm tra. Phải có cơ chế tạo session mới hoặc refresh context đã được PoC chứng minh hoạt động. Ghi audit log về việc phát hiện thay đổi và cách xử lý context.
  - D-021 chỉ được xem là chấp thuận có điều kiện cho đến khi ba tiêu chí trên được kiểm thử thành công.
- **Q-19 Event naming:** database event dùng `lowercase.dot.notation`, ví dụ `task.started`, `task.completed`, `agent.blocked`. API hoặc lớp tương thích bên ngoài có thể ánh xạ sang `UPPER_SNAKE_CASE` nếu cần. Không tự ý tạo hai hệ thống event naming độc lập.
- **Q-20 Workflow Conductor:** chấp thuận định hướng BullMQ + PostgreSQL. Chưa khóa implementation cho đến khi PoC chứng minh: crash recovery; idempotency; không chạy lặp node ngoài ý muốn; retry an toàn; approval state lưu bền vững; worker không bị giữ trong lúc chờ Owner approval. Nếu không đạt, báo cáo phương án thay thế như Temporal hoặc LangGraph để Owner quyết định.

## 2. Q-02 đến Q-15
- Q-02: không tích hợp OpenHands ACP trong MVP.
- Q-03: chấp thuận permission level L0 đến L5, dùng `permission_level` và `permission_profile`. Không hỗ trợ quyền `full`.
- Q-04: Agent được merge vào nhánh `integration/*` khi vượt qua các gate. Merge vào `main` luôn cần Owner approval. Lead Agent không được thay thế quyền phê duyệt của Owner.
- Q-05: approval wait timeout tối đa 540 giây. Hết thời gian thì lưu trạng thái và chuyển sang `WAITING_HUMAN`, không giữ worker. Có thể nhắc Owner sau 24 giờ. Production deploy, data deletion và merge main cần approval riêng. Approval cho các hành động này hết hiệu lực sau 15 phút.
- Q-06: audit log online giữ 365 ngày; signed export giữ 2 năm.
- Q-07: giữ concurrency mặc định 5. Đây là giới hạn mặc định của ứng dụng, **không phải xác nhận rằng subscription cho phép chạy đồng thời 5 session**. Không tự tăng lên 10 hoặc 18 nếu chưa xác minh điều khoản provider.
- Q-08: không tự động failover provider khi hết quota. Thông báo Owner và chờ quyết định.
- Q-09: FR-109 dùng công thức deterministic, không dùng LLM chỉ để tính điểm. PM Agent chỉ can thiệp khi task bị mắc kẹt.
- Q-10: Docker cần thiết cho sandbox validation. Codex CLI chưa cài trong giai đoạn này.
- Q-11: chấp thuận 9 UI node types và 11 engine kinds; cập nhật FR-113 đồng bộ.
- Q-12: database dùng dotted lowercase event naming; API/output có thể ánh xạ UPPER_SNAKE_CASE.
- Q-13: giữ `task_status` uppercase; các enum khác lowercase theo quy ước đã thống nhất.
- Q-14: agent idle không tự động chuyển sang trạng thái coffee/relax mặc định. Có thể có setting tùy chọn.
- Q-15: chỉ dùng asset CC0 mặc định. Cho phép font OFL nếu lưu đầy đủ license/NOTICE. CC-BY chỉ dùng khi Owner phê duyệt từng package.
- Nếu tài liệu gốc có giá trị hoặc quy tắc khác với nội dung trên, ghi nhận rõ xung đột và báo cáo, không âm thầm ghi đè.

## 3. D-003, D-005 đến D-020
- **D-003 Conductor:** giữ chấp thuận về định hướng, không khóa implementation cho đến khi vượt các tiêu chí PoC ở Q-20.
- **D-005 Codex:** hoãn tích hợp Codex khỏi MVP. Giữ abstraction `AgentAdapter` để mở rộng sau. Không cài Codex CLI hoặc live test trong phạm vi hiện tại.
- **D-006 OpenHands:** hoãn đến sau Phase 4. Không tích hợp OpenHands như provider thứ ba trong MVP.
- **D-008 Credential security:** ứng dụng không tự ý đọc `.credentials.json` hoặc `auth.json`. Không lưu plaintext token trong database hoặc frontend. Secret khác quản lý bằng OS keychain hoặc cơ chế phù hợp. Không mount thư mục credential của Owner vào Agent sandbox.
- **D-009 Git workflow:** mỗi task một worktree/branch riêng. Review diff trước khi merge. Không tự động merge vào `main`. Owner là người phê duyệt cuối cùng.
- **D-011 Virtual Office renderer:** chọn Phaser 3.90 qua abstraction `OfficeRenderer`. Benchmark trước khi khóa implementation. Animation và trạng thái nhân vật phải phản ánh event thực tế, không giả lập trạng thái Agent.
- **D-012 Source reuse:** được tái sử dụng code Apache-2.0 và MIT sau khi xác minh license và attribution. My Virtual Office chỉ làm tham khảo ý tưởng. Không sao chép code, prompt, layout hoặc asset của dự án AGPL. Asset theo Q-15.
- **D-013 Dynamic skill standard:** dùng `SKILL.md` làm chuẩn. Có thể mở rộng metadata bằng `skill.yaml`. Skill và Memory là hai thành phần riêng. Memory cần cơ chế quản lý và approval phù hợp.
- **D-014 Event architecture:** dùng Transactional Outbox. Một relay leader chịu trách nhiệm phát event. Redis Pub/Sub hoặc Streams cho realtime delivery. Hỗ trợ replay bằng `lastEventId`. Event naming theo Q-19. Nếu tài liệu gốc thiếu danh sách event hoặc nội dung mục 8.4, ghi nhận thiếu dữ liệu và yêu cầu Owner cung cấp. Không tự tạo danh sách event rồi tuyên bố đó là nội dung gốc đã được phê duyệt.
- **D-015 UI design system:** primary accent Purple; font Inter + JetBrains Mono; density Comfortable; Agent avatar Square; Owner avatar Round; default locale Vietnamese; secondary locale English.
- **D-016 Agent identity:** 18 Agent roles là seed data, không hardcode. Phân tách Agent Identity, Provider Identity, User Identity.
- **D-017 Attention states:** 5 trạng thái `blocked`, `working`, `done`, `idle`, `unknown`. Chỉ suy ra trạng thái từ lifecycle event và heartbeat thực tế. Không tự suy diễn khi thiếu dữ liệu.
- **D-018 Agent Rail:** chia 2 section. Cho phép ẩn/hiện. Mặc định hiển thị trên màn hình lớn.
- **D-019 Agent CLI/API:** chỉ triển khai trong Phase 3 sau Security Review. Short-lived token với least privilege. Có redaction. Ngăn nested execution không được phép.
- **D-020 herdr:** chỉ làm design reference. Không tái sử dụng Rust runtime. Không dùng screen heuristic làm nguồn dữ liệu sự thật.

## 4. Làm rõ FR-122, FR-181, FR-182, FR-183
Các nội dung dưới đây là đề xuất Acceptance Criteria từ Owner, không được tuyên bố là trích nguyên văn FR gốc. Phải đối chiếu lại với tài liệu gốc trước khi cập nhật.
- **FR-122 Workflow Deliverables (đề xuất):** Workflow Run phải có deliverable package gồm node outputs, run state, logs và liên kết tới generated artifacts. Truy xuất được theo workflow run. Artifact có metadata và nguồn tạo ra. Run thất bại phải xác định node gây lỗi. Không được đánh dấu hoàn thành nếu required nodes chưa hoàn thành. **Cần hỏi tiếp Owner:** artifact nào bắt buộc? Có yêu cầu export/download không, hay chỉ lưu và truy xuất trong workspace? Không tự quyết định hai điểm này.
- **FR-181 Incident Dashboard (đề xuất):** incident là sự kiện vận hành bất thường ảnh hưởng task, agent, workflow hoặc infrastructure. Bao gồm execution error, timeout, provider rate limit, sandbox failure, lost connection/heartbeat, approval timeout. Mỗi incident có ID, type, severity, timestamp, target, status. Không tự suy diễn incident khi không có bằng chứng. Cần quy định grouping incident lặp lại. **Cần Owner xác nhận:** approval chờ lâu là incident hay chỉ là operational alert.
- **FR-182 Progress Calculation (đề xuất):** `Progress = 100 × (Tổng trọng số task đã hoàn thành / Tổng trọng số task trong baseline)`. Task `completed` tính hoàn thành. `running` và `blocked` không tính. `cancelled` cần quy tắc riêng. MVP dùng trọng số bằng nhau nếu chưa có hệ thống estimate đáng tin cậy. **Cần Owner xác nhận:** cách xử lý cancelled task và có chấp thuận equal-weight không.
- **FR-183 Performance Metrics (đề xuất):** không đồng nhất Performance với Progress. Dashboard gồm lead time, execution time, success rate, retry rate, throughput, token/cost nếu provider cung cấp dữ liệu. Breakdown theo Agent, Task, Workflow và thời gian. **Cần Owner xác nhận:** dashboard nhiều metric hay một KPI tổng hợp; nếu KPI tổng hợp thì cần công thức và trọng số được phê duyệt.

## 5. Phạm vi được phép thực hiện trên máy của Owner
- **5.1 Claude Team và ChatGPT/Codex:** không mặc định subscription cho phép 5, 10 hoặc 18 session đồng thời. Không chia sẻ credential. Không dùng một tài khoản cá nhân làm backend multi-user trái điều khoản. Phải kiểm tra usage limit, điều khoản và điều kiện API. Nếu cần production nhiều Agent, phải có provider contract/API billing hợp lệ. Không tuyên bố giới hạn concurrency cụ thể khi chưa có nguồn xác nhận.
- **5.2 Codex CLI:** chưa cho phép cài đặt. Không chạy `npm i -g @openai/codex`.
- **5.3 Docker Desktop:** Owner cho phép bắt đầu kiểm tra Docker Desktop thủ công để xác thực sandbox. Quy trình: Owner đóng ứng dụng nặng nếu cần; Owner tự mở Docker Desktop; chờ Docker Engine sẵn sàng; chạy `docker version`; chạy `docker info`; ghi RAM trước và sau khi Docker hoạt động; nếu máy thiếu RAM hoặc mất ổn định thì dừng Docker Desktop. Không tự động khởi động Docker cùng ứng dụng. Không thay đổi WSL configuration hoặc chạy `wsl --shutdown` khi chưa có phê duyệt riêng.
- **5.4 Drizzle PoC:** được thực hiện trong thư mục scratch, không ảnh hưởng production schema. Phạm vi: kiểm tra Drizzle stable, ưu tiên pin `0.45.4` nếu xác minh được tồn tại và tương thích; test partition `agent_events`; partial unique index; circular foreign keys; `drizzle-kit generate --custom`; chạy migration hai lần; kiểm tra schema drift; kiểm tra idempotency. Nếu thất bại, báo cáo và đưa lựa chọn. Không tự đổi ORM.
- **5.5 Dynamic Skill/Resume PoC:** được thực hiện giới hạn trong scratch. Test bắt buộc: SK-01 sửa Skill và task mới nhận nội dung mới; SK-02 không cần restart/recompile; SK-03 resume sau khi Skill/System Prompt đổi phải phát hiện hash đổi và refresh context; SK-04 Agent sửa overlay bằng host Bash phải tạo audit log và tạm dừng Agent; SK-05 Owner xác nhận thì Agent mới được tiếp tục. Điều kiện: không sửa Skill production, không gọi external API, không dùng secret thật, hạn chế tiêu thụ quota, báo cáo rõ giới hạn phân biệt Owner và Agent khi dùng chung Windows OS user.
- **5.6 Test runbooks:** không tự động chạy toàn bộ 27 runbook. Hiện chỉ cho phép Drizzle PoC, Dynamic Skill/Resume PoC, và Docker sandbox validation sau khi Owner khởi động Docker và kiểm tra tài nguyên. Test stress, destructive, liên quan credential, provider quota hoặc production phải trình Owner phê duyệt riêng.

## 6. Git
Thông tin cũ về commit `1aec2b3` và trạng thái clean không phải bằng chứng trạng thái hiện tại. Trước khi commit chạy `git status --short`, `git diff --check`, `git diff --stat`, `git status --porcelain=v1`. Phân loại file modified/untracked, kiểm tra diff thực, quét secret/`.env`/OAuth/token/credential/log nhạy cảm/build artifact, đối chiếu với Decision Register và ADR, báo cáo file nào được commit. Không commit khi chưa kiểm tra diff. Nếu thay đổi hợp lệ, chia hai commit: (1) `docs(phase0): record owner decisions and architecture ADRs`; (2) `docs(phase0): update validation, security gates, and acceptance criteria`. Không đưa secret vào Git. Không push khi chưa được Owner cho phép.

## 7. Nguyên tắc cập nhật hồ sơ
Không ghi nhận đề xuất là quyết định đã phê duyệt nếu Owner chưa xác nhận. Không biến PROPOSED AC thành APPROVED AC. Không tự điền thông tin thiếu bằng suy đoán. Mỗi thay đổi có traceability tới Q/D/FR. ADR phản ánh đúng `PENDING_OWNER_APPROVAL`, `APPROVED` hoặc `VERIFIED`. Không đánh dấu Verified khi chưa có bằng chứng kiểm thử. Không xóa lịch sử quyết định hoặc ghi đè dữ liệu audit.

## 8. Security gates và phạm vi phase
- Không cho Agent thực thi arbitrary code trên Windows host khi chưa có sandbox đạt chuẩn.
- Không mount host credential directory hoặc Docker socket vào Agent.
- Nếu sandbox không hoạt động, fail closed.
- Không đưa production-capable Agent execution vào trước khi authentication, authorization, permission model, credential isolation, audit, sandbox và emergency stop hoàn thành.
- Phase 2 chỉ được xây provider adapter ở mock/sandbox mode nếu các security gate chưa đạt.
- Không tuyên bố 18 Agent concurrent đã được kiểm thử.
- Không giả lập Agent hoạt động hoặc workflow thành công khi không có execution evidence.
- **Phase 0 vẫn đang mở. Phase 1 chưa được Owner cho phép bắt đầu.**

## 9. Cấm tự ý
Cài Codex CLI; chạy toàn bộ 27 runbook; đổi production schema; sửa cấu hình WSL/Docker ngoài phạm vi; dùng credential thật trong PoC; tăng concurrency; chuyển Phase 0 thành Completed; bắt đầu Phase 1; commit hoặc push khi chưa có quyền. Nếu thiếu dữ liệu để quyết định, dừng đúng phần đó, ghi câu hỏi cần Owner xác nhận và tiếp tục các phần độc lập đã được cấp phép. Mục tiêu: hoàn thiện Phase 0 bằng bằng chứng thực, tài liệu nhất quán và security gate rõ ràng, không phải hoàn thành hình thức bằng cách đánh dấu tất cả PASS.

## 10. Định dạng báo cáo yêu cầu
A Decision Register; B FR Acceptance Criteria; C Documentation Changes; D PoC Results (PASS/FAIL/BLOCKED/NOT RUN kèm bằng chứng); E Git Status; F Remaining Risks; G Phase 0 Exit Criteria (bảng: requirement, expected result, evidence, status, remaining action, owner approval required).
