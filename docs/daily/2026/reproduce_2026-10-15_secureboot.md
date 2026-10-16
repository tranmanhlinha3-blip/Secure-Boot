# Daily Reproduce — Secure Boot / Kernel Driver — 2026-10-15

**Platform:** Terasic DE10-Nano / Cyclone V SoC, ARM Cortex-A9 (ARM32)  
**Host:** WSL Ubuntu, cross compiler `arm-linux-gnueabihf-gcc`  
**Kernel:** Linux `5.15.221-dirty` (`SMP mod_unload ARMv7 p2v8`)  
**Stack:** SPL → TF-A AArch32 SP_MIN v2.13 → OP-TEE 3.21 → U-Boot → Linux → FPGA kernel driver / image-processing TA  
**Phạm vi báo cáo:** các lần reproduce và kết quả quan sát được ngày 15/10/2026. `PASS` chỉ áp dụng ở cấp kiểm thử được ghi cụ thể; không suy ra toàn bộ secure boot hoặc 7 ioctl đã hoàn thành.

## 1. Executive summary

- **PASS trên DE10-Nano (baseline đã chạy hôm nay):** R34E `SET_CONFIG`, threshold và `GET_RAW_DIAG` cross-check với MMIO; pixel counter tăng từ 0 lên 1; `RESET` phục hồi thanh ghi, FPGA vẫn `operating`. Linux 5.15 đã boot sau khi vô hiệu hóa `timer1@ffc09000` **chỉ trên DTB trong RAM**. HW88: OP-TEE 3.21, `xtest 1006`, image-processing CA (SIGN/VERIFY/WATERMARK) và read-only SMC probe đều PASS trong các phạm vi tương ứng.
- **PASS trên WSL / host:** phục hồi source R34E đúng module SHA lịch sử; rebuild R34E; tạo ABI unified R35K_2E; tích hợp kernel TEE Client, sửa SIGN/WATERMARK ABI theo source TA HW76; build ARM32 `R35K_3B` và userspace regression static; tạo initramfs R35K_3D mới, kiểm tra checksum và nội dung CPIO.
- **Chưa PASS trên hardware đối với R35K mới:** R35K_3D chưa boot trên board, R35K_3B ioctl qua kernel TEE chưa được thực thi; `GET_STATUS` còn thiếu backend, HWUID firmware chưa hỗ trợ, DMA/IRQ/V4L2 chưa hoàn chỉnh, xác thực chữ ký image/kernel ở U-Boot chưa được bật/chứng minh.
- **An toàn:** Log cho thấy không ghi SD trong các bước R35K builder/audit; image HW88 và source gốc được giữ nguyên. Chưa có log PuTTY R35K_3E của DTB/FPGA/transfer mới.

## 2. Các bước đã reproduce và bằng chứng

### 2.1 R34E hardware regression và R35A audit

- `HW96E_R34E_A7_RESULT=PASS`, exit code 0. `SET_CONFIG` thành công và `GET_RAW_DIAG` trả `CTRL=0x0000006c`, `STATUS=0x00000001`, `THRESHOLD=0x000000f9`. Lệnh đọc MMIO trực tiếp xác nhận đúng các giá trị. Reset qua ioctl đưa CTRL/threshold về 0 và giữ FPGA `operating`.
- `HW96E_R34E_A8_RESULT=PASS`, exit code 0. Khi enable và nạp 1 pixel: `PIXEL_IN=0x5a`, `PIPE_IN_COUNT=1`; GET_RAW_DIAG báo cùng giá trị; reset đưa counters về 0. **Chưa có bằng chứng pipeline output tăng** ở phép thử một pixel (`PIPE_OUT_COUNT=0`).
- `R34E_A9_0B_BOARD_PREFLIGHT=PASS`: module lịch sử đúng SHA, FPGA `operating`, bridge register `0x00000005`.
- `HW96E_R35A_BOARD_AUDIT_COMPLETE`: `/dev/fpga_imgproc`, `/dev/tee0`, `/dev/teepriv0` tồn tại; `/dev/video0`, `/dev/video1` chưa có. Node `/proc/device-tree/soc/fpga-imgproc@ff200000` xuất hiện, nhưng không có các property `interrupts`, `interrupt-parent`, `dmas`, `dma-names`. Cần kiểm tra lại trên boot R35K mới, không suy ra DMA/IRQ đã sẵn sàng.

### 2.2 U-Boot / Linux / OP-TEE / SMC baseline

