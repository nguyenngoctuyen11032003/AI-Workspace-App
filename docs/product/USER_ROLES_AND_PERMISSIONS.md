# USER ROLES AND PERMISSIONS — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30

> Quy ước: [Owner] = yêu cầu gốc; [Giả định] = BA suy diễn; [Đề xuất] = cần Owner chốt.

## 1. Mục đích
Định nghĩa (a) vai trò con người và quyền của họ trong chế độ single-user desktop (MVP) và multi-user (tương lai); (b) permission level của agent; (c) approval policy — ai phải duyệt cái gì, khi nào. Đây là cơ sở cho FR-215, FR-198/199 và các AC nhóm F.

## 2. Phạm vi
- Trong: vai trò human, permission level agent, approval policy, ma trận quyền, luật bất biến.
- Ngoài: cơ chế xác thực người dùng (MVP không có login app vì single-user), SSO, audit chi tiết (xem `docs/security/`).

## 3. Vai trò con người (Human Roles)

### 3.1 Chế độ MVP — single-user desktop [Owner]
Chỉ có **một** người dùng, giữ đồng thời ba "mũ": **Owner / Product Owner / Director**. Không có đăng nhập ứng dụng; quyền = toàn quyền.

### 3.2 Chế độ multi-user — tương lai [Giả định]
Owner chưa nêu chi tiết; BA đề xuất ba vai trò để thiết kế data model sẵn sàng (role field), không triển khai UI phân quyền ở MVP:

| Vai trò | Mô tả | Phạm vi |
|---|---|---|
| **Owner / Director** | Chủ công ty ảo. Toàn quyền: company, department, agent, provider, skill, settings, security, emergency stop, mọi approval. | Toàn workspace |
| **Project Owner** | Chủ một hoặc nhiều project. Tạo project, duyệt BA/UI/Acceptance của project mình, assign task, pause/resume agent trong project. Không sửa provider, security settings, không xóa agent dùng chung. | Theo project |
| **Viewer** | Chỉ xem: dashboard, Virtual Office, task board, activity. Không thao tác, không approve, không chat với agent (hoặc chat read-only). | Theo workspace/project |

### 3.3 Ma trận quyền (Human) — [Đề xuất]

| Hành động | Owner | Project Owner | Viewer |
|---|---|---|---|
| Tạo/sửa/xóa company, workspace | ✔ | ✘ | ✘ |
| Tạo/sửa/xóa department, org chart | ✔ | ✘ | ✘ |
| Tạo/sửa/xóa agent (Agent Builder) | ✔ | ✘ (chỉ xem) | ✘ |
| Pause/Resume/Disable agent | ✔ | ✔ (agent trong project của mình, chỉ Pause/Resume) | ✘ |
| Chat với agent | ✔ | ✔ | ✘ |
| Kết nối/ngắt provider, xem usage | ✔ | xem | ✘ |
| Tạo/sửa skill, Teach Agent | ✔ | ✔ (skill thuộc project) [Giả định] | ✘ |
| Tạo project từ prompt | ✔ | ✔ | ✘ |
| Assign/reassign task (MANUAL) | ✔ | ✔ (project mình) | ✘ |
| Approve/Reject tại Approval Center | ✔ | ✔ (project mình) | ✘ |
| Human Acceptance (stage 8) | ✔ | ✔ (project mình) | ✘ |
| Workflow Builder, lưu template | ✔ | ✔ | ✘ |
| Thay đổi approval policy | ✔ | ✘ | ✘ |
| Thay đổi security settings (allow/deny, network, limits) | ✔ | ✘ | ✘ |
| Emergency stop toàn cục | ✔ | ✔ [Đề xuất: ai cũng được dừng khẩn cấp, chỉ Owner resume] | ✘ |
| Xem audit log | ✔ | ✔ (project mình) | ✘ |
| Xem dashboard, Virtual Office, Activity | ✔ | ✔ | ✔ |
| Cost Management, đặt ngân sách | ✔ | xem | ✘ |
| Settings (concurrency, ngôn ngữ...) | ✔ | ✘ | ✘ |

## 4. Permission level của Agent

### 4.1 Nguyên tắc [Owner]
- Agent có trường `permission level` trong Agent Builder.
- Agent không sửa code của agent khác nếu không được phép.
- Allow/deny list, hạn chế FS/network, credential isolation áp dụng cho mọi agent.
- Không chia sẻ credential giữa agent.

