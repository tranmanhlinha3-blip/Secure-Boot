# Secure Boot / FPGA Kernel Driver — Reproduce ngày 14/10/2026

> **Board:** Terasic DE10-Nano (Cyclone V SoC, ARM Cortex-A9 / AArch32)  
> **Môi trường:** WSL Ubuntu, U-Boot SPL/U-Boot 2025.07, TF-A SP_MIN 2.13, OP-TEE 3.21, Linux 5.15.221  
> **Phạm vi phiên kiểm thử:** HW96E-R19 → HW96E-R32A  
> **Checkpoint tiếp theo:** **HW96E-R32C — chuẩn bị initramfs có shell tương tác**  
> **Nguồn:** Log WSL và PuTTY đã trao đổi trong phiên làm việc ngày 14/10/2026. Những điều chưa được log chứng minh được ghi rõ là *PENDING / UNVERIFIED*.

## 1. Executive summary

Ngày 14/10/2026 đã xác minh thành công chuỗi khởi động trên **test SD**:

```text
BootROM → U-Boot SPL → TF-A SP_MIN (R10 diagnostic remap)
        → OP-TEE → U-Boot → Linux 5.15.221-dirty
        → OP-TEE Linux driver → tee-supplicant → xtest 1006 PASS
```

Đồng thời, FPGA được nạp qua U-Boot, LWH2F bridge được giải phóng reset và **đọc/ghi register bank custom Avalon-MM tại `0xFF200000` đã PASS**. Kết quả này là bằng chứng phần cứng thực tế cho đường HPS ↔ FPGA ở U-Boot.

**Chưa hoàn thành:** Linux shell tương tác (HW68 initramfs không có `/bin/sh`), driver `fpga_imgproc.ko` trên kernel R4B (`vermagic` không khớp), DMA/IRQ/V4L2, xác thực chữ ký Secure Boot. `xtest` kiểm tra OP-TEE; **không phải** chức năng của U-Boot.

### Trạng thái cuối ngày

| Thành phần / gate | Trạng thái | Bằng chứng chính |
|---|---|---|
| SPL load + SHA-256 image | **PASS** | ATF/TEE/U-Boot được tải; SHA-256 PASS |
| TF-A SP_MIN R10 và remap `0x11` | **PASS (write issued)** | Marker `C5TF:HW96E:R10:REMAP_0X11_ISSUED` |
| OP-TEE và U-Boot handoff | **PASS** | U-Boot prompt `=>` |
| FPGA RBF, FPGA USERMODE | **PASS** | CRC32 `87d23a06`, FPGA status `0x54`, monitor `0xF07` |
| LWH2F release | **PASS** | `BRGMODRST = 0x5` |
| Avalon-MM reads (8 registers) | **PASS** | R25: 8/8 readable, no Data Abort |
| Avalon-MM threshold write/readback/restore | **PASS** | R26: `0 → 0x1F4 → 0` |
| Linux 5.15.221-dirty | **PASS** | Linux kernel log + `/init` start |
| Linux ↔ OP-TEE driver | **PASS** | `/dev/tee0`, `/dev/teepriv0`, `optee: initialized driver` |
| `xtest 1006` | **PASS** | 2 subtests, 0 failed; exit code 0 |
| Linux shell tương tác | **BLOCKED** | `/bin/sh` và `/bin/busybox` thiếu trong HW68 cpio |
| `fpga_imgproc.ko` | **PENDING** | Module `5.15.221`, kernel R4B `5.15.221-dirty` |
| Signature authentication | **NOT ENABLED** | SPL: `Authentication: NOT ENABLED YET` |
| OP-TEE RNG production-ready | **NOT READY** | `seeding RNG with zeroes` ở log trước |

---

## 2. Chronological checkpoints (R19–R32A)

