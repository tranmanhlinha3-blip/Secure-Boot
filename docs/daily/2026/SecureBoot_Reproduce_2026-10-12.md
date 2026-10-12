# Secure Boot / DE10-Nano — Daily Reproduce & Handoff

**Ngày:** 12/10/2026 (giờ Việt Nam)  
**Platform:** Terasic DE10-Nano / Altera Cyclone V SoCFPGA / Cortex-A9 ARMv7 AArch32  
**Boot chain thử nghiệm:** U-Boot SPL 2025.07 → TF-A 2.13 SP_MIN → OP-TEE 3.21 → U-Boot → Linux 5.15.221-dirty → initramfs → tee-supplicant / TA.  
**Mục tiêu của bản ghi:** ghi lại tiến trình đã kiểm chứng, cách reproduce không phá checkpoint và bước tiếp theo. Những lệnh và kết quả đánh dấu **DỰ KIẾN** không được tính là đã chạy trên hardware.

## A. Executive outcome

- **HW77 — KNOWN GOOD, PASS trên board:** Linux `/init`, OP-TEE device, `xtest 1006`, ImgProc `SIGN`, `VERIFY_GOOD`, `VERIFY_BAD_REJECT`, `WATERMARK`; `HWUID` từ chối an toàn.
- **HW86 — PASS host simulation:** 9 nhóm unit test SMC32 custom handler, không phát SMC trên board.
- **HW87 — PASS host build + QA:** `c5_sip_readonly.ko` ELF32 ARM, `5.15.221-dirty SMP mod_unload ARMv7 p2v8`, khớp kernel source/Module.symvers.
- **HW89 — PASS host & read-only SD:** HW88 initramfs tồn tại, CPIO 56 entries, SHA-256 đúng và readback trực tiếp từ slot SD khớp.
- **HW88A/HW88B — PENDING:** lần boot thử HW88 đầu đã nạp **0 sector** vì dùng placeholder `<COUNT>` và initrd size `<SIZE>`; Linux panic do không có rootfs. Đã phục hồi HW77 PASS. Chưa có log module init HW88 thực trên board.
- **Không tuyên bố secure boot production:** SPL chỉ SHA256 integrity, vẫn `Authentication: NOT ENABLED YET`; nguồn entropy OP-TEE còn cảnh báo zero seed; HWUID thật/HUK/firewall/persistence chưa đạt.

## B. Diễn tiến hôm nay (theo chuỗi checkpoint)

