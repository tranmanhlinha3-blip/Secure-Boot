# Daily Reproduce — Secure Boot DE10-Nano — 2026-10-19

**Project:** Secure Boot / Secure Image Processing on Intel DE10-Nano (Cyclone V, Cortex-A9, AArch32)  
**Workstream:** `F1D3C7`, TF-A SP_MIN, MMU/XLAT runtime descriptor probe, ATF packaging, test SD deployment  
**Environment:** WSL Ubuntu on Windows; project root `/home/manhlinh/secure_imgproc`; audit/output root `/mnt/d/SecureBoot_share/output/`  
**Scope of this report:** Activities and evidence recorded for 19 October 2026. Historical PuTTY boot is used only as **pre-deployment baseline**. It does **not** prove the new debug ATF boots.

## 1. Executive summary

- **PASS — Host/static TF-A:** Audited AArch32 SP_MIN XLAT mapping and table allocation, including expected RO executable, RW execute-never and Device mappings. `G3B_STATIC_GATE=PASS` is a **model/source-based** assessment, not a hardware translation-table readback.
- **PASS — Probe build:** Produced TF-A G3C2A debug `bl32.bin` (20,516 bytes) with `c5_xlat_debug_dump` and `C5XLAT` UART instrumentation, with source, binary and linker/call-path checks passing G3C2B.
- **PASS — Package:** Discovered historical ATF slot at absolute SD **LBA 4096**, **2,048 sectors × 512 B = 1 MiB**. Verified `SPL1` version 2 header and SHA-256 payload at byte offset **512**. Created G3C3C2 debug ATF slot with valid entry `0x3E0048A0` and SHA-256.
- **PASS — SD evidence and recovery:** Identified **test SD** `/dev/sde` by serial `121220160204`, size `4,007,657,472` B and 512-B logical sectors. Read and froze the original ATF slot and first 2 MiB of SD, and verified the contents were unchanged immediately before writing.
- **PASS — Actual SD write + readback:** G3C3E `apply` was manually confirmed, wrote exactly 1 MiB ATF slot starting at LBA 4096, then independently read it back. Readback matched the candidate **byte-for-byte** (`ff311083...`, PASS).
- **PENDING — New hardware execution:** No UART boot log for the new G3C2A debug ATF after deployment has been provided. In particular, **`C5XLAT:BEGIN` / `C5XLAT:END` and descriptor values are not yet observed on board**. Full boot regression and CPU1/PSCI remain unproven.

## 2. Chronological gate history

| Gate | Evidence / work done | Result / limitation |
|---|---|---|
| `F1D3C7-B/C/E/F` | Inspected TF-A source/artifacts and AArch32 MMU/Xlat initialization flow, linker objects and boot path | Host analysis completed; not runtime proof |
| `G2`, `G3A`, `G3B` | Audited secure mapping model, L1/L2 alignment, table size and BL32 bounds | `G3B_STATIC_GATE=PASS`; runtime descriptors not tested |
| `G3C0` | Warmboot and MMIO preflight | Hardware warmboot behavior not proven |
| `G3C1` | PSCI gate | `PSCI_CALLBACK_GATE=BLOCKED_NO_CALLBACKS`; CPU1 not supported/proven |
| `G3C2A` | Built debug `bl32.bin`, `bl32.elf`, inserted `c5_xlat_debug_dump` after `init_xlat_tables()` | `DEBUG_BUILD_RC=0`, `PROBE_LINKAGE=PASS` |
| `G3C2B` | Verified ELF entry, `mmap_add_region`, `init_xlat_tables`, probe call, symbol and bounds | `G3C2B_HOST_GATE=PASS` |
| `G3C3A/B` | Compared debug/release artifacts; audited historical images and slot tool | Historical ATF slot geometry established |
| `G3C3C0` | Compared baseline, historical candidate and R22 before/after; decoded historical `SPL1` manifests | Source files and historical identities PASS |
| `G3C3C1` | Recomputed SHA-256 over historical payload, located payload offsets, found SPL verifier source | Three samples matched at offset `512` |
| `G3C3C2` | Created new 1 MiB slot from known baseline header + debug payload; checked preserved header/padding | `OFFLINE_PACKAGE_GATE=PASS` |
| `G3C3C3` | Compared candidate manifest to `spl_secure_load.c`: magic, version, slot, entry checks, hash-before-handoff | `HOST_CONTRACT_GATE=PASS`; source-to-live-binary **not proven** |
| `G3C3D0` | Test SD identification and **read-only** 2 MiB + ATF slot readback | Device identity PASS; current slot not a known historical backup |
| `G3C3D1` | Frozen the captured readback files byte-for-byte | `G3C3D1_OFFLINE_FREEZE_GATE=PASS` |
| `G3C3D2` | Reused and reconciled historical PuTTY serial boot log with current SD manifest values | **Baseline CPU0** boots SPL → SP_MIN → OP-TEE init → U-Boot prompt; does not prove Linux |
| `G3C3D3` | Fresh read-only SD comparison before write | First 2 MiB and ATF slot unchanged: PASS |
| `G3C3E apply` | Manually confirmed **write** to exactly ATF LBA 4096, 2,048 sectors; mandatory full readback | `G3C3E_APPLY_GATE=PASS`, `SLOT_BYTE_EXACT=PASS` |
| `G3C3E runtime` | Boot debug image and capture `C5XLAT` | **PENDING — no post-write UART log** |

