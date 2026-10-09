# Quickstart cho dev mới

**Trạng thái: đang chuẩn bị.** Chưa có baseline tag chứa đầy đủ source, script và manifest trong gói khung này.

Quickstart hoàn thiện phải có đúng một đường đi chính tới mốc đã công bố, theo trình tự:

1. Chọn tag của baseline và clone đúng repo.
2. Chuẩn bị môi trường ở environment.md.
3. Kiểm tra dependency bằng script.
4. Fetch source revision và áp patch.
5. Build toàn bộ thành phần cần cho baseline.
6. Tạo SD image theo layout.
7. Chọn thiết bị SD, flash riêng và xác minh.
8. Kết nối UART, power-on và lưu toàn bộ log.
9. Chạy bộ test và đối chiếu kết quả.

Với mỗi bước phải điền: thư mục chạy, câu lệnh nguyên văn, input, output path, output mong đợi và cách nhận biết thất bại. Ghi các thao tác Windows/WSL/U-Boot/Linux board bằng nhãn môi trường rõ ràng.

Không yêu cầu người mới đọc daily để tìm một lệnh vá còn thiếu. Nếu bước trong troubleshooting là điều kiện để build thành công, fix đó phải có trong source/patch/script của baseline.

## Hai đường sử dụng

- Build từ source: dùng lock manifest và script, phù hợp người muốn phát triển.
- Dùng image đã phát hành: tải asset, kiểm tra checksum rồi flash, phù hợp bring-up nhanh.

Chạy được image phát hành chỉ xác nhận đường sử dụng binary; chưa chứng minh đường build source hoạt động.

Sau khi đã có baseline, điền lệnh clone/checkout với tag thật. Không dùng một tag ví dụ chưa tồn tại.

