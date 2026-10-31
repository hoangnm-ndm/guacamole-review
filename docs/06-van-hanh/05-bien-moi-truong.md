# 06/05 · Biến môi trường (nguồn duy nhất)

> Ai đọc: người vận hành khi điền `.env` / `compose.yaml`, khi đổi cấu hình, khi
> gỡ lỗi "đổi biến mà không ăn". **Đây là nơi duy nhất liệt kê đầy đủ biến**; file
> khác chỉ dẫn link về đây.
>
> Trước: [04-sao-luu-va-xu-ly-su-co.md](04-sao-luu-va-xu-ly-su-co.md) · Quay lại: [README](README.md)

Áp dụng cho image **`guacamole/guacamole:1.6.0`** và **`guacamole/guacd:1.6.0`**
(đối chiếu `git diff 1.6.0 HEAD` ngày 2026-10-31: cơ chế biến giống `main`, chỉ khác
phiên bản JDBC driver đóng gói và `main` tự thêm `enable-file-environment-properties`
— ở 1.6.0 cờ này bật qua biến `ENABLE_FILE_ENVIRONMENT_PROPERTIES=true` có sẵn trong `Dockerfile`).

## Cơ chế — đọc trước khi sửa biến

1. **Mọi property của `guacamole.properties` đều đặt được bằng biến môi trường.**
   Tên biến = tên property viết hoa, `-` → `_` (`guacd-hostname` → `GUACD_HOSTNAME`,
   `totp-issuer` → `TOTP_ISSUER`). Lý do: entrypoint ghi `enable-environment-properties: true`
   vào `guacamole.properties` sinh ra lúc khởi động. Code: [04/02 § Thuộc tính → biến môi trường](../04-guacamole-client/02-webapp-rest-va-tunnel.md#thuộc-tính--biến-môi-trường).
2. **Extension bật theo tiền tố.** Có ít nhất một biến bắt đầu bằng tiền tố (vd. `POSTGRESQL_`)
   → extension tương ứng được link vào `GUACAMOLE_HOME/extensions/`. Bật / tắt tường minh:
   `<TIỀN_TỐ>ENABLED=true|false` (vd. `TOTP_ENABLED=true`, `BAN_ENABLED=false`).
   Code: `guacamole-docker/entrypoint.d/700-configure-features.sh`.
3. **Secret qua file**: mọi biến property có thể thay bằng `<BIẾN>_FILE=/run/secrets/<tên>`
   (đọc nội dung file). Biến thường thắng `_FILE` nếu đặt cả hai **ở tầng property**;
   README image ghi ngược lại cho biến DB ("secret takes precedence") — **chưa kiểm
   chứng**, đừng đặt cả hai.
4. **Ưu tiên**: giá trị trong `guacamole.properties` (kể cả file bạn mount vào làm template)
   **thắng** biến môi trường. Đã ghi key trong file thì đổi biến không có tác dụng.
5. **Mọi biến là runtime** — không có biến build-time. Nhưng **`docker compose restart`
   không áp dụng biến mới**; phải `docker compose up -d` (tạo lại container).
6. **Bỏ trống = không khai báo?** Docker Compose truyền biến rỗng `FOO=` vào container
   như một biến có giá trị rỗng. Với cơ chế tiền tố (mục 2), **biến rỗng vẫn bật
   extension** (script chỉ kiểm tra tên biến tồn tại — suy từ `is_feature_enabled`).
   → Trong `compose.yaml` chỉ liệt kê biến thật sự dùng; đừng để sẵn `LDAP_HOSTNAME=` trống.

## File `.env` trên VPS (`/opt/guacamole/.env`)

Compose đọc file này để thay `${…}` trong `compose.yaml`. **Không commit, không chép vào
docs.** Quyền: `chmod 600`.

| Biến | Bắt buộc | Ý nghĩa | Bỏ trống thì | Ví dụ / nơi lấy |
| --- | --- | --- | --- | --- |
| `GUAC_DOMAIN` | có | Tên miền truy cập Guacamole; Caddy xin chứng chỉ cho tên này | Caddy không có site → không phục vụ HTTPS | `remote.example.com` (bản ghi A trỏ IP VPS) |
| `GUACAMOLE_VERSION` | không | Tag image `guacamole/guacamole` và `guacamole/guacd` (luôn cùng giá trị) | Dùng mặc định trong `compose.yaml` (`1.6.0`) | Xem trang releases |
| `POSTGRES_PASSWORD` | có | Mật khẩu user DB `guacamole_user`, dùng cho cả container `postgres` và `guacamole` | `postgres` không khởi tạo được; `guacamole` báo lỗi kết nối DB | Sinh ngẫu nhiên: `openssl rand -base64 32 \| tr -d '/+='` |
| `GUACD_LOG_LEVEL_VALUE` | không | Mức log guacd (`trace`/`debug`/`info`/`warning`/`error`) | `info` | Bật `debug` khi gỡ lỗi kết nối |
| `GUACAMOLE_LOG_LEVEL` | không | Mức log web app | `info` | `debug` khi gỡ lỗi đăng nhập / extension |

Tên `GUACD_LOG_LEVEL_VALUE` / `GUACAMOLE_LOG_LEVEL` là biến **của file compose** (không
truyền thẳng vào container) để tránh trùng tên `LOG_LEVEL` giữa 2 container.

## Container `guacamole` (web app)

### Nhóm lõi

| Biến | Property | Mặc định | Đặt trong phương án chuẩn | Ý nghĩa / sai thì sao |
| --- | --- | --- | --- | --- |
| `GUACD_HOSTNAME` | `guacd-hostname` | `localhost` | `guacd` (tên service compose) | Sai / thiếu → mọi kết nối lỗi ngay khi mở (web app không tới được guacd) |
| `GUACD_PORT` | `guacd-port` | `4822` | (không đặt) | |
| `GUACD_SSL` | `guacd-ssl` | `false` | (không đặt) | Chỉ `true` khi guacd chạy với `-C/-K` |
| `WEBAPP_CONTEXT` | — (entrypoint `500-…`) | `guacamole` → URL `/guacamole/` | `ROOT` → URL `/` | Đổi xong phải sửa đường dẫn ở proxy nếu có |
| `LOG_LEVEL` | `log-level` | `info` | `${GUACAMOLE_LOG_LEVEL:-info}` | |
| `API_SESSION_TIMEOUT` | `api-session-timeout` | `60` (phút) | (không đặt) | Phiên đăng nhập không hoạt động bị hủy |
| `EXTENSION_PRIORITY` | `extension-priority` | theo tên file | (không đặt) | Vd. `postgresql, totp, *` |
| `SKIP_IF_UNAVAILABLE` | `skip-if-unavailable` | — | (không đặt) | Vd. `ldap` để LDAP sập vẫn đăng nhập bằng DB |
| `GUACAMOLE_HOME` | — | `/etc/guacamole` (`Dockerfile`) | (không đặt) | Thư mục **template**, xem [mục dưới](#guacamole_home-tùy-biến-trong-docker) |

### Nhóm PostgreSQL (extension `guacamole-auth-jdbc-postgresql`, tiền tố `POSTGRESQL_`)

| Biến | Bắt buộc | Đặt trong phương án chuẩn | Ghi chú |
| --- | --- | --- | --- |
| `POSTGRESQL_HOSTNAME` | có | `postgres` | |
| `POSTGRESQL_PORT` | không | (không đặt, mặc định 5432) | |
| `POSTGRESQL_DATABASE` | có | `guacamole_db` | Phải trùng `POSTGRES_DB` của container `postgres` |
| `POSTGRESQL_USERNAME` | có | `guacamole_user` | Tên cũ `POSTGRESQL_USER` / `POSTGRES_USER` vẫn chạy nhưng in cảnh báo deprecated |
| `POSTGRESQL_PASSWORD` | có | `${POSTGRES_PASSWORD}` | |
| `POSTGRESQL_SSL_MODE` | không | (không đặt) | Cần khi DB ở máy khác |
| `POSTGRESQL_AUTO_CREATE_ACCOUNTS` | không | (không đặt) | `true` khi kết hợp LDAP / SSO: tự tạo user DB lúc đăng nhập lần đầu |
| `POSTGRESQL_USER_PASSWORD_MIN_LENGTH`, `…_REQUIRE_DIGIT`, `…_REQUIRE_SYMBOL`, `…_REQUIRE_MULTIPLE_CASE`, `…_PROHIBIT_USERNAME`, `…_MAX_AGE`, `…_HISTORY_SIZE` | không | Khuyến nghị đặt `MIN_LENGTH=12` | Chính sách mật khẩu user Guacamole |
| `POSTGRESQL_DEFAULT_MAX_CONNECTIONS`, `…_PER_USER`, `POSTGRESQL_ABSOLUTE_MAX_CONNECTIONS` | không | (không đặt) | Giới hạn phiên đồng thời |

Danh sách đầy đủ: `grep -o 'return "postgresql-[a-z-]*"' -r guacamole-client/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-postgresql/src`.

### Nhóm chặn brute-force (extension `guacamole-auth-ban`, tiền tố `BAN_`)

| Biến | Mặc định | Ghi chú |
| --- | --- | --- |
| `BAN_ENABLED` | `true` (đặt sẵn trong `Dockerfile`) | Muốn tắt: `false` |
| `BAN_MAX_INVALID_ATTEMPTS` | `5` | |
| `BAN_ADDRESS_DURATION` | `300` (giây) | |
| `BAN_MAX_ADDRESSES` | `10485760` | Số IP theo dõi tối đa |

**Bắt buộc đi kèm `REMOTE_IP_VALVE_ENABLED=true` khi có reverse proxy** — nếu không,
mọi người dùng có chung IP của Caddy → 5 lần sai của bất kỳ ai khóa **tất cả** 5 phút.

### Nhóm reverse proxy (Tomcat `RemoteIpValve`, tiền tố `REMOTE_IP_VALVE_`)

| Biến | Đặt trong phương án chuẩn | Ghi chú |
| --- | --- | --- |
| `REMOTE_IP_VALVE_ENABLED` | `true` | Tomcat lấy IP thật từ `X-Forwarded-For` |
| `REMOTE_IP_VALVE_INTERNAL_PROXIES` | (không đặt) | Regex IP proxy được tin. Mặc định của Tomcat 9 gồm dải private (10/8, 172.16/12, 192.168/16, 127/8…) — đủ cho Caddy trong cùng Docker network (theo tài liệu Tomcat, chưa kiểm chứng) |
| `REMOTE_IP_VALVE_REMOTE_IP_HEADER`, `…_PROTOCOL_HEADER`, `…_PROXIES_HEADER`, `…_TRUSTED_PROXIES`, `…_PROTOCOL_HEADER_HTTPS_VALUE`, `…_HTTP_SERVER_PORT`, `…_HTTPS_SERVER_PORT` | (không đặt) | Map 1-1 sang thuộc tính valve (`guacamole-docker/environment/REMOTE_IP_VALVE_/configure.sh`); không đặt thì log `Using default RemoteIpValve value for "…"` — **bình thường** |

### Nhóm recording (extension `guacamole-history-recording-storage`, tiền tố `RECORDING_`)

| Biến | Đặt khi bật recording | Ghi chú |
| --- | --- | --- |
| `RECORDING_SEARCH_PATH` | `/var/lib/guacamole/recordings` | Trùng mặc định; đặt để bật extension. Phải là cùng thư mục guacd ghi (mount chung). |

### Extension khác (bật khi cần)

| Tiền tố | Extension | Property tối thiểu | Tham khảo |
| --- | --- | --- | --- |
| `TOTP_` | 2FA TOTP | `TOTP_ENABLED=true` (+ `TOTP_ISSUER`) | [04/04](../04-guacamole-client/04-extension.md#extension-có-sẵn-extensions) |
| `LDAP_` | LDAP / AD | `LDAP_HOSTNAME`, `LDAP_USER_BASE_DN`, … | manual chương LDAP |
| `OPENID_` / `SAML_` / `CAS_` / `SSL_AUTH_` | SSO | xem manual | |
| `DUO_` | Duo 2FA | `DUO_API_HOSTNAME`, `DUO_CLIENT_ID`, `DUO_CLIENT_SECRET`, `DUO_REDIRECT_URI` | |
| `HTTP_AUTH_` | Header SSO | `HTTP_AUTH_HEADER` | **Chỉ dùng khi web app không truy cập được trực tiếp** |
| `QUICKCONNECT_` | Ô nhập URI tạo kết nối tức thì | **Không bật** trên hệ thống nhiều người dùng. Nếu bật, bắt buộc `QUICKCONNECT_ALLOWED_PARAMETERS=hostname,port,username` | Code `QCParser.java`: không đặt danh sách → **cho phép mọi tham số** (kể cả `drive-path`, `recording-path`) |
| `JSON_`, `RESTRICT_`, `KSM_`, `DISPLAY_STATISTICS_` | | | |
| `RADIUS_` | Không có trong image chính thức (cần build `-P lgpl-extensions`) | | |

### Tên cũ (deprecated) — entrypoint tự đổi và in cảnh báo

| Tên cũ | Tên mới |
| --- | --- |
| `POSTGRES_*` (mọi biến tiền tố này trong container guacamole) | `POSTGRESQL_*` |
| `POSTGRESQL_USER`, `POSTGRES_USER` | `POSTGRESQL_USERNAME` |
| `MYSQL_USER` / `SQLSERVER_USER` | `MYSQL_USERNAME` / `SQLSERVER_USERNAME` |
| `PROXY_ALLOWED_IPS_REGEX`, `PROXY_IP_HEADER`, `PROXY_PROTOCOL_HEADER` | `REMOTE_IP_VALVE_INTERNAL_PROXIES`, `…_REMOTE_IP_HEADER`, `…_PROTOCOL_HEADER` |
| `LOGBACK_LEVEL` | `LOG_LEVEL` |
| Docker link `--link guacd:guacd`, `postgres`, `mysql`, `sqlserver` | `GUACD_HOSTNAME`/`GUACD_PORT`, `POSTGRESQL_HOSTNAME`/`_PORT`… |

Nguồn: `guacamole-docker/entrypoint.d/000-migrate-docker-links.sh`, `010-migrate-legacy-variables.sh`.
README image (`guacamole-docker/README.md`) còn dùng tên cũ `POSTGRESQL_USER` — làm theo bảng này.

## Container `guacd`

| Biến | Mặc định | Đặt trong phương án chuẩn | Ghi chú |
| --- | --- | --- | --- |
| `LOG_LEVEL` | `info` | `${GUACD_LOG_LEVEL_VALUE:-info}` | Tên cũ `GUACD_LOG_LEVEL` còn chạy, in cảnh báo |

Không còn biến nào khác: entrypoint chỉ chạy `guacd -f -b 0.0.0.0 -L "$LOG_LEVEL" "$@"`.

## Container `postgres` (image chính thức `postgres:16`)

| Biến | Đặt | Ghi chú |
| --- | --- | --- |
| `POSTGRES_DB` | `guacamole_db` | Chỉ có tác dụng **lần khởi tạo đầu** (thư mục data rỗng) |
| `POSTGRES_USER` | `guacamole_user` | Như trên; user này là superuser của cluster riêng này |
| `POSTGRES_PASSWORD` | `${POSTGRES_PASSWORD}` | Như trên — **đổi biến sau đó không đổi mật khẩu DB**, xem [03 § Đổi mật khẩu DB](03-van-hanh-hang-ngay.md#đổi-mật-khẩu-db) |

## Container `caddy` (image `caddy:2`)

| Biến | Đặt | Ghi chú |
| --- | --- | --- |
| `GUAC_DOMAIN` | `${GUAC_DOMAIN}` | Caddyfile dùng `{$GUAC_DOMAIN}` |

## Entrypoint của image guacamole làm gì

Chạy lần lượt `guacamole-docker/entrypoint.d/*.sh` mỗi lần container khởi động:

| Script | Làm gì |
| --- | --- |
| `000-migrate-docker-links.sh` | Đổi biến Docker link cũ sang `GUACD_HOSTNAME`… |
| `010-migrate-legacy-variables.sh` | Đổi tên biến cũ (bảng trên) |
| `100-generate-guacamole-home.sh` | Tạo `GUACAMOLE_HOME` **tạm** `/tmp/guacamole-home.XXXX`: link `lib/`, `extensions/`, file khác từ template `$GUACAMOLE_HOME` (mặc định `/etc/guacamole`), nối nội dung `guacamole.properties` template, thêm `enable-environment-properties: true` (và `enable-file-environment-properties: true` trên `main`) nếu chưa có |
| `500-generate-tomcat-catalina-base.sh` | Tạo `CATALINA_BASE` tạm, ghi `guacamole.home=…` vào `catalina.properties`, link war thành `webapps/${WEBAPP_CONTEXT:-guacamole}.war` |
| `700-configure-features.sh` | Với mỗi tiền tố trong `/opt/guacamole/environment/` đang bật (mục Cơ chế 2): chạy `configure.sh` (chỉ `REMOTE_IP_VALVE_` có, sửa `server.xml` bằng xmlstarlet), link jar extension / driver vào `GUACAMOLE_HOME` |
| `999-start-tomcat.sh` | `exec catalina.sh run` |

Hệ quả: container **không giữ trạng thái** ở filesystem — mọi dữ liệu ở PostgreSQL và
thư mục recording mount ngoài.

## GUACAMOLE_HOME tùy biến trong Docker

Khi cần file (extension tự viết, branding, `logback.xml`, `protocols/*.json`) thay vì biến:

1. Trên VPS tạo `/opt/guacamole/guacamole-home/` với cấu trúc `extensions/`, `lib/`, (tùy) `guacamole.properties`.
2. Mount vào container: `./guacamole-home:/etc/guacamole:ro` (đúng đường dẫn `GUACAMOLE_HOME` mặc định).
3. `docker compose up -d guacamole`. Kiểm tra log có `Extension "…" loaded.`

Nhớ mục Cơ chế 4: key ghi trong `guacamole.properties` template sẽ **thắng** biến môi trường.
