# ADR-0007 — Mô hình sandbox hai tầng (Tier 1 host + hook, Tier 2 container hóa tùy chọn)

## 1. Metadata

- **ID:** ADR-0007
- **Ngày:** 2026-10-30
- **Decision ID:** D-007 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED_CONDITIONAL
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-1; OWR-2 không nâng trạng thái)
- **Nguồn:** SECURITY.md §10 (R1–R10), security/SANDBOX_DESIGN.md, security/THREAT_MODEL.md, PROJECT_RISKS.md

## 2. Context

- Yêu cầu gốc «không chạy code AI trực tiếp trên host» mâu thuẫn với việc Claude Code/Codex chạy trên host bằng đăng nhập OAuth gắn user Windows (DECISION_LOG mâu thuẫn 1).
- Claude Code trên native Windows chưa có OS-level sandbox; token OAuth file quyền 644/ACL đọc được bởi mọi process cùng user.

## 3. Problem

- Dung hòa yêu cầu gốc với thực tế kỹ thuật và nêu rõ rủi ro còn lại để chủ sở hữu ký nhận.
- Docker daemon chưa chạy lúc khảo sát; QA chỉ thấy Docker CLI 29.8.1, nên cgroup/ECI/rootless chưa xác minh.

## 4. Options considered

### Phương án A — Tier 1 (CLI trên host quyền thường + hook + Docker cho build/test/run) và Tier 2 tùy chọn (container hóa hoàn toàn) [ĐỀ XUẤT]

Chính sách CLI là lớp chính sách, không phải ranh giới bảo mật; lệnh `executes-code` bọc `aiw-exec` chạy trong Docker, fail-closed khi daemon tắt. Container per project (warm), ephemeral cho preview/custom (CF-016).
- **Lý do chọn đề xuất:** Tier 1 là cách khả thi duy nhất giữ được đăng nhập CLI; hook + Docker thu hẹp bề mặt.

### Phương án B — Chạy mọi thứ trên host [Bị loại]

Đơn giản.
- **Lý do loại:** Vi phạm yêu cầu gốc; không có ranh giới nào ngoài hook.

### Phương án C — Container hóa hoàn toàn làm mặc định [Bị loại]

Ranh giới mạnh.
- **Lý do loại:** Phải đưa credential vào container (đụng D-008), OpenHands/Docker thành bắt buộc; giữ làm hướng dự phòng nếu chủ sở hữu không chấp nhận R1–R6.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Mô hình hai tầng như trên; chủ sở hữu ký xác nhận R1, R2, R3, R5, R6 trước Phase 4.
- Nếu không chấp nhận, chuyển Tier 2 làm mặc định (ADR-0006).
- Giữ policy engine, hook, `assertSafeSpec` và test SB-00..SB-17.

## 6. Rationale

- Không có per-agent OS user vì OAuth gắn user chính.
- Hook rewrite + Docker thu hẹp bề mặt cho lệnh thực thi code.
- Fail-closed: không có container thì không chạy lệnh thực thi.

## 7. Consequences

**Tích cực**
- Giữ được đăng nhập CLI sẵn có.
- Rủi ro được nêu tường minh để ký nhận.

**Tiêu cực**
- R1 (không OS-level sandbox) và R2 (OAuth file đọc được) được thừa nhận, không loại bỏ.
- Cách ly giữa agent chỉ ở mức logical (R4).
- Egress dựa proxy allowlist, không phải firewall kernel (R6).

**Trung tính**
- Phụ thuộc Docker Desktop (VM root-equivalent, R5).
- Hiệu quả thực tế chỉ biết sau test SB-00..SB-17 trên máy có daemon.

## 8. Security implications

- Đây là ADR bảo mật trung tâm: Tier 1 không phải ranh giới; obfuscation chưa biết hoặc bug parser có thể vượt hook (R1).
- Token OAuth đọc được bởi process cùng user (R2); app không thể sửa quyền file do CLI quản.
- Docker Desktop VM root-equivalent (R5): người dùng phải biết để quyết định.

## 9. Cost and operational implications

- Docker Desktop/WSL2 tốn RAM, đĩa; cần cấu hình `.wslconfig` (chưa kiểm).
- Container warm theo project: chi phí tài nguyên cố định mỗi project.
- Duy trì policy engine, parser và bộ test SB là chi phí thường trực.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Bảng R1–R10 đã đọc; top 5 cần ký: R1, R2, R3, R5, R6 | `docs/SECURITY.md §10` | Đã xác minh |
| Docker CLI 29.8.1 có mặt | `docs/QA_REVIEW_PHASE0.md (mục 4)` | Đã xác minh |
| Daemon, cgroup, ECI, rootless hoạt động như thiết kế | `docs/security/SANDBOX_DESIGN.md §14` | Chưa kiểm chứng |
| Hook chặn/viết lại lệnh đủ hiệu quả | `docs/security/SANDBOX_DESIGN.md §11 (SB-00..SB-17)` | Chưa kiểm chứng |
| Mâu thuẫn host vs sandbox được ghi nhận | `DECISION_LOG.md mâu thuẫn 1, 3` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Chưa chạy bất kỳ kiểm thử sandbox nào; daemon chưa chạy, nhiều rủi ro chỉ được thừa nhận bằng văn bản.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Ký xác nhận từng rủi ro còn lại R1, R2, R3, R5, R6 (hoặc yêu cầu chuyển Tier 2 mặc định).
- Chấp nhận chính sách CLI không phải ranh giới bảo mật.
- **Khả năng đảo ngược:** Khó: kiến trúc cấp hệ thống; đổi sau Phase 4 ảnh hưởng adapter, credential và UX.
- **Bị chặn trong lúc chờ duyệt:** Phase 4 và mọi tính năng chạy lệnh thực thi code của agent.

