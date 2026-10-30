# PHASE_0_OWNER_DECISION_REGISTER — AI WORKSPACE

Ngày lập: 2026-10-30. Người lập: Decision Analyst (Phase 0, chỉ tài liệu). Phạm vi: gom D-001..D-020 (`DECISION_LOG.md`) và các mâu thuẫn cần owner (`docs/adr/CONFLICTS_TO_RESOLVE.md`, Phiên bản 2) thành một register để owner quyết định.

**Cập nhật 2026-10-30 (OWR-1, OWR-2):** Owner đã phản hồi; trạng thái hiện hành nằm ở bảng mục 2, 4 và mục 9. Phase 0 vẫn mở, Phase 1 chưa được cho phép. Đoạn "Quy tắc cứng" ban đầu dưới đây giữ làm lịch sử.

**Quy tắc cứng (bản gốc):** owner chưa duyệt bất kỳ mục nào. Mọi mục dưới đây đang `PENDING_OWNER_APPROVAL`. Không agent nào được đổi trạng thái sang APPROVED hay REJECTED; chỉ owner ghi vào cột "Owner response" (xem §6). Phase 1 chưa được bắt đầu.

## 1. Từ vựng trạng thái

| Trạng thái | Ý nghĩa | Ai đặt |
|---|---|---|
| PROPOSED | Agent đã đề xuất, chưa đưa vào hàng chờ owner | Agent |
| PENDING_OWNER_APPROVAL | Đã đưa vào register, đang chờ owner trả lời | Agent (trạng thái hiện tại của mọi D-xxx và Q-xx) |
| APPROVED | Owner chấp nhận; chỉ có hiệu lực khi owner tự ghi kèm ngày | Chỉ owner |
| REJECTED | Owner từ chối hoặc chọn phương án khác; ghi phương án thay thế | Chỉ owner |
| APPROVED_CONDITIONAL | Owner duyệt nhưng một điều kiện nêu rõ chưa đạt; chưa được coi là hoàn tất | Ghi theo lời Owner |
| PENDING_OWNER_CLARIFICATION | Owner chưa trả lời hoặc câu trả lời để hở một điểm; agent không tự điền | Agent ghi câu hỏi |
| VERIFIED | Chỉ khi có bằng chứng kiểm thử; hiện chưa có mục nào | Cần bằng chứng |

Nhãn bằng chứng: **Đã xác minh** = có lệnh chạy hoặc đọc mã thật (nêu nguồn); **Giả định** = chưa kiểm chứng; **Tài liệu tự báo cáo** = tài liệu nói đã đo nhưng QA (`docs/QA_REVIEW_PHASE0.md` §7) chưa tái hiện. Kiểm chứng bổ sung ngày 2026-10-30 cho register này: `docker info` (daemon không chạy), `where codex` (không có), `npm view drizzle-orm` (stable 0.45.4, 1.0 beta), `npm view phaser` (latest 4.2.1), `claude auth status` với `CLAUDE_CONFIG_DIR` rỗng (`loggedIn:false`).

## 2. Bảng tổng hợp D-001..D-021

Cập nhật sau OWR-1 và OWR-2 (2026-10-30). Cột Khuyến nghị là khuyến nghị gốc của analyst; cột Owner response ghi quyết định thực của Owner.

| ID | Chủ đề | Khuyến nghị | Trạng thái | Owner response | Ngày |
|---|---|---|---|---|---|
| D-001 | Monorepo TS, web-first, Tauri sau | Chọn A (pnpm thuần) | APPROVED | OWR-1 mục 8; OWR-2 Q-16: React 19 + Vite, không Next.js | 2026-10-30 |
| D-002 | PostgreSQL 16 + Redis + Drizzle | Chọn A, spike partition trước khi khóa | APPROVED | OWR-1 mục 8; OWR-2 §5.4 chỉ cho phép Drizzle PoC trong scratch | 2026-10-30 |
| D-003 | Conductor tự xây | Chọn A, kèm spike | APPROVED_CONDITIONAL | OWR-2 D-003, Q-20: định hướng; khóa implementation sau PoC | 2026-10-30 |
| D-004 | Claude qua Agent SDK | Chọn A | APPROVED | OWR-2 Q-17: Agent SDK; ToS/usage limit chưa xác nhận (Q-07) | 2026-10-30 |
| D-005 | Codex qua app-server | Chọn A nhưng đặt Codex sau Claude | APPROVED | OWR-2 D-005: hoãn Codex khỏi MVP (xung đột nhóm chấp nhận MVP B, X-2) | 2026-10-30 |
| D-006 | OpenHands provider thứ ba | Chọn A, hoãn sau Phase 4 | APPROVED | OWR-2 D-006: hoãn sau Phase 4 | 2026-10-30 |
| D-007 | Sandbox hai tầng | Chọn A kèm ký R1,R2,R3,R5,R6 | APPROVED_CONDITIONAL | OWR-1 mục 8.1, 8.3; OWR-2 §8 security gates | 2026-10-30 |
| D-008 | Credential do CLI quản | Chọn A | APPROVED | OWR-2 D-008 | 2026-10-30 |
| D-009 | Worktree per task | Chọn A | APPROVED | OWR-2 D-009, Q-04 | 2026-10-30 |
| D-010 | Concurrency 5, trần 18 | Chọn A | APPROVED | OWR-1 mục 8; OWR-2 Q-07: 5 là giới hạn ứng dụng, ToS mở | 2026-10-30 |
| D-011 | Phaser 3.90 | Chọn A, ghi lý do ở 3.x | APPROVED_CONDITIONAL | OWR-2 D-011: Phaser 3.90 qua OfficeRenderer; benchmark trước khi khóa | 2026-10-30 |
| D-012 | Giấy phép và tái dùng | Chọn A | APPROVED | OWR-2 D-012, Q-15 | 2026-10-30 |
| D-013 | Agent Skills | Chọn A | APPROVED | OWR-2 D-013 | 2026-10-30 |
| D-014 | Event: outbox + Redis | Chọn A (bảng `event_outbox`) | APPROVED | OWR-2 D-014, Q-19; PENDING_OWNER_CLARIFICATION: danh sách event mục 8.4 chưa có trong repo | 2026-10-30 |
| D-015 | Design accent tím, Inter | Chọn A | APPROVED | OWR-2 D-015: locale mặc định Vietnamese (xung đột X-1) | 2026-10-30 |
| D-016 | 18 vai trò là seed data | Chọn A | APPROVED | OWR-2 D-016 | 2026-10-30 |
| D-017 | 5 attention state | Chọn A | APPROVED | OWR-2 D-017 | 2026-10-30 |
| D-018 | Agent Rail + theme token | Chọn A | APPROVED | OWR-2 D-018 | 2026-10-30 |
| D-019 | Điều khiển bởi agent (`aiw`) | Chọn A, hoãn tới Phase 3 | APPROVED_CONDITIONAL | OWR-2 D-019: chỉ Phase 3 sau Security Review | 2026-10-30 |
| D-020 | herdr chỉ tham chiếu | Chọn A | APPROVED | OWR-2 D-020 | 2026-10-30 |
| D-021 | Skill Overlay Directory (mục 7) | Chọn A | APPROVED_CONDITIONAL | OWR-2 Q-18: chỉ chấp thuận có điều kiện đến khi OWN-AC-1..3 được kiểm thử thành công | 2026-10-30 |

## 3. Chi tiết từng quyết định

Mỗi mục có: Câu hỏi cho owner; Phương án; Khuyến nghị; Yếu tố quyết định; Hệ quả chi phí/bảo mật; Bằng chứng; Bị chặn tới khi quyết; Khả năng đảo ngược; Mâu thuẫn được đóng khi duyệt. Nguồn đầy đủ: `docs/adr/ADR-00xx` cùng số.

### D-001 Monorepo TypeScript, web-first, Tauri 2 sau (ADR-0001)
- Câu hỏi owner: "Bạn đồng ý xây AI WORKSPACE bằng pnpm monorepo TypeScript (NestJS + React 19/Vite/Tailwind v4/shadcn), chạy web app + backend Node cục bộ ở MVP và đóng gói desktop bằng Tauri 2 ở phase sau? Dùng pnpm thuần, chưa dùng Turborepo?"
- Phương án: A) pnpm workspaces + NestJS + Vite, Tauri 2 sau (P: backend Node spawn CLI trực tiếp, tránh chặn bởi sidecar/ký code ở Phase 1; N: Phase 1 chưa có tray/auto-start; cần người biết Rust khi lên Tauri). B) Electron (P: Node có sẵn, mẫu từ OpenHands; N: installer 80-240 MB, RAM 200-300 MB). C) Next.js full-stack (loại: không hợp worker dài hạn, WebSocket, desktop). Tùy chọn con: thêm Turborepo ngay (CF-027) hay đợi > ~5 package.
- Khuyến nghị: A, pnpm thuần.
- Yếu tố quyết định: năng lực Rust của nhóm; việc có cần desktop installer trong MVP không; chi phí packaging native module (`node-pty`) trên Windows đường dẫn có khoảng trắng.
- Chi phí/bảo mật: A rẻ nhất ở Phase 1; Tauri thêm capability model (lợi bảo mật) nhưng thêm chi phí sidecar sau. Không phát sinh chi phí vận hành.
- Bằng chứng: Đã xác minh pnpm 10.34.6 và Node v24.19.0 trên máy (TE-10, QA §3 #15). Giả định: số liệu Tauri (8-15 MB, 30-50 MB RAM); phiên bản React 19.2/Vite 7/Tailwind v4/TS 5.9 lấy từ web search, cần `npm view` trước khi tạo `package.json`; Playwright + WebView2 chưa thử.
- Bị chặn: P1-01 (khởi tạo repo; thư mục hiện chưa là git repo), mọi quyết định tooling (CF-027, CF-028, CF-041).
- Đảo ngược: dễ ở Phase 1 (chưa có mã); khó sau khi có sidecar. Điều kiện đảo: spike sidecar NestJS trên Windows thất bại thì chuyển Electron.
- Đóng khi duyệt: CF-027, CF-028, CF-041, CF-042 (phần `apps/api`).

### D-002 PostgreSQL 16 + Redis, Drizzle, event store phân vùng (ADR-0002) — sâu
- Câu hỏi owner: "Bạn đồng ý dùng PostgreSQL 16 + Redis làm kho dữ liệu chính, ORM Drizzle (bản stable 0.45.x), 44 bảng với event store append-only phân vùng theo tháng, thay vì Prisma?"
- Phương án: A) Drizzle + drizzle-kit + postgres.js (P: SQL đọc được, partial index, trigger, không binary engine nên hợp đường dẫn có khoảng trắng; N: phải viết SQL tay cho bảng phân vùng và FK vòng, tự viết `DrizzleModule`, API quan hệ kém chín, chưa có prototype). B) Prisma 6/7 (P: hệ sinh thái chín, Studio; N: không mô tả partition/trigger trong schema, dịch lại 44 bảng). C) TypeORM (không được ai đánh giá). Phương án con: có làm partition từ đầu hay chỉ bảng thường rồi partition sau.
- Khuyến nghị: A, kèm spike 1 ngày (partition `agent_events` + partial unique index bằng `drizzle-kit generate --custom`, chạy migrate hai lần) trước khi khóa schema; ghi đúng phiên bản (0.45.x, 1.0 còn beta) thay cho nhãn "v1.x".
- Yếu tố quyết định: kết quả spike drift; khả năng nhóm viết SQL; nhu cầu partition từ MVP (18 agent, 5-30 audit/s cao điểm là giả định); đường dẫn schema (`apps/api/src/database/**` theo DATABASE_DESIGN, hay `packages/db/**` theo ADR-0002; đề xuất theo DATABASE_DESIGN).
- Chi phí/bảo mật: không có chi phí license; lợi bảo mật: trigger `reject_mutation` cho audit append-only, REVOKE UPDATE/DELETE. Chi phí đổi ORM muộn rất cao.
- Bằng chứng: Đã xác minh `drizzle-orm` stable = 0.45.4 (npm view hôm nay); Giả định/Tài liệu: so sánh 12 tiêu chí trong DATABASE_DESIGN §1; chưa có prototype migration; partitioned unique index với `seq` chưa kiểm trên PG 16 (AUDIT_LOGGING §15). Chưa kiểm daemon Postgres/Redis trên máy dev.
- Bị chặn: P1-04 (contract freeze), migration `0000_enums_core`, mọi tài liệu có `prisma/**`, CF-001, CF-005/006 (audit), CF-010, CF-035, CF-036.
- Đảo ngược: dễ trước migration đầu tiên, rất đắt sau Phase 2. Nên khóa sau spike.
- Đóng khi duyệt: CF-001; liên quan CF-004, CF-035, CF-036.