| Checkpoint | Evidence quan trọng | Kết luận |
|---|---|---|
| HW58 | Linux 5.15.221-dirty vào `/init`, `/dev/tee0`, `/dev/teepriv0`, `HW58_INIT_REACHED=PASS` | Board boot/TEE device PASS |
| HW65A/B | Xác định xtest/TA deployment; `xtest 1001` exit 0 nhưng **0 subtests** và TA liên quan bị skip | Chỉ PASS smoke command, **không** phải full xtest PASS |
| HW68 | `xtest 1006`: 2 subtests, 0 failed; `HW68_XTEST_SMOKE=PASS` | OP-TEE TA execution PASS |
| HW73 | `SIGN`, `VERIFY_GOOD`, `VERIFY_BAD_REJECT` PASS; watermark `BITCOUNT_GATE=FAIL`, CA exit 1 | C3 chưa PASS ở HW73 |
| HW76 | Sửa TA/CA giao diện watermark; `HW76_TA_PATCH=PASS`, `HW76_CA_PATCH=PASS`, TA + CA build PASS | Sửa lỗi host PASS |
| HW77 | `xtest 1006` PASS; `IMGPROC_C2C_CA_RESULT=PASS`; watermark 128-bit / 256 B / 43 changed bytes / max delta 1; HWUID fail-closed | **Checkpoint Gate C3 known-good trên board** |
| HW81 | PSNR watermark 55.8785 dB, MSE 0.16796875; security audit chỉ ra entropy/HUK/HWUID/secure storage còn thiếu | Functional PASS ≠ production-security PASS |
| HW83 | Quét firmware và đọc SD chế độ read-only; `HW83A_PROVENANCE_AUDIT=PASS` | Xác nhận SD artifact, không thực hiện ghi |
| HW84 | Snapshot exact ATF SD `16412 B`; tìm 56 file cùng kích thước nhưng 0 exact historical match; active ATF có marker `c5_project_sip_svc`; OP-TEE artifact match | Binary lineage ATF còn chưa truy vết trọn vẹn |
| HW85 | Source `cyclone5_project_sip.c`/`.h`; 4 SiP IDs; service được tham chiếu trong Cyclone5 `sp_min-cyclone5.mk` | C4 handler source hiện hữu |
| HW86 | `HW86_HOST_UNIT_TEST=PASS`: ABI IDs, HWUID/LOCK fail-closed, boot status, PMCCNTR mock, invalid/unknown calls | Host-only; hardware chưa test |
| HW87 | `make modules` từ **kernel build R4B đúng SHA** → `Module.symvers`; build `c5_sip_readonly.ko`; `HW87_HOST=PASS`, `HW87_MODULE_QA=PASS` | Module host build PASS |
| HW88 | Build script đầu chặn `vermagic mismatch` do dấu cách cuối; fixed script. Lần boot thử dùng placeholder → `count 0`, ramdisk 0B → kernel panic | Root cause: lệnh U-Boot, **không** chứng minh lỗi module |
| HW77 recovery | Reboot slot cũ `0x18000`, `0x4cce` sectors và size `0x999a64`; `xtest 1006` + ImgProc CA PASS | **Recovery hardware verified** |
| HW89 host | 5 firmware/artifact hash PASS, CPIO PASS 56 entries, manifest size/count/hash | **PASS** |
| HW89 SD | `/dev/sde` đúng thời điểm chạy; kernel/ATF/TEE/HW77 ID PASS, `HW89_SD_HW88_IMAGE=PASS_READBACK_MATCH`; không ghi SD | **HW88 image sẵn sàng để boot thử** |
| HW90 | Runbook HW88A execute=0 / HW88B execute=1 / HW77 rollback được chuẩn bị | Tài liệu hướng dẫn; chưa phải hardware PASS |

## C. Chuẩn hash hiện tại

| Artifact | SHA-256 |
|---|---|
| Linux zImage R4B / kernel SD | `721e815683af99cf56b219a3b040e9302690db306a3c1ea6a23e94ba9d72b9ff` |
| Active TF-A SD, 16412 B | `da5d759792cd86dc078a741589e8cd05b5809b81ddc64002e362b63973d4dfa2` |
| Active OP-TEE SD, 321608 B | `2987d05a8920abc87e6afbc9fc0e25459a70be69d84f272e2c3e24bc5dcc8db1` |
| HW77 initramfs | `16a80c86cd4793efba6a24b5381c16d5934fe137541aa806427fa3fc31867c8d` |
| HW87 `.ko` | `7943378ee45fef114537f10a6d3cd72ee01fb6fdf3ae58f800ec0c430679b27a` |
| HW88 initramfs | `735c25fc049b6b20742f24934908135a21d48e6d2f6fae4452e9d148734495cc` |
| Cyclone5 project SiP `.h` | `f3552c8b2ed14cc031df24813b94e80e48b3b5da1a639cdbc700fac1d103948b` |
| Cyclone5 project SiP `.c` | `11d326c4909939ffde2299073412baa9badaef51610aa647a9e03c51a9d420a3` |

**Các đường dẫn (WSL):**

