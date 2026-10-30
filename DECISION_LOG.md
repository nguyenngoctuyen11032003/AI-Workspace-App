# DECISION_LOG.md — AI WORKSPACE

Trạng thái: tất cả quyết định dưới đây là **ĐỀ XUẤT** do Orchestrator tổng hợp từ nghiên cứu Phase 0. Chưa có quyết định nào được chủ sở hữu phê duyệt, trừ khi cột "Phê duyệt" ghi "Đã duyệt".
Ngày: 2026-10-30. Nguồn chi tiết: tài liệu trong `docs/` (cột Nguồn).

| ID | Quyết định đề xuất | Phương án đã loại | Nguồn | Phê duyệt |
|---|---|---|---|---|
| D-001 | Monorepo TypeScript (pnpm workspaces). Backend NestJS, frontend React 19 + Vite + Tailwind v4 + shadcn/ui. MVP chạy dạng web app + backend Node cục bộ. Tauri 2 shell ở phase sau, backend làm sidecar. | Electron, Next.js | research/LICENSE_AND_REUSE.md, development/DEPLOYMENT.md | APPROVED (Owner, 2026-10-30; xem sổ quyết định mục 8) |
| D-002 | Database PostgreSQL 16 + Redis. ORM Drizzle (postgres.js). 44 bảng, event store append-only phân vùng theo tháng. | Prisma | DATABASE_DESIGN.md | APPROVED (Owner, 2026-10-30; xem sổ quyết định mục 8) |
| D-003 | Workflow engine tự xây ("Conductor"): DAG executor trên BullMQ + trạng thái run lưu Postgres. Approval là trạng thái `WAITING_HUMAN`, không giữ worker. Semaphore đồng thời bằng Redis Lua (global + per-provider). | LangGraph JS, Temporal, BullMQ Flows thuần | research/ORCHESTRATION_WORKFLOW_RESEARCH.md | Chờ duyệt |
| D-004 | Claude Code tích hợp bằng `@anthropic-ai/claude-agent-sdk` (kế thừa đăng nhập OAuth của CLI). Cổng phê duyệt bằng `canUseTool` + PreToolUse hook. `claude auth status` chỉ dùng kiểm tra đăng nhập. | Spawn CLI thô | research/CLAUDE_CODE_INTEGRATION.md | Chờ duyệt |
| D-005 | Codex tích hợp ưu tiên qua `codex app-server` (JSON-RPC stdio), dự phòng `codex exec --json`. Codex CLI chưa cài trên máy; đã đăng nhập ChatGPT qua extension VS Code. | SDK `@openai/codex-sdk` làm đường chính | research/CODEX_INTEGRATION.md | Chờ duyệt |
| D-006 | OpenHands là provider thứ ba qua `openhands-agent-server` + `@openhands/typescript-client`, dùng cho Tier 2 sandbox. Không vendor mã nguồn. | Nhúng Python vào backend | research/OPENHANDS_ANALYSIS.md | Chờ duyệt |
| D-007 | Mô hình sandbox hai tầng. Tier 1: CLI chạy trên host với quyền người dùng thường, chính sách do app sinh ra, hook chặn/viết lại lệnh, lệnh build/test/run chạy trong Docker. Tier 2 (tùy chọn): runtime container hóa hoàn toàn. Chính sách CLI là lớp chính sách, không phải ranh giới bảo mật. | Chạy mọi thứ trên host | SECURITY.md | APPROVED (Owner, 2026-10-30; xem sổ quyết định mục 8) |
| D-008 | Credential do CLI tự quản. App không đọc `.credentials.json` hay `auth.json`. Secret khác lưu OS keychain. Không secret nào vào Postgres plaintext hoặc frontend. | Lưu token trong DB | SECURITY.md | Chờ duyệt |
| D-009 | Mỗi task một git worktree + branch `agent/<slug>/<task-id>`. Manifest quyền sở hữu file theo glob. Hook là lớp bắt buộc, kiểm diff sau cùng là cổng quyết định. Luồng `task → integration/<project> → main`, không tự merge vào main. | Workspace chung | architecture/WORKSPACE_AND_GIT_ISOLATION.md | Chờ duyệt |
| D-010 | Concurrency mặc định 5, cấu hình tối đa 18. Giới hạn thực tế là quota subscription, không phải số process. Chỉ nâng khi có bằng chứng thực (5 → 10 → 18). | Mặc định 18 | architecture/RESOURCE_MANAGEMENT.md, research/MULTI_AGENT_SCALABILITY.md | APPROVED (Owner, 2026-10-30; xem sổ quyết định mục 8) |
| D-011 | Virtual Office dùng Phaser 3.90 (cân nhắc Phaser 4 sau). React ↔ Phaser qua Zustand store + event bus. Mô hình engine tham khảo Pixel Agents (BFS, FSM, layout editor). Chuyển động chỉ do event thực điều khiển. | PixiJS, Canvas thuần | research/PIXEL_AGENTS_ANALYSIS.md, research/LICENSE_AND_REUSE.md | Chờ duyệt |
| D-012 | Giấy phép: ChatDev (Apache-2.0), Pixel Agents và OpenHands (MIT) được tái sử dụng có attribution. My Virtual Office (AGPL-3.0) chỉ tham khảo ý tưởng, cấm sao chép mã, prompt, layout, asset. Asset đóng gói chỉ CC0/MIT/OFL. | Copy mã MVO | research/LICENSE_AND_REUSE.md | Chờ duyệt |
| D-013 | Skill theo chuẩn Agent Skills. Mở rộng đặt trong `skill.yaml` + `metadata.aiw-*`. Claude nạp qua `--add-dir`, Codex qua junction vào `.agents/skills`. Memory tách khỏi Skill, có luồng duyệt. | Ghi trực tiếp vào repo | architecture/SKILL_SYSTEM.md | Chờ duyệt |
| D-014 | Event system: outbox giao dịch → Redis pub/sub (Socket.IO) + Redis Stream cho Virtual Office, replay bằng `lastEventId`. Bộ event theo đặc tả Prompt 1 mục 8.4. | Chỉ WebSocket thô | research/ORCHESTRATION_WORKFLOW_RESEARCH.md | Chờ duyệt |
| D-015 | Design: accent AI màu tím (`--ai-accent`), font Inter + JetBrains Mono, mật độ Comfortable, avatar agent bo vuông, avatar người tròn, giao diện mặc định tiếng Anh (VI là locale thứ hai). | Accent xanh, Geist | design/DESIGN_TOKENS.md | Chờ duyệt |
| D-016 | Chuẩn hóa: 18 vai trò agent mặc định là dữ liệu seed, không hardcode. Tách Agent Identity, Provider Identity, User Identity. | Danh sách cố định trong code | architecture/AGENT_ORGANIZATION.md | Chờ duyệt |
| D-017 | Áp dụng mô hình 5 attention state (blocked/working/done/idle/unknown) làm lớp hiển thị trên 12 lifecycle state; `done` chưa xem theo từng client; trạng thái project/department tổng hợp từ agent. | Chỉ hiển thị lifecycle thô | design/HERDR_DESIGN_INTEGRATION.md | Chờ duyệt |
| D-018 | Thêm Agent Rail (sidebar hai vùng: projects trên, agents dưới, kéo tỉ lệ, dòng cấu hình bằng token) và cấu trúc token theme kiểu herdr (preset + `theme.custom` + auto_switch). Giữ accent AI tím. | Sidebar một vùng | design/HERDR_DESIGN_INTEGRATION.md | Chờ duyệt |
| D-019 | Điều khiển bởi agent: biến môi trường `AIW_*`, CLI `aiw` trả JSON, skill cho agent, báo cáo trạng thái có `source` + `seq` đơn điệu. Token API riêng theo agent. | Agent chỉ giao tiếp qua prompt | design/HERDR_DESIGN_INTEGRATION.md | Chờ duyệt |
| D-020 | herdr (Apache-2.0) là tham chiếu thiết kế. HerdrRuntime và chế độ terminal tương tác chỉ là tùy chọn sau PoC (Phase 2 hoặc 7). Không dùng mã Rust. Heuristic màn hình không thay sự kiện SDK. | Dùng herdr làm đường chính | research/HERDR_ANALYSIS.md | Chờ duyệt |