| Checkpoint | Công việc | Kết quả |
|---|---|---|
| **R19** | Offline ATF header/entry/payload SHA, single-source delta, nhận diện SD | **PASS**, không ghi |
| **R20** | Đọc raw 1 MiB ATF slot từ test SD, so với backup | **PASS**, byte-for-byte |
| **R21** | Kiểm tra hash backup/readback/candidate, device identity, unmounted | **PASS** |
| **R22** | Boot TF-A R10 diagnostic trên board | **PASS** — marker R10 và U-Boot prompt; *log readback hậu ghi R22 không được cung cấp trong phiên này* |
| **R23** | Đọc FPGA RBF/CRC, nạp FPGA, đọc trạng thái | **FPGA PASS**; Data Abort tại lần đọc lặp tới vùng `0xFFD05024` |
| **R24** | Thử lại đúng trình tự; release LWH2F; đọc Avalon base | **PASS**, `0xFF200000 = 0` |
| **R25** | Đọc 8 thanh ghi custom MMIO | **PASS** |
| **R26** | Ghi THRESHOLD=500, readback, restore=0 | **PASS** |
| **R27** | U-Boot Linux env / SD partitions / tools | **AUDIT COMPLETE**; raw SD không có partition table nhận diện |
| **R28** | Host discovery Linux artifacts | **COMPLETE**; tìm thấy R4B, HW68, source/`.ko` |
| **R29** | Audit kernel/DTB/module | **SCRIPT ERROR** (`shutil_which` khai báo sau lời gọi); phần output trước lỗi dùng được |
| **R29B** | Audit R4B zImage, OP-TEE DTB, HW68, module `vermagic` | **HOST PASS**, module mismatch |
| **R30** | Kiểm CRC32 trên SD ↔ host cho 3 raw Linux slots | **PASS**, DTB audit |
| **R31** | `bootz` Linux 5.15 + OP-TEE + tự chạy `xtest 1006` | **PASS** |
| **R32A** | Audit HW68 initramfs shell | **BLOCKED**; `/init` là ELF, không phải shell script; thiếu `/bin/sh` |

### Lưu ý về Data Abort ở R23

Sau `md.l ffd0501c 1`, terminal có dòng `ffd05020: 00000000`, rồi Data Abort với `r8=ffd05024`. U-Boot có thể lặp lệnh `md` khi nhấn Enter trống và tăng địa chỉ. Đây là **nguyên nhân có khả năng cao**, không phải một phép đọc trực tiếp `0xFF200000` thất bại. R24–R26 sau đó chứng minh giao tiếp tại `0xFF200000` chạy thành công. Không đọc vùng reserved Reset Manager và không chạy `bridge enable` từ Normal World.

---

## 3. Artifact, identity, địa chỉ và hash

### 3.1 TF-A / SD

**Test SD đã kiểm tra khi gắn trong WSL:**

```text
Device lúc xác minh : /dev/sde  (có thể thay đổi giữa các lần kết nối)
Type                : disk
Size                : 4007657472 bytes
Serial              : 121220160204
Logical sector size : 512 bytes
ATF slot            : LBA 0x1000, 0x800 sectors = 1 MiB
TEE slot start      : LBA 0x1800
```

> **An toàn:** Device path `/dev/sde` KHÔNG phải danh tính cố định. Trước bất kỳ thao tác ghi hay rollback nào phải xác minh lại serial/size và xác nhận golden SD không phải thiết bị được chọn. Không ghi golden SD.

| File | Path | SHA-256 |
|---|---|---|
| ATF backup 1 MiB | `~/secure_imgproc/boot_phaseA/HW95G_DEPLOY_4b8AbVpz/atf_before.bin` | `fdc9b895f48a025860c740132fd13b4aa70e97061eabba52a9c5801320a2063c` |
| TF-A R10 slot candidate 1 MiB | `~/secure_imgproc/boot_phaseA/HW96E_R18_OFFLINE_PACKAGE_CZEFZrp7/r10_atf_test_slot.img` | `6093f5f8a9e26e66d93d3443909edd2e5beb117bc430b3d0859c920c3336689a` |
| TF-A R10 payload (16,412 B) | `~/secure_imgproc/boot_phaseA/HW96E_R10_TFA_DIAG_tAqYFZ17/source/build/cyclone5/release/bl32.bin` | `e99e1a4a4b2db4c237c91349601fba3918f764a96ce3ca29b6b84674abf5c7ce` |