### D-003 Workflow engine tự xây "Conductor" (ADR-0003) — sâu
- Câu hỏi owner: "Bạn chấp nhận tự xây workflow engine (DAG executor trên BullMQ + trạng thái trong Postgres, approval là `WAITING_HUMAN`) thay vì dùng LangGraph JS hoặc Temporal, biết rằng phần lớn công sức Phase 1-3 dồn vào đây?"
- Phương án: A) Conductor (P: JSON định nghĩa khớp React Flow, approval chờ nhiều ngày không giữ worker, thêm 0 hạ tầng; N: chi phí xây cao, rủi ro idempotency, cần test chaos, ước lượng Phase 3 có thể thấp). B) LangGraph JS + PostgresSaver (P: ít code; N: `interrupt()` chạy lại node từ đầu, graph là code, không cap xuyên process). C) Temporal TS (P: bền nhất; N: 4 service + DB, workflow là code, quá nặng cho ~10 agent). D) BullMQ Flows thuần (loại: cây, không diamond/wait-human).
- Khuyến nghị: A, với spike "scheduler + tool node giả" qua test chaos (crash recovery 20/20, AC-127) trước khi cam kết Phase 3.
- Yếu tố quyết định: ngân sách thời gian; mức chấp nhận rủi ro tự xây; việc workflow có phải JSON do người dùng sửa trên UI (đang là yêu cầu).
- Chi phí/bảo mật: chi phí phát triển cao, chi phí vận hành thấp (PG + Redis đã có). Bảo mật: trạng thái nằm trong DB của ta, audit/replay dễ.
- Bằng chứng: Đã xác minh: đối chiếu mã ChatDev (TE-02); Giả định: chưa có prototype; license Inngest còn mâu thuẫn; ma trận R1-R10 là Tài liệu.
- Bị chặn: Phase 3, CF-010 (enum node), CF-029, CF-055, CF-007 (WAITING_HUMAN).
- Đảo ngược: trung bình trước Phase 3; đắt sau khi có definitions người dùng. Điều kiện đảo: spike chaos thất bại thì Temporal hoặc LangGraph bọc ngoài.
- Đóng khi duyệt: CF-010, CF-055 (một phần), liên quan Q-11, Q-09.

### D-004 Claude Code qua `@anthropic-ai/claude-agent-sdk` (ADR-0004) — sâu
- Câu hỏi owner: "Bạn đồng ý điều khiển Claude Code bằng Agent SDK (kế thừa đăng nhập OAuth của CLI, cổng duyệt `canUseTool` + PreToolUse hook), thay vì spawn CLI thô; và xác nhận bạn được phép dùng gói subscription `team` cho tự động hóa kiểu này?"
- Phương án: A) Agent SDK (P: đã chạy thật, kế thừa OAuth, interrupt/resume; N: SDK 0.3.x đổi nhanh, phải ghim phiên bản, binary lớn). B) spawn `claude -p --output-format stream-json` (P: không phụ thuộc SDK; N: tự dựng cổng duyệt và parse NDJSON, `ENAMETOOLONG` trên Windows). C) hook + transcript tail (loại: không điều khiển được). Nhánh dự phòng nếu ToS cấm: API key trả theo token.
- Khuyến nghị: A, ghim phiên bản, `RawCliClaudeAdapter` chỉ cho `claude auth status` và `--version`; spawn mặc định `default` + whitelist (CF-014); không dùng `CLAUDE_CONFIG_DIR` riêng (CF-047, đã kiểm chứng mất đăng nhập).
- Yếu tố quyết định: điều khoản dịch vụ subscription cho tự động hóa (chưa xác minh); khả năng chấp nhận SDK thay đổi; chi phí API key nếu phải đổi.
- Chi phí/bảo mật: subscription không tính phí theo token nhưng quota 5 giờ + tuần là trần; mỗi spawn ~48k token cố định. Bảo mật: hai tầng duyệt dùng chung policy engine; credential do CLI quản (D-008).
- Bằng chứng: Đã xác minh: CLI 2.1.312, `claude auth status` (`claude.ai`, `team`), SDK 0.3.314 tồn tại (QA §3 #11-13,17); Tài liệu tự báo cáo: smoke test `canUseTool` chặn `Write`, 5 process song song cùng login (chưa tái hiện); Giả định: hook `type:"http"` trên Windows, hành vi `PermissionRequest` trùng `canUseTool`, `interrupt()` trước `steer`.
- Bị chặn: adapter Claude (Phase 2), SECURITY §5.1 viết lại (CF-013), CF-014, CF-047, CF-007.
- Đảo ngược: trung bình (adapter nằm sau interface `AgentAdapter`); điều kiện đảo: ToS cấm hoặc SDK phá API liên tục thì raw CLI + MCP permission tool.
- Đóng khi duyệt: CF-013, CF-014, CF-047; liên quan Q-05, Q-07.

### D-005 Codex qua `codex app-server` (ADR-0005) — sâu
- Câu hỏi owner: "Bạn muốn Codex nằm trong MVP không? Nếu có, đồng ý dùng `codex app-server` (JSON-RPC, experimental) làm đường chính và `codex exec --json` dự phòng, và bạn sẽ cài Codex CLI để chạy live test (tốn quota ChatGPT)?"
- Phương án: A) app-server + fallback exec (P: có approval tương tác, nhiều thread một process, đọc quota; N: API experimental, mọi hành vi hiện chỉ là giả định). B) `@openai/codex-sdk` (loại: không approval callback). C) chỉ `exec --json` (P: ổn định; N: không duyệt tương tác). D) hoãn Codex khỏi MVP (P: ít rủi ro, ít quota; N: một provider).
- Khuyến nghị: A nhưng triển khai sau Claude; nếu owner không muốn cài hoặc tốn quota, chọn D.
- Yếu tố quyết định: có muốn provider thứ hai trong MVP; sẵn sàng cài CLI; hạn mức ChatGPT; ToS fair-use.
- Chi phí/bảo mật: live test tiêu quota ChatGPT; app sinh cấu hình riêng, không kế thừa `~/.codex/config.toml`; `mxc` sandbox Windows build 26300 chưa kiểm (R10).
- Bằng chứng: Đã xác minh: `codex` không có trên PATH (QA §3 #14 và `where codex` hôm nay), binary bundled 0.164.0-alpha.11.1, npm latest 0.166.0; Giả định: toàn bộ hành vi app-server, `approvals$`, `CODEX_HOME` riêng.
- Bị chặn: adapter Codex, CF-015, CF-050, CF-049 (Codex), Q-10.
- Đảo ngược: dễ (adapter độc lập). Điều kiện đảo: live test thất bại thì `exec --json` + Tier 2.
- Đóng khi duyệt: CF-015, CF-050.

### D-006 OpenHands là provider thứ ba, Tier 2 (ADR-0006)
- Câu hỏi owner: "Bạn muốn OpenHands (agent-server) làm provider thứ ba cho sandbox Tier 2 không? Nếu có, nó cần LLM API key riêng (tốn token thật), ACP sub-mode tắt."
- Phương án: A) `openhands-agent-server` + `@openhands/typescript-client`, không vendor mã (P: dùng chung giao thức event, MIT; N: thêm service ngoài cần quản vòng đời, cần API key riêng). B) tự xây runtime container tối giản (P: toàn quyền; N: tốn công). C) bỏ provider này (P: gọn; N: Tier 2 yếu).
- Khuyến nghị: A, đặt sau Phase 4.
- Yếu tố quyết định: Tier 2 có thực sự dùng không; chấp nhận chi phí API key; Docker chạy được.
- Chi phí/bảo mật: chi phí token LLM riêng; `uvx` mặc định không sandbox, phải dùng container; rủi ro cùng origin localStorage (SELF_HOSTING).
- Bằng chứng: Đã xác minh: license MIT, repo agent-canvas v1.34.0, `@openhands/typescript-client` 1.59.0 tồn tại (QA §3 #3,17); Giả định: chưa khởi chạy agent-server vì Docker daemon không chạy.
- Bị chặn: Tier 2 (Phase 4), CF-024, CF-054.
- Đảo ngược: dễ. Đóng khi duyệt: CF-054 (một phần), liên quan Q-02.

### D-007 Sandbox hai tầng (ADR-0007) — sâu, quyết định rủi ro cao nhất
- Câu hỏi owner: "Yêu cầu gốc của bạn là không chạy code AI trực tiếp trên máy host. D-007 cho CLI (Claude/Codex) chạy trên host với quyền người dùng thường, còn mọi lệnh build/test/run chạy trong Docker, kèm policy và hook. Bạn có chấp nhận và ký xác nhận bằng văn bản các rủi ro R1, R2, R3, R5, R6 không? Nếu không, Tier 2 (toàn bộ trong container) làm mặc định."
- Phương án: A) hai tầng (P: giữ được đăng nhập OAuth subscription; N: không có OS-level sandbox cho CLI (R1), `.credentials.json` đọc được bởi mọi process cùng user (R2), prompt injection không chặn triệt để (R3), Docker Desktop là root-equivalent (R5), egress bằng proxy không phải firewall kernel (R6); chính sách CLI chỉ là lớp chính sách, không phải ranh giới bảo mật). B) Tier 2 làm mặc định (P: đúng yêu cầu gốc; N: cần API key riêng cho từng agent, mất subscription login, thêm chi phí token và OpenHands). C) chạy hoàn toàn trên host (loại: vi phạm yêu cầu gốc).
- Khuyến nghị: A, với điều kiện fail-closed khi daemon tắt (không có container thì không chạy lệnh thực thi code), test SB-00..SB-17 phải qua trên máy có daemon, và owner ký R1, R2, R3, R5, R6 (R4, R7, R8, R9 chấp nhận; R10 ghi nhận).
- Yếu tố quyết định: mức chịu rủi ro cho tài khoản `team` và máy dùng chung với dữ liệu cá nhân; chấp nhận chi phí API key nếu chọn B; chạy máy riêng/VM có làm giảm rủi ro không.
- Chi phí/bảo mật: A không tốn thêm token; B tốn token theo API. Rủi ro tồn dư R1-R10 ở SECURITY §10.
- Bằng chứng: Đã xác minh: Docker CLI 29.8.1 có mặt nhưng daemon không chạy (kiểm lại hôm nay); `.credentials.json` quyền đọc (QA §3 #16: kích thước đổi, mode 644 là artifact của Git Bash; ACL Windows); Giả định: cgroup, ECI, rootless, `mxc` build 26300, egress allowlist, mọi test SB-xx chưa chạy.
- Bị chặn: toàn bộ Phase 4 (sandbox), CF-016, CF-053, CF-043, CF-045, và Tier 1 hook rewrite.
- Đảo ngược: khó sau khi có mã sandbox; nhưng Tier 2 mặc định là đường lùi đã thiết kế. Điều kiện đảo: owner không ký, Anthropic phát hành sandbox native Windows, test SB-xx phát hiện vượt ranh giới.
- Đóng khi duyệt: CF-016, CF-043, CF-053 (liên quan Q-01, Q-10).

### D-008 Credential do CLI tự quản (ADR-0008)
- Câu hỏi owner: "Bạn đồng ý: app không bao giờ đọc `.credentials.json`/`auth.json`, secret khác chỉ lưu OS keychain (Windows Credential Manager), không secret nào trong Postgres plaintext hay frontend?"
- Phương án: A) như đề xuất (P: giảm bề mặt tấn công; N: không kiểm soát quyền file của CLI, R2). B) lưu token trong DB (loại).
- Khuyến nghị: A; chỉ `credential_ref`, không bảng `secrets` ở MVP (CF-052).
- Yếu tố quyết định: chấp nhận R2; thư viện keychain (`@napi-rs/keyring` vs `keytar`) trên Node 24 chưa thử.
- Chi phí/bảo mật: không chi phí; lợi bảo mật cao. Redaction corpus >= 200 mẫu x 6 encoding, 0 leak là tiêu chí.
- Bằng chứng: Đã xác minh: `claude auth status` kế thừa login của một tài khoản; QA §4 không thấy đọc credential; Giả định: thư viện keychain; redaction chưa kiểm.
- Bị chặn: module credential/redaction, CF-052, CF-051, CF-024.
- Đảo ngược: trung bình; đổi sang API key per agent thì thiết kế lại vault.
- Đóng khi duyệt: CF-051, CF-052; liên quan Q-02.

