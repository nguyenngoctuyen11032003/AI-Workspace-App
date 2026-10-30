# ADR-0012 — Chính sách giấy phép và tái sử dụng (Apache-2.0/MIT có attribution, AGPL chỉ tham khảo)

## 1. Metadata

- **ID:** ADR-0012
- **Ngày:** 2026-10-30
- **Decision ID:** D-012 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** research/LICENSE_AND_REUSE.md, research/REUSE_DECISIONS.md §4.4, QA_REVIEW_PHASE0.md (P1-10)

## 2. Context

- Bốn repo tham chiếu: ChatDev (Apache-2.0), Pixel Agents và OpenHands (MIT), My Virtual Office, viết tắt MVO (AGPL-3.0-or-later, kèm license thương mại).
- Giả định sản phẩm là proprietary; copy mã AGPL buộc công bố mã nguồn.
- LICENSE_AND_REUSE.md:385 và REUSE_DECISIONS.md:265 đều yêu cầu một ADR cho MVO = REF-ONLY clean-room; QA (P1-10) ghi `docs/adr/` từng rỗng.

## 3. Problem

- Cần quy tắc rõ cho mã, prompt, layout, asset, và cơ chế thực thi để clean-room không chỉ nằm trên giấy (rủi ro quy trình RR-1: dev đọc mã MVO rồi viết lại).
- Yêu cầu asset «chỉ CC0» vs cho phép MIT/OFL/CC-BY còn lệch (CF-026); sprite JIK-A-4 thiếu bằng chứng license (CF-025).

## 4. Options considered

### Phương án A — Tái sử dụng có attribution cho Apache/MIT; MVO chỉ tham khảo ý tưởng (clean-room); asset đóng gói chỉ CC0/MIT/OFL [ĐỀ XUẤT]

THIRD_PARTY_NOTICES, header attribution trên file port, cấm sao chép mã/prompt/layout/asset MVO.
- **Lý do chọn đề xuất:** Giữ giá trị kỹ thuật của ChatDev/Pixel Agents/OpenHands với nghĩa vụ rõ; tránh nhiễm AGPL.

### Phương án B — Copy mã My Virtual Office [Bị loại]

Nhanh nhất.
- **Lý do loại:** AGPL-3.0 lây sang sản phẩm khi phân phối/chạy dịch vụ mạng; cần license thương mại riêng.

### Phương án C — Viết mới hoàn toàn, không dùng repo tham chiếu [Bị loại]

An toàn pháp lý tối đa.
- **Lý do loại:** Bỏ phí logic có giá trị của Pixel Agents/ChatDev; chậm.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- **MVO là REF-ONLY, clean-room:** chỉ được ghi lại ý tưởng ở mức khái niệm; cấm sao chép mã, prompt, layout, asset, tên/logo, cấu trúc layout.
- Asset bundle chỉ CC0/MIT/OFL; không bundle PNG trước commit `add749a` của Pixel Agents, font `FSPixelSansUnicode`, tên/logo Pixel Agents; JIK-A-4 chỉ bundle sau khi chụp trang license làm bằng chứng.
- **Reviewer checklist (mọi PR liên quan Office/layout/prompt):** (1) người viết có đọc mã MVO không, nếu có thì không được làm phần này; (2) mô tả nguồn ý tưởng chỉ từ tài liệu ý tưởng đã duyệt; (3) không file nào chứa chuỗi/đường dẫn `my-virtual-office` hay `.mvo-layout.json`; (4) layout, prompt, asset mới có provenance ghi rõ; (5) mỗi dependency/asset mới có license trong allowlist và dòng THIRD_PARTY_NOTICES; (6) file port từ Apache/MIT có header attribution và ghi thay đổi; (7) diff không giống mã MVO (so sánh ngẫu nhiên khi nghi ngờ).
- **Lint/CI rule (đề xuất):** quy tắc chặn import hoặc đường dẫn chứa `my-virtual-office` (và `.mvo-layout.json`) trong mã nguồn, cấu hình, asset manifest; fail CI khi gặp. Bổ sung kiểm `pnpm licenses list` theo allowlist.
- Giữ OpenHands UI là reference-only dù LICENSE_AND_REUSE §3.1 ghi ALLOW (C-6).

## 6. Rationale

- Giữ giá trị kỹ thuật với nghĩa vụ rõ.
- Clean-room với MVO tránh R-AGPL-CONTAM.
- Thực thi bằng checklist + lint + CI thay vì chỉ quy định.

## 7. Consequences

**Tích cực**
- Rủi ro pháp lý thấp nếu tuân thủ.
- Quy trình có thể kiểm tra tự động.

