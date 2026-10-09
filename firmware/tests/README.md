# Tiêu chí kiểm thử

Mỗi baseline công bố phạm vi và bộ gate riêng. Build PASS trên CI không thay thế test phần cứng.

| Gate | Bằng chứng cần lưu |
|---|---|
| Source reconstruction | Full commit, patch series, config hash; clean checkout |
| Build | Log và exit code; đủ binary/ELF/map đã khai báo |
| Image packaging | Layout đúng, không overlap, checksum artifact |
| Boot tới U-Boot | UART từ power-on qua các stage tới prompt; không crash trước prompt |
| Boot tới Linux | UART/dmesg, kernel version, endpoint userspace xác định |
| OP-TEE regression | Lệnh và selector, manifest bộ TA, số case/subtest, exit code |
| FPGA programming | Lệnh load hoàn tất, bằng chứng trạng thái cấu hình, test register phù hợp |
| Authenticated boot | Positive/negative test phù hợp threat model; không dùng SHA256 PASS đơn lẻ |
| Driver và app | Các test trong spec khi phần này được đưa vào scope baseline |

Nếu một lần thử boot tới U-Boot rồi lỗi fpga load, ghi cả boot_to_uboot=PASS và fpga_programming=FAIL. Không biến một marker đơn lẻ thành PASS cho toàn bộ phiên.

Đối với automated checker, chỉ nhận diện marker là chưa đủ: cần endpoint mong đợi, phát hiện data abort/panic và timeout. Bản khung chưa có parser hay test firmware thực thi.

