# ADR-0005 — Tích hợp Codex qua codex app-server (JSON-RPC stdio), dự phòng codex exec --json

## 1. Metadata

- **ID:** ADR-0005
- **Ngày:** 2026-10-30
- **Decision ID:** D-005 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** research/CODEX_INTEGRATION.md, research/TECHNOLOGY_EVALUATION.md TE-04

## 2. Context

- Cần tích hợp Codex CLI với đăng nhập ChatGPT, approval tương tác, nhiều thread song song và đọc quota.
- Codex CLI chưa cài trên máy; chỉ có binary bundled của extension VS Code (0.164.0-alpha.11.1) dùng để đọc help và sinh binding.

## 3. Problem

- Chọn đường tích hợp khi không có bằng chứng chạy thật.
- SECURITY §5.1 dùng tên event v1 legacy `exec_approval_request`, trong khi v2 là `item/commandExecution/requestApproval`.

## 4. Options considered

### Phương án A — `codex app-server` (JSON-RPC stdio), fallback `codex exec --json` [ĐỀ XUẤT]

Nhiều thread trên một process; approval tương tác; delta token; interrupt/steer; đọc rate limit/account/model. Đánh dấu experimental.
- **Lý do chọn đề xuất:** Chỉ app-server cung cấp approval gating tương đương Claude để dùng chung policy engine.

### Phương án B — `@openai/codex-sdk` làm đường chính [Bị loại]

Typed, có `runStreamed`.
- **Lý do loại:** Chỉ bọc `codex exec`: không approval callback, mỗi `run()` một process; chỉ dùng cho batch.

### Phương án C — Chỉ `codex exec --json` [Bị loại]

API ổn định, đơn giản.
- **Lý do loại:** 1 process = 1 turn, không approval tương tác, không delta, không interrupt; giữ làm dự phòng.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- app-server là đường chính, `exec --json` dự phòng, SDK chỉ cho batch.
- Concurrency 2–3 turn/account, throttle khi `usedPercent > 80`.
- Cấu hình do app sinh (per-thread `config`, `sandbox: workspace-write`), không kế thừa `~/.codex/config.toml`; approval policy xem CF-015.
- Codex có thể để ngoài MVP nếu chủ sở hữu không muốn cài hoặc tốn quota cho live test.

## 6. Rationale

- Binding sinh bằng `generate-ts/generate-json-schema` cho phép kiểm contract bằng fixture.
- `exec --json` đủ cho chạy không giám sát khi app-server lỗi.
- Tách adapter sau `AgentAdapter` giữ đường lui.

## 7. Consequences

**Tích cực**
- Approval và quota đọc được qua một giao thức.
- Có fallback.

**Tiêu cực**
- Phụ thuộc API experimental: ghim phiên bản, sinh lại binding mỗi lần nâng.
- Adapter phức tạp hơn exec.
- Phải sửa SECURITY §5.1 theo tên event v2.

**Trung tính**
- Mọi hành vi Codex là giả định cho tới khi có live test (tốn quota).
- `mxc` sandbox trên Windows build 26300 chưa kiểm (SECURITY R10).

## 8. Security implications

- Approval policy mặc định phải chặt (CF-015); không kế thừa config người dùng để tránh `danger-full-access`.
- Credential `auth.json` do Codex quản, app không đọc (ADR-0008).
- App-server experimental có thể đổi hành vi approval: cần fixture contract và fail-closed.

## 9. Cost and operational implications

- Cài `npm i -g @openai/codex` và live test tiêu quota ChatGPT (cửa sổ 5 giờ + tuần).
- ToS fair-use cho nhiều phiên song song chưa xác minh.
- Ghim phiên bản và sinh lại binding mỗi lần nâng là chi phí lặp lại.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| `codex` không có trên PATH; login ChatGPT từ binary extension; `generate-ts` chạy được | `docs/research/CODEX_INTEGRATION.md §0` | Đã xác minh |
| Không có session Codex thật nào được tạo; độ tin cậy Thấp | `docs/research/TECHNOLOGY_EVALUATION.md TE-04` | Giả định |
| App-server ổn định và có approval như binding mô tả | `docs/research/CODEX_INTEGRATION.md §11.4` | Chưa kiểm chứng |
| Lệch tên event approval v1/v2 | `docs/PROVIDER_INTEGRATION.md §25 mục 7` | Đã xác minh |
| ToS cho nhiều phiên song song | `DECISION_LOG.md mâu thuẫn 2, 4` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Codex CLI chưa cài; chưa có live test nào, mọi hành vi tích hợp chỉ suy ra từ help và binding.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận phụ thuộc API experimental hay để Codex ngoài MVP.
- Cho phép cài Codex và chi quota cho live test (phải trả lời trước Phase 2).
- **Khả năng đảo ngược:** Dễ: adapter độc lập; đổi sang exec/SDK chỉ đổi adapter.
- **Bị chặn trong lúc chờ duyệt:** Viết `CodexAdapter`, test approval Codex; Codex không được coi là provider hỗ trợ.

## 12. Reversal conditions

- Live test cho thấy app-server không ổn định hoặc thiếu approval: chuyển `exec --json` + Tier 2 sandbox.
- Chủ sở hữu không muốn cài Codex hoặc tốn quota: để Codex ngoài MVP.
- OpenAI công bố SDK có approval callback ổn định.