### 4.2 Các cấp permission — [Đề xuất] (Owner chưa định nghĩa số cấp)

| Level | Tên | Được phép | Không được phép |
|---|---|---|---|
| L0 | **Read-only** | Đọc file trong workspace task; đọc tài liệu; gửi message; tạo báo cáo. | Ghi file, chạy lệnh, network. |
| L1 | **Write-own** | L0 + ghi file thuộc file ownership của task mình; chạy lệnh trong allow list "safe" (lint, test unit) trong sandbox. | Ghi ngoài ownership; lệnh ngoài allow list; network ngoài allow list. |
| L2 | **Developer** | L1 + tạo branch/commit trong worktree của mình; chạy build/test tích hợp trong sandbox; tạo subtask; handoff. | Merge vào nhánh chính; deploy; sửa file ownership của task khác (trừ khi được cấp quyền tạm thời). |
| L3 | **Lead / Reviewer** | L2 + review và yêu cầu sửa; phê duyệt merge nội bộ (code review); cấp quyền tạm thời cho agent khác sửa file trong phạm vi task mình quản lý; tổ chức meeting. | Deploy; thay đổi cấu hình hệ thống; approve thay con người tại approval gate. |
| L4 | **Operator (DevOps/Release)** | L3 + build artifact; chạy preview deployment (non-prod) trong sandbox; quản lý container preview. | Deploy production (luôn cần human approval); thay đổi allow/deny list. |
| L5 | **PM / Orchestrator** | L3 + tạo task, AUTO assignment, handover, thành lập team, điều phối workflow. | Chạy code/lệnh (PM không sửa code) [Đề xuất]; deploy. |

Mapping mặc định vai trò → level [Đề xuất]: PM → L5; BA, UX Research, Documentation → L1 (ghi tài liệu); Architect, UI/UX → L1/L2; FE, BE, DB&Infra, R&D, Integration, Performance, Test Automation → L2; QA, Security, Code Reviewer/Product Evaluator → L3 (đọc toàn repo, ghi báo cáo/bug, không sửa code sản phẩm); DevSecOps, Release Manager → L4.

### 4.3 Quyền được cấu hình chi tiết trong Agent Builder [Owner]
- `tools` (danh sách tool được dùng) và `allowed tools` trong skill (FR-067): giao của hai tập.
- `working directory`: giới hạn FS.
- `max token budget`, `max execution time`, `max retry`: giới hạn tài nguyên.
- `memory config`: phạm vi bộ nhớ.

### 4.4 File ownership [Owner]
- Mỗi task khai báo tập file/thư mục nó sở hữu. Agent chỉ ghi trong tập này.
- Xung đột ownership (hai task cùng file) phải được PM giải quyết trước khi task READY [Đề xuất].
- Cấp quyền tạm thời (grant) do L3+ hoặc Owner cấp, có thời hạn, ghi audit.

### 4.5 Luật bất biến cho agent (không cấu hình tắt được) [Owner]
1. Không hỏi/nhận mật khẩu của người dùng.
2. Không đọc/trích xuất credential/token của CLI.
3. Không chia sẻ credential giữa agent.
4. Không deploy production tự động.
5. Không ghi ngoài workspace được phép.
6. Mọi lệnh qua allow/deny list.
7. Mọi hành động ghi audit log.
8. Emergency stop dừng tất cả, bất kể level.

## 5. Approval Policies

### 5.1 Yêu cầu [Owner]
- Approval Center với approval policy cấu hình được.
- Template SDLC: BA (stage 2) và UI/UX (stage 4) có approval; Human Acceptance (stage 8) bắt buộc; không auto deploy production.
- Reject → revision task có mục tiêu.

### 5.2 Mô hình policy — [Đề xuất]

Mỗi policy gồm: `trigger` (stage hoàn thành / hành động cụ thể / ngưỡng), `scope` (toàn cục, project, department, agent), `approver` (Owner / Project Owner), `required` (bắt buộc hay tùy chọn), `can_disable` (có tắt được không), `auto_decision` (không có — human luôn quyết định).

### 5.3 Policy mặc định — [Đề xuất]