- SPL log: tải `atf.bin`, `tee.bin`, `u-boot.img` và kiểm tra **SHA-256 PASS**. **Không được đồng nhất SHA-256 với signature authentication**; log SPL còn ghi `Authentication: NOT ENABLED YET`.
- Audit U-Boot: `help hash` báo `Unknown command`; `run r35b_verify_kernel` báo biến chưa định nghĩa, `kernel_sha_actual` chưa có. Tự động verify kernel bằng script **chưa đạt**.
- Lần boot Linux cũ có clocksource/timer bất thường, dmesg đứng ở timestamp 0.000000. Dùng DTB trên RAM:

  ```text
  fdt addr 0x03000000
  fdt resize 0x1000
  fdt set /soc/timer1@ffc09000 status disabled
  ```

  Sau đó kernel Linux 5.15 boot, `ttyS0` có output, OP-TEE 3.21 probe/initialize, chạy `/init` thành công. Kiểm thử `rdinit=/bin/sh` không phù hợp với initramfs tùy chỉnh (không có `/bin/sh`) và đã gây kernel panic mount root; **không lặp lại**.
- HW88 initramfs + CA: `HW68_XTEST_SMOKE=PASS` (xtest 1006). `IMGPROC_C2C_CA_RESULT=PASS`: SIGN thành công với MAC 32 byte; VERIFY hợp lệ PASS, tamper bị từ chối; WATERMARK 256 byte, 43 byte thay đổi, độ chênh lớn nhất 1; HWUID thất bại theo fail-closed — không có UID thật.
- HW87 read-only SMC probe (`hw88.execute=1`): `HW87_BOOT_STATUS_GATE=PASS`, TF-A 2.13.0, status `0x020d001f`; `HW87_PERF_STATUS_GATE=PASS` về trạng thái lời gọi nhưng hai lần đọc PMCCNTR đều 0 nên `HW87_PERF_PROGRESS=INCONCLUSIVE`; invalid command bị từ chối; `HW87_READONLY_SMC_PROBE=PASS`. `HW87_AUTHENTICATED_BOOT=NOT_PROVEN`.

### 2.3 R35K_2A → R35K_2E: hồi phục source và ABI

1. `R35K_2A_UAPI_LAYOUT=PASS`, preflight phát hiện `fpga_regs.h` và cross-compiler. Tìm được đúng kernel build output tại `optee_phase/GATE_C_R4B_linux515_optee_candidate/20261005_042159/build`: `.config`, `Module.symvers` (8.934 dòng), `UTS_RELEASE=5.15.221-dirty`.
2. R35K_2A module compile ARM32 PASS. SHA-256 `a5f458d1b3140c67e8ac1699eb39f60aba02797abeea2e22bd51bbbca25466d5`.
3. Khôi phục source R34E tại `boot_phaseA/HW96E_R34E_A1_oBxukkLy/`; module lịch sử đúng SHA `d61db1f4f50bdcab51593ecb540ed2163e37ad18200dd233396cecbf0fd23608`. Rebuild R34E PASS, SHA mới `3226da44389eb69af1e4dfe034a996defe76b054bb6138fa9076d1586048a39f`.
4. Merge source R34E + V2, bảo toàn ioctl legacy `0x00–0x05`, kết hợp lệnh V2 `0x10–0x16` (bao gồm truy vấn ABI V2). Giữ logic MMIO config, threshold, reset, raw diagnostic; chưa coi 7/7 ioctl hardware PASS.

### 2.4 R35K_3 → R35K_3B: kernel TEE integration

- R35K_3 tích hợp `tee_client_open_context/open_session/invoke_func` + shared memory; module ARM32 build PASS (SHA `012f5d29bd4af167d92ce2113b58b59d13f60218a3eb16bdc83a6eceb20578ae`).
- Phát hiện ABI khác giữa spec/candidate và source TA thực tế. `CMD_SIGN_FRAME=0`: `MEMREF_INPUT, MEMREF_OUTPUT, NONE, NONE`. `CMD_WATERMARK=1`: nhánh HW76 đã sửa dùng `MEMREF_INOUT, MEMREF_INPUT, VALUE_INOUT, NONE` (nhánh C2C cũ dùng `VALUE_INPUT`).
- R35K_3A chỉnh lại TEE ABI; kiểm tra tĩnh PASS; ARM32 module build PASS, SHA `a523440c76072d58951dadc69ebf1dadc924bb2ec3a4d824d893a48aa9a16e0a`.
- Source TA HW76/HW77 được kiểm tra checksum, command ID, UUID. TA host: `7c9b7d9e-2f1c-4e7b-93a9-6e8d5410c2b1.ta`, SHA `b77551fc135bed301d2f8641090ed84a734369237b3551335ecb84dc30bfbe91`. **Binary TA thực sự đã load trên board cho R35K runtime chưa được xác minh độc lập.**
- R35K_3B: source SHA PASS; `R35K_3_UAPI_LAYOUT=PASS`, `R35K_2E_R34E_HEADER_ABI=PASS`, `R35K_2E_SOURCE_MERGE=PASS`, `R35K_3B_TA_CONTRACT=PASS`; ARM32 module build PASS, `vermagic=5.15.221-dirty SMP mod_unload ARMv7 p2v8`, SHA **`319a16252d1f6dd636747117e44b067efaa4c693d8410b627cc07477e6f4d161`**.
- `test_ioctl_r35k3b_static` là ELF ARM32 statically linked, SHA **`adfcc882e4ab51b4b4fb1295f378e8298f8ba3b9e4895b04da0e1218622b7c30`**. Chưa chạy trên board.