## 3. TF-A artifacts and architecture

### 3.1 Release and debug builds

| Artifact | Location / metadata | SHA-256 |
|---|---|---|
| Release SP_MIN `bl32.bin` | `/home/manhlinh/secure_imgproc/F1D3C7_G_XLAT_HOST_20261019_095557/source/build/cyclone5/release/bl32.bin` (16,412 B) | `fda5631ffe635eb7883380b9e4c7c59e7279334685f6a7a007fc544275333743` |
| Debug SP_MIN `bl32.bin` | `/home/manhlinh/secure_imgproc/F1D3C7_G3C2A_XLAT_PROBE_20261019_141442/source/build/cyclone5/debug/bl32.bin` (20,516 B) | `1631b933f6327a828df17fe35bfdc5f993a83f6077cefb291ddcc42dea73a70b` |
| Debug SP_MIN ELF | `.../source/build/cyclone5/debug/bl32/bl32.elf`; ELF entry `0x3E0048A0` | `2d1836dc55d5181ef6730ca7ed218a24a40ec83a7c3da0893890397607ad3fed` |
| SPL loader source audited | `boot_phaseA/u-boot-socfpga/arch/arm/mach-socfpga/spl_secure_load.c` | `26e89b319598b6b3bdab679a5f63351eb2d1d6072958abcdb0a243b059c9450e` |

The `G3B` static model includes RO executable mapping near `0x3E000000`; RW XN at `0x3E004000`; XLAT L1 at `0x3E00C000`; coherent area at `0x3E010000`; and Device mappings for UART0, clocks, reset, system manager, SCU, GICC/GICD. Some addresses (such as HPS remap and LWH2F) are **unmapped in the audited static model**, not necessarily inaccessible under every later runtime configuration. Hardware descriptor verification is pending.

`G3C1` found missing PSCI CPU1 callbacks. The test scope therefore stays CPU0; CPU1 is not marked PASS.

### 3.2 ATF package format

| Property | Value |
|---|---|
| Absolute ATF slot start | LBA `4096` = byte offset `0x00200000` |
| ATF slot length | 2,048 sectors = **1,048,576 B** |
| Header | `SPL1`, version `2`, 512 B |
| Payload start relative to slot | Byte `512` = absolute LBA `4097` |
| Manifest image name | `atf.bin` |
| New payload length | **20,516 B** |
| New load | `0x3E000000` |
| New entry | `0x3E0048A0` |
| Payload digest | `1631b933f6327a828df17fe35bfdc5f993a83f6077cefb291ddcc42dea73a70b` |
| New slot SHA-256 | `ff311083dc939ec7a4945f41a83b56496f5f9d136856b91ebb09ed5f162eaf07` |
| New slot path | `/mnt/d/SecureBoot_share/output/F1D3C7_G3C3C2/G3C3C2_debug_atf_slot_HOST_ONLY.img` |

In the **audited** SPL source, the loader checks the header, image name, load and entry constraints and sector bounds, then loads the payload, verifies its SHA-256, and records the ATF entry only on success. **Authenticity / signed boot is not established by this digest comparison alone.** The source file was marked modified in Git, and an exact provenance match to the live SPL binary remains unproved.

## 4. Baseline board evidence (before G3C3E write)

Previously captured PuTTY log on 19 October (also discussed in the chat) showed:

