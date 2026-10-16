# Daily Reproduce — Secure Boot / FPGA Kernel Driver — 2026-10-16

**Project:** Secure Image Processing on Terasic DE10-Nano / Cyclone V SoC, Cortex-A9 (ARM32)  
**Host:** Windows PowerShell + YMODEM (COM6) / WSL Ubuntu, `arm-linux-gnueabihf-gcc`  
**Boot stack:** U-Boot SPL 2025.07 → TF-A SP_MIN 2.13 → OP-TEE 3.21 → U-Boot 2025.07 → Linux 5.15.221-dirty → FPGA kernel driver / image-processing TA  
**Reproduce range:** R35K_3E → R35K_3F → R35K_3G → R35K_3H → R35K_3I → R35K_4A  
**Evidence:** UART logs `Pasted text(20261016-013948).txt`, `(20261016-014421).txt`, `(20261016-021739).txt`, `(20261016-025429).txt`, `(20261016-031856).txt`; user-supplied FPGA/bridge recovery and CRC logs; WSL R35K_4A builder output.  
**Evidence policy:** `PASS` means the specific gate was observed passing. It does **not** imply completed authenticated secure boot, all seven specification ioctls, or production security.

## 1. Executive summary

**Completed on actual DE10-Nano:**

- **R35K_3F:** Transferred the R35K_3D initramfs into RAM via PowerShell/YMODEM, checked size `0xB00A00` and CRC32 `9C7621AE`, booted Linux/OP-TEE/HW77/HW88 in **inventory-only** mode. `xtest 1006`, image-processing CA, and read-only SMC checks passed within their documented scope. Worker confirmed no driver load and no FPGA write.
- **R35K_3G:** Identified FPGA configuration and Lightweight HPS-to-FPGA bridge issue. `HW95O_compressed.rbf` loaded from SD with CRC32 `87D23A06`; FPGA Manager entered User Mode (`STAT=0x54`). Read of FPGA MMIO initially **HUNG** with `BRGMODRST=0x7`. Changed `BRGMODRST` to `0x5` after confirming preconditions, restoring FPGA STATUS read at `0xFF200004` to `0x1`.
- **R35K_3H:** Booted same R35K_3D initramfs with `r35k3d.load=1 r35k3d.fpga_verified=1`. `fpga_imgproc.ko` probed successfully; ABI, status, raw diagnostics and argument-validation regression passed. The test made **no FPGA MMIO writes**. `GET_STATUS` and `GET_HWUID` returned unsupported as expected; these functions are **not implemented**.
- **R35K_3I:** Booted with additional `r35k3d.tee=1`. Successfully executed actual **SIGN** and **WATERMARK** ioctls via kernel driver → kernel TEE Client → OP-TEE TA. `--tee-test` exited 0; worker exited 0. TA uses **test keys**, not production keys. SIGN regression verified success/nonzero signature, not a full independent kernel-path signature verification.

**Completed on host only:**

- **R35K_4A:** Repacked the previously booted R35K_3D CPIO with a new guarded ARM32 static worker for MMIO write regression. Build, unchanged payload audit, CPIO NEWC roundtrip and base identity passed. **New image has not yet been booted or subjected to MMIO write regression on board.**

**End-of-day checkpoint:** `R35K_3I_HARDWARE_TEE_REGRESSION=PASS`, `R35K_4A_HOST_BUILD=PASS`, `R35K_4A_HARDWARE_PREFLIGHT=PENDING`.

## 2. Chronological reproduce and gates

### 2.1 R35K_3E — U-Boot, DTB, FPGA Manager and transfer-path audit

- SPL verified SHA-256 of `atf.bin`, `tee.bin`, `u-boot.img`; boot reached U-Boot via TF-A/OP-TEE. **SPL authentication remained `NOT ENABLED YET`, not a signature-verified boot.**
- SD DTB loaded at `0x03000000`; `/soc/fpga-imgproc@ff200000` exists with `compatible="tecotec,fpga-imgproc"`, `reg=<0xff200000 0x1000>`, `status="okay"`. `timer1@ffc09000` needed to be disabled in RAM for the known Linux boot path.
- U-Boot Ethernet unavailable: `No valid MAC address found`, `No ethernet found`, `ipaddr/serverip/ethaddr` undefined. `loady` and `loadb` were available. Chose the **known-working PowerShell + PySerial/YMODEM** path rather than TFTP.
- FPGA Manager STAT initially `0x00000050` (`MODE 0`, `power off`); `bdinfo` showed 1 GiB DDR address range with U-Boot reserved high memory, compatible with using the established lower RAM addresses.

