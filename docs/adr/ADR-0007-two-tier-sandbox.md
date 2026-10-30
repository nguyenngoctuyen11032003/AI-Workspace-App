# ADR-0007 — Mô hình sandbox hai tầng (Tier 1 host + hook, Tier 2 container hóa tùy chọn)

## 1. Metadata

- **ID:** ADR-0007
- **Ngày:** 2026-10-30
- **Decision ID:** D-007 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
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
