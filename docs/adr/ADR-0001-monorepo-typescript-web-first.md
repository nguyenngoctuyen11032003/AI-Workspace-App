# ADR-0001 — Monorepo TypeScript, web app + backend Node cục bộ, Tauri 2 ở phase sau

## 1. Metadata

- **ID:** ADR-0001
- **Ngày:** 2026-10-30
- **Decision ID:** D-001 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-1; hoàn tất khung frontend theo OWR-2 Q-16)
- **Nguồn:** research/LICENSE_AND_REUSE.md §5, development/DEPLOYMENT.md, research/TECHNOLOGY_EVALUATION.md TE-06/TE-09/TE-10

## 2. Context

- Backend phải spawn CLI agent (Claude Code, Codex) và quản lý process nền trên Windows; frontend cần React 19, Phaser, Monaco, React Flow.
- Máy dev đã có pnpm 10 và Node 24 (TE-10). Chưa có quyết định về shell desktop.
- CI_CD.md thêm Turborepo trong khi D-001 không nhắc (CF-027).

## 3. Problem

- Chọn nền tảng ứng dụng và cấu trúc repo sao cho spawn process CLI dài hạn ổn định, giảm rủi ro đóng gói ở MVP và vẫn có đường lên desktop.
- Phiên bản stack (React 19.2, Vite 7, Tailwind v4, TS 5.9) lấy từ web search của agent khác, chưa qua `npm view`.

## 4. Options considered

### Phương án A — Monorepo pnpm, NestJS + React 19/Vite/Tailwind v4/shadcn, web trước, Tauri 2 sau [ĐỀ XUẤT]

MVP chạy web app + backend Node cục bộ; Tauri 2 bọc ở phase sau, backend là sidecar. Cấu trúc `apps/api`, `apps/web`, `apps/desktop` (sau), `packages/*`.
- **Lý do chọn đề xuất:** CLI được spawn từ NestJS (Node) nên shell không cần quyền spawn; tránh bị chặn bởi sidecar, code-signing và ABI native module ở Phase 1.

### Phương án B — Electron [Bị loại]

Node sẵn có (`fork()` NestJS), Chromium đồng nhất, OpenHands có mẫu `electron/main.mjs`.
- **Lý do loại:** Installer 80–240 MB, RAM ~200–300 MB, cấu hình bảo mật thủ công; chưa cần desktop ở MVP.

### Phương án C — Next.js full-stack [Bị loại]

Một framework cho UI và API.
- **Lý do loại:** Không hợp backend dài hạn (BullMQ worker, process manager, WebSocket gateway), phải tắt SSR cho Phaser và không giải bài toán desktop.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Monorepo pnpm workspaces; backend NestJS; frontend React 19 + Vite + Tailwind v4 + shadcn/ui; MVP là web app + backend Node cục bộ.
- Tauri 2 chỉ ở phase sau, backend làm sidecar.
- Bắt đầu bằng pnpm thuần; việc thêm Turborepo chưa chốt (CF-027).

## 6. Rationale

- Backend là Node nên spawn CLI, `taskkill /T /F`, BullMQ worker chạy trực tiếp, không cần cầu nối IPC.
- Web-first tránh phụ thuộc sidecar, code-signing và `node-pty` ABI trong Phase 1.
- pnpm hỗ trợ `pnpm licenses list --json` phục vụ THIRD_PARTY_NOTICES (ADR-0012).

## 7. Consequences

**Tích cực**
- Vòng lặp phát triển nhanh, ít rủi ro đóng gói.
- Một ngôn ngữ (TypeScript) cho cả hai phía.

**Tiêu cực**
- Phase 1 chưa có tray, auto-start, quản lý vòng đời backend bằng shell.
- Cần spike đóng gói NestJS sidecar (Node SEA, `@yao-pkg/pkg` hoặc ship Node runtime) trước khi lên Tauri; cần người biết Rust.

**Trung tính**
- Quyết định Turborepo để mở.
- Số phiên bản phải được `npm view` lại trước khi tạo `package.json`.

## 8. Security implications

- Backend lắng nghe localhost phải bind 127.0.0.1 và có token phiên; không để lộ API ra mạng LAN mặc định.
- Dependency npm lớn: cần lockfile, `pnpm audit` và kiểm license trong CI.
- Tauri capability model (phase sau) giảm bề mặt so với Electron nhưng chưa được đánh giá.

## 9. Cost and operational implications

- Chi phí hạ tầng gần 0 ở MVP (chạy cục bộ).
- Chi phí vận hành: duy trì pnpm workspace, CI cache; spike sidecar ước tính vài ngày.
- Tauri thêm chi phí code-signing và kênh cập nhật ở phase sau.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| pnpm 10.34.6 và Node v24.19.0 đã cài trên máy dev | `docs/research/TECHNOLOGY_EVALUATION.md TE-10` | Đã xác minh |
| Electron của OpenHands đã đọc mã (`electron/main.mjs`) | `docs/research/TECHNOLOGY_EVALUATION.md TE-06` | Đã xác minh |
| Số liệu Tauri (installer 8–15 MB, RAM 30–50 MB, sidecar Node) | `docs/research/TECHNOLOGY_EVALUATION.md TE-06` | Giả định |
| Phiên bản React 19.2, Vite 7, Tailwind v4, TS 5.9 | `docs/research/LICENSE_AND_REUSE.md §5` | Chưa kiểm chứng |
| Native module (`node-pty`, `better-sqlite3`) chạy trên Windows với đường dẫn có khoảng trắng | `docs/research/TECHNOLOGY_EVALUATION.md TE-09` | Chưa kiểm chứng |
| Turborepo cần thiết | `docs/development/CI_CD.md` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Chỉ môi trường dev (pnpm/Node) và mã Electron tham chiếu được xác minh; stack frontend, Tauri và native module là giả định.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận web-first (không desktop ở Phase 1) và Tauri 2 là hướng desktop duy nhất.
- Chọn pnpm thuần hay thêm Turborepo (CF-027).
- **Khả năng đảo ngược:** Dễ ở đầu dự án (chỉ cấu trúc repo); đắt dần khi có nhiều package và CI phụ thuộc.
- **Bị chặn trong lúc chờ duyệt:** Khởi tạo repo, `package.json`, CI và mọi mã Phase 1 (cấm bắt đầu Phase 1 trước khi duyệt).

## 12. Reversal conditions

- Spike sidecar (NestJS + native module) trên Windows thất bại hoặc vượt ngân sách thời gian: chuyển Electron.
- Chủ sở hữu chỉ cần web/server, không desktop: bỏ Tauri khỏi lộ trình.
- Đội không có năng lực Rust sau Phase 2: đánh giá lại Electron.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §1 Q-16 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Q-16 đã được trả lời: React 19 + Vite, NestJS backend, monorepo TypeScript, pnpm workspace. **Không chuyển sang Next.js trong MVP.** Bảo lưu "trừ khung frontend" trước đây không còn hiệu lực; D-001 là APPROVED đầy đủ.
- Tauri 2 ở phase sau, pnpm thuần không Turborepo: không đổi so với nội dung ADR.
- Tài liệu liên quan cần đồng bộ (do agent khác thực hiện, chưa xác nhận): `FRONTEND_ARCHITECTURE.md`, `development/DEPLOYMENT.md`.
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