```text
ROOT=/home/manhlinh/secure_imgproc
HW77=$ROOT/linux_phase4/HW77_IMGPROC_WATERMARK_FIXED/hw77_initramfs.cpio
HW87=$ROOT/optee_phase/HW87_C4_SMC_READONLY_PROBE/c5_sip_readonly.ko
HW88=$ROOT/linux_phase4/HW88_C4_SIP_PROBE/hw88_sip_probe_initramfs.cpio
KERNEL=$ROOT/optee_phase/GATE_C_R4B_linux515_optee_candidate/20261005_042159/build/arch/arm/boot/zImage
ATF_SNAPSHOT=$ROOT/optee_phase/HW84_C4_ACTIVE_ATF/active_sd_atf_16412.bin
OPTEE_BIN=$ROOT/optee_phase/GATE_C_C1D_R1_quiet_optee/20261005_092333/build/core/tee-pager_v2.bin
HW89_REPORT=/mnt/d/SecureBoot_share/HW89_RESUME_20261012_082233
WINDOWS_BACKUP=/mnt/d/SecureBoot_share/HW88_SAVE_20261012_Ii42W1
```

Thư mục backup Windows **đã được phát hiện** bởi HW89, nhưng log được cung cấp chưa kiểm tra đầy đủ nội dung/manifest trong folder backup. `HW89` xác minh các file gốc WSL và image trên SD.

## D. C4 SiP ABI — functional scope thật sự

| SMC32 FID | Chức năng | Triển khai đã đọc từ source | Tình trạng |
|---|---|---|---|
| `0x82000001` | GET_HWUID | `r0=1` NOT_SUPPORTED, UID=0 | Fail-closed; không có UID tin cậy |
| `0x82000002` | LOCK_REGION | `r0=1` NOT_SUPPORTED | Fail-closed; chưa chứng minh firewall |
| `0x82000003` | GET_BOOT_STATUS | status, boot flags/version, magic `0x43354253`, ABI `0x00020000` | Host unit PASS; hardware pending |
| `0x82000004` | PERF_COUNTER | snapshot ARMv7 PMCCNTR; invalid argument → NOT_SUPPORTED | Host unit PASS; hardware pending |

**Cảnh báo:** `c5_project_sip_svc` nằm trong binary đang chạy theo `strings`, nhưng chưa chứng minh exact source-to-binary match hoặc SMC runtime behavior. SiP IDs ở đây là SMC32 phù hợp AArch32; spec gốc dùng SMC64 `0xC2000001..04` cần đối chiếu tài liệu ABI. Giá trị `C5_SIP_STATUS_NOT_SUPPORTED` của source này là `1` (khác quy ước `0xffffffff` ở một số tài liệu khác).

## E. Reproduce trên WSL — READ ONLY / KHÔNG BUILD LẠI NẾU KHÔNG CẦN

### E1. Xác minh checkpoint HW89 trên host

Các script đã được lưu vào `D:\SecureBoot_share` theo workflow hiện tại:

```bash
bash /mnt/d/SecureBoot_share/HW89_RESUME_HW88_HOST.sh
```

Yêu cầu: `HW89_HOST=PASS`, `HW89_HW88_CPIO=PASS`, SHA-256 image HW88 đúng và CPIO đủ init/TA/CA/`.ko`. Script tự sinh thư mục mới `HW89_RESUME_YYYYMMDD_HHMMSS` trên Windows.

### E2. Xác minh nội dung SD đã stage (không ghi)

**Chỉ khi board tắt và SD đang nằm trong USB reader.** Trên một phiên WSL mới phải nhận diện lại block device, không mặc định `/dev/sde`:

```bash
lsblk -p -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
# Thay /dev/sdX bằng thiết bị được xác minh đúng bằng model/size/identity.
AUDIT_DIR="$(ls -dt /mnt/d/SecureBoot_share/HW89_RESUME_* | head -n1)"
DEV=/dev/sdX AUDIT_DIR="$AUDIT_DIR" \
  bash /mnt/d/SecureBoot_share/HW89_SD_READONLY_VERIFY.sh
```

Chỉ tiếp tục khi `HW89_SD_HW88_IMAGE=PASS_READBACK_MATCH`. Script SD ở chế độ **READ_ONLY**; không chạy `write`/`dd`/`saveenv`.

