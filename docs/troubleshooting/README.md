# Lỗi và cách xử lý

Mỗi lỗi có mã ổn định và một bản ghi theo [mẫu](TEMPLATE.md). GitHub Issue dùng để thảo luận; file Markdown giữ kết luận có thể đọc cùng source ở một commit cụ thể.

| Mã | Triệu chứng | Trạng thái | Hồ sơ |
|---|---|---|---|
| HW24 | fpga load gây data abort và reset | OPEN; chưa có fix đã xác nhận | [HW24](HW24-fpga-load-data-abort.md) |

Chỉ ghi FIXED sau khi có commit sửa, log retest trên đúng artifact và test hồi quy cần thiết. Phân biệt giả thuyết, phép thử, workaround, nguyên nhân đã xác nhận và fix cuối cùng.

Một lỗi có thể có nhiều attempt; không tạo nhiều “fix thành công” chỉ vì fault đổi địa chỉ. Daily trỏ về hồ sơ lỗi, còn hồ sơ lỗi trỏ về commit/run_id.

