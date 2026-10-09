# Build và boot

Phần này là đường đi chuẩn cho dev mới. Các bước sửa lỗi lịch sử nằm ở docs/troubleshooting/, không được trở thành điều kiện ngầm để build.

## Đầu vào bắt buộc

| Nhóm | Cần bổ sung từ máy đã build |
|---|---|
| Source | U-Boot/SPL, TF-A, OP-TEE OS/client/test, Linux và hệ tạo rootfs thực tế |
| Revision | Remote URL tải được, full commit, patch series hoặc fork đã push |
| Config | Defconfig, config fragment, platform Makefile, build flags và biến môi trường |
| Board | Handoff DDR/clock/pinmux, DTS/DTB nguồn, memory map, SD layout, boot args |
| Toolchain | Tên, version, target triple, ABI, download URL và checksum |
| Scripts | Lệnh thực tế theo đúng phụ thuộc và thứ tự build |
| Validation | UART log từ power-on, lệnh test, exit code và run manifest |

[Quickstart](../docs/quickstart.md) là hướng dẫn chính. Chưa được điền lệnh build giả hoặc khẳng định build thành công khi mới tạo cấu trúc.

Bản source được tái tạo trong work/. Các output nằm trong out/<build_id>/. Một build_id phải gắn đúng source manifest, build log, file ELF/map, binary và checksum.

Nếu mục tiêu baseline bao gồm FPGA programming, bổ sung nguồn Quartus hoặc RBF xác định cùng hash, cấu hình board, yêu cầu tool và provenance của RBF. Phân biệt rõ sử dụng RBF phát hành sẵn với tự build RTL.

