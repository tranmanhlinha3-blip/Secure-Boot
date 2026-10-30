# REPRODUCE — SECURE BOOT DE10-NANO — 30/10/2026

**Ngày:** 30/10/2026  
**Máy phát triển:** Windows + WSL Ubuntu; project `~/secure_imgproc`  
**Board mục tiêu:** DE10-Nano / Intel Cyclone V SoC (ARM Cortex-A9)  
**Mục tiêu:** tạo full-stack SD image tự động boot với SPL → TF-A/SP_MIN → U-Boot R24 → signed FIT R36 → Linux 5.15.221 → `/init`, đồng thời xác minh RSA FIT và tìm điểm treo Linux bằng dấu mốc R36-HANDOFF.

## 1. Tóm tắt kết quả

- **PASS offline:** U-Boot R24 có `bootcmd`, `bootargs` tích hợp; R36 Linux được build; R36 FIT xác minh RSA offline; image R37 có kích thước, SHA-256, FIT readback hợp lệ; vùng boot của raw image chứa FDT với `key-dev` và `required=conf`.
- **FAIL / blocker:** SD gốc `/dev/sde` (4.007.657.472 byte) có kết quả đọc toàn thẻ không nhất quán giữa hai lượt. Chưa có backup full-card được xác minh; không thể an toàn ghi đè SD duy nhất.
- **Chưa thực hiện:** Flash R37 vào SD và cold boot toàn chuỗi trên board. Không được ghi trạng thái PASS đối với automatic secure boot phần cứng.

## 2. Các mốc R19 / R24 / R31 / R36

| Mốc | Nội dung | Bằng chứng / trạng thái |
|---|---|---|
| R19 | FIT trước đó; kernel 5.15.221, clocksource timer2 25 MHz, giải nén initramfs và nhiều late initcalls | Đã boot thủ công trên board; dừng sau dòng `dw-apb-uart ffc02000.serial0: forbid DMA for kernel console`; chưa thấy `/init`. Đây không nhất thiết là lỗi DMA. |
| R24 | U-Boot tích hợp `CONFIG_FIT_SIGNATURE=y`, `CONFIG_RSA=y`, `CONFIG_BOOTDELAY=5` | PASS cấu hình và binary. `bootcmd` tự gọi `mmc dev 0; mmc rescan; mmc read 0x08000000 0x2f800 0x40d9; bootm 0x08000000#conf-fullstack` với xử lý lỗi. |
| R24 | Bootargs mặc định | `CONFIG_USE_BOOTARGS=y`; binary có `bootargs=console=tty0 earlycon=uart8250,mmio32,0xffc02000 keep_bootcon ignore_loglevel loglevel=8 initcall_debug log_buf_len=1M rdinit=/init mem=992M panic=0 initcall_blacklist=socfpga_dwmac_driver_init,dwmac_generic_driver_init`. Không cần thêm bootargs vào cấu hình chỉ để test R37. |
| R31 | Image nền chứa U-Boot mới / layout SD | Dùng làm image nền cho R37; bước kiểm tra đầu vào R37 `BASE_R31_SHA256=PASS`. |
| R36 | Kernel chẩn đoán Linux 5.15.221, thêm 17 marker `R36-HANDOFF` tại handoff / `console_on_rootfs` / `/init` | Kernel build PASS. FIT R36 size **8.097.098 byte**, SHA-256 `85410e7bec2aae2654ff0e16fcbf5b12be60951665b2b06e18121a9fa3b183ef`, RSA offline PASS. Chưa boot R36 trên board. |

**Lưu ý:** Kết quả build và kiểm tra offline không thay thế cho xác nhận kernel chạy tới userspace trên board.

## 3. R37 integrated image — kiểm tra PASS

**Artifact:** `~/secure_imgproc/P74R5R37_integrated_sd/R37_UBOOT_R24_KERNEL_R36.img`  
**R36 FIT:** `~/secure_imgproc/P74R5R36_kernel_fix/artifacts/R36_signed.itb`

