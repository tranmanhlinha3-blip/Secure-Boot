# Phạm vi bảo mật và trạng thái thực tế

Ghi rõ những gì baseline bảo vệ, root of trust ở đâu, stage nào xác minh stage nào và attacker có thể sửa thành phần nào.

Trong log hiện có:
- SPL in SHA256 PASS cho ba image, đồng thời in Authentication: NOT ENABLED YET.
- OP-TEE có cảnh báo RNG seed bằng zero và thiếu ASLR seed.
- HW24 đã đi qua marker GPV_POLICY_WRITES_ISSUED nhưng vẫn lỗi fpga load.

Vì vậy tài liệu release phải tách boot-chain bring-up, kiểm tra toàn vẹn bằng hash và authenticated secure boot. Chỉ đánh dấu hoàn tất cơ chế bảo mật sau khi có thiết kế và test tương ứng.

Theo dõi tài liệu kiến trúc thực tế theo source đang chạy. Log ghi TF-A SP_MIN v2.13.0; spec cũ mô tả BL31 v2.9. Cần rà soát khác biệt trước khi dùng spec làm hướng dẫn triển khai.

Private signing key, token và secret thực không đưa lên repo public. Với bản demo, ghi rõ cách tạo key thử nghiệm, vị trí public verification key và ảnh hưởng của key mới tới binary/test. Không dùng key production làm dependency của build công khai.