- Active pre-R22 ATF slot matched backup exactly in R20 (`CURRENT_MATCH_BACKUP=True`).
- Active baseline header entry cũ: `0x3E0033E0`; TF-A R10 candidate entry mới: `0x3E003480`, nhảy về `0x3E000000`.
- R19 source delta: R10 vs R11 chỉ khác `plat/altera/soc/cyclone5/sp_min/cyclone5_sp_min_setup.c`; marker remap present.
- Marker `REMAP_0X11_ISSUED` chứng minh đường ghi từ TF-A đã được thực thi, **không phải** phép readback NIC-301 GPV. Không đọc GPV register ở Normal World.
- **Source provenance còn mở:** source HW94I của TF-A R10 chưa tái lập bit-for-bit firmware active cũ. Lần boot hardware PASS không đóng hoàn toàn bài toán provenance.

### 3.2 FPGA

```text
RBF SD LBA                  : 0x772800
MMC read sector count       : 0xF67 (3943)
RBF payload length          : 0x001ECD4C (2,018,636 bytes)
RBF expected CRC32          : 87d23a06
FPGA manager status         : 0xFF706000 = 0x00000054
FPGA monitor                : 0xFF706850 = 0x00000F07
Bridge reset before release : 0xFFD0501C = 0x00000007
Bridge reset after release  : 0xFFD0501C = 0x00000005
Custom IP base              : 0xFF200000
```

### 3.3 Linux 5.15 / OP-TEE R4B & HW68

```text
R4B root:
~/secure_imgproc/optee_phase/GATE_C_R4B_linux515_optee_candidate/20261005_042159

Linux source tree:
~/secure_imgproc/linux_phase4/linux-5.15.y

zImage:
<R4B root>/build/arch/arm/boot/zImage

OP-TEE DTB:
<R4B root>/package/socfpga-de10-optee.dtb

HW68 xtest initramfs:
~/secure_imgproc/linux_phase4/HW68_XTEST_1006/xtest_1006_initramfs.cpio

Kernel release: 5.15.221-dirty
DTB compatible: tecotec,fpga-imgproc
DTB MMIO node: /soc/fpga-imgproc@ff200000, reg=<0xff200000 0x1000>
DTB memory: reg=<0x0 0x3e000000> (992 MiB)
```

| Artifact | Exact size | SHA-256 | CRC32 | Raw SD LBA / sectors | Load DDR |
|---|---:|---|---|---|---|
| R4B `zImage` | 5,796,264 B | `721e815683af99cf56b219a3b040e9302690db306a3c1ea6a23e94ba9d72b9ff` | `9ec4f162` | `0x3000 / 0x2C39` | `0x01000000` |
| OP-TEE DTB | 19,630 B | `e9aac78824d83705188f99dd3791d968545cbfebcb00038db241c98c9597d537` | `f153f734` | `0x6000 / 0x27` | `0x03000000` |
| HW68 initramfs | 9,928,660 B | `86e45da98af51517bc844651da846bf32ac12c21ada627fcc181932a2e22784c` | `7bf9fc64` | `0xD000 / 0x4BC0` | `0x04000000` |

**Kernel driver artifacts (chưa load trong R31):**

```text
/mnt/d/internFPGA/HPS_FPGA/phase4_kernel_driver/src/kernel/fpga_imgproc.c
/mnt/d/internFPGA/HPS_FPGA/phase4_kernel_driver/src/kernel/fpga_imgproc.ko
/mnt/d/internFPGA/HPS_FPGA/phase4_kernel_driver/staging/K4C_linux515_basic/fpga_imgproc.ko
```

Cả hai `.ko` đã tìm thấy có `vermagic=5.15.221 SMP mod_unload ARMv7 p2v8`, **khác** `5.15.221-dirty` của kernel R4B. Không ép nạp module; build lại theo đúng source/config R4B và kiểm tra driver source trước khi chạy.

---

## 4. Cách reproduce những bước đã PASS

> **Chỉ áp dụng với test SD có đúng các raw slots đã được xác minh ở R19–R30.** Không áp dụng cho golden SD. Các lệnh dưới đây chạy tại U-Boot `=>` trừ khi ghi khác. Không có lệnh ghi lên SD.

### 4.1 Boot firmware + verify marker

Power-cycle hoặc khởi động test board với test SD; xác nhận console có:

```text
SPL: ALL IMAGES LOAD + SHA256 PASS
SPL: Handoff -> TF-A SP_MIN entry=0x3e003480
C5TF:HW96E:R10:REMAP_0X11_ISSUED
C5TF:HW24:GPV_POLICY_WRITES_ISSUED
...
U-Boot 2025.07
=>
```

SPL cũng in `Authentication: NOT ENABLED YET`: **SHA-256 integrity != authenticated secure boot**.

### 4.2 FPGA load, LWH2F read/write

**Bước A: Đọc và kiểm CRC32 RBF**

```text
mmc dev 0
mmc read 0x08000000 0x772800 0xF67
crc32 0x08000000 0x001ECD4C
```

Expected CRC32: **`87d23a06`**. Nếu khác, **DỪNG**.

**Bước B: Nạp FPGA và quan sát trạng thái**

```text
fpga load 0 0x08000000 0x001ECD4C
md.l ff706000 1
echo FPGA_STATUS_DONE
md.l ff706850 1
echo FPGA_MONITOR_DONE
md.l ffd0501c 1
echo BRIDGE_RESET_BEFORE_DONE
```

Expected lần lượt: **`0x54`, `0xF07`, `0x7`**. Không nhấn Enter trống sau `md.l` vì U-Boot có thể lặp đọc và tự tăng địa chỉ. Không đọc `0xFFD05024`.

**Bước C: Release riêng LWH2F và kiểm tra 8 thanh ghi**

> **Có thay đổi register runtime (không ghi SD).** Chỉ thực hiện trên test board đã có R10, FPGA USERMODE và đúng bitstream; nếu điều kiện khác dừng.

```text
mw.l ffd0501c 00000005 1
md.l ffd0501c 1
echo BRIDGE_RELEASE_DONE
md.l ff200000 8
echo AVALON_REG_READ_DONE
```

Expected `BRGMODRST=0x5`, 8 registers read without Data Abort:

| Offset | Register | R25 initial read |
|---|---|---|
| `0x00` | CTRL | `0x00000000` |
| `0x04` | STATUS | `0x00000001` |
| `0x08` | ADC_RAW | `0x00000123` |
| `0x0C` | FIR_OUT | `0x00000000` |
| `0x10` | FFT_BIN0 | `0x00000000` |
| `0x14` | FFT_BIN1 | `0x00000000` |
| `0x18` | THRESHOLD | `0x00000000` |
| `0x1C` | LED | `0x00000000` |

**Bước D: THRESHOLD write/readback/restore**

> Chỉ làm nếu THRESHOLD ban đầu là `0x00000000`; không làm nếu thiết kế IP hoặc trạng thái ban đầu khác.

```text
md.l ff200018 1
echo THRESHOLD_BEFORE
mw.l ff200018 000001f4 1
md.l ff200018 1
echo THRESHOLD_WRITTEN
mw.l ff200018 00000000 1
md.l ff200018 1
echo THRESHOLD_RESTORED
```

Expected: **`0x0 → 0x1F4 → 0x0`**, vẫn có U-Boot prompt, bridge reset `0x5`.

### 4.3 Linux R4B/OP-TEE/HW68 boot và xtest

**Bước A: Đọc raw Linux slots từ test SD, kiểm CRC trước khi boot**

```text
mmc dev 0
mmc read 0x01000000 0x3000 0x2C39
crc32 0x01000000 0x5871A8
mmc read 0x03000000 0x6000 0x27
crc32 0x03000000 0x4CAE
mmc read 0x04000000 0xD000 0x4BC0
crc32 0x04000000 0x977FD4
```

CRC expected theo thứ tự: **`9ec4f162`**, **`f153f734`**, **`7bf9fc64`**. Nếu có bất kỳ mismatch nào, **không `bootz`**.

**Bước B: Thay đổi DTB trong RAM (không sửa file SD)**

```text
fdt addr 0x03000000
fdt resize 0x1000
fdt set /soc/timer1@ffc09000 status disabled
fdt print /soc/timer1@ffc09000 status
fdt print /memory
fdt print /firmware
```

Expected `timer1 status="disabled"`; `memory reg=<0x0 0x3e000000>`; `/firmware/optee` có `compatible="linaro,optee-tz"` và `method="smc"`.

