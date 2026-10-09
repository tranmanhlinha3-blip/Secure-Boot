# Patch series

Tách patch theo component: u-boot/, tf-a/, optee-os/, optee-client/, optee-test/, linux/ và rootfs/ khi có thay đổi thực.

Mỗi component có base full commit và file series xác định thứ tự áp patch. Mỗi patch giải thích mục đích và trỏ tới issue nếu sửa lỗi. Source sau patch phải được tái tạo từ một checkout sạch.

Đối với dữ liệu hiện tại, cần thu cả dirty diff, staged changes, file untracked và mọi thao tác vá binary. Nếu vẫn cần vá binary để boot, bước đó phải có script, offset, input/output hash và phiên bản phù hợp; không được bỏ bước ấy khỏi reproduce. Ưu tiên chuyển thay đổi đã xác minh về source khi có thể.

Chưa có patch firmware được nhập vào gói khung.

