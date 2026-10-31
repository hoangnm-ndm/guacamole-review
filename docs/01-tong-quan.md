# 01 · Tổng quan

> Đọc đầu tiên. Ai đọc: mọi người. Trả lời: Guacamole là gì, 2 repo gồm gì, đang
> ở phiên bản nào, làm việc với code chưa review thế nào cho an toàn.
>
> Trước: [README](README.md) · Tiếp: [02-kien-truc.md](02-kien-truc.md)

## Guacamole là gì

Apache Guacamole là **remote desktop gateway không cần client** (clientless): người
dùng mở trình duyệt, đăng nhập, rồi điều khiển máy từ xa qua **RDP, VNC, SSH,
Telnet hoặc Kubernetes** (exec vào pod) mà không cài phần mềm gì. Trình duyệt chỉ
nói một giao thức text riêng — **giao thức Guacamole** — còn việc dịch sang các
giao thức nhị phân (RDP, VNC…) do daemon `guacd` ở phía server làm.

## Hai phần của hệ thống

| Phần | Repo | Thành phần chính | Chạy ở đâu |
| --- | --- | --- | --- |
| **Server** | `guacamole-server/` | `guacd` (daemon, cổng `4822`), `libguac` (thư viện lõi), plugin `libguac-client-{rdp,vnc,ssh,telnet,kubernetes}`, công cụ `guacenc` / `guaclog` | Tiến trình native trên Linux (hoặc container `guacamole/guacd`) |
| **Client** | `guacamole-client/` | `guacamole.war` (Java servlet + REST API + frontend AngularJS), thư viện `guacamole-common` (Java) / `guacamole-common-js` (JS), API extension `guacamole-ext`, 14 extension (JDBC, LDAP, TOTP, SSO…) | Servlet container (Tomcat 9) hoặc container `guacamole/guacamole` |

Chữ "client" dễ gây nhầm: `guacamole-client` **không** phải app cài trên máy người
dùng — nó là web app chạy trên server; "client" thật là trình duyệt. Chi tiết luồng:
[02-kien-truc.md](02-kien-truc.md).

## Phiên bản trong repo (kiểm tra 2026-10-31)

| Mục | `guacamole-server` | `guacamole-client` |
| --- | --- | --- |
| Nhánh đang checkout | `main` | `main` |
| Commit HEAD | `5f9bb3b4` (2026-10-16) | `31d2a6f6a` (2026-10-14) |
| Phiên bản khai báo | `1.6.1` (`configure.ac:21`) | `1.6.1` (`<revision>` trong `pom.xml:42`) |
| Tag phát hành gần nhất | `1.6.0` (HEAD đi trước 358 commit) | `1.6.0` (HEAD đi trước 379 commit) |
| Image Docker Hub mới nhất có số phiên bản | `guacamole/guacd:1.6.0` | `guacamole/guacamole:1.6.0` |
| Trạng thái working tree | sạch | sạch |

Nghĩa là: code trên `main` là **1.6.1 chưa phát hành** (còn nhánh
`origin/staging/1.6.1`). Docs dev mô tả code `main`; docs vận hành dùng image
**1.6.0** đã phát hành. Những khác biệt liên quan đến vận hành giữa 1.6.0 và `main`
đã đối chiếu bằng `git diff 1.6.0 HEAD` (entrypoint Docker, schema PostgreSQL) và
được ghi trong [06-van-hanh/05-bien-moi-truong.md](06-van-hanh/05-bien-moi-truong.md).

Cập nhật bảng này: `git -C guacamole-server log -1 --format='%h %ad' --date=short`,
`git -C guacamole-server describe --tags` (tương tự cho client).

## An toàn khi làm việc với mã nguồn

Yêu cầu khi viết docs: **chỉ đọc, không thực thi** — code chưa được review kỹ nên
có thể chứa mã độc. Những gì đã làm / chưa làm:

| Việc | Đã làm? | Ghi chú |
| --- | --- | --- |
| Đọc file bằng `cat` / `grep` / `sed` | Có | |
| `git log`, `git diff`, `git tag` | Có | Đã đọc `.git/config` của cả 2 repo trước: không có `core.fsmonitor`, `core.hooksPath` hay hook tự chạy (thư mục `.git/hooks` chỉ có file `*.sample`). |
| `./configure`, `make`, `mvn`, `npm`, `docker build`, chạy script trong `bin/`, `util/`, `guacamole-docker/` | **Không** | Các lệnh này trong docs ghi "chưa kiểm chứng". |

Nếu sau này muốn build, nên biết các bước build **tự chạy code và tải code ngoài**:

| Lệnh | Tự chạy gì | Tải gì từ Internet |
| --- | --- | --- |
| `autoreconf` / `./configure` / `make` | Script m4/shell của autotools, `util/generate-test-runner.pl` (khi `make check`) | Không |
| `docker build` ở `guacamole-server/` | `src/guacd-docker/bin/autobuild.sh` | `git clone` FreeRDP, libssh2, libtelnet, libvncserver, libwebsockets từ GitHub (tag mới nhất khớp regex) |
| `mvn package` ở `guacamole-client/` | Plugin Maven, `doc/licenses/generate-license-files.sh`, `npm ci` + webpack | Dependency Maven Central, Node v24.13.0 + gói npm (`package-lock.json`) |
| `docker build` ở `guacamole-client/` | Mọi thứ của `mvn package` + script `guacamole-docker/build.d/*.sh` + test JS bằng Firefox | Thêm 3 JDBC driver qua `curl` (dev.mysql.com, jdbc.postgresql.org, github.com/microsoft) |

Khuyến nghị: build trong **máy ảo / container dùng một lần**, không mount thư mục
home, không có SSH key / token. Vận hành thì dùng image chính thức đã ghim phiên
bản — xem [ADR-002](07-quyet-dinh/ADR-002-dung-image-chinh-thuc-ghim-phien-ban.md).

## Thuật ngữ

| Thuật ngữ | Nghĩa |
| --- | --- |
| **guacd** | Guacamole proxy daemon — nhận kết nối giao thức Guacamole từ web app, mở kết nối RDP/VNC/SSH… tới máy đích. |
| **Giao thức Guacamole** | Giao thức text dạng `4.size,4.1024,3.768;` giữa trình duyệt ↔ web app ↔ guacd. Xem [03-libguac-va-giao-thuc.md](03-guacamole-server/03-libguac-va-giao-thuc.md). |
| **Tunnel** | Kênh mang giao thức Guacamole giữa trình duyệt và web app: WebSocket (`/websocket-tunnel`) hoặc HTTP long-polling (`/tunnel`). |
| **Connection (kết nối)** | Cấu hình một đích: giao thức + tham số (`hostname`, `port`, `password`…). |
| **Sharing profile** | Cấu hình để chia sẻ một phiên đang chạy cho người khác (xem / cùng điều khiển). |
| **AuthenticationProvider** | Interface Java của extension xác thực (JDBC, LDAP…). Xem [04-extension.md](04-guacamole-client/04-extension.md). |
| **UserContext** | Những gì một user đã đăng nhập nhìn thấy: danh sách connection, user, group… do từng provider cung cấp. |
| **GUACAMOLE_HOME** | Thư mục cấu hình của web app (`guacamole.properties`, `extensions/`, `lib/`). Mặc định `/etc/guacamole`. |
| **Extension** | File `.jar` có `guac-manifest.json`, đặt trong `GUACAMOLE_HOME/extensions/`. |
| **Recording** | File ghi lại phiên (định dạng giao thức Guacamole), xem lại trong web app hoặc chuyển thành video bằng `guacenc`. |
