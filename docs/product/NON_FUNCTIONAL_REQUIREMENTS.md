# NON-FUNCTIONAL REQUIREMENTS — AI WORKSPACE

**Phiên bản:** 0.1 (Phase 0) · **Tác giả:** Agent 12 — BA · **Trạng thái:** Draft · **Ngày:** 2026-10-30

> Quy ước: [Owner] = giá trị do Owner đưa ra; **Đề xuất** = giá trị do BA đề xuất vì Owner chưa nêu, cần Owner chốt. Pri theo MoSCoW; Phase MVP/P2.
> Dải mã: 001–020 Performance · 021–040 Reliability · 041–070 Security · 071–085 Usability · 086–100 Observability · 101–115 Maintainability · 116–130 Compatibility/Portability · 131–140 Data.

## 1. Mục đích
Xác định các yêu cầu chất lượng (NFR) đo lường được để kiến trúc, QA và Security có mục tiêu rõ ràng, đặc biệt với kịch bản tối đa 18 agent chạy đồng thời.

## 2. Phạm vi
Toàn hệ thống AI WORKSPACE chạy local trên Windows 11, single-user. Không bao gồm NFR của provider bên ngoài (chỉ cách hệ thống ứng xử với chúng).

## 3. Kịch bản tải tham chiếu (Load Profiles) — Đề xuất

| Profile | Mô tả |
|---|---|
| L0 — Idle | 0 agent chạy; UI mở, Virtual Office render. |
| L1 — Default | 5 agent chạy đồng thời [Owner mặc định], 1 project, ~50 task. |
| L2 — Max | 18 agent chạy đồng thời [Owner tối đa], 2 project, ~200 task, ~20 event/giây đỉnh. |
| L3 — Recovery | Restart app khi L2 đang chạy. |

## 4. Danh sách NFR

### 4.1 Performance (NFR-001 … NFR-020)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-001 | Số agent chạy đồng thời tối đa được hỗ trợ | 18 | M | MVP | [Owner] |
| NFR-002 | Concurrency mặc định | 5 | M | MVP | [Owner] |
| NFR-003 | Độ trễ từ event phát sinh đến hiển thị trên UI/Virtual Office | ≤ 500 ms p95 tại L2 (Đề xuất) | M | MVP | Đề xuất |
| NFR-004 | Frame rate Virtual Office với 18 nhân vật | ≥ 30 FPS (Đề xuất; 60 FPS mục tiêu) trên máy tham chiếu | S | MVP | Đề xuất |
| NFR-005 | Thời gian mở màn hình (dashboard, task board) | ≤ 1 s p95 với 200 task (Đề xuất) | S | MVP | Đề xuất |
| NFR-006 | Thông lượng event store | ≥ 100 event/giây ghi bền (Đề xuất) | M | MVP | Đề xuất |
| NFR-007 | Thời gian từ CancelTask/Emergency stop tới khi mọi tiến trình dừng | ≤ 5 s (Đề xuất) | M | MVP | Đề xuất |
| NFR-008 | Thời gian khởi động app tới trạng thái sẵn sàng (đã khôi phục state) | ≤ 10 s với 10k event (Đề xuất) | S | MVP | Đề xuất |
| NFR-009 | Overhead CPU/RAM của AI WORKSPACE (không tính container và CLI) tại L2 | ≤ 2 CPU core, ≤ 2 GB RAM (Đề xuất) | S | MVP | Đề xuất |
| NFR-010 | Thời gian tạo sandbox container cho task | ≤ 10 s p95 (Đề xuất, image đã cache) | S | MVP | Đề xuất |
| NFR-011 | Streaming output provider tới UI | ≤ 300 ms p95 từ CLI stdout tới màn hình (Đề xuất) | S | MVP | Đề xuất |
| NFR-012 | Hàng đợi task khi vượt concurrency không làm UI treo | UI responsive < 100 ms input latency (Đề xuất) | M | MVP | Đề xuất |

