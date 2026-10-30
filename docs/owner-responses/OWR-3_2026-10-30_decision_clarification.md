# OWR-3 — Owner Final Response, Phase 0 Decision Clarification

Nguồn: tin nhắn của Owner trong phiên làm việc ngày 2026-10-30, sao chép lại nội dung quyết định làm nguồn truy vết. Mã tham chiếu: **OWR-3**. Phản hồi này trả lời danh sách ở tin nhắn trước của agent (16 mục, nhóm xung đột X, 12 FR, phạm vi chạy trên máy). Không phải tài liệu do agent sinh ra.

Quy tắc kèm theo của Owner: **Phase 0 vẫn mở. Không tự chuyển sang Completed và không bắt đầu Phase 1. Chỉ bàn Phase 1 khi Owner xác nhận rõ ràng.**

## A. 16 mục cần làm rõ
1. **D-014:** xác nhận 16 event trong Prompt 1 là danh sách gốc. Cho phép chép nguyên văn vào repo và ánh xạ dotted lowercase theo bảng Owner cung cấp. Không tự thêm, xóa hoặc đổi tên event gốc.
2. **Q-07:** giữ concurrency mặc định 5. Không tăng lên 10 hoặc 18. Đây là giới hạn scheduler nội bộ, không phải giới hạn được provider bảo đảm.
3. **FR-122:** artifact bắt buộc là output của các node required trong workflow definition. MVP lưu và xem trong workspace; export ZIP để phase sau.
4. **FR-181:** approval chờ lâu là operational alert, không phải incident. Incident cùng loại và cùng đối tượng trong cửa sổ 15 phút được grouping, có bộ đếm.
5. **FR-182:** task cancelled loại khỏi cả tử số và mẫu số. Baseline gồm task ban đầu và các thay đổi phạm vi đã được duyệt. Chấp thuận equal-weight cho MVP.
6. **FR-183:** dashboard nhiều metric, không tạo KPI tổng hợp trong MVP.
7. **Tile:** art 16 px, hiển thị phóng 2 lần. Benchmark sau.
8. **D-005:** MVP chỉ tích hợp Claude. Codex để sau; giữ AgentAdapter để mở rộng.
9. **Q-15:** asset MIT chỉ được dùng nếu xác minh nguồn gốc, license và ghi attribution vào NOTICE. Không rõ nguồn gốc thì không dùng. CC0 vẫn là mặc định.
10. **Tier 2/OpenHands:** OpenHands hoãn khỏi MVP. Không cho Agent chạy arbitrary code trên Windows host. Mọi Agent-generated code cần thực thi trong sandbox được cô lập. Docker chỉ chạy build/test/run không đủ nếu Agent vẫn có quyền thực thi trên host. Sandbox không hoạt động thì fail closed.
11. **Phase 2:** đồng ý sửa roadmap: chỉ mock hoặc sandbox khi security gate chưa đạt; không production-capable execution.
12. **Approval expiry:** tách `remind_at` và `expires_at`. `remind_at` phục vụ nhắc Owner sau 24 giờ; `expires_at` là 15 phút đối với approval của hành động quan trọng.
13. **Permission:** chuẩn hóa L0 đến L5.
14. **UI node types:** chấp thuận 9 loại: Agent, Task, Approval, Condition, Parallel, Merge, Tool, Test, Deployment. Đây là UI node types, không phải 11 engine kinds.
15. **WAITING_APPROVAL:** không tính vào giới hạn 5 active workers nếu worker đã được giải phóng và trạng thái được persist. Vẫn phải tuân thủ quota provider.
16. **Event naming:** chỉ dùng dotted lowercase làm canonical event naming. Không dùng underscore naming làm chuẩn.

## B. Các thay đổi X
Đồng ý đồng bộ: X-1 locale mặc định tiếng Việt; X-3 bỏ permission `full`; X-4 event dotted lowercase; X-7 timeout 540 giây; X-8 audit log online giữ 365 ngày. X-11 chỉ là ghi chú, không cần hành động.

## C. 12 FR còn thiếu thông tin
Chấp thuận các đề xuất với Acceptance Criteria như sau:
- **FR-006:** một quản lý trực tiếp mỗi nhân sự, không giới hạn độ sâu, chặn vòng lặp.
- **FR-009:** budget tính theo chi phí ước tính quy đổi từ token và các khoản phí liên quan. Theo dõi token riêng. Cảnh báo 80%; vượt 100% dừng cấp task và gọi model mới và yêu cầu Owner quyết định.
- **FR-038:** meeting là thread thực, PM ghi biên bản; animation chỉ phản ánh sự kiện.
- **FR-078:** WON'T DO IN MVP theo D-006; vẫn giữ trong traceability.
- **FR-088:** không lưu hoặc tự ý đọc credential trong ứng dụng theo D-008.
- **FR-098:** không có API key fallback trong MVP.
- **FR-153:** một tầng trong MVP; không tự mở rộng.
- **FR-186:** success rate = completed / (completed + failed) × 100%. Không tính running, blocked, cancelled. Nếu mẫu số bằng 0 thì hiển thị N/A.
- **FR-200:** N = 24 giờ, là operational alert.
- **FR-283:** dùng checklist usability 10 tiêu chí. Không tự xóa P0 nếu chưa có bằng chứng QA.
- **FR-284:** tiếng Việt mặc định, hỗ trợ tiếng Anh.
- **FR-292:** MVP chỉ hướng dẫn PostgreSQL dump; workspace export để phase sau.