### D-009 Worktree per task, manifest, diff gate (ADR-0009)
- Câu hỏi owner: "Bạn đồng ý mỗi task một git worktree + branch `agent/<slug>/<task-id>`, quyền ghi theo manifest, kiểm diff là cổng cuối, luồng `task -> integration/<project> -> main`, không tự merge vào `main`?"
- Phương án: A) như đề xuất (P: cô lập logic, rollback dễ; N: cách ly chỉ ở mức logical (R4), chi phí đĩa; giới hạn 6/12 worktree). B) workspace chung (loại).
- Khuyến nghị: A; manifest JSON `schemaVersion: 1`, key `readOnly`, ngoài worktree (CF-046).
- Yếu tố quyết định: chấp nhận R4; Windows đường dẫn dài/có khoảng trắng (chưa test quoting).
- Chi phí/bảo mật: tốn đĩa theo số worktree; tăng an toàn.
- Bằng chứng: Tài liệu (chưa kiểm chứng): thời gian tạo worktree < 2 s là tiêu chí chưa đo; thư mục hiện chưa là git repo.
- Bị chặn: Git Service, CF-032, CF-046, Q-04.
- Đảo ngược: trung bình. Đóng khi duyệt: CF-032, CF-046; liên quan CF-017 (Q-04), CF-053.

### D-010 Concurrency mặc định 5, trần 18 (ADR-0010) — sâu
- Câu hỏi owner: "Bạn đồng ý mặc định 5 agent chạy đồng thời, cho phép cấu hình tới 18, và chỉ nâng 5 -> 10 -> 18 sau khi đo thực? Xác nhận bạn hiểu giới hạn thật là quota subscription, không phải số process."
- Phương án: A) mặc định 5, trần 18, nâng theo bằng chứng (P: tiết kiệm quota và RAM 16 GB; N: không đạt 18 ngay). B) mặc định 18 (loại: quota và RAM). C) mặc định 3 (P: an toàn nhất; N: không đạt kỳ vọng nhiều agent).
- Khuyến nghị: A; cap provider Claude 5, Codex 3; `max_retry` 2 (CF-038).
- Yếu tố quyết định: gói subscription (hạn mức 5 giờ + tuần); RAM thực 15,77 GB (free ~4,1 GB lúc QA đo); ToS.
- Chi phí/bảo mật: mỗi spawn ~48k token cố định, 18 agent là 864k token chỉ riêng khởi động; Docker Desktop VM 2-6 GB; khuyến nghị `.wslconfig 12GB` áp cho máy 32 GB sẽ chiếm 76% máy này (QA P1-12).
- Bằng chứng: Tài liệu tự báo cáo: 1 process 5 975 ms/235 MB, 3 process 8 172 ms/đỉnh 685,6 MB (QA §7 chưa tái hiện); Đã xác minh: cấu hình máy (i5-11400H, 15,8 GB, QA §3 #15); Giả định: bảng ngoại suy 5/10/18, T7 (18 session) chưa chạy.
- Bị chặn: scheduler/semaphore, seed DB (`max_concurrent_agents`), CF-002, CF-038, CF-048, CF-049.
- Đảo ngược: rất dễ (cấu hình). Đóng khi duyệt: CF-002, CF-038, CF-048, CF-049; liên quan Q-07, Q-08.

### D-011 Virtual Office dùng Phaser 3.90 (ADR-0011)
- Câu hỏi owner: "Bạn đồng ý dùng Phaser 3.90 (không phải bản 4.x mới nhất 4.2.1) cho Virtual Office, nhân vật chỉ chuyển động theo event thật?"
- Phương án: A) Phaser 3.90 sau `OfficeRenderer` (P: ổn định, có template React; N: bản 4 đã ra, nâng cấp sau). B) Phaser 4.x ngay (P: mới; N: rủi ro API, ít tài liệu). C) PixiJS/Canvas (loại theo D-011).
- Khuyến nghị: A, ghi lý do ở 3.x và điều kiện nâng cấp (QA P2-09).
- Yếu tố quyết định: độ ổn định; hiệu năng 18+8 helper trong 16,6 ms/khung (mục tiêu chưa đo).
- Chi phí/bảo mật: không chi phí license (MIT); chi phí nâng cấp sau.
- Bằng chứng: Đã xác minh: `phaser@3.90.0` tồn tại, latest 4.2.1 (npm view hôm nay); Giả định: hiệu năng, headless test.
- Bị chặn: P-Office (Phase 5), CF-018, CF-019, CF-020, CF-022, CF-023.
- Đảo ngược: dễ nhờ `OfficeRenderer`. Đóng khi duyệt: CF-018, CF-019, CF-020, CF-023; liên quan Q-14.

### D-012 Giấy phép và tái sử dụng (ADR-0012)
- Câu hỏi owner: "Bạn đồng ý: tái dùng ChatDev (Apache-2.0), Pixel Agents và OpenHands (MIT) có attribution; My Virtual Office (AGPL-3.0) chỉ tham khảo ý tưởng, cấm sao chép mã/prompt/layout/asset; asset đóng gói theo danh sách giấy phép nào?"
- Phương án: A) như đề xuất, asset CC0/MIT/OFL (P: pháp lý sạch; N: ít lựa chọn). B) thêm CC-BY (P: nhiều asset; N: phải credit). C) chỉ CC0 (đúng yêu cầu gốc; N: hạn chế).
- Khuyến nghị: A, ADR clean-room cho MVO với checklist reviewer và lint chặn path `my-virtual-office` (QA P1-10).
- Yếu tố quyết định: yêu cầu gốc "chỉ CC0" (Q-15); tư vấn pháp lý (tài liệu tự khuyến nghị luật sư rà soát).
- Chi phí/bảo mật: rủi ro pháp lý nếu sai; không chi phí tiền.
- Bằng chứng: Đã xác minh: văn bản LICENSE của ChatDev (Apache-2.0), pixel-agents (MIT), MVO (AGPL-3.0), OpenHands (MIT), herdr (Apache-2.0) và commit id (QA §3 #1-4); Giả định: license sprite JIK-A-4 (CC0) chưa lưu bằng chứng, itch.io trả 403 khi tự động.
- Bị chặn: bundle asset, `THIRD_PARTY_NOTICES.md`, CF-025, CF-026, CF-054.
- Đảo ngược: dễ trước khi phát hành; khó sau khi phân phối. Đóng khi duyệt: CF-025, CF-054; liên quan Q-15.

### D-013 Agent Skills (ADR-0013)
- Câu hỏi owner: "Bạn đồng ý dùng chuẩn Agent Skills (SKILL.md), mở rộng trong `skill.yaml`, nạp cho Claude qua `--add-dir` và Codex qua junction `.agents/skills`; Memory tách khỏi Skill và có luồng duyệt?"
- Phương án: A) như đề xuất (P: chuẩn mở, tương thích ChatDev/OpenHands; N: Windows junction cho Codex chưa thử). B) ghi trực tiếp vào repo (loại).
- Khuyến nghị: A.
- Yếu tố quyết định: Codex trong MVP hay không (D-005).
- Chi phí/bảo mật: skill nạp làm tăng prompt; kiểm soát memory giảm rủi ro injection.
- Bằng chứng: Đã xác minh: cấu trúc skill ở ChatDev/OpenHands (QA §3 #8-10); Giả định: junction với Codex, giới hạn 1 536 ký tự mô tả skill.
- Bị chặn: Skill system (Phase 2-3). Đảo ngược: dễ. Đóng khi duyệt: không có CF.

### D-014 Event system: outbox + Redis (ADR-0014) — quyết định hợp đồng
- Câu hỏi owner: "Bạn đồng ý event system gồm: ghi event cùng transaction với trạng thái, bảng `event_outbox`, Redis pub/sub (Socket.IO) và Redis Stream cho Virtual Office, replay bằng `lastEventId`? (Tên event: xem Q-12.)"
- Phương án: A) outbox ở mọi chế độ với một leader relay (P: không mất/mồ côi event, giữ append-only; N: thêm 1 bảng). B) con trỏ `global_seq` (loại: `bigserial` không đúng thứ tự commit nên có thể bỏ sót). C) publish-after-commit cho desktop (P: đơn giản; N: mất event khi crash, hai nhánh mã).
- Khuyến nghị: A (đã sửa đề xuất v1 của CF-004).
- Yếu tố quyết định: mức chấp nhận mất event; thứ tự cho timeline/office.
- Chi phí/bảo mật: chi phí ghi thêm nhỏ. 
- Bằng chứng: Đã xác minh (đọc DB §5.1 và EVENT_SYSTEM:226,247); Giả định: "Prompt 1 mục 8.4" mà D-014 trích không có trong `docs/` (QA P1-13), nghĩa là đặc tả event gốc của owner chưa được chép vào repo.
- Bị chặn: P1-04 (contract freeze), CF-003, CF-004, CF-035, CF-042. Đảo ngược: trung bình (đổi trước freeze). Đóng khi duyệt: CF-004, CF-035; liên quan Q-12.
- Việc owner cần làm: chép đặc tả event của Prompt 1 mục 8.4 vào `docs/` hoặc xác nhận danh sách event hiện tại thay thế.

### D-015 Design: accent tím, Inter + JetBrains Mono (ADR-0015) — sâu
- Câu hỏi owner: "Bạn chọn accent AI màu tím (`--ai-accent`), font Inter + JetBrains Mono (thay vì Geist), mật độ Comfortable, avatar agent bo vuông, avatar người tròn, giao diện mặc định tiếng Anh (tiếng Việt là locale thứ hai)?"
- Phương án: A) như đề xuất (P: tách màu hành động/AI khỏi màu trạng thái, WCAG text-tertiary đã tính; N: tím có thể bị xem là "AI cliché", font self-host làm tăng bundle, EN mặc định có thể không hợp người dùng VI). B) accent xanh + Geist (P: gần phong cách phổ biến; N: Geist chưa đánh giá, accent xanh dễ nhầm trạng thái `running`). C) locale mặc định VI (P: hợp owner VI; N: cần i18n đầy đủ ngay).
- Khuyến nghị: A, nhưng owner nên cân nhắc đặt VI làm locale mặc định nếu người dùng đầu tiên là người Việt (đổi chỉ là cấu hình mặc định).
- Yếu tố quyết định: nhận diện thương hiệu; ngôn ngữ người dùng đầu tiên; kiểm thử người dùng.
- Chi phí/bảo mật: không chi phí tiền; font Inter/JetBrains Mono self-host (kiểm license OFL).
- Bằng chứng: Đã xác minh: spacing 4px và WCAG text-tertiary, cơ chế Tailwind v4 `@theme` (DESIGN_TOKENS §4 T2, T4); Giả định: bảng màu primitives và chọn font (DESIGN_TOKENS §3.2, §3.3 tự đánh dấu Cần phê duyệt); chưa có kiểm thử người dùng.
- Bị chặn: bộ token trong `packages/ui` (Phase 1-2), CF-021. Đảo ngược: rất dễ trước khi có nhiều component, vẫn dễ sau nhờ token. Đóng khi duyệt: CF-021.

### D-016 18 vai trò agent là seed data (ADR-0016)
- Câu hỏi owner: "Bạn đồng ý 18 vai trò mặc định chỉ là dữ liệu seed (không hardcode trong code), và tách Agent Identity, Provider Identity, User Identity?"
- Phương án: A) seed data (P: thêm/bớt vai trò không cần sửa code; N: cần bảng template/version, CF-036). B) danh sách cố định (loại).
- Khuyến nghị: A. Yếu tố: nhu cầu tùy biến vai trò. Chi phí/bảo mật: thêm 3 bảng; tách identity giúp audit.
- Bằng chứng: Đã xác minh: QA §4 không thấy hardcode 12/18 (ngoại lệ nhẹ SCREEN_SPECIFICATION:320); Giả định: thiết kế bảng template.
- Bị chặn: seed, CF-008, CF-031, CF-036, CF-055. Đảo ngược: dễ. Đóng khi duyệt: CF-036 (cùng D-017).

### D-017 5 attention state trên 12 lifecycle state (ADR-0017) — sâu
- Câu hỏi owner: "Bạn đồng ý thêm lớp hiển thị 5 trạng thái chú ý (blocked/working/done/idle/unknown) phía trên 12 lifecycle state; `done` theo dõi 'đã xem' theo từng client; trạng thái project/department được tổng hợp từ agent (ưu tiên blocked > working > done > idle)?"
- Phương án: A) như đề xuất (P: phù hợp quy mô 18 agent, khớp luật không giả lập tiến độ; N: thêm bảng `notification_reads`, thêm lớp ánh xạ phải test, nguy cơ lệch SCREEN_SPEC). B) chỉ hiển thị 12 lifecycle (P: không thêm gì; N: quá nhiều trạng thái trên 18 agent). C) chỉ cờ cục bộ client cho "đã xem" (P: không bảng; N: không đồng bộ giữa client).
- Khuyến nghị: A, hàm thuần 12 -> 5 -> 8 tone trong `packages/contracts` có test; `working` chỉ khi có event thật trong cửa sổ TTL, `unknown` khi mất heartbeat.
- Yếu tố quyết định: số client dùng đồng thời; muốn đồng bộ "đã xem" hay không.
- Chi phí/bảo mật: thêm 1 bảng nhỏ; không ảnh hưởng bảo mật; heuristic nhận dạng từ màn hình bị loại.
- Bằng chứng: Giả định: đánh dấu Đề xuất, chưa triển khai (HERDR_DESIGN_INTEGRATION §2); Đã xác minh: mô hình trạng thái herdr đọc từ tài liệu/mã (HERDR_ANALYSIS §4).
- Bị chặn: StatusBadge/Agent Rail, CF-009, CF-011, CF-036, Q-13. Đảo ngược: dễ (lớp hiển thị). Đóng khi duyệt: CF-009, CF-036 (cùng D-016).