## Mâu thuẫn và khoảng trống cần chủ sở hữu giải quyết
1. Chạy CLI trên host mâu thuẫn với yêu cầu "không chạy code AI trực tiếp trên host". D-007 là cách dung hòa, cần xác nhận.
2. 18 agent đồng thời so với quota subscription dùng chung và điều khoản dịch vụ. Cần xác minh điều khoản cho dùng nhiều phiên song song.
3. Docker daemon chưa chạy lúc kiểm tra, nên cgroup, ECI và rootless chưa xác minh.
4. Codex CLI chưa cài, nên mọi kiểm thử Codex thực chưa chạy.
5. Quy ước `task_status` viết HOA trong khi enum khác viết thường (DATABASE_DESIGN mục 12).

## Quyết định bổ sung
| ID | Quyết định đề xuất | Phương án đã loại | Nguồn | Phê duyệt |
|---|---|---|---|---|
| D-021 | Skill Overlay Directory: Owner thêm/sửa/xóa file Markdown trong `.ai/skills/<agent-slug>/` (ngoài worktree). Quét và nạp lúc giao mỗi task, tiêm vào system prompt Claude Code qua SDK `systemPrompt.append` hoặc `--append-system-prompt-file`, không hot-reload giữa task. Overlay không cấp quyền, agent không được sửa `.ai/**`. Resume cần xử lý do `--system-prompt-snapshot` (mặc định bật) giữ nguyên system prompt cũ: chờ PoC. | Chỉ quản lý skill qua Library; hot-reload giữa task | architecture/DYNAMIC_SKILL_INJECTION.md | PENDING_OWNER_APPROVAL |

## Phản hồi của Owner 2026-10-30
Chi tiết, kể cả phần không khớp nhãn và câu hỏi xác nhận Q-16..Q-20: `PHASE_0_OWNER_DECISION_REGISTER.md` mục 8.
- **APPROVED:** D-001 (trừ khung frontend, Q-16), D-002, D-007 (có điều kiện), D-010 (Q-07 còn mở).
- **PENDING_OWNER_APPROVAL:** D-003, D-004 (nhãn Owner không khớp nội dung, Q-17), D-005, D-006, D-008, D-009, D-011..D-021 (D-021 kèm OWN-AC-1..3, Q-18).
- **Chỉ thị chính sách:** hành động quan trọng cần duyệt thủ công (Human-in-the-loop), chưa trao tự trị hoàn toàn.
- **Chấp nhận có điều kiện** 13 rủi ro bảo mật Tier 1: Phase 1 không cho agent chạy thật; xác thực API, hardening Docker và audit kiểm tra ở đầu Phase 2.
