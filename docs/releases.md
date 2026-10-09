# Đóng gói baseline và release

Một release cần có:

- Tag và full commit của repo điều phối.
- Source lock với revision của từng component, patch/config hash.
- Environment/toolchain manifest và hướng dẫn build.
- Binary/image đúng với run đã kiểm thử; có thể nén/split theo giới hạn dịch vụ hiện hành.
- SHA256SUMS, file size, image layout và ghi chú phiên bản.
- UART log, kết quả test, run_id/build_id; file ELF/map phục vụ debug.
- Known issues, phạm vi hỗ trợ và trạng thái bảo mật.

Code/config/patch/docs lưu trong Git. Thư mục object/cache/output tạm nằm trong out/ và work/. Binary phát hành lớn nên phân phối bằng release asset hoặc nguồn artifact phù hợp; dùng Git LFS nếu muốn quản lý binary theo version trong repo và đã có quy trình tải LFS rõ ràng.

SHA256SUMS xác minh artifact tải về; không tự chứng minh tính xác thực của bản phát hành hoặc thay thế chữ ký. Build có timestamp/key mới có thể khác hash; phạm vi bit-for-bit reproducibility phải được công bố riêng.

Không tạo LICENSE chung áp lên toàn bộ mã bên thứ ba một cách tùy ý. Giữ giấy phép upstream, ghi nguồn và chọn giấy phép cho mã do dự án sở hữu.

Tài liệu chính thức:
- https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
- https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github

