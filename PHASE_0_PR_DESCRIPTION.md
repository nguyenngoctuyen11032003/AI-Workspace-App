# PHASE_0_PR_DESCRIPTION.md

Chuẩn bị theo OWR-3 mục F. **Chưa mở PR.** Kiểm tra lúc kết thúc: branch đã có trên `origin` tại `ffa065e` (hai commit); agent không push, nên do Owner thực hiện. Remote: `origin` (GitHub). Branch: `docs/phase0-owner-decisions`, đi từ `main` (`32e7155`). Branch đã ở remote; để mở PR dùng link `/pull/new/docs/phase0-owner-decisions` của remote. Phần thay đổi chưa commit (OWR-3) sẽ nằm trong commit tiếp theo, sau khi Owner cho phép.

**Title:**
```
docs(phase0): ghi nhận quyết định của Owner, đồng bộ tài liệu và bổ sung tiêu chí nghiệm thu
```

**Description:**
```markdown
## Vì sao
Phase 0 cần một nguồn quyết định rõ ràng trước khi bắt đầu Phase 1. Quyết định của Owner nằm rải rác trong tin nhắn, còn tài liệu nguồn có giá trị trái ngược nhau (ORM, tên event, timeout approval, retention audit, mức quyền, concurrency). Nhiều yêu cầu chức năng chưa có tiêu chí nghiệm thu.

## Thay đổi
- Ghi nhận phản hồi của Owner (OWR-1, OWR-2, OWR-3) làm nguồn truy vết trong `docs/owner-responses/`, và sổ quyết định `PHASE_0_OWNER_DECISION_REGISTER.md`.
- 21 ADR theo cấu trúc thống nhất (thêm ADR-0021 cho nạp skill Markdown động). Trạng thái: 16 APPROVED, 5 APPROVED_CONDITIONAL, không mục nào VERIFIED.
- Đồng bộ tài liệu nguồn theo các quyết định đã duyệt, có nhật ký từng thay đổi (`docs/DOC_SYNC_LOG_*`). Những giá trị phụ thuộc quyết định chưa duyệt được để lại có đánh dấu, không ghi đè.
- Bảng truy vết đủ 199 yêu cầu chức năng. Thêm tiêu chí nghiệm thu ở trạng thái PROPOSED.
- Ba báo cáo PoC trong `docs/poc/`: Drizzle (9 pass, 1 fail, 1 blocked, trên PGlite), skill động và resume (SK-04 chỉ mô phỏng), Docker (chưa được phép chạy).
- Báo cáo rà soát độc lập vòng hai và ba, checklist bảo mật, sổ rủi ro, báo cáo cuối Phase 0.

## Quyết định cần biết
- Event lưu trong database dạng dotted lowercase; 16 event gốc của Owner được chép nguyên văn. Bảng ánh xạ chưa được Owner cung cấp nên ánh xạ chỉ là đề xuất.
- Mặc định 5 agent là giới hạn scheduler nội bộ, không phải giới hạn provider bảo đảm.
- Chỉ thị mới của Owner (không chạy mã tùy ý trên host) chặt hơn mô hình hai tầng đã duyệt. Xung đột được ghi lại, chưa tự quyết.

## Deploy
- Không có mã, migration hay thay đổi cấu hình. Chỉ tài liệu Markdown.
- Không có việc ops phải làm tay.

## Kiểm
- `git diff --check`: không lỗi. Quét mẫu bí mật trên diff và file mới: 0 kết quả.
- Truy vết: 199 yêu cầu, mỗi yêu cầu một dòng, kiểm bằng lệnh.
- Rà soát độc lập vòng ba: 0 P0, 1 P1, 9 P2, 8 P3. Các sửa sau đó chưa được rà lại.
- Không chạy kiểm thử ứng dụng vì chưa có mã.

## Còn lại (không thuộc PR này)
- Phase 0 vẫn mở, Phase 1 chưa được cho phép.
- Docker, Conductor, kiểm thử agent thật qua SDK và Codex chưa được phép chạy.
- Các câu hỏi chờ Owner: bảng ánh xạ event, X-13, Q-21 đến Q-26.
```
