# Báo cáo reproduce / reproducible build — C3B1 TF-A SP_MIN

**Ngày:** 08/10/2026 (giờ Việt Nam)  
**Nền tảng:** DE10-Nano / Cyclone V SoC, TF-A AArch32 SP_MIN, WSL Ubuntu  
**Phạm vi:** HW19B–HW19K và chuyển tiếp HW20A, trọng tâm là quá trình xử lý lỗi **baseline reproducibility** từ HW19H đến HW19H7.  
**Cơ sở báo cáo:** các log terminal người dùng cung cấp trong cuộc trò chuyện ngày 08/10/2026. Các lệnh dưới đây là cách **tái hiện** build đã PASS, không phải xác nhận rằng đã thực thi lại khi tạo báo cáo.  
**Trạng thái cuối ngày:** **HOST VALIDATION PASS — RELEASE FREEZE PASS — CHƯA GHI SD — CHƯA BOOT DIAGNOSTIC TRÊN BOARD**.

> **Xem bảng đúng định dạng trong VS Code:** mở Markdown Preview bằng `Ctrl + Shift + V` (hoặc `Ctrl + K`, rồi `V` để xem song song). Bản này cũng căn đều các cột trong mã nguồn `.md`.

---

## 1. Mục tiêu và kết quả cuối cùng

Mục tiêu là tạo một bản TF-A diagnostic chỉ bổ sung phép đọc/ghi log trạng thái bit quyền Non-secure của FPGA Manager trong thanh ghi bảo mật L4, **không thay đổi security mask** và không thăm dò trực tiếp FPGA Manager MMIO từ U-Boot/Linux. Sau đó kiểm tra package ATF, vùng nhớ, và rollback bằng mô phỏng trên host, trước khi xin phép ghi lên **test SD**.

**Kết quả chính:**

1. Đã tái lập **R3D1 baseline byte-for-byte**: `control/bl32.bin` có cùng SHA-256 với bản baseline đã được kiểm chứng trước đây.
2. Đã xây dựng lại diagnostic với đúng cấu hình R3D1; binary mới có SHA-256 `e3624fb4...61442ba` và entry `0x3E0036E0`.
3. Đã tạo gói ATF HW19I mới (header + payload + padding), **17.408 byte / 34 sector**; package SHA-256 `64c3aa8f...89cd69`.
4. Đã mô phỏng ghi package và rollback trên **synthetic image**: thay đổi chỉ trong 34 sector của package, vùng ngoài không thay đổi, rollback khôi phục byte-for-byte.
5. HW19J kiểm tra độc lập, HW19K đóng băng release: **PASS**.
6. **Chưa ghi diagnostic lên test SD**. Không có log runtime HW16 trên board. Không có bằng chứng xác định `FPGAMGR_NS=0` hay `1` ở runtime.

---

## 2. Timeline xử lý lỗi build reproducibility

| Bước        | Quan sát / nguyên nhân                                                                                                          | Kết quả                                   |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| HW18        | Binary, source delta, memory layout của diagnostic đời đầu đã kiểm tra                                                          | PASS ở mức host                           |
| HW19B–HW19D | Giải mã header ATF, phân biệt bản trước và sau lần ghi lịch sử; bản `after` chứa đúng R3D1                                      | PASS                                      |
| HW19E       | Tạo package diagnostic **HW17B đời đầu**, entry `0x3E003460`                                                                    | PASS, sau đó **không dùng để triển khai** |
| HW19F       | Đọc chỉ-đọc 34 sector từ test SD được người dùng xác nhận; khớp backup lịch sử R3D1                                             | PASS                                      |
| HW19G       | Kiểm tra độc lập package đời đầu và rollback snapshot                                                                           | PASS                                      |
| **HW19H**   | Build control `RESET_TO_SP_MIN=0`, khác baseline **11.538 bytes** mặc dù cùng kích thước 16.412 byte                            | `NEEDS_REVIEW`                            |
| **HW19H2**  | ELF entry lệch `0x3E0036A0` → `0x3E003420`; các symbol về sau cũng lệch                                                         | Chưa tái lập                              |
| **HW19H3**  | Source object R3D1 có **90**, control có **87**; thiếu `pmf_main.o`, `pmf_smc.o`, `ven_el3_svc.o`; 1 object chung khác          | Khoanh vùng PMF                           |
| **HW19H4**  | Thêm `ENABLE_PMF=1`, build chạy nhưng SHA-256 control vẫn `85f2acfe...`                                                         | `NEEDS_REVIEW`                            |
| **HW19H5**  | Object đạt **90/90**; **89 object trùng byte-for-byte**; chỉ `bl32/sp_min_main.o` khác                                          | Khoanh vùng build metadata                |
| **HW19H6**  | Khôi phục Git build string đúng, nhưng build timestamp khác: `02:41:40, Oct  5 2026` so với `03:38:12, Oct  8 2026`             | `NEEDS_REVIEW`                            |
| **HW19H7**  | Cố định `ENABLE_PMF=1`, `BUILD_STRING`, `BUILD_MESSAGE_TIMESTAMP`; `sp_min_main.o` và toàn bộ `bl32.bin` control trùng baseline | **PASS**                                  |
| HW19I       | Đóng gói diagnostic **mới**, mock update + rollback                                                                             | **PASS**                                  |
| HW19J       | Release readiness independent audit                                                                                             | **PASS**                                  |
| HW19K       | Release freeze, SHA256SUMS, rollback bundle                                                                                     | **PASS**                                  |
| HW20A       | Chuẩn bị UART trên Windows đã được hướng dẫn, **chưa có log COM discovery/boot**                                                | PENDING                                   |