### D-018 Agent Rail + theme token kiểu herdr (ADR-0018) — sâu
- Câu hỏi owner: "Bạn đồng ý thêm Agent Rail (sidebar hai vùng: projects trên, agents dưới, kéo chia tỉ lệ) và cấu trúc token theme (preset + `theme.custom` + auto_switch), giữ accent AI tím, bổ sung vào NAVIGATION_STRUCTURE (sidebar chính giữ nguyên)?"
- Phương án: A) như đề xuất (P: ngữ cảnh agent ở mọi màn hình; N: layout hai vùng phức tạp trên mobile/tablet, rail chiếm không gian màn nhỏ, âm thanh thông báo cần kiểm license). B) Rail tùy chọn (P: ít xâm lấn; N: mất ngữ cảnh mặc định). C) giữ sidebar một vùng (loại theo DECISION_LOG).
- Khuyến nghị: A nhưng xem Rail là panel có thể ẩn, mặc định hiện trên màn hình lớn.
- Yếu tố quyết định: kích thước màn hình mục tiêu; mức chấp nhận thay đổi nhiều tài liệu design (HERDR_DESIGN_INTEGRATION §9).
- Chi phí/bảo mật: chi phí thiết kế/FE; không bảo mật.
- Bằng chứng: Đã xác minh: palette và bố cục herdr đọc từ `src/app/state.rs` và screenshot (nhãn ĐX); Giả định: chưa có kiểm thử người dùng, đồng bộ 20 token herdr với 8 status.
- Bị chặn: layout app shell (Phase 1-2). Đảo ngược: dễ thành tùy chọn. Đóng khi duyệt: không có CF.

### D-019 Điều khiển bởi agent: `AIW_*`, CLI `aiw`, token API riêng theo agent (ADR-0019) — sâu
- Câu hỏi owner: "Bạn cho phép mỗi agent nhận biến môi trường `AIW_*` và token API riêng (hết hạn theo task, scope tối thiểu) để gọi CLI `aiw` báo trạng thái/tạo task con/chờ blocked? Việc này mở thêm bề mặt tấn công (token trong env của agent)."
- Phương án: A) CLI `aiw` + token per-agent, Phase 3 (P: báo cáo có cấu trúc đáng tin hơn suy luận từ output, `source` + `seq` đơn điệu; N: token lộ qua env/`docker inspect`, agent giả mạo trạng thái nếu token bị đánh cắp, nested execution có thể lặp, phải duy trì CLI ổn định). B) kênh một chiều qua stdout (P: không token; N: ít khả năng). C) dùng MCP khi provider chuẩn hóa (P: chuẩn; N: chưa chín). D) không làm (P: không bề mặt mới; N: mất báo cáo trạng thái có cấu trúc).
- Khuyến nghị: A ở Phase 3, kèm redaction token, listener hook riêng với `AIW_HOOK_TOKEN`, chặn nested execution, và kiểm tra bảo mật trước khi bật; chưa cần ở Phase 1-2.
- Yếu tố quyết định: mức chấp nhận thêm bề mặt tấn công trong khi R1/R2 đã tồn tại (token trong env của process cùng user); giá trị của trạng thái có cấu trúc.
- Chi phí/bảo mật: rủi ro bảo mật tăng; chi phí duy trì CLI.
- Bằng chứng: Giả định: đề xuất Phase 3 (HERDR_DESIGN_INTEGRATION §6); Đã xác minh: mô hình điều khiển của herdr đọc từ mã (HERDR_ANALYSIS §7), endpoint hook riêng trong BACKEND_ARCHITECTURE.
- Bị chặn: không chặn Phase 1-2; chặn skill `aiw` và báo cáo `source/seq` ở Phase 3. Đảo ngược: dễ (tính năng thêm). Đóng khi duyệt: không có CF.

