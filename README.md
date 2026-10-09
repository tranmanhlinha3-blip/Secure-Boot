# Secure Boot trên DE10 Nano

Repo dành cho việc build lại firmware từ source, theo dõi lỗi và lưu bằng chứng kiểm thử theo ngày.

**Trạng thái gói khung:** đã có cấu trúc tài liệu và mẫu manifest. Chưa nhập source firmware, cấu hình build và script build/deploy từ máy phát triển; vì vậy gói này chưa build được firmware. Không đánh dấu bản này là release đã kiểm chứng.

## Ba phần chính

| Phần | Lối vào | Mục đích |
|---|---|---|
| Build và boot | [Hướng dẫn build](firmware/README.md) | Source có khóa revision, patch, config, script, image layout và kiểm thử |
| Lỗi và cách xử lý | [Troubleshooting](docs/troubleshooting/README.md) | Triệu chứng, cách tái hiện, nguyên nhân, fix và bằng chứng retest |
| Reproduce daily | [Daily](docs/daily/README.md) | Mỗi ngày và mỗi lần chạy gắn với đúng source và bộ image |

## Bắt đầu

1. Xem [trạng thái thực tế](docs/status.md) và chọn baseline đã có đủ source/manifest/test.
2. Chuẩn bị [phần cứng](docs/hardware.md) và [môi trường](docs/environment.md).
3. Làm theo [quickstart](docs/quickstart.md) sau khi baseline đã được hoàn thiện.
4. Đối chiếu kết quả với [tiêu chí kiểm thử](firmware/tests/README.md).

Mục tiêu cho dev mới là clone một tag xác định, build bằng script, tạo image, boot board và kiểm tra kết quả mà không cần đọc toàn bộ nhật ký.

## Phạm vi đã có bằng chứng

Các log ngày 09/10/2026 cho thấy boot tới U-Boot qua TF-A SP_MIN và OP-TEE; một phiên regression OP-TEE có 104 case và 27.002 subtest PASS. Hai lượt HW24 vẫn lỗi khi nạp FPGA. Chưa có manifest nối phiên xtest với đúng binary HW24. Chi tiết nằm trong [daily ngày 09/10](docs/daily/2026/2026-10-09.md).

Tên dự án không tự chứng minh authenticated secure boot đã hoàn tất. Xem [phạm vi bảo mật](docs/security-model.md).

## Dành cho người đóng gói repo

- [Kế hoạch tổ chức và nhập source](docs/repository-plan.md)
- [Source manifest mẫu](firmware/manifests/sources.lock.example.json)
- [Run manifest mẫu](firmware/manifests/run.example.json)
- [Quy ước scripts](firmware/scripts/README.md)
- [Hồ sơ release](docs/releases.md)

