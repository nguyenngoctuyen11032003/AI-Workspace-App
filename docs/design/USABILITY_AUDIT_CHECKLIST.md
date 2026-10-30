# USABILITY_AUDIT_CHECKLIST.md — Checklist kiểm tra usability theo từng màn hình

Mục đích: QA, Product Evaluator và UX Research dùng chung một danh sách để không coi một trang là hoàn thành chỉ vì nó hiển thị được. Ánh xạ trực tiếp 10 bước kiểm tra chất lượng của chủ sở hữu (Prompt 3, mục 14).
Phạm vi: mọi màn hình trong `SCREEN_SPECIFICATION.md`. Trạng thái: **[ĐXuất]**, chờ chủ sở hữu duyệt.
Cách dùng: mỗi màn hình có một bản sao checklist; mỗi mục ghi Pass, Fail hoặc N/A kèm bằng chứng (ảnh chụp, video, đường dẫn test). Fail mức P0 chặn hoàn thành.

## 1. Mười bước của chủ sở hữu
| # | Bước | Mục kiểm tra | Bằng chứng |
|---|---|---|---|
| 1 | Bố cục | Có tiêu đề trang, mô tả ngắn khi cần, một hành động chính, các hành động phụ; căn lưới đúng `DESIGN_TOKENS.md`; không card lồng card | Ảnh chụp ở 1440, 1024, 768, 390 px |
| 2 | Typography | Chỉ dùng thang cỡ chữ đã định nghĩa; mono cho ID, log, nhánh; nội dung quan trọng không nhỏ hơn 13 px | Danh sách cỡ chữ từ DevTools |
| 3 | Spacing | Mọi khoảng cách là bội số của 4 px; không tự đặt giá trị lẻ | Kiểm tra lint token |
| 4 | Màu sắc | Chỉ dùng token; mỗi màn hình dùng tối đa một màu nhấn hành động; màu trạng thái nhất quán; không gradient hay glow | Quét CSS tìm hex ngoài token |
| 5 | Responsive | Không tràn ngang; hit target tối thiểu 40 px (touch 44 px); hành vi đúng theo `RESPONSIVE_DESIGN.md` | Test Playwright theo ma trận viewport |
| 6 | Accessibility | Điều hướng bàn phím đầy đủ, thứ tự focus hợp lý, focus ring rõ, nhãn truy cập, tương phản AA, `prefers-reduced-motion` | Kết quả axe và duyệt tay |
| 7 | Trạng thái | Có loading (skeleton), empty, error, success; error có hành động khắc phục; không spinner vô hạn | Ảnh chụp từng trạng thái |
| 8 | Tương tác thật | Mọi nút có chức năng nối với backend; không dữ liệu giả; hành động nguy hiểm có xác nhận | Test E2E trên dữ liệu thật |
| 9 | Nhất quán | Dùng đúng component của Design System; cùng một thao tác giống nhau ở mọi màn hình; thuật ngữ thống nhất | Đối chiếu danh sách component |
| 10 | Người mới | Một người chưa biết sản phẩm hoàn thành tác vụ chính không cần hướng dẫn | Kết quả usability test |

## 2. Mục dành riêng cho điều hành nhiều agent
| # | Mục kiểm tra | Mức |
|---|---|---|
| M1 | Trong 5 giây, người dùng chỉ ra được agent nào cần mình (`blocked`) | P0 |
| M2 | Trạng thái và tiến độ lấy từ sự kiện thật; không có `working` khi không có event trong TTL | P0 |
| M3 | Với 18 agent: danh sách vẫn đọc được, có nhóm theo phòng ban và bộ lọc "cần chú ý" | P0 |
| M4 | Từ thông báo đến hành động xử lý không quá 3 lần nhấp | P1 |
| M5 | Nút Stop all luôn truy cập được, hiển thị trạng thái thật theo từng agent sau khi dừng | P0 |
| M6 | Huy hiệu `done` chỉ mất khi chính client đó đã xem | P1 |
| M7 | Hành động Approve và Reject khác nhau rõ ràng về vị trí, màu và nhãn; Reject bắt buộc có feedback | P0 |
| M8 | Chi phí và token hiển thị kèm nhãn "ước tính" khi nguồn là ước tính | P1 |

## 3. Mục dành riêng cho Virtual Office
| # | Mục kiểm tra | Mức |
|---|---|---|
| O1 | Mọi chuyển động và animation có event thật đứng sau (kiểm tra bằng replay xác định) | P0 |
| O2 | Có chế độ danh sách/bảng thay thế, không bắt buộc đi qua Office để quản lý agent | P0 |
| O3 | Nhấp nhân vật mở panel chi tiết với đủ hành động trong đặc tả | P0 |
| O4 | 18 nhân vật chạy mượt trên laptop mục tiêu, không giật khi có luồng event dày | P1 |
| O5 | Không chồng lấn nhãn và bong bóng thoại khi nhiều agent ở cùng phòng | P2 |
| O6 | Trên mobile Office hiển thị dạng danh sách trạng thái | P1 |

## 4. Mức độ lỗi
- **P0**: chặn hoàn thành màn hình. Sai sự thật dữ liệu, mất khả năng điều khiển, lỗi an toàn.
- **P1**: sửa trước khi nghiệm thu phase.
- **P2**: ghi vào backlog, sửa khi có thể.

## 5. Quy trình
1. UI/UX Designer và Frontend tự kiểm tra trước khi yêu cầu review.
2. QA và Test Automation chạy các mục có thể tự động hóa (5, 6, 7, 8).
3. UX Research thực hiện mục 10 và M1, M3 với người dùng thật khi có thể.
4. Product Evaluator tổng hợp, không cho đánh dấu hoàn thành nếu còn P0.

## 6. Acceptance Criteria
- [ ] Mỗi màn hình trong Phase 3 có một bản checklist đã điền kèm bằng chứng.
- [ ] Không còn mục P0 ở trạng thái Fail khi nghiệm thu.
- [ ] Mục M1 đo bằng thao tác thật với ít nhất 3 người dùng, tỉ lệ thành công tối thiểu 80% [ĐXuất].

## 7. Vấn đề chưa giải quyết
1. Ngưỡng 80% ở M1 và độ lớn mẫu cần chủ sở hữu duyệt.
2. Chưa có người dùng thật ngoài chủ sở hữu để làm usability test [GĐ].