**Bước C: Boot Linux**

```text
setenv bootargs 'console=ttyS0,115200n8 earlycon=uart8250,mmio32,0xffc02000 keep_bootcon rdinit=/init maxcpus=1 mem=992M ignore_loglevel loglevel=7'
bootz 0x01000000 0x04000000:0x977FD4 0x03000000
```

Expected Linux `5.15.221-dirty`, OP-TEE driver, HW68 init:

```text
Linux version 5.15.221-dirty
...
optee: revision 3.21 (e8abbcfb)
optee: initialized driver
...
/dev/tee0 = FOUND
/dev/teepriv0 = FOUND
HW68: START TEE-SUPPLICANT
HW68: START XTEST 1006
...
regression_1006 OK
2 subtests of which 0 failed
1 test case of which 0 failed
103 test cases were skipped
HW68_XTEST_EXIT_CODE=0
HW68_XTEST_SMOKE=PASS
```

**Chú ý:** HW68 `/init` tự chạy test và không tạo shell tương tác. Sau `HW68: DIAGNOSTIC FINISHED`, các lệnh `uname -a`, `help` gõ vào UART không được xử lý như shell; đây không phải bằng chứng Linux crash.

---

## 5. Root-cause / lỗi đã gặp và phân loại

### 5.1 Avalon-MM Data Abort ban đầu

- Trước R10: lệnh đọc LWH2F base từng gây Data Abort dù FPGA đã USERMODE.
- Giả thuyết chính: thiếu LWH2F remap NIC-301 bit `0x10` trong khởi tạo firmware.
- TF-A R10 ghi `0x11` bằng Secure World; R24–R26 kiểm thử đọc và ghi FPGA thành công.
- **Kết luận có giới hạn:** R10 hoạt động với chuỗi này. Chưa có A/B test bit-for-bit cô lập nên không khẳng định remap là yếu tố duy nhất.

### 5.2 Data Abort R23 tại Reset Manager

- Log xuất hiện đọc `0xFFD05020`, sau đó Data Abort khi U-Boot có `r8=0xFFD05024`.
- Phù hợp hành vi `md.l` có thể lặp khi nhấn Enter trống.
- Tránh `md.l` tiếp vào vùng reserved và tách lệnh với `echo`.
- R24 ngay sau đó không tái hiện abort khi dùng lệnh rõ ràng.

### 5.3 R29 Python exception

```text
NameError: name 'shutil_which' is not defined
```

- Hàm trợ giúp được khai báo sau lần gọi trong script; đây là lỗi kiểm tra trên host.
- R29B đã thay bằng `shutil.which` và audit tiếp thành công.

### 5.4 R32A không có Linux shell

```text
init=FOUND MODE=0o100755 SIZE=361560
bin/sh=MISSING
bin/busybox=MISSING
usr/bin/xtest=FOUND
usr/sbin/tee-supplicant=FOUND
HW96E_R32A_SHELL_PATH=BLOCKED
```

- `/init` là **ELF binary** (361,560 B), không phải shell script. In binary như text trong R32A gây log ký tự rác; không phải lỗi firmware.
- Không boot HW68 với `rdinit=/bin/sh`: file `/bin/sh` không tồn tại.
- Chuẩn bị bản sao initramfs có BusyBox ARM32 static và shell; **giữ nguyên HW68 nguyên bản**.

---

## 6. Open issues / checklist theo hướng Kernel Driver

| Mục tiêu | Hiện trạng | Việc tiếp theo |
|---|---|---|
| Linux interactive shell | BLOCKED | **R32C:** tạo initramfs mới, shell ARM32 static, kiểm tra `/init`/PID1 |
| Linux FPGA MMIO | Chưa test trong Linux | Sau shell: kiểm clock/reset/driver trước khi đọc MMIO |
| `fpga_imgproc.ko` | Có source + 2 modules, vermagic không khớp | Audit source, rebuild cho `5.15.221-dirty` |
| Device Tree binding | Có node `tecotec,fpga-imgproc` | Kiểm tra `reg`, `compatible`, driver match, IRQ/DMA thật |
| IRQ + DMA | Chưa chứng minh | Chỉ triển khai sau khi RTL/IP thực sự có IRQ/DMA và driver base ổn định |
| V4L2 / image processing | Chưa chứng minh | Thử sau register driver + data path |
| OP-TEE client / trust boundary | `xtest 1006` PASS, chưa chứng minh app xử lý ảnh | Kiểm CA/TA tương ứng theo spec |
| Secure Boot authentication | NOT ENABLED | Xác thực chữ ký + kiểm failure cases; SHA256 chỉ integrity |
| OP-TEE RNG | Warning seeded zero | Khắc phục trước đánh giá security production |
| TF-A source provenance | Unresolved | Reconcile active-source history và artifacts |

