# Trạng thái dự án

Ngày đối chiếu: 09/10/2026. Các kết quả dưới đây là trạng thái từ log; chưa có manifest đủ để gán release tag đã kiểm chứng.

| Phạm vi | Trạng thái | Giới hạn |
|---|---|---|
| SPL load và hash check | Có PASS trong log HW24 | Không tự chứng minh authenticated boot |
| SP_MIN → OP-TEE → U-Boot | Có bằng chứng chạy | Hai TF-A có build banner khác nhau |
| OP-TEE regression | 104 case, 27.002 subtest PASS | Baseline chưa được nối với HW24 bằng hash |
| FPGA load | OPEN / FAIL | Data abort trong hai lần thử |
| Kernel driver/camera pipeline | Chưa xác nhận từ log hôm nay | Không ghi PASS dựa vào mục tiêu spec |
| Clean source build qua repo này | NOT VERIFIED | Thiếu source lock, patch, config và script |

Xem [daily](daily/2026/2026-10-09.md) và [lỗi HW24](troubleshooting/HW24-fpga-load-data-abort.md).

## Quy ước nhánh và tag đề xuất

- main: tài liệu và trạng thái tích hợp được giữ chính xác; chưa gọi stable khi chưa qua gate.
- Nhánh exp/hw24-fpga-load: nghiên cứu lỗi nạp FPGA.
- Tag theo milestone sau khi đã kiểm chứng: ví dụ v0.1-bootchain, v0.2-optee-regression. Đây là ví dụ đặt tên, chưa phải tag đã tồn tại.
- Tag release trỏ tới commit cụ thể; không dời tag đã công bố sang code mới. Mỗi release gắn source lock và run manifest.

README phải nói rõ bản nào dev mới nên dùng. HEAD mới nhất có thể vẫn đang thử nghiệm.