**Tiêu cực**
- Duy trì sổ attribution và kiểm asset từng file.
- Lint chỉ bắt đường dẫn/chuỗi, không bắt được sao chép nội dung đã đổi tên.

**Trung tính**
- Tài liệu là phân tích kỹ thuật, không phải tư vấn pháp lý.
- Nếu đổi license dự án, verdict thay đổi.

## 8. Security implications

- Nhiễm AGPL là rủi ro pháp lý-bảo mật chuỗi cung ứng: cần SBOM/license scan trong CI.
- Asset/phông không rõ nguồn là rủi ro; bằng chứng license phải lưu.
- Prompt và layout MVO không được đưa vào repo vì có thể buộc công bố mã.

## 9. Cost and operational implications

- Công sức review thủ công mỗi PR liên quan.
- Chi phí tư vấn pháp lý nếu chủ sở hữu muốn xác nhận AGPL.
- CI license scan chi phí thấp.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Bảng license đọc từ file LICENSE của từng repo | `docs/research/LICENSE_AND_REUSE.md §1.1` | Đã xác minh |
| MVO = REF-ONLY/DENY RD-060, clean-room, cấm đường dẫn `my-virtual-office` | `docs/research/REUSE_DECISIONS.md §4.4` | Đã xác minh |
| Hai mâu thuẫn license C-2, C-6 | `docs/research/ARCHITECTURE_COMPARISON.md §6.3` | Đã xác minh |
| JIK-A-4 là CC0 | `docs/research/LICENSE_AND_REUSE.md §6.1` | Giả định |
| Cách diễn giải AGPL đúng về mặt pháp lý | `docs/research/LICENSE_AND_REUSE.md` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. License file và mâu thuẫn đã đọc trực tiếp; JIK-A-4 và diễn giải pháp lý chưa kiểm chứng.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận MVO REF-ONLY clean-room kèm checklist và lint rule ở trên.
- Xác nhận giả định sản phẩm proprietary và quy tắc asset (CF-025, CF-026).
- **Khả năng đảo ngược:** Dễ nới sau (cho phép thêm); khó siết sau nếu đã lỡ nhập mã không rõ nguồn.
- **Bị chặn trong lúc chờ duyệt:** Bundle asset, port mã từ repo tham chiếu, mọi mã MVO-ý-tưởng.

## 12. Reversal conditions

- Luật sư xác nhận cách diễn giải AGPL khác: nới quy tắc.
- Chủ sở hữu chọn phát hành mã nguồn mở tương thích: cập nhật verdict.
- Phát hiện asset vi phạm license: gỡ và thay thế.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-012, §2 Q-15 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Được tái dùng code Apache-2.0 và MIT sau khi xác minh license và attribution. My Virtual Office (AGPL-3.0) chỉ làm tham khảo ý tưởng; không sao chép code, prompt, layout hoặc asset.
- Asset theo Q-15: chỉ CC0 mặc định; font OFL được phép nếu lưu đầy đủ license/NOTICE; CC-BY chỉ khi Owner phê duyệt từng package.
- **Xung đột:** danh sách asset trong ADR này (CC0/MIT/OFL) và `research/LICENSE_AND_REUSE.md:134` (CC0 hoặc MIT/CC-BY) rộng hơn Q-15. OWR-2 không nhắc MIT cho asset: coi là chưa được duyệt (register P-9, X-5). Giấy phép sprite JIK-A-4 vẫn chưa có bằng chứng lưu (CF-025).
- **Validation evidence (trung thực):** mức giữ nguyên **Partial** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

## 14. Cập nhật OWR-3 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (OWR-3) (A.9). Các mục 1 đến 13 ở trên giữ làm lịch sử; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**. Việc đồng bộ tài liệu khác do agent khác thực hiện, xem `docs/DOC_SYNC_LOG_OWR3_A.md` và `docs/DOC_SYNC_LOG_OWR3_B.md`; ADR này không khẳng định đã đồng bộ.

- Decision status giữ **APPROVED**; điểm asset MIT là **APPROVED_CONDITIONAL**.
- A.9 (Q-15): asset MIT chỉ được dùng nếu xác minh được nguồn gốc, license và ghi attribution vào NOTICE; **không rõ nguồn gốc thì không dùng**; CC0 vẫn là mặc định. Đóng P-9 và X-5.
- Hệ quả: sprite JIK-A-4 (CF-025) chưa có bằng chứng license/nguồn gốc thì không được dùng cho đến khi có. Danh sách CC0/MIT/OFL ở các mục trên được hiểu theo quy tắc này.
- **Validation evidence (trung thực):** mức giữ nguyên (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