### 2.2 R35K_3F — YMODEM transfer, CRC fix, inventory boot

**Transferred:** `R35K_3D_preflight_initramfs.cpio` to RAM `0x04000000` using U-Boot `loady 0x04000000 115200` and PowerShell Python sender on COM6. Host sender reported final YMODEM handshake PASS.

- RAM size `filesize=b00a00` (11,536,896 bytes).
- Initial `crc32 -v 0x04000000 0xB00A00 0x9C7621AE` failed with `Expected 8 hex digits in env var`: this U-Boot variant interpreted the last operand of `-v` as an environment variable name. **Not a transfer corruption.**
- Fixed check with `crc32 0x04000000 0xB00A00`: returned **`9c7621ae`**, matching host.
- Loaded tested kernel/DTB from SD into separate RAM addresses, disabled `timer1` in RAM, then booted `bootz 0x01000000 0x04000000:0xB00A00 0x03000000` with normal `hw88.execute=1` and no R35K module flags.
- Kernel 5.15.221-dirty booted `/init`; `/dev/tee0`, `/dev/teepriv0`, OP-TEE 3.21, `xtest 1006` (0 failures), CA SIGN/VERIFY/WATERMARK, HW88 read-only SMC checks succeeded.
- Worker preflight confirmed `TA_PRESENT=1`, `DRIVER_PRESENT=1`, `STATIC_TEST_PRESENT=1`, `TEE0_PRESENT=1`, `DT_COMPATIBLE=FOUND`, `FPGA_MANAGER_STATE=power off`, `MODULE_LOAD_REQUESTED=0`, `MODE=INVENTORY_ONLY`, `NO_FPGA_WRITE=YES`, `WORKER_EXIT=0`.

### 2.3 R35K_3G — FPGA configuration and LWH2F bridge recovery

Read **existing known-good bitstream** from SD (not a write to SD):

```text
mmc read 0x08000000 0x772800 0xF67
crc32 0x08000000 0x001ECD4C
=> 87d23a06
fpga load 0 0x08000000 0x001ECD4C
md.l 0xff706000 1
=> 00000054       # FPGA Manager User Mode
```

**Observed failure:** `md.l 0xff200004 1` hung after the first FPGA load even though `fpga load` returned and the Manager was in User Mode.

**Root cause supported by recovery test:** LWH2F bridge remained asserted in reset:

```text
md.l 0xffd0501c 1  => 00000007  # before
mw.l 0xffd0501c 0x00000005
md.l 0xffd0501c 1  => 00000005  # after
md.l 0xff200004 1  => 00000001  # FPGA STATUS, read succeeds
```

**Result:** FPGA configuration **PASS**, LWH2F bridge recovery **PASS**, FPGA STATUS MMIO read **PASS**. The accidentally entered `d.l` (unknown command) was corrected to `md.l`; no effect on the final result. Only perform the reset-register write when the same FPGA Manager and register preconditions hold.

### 2.4 R35K_3H — Unified Kernel Driver probe and read-only regression

Booted again from the verified R35K_3D CPIO (`size=0xB00A00`, `CRC32=9C7621AE`) after restoring FPGA and LWH2F. Bootargs added `r35k3d.load=1 r35k3d.fpga_verified=1` (no TEE or MMIO-write flag).

Key evidence:

```text
R35K_3D_FPGA_MANAGER_STATE=operating
fpga_imgproc ff200000.fpga-imgproc: probe OK: MMIO [mem 0xff200000-0xff200fff], STATUS=0x00000001, ABI=0x00010000
R35K_3D_FINIT_MODULE_RC=0 ERRNO=0
R35K_3B_LEGACY_VERSION=PASS
R35K_3B_EXT_VERSION=PASS
R35K_3B_READ_STATUS=PASS
R35K_3B_RAW_DIAG=PASS
FPGA_STATUS=0x00000001
IOCTL_GET_STATUS_RC=-1 ERRNO=95
R35K_3B_GET_STATUS_FAIL_CLOSED=PASS
IOCTL_GET_HWUID_RC=-1 ERRNO=95
R35K_3B_HWUID_FAIL_CLOSED=PASS
IOCTL_SIGN_NULL_POINTER_RC=-1 ERRNO=22
R35K_3B_SIGN_VALIDATE=PASS
IOCTL_WM_NULL_POINTER_RC=-1 ERRNO=22
R35K_3B_WM_VALIDATE=PASS
R35K_3B_RESULT=PASS
R35K_3D_PROBE_REGRESSION=PASS
R35K_3D_TEE_REGRESSION=SKIPPED
R35K_3D_FPGA_MMIO_WRITES=NO
R35K_3D_WORKER_EXIT=0
```

**Interpretation:** Driver/legacy and V2 ABI paths passed their read-only checks. `ERRNO=95 (EOPNOTSUPP)` shows missing full `GET_STATUS` / HWUID backends; a fail-closed regression PASS is **not** completion of the specification feature. NULL-pointer checks returned `EINVAL=22` correctly.

### 2.5 R35K_3I — Real kernel → OP-TEE SIGN/WATERMARK ioctl

Restored FPGA/bridge and booted verified R35K_3D with `r35k3d.load=1 r35k3d.fpga_verified=1 r35k3d.tee=1`. Earlier HW77/88 tests still passed; then kernel module probe and read-only ioctl regression passed again.

Key evidence:

```text
IOCTL_SIGN_FRAME_RC=0 ERRNO=0
R35K_3B_TEE_SIGN_IOCTL=PASS
R35K_3B_TEE_SIGN_NONZERO=PASS
IOCTL_WATERMARK_RC=0 ERRNO=0
R35K_3B_TEE_WM_IOCTL=PASS
WATERMARK_CHANGED_BYTES=96 MAX_ABS_DIFF=1
R35K_3B_TEE_WM_DELTA=PASS
R35K_3B_TEE_WM_RANGE=PASS
R35K_3B_TEE_WM_BITS=PASS
R35K_3B_TEE_KEY_CONTEXT=TEST_KEY_ONLY_NOT_PRODUCTION
R35K_3B_RESULT=PASS
R35K_3D_TEST_--tee-test_EXIT=0
R35K_3D_TEE_REGRESSION=PASS
R35K_3D_WORKER_EXIT=0
```

**Interpretation:** Hardware-backed call chain `userspace ioctl → fpga_imgproc.ko → Linux TEE Client → OP-TEE TA` demonstrated for SIGN and WATERMARK. Regression tested a 16×16 grayscale sample and 128-bit watermark; 96 bytes changed by at most one grayscale unit. This **does not** establish production signing keys or proof that authenticated boot is enforced. Host CA test and kernel ioctl test are distinct.

### 2.6 R35K_4A — Guarded MMIO writer repack (WSL host only)

Repacked the **exact R35K_3D previously booted** CPIO, replacing only the static worker and auditing all remaining payloads. New worker preserves inventory-only default; module load still requires `r35k3d.load=1` and `r35k3d.fpga_verified=1`, FPGA Manager `operating`, DT compatible, etc. The MMIO writer is additionally gated on **both** explicit flags `r35k4a.mmio=1` and `r35k4a.mmio_write_approved=1` plus a passing `--probe` and `CTRL.PIPE_ENABLE=0`. The write test is intended to exercise `SET_CONFIG`, `SET_THRESHOLD`, and custom-IP `RESET`, **not** write SD or reprogram FPGA.

Actual user WSL build log:

```text
R35K_4A_BASE_IDENTITY=PASS
R35K_4A_WORKER_ARM32_STATIC=PASS
R35K_4A_UNCHANGED_PAYLOADS=PASS
R35K_4A_CPIO_NEWC_ROUNDTRIP=PASS
R35K_4A_SIZE=11536896 HEX=0xB00A00
R35K_4A_CRC32=C53E7541
R35K_4A_SHA256=366325fa781165f308040c8ba4be87ab90e50af96f094e5a5daa4d0b82eb9557
R35K_4A_HOST_BUILD=PASS
BOARD_BOOT=NOT_CALLED SD_WRITE=NO
```