### 4.2 Reliability & Availability (NFR-021 … NFR-040)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-021 | Không mất state khi app restart/crash | 100% workflow/task/approval/event được khôi phục (L3) | M | MVP | [Owner] |
| NFR-022 | Checkpoint được ghi bền trước khi báo bước hoàn tất | Durable write (fsync) trước khi phát event COMPLETED (Đề xuất) | M | MVP | [Owner] checkpoint |
| NFR-023 | Retry tự động với backoff | Mặc định max 3, exponential backoff 2s/4s/8s (Đề xuất) | M | MVP | [Owner] retry; giá trị Đề xuất |
| NFR-024 | Phát hiện tiến trình agent chết | Heartbeat ≤ 30 s; phát hiện ≤ 60 s (Đề xuất) | M | MVP | Đề xuất |
| NFR-025 | Tính toàn vẹn event store | Append-only; không mất/không trùng event (idempotent id) | M | MVP | Đề xuất |
| NFR-026 | Tỷ lệ recovery thành công trong test L3 | 100% trên 20 lần chạy (Đề xuất) | M | MVP | Đề xuất |
| NFR-027 | Mất kết nối provider không làm mất task | Task → BLOCKED/WAITING, tự nối lại khi provider trở lại | M | MVP | [Owner] reconnect |
| NFR-028 | Dữ liệu dashboard nhất quán với event store | Sai lệch 0 sau khi rebuild read-model | S | MVP | [Owner] real data |
| NFR-029 | Không có trạng thái "treo" vô hạn | Mọi RUNNING có max execution time; vượt → FAILED/BLOCKED | M | MVP | [Owner] max exec time |

### 4.3 Security (NFR-041 … NFR-070)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-041 | Không yêu cầu mật khẩu provider | 0 màn hình/ô nhập mật khẩu | M | MVP | [Owner] |
| NFR-042 | Không trích xuất token từ CLI | Code review + test: không đọc file credential CLI | M | MVP | [Owner] |
| NFR-043 | Không lưu OAuth/API key plaintext trong DB | 0 bản ghi; scan DB sau test | M | MVP | [Owner] |
| NFR-044 | Không gửi credential tới frontend | 0 trường credential trong payload IPC/HTTP (test tự động) | M | MVP | [Owner] |
| NFR-045 | Secret (nếu có) lưu bằng OS credential manager | 100% | S | MVP | [Owner] |
| NFR-046 | Không credential trong log | Scan log với pattern secret: 0 kết quả | M | MVP | [Owner] |
| NFR-047 | Không chia sẻ credential giữa agent | Mỗi agent chỉ gọi qua adapter; sandbox không mount credential | M | MVP | [Owner] |
| NFR-048 | Sandbox cô lập FS | Agent không đọc/ghi ngoài workspace (test thử escape: 0 thành công) | M | MVP | [Owner] |
| NFR-049 | Sandbox cô lập network | Mặc định deny; chỉ allow list (Đề xuất mặc định) | M | MVP | [Owner]; mặc định Đề xuất |
| NFR-050 | Giới hạn tài nguyên sandbox | Mặc định 2 CPU, 4 GB RAM, 10 GB disk, 30 phút/task (Đề xuất) | M | MVP | [Owner] limits; giá trị Đề xuất |
| NFR-051 | Allow/deny list lệnh được thực thi trước mọi lệnh | 100% lệnh qua kiểm tra; deny → chặn + audit | M | MVP | [Owner] |
| NFR-052 | Audit log đầy đủ và không sửa được | Append-only; mọi lệnh/thay đổi file/approval/stop | M | MVP | [Owner] |
| NFR-053 | Emergency stop luôn khả dụng | Nút hiển thị ở mọi màn hình; hoạt động cả khi UI đang tải | M | MVP | [Owner] |
| NFR-054 | Phân quyền agent theo permission level được thực thi tại runtime | 100% tool/lệnh kiểm tra permission | M | MVP | [Owner] |
| NFR-055 | Dữ liệu local được bảo vệ | DB file trong thư mục người dùng; tùy chọn mã hóa at-rest (Đề xuất P2) | C | P2 | Đề xuất |
| NFR-056 | Không auto deploy production | Deployment node loại production luôn cần approval, không tắt được | M | MVP | [Owner] |
| NFR-057 | Dependency/supply chain | Lock file; quét lỗ hổng định kỳ (Đề xuất) | S | P2 | Đề xuất |

