# ADR-001 · Đặt docs ở thư mục gốc, ngoài 2 repo upstream (2026-10-31)

> Danh sách ADR: [README](README.md)

- **Bối cảnh**: `/Users/phecode/HOCTAP/guacamole/` chứa 2 bản clone upstream Apache
  (`guacamole-client/`, `guacamole-server/`, nhánh `main`, working tree sạch). Docs phải
  bao cả hai phần (kiến trúc chung, luồng đi qua cả 2 repo) và vận hành. Thư mục gốc chưa
  phải git repo.
- **Quyết định**: đặt bộ docs ở `docs/` của thư mục gốc; `git init` thư mục gốc với
  `.gitignore` bỏ qua `/guacamole-client/` và `/guacamole-server/`, chỉ track `docs/`,
  `.gitignore`, `CLAUDE.md`. Bỏ phương án "docs trong từng repo" vì làm lệch upstream,
  `git pull` dễ conflict, phần kiến trúc chung phải tách / chép hai nơi.
- **Hệ quả**:
  - Được: 2 repo con giữ nguyên upstream; một nguồn docs duy nhất; có lịch sử commit docs.
  - Mất: docs không đi kèm khi clone riêng từng repo; repo gốc không ghi phiên bản 2 repo
    con → bảng phiên bản trong [01-tong-quan](../01-tong-quan.md#phiên-bản-trong-repo-kiểm-tra-2026-10-31)
    phải cập nhật tay sau mỗi lần pull (ghi ngày kiểm tra).
  - Đường dẫn trong docs viết tương đối so với `guacamole-client/` / `guacamole-server/`
    (mỗi file ghi rõ gốc tính đường dẫn ở đầu file).
