# ADR-0004 — Tích hợp Claude Code bằng @anthropic-ai/claude-agent-sdk, cổng phê duyệt canUseTool + PreToolUse

## 1. Metadata

- **ID:** ADR-0004
- **Ngày:** 2026-10-30
- **Decision ID:** D-004 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** research/CLAUDE_CODE_INTEGRATION.md, research/TECHNOLOGY_EVALUATION.md TE-03, PROVIDER_INTEGRATION.md §17

## 2. Context

- Cần điều khiển Claude Code (spawn, stream, cancel, resume, approval) bằng đăng nhập OAuth subscription sẵn có mà app không đọc credential.
- Research chọn `canUseTool`, SECURITY §5.1 chọn PreToolUse hook: hai tài liệu lệch nhau.

## 3. Problem

- Chọn cách tích hợp và hợp nhất hai lớp approval dùng chung một policy engine.
- Điều khoản dùng OAuth subscription trong sản phẩm đa-agent chưa xác minh (DECISION_LOG mâu thuẫn 2).

## 4. Options considered

### Phương án A — `@anthropic-ai/claude-agent-sdk` `query()` + canUseTool + PreToolUse [ĐỀ XUẤT]

Kế thừa OAuth login, approval in-process, `interrupt()`, `streamInput()`. `RawCliClaudeAdapter` nhỏ chỉ cho `claude auth status` và `--version`.
- **Lý do chọn đề xuất:** Tác giả tài liệu báo cáo smoke test chạy được; cung cấp điểm duyệt đồng bộ cho UI.

### Phương án B — Spawn `claude -p --output-format stream-json` thô [Bị loại]

Zero dependency.
- **Lý do loại:** Phải tự dựng permission prompt tool/HTTP hook, tự parse NDJSON, phụ thuộc phiên bản PATH, hủy chỉ bằng SIGTERM, dễ `ENAMETOOLONG` trên Windows.

### Phương án C — Hook + transcript tail kiểu Pixel Agents [Bị loại]

Quan sát session người dùng tự chạy.
- **Lý do loại:** Không điều khiển được; format JSONL đổi theo version.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Dùng Agent SDK làm đường chính; `canUseTool` là đường in-process, PreToolUse hook là lớp bắt buộc không bypass; cả hai gọi cùng policy engine (PERMISSION_MODEL §14 mục 4).
- Chế độ spawn mặc định `default` + whitelist an toàn (CF-014); ghim version SDK, kiểm contract bằng fixture.
- Cập nhật SECURITY §5.1 theo mô hình hai tầng.

## 6. Rationale

- Tác giả báo cáo: CLI 2.1.312, SDK 0.3.314, 5 process song song cùng login 7 s không lỗi (QA không tái hiện).
- `canUseTool` chặn được `Write` (`permission_denials = 1`) theo báo cáo.
- Hook giữ vai trò phòng thủ chiều sâu khi SDK cấu hình sai.

## 7. Consequences

**Tích cực**
- Approval và interrupt có cấu trúc.
- Không tự dựng parser NDJSON.

**Tiêu cực**
- API lớn, đổi nhanh (0.x); `env` thay toàn bộ env; binary lớn theo platform.
- Approval mức tool giữ process sống nên chiếm slot (RESOURCE §4.1 cần sửa).

**Trung tính**
- Ghim phiên bản và nâng cấp có kiểm contract.
- `PermissionRequest` hook và `canUseTool` có thể trùng quyết định.

## 8. Security implications

- App không đọc `.credentials.json` (ADR-0008); SDK kế thừa login do CLI quản.
- `env` phải whitelist; hook và `canUseTool` fail-closed khi policy engine lỗi.
- Prompt injection qua nội dung repo vẫn là rủi ro; approval là điểm chặn, không phải ranh giới (ADR-0007).

## 9. Cost and operational implications

- Chi phí cố định ~48k token mỗi lần spawn theo báo cáo, tiêu quota subscription.
- Chưa đo task dài hay hành vi khi chạm quota 5 giờ/tuần.
- Nâng cấp SDK 0.3.x thường xuyên tốn công kiểm fixture.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| CLI 2.1.312 + SDK 0.3.314 smoke test, 5 process song song | `docs/research/CLAUDE_CODE_INTEGRATION.md` | Chưa kiểm chứng |
| Package SDK tồn tại trên npm (0.3.314) | `docs/QA_REVIEW_PHASE0.md (bảng npm view)` | Đã xác minh |
| Mâu thuẫn canUseTool vs PreToolUse tồn tại | `docs/PROVIDER_INTEGRATION.md §25 mục 1, 13` | Đã xác minh |
| Hành vi exit 2/`updatedInput` trong `-p` trên 2.1.312 | `docs/security/PERMISSION_MODEL.md §14` | Giả định |
| Điều khoản OAuth subscription cho tự động hóa đa-agent | `DECISION_LOG.md mâu thuẫn 2` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Tác giả báo cáo smoke test thật và QA xác nhận package tồn tại, nhưng QA không tái hiện live test và điều khoản dịch vụ chưa rõ.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận dùng SDK và rủi ro điều khoản OAuth/quota chưa xác minh.
- Chấp nhận mô hình approval hai tầng và chế độ spawn mặc định.
- **Khả năng đảo ngược:** Trung bình: adapter nằm sau interface `AgentAdapter`, đổi sang raw CLI tốn vài tuần.
- **Bị chặn trong lúc chờ duyệt:** Viết `ClaudeAdapter`, policy engine hooks và test approval.

## 12. Reversal conditions

- Điều khoản dịch vụ cấm OAuth subscription cho tự động hóa: chuyển API key (cần quyết định chi phí).
- SDK phá vỡ API liên tục: quay về raw CLI + MCP permission tool.
- Hai tầng approval gây quyết định trùng không giải được.