### 2.5 R35K_3C → R35K_3D: chuẩn bị initramfs mới

- R35K_3C inventory PASS: HW88 baseline initramfs `10173748` byte, SHA `735c25fc049b6b20742f24934908135a21d48e6d2f6fae4452e9d148734495cc`, CPIO `ASCII NEWC`, **56 entry**, `/init` ELF tùy chỉnh; module R35K_3B và TA HW77 đúng hash. `R35K_3C_BOOT_INPUT_GATE=PASS`. Boot inputs ZIP SHA `531769ca68bc253da74f3c4b9dbc25835886afe90357f906cec65fcf18f33620`.
- R35K_3D builder cross-compile ARM32 static cho `/init` mới và worker; giữ nguyên 55 payload còn lại, thêm worker, module, test; tổng **59 entry**. `INPUT_ZIP_IDENTITY=PASS`, `HW88_BASE_CONTENT=PASS`, `BASE_UNCHANGED_PAYLOADS=PASS`, `NEW_PAYLOADS=PASS`, `CPIO_NEWC_ROUNDTRIP=PASS`, `R35K_3D_HOST_BUILD=PASS`.
- **Output:** `/mnt/d/SecureBoot_share/R35K_3D_preflight_initramfs.cpio` — **11.536.896 byte = `0xB00A00`**, CRC32 **`9C7621AE`**, SHA-256 **`bd2a84591463bb84013029f521010927895205025c08b69187cbfe69ab90e863`**. Hash đã được đối chiếu độc lập với file người dùng tải lên.
- `/init` mới duy trì flow HW88 trước khi gọi worker. Mặc định chỉ inventory; chưa cho phép tự load module, chưa ghi/reset FPGA, chưa boot bản R35K_3D trên board. `SD_WRITE=NO`, `BOARD_BOOT=NOT_CALLED` trong build log.

## 3. Trạng thái checklist theo specification

| Hạng mục | Trạng thái đến cuối ngày | Giới hạn bằng chứng |
|---|---|---|
| SPL nạp image, SHA-256 | PASS ở baseline | Không chứng minh chữ ký/authenticated boot |
| Linux / OP-TEE / HW88 CA | PASS ở baseline sau patch timer1 trong RAM | TA test key; không chứng minh production security |
| R34E MMIO: SET_CONFIG, threshold, RAW_DIAG, RESET | Hardware PASS | Binary R34E, không áp cho binary R35K mới |
| R35K_3B module build + legacy/UAPI regression | Host/WSL PASS | Chưa insmod/probe bản mới trên board |
| SIGN/WATERMARK kernel ioctl | Implemented + source TA ABI PASS | **Hardware NOT_TESTED** |
| GET_HWUID thật | FAIL-CLOSED / NOT_SUPPORTED | Không có UID thật; chưa đạt spec |
| GET_STATUS đầy đủ | PENDING | RTL backend/counters còn thiếu |
| DMA, IRQ, /dev/video1 | PENDING | DT node cũ thiếu interrupts/dmas, video nodes missing |
| U-Boot kernel verify/signature | PENDING | `hash` command và script verification chưa sẵn sàng |
| R35K_3D initramfs builder | Host PASS | Chưa có log boot R35K_3D |

## 4. Vấn đề phát sinh và cách xử lý

| Vấn đề | Nguyên nhân/bằng chứng | Hướng xử lý/kết quả |
|---|---|---|
| WSL `unzip` / `make uapi-check` báo lỗi | ZIP không có trong working directory; công cụ `unzip` ban đầu chưa cài | Cài `unzip`, dùng đường dẫn chính xác; UAPI test PASS |
| Kernel preflight ban đầu không thấy `.config`, `Module.symvers` | Kiểm tra nhầm kernel tree | Chuyển về `.../GATE_C_R4B_linux515_optee_candidate/20261005_042159/build`; kernel build PASS |
| Bản R35K_2A mới thiếu implementation MMIO R34E | Dựa trên snapshot source 3 ioctl | Tìm lại source/module R34E chuẩn và merge tương thích legacy |
| TA WATERMARK ABI không tương thích nhánh C2C cũ | `VALUE_INPUT` so với HW76 `VALUE_INOUT` | Sửa R35K_3A, xác minh source TA HW76 và CA |
| Linux đứng tại UART hoặc clocksource không chạy | `timer1@ffc09000` trên DTB hiện tại gây bất thường | Disable node trong DTB RAM; HW88 Linux boot PASS |
| `rdinit=/bin/sh` kernel panic | Initramfs tùy chỉnh không chứa shell này | Dùng `rdinit=/init`, giữ HW88 custom init |
| U-Boot thiếu kernel hash command/script | `help hash` unknown; script chưa định nghĩa | Ghi PENDING; chưa chỉnh SD env hay tuyên bố verified kernel |