### D-020 herdr chỉ là tham chiếu thiết kế (ADR-0020) — sâu
- Câu hỏi owner: "Bạn đồng ý herdr (Apache-2.0) chỉ là tham chiếu thiết kế (ý tưởng, token, dữ liệu manifest TOML kèm NOTICE); `HerdrRuntime` và terminal tương tác chỉ là tùy chọn sau PoC (Phase 2 hoặc 7); không dùng mã Rust; heuristic màn hình không thay sự kiện SDK?"
- Phương án: A) tham chiếu, runtime tùy chọn sau PoC (P: dữ liệu đáng tin hơn vì app sở hữu phiên; N: MVP không có runtime terminal sẵn). B) dùng herdr làm runtime chính (loại: lệch kiến trúc, heuristic quan sát). C) bỏ hoàn toàn tham chiếu (P: gọn; N: mất mẫu thiết kế đã chọn).
- Khuyến nghị: A.
- Yếu tố quyết định: có cần terminal tương tác trong MVP không (nếu có, lùi lịch).
- Chi phí/bảo mật: không chi phí; cần kiểm license asset âm thanh; herdr 0.9.5 đang chạy trên máy dev, tránh nhầm là phụ thuộc chính thức.
- Bằng chứng: Đã xác minh: herdr 0.9.5 và socket thấy trên máy, license Apache-2.0, repo không có NOTICE gốc (HERDR_ANALYSIS §8.2, §9; QA §3 #3); Giả định: bảng tái dùng; hành vi socket Windows chưa thử.
- Bị chặn: không chặn MVP. Đảo ngược: rất dễ. Đóng khi duyệt: không có CF.

## 4. Các quyết định Q-01..Q-15 (từ mâu thuẫn `OWNER_DECISION`)

Gom theo chủ đề. Mỗi Q tham chiếu CF trong `docs/adr/CONFLICTS_TO_RESOLVE.md` (chi tiết phương án ở đó). Cột "Mặc định đề xuất" là khuyến nghị, không phải quyết định. 40 mâu thuẫn còn lại (7 giải bằng bằng chứng, 33 đồng bộ tài liệu) chỉ là đồng bộ tài liệu theo D-xxx và không cần Q riêng.

### Nhóm A. Bảo mật và sandbox

| Q | CF | Câu hỏi cho owner | Mặc định đề xuất | Chặn | Owner response |
|---|---|---|---|---|---|
| Q-01 | CF-043 | Bạn ký xác nhận bằng văn bản rủi ro R1, R2, R3, R5, R6 của D-007 (CLI chạy trên host, không có OS-level sandbox; token OAuth đọc được bởi process cùng user; prompt injection không chặn triệt để; Docker Desktop là root-equivalent; egress bằng proxy) hay chọn Tier 2 làm mặc định? | Ký (kèm fail-closed khi daemon tắt) | Phase 4 sandbox | OWR-1: xác nhận có điều kiện (APPROVED_CONDITIONAL); Owner chưa tick từng ô checklist; OWR-2 không đổi |
| Q-02 | CF-024 | Giữ chế độ ACP của OpenHands tắt ở MVP (OpenHands dùng API key riêng trong keychain, không đọc `auth.json`)? | Tắt | Tier 2 / OpenHands | OWR-2 Q-02: APPROVED. Không tích hợp OpenHands ACP trong MVP |
| Q-03 | CF-008 | Duyệt thang quyền agent L0-L5 ánh xạ sang `permission_level` (`read_only`, `standard`, `elevated`) + `permission_profile`, và cấm hẳn giá trị `full` bằng CHECK? | Duyệt | Contract freeze P1-04 | OWR-2 Q-03: APPROVED. L0..L5, `permission_level` + `permission_profile`, không `full` |
| Q-04 | CF-017 | Agent tự merge vào `integration/*` sau kiểm diff + review L3; merge vào `main` luôn cần bạn duyệt; Lead agent không duyệt thay bạn; `dependency_add` hỏi ở project chưa tin cậy, allow + audit ở project tin cậy. Đồng ý? | Đồng ý | PERMISSION_MODEL, Git Service | OWR-2 Q-04: APPROVED. Merge `integration/*` khi qua gate; `main` luôn cần Owner |
| Q-05 | CF-007 | Chờ duyệt: hook giữ process tối đa 540 s rồi task chuyển `WAITING_HUMAN` và chạy lại khi bạn duyệt (tốn thêm một lần spawn); nhắc sau 24 giờ; deploy production/xóa dữ liệu/merge main luôn duyệt từng lần, hết hạn sau 15 phút. Đồng ý? | Đồng ý | Approval engine, workflow Phase 3 | OWR-2 Q-05: APPROVED. 540 s, WAITING_HUMAN, nhắc 24 h, 15 phút cho hành động quan trọng |
| Q-06 | CF-006 | Audit log giữ online bao lâu: 365 ngày + export ký thêm 2 năm, hay 180 ngày? | 365 ngày | Migration audit, retention job | OWR-2 Q-06: APPROVED. 365 ngày online, export ký 2 năm |

### Nhóm B. Chi phí và quota

| Q | CF | Câu hỏi cho owner | Mặc định đề xuất | Chặn | Owner response |
|---|---|---|---|---|---|
| Q-07 | CF-044 | Bạn xác nhận gói Claude (`team`) và ChatGPT cho phép nhiều phiên song song theo điều khoản; chấp nhận mặc định 5 agent, nâng 10/18 chỉ sau khi đo thực? | Xác nhận + mặc định 5 | Phase 7, D-010 | OWR-2 Q-07: APPROVED chỉ mặc định 5 của ứng dụng; KHÔNG phải xác nhận subscription; ToS còn mở (PENDING_OWNER_CLARIFICATION) |
| Q-08 | CF-039 | Khi một provider hết quota, hệ thống chỉ báo và chờ bạn chuyển thủ công, hay tự chuyển task sang provider khác? | Không auto-failover | Scheduler | OWR-2 Q-08: APPROVED. Không auto-failover, thông báo Owner và chờ |
| Q-09 | CF-031 | Đổi FR-109: hệ thống chấm điểm chọn agent bằng công thức xác định (không tốn token), PM agent chỉ can thiệp khi bế tắc? | Đồng ý đổi | Scheduler/assignment | OWR-2 Q-09: APPROVED. FR-109 công thức deterministic |

### Nhóm C. Môi trường và phạm vi

| Q | CF | Câu hỏi cho owner | Mặc định đề xuất | Chặn | Owner response |
|---|---|---|---|---|---|
| Q-10 | CF-045 | Bạn sẽ bật Docker Desktop và cài `@openai/codex` để chạy kiểm chứng (SB-00..SB-17, live test Codex) trước Phase 4? Nếu không, Codex ra khỏi MVP? | Bật Docker (bắt buộc); Codex tùy quota | Phase 4, D-005, D-007 | OWR-2 Q-10: APPROVED. Docker cần cho sandbox validation; Codex CLI chưa cài |
| Q-11 | CF-029 | FR-113 nói 8 loại node, yêu cầu mới nói 9. Chấp nhận palette UI 9 loại (cập nhật FR-113) trong khi engine có 11 kind? | Chấp nhận | Workflow editor | OWR-2 Q-11: APPROVED. 9 UI node types, 11 engine kinds |

### Nhóm D. Quy ước hợp đồng (liên quan P1-04)

| Q | CF | Câu hỏi cho owner | Mặc định đề xuất | Chặn | Owner response |
|---|---|---|---|---|---|
| Q-12 | CF-003 | FR-117 dùng tên event UPPER_SNAKE. Chấp nhận lưu nội bộ dạng dotted lowercase và chỉ hiển thị/xuất API dạng UPPER_SNAKE, hay muốn UPPER_SNAKE cả trong DB? | Dotted trong DB + ánh xạ UPPER_SNAKE | Contract freeze P1-04 | OWR-2 Q-12: APPROVED. DB dotted lowercase, API có thể UPPER_SNAKE |
| Q-13 | CF-011 | Giữ `task_status` HOA (theo spec của bạn) và enum khác chữ thường, hay thống nhất một kiểu? | Giữ như hiện tại | Contract freeze P1-04 | OWR-2 Q-13: APPROVED. `task_status` UPPERCASE, enum khác lowercase |

### Nhóm E. Sản phẩm và pháp lý

| Q | CF | Câu hỏi cho owner | Mặc định đề xuất | Chặn | Owner response |
|---|---|---|---|---|---|
| Q-14 | CF-022 | Agent idle có tự đi tới phòng Relax/Coffee không (chỉ theo sự kiện thật, tắt mặc định)? | Tắt mặc định, bật bằng tùy chọn | Office behavior (Phase 5) | OWR-2 Q-14: APPROVED. Không tự đi coffee/relax mặc định, setting tùy chọn |
| Q-15 | CF-026 | Yêu cầu gốc là asset chỉ CC0. Cho phép thêm MIT và OFL (font) kèm file license và THIRD_PARTY_NOTICES; CC-BY chỉ khi bạn duyệt từng gói? | Cho phép MIT/OFL, CC-BY duyệt từng gói | Bundle asset | OWR-2 Q-15: APPROVED. CC0 mặc định, OFL kèm license/NOTICE, CC-BY duyệt từng package |

## 5. "Quyết định trước": thứ tự theo mức mở khóa công việc

Cơ sở: số công việc/phase bị chặn nếu chưa quyết. Mỗi đợt có thể trả lời cùng lúc.

**Đợt 1 (mở khóa khởi tạo Phase 1)**
1. **D-001** (monorepo, pnpm thuần): chặn P1-01 và mọi tooling; trả lời "có" mở khóa repo.
2. **D-002** (PostgreSQL + Drizzle): chặn schema và contract freeze; nên duyệt kèm "spike partition trước khi khóa".
3. **D-014 + Q-12 + Q-13 + Q-03** (hợp đồng event/enum/quyền): chặn contract freeze P1-04 (event naming, outbox, casing, thang quyền). Gom một lượt để freeze một lần.
4. **D-010 + Q-07** (concurrency 5/18, quota): cấu hình seed và scheduler; rẻ, đảo ngược dễ, nhưng Q-07 cần owner kiểm điều khoản.

**Đợt 2 (mở khóa lõi provider và workflow, trước Phase 2-3)**
5. **D-004** (Claude Agent SDK, hai tầng duyệt): mở khóa adapter chính và viết lại SECURITY §5.1.
6. **D-003** (Conductor) cùng **Q-05** (timeout approval) và **Q-11** (loại node): mở khóa Phase 3.
7. **D-008, D-009, Q-04** (credential, worktree, policy merge): mở khóa Git Service và permission engine.

**Đợt 3 (rủi ro cao, chốt trước Phase 4)**
8. **D-007 + Q-01 + Q-10** (sandbox hai tầng, ký R1,R2,R3,R5,R6, bật Docker): quyết định rủi ro lớn nhất; nên đưa sớm vì nếu owner từ chối thì Tier 2 thành mặc định và đổi D-004, D-006, D-008, chi phí token. Về mặt thời gian có thể trả lời đợt 3, nhưng nên đọc sớm vì tác động đảo.
9. **D-005, D-006, Q-02** (Codex, OpenHands): có thể hoãn, quyết định có đưa Codex vào MVP không.

**Đợt 4 (không chặn)**: D-011 + Q-14 (Office), D-012 + Q-15 (asset), D-013, D-015..D-020, Q-06, Q-08, Q-09.

**Năm quyết định nên lấy đầu tiên**: D-001, D-002, D-014 (cùng Q-12/Q-13/Q-03), D-004 (cùng Q-07/D-010), D-007 (cùng Q-01).

## 6. Cách ghi nhận một quyết định

1. Owner chỉ ghi vào cột "Owner response" của mục tương ứng (bảng §2 cho D-xxx, bảng §4 cho Q-xx) hoặc trả lời trực tiếp cho người giữ register. Nội dung bắt buộc: `APPROVED` hoặc `REJECTED`, ngày (YYYY-MM-DD), và nếu chọn khác khuyến nghị thì ghi phương án thay thế. Ví dụ: `APPROVED 2026-11-02, phương án A`; `REJECTED 2026-11-02, chọn phương án B (Electron)`.
2. Không agent nào được tự điền cột này, đổi trạng thái sang APPROVED/REJECTED, hay suy ra "đã đồng ý" từ việc owner im lặng hoặc từ việc một tài liệu đã viết theo đề xuất.
3. Chỉ sau khi owner ghi quyết định, agent đồng bộ tài liệu theo thứ tự: (a) cột "Phê duyệt" trong `DECISION_LOG.md` và trạng thái trong ADR tương ứng; (b) áp các chỉnh sửa có điều kiện `SAU D-xxx` trong `docs/adr/CONFLICTS_TO_RESOLVE.md` (đổi trạng thái dòng CF đó thành `ALREADY_FIXED` sau khi sửa); (c) cập nhật `PROJECT_STATUS.md`, `KNOWN_ISSUES.md`, `NEXT_SESSION.md`.
4. Nếu owner `REJECTED`: agent lập phương án thay thế dựa trên mục "Điều kiện đảo ngược" của ADR, không sửa tài liệu theo đề xuất cũ, và đưa lại một mục PROPOSED mới.
5. Các chỉnh sửa có điều kiện `NGAY` (CF-005, CF-025, CF-030, CF-034, CF-037, CF-040, CF-049) không phụ thuộc owner và có thể áp dụng bất cứ lúc nào; các thay đổi này vẫn không có nghĩa là bất kỳ quyết định nào đã được duyệt.
6. Trước khi bắt đầu Phase 1, tối thiểu phải có APPROVED cho D-001 và D-002 và quyết định về contract freeze (D-014, Q-12, Q-13, Q-03). Phase 1 chưa được bắt đầu trong phiên này.

## 7. Quyết định bổ sung sau khi sổ này được lập

### D-021 — Skill Overlay Directory (nạp skill Markdown động theo agent)
- **Trạng thái (lịch sử lúc lập):** PENDING_OWNER_APPROVAL; **hiện hành: APPROVED_CONDITIONAL theo OWR-2 Q-18 (xem mục 9.3)**. Nguồn: yêu cầu của Owner trong phiên 2026-10-30; đặc tả ở `docs/architecture/DYNAMIC_SKILL_INJECTION.md`.
- **Câu hỏi cho Owner:** có duyệt cơ chế Owner chỉnh file Markdown trong `.ai/skills/<agent-slug>/`, quét và nạp lúc giao mỗi task, không hot-reload giữa task, overlay không cấp quyền, agent không được sửa `.ai/**` hay không.
- **Phương án:** (A) Overlay trên đĩa cộng Library (đề xuất); (B) chỉ Library, chỉnh qua giao diện; (C) hot-reload giữa task.
- **Khuyến nghị:** A. C bị loại vì Claude Code mặc định giữ nguyên system prompt của hội thoại (`--system-prompt-snapshot`), nên hot-reload không đạt được nếu không đổi cấu hình và làm tăng chi phí.
- **Bằng chứng:** cờ `--append-system-prompt[-file]`, `--add-dir`, `--system-prompt-snapshot` có trong `claude --help` bản 2.1.312 (Đã xác minh ngày 2026-10-30). Hành vi thực tế khi sửa file giữa hai task và khi resume **chưa kiểm thử**: cần PoC (bước P0 mục 6 của đặc tả).
- **Cần Owner quyết thêm:** vị trí thư mục (trong repo dự án hay ngoài), cách xử lý resume (a hoặc b), có cho `references/` và `scripts/` không, giới hạn kích thước, FR-293..FR-298.
- **Bị chặn nếu chưa duyệt:** triển khai bộ quét overlay, thay đổi schema `agents.skills_config`, thêm `.ai/**` vào danh sách forbidden mặc định.
- **Khả năng đảo ngược:** cao ở mức thiết kế, vì chưa có mã.


## 8. Phản hồi của Owner, ghi nhận ngày 2026-10-30 (OWR-1, lịch sử; trạng thái hiện hành ở mục 9)
Nguồn: tin nhắn của Owner trong phiên làm việc, nhóm "Phản hồi các Quyết định Kiến trúc Trọng tâm". Ghi lại trung thực. Chỉ đánh dấu APPROVED cho phần khớp rõ ràng với nội dung của quyết định trong sổ này. Phần nhãn không khớp hoặc chưa trả lời được giữ PENDING_OWNER_APPROVAL và nêu câu hỏi xác nhận.

### 8.1 Bảng ghi nhận
| Mục | Owner nói | Ghi nhận | Trạng thái |
|---|---|---|---|
| D-001 | "Phê duyệt theo đề xuất hiện tại (Next.js/NestJS/TypeScript/Claude Code làm nền tảng cốt lõi)" | Duyệt: pnpm monorepo TypeScript, NestJS, web app cộng backend Node cục bộ ở MVP, Tauri 2 sau, pnpm thuần không Turborepo, Claude Code làm provider nền. **Không khớp:** đề xuất ghi frontend là React 19 + Vite, không phải Next.js. Khung frontend chưa được chốt, xem Q-16 | ~~APPROVED (trừ khung frontend, Q-16)~~ (giữ làm lịch sử; thay bằng APPROVED đầy đủ theo OWR-2 Q-16, xem mục 9) |
| D-002 | "Cho phép sử dụng Drizzle ORM" (nhóm D-004 và D-010) | Duyệt Drizzle, PostgreSQL 16, Redis, event store phân vùng. Giữ khuyến nghị trong mục D-002: ghi đúng phiên bản 0.45.x và chạy spike partition và FK vòng trước khi khóa schema | APPROVED |
| D-010 | "concurrency limit = 5 như thiết kế" | Duyệt mặc định 5, trần cấu hình 18, nâng 5, 10, 18 chỉ sau khi đo thực. **Chưa trả lời:** xác nhận điều khoản subscription (Q-07) | APPROVED (Q-07 còn mở) |
| D-007 | "Phê duyệt mô hình 2 tầng", chấp nhận rủi ro agent ghi vào overlay qua Bash trên host ở Phase 0 và 1, giảm bằng checksum và audit log, rồi chuyển sang Linux/Docker ở phase sau | Duyệt sandbox hai tầng. Điều kiện của Owner: chấp nhận rủi ro ghi overlay ở Phase 0 đến 1 với phát hiện bằng checksum và audit. Điều kiện trong khuyến nghị D-007 vẫn áp dụng: fail-closed khi Docker tắt, test SB-00..SB-17 phải qua trên máy có Docker trước khi mở quyền thực thi. Phase 1 không cho agent chạy thật (xem 8.3) | APPROVED, có điều kiện |
| D-004 | Gọi là "Kiến trúc Database và Quản lý State/Event" | **Không khớp.** D-004 trong sổ là điều khiển Claude Code bằng Agent SDK (cổng `canUseTool` và hook, so với spawn CLI thô) và yêu cầu xác nhận điều khoản gói `team`. Nội dung Owner mô tả (Drizzle, event naming, concurrency) thuộc D-002, Q-12, D-010. Owner có nhắc "Claude Code làm nền tảng cốt lõi" nhưng chưa chọn SDK hay spawn CLI, chưa trả lời điều khoản | PENDING_OWNER_APPROVAL, cần xác nhận (Q-17) |
| D-014 | Gọi là "Cổng approval và cơ chế phân quyền Claude"; duyệt phương án có Human-in-the-loop cho hành động quan trọng trước khi giao tự trị | **Không khớp.** D-014 trong sổ là event system (outbox, Redis, replay). Phần Owner nói được ghi nhận như **chỉ thị chính sách**: hành động quan trọng cần duyệt thủ công, chưa trao tự trị hoàn toàn. Chỉ thị này khớp với CF-007, CF-013, POL-11, POL-12 và Q-04, Q-05 | D-014 PENDING; chỉ thị HITL được ghi nhận |
| Event naming | "giữ nguyên event naming chuẩn hóa" | Hiểu là giữ tên UPPER_SNAKE theo yêu cầu gốc (FR-117). **Chưa rõ** lưu nội bộ dạng dotted lowercase rồi ánh xạ, hay UPPER_SNAKE cả trong DB (Q-12, Q-19) | Q-12 trả lời một phần |
| Rủi ro bảo mật | "Thừa nhận và chấp nhận có điều kiện 13 rủi ro Tier 1" | Ghi nhận là xác nhận bằng văn bản có điều kiện cho Q-01 (R1, R2, R3, R5, R6) và 13 rủi ro NOT_ACCEPTABLE. Điều kiện ở 8.3. Tôi chưa tick từng ô trong checklist ký duyệt: các ô do Owner tự ký | Q-01 xác nhận có điều kiện |
| Bốn FR bắt buộc | Owner đề xuất BDD cho "FR-122, FR-181, FR-182, FR-183" | **Không khớp mã FR.** Trong `FUNCTIONAL_REQUIREMENTS.md`: FR-122 là deliverable Stage 3 của workflow, FR-181 là định nghĩa "sự cố" của dashboard, FR-182 là công thức "tiến độ", FR-183 là chỉ số "hiệu suất". Ba tiêu chí Owner đưa (nạp skill động, phát hiện sửa overlay trái phép, resume và quota) thuộc tính năng D-021, không phải bốn FR này. Ghi nhận ở 8.2. Bốn FR cũ vẫn NEEDS_OWNER_CLARIFICATION | Xem 8.2, Q-18 |
| Commit | Khuyến nghị commit gom nhóm | Cây làm việc đã sạch: Owner đã commit và đẩy lên main (commit `1aec2b3`, trước tin nhắn này). Không có gì để commit. Quét bí mật trên các file đã track: không có | Đã hoàn tất bởi Owner |

### 8.2 Ba tiêu chí Owner cung cấp cho tính năng skill động (D-021)
Ghi nhận ý Owner làm yêu cầu cho D-021. D-021 **vẫn PENDING_OWNER_APPROVAL** vì Owner chưa nói rõ duyệt D-021 và mã FR chưa khớp.

| Mã | Given | When | Then (theo Owner) | Ánh xạ tạm | Ghi chú |
|---|---|---|---|---|---|
| OWN-AC-1 | Quản trị viên đã sửa hoặc thêm file Markdown vào `.ai/skills/<agent_name>/` | Hệ thống kích hoạt task kế tiếp cho agent đó | Nội dung file được tự động đọc và đưa vào ngữ cảnh Claude Code, không khởi động lại ứng dụng, không biên dịch lại | FR-293, FR-294 đề xuất; AC-501, AC-502 | Khớp đặc tả: quét lúc giao task |
| OWN-AC-2 | Agent chạy lệnh Bash trên host Windows | Có thay đổi file overlay ngoài luồng phân quyền chuẩn | Hệ thống ghi log cảnh báo, đánh dấu thay đổi là nghi vấn, **chặn agent chạy tiếp cho đến khi người quản lý xác nhận** | FR-296 đề xuất; AC-513 | **Mạnh hơn đặc tả trước đó**, vốn chỉ cảnh báo và cách ly nội dung. Đặc tả đã cập nhật. Hệ thống chỉ biết thời điểm đổi, không biết ai sửa (cùng một người dùng hệ điều hành), nên chặn dựa trên "đổi trong lúc agent chạy" |
| OWN-AC-3 | Phiên Claude Code bị gián đoạn (kể cả hết hạn mức 5 giờ) hoặc cần resume | Hệ thống nạp lại tiến trình | Kiểm tra thay đổi system prompt hoặc skill, nếu lệch thì **ép làm mới ngữ cảnh** thay vì giữ system prompt cũ | AC-511 | Chọn hướng "ép làm mới" thay cho chỉ gửi thông điệp cập nhật. Cách kỹ thuật (tắt snapshot hay mở hội thoại mới) vẫn cần PoC. Hết hạn mức là sự kiện của provider, không do hệ thống skill tạo ra |

### 8.3 Điều kiện của Owner cho việc chấp nhận rủi ro bảo mật
1. **Phase 1:** tuyệt đối không cho agent thực thi lệnh ghi dữ liệu hệ thống hoặc gọi external API thực. Chỉ chạy mô phỏng, dry-run, hoặc qua các runbook kiểm thử thủ công.
2. **Đầu Phase 2:** xác thực API, hardening Docker sandbox và audit log phải được kiểm tra khắt khe **trước khi** cấp quyền tự trị rộng hơn.
3. Rủi ro agent ghi vào overlay chấp nhận ở Phase 0 đến 1, giảm bằng checksum và audit log để phát hiện, trước khi chuyển sang môi trường Linux/Docker ở phase sau.

Hệ quả cho roadmap: cổng vào Phase 2 cần thêm ba hạng mục trên. Đã ghi vào `docs/IMPLEMENTATION_ROADMAP.md` (phụ lục) và `PHASE_0_REMAINING_RISKS.md`.

Về các runbook: Owner nhắc "27 runbook kiểm thử thủ công" như một cách chạy an toàn. **Chưa coi là cho phép** cài Codex CLI hay bật Docker. Cần Owner nói rõ nếu muốn tôi chạy, hoặc tự chạy theo runbook.

### 8.4 Câu hỏi cần Owner xác nhận sau phản hồi này
| Q | Câu hỏi |
|---|---|
| Q-16 | Khung frontend là Next.js (như Owner viết) hay React + Vite (như đề xuất đã duyệt)? Ảnh hưởng `FRONTEND_ARCHITECTURE.md`, mô hình triển khai (Next.js cần server riêng hoặc chế độ export) và Tauri sau này |
| Q-17 | Xác nhận D-004: dùng Claude Agent SDK (đề xuất) thay vì spawn CLI thô? Và xác nhận điều khoản gói `team` cho tự động hóa (Q-07) |
| Q-18 | Xác nhận D-021 (skill overlay) kèm OWN-AC-1..3. Và định nghĩa thật của bốn FR bắt buộc: FR-122 (deliverable Stage 3), FR-181 ("sự cố"), FR-182 ("tiến độ"), FR-183 ("hiệu suất") |
| Q-19 | Event naming: lưu trong DB dạng dotted lowercase rồi ánh xạ UPPER_SNAKE khi hiển thị (đề xuất), hay UPPER_SNAKE cả trong DB (Q-12)? |
| Q-20 | D-003 (workflow engine tự xây trên BullMQ) có được duyệt cùng đợt không? Owner gọi nhóm "công cụ điều phối chính" nhưng không nêu D-003 |


## 9. Phản hồi OWR-2 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md` (OWR-2, nội dung Owner chép nguyên văn). Mục 8 (OWR-1) được giữ nguyên làm lịch sử; **trạng thái hiện hành là mục 9 này** và bảng mục 2. Ngày ghi nhận: 2026-10-30.

**Không hiểu rằng Phase 0 đã hoàn tất hoặc Phase 1 đã được cho phép.** OWR-2 §8 ghi rõ: Phase 0 vẫn đang mở, Phase 1 chưa được Owner cho phép. Không mục nào dưới đây là VERIFIED: chưa có PoC report hay bằng chứng kiểm thử nào (thư mục `docs/poc/` chưa tồn tại khi ghi nhận). Việc đồng bộ các tài liệu khác do agent khác thực hiện; cột "Tài liệu bị ảnh hưởng" chỉ liệt kê, không khẳng định đã đồng bộ.

Từ vựng bổ sung: **APPROVED_CONDITIONAL** = Owner đã duyệt nhưng một điều kiện nêu rõ chưa đạt; **PENDING_OWNER_CLARIFICATION** = Owner chưa trả lời hoặc câu trả lời để hở một điểm, agent không tự điền; **VERIFIED** = chỉ khi có bằng chứng kiểm thử (hiện không có mục nào).

### 9.1 Q-16..Q-20 (câu hỏi xác nhận từ mục 8.4)

| ID | Quyết định của Owner (rút gọn) | Trạng thái ghi nhận | Điều kiện | Nguồn OWR-2 | Tài liệu bị ảnh hưởng | Câu hỏi còn lại (PENDING_OWNER_CLARIFICATION) |
|---|---|---|---|---|---|---|
| Q-16 | React 19 + Vite, NestJS backend, monorepo TypeScript, pnpm workspace. Không chuyển sang Next.js trong MVP | APPROVED (đã trả lời) | Không | §1 Q-16 | FRONTEND_ARCHITECTURE.md, ADR-0001, DECISION_LOG D-001, development/DEPLOYMENT.md | Không |
| Q-17 | Claude Agent SDK là adapter chính cho Claude; không dùng raw CLI spawn làm kiến trúc chính; CLI chỉ cho tác vụ hỗ trợ, kiểm tra hoặc PoC được cấp phép | APPROVED (đã trả lời) | Phải phân biệt subscription authentication, API authentication và usage limits; không mặc định Agent SDK nghĩa là số session không giới hạn | §1 Q-17 | ADR-0004, PROVIDER_INTEGRATION.md, SECURITY.md §5.1, research/CLAUDE_CODE_INTEGRATION.md | Điều khoản dịch vụ cho tự động hóa vẫn chưa xác nhận (xem Q-07) |
| Q-18 | Chấp thuận có điều kiện D-021 với OWN-AC-1..3 | APPROVED_CONDITIONAL (D-021) | Chỉ là chấp thuận có điều kiện cho đến khi OWN-AC-1, OWN-AC-2, OWN-AC-3 được kiểm thử thành công (xem 9.3, D-021) | §1 Q-18, §4, §5.5 | ADR (không có ADR-0021 trong phạm vi file này), architecture/DYNAMIC_SKILL_INJECTION.md, ACCEPTANCE_CRITERIA.md | Bốn FR **FR-122, FR-181, FR-182, FR-183** chỉ là đề xuất Acceptance Criteria của Owner, **không phải APPROVED AC** và không phải trích nguyên văn FR gốc (OWR-2 §4, §7). Cần hỏi tiếp: FR-122: artifact nào bắt buộc, có yêu cầu export/download hay chỉ lưu/truy xuất trong workspace; FR-181: approval chờ lâu là incident hay chỉ operational alert, và quy tắc grouping incident lặp lại; FR-182: cách xử lý `cancelled` và có chấp thuận equal-weight không; FR-183: dashboard nhiều metric hay một KPI tổng hợp (nếu KPI thì cần công thức và trọng số được phê duyệt) |
| Q-19 | Database event dùng `lowercase.dot.notation` (ví dụ `task.started`, `task.completed`, `agent.blocked`). API hoặc lớp tương thích bên ngoài có thể ánh xạ sang `UPPER_SNAKE_CASE`. Không tạo hai hệ thống event naming độc lập | APPROVED (đã trả lời) | Chỉ một hệ thống tên; ánh xạ 1-1 | §1 Q-19, §2 Q-12 | EVENT_SYSTEM.md (M-E2), DATABASE_DESIGN.md §5, VIRTUAL_OFFICE.md, ADR-0014, FR-117 hiển thị | Không (ánh xạ ở `@aiw/contracts` là việc thiết kế, không phải câu hỏi Owner) |
| Q-20 | Chấp thuận định hướng BullMQ + PostgreSQL cho Workflow Conductor | APPROVED_CONDITIONAL | Chưa khóa implementation cho đến khi PoC chứng minh: crash recovery; idempotency; không chạy lặp node ngoài ý muốn; retry an toàn; approval state lưu bền vững; worker không bị giữ lúc chờ Owner approval. Nếu không đạt, báo cáo phương án thay thế (Temporal hoặc LangGraph) để Owner quyết định | §1 Q-20 | ADR-0003, WORKFLOW_ENGINE.md, ORCHESTRATION_WORKFLOW_RESEARCH.md | Không (kết quả PoC chưa có) |

### 9.2 Q-01..Q-15

| ID | Quyết định của Owner (rút gọn) | Trạng thái ghi nhận | Điều kiện | Nguồn OWR-2 | Tài liệu bị ảnh hưởng | Câu hỏi còn lại |
|---|---|---|---|---|---|---|
| Q-01 | OWR-1 (mục 8.1, 8.3): xác nhận bằng văn bản **có điều kiện** R1, R2, R3, R5, R6. OWR-2 không thay đổi | APPROVED_CONDITIONAL | Điều kiện OWR-1 mục 8.3 (Phase 1 không chạy agent thật; đầu Phase 2 kiểm tra xác thực API, hardening Docker, audit). OWR-2 §8: fail closed nếu sandbox không hoạt động, không mount credential directory hoặc Docker socket vào Agent, không đưa Agent production-capable vào trước khi authentication, authorization, permission model, credential isolation, audit, sandbox, emergency stop hoàn thành | OWR-1 §8; OWR-2 §8 | ADR-0007, SECURITY.md, security/SANDBOX_DESIGN.md | Các ô checklist ký duyệt do Owner tự ký, chưa được tick (OWR-1) |
| Q-02 | Không tích hợp OpenHands ACP trong MVP | APPROVED | Không | §2 Q-02 | ADR-0006, PROVIDER_INTEGRATION.md §9.5 (CF-024) | Không |
| Q-03 | Permission level L0 đến L5, dùng `permission_level` và `permission_profile`. Không hỗ trợ quyền `full` | APPROVED | Không | §2 Q-03 | DATABASE_DESIGN.md:170 (enum còn `full`), PERMISSION_MODEL.md, USER_ROLES_AND_PERMISSIONS.md, AGENT_ARCHITECTURE.md | Không |
| Q-04 | Agent merge vào `integration/*` khi vượt các gate. Merge vào `main` luôn cần Owner approval. Lead Agent không thay thế quyền phê duyệt của Owner | APPROVED | Phải qua các gate (kiểm diff, review) | §2 Q-04 | USER_ROLES_AND_PERMISSIONS.md POL-11/POL-12, PERMISSION_MODEL.md R030/R040/R050, ADR-0009 | Không |
| Q-05 | Approval wait timeout tối đa 540 giây; hết thời gian thì lưu trạng thái và chuyển `WAITING_HUMAN`, không giữ worker; có thể nhắc Owner sau 24 giờ; production deploy, data deletion, merge main cần approval riêng, hết hiệu lực sau 15 phút | APPROVED | Không | §2 Q-05 | SECURITY.md:400, BACKEND_ARCHITECTURE.md:149, DATABASE_DESIGN.md:832, PROVIDER_INTEGRATION.md:411, RESOURCE_MANAGEMENT.md:104, USER_ROLES_AND_PERMISSIONS.md §5.5, WORKFLOW_ENGINE.md:201,359 | Không |
| Q-06 | Audit log online giữ 365 ngày; signed export giữ 2 năm | APPROVED | Không | §2 Q-06 | SECURITY.md:498 (còn 180 ngày), AUDIT_LOGGING.md, DEVSECOPS_GUIDE.md | Không |
| Q-07 | Giữ concurrency mặc định 5. Đây là giới hạn mặc định của **ứng dụng**, **không phải xác nhận subscription cho phép chạy đồng thời 5 session**. Không tự tăng lên 10 hoặc 18 nếu chưa xác minh điều khoản provider | APPROVED (chỉ giá trị mặc định 5 của ứng dụng) | Không tự nâng 10/18 khi chưa xác minh điều khoản provider. Không tuyên bố giới hạn concurrency cụ thể của subscription khi chưa có nguồn xác nhận (OWR-2 §5.1) | §2 Q-07, §5.1 | ADR-0010, ADR-0004, RESOURCE_MANAGEMENT.md, MULTI_AGENT_SCALABILITY.md | Xác minh ToS/usage limit của Claude Team và ChatGPT/Codex vẫn **mở** (PENDING_OWNER_CLARIFICATION); production nhiều Agent cần provider contract/API billing hợp lệ |
| Q-08 | Không tự động failover provider khi hết quota; thông báo Owner và chờ quyết định | APPROVED | Không | §2 Q-08 | ADR-0010, RESOURCE_MANAGEMENT.md, PROVIDER_INTEGRATION.md (CF-039) | Không |
| Q-09 | FR-109 dùng công thức deterministic, không dùng LLM chỉ để tính điểm; PM Agent chỉ can thiệp khi task bị mắc kẹt | APPROVED | Không | §2 Q-09 | FUNCTIONAL_REQUIREMENTS.md FR-109, MULTI_AGENT_ORCHESTRATION.md, AGENT_ARCHITECTURE.md | Công thức cụ thể chưa được Owner nêu (việc thiết kế, không tự điền) |
| Q-10 | Docker cần thiết cho sandbox validation. Codex CLI chưa cài trong giai đoạn này | APPROVED | Không cài Codex CLI (`npm i -g @openai/codex` bị cấm, OWR-2 §5.2, §9); Docker do Owner tự mở thủ công theo quy trình OWR-2 §5.3; không ghi nhận Docker đã chạy hay sandbox đã validate | §2 Q-10, §5.2, §5.3 | ADR-0007, ADR-0005, QA/Runbook sandbox | Không |
| Q-11 | 9 UI node types và 11 engine kinds; cập nhật FR-113 đồng bộ | APPROVED | Không | §2 Q-11 | FUNCTIONAL_REQUIREMENTS.md FR-113, WORKFLOW_ENGINE.md M-W2, SCREEN_SPECIFICATION.md:471 | Không |
| Q-12 | Database dùng dotted lowercase event naming; API/output có thể ánh xạ UPPER_SNAKE_CASE | APPROVED | Cùng Q-19: một hệ thống tên | §2 Q-12 | Như Q-19 | Không |
| Q-13 | Giữ `task_status` UPPERCASE; các enum khác lowercase theo quy ước đã thống nhất | APPROVED | Không | §2 Q-13 | DATABASE_DESIGN.md §12 | Không |
| Q-14 | Agent idle không tự chuyển sang trạng thái coffee/relax mặc định. Có thể có setting tùy chọn | APPROVED | Không | §2 Q-14 | VIRTUAL_OFFICE.md:157-158, SCREEN_SPECIFICATION.md:143,678, FR-151, FR-164 | Không |
| Q-15 | Chỉ dùng asset CC0 mặc định. Cho phép font OFL nếu lưu đầy đủ license/NOTICE. CC-BY chỉ khi Owner phê duyệt từng package | APPROVED | Lưu license/NOTICE cho font OFL; CC-BY cần duyệt từng package | §2 Q-15 | ADR-0012, research/LICENSE_AND_REUSE.md, VIRTUAL_OFFICE.md §21, THIRD_PARTY_NOTICES | OWR-2 không nhắc MIT cho asset: MIT chưa được duyệt cho asset đóng gói (xem xung đột X-5). Giấy phép sprite JIK-A-4 vẫn chưa có bằng chứng lưu (CF-025) |

### 9.3 D-xxx theo OWR-2 §3 (và D-021 theo Q-18)

| ID | Quyết định của Owner (rút gọn) | Trạng thái ghi nhận | Điều kiện | Nguồn OWR-2 | Tài liệu bị ảnh hưởng | Câu hỏi còn lại |
|---|---|---|---|---|---|---|
| D-003 | Giữ chấp thuận về **định hướng** Conductor trên BullMQ + PostgreSQL | APPROVED_CONDITIONAL | Implementation **chỉ được khóa sau khi** PoC đạt các tiêu chí Q-20 (crash recovery, idempotency, không chạy lặp node, retry an toàn, approval state bền, worker không bị giữ). Không đạt thì báo phương án thay thế (Temporal, LangGraph) để Owner quyết | §3 D-003, §1 Q-20 | ADR-0003, WORKFLOW_ENGINE.md | Không |
| D-004 | Claude Agent SDK là adapter chính (Q-17) | APPROVED | Phân biệt subscription auth, API auth, usage limits; không có phiên không giới hạn | §1 Q-17 | ADR-0004, PROVIDER_INTEGRATION.md | ToS tự động hóa gói Team chưa xác nhận (Q-07) |
| D-005 | Hoãn tích hợp Codex **khỏi MVP**. Giữ abstraction `AgentAdapter` để mở rộng sau. Không cài Codex CLI hoặc live test trong phạm vi hiện tại | APPROVED (hoãn Codex khỏi MVP) | Không cài Codex CLI, không live test. **Xung đột cần Owner biết:** quyết định này THAY ĐỔI nhóm chấp nhận MVP B (Provider) của đặc tả gốc, vốn yêu cầu kết nối Codex (AC-022, FR-077 "M, MVP, [Owner]", GP-1 của ACCEPTANCE_CRITERIA). Xem xung đột X-2 | §3 D-005, §5.2 | ADR-0005, ADR-0013 (junction Codex), FUNCTIONAL_REQUIREMENTS.md FR-077, ACCEPTANCE_CRITERIA.md nhóm B, PRD.md:88, PROVIDER_INTEGRATION.md | Owner xác nhận đã biết việc đổi nhóm chấp nhận MVP B (không tự sửa FR/AC gốc) |
| D-006 | Hoãn OpenHands đến **sau Phase 4**. Không tích hợp như provider thứ ba trong MVP | APPROVED (hoãn) | Không tích hợp trong MVP; xem xung đột X-9 (Tier 2 của D-007 phụ thuộc OpenHands) | §3 D-006, §2 Q-02 | ADR-0006, ADR-0007, PRD.md:88, SRS.md:28,53 | Không |
| D-008 | App không tự ý đọc `.credentials.json` hoặc `auth.json`; không plaintext token trong DB hoặc frontend; secret khác dùng OS keychain hoặc cơ chế phù hợp; không mount thư mục credential của Owner vào Agent sandbox | APPROVED | Không | §3 D-008, §8 | ADR-0008, SECURITY.md §7, CREDENTIAL_MANAGEMENT.md | Không |
| D-009 | Mỗi task một worktree/branch; review diff trước merge; không tự merge `main`; Owner phê duyệt cuối | APPROVED | Không | §3 D-009, §2 Q-04 | ADR-0009, WORKSPACE_AND_GIT_ISOLATION.md | Không |
| D-011 | Phaser 3.90 qua abstraction `OfficeRenderer`; animation và trạng thái nhân vật phản ánh event thực, không giả lập | APPROVED_CONDITIONAL | **Benchmark trước khi khóa implementation** (chưa chạy benchmark nào) | §3 D-011 | ADR-0011, VIRTUAL_OFFICE.md, SCREEN_SPECIFICATION.md | Kích thước tile 16 so với 32 (CF-019) OWR-2 **không quyết định**: cần Owner/Frontend chốt qua benchmark; xem X-6 |
| D-012 | Tái dùng code Apache-2.0 và MIT sau khi xác minh license và attribution. My Virtual Office chỉ tham khảo ý tưởng; không sao chép code, prompt, layout hoặc asset AGPL. Asset theo Q-15 | APPROVED | Xác minh license và attribution trước mỗi lần tái dùng | §3 D-012, Q-15 | ADR-0012, LICENSE_AND_REUSE.md | Không (xem xung đột asset X-5) |
| D-013 | `SKILL.md` là chuẩn; metadata mở rộng bằng `skill.yaml`; Skill và Memory tách riêng; Memory cần cơ chế quản lý và approval | APPROVED | Không | §3 D-013 | ADR-0013, SKILL_SYSTEM.md | Không |
| D-014 | Transactional Outbox; một relay leader phát event; Redis Pub/Sub hoặc Streams cho realtime; replay bằng `lastEventId`; naming theo Q-19 | APPROVED (kiến trúc) | Không | §3 D-014, Q-19 | ADR-0014, EVENT_SYSTEM.md, DATABASE_DESIGN.md | **PENDING_OWNER_CLARIFICATION:** danh sách event của "Prompt 1 mục 8.4" **không được lưu trong repo**. Owner chỉ ra không được tự tạo danh sách. Ghi nhận: mười sáu tên event có xuất hiện trong văn bản prompt đầu tiên của Owner; Owner cần xác nhận một bản sao được lưu trong repo. Register này không chép danh sách đó và không tuyên bố danh sách event hiện tại là nội dung gốc đã phê duyệt |
| D-015 | Primary accent Purple; Inter + JetBrains Mono; density Comfortable; Agent avatar Square; Owner avatar Round; **default locale Vietnamese**; secondary locale English | APPROVED | Không | §3 D-015 | ADR-0015, DESIGN_TOKENS.md, UI_UX_GUIDELINES.md, FRONTEND_ARCHITECTURE.md, SYSTEM_ARCHITECTURE.md | Không (xem xung đột X-1: tài liệu cũ ghi tiếng Anh mặc định) |
| D-016 | 18 Agent roles là seed data, không hardcode. Tách Agent Identity, Provider Identity, User Identity | APPROVED | Không | §3 D-016 | ADR-0016, AGENT_ORGANIZATION.md | Không |
| D-017 | 5 trạng thái `blocked`, `working`, `done`, `idle`, `unknown`; chỉ suy ra từ lifecycle event và heartbeat thực; không suy diễn khi thiếu dữ liệu | APPROVED | Không | §3 D-017 | ADR-0017, HERDR_DESIGN_INTEGRATION.md | Không |
| D-018 | Agent Rail chia 2 section; cho phép ẩn/hiện; mặc định hiển thị trên màn hình lớn | APPROVED | Không | §3 D-018 | ADR-0018, HERDR_DESIGN_INTEGRATION.md, NAVIGATION_STRUCTURE | Không |
| D-019 | Agent CLI/API chỉ triển khai trong Phase 3 **sau Security Review**; short-lived token least privilege; có redaction; ngăn nested execution không được phép | APPROVED_CONDITIONAL | Security Review (chưa thực hiện) trước khi triển khai; chỉ Phase 3 | §3 D-019 | ADR-0019, HERDR_DESIGN_INTEGRATION.md §6 | Không |
| D-020 | herdr chỉ là design reference. Không tái dùng Rust runtime. Không dùng screen heuristic làm nguồn dữ liệu sự thật | APPROVED | Không | §3 D-020 | ADR-0020, HERDR_ANALYSIS.md | Không |
| D-021 | Skill Overlay Directory (Dynamic Skill Injection) | APPROVED_CONDITIONAL | Chỉ là chấp thuận có điều kiện **cho đến khi OWN-AC-1, OWN-AC-2, OWN-AC-3 được kiểm thử thành công**: OWN-AC-1 task mới tự nhận Skill mới, không restart/recompile; OWN-AC-2 sửa overlay trái phép thì audit log, đánh dấu đáng ngờ, tạm dừng Agent đến khi Manager/Owner xác nhận (hệ thống không tuyên bố xác định tuyệt đối danh tính người sửa vì cùng Windows OS user, chỉ xác định thay đổi, thời điểm, phiên thực thi, bằng chứng audit); OWN-AC-3 resume kiểm tra hash Skill/System Prompt, đổi thì tạo session mới hoặc refresh context đã được PoC chứng minh, ghi audit log. PoC trong scratch cho SK-01..SK-05 (OWR-2 §5.5): chưa chạy | §1 Q-18, §5.5 | architecture/DYNAMIC_SKILL_INJECTION.md, ACCEPTANCE_CRITERIA.md (AC-501..AC-513), SKILL_SYSTEM.md | FR-122/181/182/183 xem Q-18 (9.1) |

Các D-001, D-002, D-007, D-010 từ OWR-1 (mục 8.1) không được OWR-2 thay đổi trừ D-001:

| ID | Trạng thái hiện hành | Ghi chú |
|---|---|---|
| D-001 | **APPROVED đầy đủ** (OWR-2 Q-16) | React 19 + Vite, NestJS, TypeScript monorepo, pnpm workspace; không Next.js trong MVP. Ghi chú bảo lưu "trừ khung frontend" và ghi chú Q-16 đang chờ của OWR-1 được thay thế |
| D-002 | APPROVED (OWR-1) | OWR-2 §5.4 chỉ cho phép Drizzle PoC trong scratch (chưa chạy, chưa có kết quả); không tự đổi ORM nếu PoC thất bại |
| D-007 | APPROVED_CONDITIONAL (OWR-1, không nâng) | Điều kiện OWR-1 mục 8.3 và OWR-2 §8 (fail closed; không mount host credential directory hoặc Docker socket; không arbitrary code trên Windows host khi chưa có sandbox đạt chuẩn) |
| D-010 | APPROVED (OWR-1) | Q-07: mặc định 5 là giới hạn ứng dụng; không tự nâng 10/18; không tuyên bố 18 Agent concurrent đã được kiểm thử (OWR-2 §8) |

### 9.4 Chỉ thị phạm vi và bảo mật từ OWR-2 (ghi nhận, không đổi trạng thái D-xxx)
- §5.1: không mặc định subscription cho phép 5, 10 hoặc 18 session; không chia sẻ credential; không dùng tài khoản cá nhân làm backend multi-user trái điều khoản.
- §5.2: chưa cho phép cài Codex CLI. §5.3: Docker Desktop do Owner tự mở và đo RAM; không tự khởi động cùng ứng dụng; không đổi WSL, không `wsl --shutdown` khi chưa phê duyệt riêng.
- §5.4, §5.5, §5.6: chỉ cho phép Drizzle PoC, Dynamic Skill/Resume PoC (scratch) và Docker sandbox validation; không chạy toàn bộ 27 runbook.
- §8: không Agent thực thi arbitrary code trên host khi chưa có sandbox đạt chuẩn; Phase 2 chỉ build provider adapter ở mock/sandbox mode nếu security gate chưa đạt; không giả lập Agent hoạt động hay workflow thành công khi thiếu execution evidence.

## 10. Quyết định còn Pending

Chưa có D-xxx nào ở trạng thái PENDING_OWNER_APPROVAL sau OWR-2. Các mục còn chờ Owner (PENDING_OWNER_CLARIFICATION) hoặc chờ kiểm thử:

| # | Mục | Loại | Nội dung còn mở |
|---|---|---|---|
| P-1 | D-014 | PENDING_OWNER_CLARIFICATION | Danh sách event "Prompt 1 mục 8.4" chưa có trong repo; Owner xác nhận bản sao lưu trong repo |
| P-2 | Q-07 | PENDING_OWNER_CLARIFICATION | Xác minh ToS, usage limit và điều kiện API cho Claude Team và ChatGPT/Codex; chưa nâng concurrency |
| P-3 | FR-122 | PENDING_OWNER_CLARIFICATION | Artifact nào bắt buộc; có export/download hay chỉ lưu và truy xuất trong workspace |
| P-4 | FR-181 | PENDING_OWNER_CLARIFICATION | Approval chờ lâu là incident hay operational alert; quy tắc grouping incident lặp lại |
| P-5 | FR-182 | PENDING_OWNER_CLARIFICATION | Xử lý task `cancelled`; chấp thuận equal-weight hay không |
| P-6 | FR-183 | PENDING_OWNER_CLARIFICATION | Nhiều metric hay một KPI tổng hợp (nếu KPI: công thức và trọng số cần duyệt) |
| P-7 | D-011 / CF-019 | PENDING_OWNER_CLARIFICATION | Tile size 16 hay 32 chưa được quyết định; benchmark chưa chạy |
| P-8 | D-005 | PENDING_OWNER_CLARIFICATION | Owner xác nhận đã biết thay đổi nhóm chấp nhận MVP B (Codex) |
| P-9 | Q-15 | PENDING_OWNER_CLARIFICATION | MIT cho asset không được OWR-2 nhắc; cần xác nhận có cho phép hay không |
| P-10 | Q-01 | APPROVED_CONDITIONAL | Owner tự ký từng ô checklist rủi ro; điều kiện OWR-1 mục 8.3 còn hiệu lực |
| P-11 | D-003 / Q-20 | APPROVED_CONDITIONAL | Chờ PoC đạt tiêu chí Q-20 (chưa chạy) |
| P-12 | D-021 / Q-18 | APPROVED_CONDITIONAL | Chờ OWN-AC-1..3 (PoC SK-01..SK-05) đạt |
| P-13 | D-011 | APPROVED_CONDITIONAL | Chờ benchmark `OfficeRenderer` |
| P-14 | D-019 | APPROVED_CONDITIONAL | Chờ Security Review (Phase 3) |
| P-15 | D-007 | APPROVED_CONDITIONAL | Điều kiện OWR-1 và OWR-2 §8; SB-00..SB-17 chưa chạy |

## 11. Xung đột với tài liệu gốc

Theo OWR-2 §2: khi tài liệu gốc có giá trị hoặc quy tắc khác với nội dung Owner, ghi nhận xung đột và báo cáo, không âm thầm ghi đè. Mỗi xung đột dưới đây tìm thấy bằng đọc/grep ngày 2026-10-30; chưa tài liệu nào được sửa bởi agent này.

| # | Xung đột | OWR-2 | Tài liệu gốc/hiện hành | Hành động |
|---|---|---|---|---|
| X-1 | Locale mặc định | D-015: Vietnamese mặc định, English thứ hai | ADR-0015 (nội dung gốc), DECISION_LOG D-015 (cũ), DESIGN_TOKENS.md, UI_UX_GUIDELINES.md, FRONTEND_ARCHITECTURE.md, SYSTEM_ARCHITECTURE.md ghi tiếng Anh mặc định | Đồng bộ tài liệu theo OWR-2 (agent khác) |
| X-2 | Codex trong MVP | D-005: hoãn Codex khỏi MVP; Q-10 không cài Codex CLI | FR-077 (Adapter Codex, M, MVP, [Owner]); ACCEPTANCE_CRITERIA nhóm B (AC-022, GP-1 yêu cầu Codex CLI login); PRD.md:88; đặc tả gốc của Owner cho nhóm chấp nhận MVP B yêu cầu kết nối Codex | **Thay đổi yêu cầu gốc của Owner: Owner cần biết (P-8).** Không tự sửa FR/AC |
| X-3 | Enum PermissionLevel có `full` | Q-03: không hỗ trợ `full` | DATABASE_DESIGN.md:170 `['read_only','standard','elevated','full']`; PERMISSION_MODEL.md; USER_ROLES_AND_PERMISSIONS.md L5 | Bỏ `full`, CHECK cấm; đồng bộ (CF-008) |
| X-4 | Event naming | Q-12/Q-19: DB dotted lowercase; API có thể UPPER_SNAKE | EVENT_SYSTEM.md M-E2 và FR-117 dùng UPPER_SNAKE; OWR-1 mục 8.1 hiểu "giữ UPPER_SNAKE" (được Q-19 làm rõ) | Đồng bộ M-E2 (CF-003) |
| X-5 | Chính sách asset | Q-15: CC0 mặc định; font OFL kèm license/NOTICE; CC-BY duyệt từng package; MIT cho asset không được nhắc | D-012/ADR-0012 đề xuất CC0/MIT/OFL; LICENSE_AND_REUSE.md:134 cho phép CC0 hoặc MIT/CC-BY; VIRTUAL_OFFICE.md §21 #1; JIK-A-4 chưa có bằng chứng license (CF-025) | Đồng bộ; P-9 |
| X-6 | Tile size | D-011: không quyết định | DATABASE_DESIGN.md:1025 `tile_size` 32 và SCREEN_SPECIFICATION.md:127 32 px; VIRTUAL_OFFICE.md:60,116,331 và Pixel Agents 16 px | STILL_OPEN (P-7) |
| X-7 | Timeout approval | Q-05: 540 s, WAITING_HUMAN, nhắc 24 h, 15 phút cho hành động quan trọng | DATABASE_DESIGN.md:832 (3600 s), PROVIDER_INTEGRATION.md:411 (10 phút), WORKFLOW_ENGINE.md:201,359 (null), USER_ROLES_AND_PERMISSIONS.md §5.5, research Claude (30 phút) | Đồng bộ (CF-007) |
| X-8 | Retention audit | Q-06: 365 ngày + export 2 năm | SECURITY.md:498 ghi 180 ngày | Đồng bộ (CF-006) |
| X-9 | OpenHands và Tier 2 | D-006: hoãn sau Phase 4, không provider thứ ba trong MVP | PRD.md:88 và SRS.md:28,53 liệt kê OpenHands là adapter; FR-078 (C, P2); ADR-0007 Tier 2 dựa vào OpenHands (ADR-0006). Tier 2 mặc định là đường lùi của D-007 nên phụ thuộc quyết định này | Owner cần biết Tier 2 chưa có runtime trong MVP; đồng bộ |
| X-10 | Phase 2 provider thực | OWR-2 §8: Phase 2 chỉ build adapter ở mock/sandbox mode nếu security gate chưa đạt | Có thể xung đột với IMPLEMENTATION_ROADMAP.md và ADR-0004 (adapter Claude ở Phase 2); chưa đối chiếu chi tiết | Kiểm tra khi đồng bộ roadmap |
| X-11 | Concurrency theo provider | Q-07: mặc định 5 là giới hạn ứng dụng, không phải xác nhận quota | ADR-0010 và RESOURCE_MANAGEMENT.md có cap Claude 5, Codex 3 và bảng ngoại suy 5/10/18; QA CF-044 | Chú thích chưa kiểm chứng; không tuyên bố 18 đã test |
| X-12 | Bốn FR | OWR-2 §4: chỉ là đề xuất AC, chưa APPROVED | FR-122, FR-181, FR-182, FR-183 trong FUNCTIONAL_REQUIREMENTS.md có định nghĩa riêng; ghi nhận OWR-1 mục 8.1 (mã không khớp) | Đối chiếu với FR gốc trước khi cập nhật; không biến PROPOSED AC thành APPROVED AC |

## 12. Nhật ký thay đổi OWR-2

Ngày: 2026-10-30. Tác nhân: Decision Records agent (chỉ tài liệu). Không chạy git add/commit/push. Không đổi bất kỳ file ngoài danh sách dưới đây.

| File | Thay đổi |
|---|---|
| `PHASE_0_OWNER_DECISION_REGISTER.md` | Cập nhật bảng mục 2 và cột Owner response mục 4; thêm D-021; thêm mục 9 đến 12; chú thích lịch sử ở mục 7, 8; giữ nguyên văn OWR-1 |
| `DECISION_LOG.md` | Cập nhật cột Phê duyệt cho D-001..D-021, thêm mục OWR-2, bỏ các nhãn "Chờ duyệt" mâu thuẫn |
| `docs/adr/ADR-0001..ADR-0020` | Cập nhật Metadata (Decision status, Approved by/date) và thêm mục "Cập nhật OWR-2" ở cuối mỗi ADR; Validation evidence không nâng |
| `docs/adr/README.md` | Cập nhật cột Status; mức bằng chứng giữ nguyên Partial/Unverified |
| `docs/adr/CONFLICTS_TO_RESOLVE.md` | Thêm mục "Cập nhật OWR-2" (RESOLVED_BY_OWNER_DECISION, đồng bộ tài liệu chưa xác nhận; hoặc STILL_OPEN) |
