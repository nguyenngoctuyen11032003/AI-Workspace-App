# POC_DOCKER_SANDBOX.md — Kiểm chứng sandbox Docker

Ngày: 2026-10-30. Cơ sở: OWR-2 mục 5.3 và Q-10. Trạng thái tổng: **BLOCKED, chờ Owner tự mở Docker Desktop.** Chưa có kết quả kiểm thử nào. Không có mục nào được coi là PASS.

## 1. Vì sao BLOCKED
OWR-2 mục 5.3 quy định Owner tự mở Docker Desktop, chờ Docker Engine sẵn sàng, rồi mới chạy kiểm tra. Tôi không khởi động Docker, không thay đổi cấu hình WSL và không chạy `wsl --shutdown`.

Bằng chứng kiểm tra hôm nay (13:39 giờ máy, lệnh chỉ đọc):
```text
$ docker version
Client:
 Version:           29.8.1
 API version:       1.56
 Context:           desktop-linux
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine;
check if the path is correct and if the daemon is running:
open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.

$ docker info
(chỉ hiện phần Client; không kết nối được daemon)
```
Tiến trình liên quan đang chạy: chỉ `wslservice` (14 MB). Không có `vmmem` hay `Docker Desktop`.

## 2. Đường cơ sở tài nguyên (trước khi bật Docker)
| Chỉ số | Giá trị | Nguồn |
|---|---|---|
| RAM tổng | 15,77 GB | `Get-CimInstance Win32_OperatingSystem`, 2026-10-30 13:39 |
| RAM trống | 4,04 GB | cùng lệnh |
| Docker/WSL | chưa chạy | `Get-Process` |
Docker Desktop theo tài liệu cần khoảng 2 đến 4 GB RAM cho VM. Với 4,04 GB trống, có nguy cơ thiếu RAM. Nếu máy mất ổn định thì dừng Docker Desktop theo OWR-2 5.3.

## 3. Quy trình Owner thực hiện
1. Đóng các ứng dụng nặng (trình duyệt nhiều tab, IDE phụ) nếu cần, rồi ghi RAM trống: `powershell -NoProfile -Command "(Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB"`.
2. Tự mở Docker Desktop. Chờ biểu tượng báo Engine running.
3. Chạy `docker version` và `docker info`. Dán kết quả vào đây hoặc báo cho agent.
4. Ghi RAM trống lần nữa. Ghi chênh lệch.
5. Nếu RAM trống dưới khoảng 1 GB hoặc máy giật: dừng Docker Desktop và báo. Không tiếp tục kiểm thử.
6. Không bật tự khởi động Docker cùng Windows. Không đổi `.wslconfig`.

## 4. Tập kiểm thử sẽ chạy sau khi Docker sẵn sàng
Chỉ các kiểm thử không phá hủy, nằm trong phạm vi OWR-2 mục 5.3 (cách ly container, giới hạn tài nguyên, cách ly credential, dừng khẩn cấp, không lộ Docker socket). Dùng ảnh nhỏ, tải một lần (`alpine`), container dùng một lần, có `--rm`, không mount thư mục home, không mount `docker.sock`.
| ID | Nội dung | Điều kiện đạt | Trạng thái |
|---|---|---|---|
| VAL-DOCKER-01 | Daemon sẵn sàng, ghi RAM trước và sau | `docker info` thành công, có số liệu RAM | BLOCKED |
| VAL-DOCKER-02 | Cách ly container: người dùng không root, không `--privileged` | `docker inspect` cho `Privileged=false`, `User` khác root | BLOCKED |
| VAL-DOCKER-03 | Giới hạn filesystem: rootfs chỉ đọc, không mount ngoài thư mục workspace thử | Ghi ngoài volume thử bị từ chối | BLOCKED |
| VAL-DOCKER-04 | Giới hạn mạng: `--network none` | Không kết nối ra ngoài được | BLOCKED |
| VAL-DOCKER-05 | Giới hạn tài nguyên: `--cpus`, `--memory`, `--pids-limit` | `docker inspect` khớp, quá giới hạn bị chặn | BLOCKED |
| VAL-DOCKER-06 | Cách ly credential: không mount thư mục home hay credential, không biến môi trường nhạy cảm | Trong container không thấy file credential của host | BLOCKED |
| VAL-DOCKER-07 | Docker socket không lộ vào container của agent | Không có `/var/run/docker.sock` trong container | BLOCKED |
| VAL-DOCKER-08 | Capabilities: `--cap-drop ALL`, `no-new-privileges` | `docker inspect` khớp | BLOCKED |
| VAL-DOCKER-09 | Dừng khẩn cấp: `docker kill` và dọn container | Container dừng trong thời gian giới hạn, không còn tiến trình | BLOCKED |
Không chạy kiểm thử stress, phá hủy hay liên quan credential thật. Nhãn "Fail closed khi Docker tắt" (D-007) kiểm riêng ở tầng ứng dụng khi có mã, không kiểm ở đây.

## 5. Ghi chú đối chiếu tài liệu
- Mọi thông số sandbox (cgroup, ECI, rootless, egress allowlist) vẫn là Giả định cho đến khi các mục trên PASS (`docs/security/SANDBOX_DESIGN.md`).
- ADR-0007 (D-007 APPROVED có điều kiện) có điều kiện "test SB-00..SB-17 phải qua trên máy có Docker". Tập ở mục 4 là một tập con. Phần còn lại thuộc runbook chưa được cấp phép.
- Kết quả của mục này **không** được dùng để tuyên bố ADR-0007 VERIFIED nếu chỉ một phần chạy.

## 6. Hành động tiếp theo
Chờ Owner thực hiện mục 3. Sau khi có `docker version` và `docker info` thành công, agent chạy mục 4 và cập nhật file này.
