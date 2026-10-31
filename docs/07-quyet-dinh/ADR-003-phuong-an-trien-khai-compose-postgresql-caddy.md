# ADR-003 · Phương án chuẩn: Docker Compose + PostgreSQL + Caddy trên một VPS (2026-10-31)

> Danh sách ADR: [README](README.md)

- **Bối cảnh**: chưa có hạ tầng nào. Guacamole cần 3 thành phần (guacd, web app, kho dữ
  liệu xác thực) + HTTPS. Repo hỗ trợ MySQL, PostgreSQL, SQL Server (extension JDBC), LDAP,
  SSO; image chính thức cấu hình hoàn toàn bằng biến môi trường.
- **Quyết định**:
  - **Docker Compose** trên một VPS Ubuntu LTS (không dùng panel Dokploy / Coolify ở giai đoạn này).
  - **PostgreSQL 16** làm DB xác thực: schema + script nâng cấp có đủ trong repo
    (`guacamole-auth-jdbc-postgresql/schema/`), image `postgres` tự chạy SQL init. Bỏ MySQL
    (tương đương tính năng nhưng không có lý do để chọn) và SQL Server (nặng, license).
  - **Caddy 2** làm reverse proxy: tự xin / gia hạn chứng chỉ Let's Encrypt, WebSocket chạy
    sẵn, cấu hình vài dòng. Bỏ Nginx + certbot (nhiều bước tay hơn: header WebSocket, buffering,
    cron gia hạn) và Traefik (label phức tạp, chỉ đáng khi dùng panel).
  - Web app ở gốc tên miền (`WEBAPP_CONTEXT=ROOT`), bật `REMOTE_IP_VALVE` vì extension ban
    bật sẵn.
- **Hệ quả**:
  - Được: một file `compose.yaml` + `Caddyfile` + `.env` mô tả toàn bộ hệ thống; chỉ cổng
    80/443 ra Internet; nâng cấp = đổi tag.
  - Mất: một VPS = một điểm hỏng; session web app trong RAM nên không scale ngang đơn giản.
  - Việc phải làm theo: backup `pg_dump` + đưa ra ngoài VPS + khôi phục thử định kỳ
    ([06/04](../06-van-hanh/04-sao-luu-va-xu-ly-su-co.md)); nâng cấp schema DB bằng tay khi lên
    phiên bản mới ([06/03](../06-van-hanh/03-van-hanh-hang-ngay.md#nâng-cấp-phiên-bản-guacamole)).
  - Đổi sang panel / CDN / VPN sau này → ADR mới + file phương án riêng trong `06-van-hanh/`.