- R37 builder đã xác nhận `BASE_R31_SHA256=PASS`, `NEW_R36_SHA256=PASS`, `R36_RSA_VERIFY=PASS` nhưng một lần chạy lại dừng với `Output exists`, vì image đã tồn tại. Đây là bảo vệ không ghi đè, không chứng minh build mới thất bại.
- R37D kiểm tra image có sẵn: `IMAGE_SIZE=PASS`, `R36_FIT_SHA256=PASS`, `R31_BOOT_PREFIX_PRESERVED=PASS`, `R36_SIGNED_FIT_READBACK=PASS`, `FIT_SLOT_TAIL_CLEAR=PASS`, `OTHER_REGION_SAMPLE_CHECK=PASS`, `FULL_IMAGE_SHA256=PASS`, `FIT_RSA_OFFLINE=PASS`.
- R37E: `CONFIG_OF_SEPARATE=y`; `build/u-boot.dtb` chưa có `/signature`, nhưng bản raw image chứa một FDT tại offset `0x1bd07c` với `key-dev`, `required=conf`: `REQUIRED_CONF_KEYS_FOUND=1`, `RESULT=R37E_KEY_PRESENT_IN_RAW_BOOT_REGION`.
- Kết quả R37E **chứng minh khóa hiện diện trong raw image**, nhưng chưa phải bằng chứng thực thi trên board hoặc một phép thử từ chối FIT bị giả mạo.

## 4. Kích thước image / SD

```text
stat -c '%s bytes' R37_UBOOT_R24_KERNEL_R36.img => 4007657472 bytes
du -h R37_UBOOT_R24_KERNEL_R36.img          => 9.2M
sudo blockdev --getsize64 /dev/sde         => 4007657472
signed FIT R36                              => 7.8M (du)
```

**Giải thích:** Image là *sparse raw image*: kích thước logic bằng 4.007.657.472 byte, trong khi vùng dữ liệu thực dùng trên WSL chỉ khoảng 9,2 MiB. Các sector trống không chiếm tương ứng 4 GB trên host; khi ghi raw image, đích vẫn cần đủ 4.007.657.472 byte. Không phải cần chừa riêng vùng reserved nội bộ SD controller trong số byte logic được hệ điều hành báo.

**Không bắt buộc dùng thẻ 8 GB.** Thẻ SD test 4 GB khác vẫn đủ nếu dung lượng *thực* ít nhất 4.007.657.472 byte; một số thẻ bán là 4 GB có thể ít byte hơn. SD test 8/16 GB được khuyến nghị để tránh giới hạn này và bảo vệ thẻ gốc.

## 5. SD audit / flash blocker

Thiết bị được nhận diện trong WSL:

```text
DEVICE=/dev/sde
SERIAL=121220160204
BYTES=4007657472
MODEL=Storage Device
TRAN=usb
```

R37 V2/V3/V4 audit đọc 5 MiB đầu:

```text
CARD_PREFIX_SHA256=994bd21069122ff27596cf936f270d31e0bdbf1c9f8264d44da6b6b70db1551d
REFERENCE_PREFIX_SHA256=994bd21069122ff27596cf936f270d31e0bdbf1c9f8264d44da6b6b70db1551d
ORIGINAL_READER_SERIAL_MATCH=YES
ORIGINAL_PREFIX_MATCH=YES
AUDIT_SD_WRITE=NO
RESULT=BLOCKED_R37_ORIGINAL_CARD_OR_READER
```

- `/dev/sdf: not a block device`: không có thiết bị `/dev/sdf` khi chạy lệnh. Một lần nhập `flash "$DEV` thiếu dấu đóng nháy, Bash hiện dấu `>`; đã Ctrl+C, không flash.
- Các lệnh flash lên `/dev/sde` bị **chặn trước khi ghi**; các thông báo `WRITE BLOCKED` là guard an toàn, không phải kernel hoặc image vượt dung lượng.
- Thẻ đọc qua WSL USB/IP; serial `121220160204` có thể là định danh đầu đọc, không phải duy nhất của thẻ.

## 6. R37G — sao lưu toàn SD thất bại

