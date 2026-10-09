# Đầu vào board DE10 Nano

Các file cần nhập và mô tả:

- Handoff DDR, clock, pinmux và version công cụ tạo handoff.
- Source DTS cùng mọi include/patch; lệnh tạo DTB.
- Memory map: địa chỉ load, entry, vùng OP-TEE/TF-A, reserved-memory và vùng staging.
- SD layout: partition table, sector size, raw image offsets, header/length và alignment.
- Boot environment, boot args, tên image và đường dẫn filesystem.
- Cấu hình switch/jumper, thẻ SD, UART và cách bật nguồn đã kiểm chứng.
- Nếu build FPGA: project Quartus, Qsys/Platform Designer, QSF, SDC, HDL và dependency cần thiết.

Không hardcode /dev/sdX hoặc raw LBA của một thẻ vào hướng dẫn chung. Các giá trị phải lấy từ layout thuộc baseline đang dùng. Địa chỉ trong daily là bằng chứng lịch sử, không tự trở thành cấu hình build chuẩn.