### 4.4 Usability (NFR-071 … NFR-085)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-071 | Onboarding tới trạng thái provider kết nối | ≤ 10 phút cho người đã cài CLI (Đề xuất) | S | MVP | Đề xuất |
| NFR-072 | Tạo agent từ template | ≤ 2 phút (Đề xuất) | S | MVP | Đề xuất |
| NFR-073 | Trạng thái agent/task nhất quán giữa mọi màn hình | 0 sai lệch (kiểm thử) | M | MVP | [Owner] |
| NFR-074 | Premium UI: design system thống nhất, dark/light | Áp dụng cho 22 màn hình | S | P2 | [Owner] premium |
| NFR-075 | Hướng dẫn cài/login provider rõ ràng từng bước | Người dùng mới hoàn thành không cần tài liệu ngoài (Đề xuất) | M | MVP | [Owner] |
| NFR-076 | Accessibility cơ bản | Keyboard navigation cho màn hình quản trị; contrast WCAG AA (Đề xuất) | C | P2 | Đề xuất |
| NFR-077 | Ngôn ngữ | [Chưa chốt] Việt/Anh/song ngữ | S | — | OI-07 |
| NFR-078 | Thông báo lỗi có hành động | Mọi lỗi provider/sandbox kèm bước khắc phục | S | MVP | Đề xuất |

### 4.5 Observability (NFR-086 … NFR-100)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-086 | Log có cấu trúc (JSON) với correlation id (task/workflow/agent) | 100% log | M | MVP | Đề xuất |
| NFR-087 | Token/cost ghi nhận cho mọi turn provider | 100% turn; đánh dấu "ước lượng" khi CLI không cung cấp | M | MVP | [Owner] cost; ước lượng Đề xuất |
| NFR-088 | Activity Center hiển thị event realtime | ≤ 500 ms | M | MVP | [Owner] |
| NFR-089 | Metric hệ thống: concurrency đang dùng, queue length, provider status, sandbox count | Cập nhật ≤ 2 s | S | MVP | [Owner] system dashboard |
| NFR-090 | Lưu log | Mặc định 30 ngày hoặc 1 GB, cấu hình được (Đề xuất) | S | MVP | Đề xuất |
| NFR-091 | Truy vết end-to-end: từ prompt → task → session → commit → preview | Mọi artifact có link về task/event | S | MVP | Đề xuất |

### 4.6 Maintainability & Extensibility (NFR-101 … NFR-115)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-101 | Thêm provider adapter mới không sửa core | Chỉ implement interface 13 operation + đăng ký | M | MVP | [Owner] generic CLI |
| NFR-102 | Thêm node type workflow mới | Plugin/registry, không sửa engine | S | P2 | Đề xuất |
| NFR-103 | Thêm phòng ban/vai trò/phòng văn phòng bằng cấu hình | Không cần sửa code | S | MVP | [Owner] user can add |
| NFR-104 | Event schema có version | Mọi event có schemaVersion; migration khi thay đổi | M | MVP | Đề xuất |
| NFR-105 | Test coverage core (engine, adapter, security) | ≥ 80% (Đề xuất) | S | MVP | Đề xuất |
| NFR-106 | Skill theo chuẩn Agent Skills để tương thích công cụ bên ngoài | 100% skill có SKILL.md | M | MVP | [Owner] |
| NFR-107 | Tài liệu kiến trúc và ADR được cập nhật cùng code | Mỗi quyết định lớn có ADR | S | MVP | Đề xuất |