| ID | Trigger | Required | Có tắt được? | Approver |
|---|---|---|---|---|
| POL-01 | Stage 2 BA hoàn thành | Có | Có (Owner) | Owner/PO |
| POL-02 | Stage 4 UI/UX hoàn thành | Có | Có (Owner) | Owner/PO |
| POL-03 | Stage 8 Human Acceptance | Có | **Không** | Owner/PO |
| POL-04 | Deployment production | Có | **Không** | Owner |
| POL-05 | Stage 3 System Design hoàn thành | Tùy chọn (mặc định tắt) | Có | Owner/PO |
| POL-06 | Lệnh ngoài allow list nhưng không trong deny list | Có | Có | Owner |
| POL-07 | Truy cập network ngoài allow list | Có | Có | Owner |
| POL-08 | Vượt ngân sách token/cost của task/phòng ban | Có | Có | Owner |
| POL-09 | Agent đề xuất thay đổi ngoài phạm vi task (propose change) | Có | Có | Owner/PO hoặc Lead agent L3 (Owner chọn) |
| POL-10 | Xóa file/thư mục lớn (> N file) hoặc thao tác git phá hủy (force push, reset hard) | Có | Có | Owner |
| POL-11 | Merge vào nhánh chính của project | Tùy chọn (mặc định: code review agent L3 đủ) | Có | Lead agent / Owner |
| POL-12 | Cài đặt dependency mới | Tùy chọn (mặc định tắt) | Có | Owner |

### 5.4 Hành vi khi reject [Owner]
- Bắt buộc feedback (văn bản; tùy chọn screenshot markup) [bắt buộc feedback là Đề xuất].
- Hệ thống tạo revision task targeted; tăng version; không chạy lại toàn bộ.

### 5.5 Approval timeout — [Đề xuất]
Không timeout mặc định; nhắc nhở sau 24 giờ (P2).

## 6. Quan hệ vai trò ↔ phòng ban ↔ cấp quản lý

- 18 vai trò mặc định [Owner] ánh xạ vào 12 phòng ban mặc định [Giả định mapping]: PM → Management; BA, UX Research → Business Analysis; Architect, R&D, Integration, Performance → System Architecture; UI/UX → UI/UX Design; FE → Frontend; BE → Backend; DB&Infra → Database; QA, Test Automation → QA; Security, DevSecOps → Cybersecurity; Release Manager → DevOps; Code Reviewer → Code Review; Product Evaluator → Product Evaluation; Documentation → Management hoặc Technical Writing (nếu tạo).
- Reporting chain: agent → department head → PM/Director (Owner). Management levels (FR-006) cho phép thêm cấp.

## 7. Dependencies
- `FUNCTIONAL_REQUIREMENTS.md` FR-039, FR-198, FR-199, FR-209…FR-215, FR-235.
- `docs/security/` cho cơ chế thực thi permission tại sandbox.

## 8. Rủi ro
- Permission quá chặt → agent thất bại nhiều (BLOCKED) → năng suất thấp; cần log lý do chặn và nút "cấp quyền tạm thời" nhanh.
- Permission quá lỏng → rủi ro bảo mật.
- Multi-user chưa thiết kế xác thực → không nên hứa trong MVP.

## 9. Vấn đề chưa giải quyết / mâu thuẫn
| ID | Vấn đề |
|---|---|
| OI-ROLE-01 | Số cấp permission và tên gọi (L0–L5 là Đề xuất). |
| OI-ROLE-02 | PM có được chạy code không? (Đề xuất: không) |
| OI-ROLE-03 | Lead agent (L3) có được "approve" thay con người ở POL-11 (merge) không — Owner nói "no editing others' code without permission" nhưng không nói ai cấp permission. |
| OI-ROLE-04 | Multi-user: có cần ngay data model `role` trong MVP? |
| OI-ROLE-05 | Code Reviewer/Product Evaluator 1 hay 2 vai trò. |
| OI-ROLE-06 | Host vs sandbox: nếu CLI chạy trên host, permission level không thể thực thi hoàn toàn bằng container — phải dựa vào permission mode/hook của CLI. |

## 10. Cần người dùng phê duyệt
| # | Hạng mục |
|---|---|
| AP-ROLE-01 | Ba vai trò human và ma trận §3.3 |
| AP-ROLE-02 | 6 cấp permission agent §4.2 và mapping vai trò mặc định |
| AP-ROLE-03 | 8 luật bất biến §4.5 |
| AP-ROLE-04 | 12 policy mặc định §5.3, đặc biệt POL-05, POL-11, POL-12 |
| AP-ROLE-05 | Bắt buộc feedback khi reject |
| AP-ROLE-06 | Mapping vai trò ↔ phòng ban §6 |
