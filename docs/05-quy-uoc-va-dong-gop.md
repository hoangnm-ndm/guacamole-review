# 05 · Quy ước code và quy trình đóng góp

> Ai đọc: dev sắp sửa code trong 2 repo (để giữ được khả năng `git pull` upstream
> hoặc gửi PR cho Apache). Quy ước dưới đây **rút từ code và file cấu hình thật**,
> mỗi mục ghi nguồn / công cụ ép buộc.
>
> Trước: [04-guacamole-client/05-build-va-test.md](04-guacamole-client/05-build-va-test.md) · Tiếp: [06-van-hanh/](06-van-hanh/README.md)

## Áp dụng cho cả hai repo

| Quy ước | Công cụ ép / nguồn | Ví dụ |
| --- | --- | --- |
| Mọi file nguồn bắt đầu bằng **header Apache License 2.0** ("Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements…") | Client: `apache-rat-plugin` (build dừng nếu thiếu; ngoại lệ `.ratignore`). Server: không có công cụ ép, nhưng mọi file đều có | `guacamole-server/src/guacd/log.c` dòng 1–18 |
| Thụt lề **4 khoảng trắng**, không tab | Quan sát (`src/guacd/daemon.c` không có ký tự tab) | |
| Dòng ≤ 80 cột là mục tiêu, không bắt buộc | Quan sát (vẫn có dòng dài hơn) | |
| **Mọi hàm / lớp / trường public có comment Javadoc-style** (`/** … @param … @return … */`), câu đầy đủ | Quan sát — kể cả code C | `guacamole-server/src/libguac/guacamole/flag.h` |
| Commit nhỏ theo đơn vị logic, message **`GUACAMOLE-<số JIRA>: <câu mô tả>.`** | `CONTRIBUTING` (2 repo giống hệt nhau) | `GUACAMOLE-2118: Ensure socket I/O cannot block indefinitely.` |
| Mỗi thay đổi có issue JIRA, gửi qua pull request GitHub | `CONTRIBUTING` | <https://issues.apache.org/jira/browse/GUACAMOLE/> |

## guacamole-server (C)

| Quy ước | Nguồn | Ghi chú |
| --- | --- | --- |
| Build với `-Werror -Wall -pedantic` → **không được có warning** | `src/*/Makefile.am` (vd. `src/guacd/Makefile.am:63`) | Thư viện ngoài sinh warning deprecation → dùng `CPPFLAGS=-Wno-error=deprecated-declarations` như `Dockerfile`, không sửa cờ trong `Makefile.am` |
| Tên public có tiền tố `guac_` (libguac), `guacd_` (daemon), `guac_<p>_` (plugin), `guac_common_` (common) | Quan sát | `guac_client_load_plugin`, `guacd_create_proc`, `guac_vnc_parse_args` |
| Thành phần "private" của struct public bắt đầu `__` | `src/libguac/guacamole/client.h` (`__users`, `__owner`, `__watchdog`) | Plugin không được đụng vào |
| Kiểu `struct guac_x` khai báo trong `x-types.h`, hàm callback trong `x-fntypes.h`, hằng trong `x-constants.h` | Cấu trúc `src/libguac/guacamole/` | |
| Cấp phát bằng `guac_mem_alloc` / `guac_mem_zalloc` / `guac_mem_free`, thread bằng `guac_thread_create(..., "tên")` | `mem.h`, `thread.h`; commit `GUACAMOLE-2283` chuyển sang API thread tối giản | Tên thread hiện trong log / debugger |
| Test: hàm `test_<suite>__<tên>()`, file trong `tests/<suite>/` | `README-unit-testing.md` | Xem [03/05](03-guacamole-server/05-build-va-test.md#test-cunit) |
| Lỗi trả về: hàm `int` trả `0` = thành công, khác 0 = lỗi; đặt `guac_error` / `guac_error_message` | `error.h` | |

## guacamole-client (Java / JS)

| Quy ước | Nguồn | Ghi chú |
| --- | --- | --- |
| Tương thích **Java 8**, `-Xlint:all`, **`failOnWarning=true`** | `pom.xml` gốc, `maven-compiler-plugin` | Không dùng API Java > 8; `@SuppressWarnings` có lý do |
| `dependencyConvergence`, `requireUpperBoundDeps` | `maven-enforcer-plugin` | Thêm dependency phải hội tụ phiên bản; dependency mới cần cập nhật LICENSE (thư mục `doc/licenses/`) |
| Dependency Injection bằng **Guice** (`@Inject`, module `*Module.java`) | `guacamole/.../GuacamoleServletContextListener.java` | Không `new` service trực tiếp trong lõi |
| REST bằng Jersey (JAX-RS): resource trả object, lỗi ném `GuacamoleException` (map sang HTTP status) | `guacamole/.../rest/` | |
| Property cấu hình: hằng `*GuacamoleProperty` với `getName()` trả tên kebab-case, đặt cạnh nơi dùng (thường `ConfigurationService.java` của extension) | Mọi extension | Tên có tiền tố extension (`totp-…`, `ldap-…`) |
| Extension phụ thuộc `guacamole-ext` scope `provided` | `extensions/*/pom.xml` | |
| Frontend: AngularJS module mỗi thư mục, file đặt theo vai trò (`controllers/`, `directives/`, `services/`, `types/`, `templates/`) | `guacamole/src/main/frontend/src/app/` | Chữ hiển thị qua khóa dịch, không viết cứng |
| Chữ giao diện mới → thêm khóa vào `translations/en.json` (bản chuẩn) | | Ngôn ngữ khác có thể thiếu khóa (rơi về en — suy luận) |

## Sửa code trong bản clone này

Hai repo là clone upstream, docs ở ngoài (xem [ADR-001](07-quyet-dinh/ADR-001-dat-docs-o-thu-muc-goc.md)).
Muốn thử sửa mà vẫn cập nhật được upstream:

1. Tạo nhánh riêng trong repo con: `git -C guacamole-client switch -c thu-nghiem/<ten>`.
2. Commit theo định dạng `GUACAMOLE-<số>: …` nếu định gửi upstream; nếu chỉ học thì
   tiền tố tùy ý nhưng **không commit lên `main`** (giữ `main` khớp `origin/main`).
3. Cập nhật upstream: `git -C guacamole-client fetch origin && git -C guacamole-client switch main && git -C guacamole-client merge --ff-only origin/main`
   (`--ff-only` để báo lỗi thay vì tạo merge commit nếu `main` đã bị sửa).
4. Trước khi `git pull` repo lạ: đọc `.git/config` không có `core.fsmonitor`, `core.hooksPath`
   (xem [01 § An toàn](01-tong-quan.md#an-toàn-khi-làm-việc-với-mã-nguồn)).

## Cập nhật docs khi đổi code

| Đổi | Cập nhật |
| --- | --- |
| Thêm / đổi tham số giao thức | [03/04](03-guacamole-server/04-plugin-giao-thuc.md) |
| Thêm property / biến Docker | [06/05 Biến môi trường](06-van-hanh/05-bien-moi-truong.md) (nguồn duy nhất của bảng biến) |
| Thêm extension | [04/04](04-guacamole-client/04-extension.md) bảng extension |
| Pull upstream lên phiên bản mới | Bảng phiên bản trong [01](01-tong-quan.md#phiên-bản-trong-repo-kiểm-tra-2026-10-31) (kèm ngày kiểm tra) |
| Quyết định kiến trúc / triển khai mới | ADR mới trong [07-quyet-dinh/](07-quyet-dinh/README.md) |