### Nguyên nhân đã chứng minh

**Lỗi không nằm ở thuật toán diagnostic, cũng không phải do khác phiên bản GCC.** Build control lúc đầu chưa khớp cả về *tập object* lẫn *metadata nhúng trong `sp_min_main.o`*:

- Thiếu ba object vì **không bật `ENABLE_PMF=1`** trong lần build đối chứng. Sau khi bật, 90/90 object xuất hiện, 89 object trùng hoàn toàn; cả `lib/libc.a` có SHA-256 giống nhau.
- Chuỗi `build_version_string` trong R3D1 dài **54 byte**, còn ở control ban đầu **18 byte** do nguồn sao chép bỏ `.git` và không cung cấp chuỗi Git tương ứng.
- Khi khôi phục Git string, object vẫn khác vì `Built : ...` nhúng timestamp của ngày build mới.
- Sau khi cố định **Git build string + build timestamp gốc**, HW19H7 xác nhận `SP_MIN_MAIN_OBJECT_GATE=PASS` và `BASELINE_REPRODUCIBILITY_GATE=PASS`.

Đây là lời giải đã được **kiểm chứng bằng kết quả byte-for-byte**, không chỉ là giả thuyết từ việc nhìn hash.

---

## 3. Cấu hình build tái lập đã PASS

**Toolchain:** `arm-none-eabi-gcc` 14.2.1 (20241119).  
**Định dạng:** TF-A `AARCH32_SP=sp_min`, target Cyclone V.

| Biến Make                 | Giá trị đã PASS                                                       |
| ------------------------- | --------------------------------------------------------------------- |
| `PLAT`                    | `cyclone5`                                                            |
| `ARCH`                    | `aarch32`                                                             |
| `ARM_ARCH_MAJOR`          | `7`                                                                   |
| `AARCH32_SP`              | `sp_min`                                                              |
| `RESET_TO_SP_MIN`         | `0`                                                                   |
| `ENABLE_PMF`              | `1`                                                                   |
| `DEBUG`                   | `0`                                                                   |
| `CROSS_COMPILE`           | `arm-none-eabi-`                                                      |
| `BUILD_STRING`            | `phaseC3b1-bl33-context-hw-pass-dirty`                                |
| `BUILD_MESSAGE_TIMESTAMP` | `"02:41:40, Oct  5 2026"` (dấu ngoặc kép là **một phần của giá trị**) |

Phiên bản nhúng trong baseline:

```text
v2.13.0(release):phaseC3b1-bl33-context-hw-pass-dirty
Built : 02:41:40, Oct  5 2026
```

### Lệnh mẫu để tái lập build trong thư mục **mới** (chỉ khi thực sự cần)

**Không cần chạy lại hiện tại:** HW19H7 đã PASS. Đây là recipe để sau này reproduce từ source được đóng băng, **không dùng để thay thế artifact hiện hành** nếu chưa so hash.