**New output on Windows D:** `D:\SecureBoot_share\R35K_4A_mmio_optin_initramfs.cpio` and corresponding `R35K_4A_mmio_optin_initramfs.json` manifest. The new CPIO **happens to have the same size** as R35K_3D but different CRC32 and SHA-256. This build log supports host gate PASS only; independent board CRC and runtime gates are still outstanding.

## 3. Problems encountered and fixes

| Problem | Evidence / cause | Resolution / status |
|---|---|---|
| U-Boot Ethernet/TFTP unavailable | MAC absent, networking not registered | Reused working COM6 PowerShell/PySerial YMODEM sender; transfer integrity proven by U-Boot CRC |
| Incorrect `crc32 -v` usage | U-Boot expected a variable name, not literal hash | Used `crc32 <addr> <size>` and compared `9C7621AE`; transfer PASS |
| FPGA Manager initially `power off` | STAT `0x50`, FPGA bitstream not configured | Reused known-good HW95O RBF (`CRC32 87D23A06`), Manager became `0x54` |
| HPS MMIO read hung after `fpga load` | `BRGMODRST=0x7`, LWH2F held in reset | Under verified preconditions wrote `0x5`; direct FPGA STATUS read returned `1` |
| Kernel previously needed timer workaround | DTB timer1 caused earlier boot problems | Disabled `timer1@ffc09000` only in RAM; Linux 5.15 and OP-TEE booted |
| Full `GET_STATUS`, real HWUID unsupported | ioctl returns `errno=95` | Negative regression PASS; features remain NOT DONE |
| New MMIO regression could modify live FPGA state | `SET_CONFIG` / `SET_THRESHOLD` / IP `RESET` are writes | R35K_4A adds dual explicit opt-in plus preconditions; NOT RUN on board yet |

## 4. Integrity checkpoints / reusable files

| Artifact | Size | CRC32 / SHA-256 |
|---|---:|---|
| Known-good `HW95O_compressed.rbf` from SD | `0x1ECD4C` | CRC32 `87D23A06` (reconfirmed on U-Boot) |
| R35K_3D initramfs | 11,536,896 (`0xB00A00`) | CRC32 `9C7621AE`; SHA-256 `bd2a84591463bb84013029f521010927895205025c08b69187cbfe69ab90e863` |
| R35K_3B kernel module | 243,724 bytes | SHA-256 `319a16252d1f6dd636747117e44b067efaa4c693d8410b627cc07477e6f4d161` (prior host source artifact) |
| R35K_4A initramfs (new, host-built) | 11,536,896 (`0xB00A00`) | CRC32 `C53E7541`; SHA-256 `366325fa781165f308040c8ba4be87ab90e50af96f094e5a5daa4d0b82eb9557` (WSL log) |

## 5. Requirements checklist at end of 16/10

| Specification area | Status | Important limitations |
|---|---|---|
| SPL/TF-A/OP-TEE/Linux boot | **HW PASS** | SHA-256 checks do not prove authenticated/signed boot |
| FPGA Manager + Lightweight bridge | **HW PASS after explicit recovery** | Correct bridge reset sequence must be repeated after board reset; not yet automated |
| Unified driver `.ko`, device probe | **HW PASS** | MMIO V2 probe with FPGA `STATUS=1` |
| Legacy/version/read-only ioctls | **HW PASS** | Raw diagnostic and read-status verified |
| Kernel ioctl `TRIGGER_SIGN` | **HW PASS** | Nonzero MAC / TA test key; independent verification test still desirable |
| Kernel ioctl `WATERMARK` | **HW PASS** | 128-bit test on 256-byte sample, max difference 1 |
| New R35K_4A MMIO SET_CONFIG/THRESHOLD/RESET | **HOST BUILD PASS; HW PENDING** | Earlier R34E wrote MMIO successfully, but R35K_4A itself not booted/tested |
| Full `GET_STATUS` backend | **NOT DONE** | Expected `EOPNOTSUPP` |
| Real `GET_HWUID` | **NOT DONE** | Firmware returns not supported; fail closed |
| DMA, IRQ, V4L2 `/dev/video1` | **NOT DONE** | No validated path to video streaming |
| Authenticated boot, production keys | **NOT PROVEN / NOT DONE** | SPL still says authentication not enabled; CA/TA test key only |

