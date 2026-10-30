# ADR-0008 — Credential do CLI tự quản, secret khác lưu OS keychain

## 1. Metadata

- **ID:** ADR-0008
- **Ngày:** 2026-10-30
- **Decision ID:** D-008 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** SECURITY.md §7, security/CREDENTIAL_MANAGEMENT.md

## 2. Context

- Credential OAuth Claude/ChatGPT nằm trong `.credentials.json`/`auth.json` do CLI quản.
- Một đăng nhập claude.ai (`authMethod: claude.ai`, `subscriptionType: team`) phục vụ mọi agent (AGENT_ORGANIZATION §8).

## 3. Problem

- App phải dùng đăng nhập sẵn có mà không trở thành nơi lưu token.
- Cần quy tắc nơi lưu secret khác (LLM key, token hook) và tránh lộ vào DB, log, frontend.

## 4. Options considered

### Phương án A — CLI tự quản credential; app không đọc file credential; secret khác trong OS keychain [ĐỀ XUẤT]

`credential_ref` chỉ là khóa keychain; redaction đa tầng; lint/CI chặn đọc đường dẫn creds.
- **Lý do chọn đề xuất:** Giảm bề mặt: app không bao giờ cầm token OAuth.

### Phương án B — Lưu token trong database [Bị loại]

Tiện cho multi-user.
- **Lý do loại:** Biến DB thành mục tiêu và lộ qua backup/log.

### Phương án C — App đọc file credential của CLI rồi truyền lại [Bị loại]

Có thể kiểm soát luồng.
- **Lý do loại:** Vi phạm nguyên tắc không chạm credential; chính là cơ chế OpenHands ACP (ADR-0006).

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- App không đọc `.credentials.json`/`auth.json`; mọi secret khác vào OS keychain (`@napi-rs/keyring` hoặc `keytar`, chưa chốt). **[OWR-3 C (FR-088, FR-098): ứng dụng không lưu hoặc tự đọc credential provider (D-008); secret khác phải làm rõ ở Q-21]**
- Không secret nào vào Postgres plaintext hay frontend; env whitelist cho process agent.
- Log/WS/DB qua redaction (corpus ≥ 200 mẫu × 6 encoding, 0 leak) và `safe-regex2` chống ReDoS.
- Chưa chốt: bảng `secrets` hay chỉ `credential_ref` (CREDENTIAL_MANAGEMENT §14).

## 6. Rationale

- App không bao giờ cầm token OAuth.
- Audit ghi bộ ba (userId, agentId, providerCredentialId) mà không lưu secret.
- Khớp thực tế một đăng nhập team phục vụ mọi agent.

## 7. Consequences

**Tích cực**
- Bề mặt lộ secret nhỏ.
- Kiểm tra tự động bằng grep CI.

**Tiêu cực**
- Không kiểm soát quyền file 644 của `.credentials.json` (R2).
- Không hỗ trợ ACP sub-mode OpenHands ở MVP.
- Env whitelist phải cân bằng: SECURITY không có `APPDATA/LOCALAPPDATA` trong khi research Claude yêu cầu giữ.

**Trung tính**
- Chuyển server mode nhiều người dùng sẽ cần Vault/SOPS/age (DEVSECOPS §14.2).
- Redaction không bắt được dạng secret mới.

## 8. Security implications

- Credential Manager không bảo vệ khỏi process cùng user.
- Rò rỉ qua env: `docker inspect` có thể lộ env nếu không lọc (CREDENTIAL_MANAGEMENT §15).
- Token hook/agent (ADR-0019) cũng là secret: hết hạn theo task, không log.

## 9. Cost and operational implications

- Thư viện keychain native thêm rủi ro đóng gói trên Node 24 + Tauri sidecar (chưa thử).
- Duy trì corpus redaction và lint rule.
- Không phát sinh chi phí lưu trữ secret ngoài.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| `claude auth status` trả `authMethod: claude.ai`, `subscriptionType: team` | `docs/architecture/AGENT_ORGANIZATION.md §8` | Chưa kiểm chứng |
| Bảng rủi ro và vấn đề chưa giải quyết tồn tại | `docs/security/CREDENTIAL_MANAGEMENT.md §14` | Đã xác minh |
| Thư viện keychain chạy trên Node 24/Tauri sidecar | `docs/security/CREDENTIAL_MANAGEMENT.md §14 mục 2` | Giả định |
| Corpus redaction 0 leak | `docs/security/CREDENTIAL_MANAGEMENT.md` | Chưa kiểm chứng |
| Mâu thuẫn ACP `CODEX_AUTH_JSON` vs D-008 | `docs/PROVIDER_INTEGRATION.md §9.5` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. Thiết kế nhất quán và có một quan sát thực (auth status do tác giả báo cáo, QA không tái hiện); keychain và redaction chưa thử.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận app không bao giờ đọc file credential của CLI.
- Chọn thư viện keychain và bảng `secrets` vs `credential_ref`.
- **Khả năng đảo ngược:** Trung bình: đổi sang vault tự quản cần thiết kế lại luồng; có thể làm tăng dần.
- **Bị chặn trong lúc chờ duyệt:** Mọi mã đụng credential, MCP/LLM key, token hook.

## 12. Reversal conditions

- Điều khoản/quota buộc chuyển API key theo agent: thiết kế lại vault và thêm bảng `secrets`.
- CLI cung cấp API token ngắn hạn an toàn hơn: đánh giá lại.
- Chuyển server mode nhiều người dùng: cần Vault/SOPS/age.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-008, §8 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- App không tự ý đọc `.credentials.json` hoặc `auth.json`; không lưu plaintext token trong database hoặc frontend; secret khác dùng OS keychain hoặc cơ chế phù hợp.
- Bổ sung: **không mount thư mục credential của Owner vào Agent sandbox** (khớp security gate OWR-2 §8). Không dùng credential thật trong PoC (§9).
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
