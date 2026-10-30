# PROJECT_STATUS.md — AI WORKSPACE

Cập nhật: 2026-10-30 (sau OWR-3)

## Phase hiện tại
**Phase 0 vẫn MỞ.** Phase 1 **chưa được Owner cho phép**. Không có mục nào Completed hay VERIFIED. Điều kiện đóng: `PHASE_0_FINAL_REPORT.md` mục G.

## Đã làm
- OWR-1, OWR-2, OWR-3 đã ghi nhận trong `PHASE_0_OWNER_DECISION_REGISTER.md` (mục 8, 9 đến 15), `DECISION_LOG.md`, ADR-0001..0021. 16 APPROVED, 5 APPROVED_CONDITIONAL, 0 VERIFIED.
- Đồng bộ tài liệu theo OWR-2 (74 thay đổi) và OWR-3 (nhóm A 46 mục, nhóm B có log). 16 event gốc chép nguyên văn vào `docs/EVENT_SYSTEM.md`.
- Truy vết 199 FR: 109 AC_DEFINED, 74 AC_ADDED_PHASE0, 15 AC_PROPOSED_OWNER_DECIDED, 1 WONT_MVP (FR-078), 0 NEEDS_OWNER_CLARIFICATION. AC giữ PROPOSED.
- QA độc lập lần ba: PASS WITH FIXES (0 P0, 1 P1, 9 P2, 8 P3). Sửa sau đó chưa được QA kiểm lại.
- PoC trước đó: Drizzle (Partial, PGlite), Skill/Resume (Partial, SK-04 SIMULATED). Docker BLOCKED / NOT AUTHORIZED.

## Đang bị block
| Việc | Chờ gì |
|---|---|
| Đồng bộ thân SECURITY mục 2 đến 5 và SANDBOX_DESIGN mục 4 | Owner trả lời X-13 (ba câu) và Q-26 |
| Bảng ánh xạ event dotted | Owner cung cấp (P-16, X-14) |
| Docker sandbox validation | Owner cho phép mở Docker (Q-23) |
| Conductor PoC, test agent thật qua SDK, Codex, 27 runbook | Không được phép |
| P1 của QA ba (keychain, API key) | Owner trả lời Q-21 |
| Phase 1 | Xác nhận rõ ràng của Owner |

## Cần Owner xác nhận
`PHASE_0_FINAL_REPORT.md` mục A và G; sổ quyết định mục 13 đến 15 (Q-21..Q-26).

## Git
Branch `docs/phase0-owner-decisions`, 2 commit (`f08240c`, `ffa065e`), đã có trên origin (do Owner push; agent không push). Thay đổi OWR-3 chưa commit. Mô tả PR: `PHASE_0_PR_DESCRIPTION.md`.