```bash
# Thực hiện trong WSL Ubuntu. Build chạy trong Bash CON,
# nên nếu FAIL cũng không đóng shell WSL đang tương tác.
bash <<'BASH'
set -euo pipefail
ROOT="$HOME/secure_imgproc/optee_phase"
BASE="$ROOT/GATE_C_C1E_R3D1_minimal_runtime_bridge/20261005_024138"
DIAG="$ROOT/C3B1_HW16_fpgamgr_ns_diagnostic"
WORK="$DIAG/reproduce_manual_NEW"  # chọn tên chưa tồn tại

if [ -e "$WORK" ]; then
    echo "STOP: WORK_ALREADY_EXISTS"
    exit 1
fi
mkdir -p "$WORK/control/source" "$WORK/diagnostic/source"
rsync -a --exclude='.git/' --exclude='build/' \
    "$BASE/source/" "$WORK/control/source/"
rsync -a --exclude='.git/' --exclude='build/' \
    "$DIAG/source/" "$WORK/diagnostic/source/"

# BUILD_STRING được cấp tường minh, không cần .git trong bản sao.
for mode in control diagnostic; do
    make -C "$WORK/$mode/source" -j2 \
        PLAT=cyclone5 ARCH=aarch32 ARM_ARCH_MAJOR=7 \
        AARCH32_SP=sp_min RESET_TO_SP_MIN=0 ENABLE_PMF=1 DEBUG=0 \
        CROSS_COMPILE=arm-none-eabi- \
        BUILD_STRING='phaseC3b1-bl33-context-hw-pass-dirty' \
        BUILD_MESSAGE_TIMESTAMP='"02:41:40, Oct  5 2026"' \
        BUILD_BASE="$WORK/$mode/build" bl32 \
        > "$WORK/$mode.build.log" 2>&1 || {
            tail -n 40 "$WORK/$mode.build.log"
            exit 1
        }
done

# Chỉ xác nhận PASS nếu cmp trả về 0:
if cmp -s \
    "$BASE/source/build/cyclone5/release/bl32.bin" \
    "$WORK/control/build/cyclone5/release/bl32.bin"; then
    echo 'BASELINE_REPRODUCIBILITY_GATE=PASS'
else
    echo 'BASELINE_REPRODUCIBILITY_GATE=NEEDS_REVIEW'
    exit 1
fi
sha256sum \
    "$BASE/source/build/cyclone5/release/bl32.bin" \
    "$WORK/control/build/cyclone5/release/bl32.bin"
BASH
echo 'WSL_SESSION_STILL_ALIVE'
```

**Lưu ý shell:** Nếu chạy script nhiều lệnh có thể FAIL, dùng `bash <<'BASH' ... BASH` hoặc chạy script file trong tiến trình Bash con. Tránh bật `set -e` trực tiếp trong shell WSL tương tác, vì gate trả về 1 có thể làm phiên WSL kết thúc, đưa terminal về PowerShell.

---

## 4. Danh tính artifact cuối cùng (SHA-256)

Các giá trị đầy đủ dưới đây lấy từ **log đã PASS**:

| Artifact                     | SHA-256                                                                |
| ---------------------------- | ---------------------------------------------------------------------- |
| R3D1 baseline `bl32.bin`     | `a0492df3a0116a4eca49531c8eefb7e73acd752825df1d9c3f143e1eb605e70c`     |
| Control HW19H7 `bl32.bin`    | `a0492df3a0116a4eca49531c8eefb7e73acd752825df1d9c3f143e1eb605e70c`     |
| Diagnostic HW19H7 `bl32.bin` | `e3624fb40c5e56d9410a61e65e1eb6682fd31cc8d94519ffae78ea25961442ba`     |
| Diagnostic HW19H7 `bl32.elf` | `e0d105c2391b486fc5fa897fc222bb978b7365911c673d2633ab99192daa2b8c`     |
| **Final package HW19I**      | **`64c3aa8fff9110f83b96773647051293f846d1e9abce201b59f60de45889cd69`** |
| Rollback slot 34 sector      | `ae6628a8199e5694294dae3c4a2a90d49b5feabe7a943d5b3154d6355a44a481`     |

**Không sử dụng package HW19E cũ**, SHA `9247ae29009c6ec9e2c7fe141b18af51b7a6cc3d7ec7ddcf265c1a4d44e08570`, entry `0x3E003460`, dùng binary đời đầu chưa khôi phục đúng PMF và metadata. Package **HW19I** với entry `0x3E0036E0` là bản đã đạt final host verification.

### ELF memory layout của diagnostic mới

```text
Load address / first segment base: 0x3E000000
Entry point:                     0x3E0036E0
LOAD #1: 0x3E000000 – 0x3E004000
LOAD #2: 0x3E004000 – 0x3E006700
LOAD #3: 0x3E008000 – 0x3E011000
Diagnostic binary size:          16412 bytes
```

HW19J đã tái tạo binary từ ELF độc lập: `ELF_BINARY_RECONSTRUCTION_GATE=PASS` và kích thước `16412` byte.

---

## 5. ATF package, test SD và rollback