```text
SPL: found  atf.bin size=16412 bytes
SPL:  atf.bin LOAD PASS entry=0x3e003a20
SPL: atf.bin SHA256 PASS
SPL: found  tee.bin size=321608 bytes
SPL: tee.bin SHA256 PASS
SPL: found u-boot.img size=528332 bytes
SPL: u-boot.img SHA256 PASS
SPL: ALL IMAGES LOAD + SHA256 PASS
SPL: Handoff -> TF-A SP_MIN entry=0x3e003a20
C5TF: GICv2 INIT PASS
C5TF: C3B2 BL33 CONTEXT PREP PASS
I/TC: Primary CPU switching to normal world boot
C5TF:G15ZN:B1 ENTRY_DONE -> RESTORE BL33
U-Boot 2025.07-gc7262815140f-dirty (Oct 14 2026 - 03:48:58 +0000)
=>
```

**Baseline result:** CPU0 SPL, TF-A SP_MIN, OP-TEE primary initialization and U-Boot proper prompt were observed. **Not observed:** Linux boot, OP-TEE Trusted Application acceptance tests, secure DDR isolation, CPU1 bring-up. Boot messages such as `OP-TEE execution: NOT ENABLED YET` and `Authentication: NOT ENABLED YET` were still printed by SPL source; they are not evidence that the later TF-A/OP-TEE chain never executed.

**Baseline warnings observed:** OP-TEE `plat_rng_init` seeding RNG with zeroes; U-Boot environment bad CRC/default environment; no valid Ethernet MAC; intermittent UART interleaving. None was fixed as part of G3C3E.

## 5. Test SD identity and rollback

| Item | Value |
|---|---|
| Identified test SD | `/dev/sde` **at audit time only**; do not assume persistent device name |
| Serial | `121220160204` |
| Capacity | `4,007,657,472 B` |
| Logical sector | 512 B |
| Frozen first 2 MiB | `F1D3C7_G3C3D1_FREEZE_2VboCTW8/first_2MiB_FROZEN.bin` |
| SHA-256, first 2 MiB | `a4829f1e0554e1c40548ad77cb03a76b1c72573fc7c01bd88d8f115f10e8a95a` |
| Frozen ATF rollback | `F1D3C7_G3C3D1_FREEZE_2VboCTW8/current_atf_slot_ROLLBACK_FROZEN.img` |
| SHA-256, rollback ATF slot | `862977395f2c0a2eb77c9192851c01094f2512645aadb45cbca6997fd5218e9f` |
| Snapshot state | Byte-for-byte verified offline, not a proof of future successful recovery |

The current baseline ATF slot differed from older `HW95G` backup (`fdc9b895...`) and historical R18 candidate (`6093f5f8...`), which is why its **actual live readback** was separately preserved before writing. Baseline ATF payload size was 16,412 B, load `0x3E000000`, entry `0x3E003A20`.

Two different families of 64 KiB SPL replicas were discovered in the first 2 MiB:

- Primary (4 copies at `0x000000` ... `0x030000`): SHA-256 `64cf610efbb5af93b0ea8713c961226ac72adf5adc0a21fdd82b6cdd6e5e6020`.
- Secondary (4 copies at `0x100000` ... `0x130000`): SHA-256 `10b220b35de014f0ce1696610ee27165626811c4ab9799fa8c26d7b18e268ce0`.

Neither matched historical production SPL SHA-256 `430203ce648db6ce43c9a9e5016a1610d71c4507ab04a01949d082c57cacab9a`. The serial log shows SPL executed, but **which replica BootROM selected remains unproved**.

## 6. G3C3D3 final read-only check and G3C3E apply

### Final prewrite G3C3D3

```text
FIRST_2M_UNCHANGED=PASS
ATF_SLOT_UNCHANGED=PASS
CANDIDATE_IDENTITY=PASS
SD_WRITE=NOT_PERFORMED
```

### Actual G3C3E apply — manually confirmed

```text
MODE=apply
DEVICE_IDENTITY=PASS DEVICE=/dev/sde SERIAL=121220160204 SIZE=4007657472 SECTOR=512
DEVICE_UNMOUNTED=PASS
FIRST_2M_UNCHANGED=PASS
LIVE_SLOT_STATE=BASELINE
FINAL_PREWRITE_GATES=PASS
2048+0 records in
2048+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied
EXPECTED_SLOT_SHA256=ff311083dc939ec7a4945f41a83b56496f5f9d136856b91ebb09ed5f162eaf07
READBACK_SLOT_SHA256=ff311083dc939ec7a4945f41a83b56496f5f9d136856b91ebb09ed5f162eaf07
READBACK_GATE=PASS
SLOT_BYTE_EXACT=PASS
SD_WRITE=PERFORMED_AFTER_MANUAL_CONFIRMATION
G3C3E_APPLY_GATE=PASS
```

The operation was performed using the separately checked diagnostic tool `F1D3C7_G3C3E_TEST_SD_SLOT_TOOL.sh`, mode `apply`, after the operator entered its exact physical-safety confirmation. **It targeted the ATF region only.** No claim is made that the SD card's remaining bytes were all read back after writing; the documented readback gate covers the entire 1 MiB ATF slot.