### Khuyến nghị cho R32C

1. Giữ R4B kernel/DTB và HW68 initramfs nguyên bản làm **known-good baseline**.
2. Tìm hoặc build `busybox` ARM32 static, xác minh `readelf -h` và khả năng chạy từ initramfs không phụ thuộc dynamic loader.
3. Tạo *artifact mới* ở thư mục run R32C chứa `bin/busybox`, `bin/sh` và `/init` debug rõ ràng (hoặc giữ `/init` và thêm shell startup có kiểm soát); kiểm tra file permission, symlink và archive CPIO newc.
4. Thử tải qua RAM / serial trước khi cân nhắc thay raw initramfs slot trên SD. Đảm bảo không đè vùng firmware secure hoặc payload đang chạy.
5. Khi shell PASS, đánh giá `BRGMODRST`, FPGA USERMODE và register read path trong Linux, rồi build driver đúng kernel R4B.

---

## 7. Quy tắc an toàn và cách dừng / tiếp tục

- **Golden SD: tuyệt đối không ghi.** Test SD cần xác minh lại bằng serial/size trước bất kỳ ghi nào; đường `/dev/sde` có thể đổi.
- Các lệnh trong mục reproduce trên U-Boot thay đổi **RAM hoặc thanh ghi runtime**, không ghi SD. Tránh `saveenv`, `bridge enable`, đọc GPV từ Normal World và đọc Reset Manager reserved.
- Nếu CRC RBF/zImage/DTB/initramfs lệch: dừng, không `fpga load` hoặc `bootz` tiếp.
- Sau khi Linux boot, `HW68: DIAGNOSTIC FINISHED` không phải dấu nhắc shell.
- Có thể kết thúc phiên WSL mà không mất source/artifact trên đĩa:

```bash
sync
exit
```

  Và, nếu muốn tắt hẳn WSL từ PowerShell Windows:

```powershell
wsl --shutdown
```

  Việc này không tự tắt nguồn DE10-Nano. Trạng thái FPGA/U-Boot trong RAM trên board sẽ mất nếu board reset/mất điện.

## 8. Checkpoint để resume

```text
DATE                  = 2026-10-14
PROJECT               = Secure Boot / FPGA Kernel Driver
BOARD                 = DE10-Nano, Cyclone V SoC
LATEST_COMPLETED      = HW96E-R31: LINUX+OPTEE XTEST_1006 PASS
LATEST_AUDIT          = HW96E-R32A: SHELL_PATH BLOCKED
CURRENT_FIRMWARE      = TF-A R10, NIC-301 remap 0x11
LINUX_BASELINE        = R4B 5.15.221-dirty
OPTEE_INITRAMFS       = HW68_XTEST_1006 (NO /bin/sh)
CUSTOM_FPGA_MMIO      = U-Boot 8/8 read, THRESHOLD R/W/restore PASS
NEXT_CHECKPOINT       = HW96E-R32C
NEXT_ACTION           = Create isolated ARM32 shell-enabled diagnostic initramfs
KEEP_EXISTING_SD      = YES; no additional SD writes without device identity/rollback checks
GOLDEN_SD_WRITE       = NEVER
```

**Tóm lại:** Chuỗi boot Linux + OP-TEE và đường FPGA Avalon-MM đã có bài kiểm thử thực tế thành công trong ngày 14/10/2026. Công việc tiếp theo là mở Linux shell trên một initramfs mới để kiểm thử driver FPGA, không lặp lại các checkpoint đã PASS nếu không có thay đổi liên quan.