### E3. Nếu thật sự cần reproduce HW86 và HW87 build

```bash
bash /mnt/d/SecureBoot_share/HW86_C4_SIP_HOST_TEST.sh

ROOT="$HOME/secure_imgproc"
BASE="$ROOT/optee_phase/GATE_C_R4B_linux515_optee_candidate/20261005_042159"
# Chỉ nếu Module.symvers thiếu: dùng đúng source & build R4B đã xác minh.
make -C "$BASE/source" O="$BASE/build" ARCH=arm \
  CROSS_COMPILE=arm-linux-gnueabihf- -j4 modules
sha256sum "$BASE/build/arch/arm/boot/zImage"

bash /mnt/d/SecureBoot_share/HW87_C4_SMC_READONLY_PROBE.sh
```

**Không chạy lại script build HW87/HW88 một cách máy móc nếu output directory đã tồn tại**: script có thể chặn overwrite. Checkpoint HW87/HW88 hiện tại đã PASS host; dùng HW89 audit trước.

## F. Bản đồ U-Boot và đúng ba lệnh boot quan trọng

| Artifact | SD start sector | Read count (512-byte sectors) | `bootz` initrd size |
|---|---:|---:|---:|
| zImage | `0x3000` | `0x2c39` | — |
| DTB | `0x6000` | `0x27` | — |
| HW77 known-good initramfs | `0x18000` | `0x4cce` | `0x999a64` |
| **HW88 proposed initramfs (readback PASS)** | **`0x20000`** | **`0x4d9f` = 19871** | **`0x9b3d34` = 10173748 B** |

**Thất bại đã reproduce hôm nay:** `setenv hw88_count <COUNT>` và `setenv hw88_size <SIZE>` gây `mmc read count 0`, ramdisk 0 byte, kernel panic `unknown-block(0,0)`. Không dùng placeholder. Khắc phục: dùng literal count/size ở bảng trên.

### F1. HW88A — chỉ chạy hardware sau preflight readback PASS

```text
mmc dev 0
mmc read 0x01000000 0x3000 0x2c39
mmc read 0x03000000 0x6000 0x27
mmc read 0x04000000 0x20000 0x4d9f
fdt addr 0x03000000
fdt resize 0x1000
fdt set /soc/timer1@ffc09000 status disabled
setenv bootargs 'console=ttyS0,115200 earlycon=uart8250,mmio32,0xffc02000 rdinit=/init loglevel=7 ignore_loglevel maxcpus=1 mem=992M hw88.execute=0'
bootz 0x01000000 0x04000000:0x9b3d34 0x03000000
```

**DỰ KIẾN:** `HW77_XTEST_SMOKE=PASS`, `IMGPROC_C2C_CA_RESULT=PASS`, `HW88_EXECUTE=0`, `HW87_PROBE_SKIPPED=PASS`, `HW88_FINIT_MODULE=PASS`. Nếu thiếu marker/treo/PANIC → dừng, lưu PuTTY log, không thực hiện HW88B.

### F2. HW88B — chỉ sau khi HW88A PASS

Reset về U-Boot, **nạp lại toàn bộ image** như F1; đổi duy nhất `hw88.execute=1` rồi `bootz` với `0x9b3d34`. Có nguy cơ treo máy dù SMC là lệnh đọc, vì firmware/dispatch chưa được xác minh trên hardware. Marker kỳ vọng `HW87_BOOT_STATUS_GATE=PASS`, `HW87_PERF_STATUS_GATE=PASS`, `HW87_PERF_INVALID_GATE=PASS`, `HW87_READONLY_SMC_PROBE=PASS`. Không gọi LOCK_REGION.

### F3. HW77 rollback — đã được xác minh thực hôm nay

