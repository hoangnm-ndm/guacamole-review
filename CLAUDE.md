# CLAUDE.md — workspace học Apache Guacamole

- Thư mục gốc chứa 2 repo upstream độc lập: `guacamole-server/` (C, autotools) và
  `guacamole-client/` (Java/Maven + AngularJS). Repo git gốc **chỉ track `docs/`**,
  `.gitignore`, `CLAUDE.md` (ADR-001). Không commit vào 2 repo con; giữ `main` của chúng
  khớp `origin/main`.
- **Code 2 repo chưa review → chỉ đọc, không thực thi**: không chạy `configure`, `make`,
  `mvn`, `npm`, `docker build`, script trong `bin/`, `util/`, `guacamole-docker/`. Trước
  lệnh git trong repo con, kiểm tra `.git/config` không có `core.fsmonitor` / `hooksPath`.
- Docs: `docs/` — mục lục `docs/README.md`. Bảng biến môi trường chỉ đầy đủ ở
  `docs/06-van-hanh/05-bien-moi-truong.md`. Docs dev mô tả code `main` (1.6.1 chưa phát
  hành); docs vận hành mô tả image chính thức `1.6.0` (ADR-002).
- Kiểm tra link docs: `python3 ~/.claude/skills/docs-du-an/scripts/check_md_links.py docs`.