## 6. Resume instructions for the next session (R35K_4A)

**Checkpoint:** `R35K_4A_HOST_BUILD=PASS`; do not rebuild unless the files differ. Copy sender `send_r35k4a_ymodem.py` to the same `D:\SecureBoot_share` directory as the new `.cpio` and `.json` manifest; confirm all exist and verify SHA-256 before exiting WSL (`exit` is safe after work is saved).

**Recommended sequence:**

1. Boot into U-Boot. Read known-good RBF from SD, check CRC32 **`87D23A06`**. Load it; verify `FPGA Manager STAT=0x54`. If and only if `BRGMODRST=0x7`, use the previously verified recovery to `0x5`, then check `0xFF200004 = 1`. No unverified MMIO operations.
2. `loady 0x04000000 115200` on U-Boot; close PuTTY without powering off; send **R35K_4A** CPIO via **PowerShell/Python YMODEM on COM6** using `send_r35k4a_ymodem.py` and manifest. Check U-Boot `filesize=b00a00` and **CRC32 `c53e7541`**. Do not mistakenly use old `9c7621ae` CRC.
3. Load known-good kernel/DTB (`mmc read 0x01000000 0x3000 0x2C39` and `mmc read 0x03000000 0x6000 0x27`), patch `timer1` in DTB **RAM only**, then **first boot with no MMIO-write flags**: `r35k3d.load=1 r35k3d.fpga_verified=1` and `hw88.execute=1`. Confirm new worker probe regression PASS and no FPGA writes.
4. **Separate deliberate boot** only after the first R35K_4A preflight PASS and operator approval: restore FPGA/bridge, retransmit/verify CPIO, add **both** `r35k4a.mmio=1 r35k4a.mmio_write_approved=1`, run guarded MMIO regression to read back `SET_CONFIG`, threshold, RESET. This step changes custom IP state. Do not run it automatically.
5. Preserve exact UART/WSL logs, hashes and gates. On any identity/state mismatch or hang, STOP. Do not use `mmc write` or `saveenv`, and do not modify HW88 known-good images.

### End-of-day status markers

```text
DATE=2026-10-16
R35K_3F_YMODEM_RAM_CRC=PASS
R35K_3F_INVENTORY_BOOT=PASS
R35K_3G_FPGA_USER_MODE=PASS
R35K_3G_LWH2F_BRIDGE_RECOVERY=PASS
R35K_3G_FPGA_STATUS_READ=PASS
R35K_3H_KERNEL_MODULE_PROBE=PASS
R35K_3H_READONLY_IOCTL_REGRESSION=PASS
R35K_3I_KERNEL_TEE_SIGN=PASS
R35K_3I_KERNEL_TEE_WATERMARK=PASS
R35K_3I_TEE_REGRESSION=PASS
R35K_4A_HOST_BUILD=PASS
R35K_4A_HARDWARE_PREFLIGHT=PENDING
R35K_4A_MMIO_WRITE_REGRESSION=NOT_RUN
GET_STATUS_FULL=NOT_IMPLEMENTED
GET_HWUID_REAL=NOT_SUPPORTED
DMA_IRQ_V4L2=NOT_DONE
AUTHENTICATED_BOOT=NOT_PROVEN
PRODUCTION_KEY=NO
SD_WRITE_BY_R35K_WORKFLOW=NO
```

**Conclusion:** On 16/10/2026, the project progressed from a bootable inventory-only image to **hardware-verified unified kernel driver + FPGA MMIO read + OP-TEE-backed SIGN/WATERMARK**, and finished building a guarded MMIO writer candidate. **Next start = R35K_4A board preflight; do not claim MMIO write PASS until there is its own board log.**