```text
mmc dev 0
mmc read 0x01000000 0x3000 0x2c39
mmc read 0x03000000 0x6000 0x27
mmc read 0x04000000 0x18000 0x4cce
fdt addr 0x03000000
fdt resize 0x1000
fdt set /soc/timer1@ffc09000 status disabled
setenv bootargs 'console=ttyS0,115200 earlycon=uart8250,mmio32,0xffc02000 rdinit=/init loglevel=7 ignore_loglevel maxcpus=1 mem=992M'
bootz 0x01000000 0x04000000:0x999a64 0x03000000
```

**Đã quan sát:** Linux unpack initramfs, chạy `/init`, `/dev/tee0`, `xtest 1006` PASS, `IMGPROC_C2C_CA_RESULT=PASS`, `HW77: DIAGNOSTIC FINISHED`.

## G. Security limitations & open gates

| Gate | Trạng thái đến cuối ngày 12/10 | Điều kiện để coi PASS |
|---|---|---|
| C1 Boot/TEE functional | PASS prototype | Giữ regression |
| C2 TA load + xtest | PASS (1006, 2 subtests) | Khai triển test bổ sung nếu spec yêu cầu |
| C3 SIGN/VERIFY/WATERMARK | PASS functional **test key**; PSNR 55.8785 dB | Production crypto/keys chưa PASS |
| C4 SMC host + module build | PASS | Hardware HW88A/B **PENDING** |
| C4 HWUID/LOCK/HUK/entropy | NOT_SUPPORTED / PENDING | Có backend phần cứng đáng tin cậy, RNG an toàn |
| C5 OP-TEE persistent storage | PENDING | Tạo/mở object sau reboot trên backing filesystem bền vững |
| C6 full-spec acceptance | PENDING | ABI alignment, end-to-end/kernel driver/perf/regression |

Nguồn cảnh báo hiện có: SPL `Authentication: NOT ENABLED YET`; OP-TEE `plat_rng_init: seeding RNG with zeroes`, `plat_get_aslr_seed: no ASLR seed`. Không báo `Secure Boot PASS` khi mới có load+SHA256.

## H. Windows/WSL handoff

- WSL data nằm trong distribution và không bị xóa bởi `exit` hoặc `wsl --shutdown` (không chạy `wsl --unregister`).
- Có thấy thư mục `/mnt/d/SecureBoot_share/HW88_SAVE_20261012_Ii42W1` và audit `/mnt/d/SecureBoot_share/HW89_RESUME_20261012_082233`.
- Runbook: `/mnt/d/SecureBoot_share/HW90_C4_BOARD_RUNBOOK.md` nếu đã chép từ file tải về; file runbook trong cuộc trò chuyện không tự động xuất hiện trên Windows.
- Nên lưu full PuTTY UART log cho HW88A và HW88B riêng, cùng HW89 `hw89_host_audit.log`, `hw89_manifest.json`, và log SD readback.
- Để thoát an toàn sau khi không còn thao tác với SD: `sync; exit`, rồi Windows PowerShell `wsl --shutdown` nếu cần. Trước khi tháo SD khỏi reader, đảm bảo WSL không còn dùng block device.

## I. NEXT (không chạy vượt trước)

1. Nạp lại thẻ đã verify vào DE10-Nano tắt nguồn; PuTTY 115200, UART log ON.
2. Chạy **HW88A (execute=0)** bằng lệnh literal ở F1, gửi log. Chỉ PASS khi marker module + HW77 regression đều đạt.
3. Chỉ sau đó chạy **HW88B (execute=1)** và kiểm tra SMC read-only thực tế; nếu fail quay lại F3.
4. Tách nhánh nghiên cứu Gate C5 REE-FS persistence, không dùng RAM-only initramfs để tuyên bố persistence, chưa triển khai production secret/HUK.

---

**Nguồn:** các log trong cuộc trò chuyện ngày 12/10/2026 (HW58, HW65B, HW68, HW73, HW76, HW77, HW81, HW83–HW89), script host HW86/HW87/HW88/HW89 và runbook HW90. Đây là tổng hợp evidence, không phải kết quả chạy mới trên thiết bị.