## 12. Reversal conditions

- Chủ sở hữu không ký R1, R2, R3, R5, R6: chuyển Tier 2 mặc định và xem lại ADR-0008.
- Anthropic phát hành sandbox native Windows ổn định: nâng Tier 1 thành ranh giới thật.
- Test SB-00..SB-17 phát hiện vượt ranh giới không vá được.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-1 mục 8.1, 8.3; OWR-2 §8, §5.3, §2 Q-10 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Điều kiện OWR-1 giữ nguyên: Phase 0 đến 1 không cho agent chạy thật; checksum và audit log cho rủi ro ghi overlay; fail-closed khi Docker tắt; SB-00..SB-17 phải qua trên máy có Docker.
- Security gates OWR-2 §8: không cho Agent thực thi arbitrary code trên Windows host khi chưa có sandbox đạt chuẩn; không mount host credential directory hoặc Docker socket vào Agent; sandbox không hoạt động thì fail closed; không đưa production-capable Agent execution vào trước khi authentication, authorization, permission model, credential isolation, audit, sandbox và emergency stop hoàn thành.
- Q-10: Docker cần cho sandbox validation. §5.3: Owner tự mở Docker Desktop, chạy `docker version`, `docker info`, ghi RAM trước và sau; không tự khởi động Docker cùng ứng dụng; không đổi WSL hay `wsl --shutdown` khi chưa phê duyệt riêng. Docker daemon chưa được ghi nhận là chạy; sandbox chưa được xác thực.
- Q-01: chỉ là xác nhận có điều kiện của Owner (các ô checklist chưa được Owner ký). Không tuyên bố 18 Agent concurrent đã kiểm thử.
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

## 14. Cập nhật OWR-3 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (OWR-3) (A.10, A.11, E). Các mục 1 đến 13 ở trên giữ làm lịch sử; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**. Việc đồng bộ tài liệu khác do agent khác thực hiện, xem `docs/DOC_SYNC_LOG_OWR3_A.md` và `docs/DOC_SYNC_LOG_OWR3_B.md`; ADR này không khẳng định đã đồng bộ.

- Decision status giữ **APPROVED_CONDITIONAL**. Không nâng trạng thái.
- **Điều kiện chặt hơn từ OWR-3 A.10 (thêm vào điều kiện OWR-1 và OWR-2 §8):** không cho Agent chạy arbitrary code trên Windows host; mọi code do Agent sinh phải thực thi trong sandbox cô lập; Docker chỉ cho build/test/run là **không đủ** nếu Agent vẫn có quyền thực thi trên host; sandbox không hoạt động thì **fail closed**. Tier 2 và OpenHands hoãn khỏi MVP.
- **Xung đột X-13** với thiết kế Tier 1 trong các mục 5 đến 12 (CLI/Agent chạy trên host với chính sách và hook, chỉ build/test/run trong Docker). Đề xuất gốc không được viết lại; phần Tier 1 đó **không còn đủ** nếu Agent vẫn thực thi trên host.
- Câu hỏi còn lại (PENDING_OWNER_CLARIFICATION, agent không tự điền):
  1. Edit/Write file trên host bên trong worktree của task có được tính là cho phép không?
  2. Bản thân tiến trình runtime của Agent (Claude Agent SDK hoặc CLI trên host, giữ phiên đăng nhập) có bị coi là "Agent thực thi trên host" không?
  3. Runtime sandbox nào cung cấp thực thi cô lập khi OpenHands hoãn? Đường suy ra là executor dựa trên Docker do dự án tự xây, nhưng cần Docker, mà Owner **chưa cho phép** mở Docker lúc này (OWR-3 E).
  4. Hệ quả: Docker (hoặc sandbox tương đương) là **điều kiện bắt buộc** cho mọi lần thực thi Agent thật; khi chưa có thì không có thực thi Agent thật (chỉ mock/dry-run).
- A.11: Phase 2 chỉ mock hoặc sandbox khi security gate chưa đạt; không production-capable execution.
- OWR-3 E: không mở Docker Desktop, không chạy VAL-DOCKER-01..09, không chạy SB-00..SB-17 lúc này (NOT RUN). Hẹp hơn OWR-2 §5.6.
- **Validation evidence (trung thực):** mức giữ nguyên (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