## 5. Artifact checkpoint

| Artifact | SHA-256 / kiểm chứng |
|---|---|
| R34E module lịch sử | `d61db1f4f50bdcab51593ecb540ed2163e37ad18200dd233396cecbf0fd23608` |
| R34E module rebuild | `3226da44389eb69af1e4dfe034a996defe76b054bb6138fa9076d1586048a39f` |
| R35K_2A module | `a5f458d1b3140c67e8ac1699eb39f60aba02797abeea2e22bd51bbbca25466d5` |
| R35K_3B final WSL module | `319a16252d1f6dd636747117e44b067efaa4c693d8410b627cc07477e6f4d161` |
| R35K_3B static regression | `adfcc882e4ab51b4b4fb1295f378e8298f8ba3b9e4895b04da0e1218622b7c30` |
| HW88 baseline initramfs | `735c25fc049b6b20742f24934908135a21d48e6d2f6fae4452e9d148734495cc` |
| HW77 TA | `b77551fc135bed301d2f8641090ed84a734369237b3551335ecb84dc30bfbe91` |
| R35K_3D initramfs (host-built) | `bd2a84591463bb84013029f521010927895205025c08b69187cbfe69ab90e863` |

## 6. Điểm dừng cuối ngày và kế hoạch reproduce tiếp

**Điểm dừng:** `R35K_3D_HOST_BUILD=PASS`; **chờ `R35K_3E_DTB_AUDIT` và `R35K_3E_PREFLIGHT_COMPLETE` trên PuTTY**. Không có log boot R35K_3D; tuyệt đối chưa gắn nhãn hardware PASS cho image hoặc kernel ioctl mới.

**Thứ tự tiếp tục (không ghi SD):**

1. Tại U-Boot: `mmc dev 0`; đọc DTB hiện tại từ SD vào RAM (`mmc read 0x03000000 0x6000 0x27`), `fdt addr 0x03000000`, `fdt print /soc/timer1@ffc09000`, `fdt print /soc/fpga-imgproc@ff200000`, `fdt list /soc`, `fpga info 0`; kiểm tra `help tftpboot`, `help loady`, `help loadb` và `printenv ipaddr serverip ethaddr`. **Chỉ inventory, không nạp module/không reset FPGA.**
2. Chọn phương thức chuyển **CPIO mới** vào RAM. **Không** dùng lại lệnh `mmc read` HW88 cũ với giả định rằng nó chứa image R35K_3D. Xác nhận kích thước/CRC/hash cùng bản host, bảo vệ vùng RAM kernel/DTB/initrd.
3. Boot R35K_3D lần đầu chỉ ở chế độ inventory, giữ patch timer1 trong RAM và `rdinit=/init`. Không thêm `r35k3d.load=1`, `r35k3d.fpga_verified=1` hay `r35k3d.tee=1` trước khi preflight phần cứng đúng.
4. Sau khi DT/FPGA và runtime TA được xác minh: tiến hành nạp module có kiểm soát, chạy `--probe` trước, rồi `--tee-test`; `--mmio-test` hoặc `--all-test` chỉ khi đồng ý rõ về thao tác ghi/reset. Thu UART log, return code, trạng thái `/dev/fpga_imgproc`, TEE result, hash artifact.
5. Tiếp tục spec gap: `GET_STATUS` backend, HWUID, FPGA IRQ/DMA, V4L2, U-Boot verify kernel/signature. Không đánh dấu 7/7 ioctl trước khi hardware regression thực tế.

## 7. Kết luận

**Ngày 15/10/2026:** tái xác minh được MMIO R34E và Linux/OP-TEE/HW88 trên board; khôi phục và merge kernel driver; đã cross-compile module R35K_3B có đường gọi TEE theo ABI HW76; đã chuẩn bị xong R35K_3D initramfs có xác minh integrity. **Checkpoint cuối: HOST BUILD PASS, BOARD BOOT R35K_3D PENDING, SECURE BOOT SIGNATURE AUTHENTICATION NOT PROVEN.**
