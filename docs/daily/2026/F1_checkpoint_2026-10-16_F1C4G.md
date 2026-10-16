# Secure Boot — F1 checkpoint, 2026-10-16

## Scope
F1 — Firmware SMC Completion, paused after F1C4G source closure audit. **Not a firmware release**. Preserve known-good HW88 / R35K_3I boot artifacts and current SD image.

## Gate history and evidence

| Gate | Status | Supporting evidence |
|---|---|---|
| F1A source inventory | PASS — evidence collected | Four SMC names/source files found; functional SMC completion NOT YET. F1A_review_bundle.zip, SHA256 `51c0ad8b5a9a943044412b53dada7361317d3c47583a41daeb4ca3b9ad75b733`. |
| F1B host SMC regression | PASS — host only | Baseline 11/11, PMU diagnostic candidate 14/14; `GET_HWUID` and `LOCK_REGION` still fail-closed stubs. |
| F1C isolated candidate | PASS — ARM32 object | Separate candidate `~/secure_imgproc/F1C_candidate_20261016_131009`, baseline source unchanged. |
| F1C2 full TF-A SP_MIN BL32 build | PASS — candidate only | `bl32.bin` 16,408 bytes, SHA256 `18ca68760225a635909300893d96e63f3db4bbc6e7907e23c28a32913a6e76f1`; ARM ELF entry `0x3e002f60`; no board test. |
| F1C3 HW feasibility | AUDIT PASS — backend NOT PROVEN | UID and SDRAM lock not yet demonstrated. |
| F1C4B/C/D RBF identity | PASS | `/mnt/d/SecureBoot_share/HW95O_compressed.rbf`, SHA256 `e41251ea21d4c76f2adcdbed88f60dc7d2ec12fb3f84410de0c32a4becdd48b8`, CRC32 `87D23A06`. |
| F1C4E/F source recovery | FOUND | QSF-referenced RTL/QIP/SDC files recovered in `/mnt/d/internFPGA/HPS_FPGA`. |
| F1C4G QSF source closure | PASS — candidate, NOT provenance | Four QSF references FOUND, source evidence bundle 12 files; `F1C4G_QUARTUS_RBF_PROVENANCE=NOT_PROVEN`. |

## F1C4G exact checkpoint

```text
F1C4G_MODE=READ_ONLY
F1C4G_HW95O_RBF_IDENTITY=PASS
F1C4G_REQUIRED=HPS_FPGA_Top.v STATUS=FOUND
F1C4G_REQUIRED=soc_system/synthesis/soc_system.qip STATUS=FOUND
F1C4G_REQUIRED=S16_board.sdc STATUS=FOUND
F1C4G_REQUIRED=phase4_image_rtl/P4D4F_led_output.sdc STATUS=FOUND
F1C4G_BASELINE_QSF_REFERENCES=4
F1C4G_BASELINE_QSF_REFS_FOUND=4
F1C4G_SOURCE_CLOSURE_CANDIDATE=PASS
F1C4G_QUARTUS_RBF_PROVENANCE=NOT_PROVEN
F1C4G_SOURCE_FILES_BUNDLED=12
F1C4G_REVIEW_BUNDLE=/mnt/d/SecureBoot_share/F1C4G_REVIEW_20261016_163142/F1C4G_review_bundle.zip
F1C4G_TFA_PATCH=NO
F1C4G_FPGA_PATCH=NO
F1C4G_SD_WRITE=NO
F1C4G_BOARD=NOT_RUN
F1C4G_RESULT=SOURCE_REVIEW_READY
```

## Open requirements (DO NOT mark PASS)

- `GET_HWUID`: true device-specific UID backend, hardware provenance, security/trust model, FPGA configuration timing, SMC ABI/runtime tests.
- `LOCK_REGION`: actual SDRAM memory isolation and safe rule implementation, secure DRAM layout and recovery method, hardware negative-access test.
- `PERF_COUNTER`: ARMv7 PMCR/PMCNTENSET/PMCCNTR runtime validation; avoid claiming live counters from host-only results.
- Full four-SMC QEMU/appropriate host tests, on-board regression, source/artifact freeze and rollback.
- F2 OP-TEE TA and later stages are intentionally not started for acceptance.

## Resume next session

1. Verify `F1C4G_review_bundle.zip`, console log, HW95O RBF and F1C2 BL32 artifact exist and have stored SHA256 checksums.
2. Upload `F1C4G_review_bundle.zip` to analyze actual QSF/Qsys/RTL/SDC. Resolve exact source provenance; a source-closure candidate is not proof that a revision generated an RBF.
3. Continue F1C5 HWUID architecture and F1C6 SDRAM `LOCK_REGION` feasibility in separate candidates; perform no protection register writes to the live board before approving a tested memory map and rollback.
4. Qualify PMU, QEMU/host functional tests, board regression and freeze F1 before moving to F2.

Do not overwrite SD or known-good firmware. Leaving WSL does not delete files on the Linux filesystem or D: drive.
