# Cấu hình build

Lưu defconfig, config fragment, platform config, rootfs overlay và các tham số build có ảnh hưởng tới output. Ghi cả lệnh tạo config cuối cùng.

Mỗi config cần có component và revision tương ứng. Không chỉ chép một dòng CONFIG_TEE hoặc một ảnh chụp menuconfig; cần tái tạo được cấu hình đầy đủ.

Manifest phải ghi vai trò toolchain cho SPL/U-Boot/TF-A, kernel, OP-TEE core, TA và userspace. Không mặc định các thành phần có cùng target triple hoặc ABI.

Chưa có config thực của bản boot thành công được nhập.

