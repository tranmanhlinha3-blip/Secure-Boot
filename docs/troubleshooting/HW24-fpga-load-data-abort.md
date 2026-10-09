# HW24 fpga load gây data abort

**Trạng thái: OPEN. Chưa có fix đã được xác nhận.**

## Triệu chứng

Boot qua SPL, SP_MIN và OP-TEE tới U-Boot. Đọc RBF từ SD vào RAM thành công; fpga load gây data abort rồi reset.

## Lệnh đã chạy

Các LBA và kích thước dưới đây chỉ áp dụng cho thẻ/image trong log lịch sử:

```text
mmc dev 0
mmc read 0x20000000 0x76f000 0x3576
fpga load 0 0x20000000 0x6aebe4
```

## Hai lần thử được ghi nhận

| Trường | Log gửi 10:19 | Log gửi 10:27 |
|---|---|---|
| TF-A build banner | 02:52:09 Oct 9 2026 | 03:23:01 Oct 9 2026 |
| Relocated PC | 0x01029b8e | 0x01002b8c |
| R3 | 0xffc25000 | 0xff800000 |
| Kết quả | data abort/reset | RSTUdata abort/reset |

Sau reboot, cả hai log có md.l ff706000 4 trả về 00000050 00000200 00000000 00000000. Đây là bằng chứng đọc thanh ghi sau reboot, chưa chứng minh FPGA cấu hình thành công.

## Phân tích hiện tại

Audit source HW23A đánh dấu vấn đề RMW/readback GPV. Marker HW24 chứng minh chương trình tới vị trí in marker; chưa chứng minh đầy đủ quyền truy cập trên toàn bộ đường nạp.

Chưa có ELF/disassembly và artifact manifest tương ứng trong gói khung để chốt nguyên nhân tại từng PC. Không ghi “đã sửa” chỉ dựa vào việc fault chuyển vị trí.

## Cần bổ sung

- Binary hash, source commit và ELF/map cho từng lần thử.
- Diff thực tế giữa hai bản.
- Đối chiếu PC với đúng binary rồi lưu kết luận.
- Retest fpga load thành công và trạng thái FPGA hợp lệ trên cùng bộ image.

Nguồn: [daily 09/10/2026](../daily/2026/2026-10-09.md), các log gốc Pasted text(4).txt và Pasted text(5).txt. Cần nhập log gốc vào evidence/ khi hoàn thiện repo.

