# 03 · guacamole-server (C)

> Mục lục phần server. Ai đọc: dev muốn hiểu / sửa `guacd`, `libguac` hoặc plugin
> giao thức. Cần đọc trước [02-kien-truc.md](../02-kien-truc.md).
>
> Trước: [02-kien-truc.md](../02-kien-truc.md) · Tiếp: [04-guacamole-client/](../04-guacamole-client/README.md)

Repo: `guacamole-server/` (upstream `github.com/apache/guacamole-server`, nhánh
`main`, phiên bản `1.6.1` chưa phát hành — xem [01-tong-quan](../01-tong-quan.md#phiên-bản-trong-repo-kiểm-tra-2026-10-31)).

| #   | File                                                         | Nội dung                                                                  |
| --- | ------------------------------------------------------------ | ------------------------------------------------------------------------- |
| 01  | [01-cau-truc-ma-nguon.md](01-cau-truc-ma-nguon.md)           | Bản đồ thư mục, thư viện nào phụ thuộc thư viện nào                       |
| 02  | [02-guacd.md](02-guacd.md)                                   | Daemon: khởi động, cấu hình, mô hình process mỗi kết nối, log, TLS        |
| 03  | [03-libguac-va-giao-thuc.md](03-libguac-va-giao-thuc.md)     | Thư viện lõi, định dạng lệnh, handshake, danh sách lệnh                   |
| 04  | [04-plugin-giao-thuc.md](04-plugin-giao-thuc.md)             | Plugin RDP / VNC / SSH / Telnet / Kubernetes, cách thêm tham số / giao thức |
| 05  | [05-build-va-test.md](05-build-va-test.md)                   | Dependency, `configure`, build, CUnit, Docker image, `guacenc` / `guaclog` |

Thứ tự đọc gợi ý: 01 → 03 (hiểu giao thức trước) → 02 → 04 → 05. Người chỉ cần
build: đọc 05.
