# TASK_BACKLOG.md

Cập nhật: 2026-10-30. Trạng thái: DONE (có sản phẩm tài liệu và đã đối chiếu), REVIEW (đang review), TODO, BLOCKED. Chi tiết Phase 1 trở đi nằm ở `docs/IMPLEMENTATION_ROADMAP.md`.

## Phase 0 — Research, Architecture, Documentation
| ID | Task | Trạng thái | Người làm | Kết quả |
|---|---|---|---|---|
| RES-001 | Phân tích ChatDev 1.0 và 2.0 | DONE | Agent nghiên cứu | docs/research/CHATDEV_ANALYSIS.md |
| RES-002 | Phân tích Pixel Agents | DONE | Agent nghiên cứu | PIXEL_AGENTS_ANALYSIS.md |
| RES-003 | Phân tích My Virtual Office (AGPL) | DONE | Agent nghiên cứu | MY_VIRTUAL_OFFICE_ANALYSIS.md |
| RES-004 | Phân tích OpenHands | DONE | Agent nghiên cứu | OPENHANDS_ANALYSIS.md |
| RES-005 | Phân tích herdr (yêu cầu bổ sung) | DONE | Orchestrator | HERDR_ANALYSIS.md, design/HERDR_DESIGN_INTEGRATION.md |
| RES-006 | License và đánh giá stack | DONE | Agent nghiên cứu | LICENSE_AND_REUSE.md |
| AI-001 | Tích hợp Claude Code (có smoke test thật) | DONE | Agent nghiên cứu | CLAUDE_CODE_INTEGRATION.md |
| AI-002 | Tích hợp Codex (chưa test live) | DONE | Agent nghiên cứu | CODEX_INTEGRATION.md |
| ARCH-001 | Orchestration và workflow engine | DONE | Agent nghiên cứu | ORCHESTRATION_WORKFLOW_RESEARCH.md |
| ARCH-002 | Kiến trúc 18 agent, scheduler, tài nguyên, collaboration | DONE | Agent kiến trúc | docs/architecture/ |
| ARCH-003 | System, backend, frontend, API | DONE | Agent kiến trúc | docs/*.md |
| ARCH-004 | Agent, orchestration, workflow, event | DONE | Agent kiến trúc | docs/*.md |
| ARCH-005 | Provider integration, Virtual Office | DONE | Agent kiến trúc | docs/*.md |
| DB-001 | Database design (Drizzle, 44 bảng) | DONE | Agent database | DATABASE_DESIGN.md |
| SEC-001 | Security và sandbox | DONE | Agent bảo mật | SECURITY.md, docs/security/ |
| UI-001 | Design system, tokens, interaction, responsive | DONE | Agent UI/UX | docs/design/ |
| UX-001 | UX research, navigation, screens, journeys, checklist | DONE | Agent UX | docs/design/ |
| BA-001 | PRD, SRS, FR, NFR, roles, flows, AC | DONE | Agent BA | docs/product/ |
| SK-001 | Skill system | DONE | Agent AI | architecture/SKILL_SYSTEM.md |
| OPS-001 | Infra, git isolation, CI/CD, deployment, env | DONE | Agent DevSecOps | docs/architecture, development, operations |
| DEV-001 | Dev guide, testing, coding standards, components, roadmap, risks | DONE | Agent delivery | docs/development, IMPLEMENTATION_ROADMAP.md |
| RES-007 | So sánh kiến trúc, reuse, technology evaluation | DONE | Agent nghiên cứu | docs/research/ |
| ADR-001 | ADR cho D-001..D-020 và danh sách mâu thuẫn | REVIEW | Agent kiến trúc | docs/adr/ |
| QA-001 | Review kỹ thuật độc lập toàn bộ tài liệu | REVIEW | Agent QA | docs/QA_REVIEW_PHASE0.md |
| DOC-001 | Hợp nhất mâu thuẫn trong tài liệu theo quyết định đã duyệt | BLOCKED | Orchestrator | Chờ chủ sở hữu duyệt CF-xxx |
| DOC-002 | PHASE_0_FINAL_REPORT.md | TODO | Orchestrator | Sau QA-001 |
| OWN-001 | Chủ sở hữu duyệt quyết định D-001..D-020 | BLOCKED | Chủ sở hữu | Điều kiện bắt đầu Phase 1 |

## Hạng mục việc môi trường (cần chủ sở hữu hoặc máy dev)
| ID | Task | Trạng thái |
|---|---|---|
| ENV-001 | Cài Codex CLI (`npm i -g @openai/codex`) để test live | TODO, cần chủ sở hữu |
| ENV-002 | Bật Docker Desktop và xác minh cgroup, ECI, flags sandbox | TODO |
| ENV-003 | Khởi tạo git repo cho AI-Workspace-App | TODO, chờ chốt chính sách commit |
| ENV-004 | Xác minh điều khoản subscription cho nhiều phiên song song | TODO, cần chủ sở hữu |

## Phase 1 (chưa bắt đầu, chờ OWN-001)
Xem `docs/IMPLEMENTATION_ROADMAP.md`, mục Phase 1 breakdown. Các task F1..F6 chạy song song sau khi API contract và event schema được đóng băng.
