# WORK_PLAN.md — AI WORKSPACE

Cập nhật: 2026-10-30 (Phase 0 MỞ, chờ Owner; nguồn trạng thái chính: TASK_BACKLOG.md và PROJECT_STATUS.md)
Orchestrator: Chief Orchestrator / AI Technical Director (Claude Code session chính)
Giới hạn đồng thời: tối đa 10 agent (Claude Code subagents) — thực thi song song thật qua Agent tool.

## 0. Bối cảnh môi trường (đã kiểm tra thực tế 2026-10-30)

| Thành phần | Trạng thái |
|---|---|
| Claude Code CLI | 2.1.312, `C:\Users\TUYEN\.local\bin\claude` — đã cài |
| Codex CLI | CHƯA cài (`codex` not found) — cần user cài `npm i -g @openai/codex` + `codex login` |
| Node.js | v24.19.0 |
| Docker | 29.8.1 (Docker Desktop) |
| Git | có (Git Bash / ucrt64) |
| Python | có (WindowsApps shim) |
| Repo tham khảo | `../ChatDev` (Apache-2.0), `../pixel-agents` (MIT, v1.5.0), `../my-virtual-office` (AGPL-3.0), `../OpenHands` (MIT) |
| Thư mục sản phẩm | `AI-Workspace-App/` (thư mục này) |

## 1. WBS — Phase 0: Research & Architecture (ĐANG CHẠY SONG SONG)

| ID | Task | Agent | Output (file ownership — chỉ agent đó ghi) | Trạng thái |
|---|---|---|---|---|
| P0-01 | Phân tích ChatDev 1.0/2.0 | A01 ChatDev Analyst | docs/research/CHATDEV_ANALYSIS.md | DONE |
| P0-02 | Phân tích Pixel Agents | A02 Pixel Agents Analyst | docs/research/PIXEL_AGENTS_ANALYSIS.md | DONE |
| P0-03 | Phân tích My Virtual Office (AGPL) | A03 MVO Analyst | docs/research/MY_VIRTUAL_OFFICE_ANALYSIS.md | DONE |
| P0-04 | Phân tích OpenHands | A04 OpenHands Analyst | docs/research/OPENHANDS_ANALYSIS.md | DONE |
| P0-05 | Tích hợp Claude Code (CLI/SDK, smoke test thật) | A05 Claude Integration | docs/research/CLAUDE_CODE_INTEGRATION.md | DONE |
| P0-06 | Tích hợp Codex (App Server, exec --json) | A06 Codex Integration | docs/research/CODEX_INTEGRATION.md | DONE |
| P0-07 | Orchestration + Workflow Engine | A07 Orchestration Architect | docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md | DONE |
| P0-08 | Database design | A08 DB Engineer | docs/DATABASE_DESIGN.md | DONE |
| P0-09 | Security & sandbox | A09 Security Engineer | docs/SECURITY.md | DONE |
| P0-10 | License & frontend stack | A10 License/Stack Analyst | docs/research/LICENSE_AND_REUSE.md | DONE |
| P0-11 | UI/UX Design Research + Design System spec (Prompt 3) | A11 UI/UX Designer | docs/design/DESIGN_PRINCIPLES.md, docs/design/DESIGN_SYSTEM.md | DONE |
| P0-12 | Tổng hợp: ARCHITECTURE_COMPARISON, REUSE_DECISIONS, SYSTEM_ARCHITECTURE, ADR, roadmap | Orchestrator | docs/research/ARCHITECTURE_COMPARISON.md, docs/research/REUSE_DECISIONS.md, docs/SYSTEM_ARCHITECTURE.md, docs/ADR/* | DONE (xem TASK_BACKLOG.md) |

Dependency: P0-12 ← tất cả. P0-01..P0-11 độc lập.

## 2. Phase 1 — Foundation (dự kiến, sẽ chốt sau Phase 0)

Nhóm song song (mỗi nhóm một worktree/branch, file ownership riêng):
- F1 Backend foundation (NestJS skeleton, config, logging, event bus, WS gateway)
- F2 Database foundation (Drizzle schema theo D-002, migrations, seed)
- F3 Frontend foundation (Vite+React+TS+Tailwind+shadcn, app shell, design tokens)
- F4 Provider adapters (ClaudeCodeAdapter thật; CodexAdapter sau khi user cài CLI)
- F5 Agent/Org/Department modules (CRUD + API contract)
- F6 Testing infrastructure (Vitest, Playwright, CI)
Shared contracts (OpenAPI + event schema) do Architect chốt TRƯỚC khi F3/F5 chạy.

## 3. Quy tắc phối hợp
- Mỗi task khai báo files được ghi / chỉ đọc.
- Không hai agent ghi cùng file.
- Orchestrator kiểm diff + chạy test trước khi tích hợp.
- Không đánh dấu COMPLETED khi chưa có bằng chứng kiểm thử.
