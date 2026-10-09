# Giao diện script cần triển khai

Các tên bên dưới là thiết kế đề xuất; chưa có file thực thi trong gói khung. Không copy các tên này vào quickstart như lệnh đã chạy thành công.

| Script dự kiến | Trách nhiệm | Điều kiện hoàn tất |
|---|---|---|
| check_env.sh | Kiểm tra OS, tool, version, dung lượng và quyền cần thiết | Không còn dependency thiếu |
| fetch_sources.sh | Lấy đúng source revision theo manifest | Revision khớp và không có thay đổi lạ |
| apply_patches.sh | Áp patch series theo thứ tự | Thất bại khi patch không áp được |
| build_all.sh | Build từng component theo phụ thuộc thực tế | Exit code đúng; đủ output mong đợi |
| pack_image.sh | Tạo SD image từ layout đã khóa | Kiểm tra size, offset, overlap và checksum |
| flash_sd.sh | Ghi image vào thiết bị được chỉ định rõ | Hiển thị đúng thiết bị và xác minh readback |
| verify_boot.py | Đối chiếu UART với baseline và mốc kết thúc | Đủ mốc, không abort/panic trước endpoint |
| collect_evidence.sh | Gom log, manifest, checksum và test result | Mọi file thuộc đúng build_id/run_id |

Build không tự flash. Flash là thao tác riêng vì ghi đè thiết bị lưu trữ. Script không tự chọn ổ đĩa dựa trên tên cố định.

Mọi script lấy root từ vị trí repo và chấp nhận output directory. Không phụ thuộc /home/manhlinh hoặc thư mục HWxx cũ. Không bỏ qua lỗi; ghi log có tên build_id.

build_all.sh cần hỗ trợ clean build hoặc hướng dẫn tạo workspace mới, đồng thời ghi toolchain version, source revision, patch hash và cấu hình thực tế vào manifest.