## D. Provider terms
Owner nói đã kiểm tra tài liệu chính thức của Anthropic và OpenAI. Không có căn cứ công khai đủ rõ để khẳng định một seat Claude Team hoặc một ChatGPT plan được phép chạy chính xác 5, 10 hay 18 session song song. Vì vậy: giữ scheduler concurrency mặc định 5; không tuyên bố đây là giới hạn được provider cho phép; không chia sẻ credential hoặc seat giữa nhiều người; không tăng concurrency nếu chưa xác minh điều khoản và usage thực tế.

## E. Máy tính Owner và quyền thực thi
RAM trống hiện tại khoảng 4,04 GB.
- **Chưa cho phép** mở Docker Desktop hoặc chạy VAL-DOCKER-01 đến 09.
- Không cài Codex CLI. Không chạy 27 runbook. Không chạy PoC Conductor. Không test Agent thật qua SDK.
- Giữ phạm vi đã cấp phép trước đó cho Drizzle PoC và Dynamic Skill/Resume PoC trong scratch, không production, không secret thật, không external API.
- **Cho phép** QA độc lập lần 3 đối với tài liệu, traceability, AC và các thay đổi đã cập nhật.

## F. Git
Hai commit hiện có trên branch `docs/phase0-owner-decisions`, chưa push. Không push. Được phép chuẩn bị PR description. Kiểm tra lại git status, diff, secret và commit history trước khi báo cáo. Không tự tạo thêm commit nếu chưa kiểm tra các thay đổi phát sinh.

## G. Công việc tiếp theo (thứ tự Owner chỉ định)
1. Đồng bộ Decision Register, ADR và các tài liệu kiến trúc.
2. Đồng bộ 16 event gốc và mapping.
3. Viết AC cho FR-122, FR-181, FR-182, FR-183 và 12 FR bổ sung.
4. Cập nhật security gates và roadmap.
5. Giữ nguyên các nội dung chưa có đủ bằng chứng ở trạng thái PENDING.
6. Chạy QA độc lập lần 3.
7. Kiểm tra Git và chuẩn bị PR description, không push.
8. Xuất báo cáo chi tiết các mục PASS, FAIL, BLOCKED, NOT RUN và các câu hỏi còn chờ Owner.

---

## Phụ lục do agent ghi: các điểm không khớp trong OWR-3 (để Owner biết, không sửa nội dung bên trên)
1. Mục A.1 nhắc "bảng Owner cung cấp" cho ánh xạ event, nhưng tin nhắn **không có bảng nào**. Chỉ có quy tắc: dotted lowercase. Ví dụ Owner nêu (`task.started`, `task.completed`, `agent.blocked`) trong OWR-2: `task.started` và `agent.blocked` **không nằm** trong 16 event gốc, trong khi `TASK_COMPLETED` có trong danh sách gốc.
2. Mục A.10 chặt hơn mô hình Tier 1 của D-007 (CLI chạy trên host với chính sách và hook; Docker chỉ cho build, test, run). Xem xung đột X-13 trong sổ quyết định.
3. Mục A.14 phân biệt 9 UI node types với engine kinds nhưng không nêu lại số engine kinds; tài liệu hiện ghi 11.

## Phụ lục: Prompt 1 mục 8.4 (trích nguyên văn từ tin nhắn Prompt 1 của Owner trong phiên; Owner xác nhận ở A.1 đây là danh sách gốc)
> ## 8.4. Event System
> Tất cả hoạt động phải phát ra event có cấu trúc.
> Ví dụ:
> AGENT_CREATED
> AGENT_ASSIGNED
> AGENT_STARTED
> AGENT_THINKING
> AGENT_TOOL_STARTED
> AGENT_TOOL_COMPLETED
> AGENT_WAITING_APPROVAL
> AGENT_COMPLETED
> AGENT_FAILED
> TASK_CREATED
> TASK_COMPLETED
> TASK_BLOCKED
> WORKFLOW_STAGE_CHANGED
> HUMAN_APPROVAL_REQUIRED
> HUMAN_APPROVED
> HUMAN_REJECTED
> Event là nguồn dữ liệu để cập nhật UI, dashboard và văn phòng 2D.
> Không giả lập hoạt động của Agent bằng animation không có dữ liệu thực tế.

Ghi chú: văn bản gốc dùng từ "Ví dụ", và Owner nay xác nhận 16 tên là danh sách gốc, không thêm, xóa hay đổi tên.