### Định dạng SPL ATF slot

| Trường                 | Giá trị                      |
| ---------------------- | ---------------------------- |
| Header magic           | `0x314C5053`                 |
| Version                | `2`                          |
| Name                   | `atf.bin`                    |
| Header size            | `512` bytes                  |
| ATF slot start LBA     | `0x1000` (4096)              |
| Payload start LBA      | `0x1001`                     |
| Package sectors        | `34` (LBA `0x1000`–`0x1021`) |
| Package size           | `17.408` bytes               |
| Payload size           | `16.412` bytes               |
| Load address           | `0x3E000000`                 |
| R3D1 entry             | `0x3E0036A0`                 |
| Diagnostic HW19I entry | `0x3E0036E0`                 |
| Next TEE slot header   | LBA `0x1800`                 |

Trong package mới, header giữ nguyên magic/version/name/load/flags, **chỉ cập nhật entry và payload SHA-256**, thay nội dung payload; phần padding 34 sector giữ nguyên từ backup. HW19I và HW19J đã xác nhận phạm vi thay đổi của header và payload.

### Test SD đã được kiểm tra như thế nào?

Tại HW19F, người dùng đã xác nhận USB SD reader chứa **test SD**. Tại thời điểm đó nó xuất hiện là `/dev/sde`, 3.7 GiB (model Storage Device, serial reader `121220160204`). Script **chỉ đọc** 34 sector tại LBA `0x1000` và xác nhận:

```text
CURRENT_HEADER_GATE=PASS
CURRENT_PACKAGE_INTEGRITY_GATE=PASS
CURRENT_R3D1_BASELINE_GATE=PASS
HISTORICAL_READBACK_MATCH=PASS
SD_WRITE_CALLED=NO
```

**Không được mặc định `/dev/sde` vẫn là test SD ở phiên sau.** Việc từng kiểm tra một snapshot không thay thế cho đối chiếu lại ngay trước khi ghi.

### Mô phỏng trên host tại HW19I

- Tạo synthetic image trong bộ nhớ, điền package R3D1 ở vùng slot ATF.
- Thay chính xác **34 sector** từ `0x1000` đến `0x1021` bằng package HW19I.
- `OUTSIDE_PACKAGE_UNCHANGED_GATE=PASS`, `TEE_REGION_UNCHANGED_GATE=PASS`.
- Chèn lại bản `rollback_atf_slot.bin`; `ROLLBACK_FULL_IMAGE_GATE=PASS`.
- Ghi các image mô phỏng và manifest trên host; kiểm tra đọc lại: PASS.
- Đây **không phải** phép ghi thử hoặc rollback vật lý trên thẻ SD.

### Backup tin cậy đang có

- Snapshot trước lần thử mới: `GATE_C_C1E_R3D3E_slot_write/20261005_031801/atf_slot_0x1000_after.bin`.
- Readback từ test SD tại HW19F: `C3B1_HW19F_20261008T031838Z/current_atf_slot.bin`.
- Hai file này khớp hoàn toàn tại HW19F/HW19J; SHA-256 của **toàn slot** là `ae6628a8...5a44a481`. Đừng nhầm với hash của **payload** `a0492df3...605e70c`.

---

## 6. Final release đã đóng băng (HW19K)

**Đường dẫn release WSL:**

```text
/home/manhlinh/secure_imgproc/optee_phase/C3B1_HW16_fpgamgr_ns_diagnostic/HW19K_release_freeze/
```

**Các file trong release bundle:**

| File                       | Vai trò                                               |
| -------------------------- | ----------------------------------------------------- |
| `baseline.bin`             | R3D1 gốc                                              |
| `control.bin`              | R3D1 build đối chứng, byte-for-byte khớp baseline     |
| `diagnostic.bin`           | TF-A HW19H7 diagnostic                                |
| `diagnostic.elf`           | ELF HW19H7, entry `0x3E0036E0`                        |
| `diagnostic_package.bin`   | **Package sử dụng cho bước chuẩn bị thử nghiệm HW20** |
| `rollback_atf_slot.bin`    | Bản khôi phục slot ATF đã đối chiếu                   |
| `sd_readback_snapshot.bin` | Snapshot từ lần đọc test SD ở HW19F                   |
| `baseline_source.c`        | Source baseline đối chứng                             |
| `diagnostic_source.c`      | Source có patch chẩn đoán HW16                        |
| `HW19I_manifest.txt`       | Manifest sau dry-run                                  |
| `SHA256SUMS`               | Danh mục hash các file được freeze                    |
| `RELEASE_README.txt`       | Hướng dẫn release và điều kiện rollback               |