### 4.7 Compatibility / Portability (NFR-116 … NFR-130)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-116 | Hỗ trợ Windows 11 | Bắt buộc | M | MVP | [Owner] môi trường |
| NFR-117 | Hỗ trợ macOS/Linux | Đề xuất P2/P3 | C | P2 | Đề xuất |
| NFR-118 | Tương thích phiên bản CLI | Khai báo phiên bản min/max đã test; cảnh báo khi ngoài dải | M | MVP | Đề xuất |
| NFR-119 | Docker Desktop (WSL2) trên Windows | Hoạt động; cảnh báo khi thiếu | M | MVP | [Owner] sandbox |
| NFR-120 | Độ phân giải màn hình tối thiểu | 1366×768; tối ưu 1920×1080 (Đề xuất) | S | MVP | Đề xuất |

### 4.8 Data (NFR-131 … NFR-140)

| ID | Yêu cầu | Giá trị mục tiêu | Pri | Phase | Nguồn |
|---|---|---|---|---|---|
| NFR-131 | Dữ liệu lưu local, thuộc người dùng | Không upload khi không bật | M | MVP | Đề xuất |
| NFR-132 | Dung lượng event store | Hỗ trợ ≥ 1 triệu event không giảm hiệu năng quá 20% (Đề xuất) | S | P2 | Đề xuất |
| NFR-133 | Xóa agent/project là soft delete, giữ audit | 100% | S | MVP | Đề xuất |
| NFR-134 | Sao lưu/khôi phục | Export/import workspace (P2) | C | P2 | Đề xuất |

## 5. Thống kê
- Tổng NFR: **72**. Must: 40 · Should: 24 · Could: 8. Giá trị [Owner]: 18, 5, 0 credential leak, 100% recovery. Phần lớn giá trị số là **Đề xuất**.

## 6. Dependencies
- Quyết định shell (Tauri/web) ảnh hưởng NFR-003, 004, 009, 044, 045.
- Quyết định sandbox (host vs container) ảnh hưởng NFR-047…051.
- Khả năng CLI cung cấp usage ảnh hưởng NFR-087.

## 7. Rủi ro
- Rate limit provider làm NFR-001 (18 đồng thời) chỉ đạt được về mặt "slot" chứ không về thông lượng thực.
- Docker trên Windows có thể không đạt NFR-010.
- Virtual Office trên máy yếu có thể không đạt NFR-004 → cần chế độ giảm hiệu ứng.

## 8. Vấn đề chưa giải quyết / mâu thuẫn
| ID | Vấn đề |
|---|---|
| OI-NFR-01 | 18 agent đồng thời vs rate limit 1 tài khoản: Owner chấp nhận "18 slot, thông lượng tùy quota" hay yêu cầu nhiều tài khoản/provider? |
| OI-NFR-02 | Host execution vs sandbox: NFR-047/048 chỉ đạt được nếu CLI không chạy trực tiếp trên host với toàn quyền. |
| OI-NFR-03 | Tauri vs web: NFR-044 (không credential tới frontend) dễ hơn với Tauri IPC; web local cần CSRF/auth cục bộ. |
| OI-NFR-04 | Máy tham chiếu để đo NFR hiệu năng chưa được xác định. |
| OI-NFR-05 | Pricing cho cost khi dùng CLI subscription. |

## 9. Cần người dùng phê duyệt
| # | Hạng mục |
|---|---|
| AP-NFR-01 | Tất cả giá trị đánh dấu Đề xuất (đặc biệt NFR-003, 004, 007, 008, 023, 050, 090) |
| AP-NFR-02 | Cấu hình máy tham chiếu để đo hiệu năng |
| AP-NFR-03 | Mặc định network deny (NFR-049) |
| AP-NFR-04 | Giải quyết OI-NFR-01…05 |