The actual run report path shown in the terminal was:

`/mnt/d/SecureBoot_share/output/F1D3C7_G3C3E_apply_bS5c0hIH/G3C3E_APPLY_REPORT.txt`

**Do not rerun `apply` to reproduce a completed operation.** Repeating a destructive storage write is not necessary to document it. Keep the frozen backup and full journal.

## 7. File evidence to preserve in GitHub / archive

**Host logs (small; safe to version after path/security review):**

- `F1D3C7_G3B_XLAT_MODEL.txt`
- `F1D3C7_G3C1_PSCI_GATE.txt`
- `F1D3C7_G3C2A_PROBE_BUILD.txt`, `F1D3C7_G3C2B_PREFLIGHT.txt`
- `F1D3C7_G3C3A_PACKAGE_AUDIT.txt`, `F1D3C7_G3C3B_SLOT_AUDIT.txt`
- `F1D3C7_G3C3C0_OFFLINE_AUDIT.txt`, `F1D3C7_G3C3C1_SPL_VERIFIER_AUDIT.txt`
- `G3C3C2_REPORT.txt`, `F1D3C7_G3C3C3_SPL_CONTRACT_AUDIT.txt`
- `G3C3D0_REPORT.txt`, `G3C3D1_REPORT.txt`, `G3C3D3_REPORT.txt`
- `G3C3E_APPLY_REPORT.txt` (from actual WSL output path above)
- Pre-deployment PuTTY log `Pasted text(20261019-083920).txt` (or a normalized baseline extract with the full source retained).

**Immutable binary evidence (store in controlled artifacts/releases, not automatically as regular source commits):**

- `G3C3C2_debug_atf_slot_HOST_ONLY.img` (1 MiB)
- `first_2MiB_FROZEN.bin` (2 MiB, contains boot metadata)
- `current_atf_slot_ROLLBACK_FROZEN.img` (1 MiB)
- Debug `bl32.bin` and debug `bl32.elf`; hashes above.

**Suggested repository path for this log:** `docs/reproduce/2026-10-19_F1D3C7_G3C3E.md`. Source trees and scripts should be committed separately with immutable Git revisions; the SPL source has local modifications (`git status: M`) and requires a recorded diff/patch for independently repeatable builds.

## 8. Remaining blockers, risks and next execution

| Item | Status | Next action |
|---|---|---|
| G3C3E runtime `C5XLAT` | **PENDING** | Safely unmount/disconnect the test SD, boot the DE10-Nano with PuTTY logging from power-on; record `C5XLAT:BEGIN`, descriptor records and `C5XLAT:END` |
| Debug boot regression | **PENDING** | Confirm debug ATF `size=20516`, entry `0x3E0048A0`, SHA-256 PASS; then SP_MIN, OP-TEE init, U-Boot prompt |
| SPL runtime provenance | **UNPROVEN** | Separate investigation of primary/secondary boot copies; do not infer chosen copy from hash alone |
| DDR/NS/DMA isolation | **NOT PROVEN** | Future F1D4 firewall/read-denial tests with safe design |
| CPU1/PSCI | **BLOCKED** | Supply real PSCI callbacks and separate testing plan |
| Signed/authenticated boot | **NOT ENABLED/NOT PROVEN** | SHA-256 verification alone is not authentication |
| OP-TEE RNG warning | **OPEN** | Resolve zero-entropy seeding prior to production/security claims |
| Linux, TA, kernel driver, image processing | **NOT TESTED IN THIS SESSION** | Later independent validation |

### Decision at end of day

```text
F1D3C7_G3C3E_APPLY_GATE=PASS
ATF_DEBUG_PAYLOAD_ON_TEST_SD=YES_VERIFIED_BY_FULL_SLOT_READBACK
ROLLBACK_BASELINE_SNAPSHOT=AVAILABLE_OFFLINE
POST_APPLY_HARDWARE_BOOT=NOT_YET_TESTED
C5XLAT_RUNTIME_DESCRIPTOR_READBACK=NOT_YET_TESTED
FULL_SECURE_BOOT_CHAIN=NOT_YET_VERIFIED
NEXT=POWER_ON_TEST_SD_AND_CAPTURE_FULL_UART_THEN_ANALYZE_XLAT
```

**Critical distinction:** Today's completed milestone is **verified deployment of a diagnostic ATF slot to a specifically identified test SD**, not completed MMU hardware validation or fully authenticated secure boot.
