# Kế hoạch tổ chức repository

## Quyết định cấu trúc

Giữ ba phần chính người dùng đề xuất. Phần build phải chứa tất cả đầu vào cần để tạo lại firmware; README dẫn dev theo quy trình hiện tại đã kiểm chứng. Troubleshooting giữ quá trình phân tích lỗi. Daily giữ lịch sử các lần thử.

| Vị trí | Nội dung cần quản lý |
|---|---|
| README.md | Phạm vi, trạng thái, baseline khuyên dùng, lối vào quickstart |
| firmware/manifests/ | URL source, full commit, thứ tự patch, toolchain, cấu hình |
| firmware/patches/ | Mọi thay đổi ngoài upstream, tách theo component |
| firmware/configs/ | SPL/U-Boot, TF-A, OP-TEE, kernel và rootfs config |
| firmware/board/de10-nano/ | Handoff, DTS, memory map, SD layout, boot environment |
| firmware/scripts/ | Kiểm tra môi trường, fetch, patch, build, pack, flash, verify |
| firmware/tests/ | Đầu vào test, điều kiện PASS, parser log và script kiểm thử thực tế |
| docs/troubleshooting/ | Bản ghi lỗi và fix có bằng chứng |
| docs/daily/ | Nhật ký theo ngày và run_id |
| evidence/ | Log UART và kết quả nhỏ, được chọn lọc, gắn manifest |
| work/ và out/ | Source tải về và đầu ra build, nằm ngoài Git |
| GitHub Releases | Binary/image/log đầy đủ đi cùng tag, manifest và checksum |

## Cách quản lý source được đề xuất

Với repo điều phối hiện tại, dùng **upstream URL + full commit + patch series**. Script fetch tải đúng revision vào work/, script patch áp dụng theo thứ tự được khai báo. Cách này cần được kiểm chứng bằng clean build trước khi gọi là baseline.

Các source mới do dự án tự viết, ví dụ driver hoặc TA riêng, có thể commit trực tiếp dưới firmware/components/. Chỉ tạo các component khi có source thực; không ghi là đã hoàn thành chỉ vì có folder.

Nếu đã duy trì fork riêng cho U-Boot/TF-A/OP-TEE, submodule trỏ tới commit đã push cũng phù hợp. Chọn một cơ chế có thẩm quyền cho mỗi component; tránh vừa fork sửa source vừa áp patch chứa cùng thay đổi.

Full commit, patch và config phải đủ để khôi phục trạng thái dirty trên máy hiện tại. Ghi lại cả file untracked cần cho build; git diff đơn thuần không chứa các file này. Không dùng tên thư mục HWxx, banner phiên bản hoặc tên branch làm identity duy nhất.

## Tiêu chí dev có thể build theo

1. Dev lấy repo/tag bằng tài khoản thông thường và tải được mọi dependency.
2. Môi trường sạch không cần thư mục, cache, binary hay toolchain riêng trên máy tác giả.
3. Script báo thiếu dependency rõ ràng và không bỏ qua lỗi.
4. Build tạo các output đã khai báo, kèm manifest và checksum.
5. Quy trình tạo thẻ xác định rõ layout, image nào ghi ở đâu và cách xác minh.
6. Boot được kiểm tra tới một mốc cụ thể: U-Boot, Linux hay regression; mỗi mốc có log cùng run_id.
7. Mọi bước sửa tay cần thiết đều được chuyển thành patch/config/script.
8. Một người khác chạy thành công từ đầu bằng chính tài liệu trong repo.

Nếu chỉ mới chứng minh chức năng tương đương, gọi là build có thể lặp lại về chức năng. Chỉ tuyên bố binary giống từng byte sau khi đã kiểm soát timestamp, toolchain, đường dẫn, key và thử đối chiếu thực tế. SHA256SUMS của release dùng để nhận diện đúng artifact phát hành.

## Thứ tự nhập dự án hiện tại

1. Chọn một bộ image và source đã boot được tới mốc muốn bàn giao; giữ các thử nghiệm FPGA chưa thành công ở nhánh riêng.
2. Thu commit, dirty diff, file untracked, config, toolchain, handoff và layout thẻ.
3. Đóng gói source thành revision cố định và patch hoặc fork đã push.
4. Chuyển các lệnh đã chạy thành script dùng đường dẫn tương đối từ root repo.
5. Điền quickstart bằng các lệnh có thật và output đã đo.
6. Clean build rồi boot trên đúng board; lưu UART và manifest.
7. Đặt tag cho baseline đã đạt gate và tạo release.
8. Nhập lịch sử lỗi và daily, trỏ về các commit/run_id tương ứng.

Không cần chờ toàn bộ camera/driver/pipeline hoàn thành mới tổ chức repo. Có thể bàn giao một baseline chỉ boot tới U-Boot, miễn là phạm vi đó được ghi đúng.

## Tài liệu tham khảo

- GitHub Releases: https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases
- Quản lý file lớn: https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github
- Git submodules: https://git-scm.com/docs/gitsubmodules

Các đường dẫn và tên script trong khung là cấu trúc đề xuất. Chúng trở thành hướng dẫn chạy thật sau khi source/script được nhập và kiểm thử.