Script: `~/secure_imgproc/P74R5R37_integrated_sd/R37G_single_sd.sh`  
Mode: `backup /dev/sde`

```text
BACKUP_PASS_1=READ_FULL_CARD
4007657472 bytes copied, ~451.6 s, 8.9 MB/s
BACKUP_PASS_2=INDEPENDENT_CARD_READ
4007657472 bytes copied, ~414.8 s, 9.7 MB/s
RESULT=FAIL_R37G: Card read #2 mismatches backup; STOP
```

Hai lượt đều trả về đủ số byte nhưng SHA-256 khác nhau. **Chưa có backup được xác minh, chưa thực hiện flash.** Cần kiểm tra SD, đầu đọc, đường USB/IP hoặc nguyên nhân nội dung thay đổi; chưa đủ bằng chứng xác định phần cứng nào lỗi.

## 7. R37H — đọc vùng mẫu và USB/IP

Script: `~/secure_imgproc/P74R5R37_integrated_sd/R37H_sd_read_diagnostic.sh`  
Dmesg có `vhci_hcd` USB/IP, quá trình USB disconnect/reconnect; không thấy trong log trích dẫn một lỗi sector I/O cụ thể đủ kết luận thẻ hỏng.

```text
REGION=BOOT
OFFSET_MIB=0
dd: IO error: Invalid input
RESULT=FAIL_R37H: First read failed at BOOT
```

Lỗi khi dùng `iflag=direct` có thể do không hỗ trợ/không tương thích direct I/O của đường USB/IP/đầu đọc; riêng lỗi này **không chứng minh thẻ SD hỏng**. Nhưng checksum full-card khác nhau từ R37G là bằng chứng độc lập về sự thiếu nhất quán cần xử lý. R37H đã dừng ở vùng BOOT; chưa có kết quả 508–516 MiB hay cuối thẻ của lần chạy này.

## 8. Trạng thái cuối ngày

| Công việc | Trạng thái |
|---|---|
| U-Boot R24 bootcmd/bootargs tích hợp | PASS offline |
| R36 zImage và signed FIT RSA | PASS offline |
| R37 full-stack raw image (size, sha, FIT, prefix) | PASS offline |
| RSA `key-dev`, `required=conf` trong FDT ở vùng boot raw image | PASS hiện diện |
| Ổn định đường đọc của SD gốc | FAIL / chưa khắc phục |
| Sao lưu full SD có SHA-256 đối chứng | FAIL |
| Flash R37 lên SD | CHƯA LÀM (không có SD write) |
| Hardware autoboot + FIT authentication | CHƯA KIỂM THỬ |
| Linux R36 đạt `/init` | CHƯA KIỂM THỬ |

## 9. Reproduce tiếp theo khi có SD test khác

1. Giữ nguyên SD gốc. Ưu tiên một SD test 8/16 GB hoặc 4 GB có số byte thực đủ dùng; nên dùng đầu đọc USB khác nếu có.
2. Xác định đúng thiết bị SD mới bằng `lsblk -o NAME,SIZE,TYPE,TRAN,MODEL,SERIAL,MOUNTPOINTS`, kiểm tra dung lượng thật bằng `sudo blockdev --getsize64 /dev/sdX`; không mặc định `/dev/sde` hoặc `/dev/sdf`.
3. Kiểm tra không mount, kiểm tra fingerprint thẻ test không phải dữ liệu gốc; xác minh image R37 SHA-256. Chỉ flash lên thẻ mới được phép xóa sau khi xác nhận đúng device; kiểm tra readback.
4. Gắn vào DE10-Nano, PuTTY COM6 `115200 8N1`, cấp nguồn và **không nhập lệnh U-Boot**.
5. Thu log SPL → TF-A → U-Boot R24 → `mmc read` → RSA verification → Linux 5.15.221 → `R36-HANDOFF` → `/init` và xử lý điểm dừng thực tế.

**Checkpoint kết luận:** `R37_OFFLINE_PASS / SD_ORIGINAL_READ_UNSTABLE / FLASH_NOT_PERFORMED / BOARD_AUTOboot_PENDING`.
