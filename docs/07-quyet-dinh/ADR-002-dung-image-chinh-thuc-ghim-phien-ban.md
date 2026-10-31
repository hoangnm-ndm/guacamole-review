# ADR-002 · Vận hành bằng image Docker chính thức, ghim phiên bản (2026-10-31)

> Danh sách ADR: [README](README.md)

- **Bối cảnh**:
  - Code trong 2 repo clone **chưa được review**; yêu cầu: chỉ đọc, không thực thi vì có
    thể chứa mã độc.
  - Build từ source tự chạy rất nhiều mã: script autotools / Maven / npm, `git clone` 5 thư
    viện C từ GitHub theo "tag mới nhất khớp regex" (`src/guacd-docker/bin/autobuild.sh`),
    `curl` tải 3 JDBC driver, cài Firefox từ PPA (xem [01 § An toàn](../01-tong-quan.md#an-toàn-khi-làm-việc-với-mã-nguồn)).
    Build còn không tái lập 100% (tag thư viện ngoài thay đổi theo thời gian).
  - `main` là `1.6.1` chưa phát hành; Docker Hub (kiểm tra 2026-10-31) có `guacamole/guacamole`
    và `guacamole/guacd` tag mới nhất có số là `1.6.0` (đẩy 2025-06-23), ngoài ra `latest`.
- **Quyết định**: production dùng `guacamole/guacamole:1.6.0` và `guacamole/guacd:1.6.0`
  từ Docker Hub, **ghim tag cụ thể** (biến `GUACAMOLE_VERSION` trong `.env`), không dùng
  `latest`. Hai image luôn cùng tag. Bỏ phương án "tự build từ source sau review" cho giai
  đoạn này (tốn công review, build nặng, chạy mã ngoài).
- **Hệ quả**:
  - Được: không chạy mã chưa review từ bản clone; image do dự án Apache phát hành; nâng
    cấp có chủ đích ([06/03 § Nâng cấp](../06-van-hanh/03-van-hanh-hang-ngay.md#nâng-cấp-phiên-bản-guacamole)).
  - Mất: không có các thay đổi trên `main` sau 1.6.0 (vd. guacd re-exec sau fork, JDBC driver
    mới hơn); extension RADIUS không có trong image.
  - Docs dev mô tả code `main`; docs vận hành mô tả image 1.6.0 — khác biệt ảnh hưởng vận hành
    đã đối chiếu bằng `git diff 1.6.0 HEAD` và ghi trong [06/05](../06-van-hanh/05-bien-moi-truong.md).
  - Vẫn phải tin nhà phát hành image (Docker Hub, tài khoản `guacamole`); muốn chặt hơn thì
    ghim theo digest (`image: guacamole/guacd@sha256:…`) — chưa làm.
  - Khi `1.6.1` phát hành: thêm ADR mới hoặc cập nhật tag theo quy trình nâng cấp.