Các gate cuối của HW19K:

```text
REPRODUCIBILITY_GATE=PASS
ROLLBACK_SNAPSHOT_GATE=PASS
DIAGNOSTIC_STATIC_AUDIT=PASS
PACKAGE_SIZE_GATE=PASS
PACKAGE_PAYLOAD_GATE=PASS
CHECKSUM_MANIFEST_GATE=PASS
ROLLBACK_INSTRUCTIONS_GATE=PASS
C3B1_HW19K_RELEASE_FREEZE=PASS
C3B1_HW19K_COMPLETE
HW19K_SCRIPT_EXIT=0
WSL_SESSION_STILL_ALIVE
```

**Xác minh lại release khi quay lại WSL (chỉ đọc file trên host):**

```bash
cd "$HOME/secure_imgproc/optee_phase/C3B1_HW16_fpgamgr_ns_diagnostic/HW19K_release_freeze"
sha256sum -c SHA256SUMS
```

---

## 7. Nội dung patch diagnostic và phạm vi bằng chứng

Diagnostic HW16 đọc lại thanh ghi bảo mật L4 sau thao tác set bit Non-secure đã có trong baseline và in một trong hai marker:

```text
C5TF:HW16:FPGAMGR_NS=0
C5TF:HW16:FPGAMGR_NS=1
```

- `C5_L4MP_NS_MASK` giữ giá trị `U(0x28)`; source audit HW19K: **PASS**.
- Các marker có mặt trong binary mới và source: **PASS**.
- **Chưa chạy binary trên board**, do đó **chưa biết marker nào thực tế xuất hiện**.
- Việc đọc bit bằng 0/1 tại thời điểm TF-A chạy **không đủ** để chứng minh toàn bộ root cause của Data Abort từng xảy ra khi U-Boot Normal World đọc FPGA Manager `0xFF706000`.
- Không thử `md.l 0xff706000 1`, `devmem 0xff2000xx` hay các thao tác bridge/FPGA Manager MMIO trực tiếp trong phép thử này.

---

## 8. HW20: bước tiếp theo, chưa triển khai ghi SD

**Đã chuyển sang bước chuẩn bị HW20A:** chuẩn bị Windows UART/PuTTY, 115200 baud, 8N1, no flow control, bật `All session output` để lưu log. Chưa có kết quả COM discovery hoặc log boot diagnostic được gửi ở thời điểm tổng hợp.

**Các điều kiện bắt buộc trước mọi lần thử HW20B thực tế:**

1. Có **chấp thuận rõ ràng mới** cho việc ghi riêng **test SD**; Golden SD tuyệt đối không ghi.
2. Nhận diện lại đúng thẻ vật lý, không tin vào tên thiết bị `/dev/sde` cũ.
3. Đọc lại package ATF hiện tại ở chế độ chỉ đọc và xác minh checksum với baseline dự kiến.
4. Tạo và kiểm tra backup **mới** ngay trước thao tác ghi, ngoài snapshot cũ.
5. Rà soát chính xác LBA đầu, số sector, package SHA-256 và phương án dừng / rollback trước thao tác bất kỳ.
6. Chỉ sau khi được phê duyệt và thực hiện cập nhật có kiểm soát mới boot board để tìm marker UART; kết thúc phép thử phải xem xét khôi phục test SD và kiểm tra baseline.

**Trạng thái cuối:** HW19K PASS; **SD_WRITE_CALLED=NO**, **HARDWARE_BOOT_CALLED=NO** trong chuỗi kiểm thử diagnostic. Không cần attach SD vào WSL lúc đang làm việc chỉ trên host.

---

## 9. Thoát và mở lại WSL

Đã thấy `WSL_SESSION_STILL_ALIVE` trong log cuối. Có thể thoát mà không mất source/binary nằm trong filesystem WSL (miễn là không xóa/reset distro):

```bash
exit
```

Mở lại từ PowerShell:

```powershell
wsl -d Ubuntu
```

Câu lệnh tiếp nối công việc: **“Tiếp tục từ C3B1-HW19K release freeze, chuẩn bị HW20A UART; chưa cho phép ghi test SD.”**

---

**Kết luận một dòng:** Ngày 08/10/2026 đã xử lý xong nguyên nhân *không tái lập được firmware* (PMF + Git build string + timestamp), xây dựng package diagnostic đúng từ bản R3D1 tái lập được, xác nhận package/rollback trên host, đóng băng release HW19K; **chưa can thiệp SD và chưa kiểm thử hardware boot**.
