# 03 — Vấn đề và rủi ro

> Commit: `guacamole-client@31d2a6f6a`, `guacamole-server@5f9bb3b4`.
> Ký hiệu đường dẫn giống `02-KIEN-TRUC-HE-THONG.md`:
>
> | Ký hiệu | Đường dẫn |
> |---|---|
> | `S/` | `guacamole-server/src/` |
> | `C/` | `guacamole-common/…/guacamole/` |
> | `E/` | `guacamole-ext/…/guacamole/` |
> | `W/` | `guacamole/src/main/java/…/guacamole/` |
> | `CJ/` | `guacamole-common-js/…/modules/` |
> | `FE/` | `guacamole/src/main/frontend/src/app/` |
> | `J/` | `auth-jdbc-base/…/auth/jdbc/` |
> | `X/` | `guacamole-client/extensions/` |
> | `MX/` | `auth-jdbc-mysql/src/main/resources/…/auth/jdbc/` |
>
> Cách đọc:
> - **Độ chắc chắn "Đã xác nhận"** = thấy trực tiếp trong code. Không có lỗi nào được chạy thử hay viết PoC, vì skill chỉ đọc tĩnh.
> - **"Rủi ro triển khai"** = hành vi có chủ đích của dự án. Nó chỉ thành lỗ hổng khi cấu hình hoặc hạ tầng triển khai sai.
> - Các phát hiện bộ nhớ trong C (`SEC-01`, `SEC-02`, `SEC-03`, `SEC-07`, `VAL-01`) có thể là lỗ hổng thật trong phần mềm phổ biến. Xem cách công bố có trách nhiệm (responsible disclosure) ở `04-BAO-MAT.md` §7.

## Bảng tổng hợp

| Mã | Nhãn | Mức độ | Nhóm | Vị trí chính | Vấn đề | Độ chắc chắn |
|---|---|---|---|---|---|---|
| SEC-01 | [NGUYÊN TẮC] | High | Bảo mật | `S/libguac/user-handlers.c:166-176, 488-506, 838-847` | Handler lệnh sau handshake không kiểm `argc`, đọc/ghi qua con trỏ cũ | Đã xác nhận (thiếu kiểm tra); khai thác: Rủi ro tiềm ẩn |
| SEC-02 | [NGUYÊN TẮC] | High | Bảo mật | `S/libguac/unicode.c:37-67`, `S/libguac/protocol.c:72-78` | guacd và Java đếm độ dài UTF-8 lỗi khác nhau, có thể chèn instruction | Rủi ro tiềm ẩn |
| SEC-03 | [NGUYÊN TẮC] | High | Bảo mật | `S/protocols/rdp/fs.c:729-737, 561-592` | RDPDR không kiểm FileId đang mở: double-close, use-after-free | Đã xác nhận |
| SEC-04 | [NGUYÊN TẮC] | High | Bảo mật | `J/security/SHA256PasswordEncryptionService.java:40-47` | Hash mật khẩu JDBC là một vòng SHA-256 | Đã xác nhận |
| SEC-05 | [NGUYÊN TẮC] | High | Bảo mật | `X/guacamole-auth-jdbc/modules/*/schema/002-create-admin-user.sql` | Admin mặc định `guacadmin`, mật khẩu công khai, không bắt đổi | Đã xác nhận (rủi ro triển khai) |
| SEC-06 | [NGUYÊN TẮC] | High | Bảo mật | `X/guacamole-auth-quickconnect/.../QCParser.java:314-326` | Quickconnect mặc định cho mọi tham số và mọi host (drive-path, recording-path…) | Đã xác nhận (rủi ro triển khai) |
| SEC-07 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/protocols/rdp/channels/rdpdr/rdpdr-fs-messages-dir-info.c:40-74`, `S/protocols/rdp/unicode.c:31-44` | Đọc vượt stack khi liệt kê thư mục, off-by-one ghi NUL | Đã xác nhận |
| SEC-08 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/guacd/connection.c:301-367`, `S/guacd-docker/bin/entrypoint.sh`, `E/environment/LocalEnvironment.java:85`, `C/net/SSLGuacamoleSocket.java:87-102` | guacd không xác thực; Docker bind 0.0.0.0; web app ↔ guacd plaintext mặc định; TLS không kiểm hostname | Đã xác nhận (rủi ro triển khai) |
| SEC-09 | [NGUYÊN TẮC] | Medium | Bảo mật | `FE/client/types/ManagedClient.js:341`, `FE/rest/services/tunnelService.js:218,304`, `W/rest/auth/TokenRESTService.java:209` | Auth token nằm trong URL, bị log DEBUG | Đã xác nhận |
| SEC-10 | [NGUYÊN TẮC] | Medium | Bảo mật | `guacamole/src/main/webapp/WEB-INF/web.xml`, `FE/auth/service/authenticationService.js:146` | Không có CSP / X-Frame-Options / nosniff; token trong localStorage | Đã xác nhận |
| SEC-11 | [NGUYÊN TẮC] | Medium | Bảo mật | `FE/auth/service/authenticationService.js:294-310` | Tham số URL ghi đè token đã lưu (đổi phiên, login CSRF) | Rủi ro tiềm ẩn |
| SEC-12 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/guacd/daemon.c:641-715`, `S/libguac/parser.c:221` | guacd không giới hạn kết nối, timeout tính theo từng lần read (slowloris) | Đã xác nhận |
| SEC-13 | [NGUYÊN TẮC] | Medium | Bảo mật | `X/guacamole-auth-totp/.../UserVerificationService.java:363-391` | TOTP fail-open khi không lưu được key; enroll kiểu TOFU | Đã xác nhận |
| SEC-14 | [NGUYÊN TẮC] | Medium | Bảo mật | `schema/001-create-schema.sql:230` (`parameter_value`) | Mật khẩu máy đích, TOTP secret lưu plaintext trong DB | Đã xác nhận |
| SEC-15 | [NGUYÊN TẮC] | Medium | Bảo mật | `X/guacamole-auth-json/.../CryptoService.java:70-79,142`, `UserData.java:374-383` | JSON auth: IV 0, cùng key cho AES và HMAC, `expires` tùy chọn, trusted-networks rỗng = mọi IP | Đã xác nhận |
| SEC-16 | [NGUYÊN TẮC] | Medium | Bảo mật | `X/guacamole-auth-sso/.../cas/ticket/TicketValidationService.java:141`, `.../openid/...:126`, `X/guacamole-auth-ldap/.../LDAPConnectionService.java:249-253` | CAS chấp nhận mọi proxy; OIDC implicit flow + claim `email`; LDAP referral hạ cấp mã hoá | Đã xác nhận |
| SEC-17 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/common-ssh/key.c:194-218`, `S/protocols/rdp/settings.c:838-843`, `S/protocols/kubernetes/settings.c:338` | Mặc định yếu khi xác minh danh tính máy đích (SSH không có host key vẫn chạy, RDP `security=any`, K8s không TLS) | Đã xác nhận (rủi ro cấu hình) |
| SEC-18 | [NGUYÊN TẮC] | Medium | Bảo mật | `X/guacamole-auth-header/.../AuthenticationProviderService.java:67-75` | Header auth tin `REMOTE_USER` vô điều kiện | Đã xác nhận (rủi ro triển khai) |
| SEC-19 | [NGUYÊN TẮC] | Medium | Bảo mật | `J/base/ModeledPermissions.java:231-233` | Group từ IdP khớp group DB theo tên, kế thừa quyền DB | Đã xác nhận (rủi ro triển khai) |
| SEC-20 | [NGUYÊN TẮC] | Medium | Bảo mật | `W/rest/user/UserResource.java:177-191`, `X/guacamole-auth-ban/.../InMemoryAuthenticationFailureTracker.java:140` | Chống brute-force yếu: chỉ theo IP, trong RAM; endpoint đổi mật khẩu thành oracle không bị đếm | Đã xác nhận |
| SEC-21 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/protocols/rdp/print-job.c:330-366` | Ghostscript chạy PostScript từ máy đích, kế thừa mọi fd, dùng `exit` sau `fork` | Rủi ro tiềm ẩn |
| SEC-22 | [NGUYÊN TẮC] | Medium | Bảo mật | `S/protocols/rdp/settings.c:1077-1079`, `S/protocols/rdp/fs.c:137-170` | `enable-drive` mà thiếu `drive-path` thì map vào `/` | Đã xác nhận (rủi ro cấu hình) |
| SEC-23 | [NGUYÊN TẮC] | Low | Bảo mật | `W/auth/file/FileAuthenticationProvider.java:106`, `W/auth/file/Authorization.java:183-218` | SAX không tắt DOCTYPE (XXE); user-mapping chỉ hỗ trợ MD5/SHA-256 không salt | Đã xác nhận |
| SEC-24 | [NGUYÊN TẮC] | Low | Bảo mật | `S/guacd/proc-title.c:243-277`, `S/guacd/connection.c:386`, `W/rest/APIError.java:184` | Lộ thông tin: process title, connection ID ở log INFO, message lỗi thô | Đã xác nhận |
| SEC-25 | [NGUYÊN TẮC] | Low | Bảo mật | `S/guacd/daemon.c:605`, `S/libguac/client.c:676-685`, init.d | guacd không tự drop privilege; pidfile không `O_NOFOLLOW`; tên protocol không whitelist | Đã xác nhận |
| SEC-26 | [NGUYÊN TẮC] | Low | Bảo mật | `W/tunnel/websocket/RestrictedGuacamoleWebSocketTunnelEndpoint.java:80-91`, `W/rest/auth/HashTokenSessionMap.java:150` | WebSocket không kiểm Origin; session không có hạn tuyệt đối; re-auth không rotate token | Rủi ro tiềm ẩn |
| SEC-27 | [NGUYÊN TẮC] | Low | Bảo mật | `S/terminal/typescript.c:72-78`, `S/libguac/file.c`, `S/protocols/rdp/fs.c` | Token `${GUAC_USERNAME}` trong path; `.timing` không `O_EXCL`; theo symlink | Rủi ro tiềm ẩn |
| SEC-28 | [NGUYÊN TẮC] | Low | Bảo mật | `S/terminal/terminal-handlers.c:1317-1360`, `S/common-ssh/sftp.c:525,566` | Output từ máy đích (OSC) đẩy được file xuống trình duyệt, không đi qua `sftp-root-directory` | Đã xác nhận (theo thiết kế) |
| SEC-29 | [NGUYÊN TẮC] | Low | Bảo mật | `FE/index/controllers/indexController.js:295-305`, `FE/client/directives/guacClient.js:623-625` | Clipboard local tự đồng bộ tới mọi connection đang mở | Rủi ro tiềm ẩn |
| SEC-30 | [NGUYÊN TẮC] | Medium | Bảo mật | `X/guacamole-auth-radius/.../RadiusConnectionService.java:216-217` | RADIUS không có Message-Authenticator (Blast-RADIUS) | Chưa đủ dữ liệu |
| MAL-01 | [NGUYÊN TẮC] | Medium | Chuỗi cung ứng | `S/guacd-docker/bin/autobuild.sh`, `guacamole-docker/build.d/020-download-drivers.sh` | Build Docker tải mã và JAR không kiểm checksum / không ghim commit | Đã xác nhận |
| ARCH-01 | [NGUYÊN TẮC] | Medium | Kiến trúc | `S/protocols/*/settings.c`, `*/argv.c` | Lặp code parse settings (recording/SFTP/WOL) ở 5 plugin; hai stack render | Đã xác nhận |
| ARCH-02 | [NGUYÊN TẮC] | Medium | Kiến trúc | `X/guacamole-auth-jdbc/modules/*/src/main/resources/**/*.xml` | Mapper/schema lặp ở 3 DB (giống 74–86%) | Đã xác nhận |
| ARCH-03 | [NGUYÊN TẮC] | Medium | Kiến trúc | `W/rest/auth/HashTokenSessionMap.java:55`, `J/tunnel/RestrictedGuacamoleTunnelService.java:63-87` | Trạng thái quan trọng nằm trong RAM từng JVM, không HA được | Đã xác nhận |
| ARCH-04 | [NGUYÊN TẮC] | Low | Kiến trúc | `W/rest/directory/DirectoryResource.java:549-801`, `S/terminal/terminal.c`, `S/protocols/rdp/settings.c:1546-2023` | File/hàm lớn, nhánh trùng nhau | Đã xác nhận |
| ARCH-05 | [NGUYÊN TẮC] | Low | Kiến trúc | `S/guacd/proc.h:33`, `S/libguac/client-watchdog.c:293-300` | Watchdog 15 s hard-code cho mọi thao tác plugin | Rủi ro tiềm ẩn |
| BE-01 | [NGUYÊN TẮC] | Medium | Kiến trúc | `W/tunnel/TunnelRequestService.java:352-364`, `C/protocol/ConfiguredGuacamoleSocket.java:203-318` | Tunnel bị veto không được đóng; socket rò khi handshake lỗi | Đã xác nhận |
| DB-01 | [NGUYÊN TẮC] | Medium | Database | `MySQLEnvironment.java:106`, `PostgreSQLEnvironment.java:110` | SSL tới DB mặc định opportunistic, không verify cert | Đã xác nhận |
| DB-02 | [NGUYÊN TẮC] | Low | Database | `X/.../mysql/schema/001-create-schema.sql` | MySQL `utf8` (utf8mb3); khác độ dài kiểu giữa các DB | Đã xác nhận |
| DB-03 | [NGUYÊN TẮC] | Low | Database | `*RecordMapper.xml` | History mồ côi `end_date IS NULL`, không có retention | Rủi ro tiềm ẩn |
| DB-04 | [NGUYÊN TẮC] | Low | Database | `J/tunnel/AbstractGuacamoleTunnelService.java:703` | Transaction mở trong suốt handshake với guacd | Rủi ro tiềm ẩn |
| DB-05 | [NGUYÊN TẮC] | Low | Database | `.../ConnectionRecordMapper.xml:119-129`, `002-create-admin-user.sql:52` | Subquery dựa vào cột của query ngoài; JOIN seed sai alias; `password_salt` cho phép NULL | Đã xác nhận |
| VAL-01 | [NGUYÊN TẮC] | Medium | Validation | `S/libguac/user-handlers.c:724-725`, `S/protocols/rdp/resolution.c:31` | DPI = 0 gây chia cho 0, sập process (kể cả từ user read-only) | Đã xác nhận |
| VAL-02 | [NGUYÊN TẮC] | Low | Validation | `W/rest/directory/DirectoryResource.java:657-664` | PATCH `replace` bỏ qua `filterExternalObject` | Rủi ro tiềm ẩn |
| VAL-03 | [NGUYÊN TẮC] | Low | Validation | `W/rest/permission/PermissionSetResource.java:196-301` | Patch quyền sai định dạng trả 500; áp từng phần không atomic | Đã xác nhận |
| ERR-01 | [NGUYÊN TẮC] | Medium | Xử lý lỗi | `S/protocols/ssh/ssh.c:419-436`, `S/common-ssh/sftp.c:381,569` | SFTP khởi tạo lỗi vẫn gán handler, dẫn tới deref NULL | Đã xác nhận |
| ERR-02 | [NGUYÊN TẮC] | Medium | Xử lý lỗi | `S/protocols/ssh/ssh.c:211-213, 496` | Ghi SSH non-blocking bỏ qua EAGAIN / ghi thiếu | Đã xác nhận |
| ERR-03 | [NGUYÊN TẮC] | Medium | Xử lý lỗi | `CJ/Display.js:850-853, 928-931` | Decode ảnh lỗi không `catch`, pipeline vẽ treo vĩnh viễn | Đã xác nhận |
| ERR-04 | [NGUYÊN TẮC] | Low | Xử lý lỗi | `S/terminal/terminal-handlers.c:758-770`, `S/pulse/pulse.c:83-94`, `S/libguac/socket.c:109-111` | Lỗi biên trong C: tràn CSI dẫn tới abort, pulse peek NULL, `SSL_write` trả 0 gây lặp | Rủi ro tiềm ẩn |
| ERR-05 | [NGUYÊN TẮC] | Low | Xử lý lỗi | `CJ/Tunnel.js:1220-1221`, `W/rest/auth/AuthenticationService.java:153…`, `C/websocket/GuacamoleWebSocketTunnelEndpoint.java:345-356` | `onerror` sai kiểu; bắt cả `Error`; `releaseWriter` không chạy khi có RuntimeException | Đã xác nhận |
| ERR-06 | [NGUYÊN TẮC] | Low | Xử lý lỗi | `S/guacd/connection.c:294`, `S/libguac/user-handshake.c:340-353`, `S/guacd/conf-file.c:136-156` | Thiếu kiểm NULL; gửi `ready` trước khi kiểm `argc`; dòng config > 8192 byte bị bỏ qua im lặng | Đã xác nhận |
| ERR-07 | [NGUYÊN TẮC] | Medium | Nghiệp vụ | `X/guacamole-auth-restrict/.../calendar/DailyRestriction.java:130-133` | Khung giờ qua nửa đêm kiểm sai ngày (`plus(1)` thay vì `minus(1)`) | Đã xác nhận |
| ERR-08 | [NGUYÊN TẮC] | Low | Nghiệp vụ | `J/JDBCAuthenticationProviderService.java:108-145` | User bị `disabled` trong DB vẫn đăng nhập qua provider khác | Đã xác nhận |
| ERR-09 | [NGUYÊN TẮC] | Low | Nghiệp vụ | `E/token/TokenFilter.java:49` | Regex modifier tham lam: `${A:LOWER}-${B}` bị thay sai | Đã xác nhận |
| ERR-10 | [NGUYÊN TẮC] | Low | Nghiệp vụ | `S/protocols/rdp/channels/rdpsnd/rdpsnd-messages.c:300`, `S/protocols/kubernetes/url.c:72` | Kiểm chỉ số format sai; URL-escape sai với ký tự ngoài ASCII | Đã xác nhận |
| CODE-01 | [NGUYÊN TẮC] | Medium | Clean code | `S/libguac/socket-broadcast.c:259-312`, `S/protocols/rdp/fs.c`, `W/GuacamoleSession.java:51-72` | Data race / lock không nhất quán (hệ thống, ≥ 3 nơi) | Rủi ro tiềm ẩn |
| CODE-02 | [NGUYÊN TẮC] | Low | Clean code | `S/protocols/ssh/ssh_agent.c`, `S/libguac/guacamole/parser-constants.h:37` | Dead code không biên dịch được; hằng định nghĩa nhưng không dùng | Đã xác nhận |
| CODE-03 | [NGUYÊN TẮC] | Low | Clean code | `S/libguac/parser.c:80-81`, `S/libguac/user-handlers.c:87-101` | Signed integer overflow (UB) khi parse số | Đã xác nhận |
| CODE-04 | [NGUYÊN TẮC] | Low | Clean code | `S/protocols/rdp/settings.c:1553-1555` (13 chỗ) | Bản sao mật khẩu bị rò trong heap; không xóa secret khỏi bộ nhớ | Đã xác nhận |
| CODE-05 | [NGUYÊN TẮC] | Low | Clean code | `CJ/Client.js:1059,1837-1839`, `FE/client/types/ManagedClient.js:568-571` | Tra opcode/key trên object có prototype | Đã xác nhận |
| CODE-06 | [NGUYÊN TẮC] | Low | Clean code | `E/token/StandardTokens.java`, `W/tunnel/StandardTokenMap.java` | Hằng token định nghĩa trùng ở hai module | Đã xác nhận |
| PERF-01 | [NGUYÊN TẮC] | Low | Clean code | `C/websocket/GuacamoleWebSocketTunnelEndpoint.java:230-309`, `CJ/Parser.js:150-239`, `S/guacd/proc-map.c:149` | Thread raw mỗi tunnel; parser JS không giới hạn; proc map cấp sẵn 131072 bucket | Đã xác nhận |
| NAME-01 | [NGUYÊN TẮC] | Low | Đặt tên | `S/guacd/proc.c:563-583, 656` | `child_socket`/`parent_socket` đặt ngược vai | Đã xác nhận |
| NAME-02 | [CHUẨN] | Low | Đặt tên | `CJ/*.js` | JS trộn snake_case/camelCase, ES5/ES6 | Đã xác nhận |
| NAME-03 | [NGUYÊN TẮC] | Low | Đặt tên | `S/common-ssh/key.c:194`, `E/net/event/TunnelConnectEvent.java:31-35` | Comment/Javadoc mô tả sai hành vi thật | Đã xác nhận |
| TEST-01 | [NGUYÊN TẮC] | High | Kiểm thử | `guacamole-client/**/src/test` | Java: 19 file test cho khoảng 111k dòng; JDBC, LDAP, REST, auth không có test | Đã xác nhận |
| TEST-02 | [NGUYÊN TẮC] | High | Kiểm thử | `guacamole/src/main/frontend`, `guacamole-common-js/src/test` | Frontend 0 test; common-js chỉ 3 spec | Đã xác nhận |
| TEST-03 | [NGUYÊN TẮC] | Medium | Kiểm thử | `S/guacd`, `S/libguac/tests/parser`, `S/protocols/rdp/channels` | C: guacd 0 test; parser chỉ test ca đúng; không fuzz, không sanitizer | Đã xác nhận |
| TEST-04 | [NGUYÊN TẮC] | Medium | Kiểm thử | `guacamole-server/.github/workflows/pr-build.yml` | CI server không chạy `make check`; không đo coverage | Đã xác nhận |
| OPS-01 | [NGUYÊN TẮC] | Medium | Vận hành | `S/../Dockerfile:27,182`, `guacamole-common-js/pom.xml:222`, `package.json` | Runtime/dependency hết hỗ trợ (EOL): Alpine 3.18, OpenSSL 1.1, Node 16, AngularJS 1.8, Java 8 target, jetty 8/tomcat 7 | Đã xác nhận (một phần EOL theo hiểu biết, xem ghi chú) |
| OPS-02 | [CHUẨN] | Medium | Vận hành | `.github/workflows/pr-build.yml`, `pom.xml:591-661` | Không lint/SAST; dependency-check chỉ opt-in; action không ghim SHA, không khai báo `permissions` | Đã xác nhận |
| OPS-03 | [NGUYÊN TẮC] | Medium | Vận hành | `W/rest/permission/PermissionSetResource.java:180-303` | Thay đổi quyền/membership không phát event, hổng audit log | Đã xác nhận |
| OPS-04 | [CHUẨN] | Low | Vận hành | `guacamole/src/main/frontend/webpack.config.js`, `guacamole/pom.xml:150-157` | WAR chứa source map và source chưa bundle | Đã xác nhận |
| OPS-05 | [NGUYÊN TẮC] | Low | Vận hành | `S/guacd/daemon.c:520-521, 746-750`, `S/protocols/rdp/client.c:178-197` | Chỉ bind địa chỉ đầu tiên; proc map không free; phụ thuộc môi trường ngầm (HOME, `gs` trong PATH) | Đã xác nhận |
| DOC-01 | [NGUYÊN TẮC] | Low | Tài liệu | README hai repo | Ranh giới tin cậy (guacd, GUACAMOLE_HOME, header auth, sftp-root) không nêu trong repo | Chưa đủ dữ liệu (manual nằm ở repo khác) |
| DOC-02 | [CHUẨN] | Low | Tài liệu | `README`, `X/guacamole-auth-json/.../UserData.java:90` | README dạng text cũ; còn link domain cũ `guac-dev.org` | Đã xác nhận |

**Thống kê:**
- Critical 0 · High 8 · Medium 34 · Low 34. Tổng 76 vấn đề, trong đó `[NGUYÊN TẮC]` 72 và `[CHUẨN]` 4.
- 1 vấn đề "Chưa đủ dữ liệu" (`SEC-30`) không bị trừ điểm. `DOC-01` vẫn bị trừ như Low, vì việc repo thiếu tài liệu ranh giới tin cậy là đã xác nhận.

---

## Chi tiết — High

### SEC-01 — Handler lệnh sau handshake không kiểm `argc` `[NGUYÊN TẮC]` · High · Đã xác nhận (khai thác: Rủi ro tiềm ẩn)

**Bằng chứng:**
- `S/libguac/user-handlers.c:838-847`: dispatcher gọi handler mà không kiểm số tham số.
  ```c
  if (strcmp(opcode, current->opcode) == 0)
      return current->handler(user, argc, argv);
  ```
- `:166-176` (`touch`) đọc `argv[0..6]`. `:488-506` (`blob`) chạy `guac_protocol_decode_base64(argv[1])`, tức là **ghi tại chỗ**. `ack`, `mouse`, `key`, `size`… cũng vậy.
- Riêng `usbconnect`/`usbdata`/`usbdisconnect` thì có kiểm (`:633, 662, 689`).
- `S/libguac/parser.c:30-41`:
  - `guac_parser_reset` không xoá `__elementv[128]`.
  - `guac_parser_alloc` dùng `guac_mem_alloc` không zero.
  - Hệ quả: `argv[i]` với `i ≥ argc` trỏ vào chuỗi của instruction cũ trong buffer 32 KiB, hoặc vào rác chưa khởi tạo nếu slot đó chưa từng được dùng.

**Nguyên nhân:** contract "handler tự kiểm argc" không được áp dụng nhất quán, và không có kiểm tra tập trung.

**Ảnh hưởng:**
- Ai làm được: bất kỳ user nào đã có tunnel, **kể cả người xem read-only qua share link**, hoặc bất kỳ ai tới được cổng 4822.
- Cách làm: gửi `4.blob;` hoặc `5.touch;`.
- Hậu quả: đọc hoặc ghi bộ nhớ heap ngoài phạm vi, crash process connection. Phiên bị sập là phiên của owner, kéo theo mọi người đang xem. Khả năng leo thang thành ghi bộ nhớ có kiểm soát chưa được đánh giá.
- Process isolation giới hạn thiệt hại trong một phiên.

**Hướng xử lý:**
1. Thêm trường `min_argc` vào `__guac_instruction_handler_mapping`. Kiểm tra ngay trong `__guac_user_call_opcode_handler`: thiếu tham số thì log và bỏ qua lệnh.
2. Dùng `guac_mem_zalloc` cho parser. Đặt `__elementv[i] = NULL` cho các slot từ `argc` trở đi trong `guac_parser_reset`.
3. Thêm unit test (CUnit) cho mọi opcode với `argc = 0`.

**Tham chiếu:** CHECKLIST P0 · `04-BAO-MAT.md`.

### SEC-02 — Lệch đếm độ dài UTF-8 giữa guacd và Java, có thể chèn instruction `[NGUYÊN TẮC]` · High · Rủi ro tiềm ẩn

**Bằng chứng:**
- `S/libguac/unicode.c:24-67`: `guac_utf8_charsize` và `guac_utf8_strlen` đếm theo *byte đầu* của ký tự, **không kiểm các byte tiếp theo** (continuation byte).
  - Ví dụ chuỗi byte `E0 3B 58` (`\xE0;X`) được tính là **1 ký tự**.
- `S/libguac/protocol.c:72-78`: độ dài này được dùng làm tiền tố `LEN.` khi gửi instruction.
- `C/net/InetGuacamoleSocket.java:111` (và `SSLGuacamoleSocket.java:108`) đọc bằng `new InputStreamReader(..., "UTF-8")`.
  - Bộ giải mã chuẩn của Java thay byte lỗi bằng U+FFFD, nên với chuỗi trên nó thấy 3 ký tự: `�`, `;`, `X`.
  - Parser Java lấy 1 ký tự (U+FFFD) làm giá trị, gặp `;` thì coi là **kết thúc instruction**. Phần còn lại do kẻ tấn công kiểm soát được parse thành **instruction mới**.
- Có ít nhất một nguồn dữ liệu thô từ máy đích đi thẳng vào luồng này: tiêu đề cửa sổ terminal. `S/terminal/terminal-handlers.c:1462-1486` lưu nguyên byte (`title[position++] = (char) c`) rồi gọi `guac_protocol_send_name`.

**Ảnh hưởng:**
- Cách kích hoạt: nội dung hiển thị trong phiên SSH/Telnet chứa chuỗi OSC kèm byte UTF-8 lỗi. Ví dụ `cat` một file log do bên thứ ba ghi.
- Kết quả: chèn được instruction tùy ý vào luồng guacd → web app → trình duyệt. Có thể là `file` (ép tải file xuống), `clipboard` (ghi đè clipboard local), hoặc làm nhiễu `StreamInterceptingTunnel` phía web app.
- Ảnh hưởng tới **mọi người đang xem** phiên chia sẻ và tới file recording.
- Có thể còn các nguồn khác ngoài tiêu đề terminal (chưa rà hết).

**Hướng xử lý:**
1. Validate và làm sạch UTF-8 trong `__guac_socket_write_length_string`: thay chuỗi lỗi bằng U+FFFD **trước khi** đếm độ dài, sao cho khớp với cách Java và JS đếm.
2. Phía Java: dùng `CharsetDecoder` với `CodingErrorAction.REPORT` để đóng tunnel khi gặp UTF-8 lỗi, không âm thầm thay thế.
3. Thêm test với các chuỗi byte lỗi cho cả hai phía.

**Độ chắc chắn:** cả hai nửa của sự sai lệch đều đã xác nhận trong code. Toàn bộ chuỗi khai thác end-to-end chưa được chạy thử.

### SEC-03 — RDPDR không kiểm FileId đang mở `[NGUYÊN TẮC]` · High · Đã xác nhận

**Bằng chứng:**
- `S/protocols/rdp/fs.c:729-737`: `guac_rdp_fs_get_file` chỉ kiểm `0 ≤ id < GUAC_RDP_FS_MAX_FILES`, không kiểm slot có đang được dùng.
  ```c
  if (file_id < 0 || file_id >= GUAC_RDP_FS_MAX_FILES) return NULL;
  return &(fs->files[file_id]);
  ```
- `fs.c:561-592`: `guac_rdp_fs_close` chạy `close(file->fd)`, free các path, `guac_pool_free_int`, `open_files--` với **bất kỳ** id nào.
- `rdpdr-fs-messages.c:265-278`: `strncmp(file->absolute_path…)` chạy khi `bytes_written > 0`. Giá trị này không được reset khi close.
- `fs` cấp bằng `guac_mem_alloc` (không zero, `fs.c:64`).

**Ảnh hưởng:**
- Điều kiện: máy đích RDP độc hại hoặc bị chiếm, và connection bật drive.
- Cách làm: gửi CLOSE/READ/WRITE với FileId chưa mở hoặc đã đóng.
- Hậu quả:
  - đóng nhầm fd khác của process (socket, file recording);
  - free hai lần (double-free) hoặc use-after-free chuỗi path;
  - trùng id trong pool, dẫn tới hai file dùng chung một slot;
  - crash.
- Đây là bề mặt tấn công **từ máy đích về gateway**. Kịch bản thực tế: admin kết nối vào máy đã bị chiếm.

**Hướng xử lý:**
1. Thêm cờ `in_use` (hoặc `fd = -1`) cho mỗi slot, trả NULL khi slot chưa mở.
2. Reset `bytes_written` và các trường khác khi close.
3. Dùng `guac_mem_zalloc`.
4. Thêm mutex cho `guac_rdp_fs` (xem `CODE-01`).
5. Unit test cho chuỗi open → close → close.

### SEC-04 — Hash mật khẩu JDBC là một vòng SHA-256 `[NGUYÊN TẮC]` · High · Đã xác nhận

**Bằng chứng:** `J/security/SHA256PasswordEncryptionService.java:40-47`. Đây là implementation duy nhất của `PasswordEncryptionService` (grep).
```java
builder.append(password);
if (salt != null) builder.append(BaseEncoding.base16().encode(salt));
MessageDigest md = MessageDigest.getInstance("SHA-256");
md.update(builder.toString().getBytes("UTF-8"));
```
So sánh bằng `Arrays.equals`, không hằng thời gian (`J/user/UserService.java:404-411`).

**Ảnh hưởng:**
- Khi lộ DB (backup, snapshot, SQLi ở hệ thống dùng chung DB), GPU thử được hàng tỷ mật khẩu mỗi giây.
- Salt 32 byte chỉ chặn được rainbow table.
- Đây cũng là chỗ lưu mật khẩu của admin gateway, tức là chìa khóa vào mọi máy đích.

**Hướng xử lý:**
1. Thêm implementation PBKDF2-HMAC-SHA256 (có sẵn trong JDK, không thêm dependency, ≥ 600.000 vòng theo OWASP 2023) hoặc Argon2id.
2. Thêm cột `password_hash_algorithm` trong upgrade script cho cả 3 DB.
3. Khi user đăng nhập thành công bằng hash cũ, rehash sang thuật toán mới.
4. So sánh bằng `MessageDigest.isEqual`.

**Tham chiếu:** ADR-02.

### SEC-05 — Admin mặc định `guacadmin` `[NGUYÊN TẮC]` · High · Đã xác nhận (rủi ro triển khai)

**Bằng chứng:**
- `X/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/002-create-admin-user.sql:20-39`, có bản tương ứng cho PostgreSQL và SQL Server.
- Script tạo user `guacadmin`, cấp mọi system permission kể cả `ADMINISTER`.
- Mật khẩu mặc định được ghi ngay trong comment dòng 20 và có trong tài liệu công khai. Báo cáo không chép giá trị.
- Không đặt `expired = 1`.

**Ảnh hưởng:** deploy mà quên đổi mật khẩu là bị chiếm toàn bộ gateway, và từ đó mọi máy đích. Kẻ tấn công thường quét đúng tổ hợp này.

**Hướng xử lý:**
1. Đặt `expired = 1` trong script seed để bắt buộc đổi mật khẩu ngay lần đăng nhập đầu. Luồng `resetExpiredPassword` đã có sẵn (`J/JDBCAuthenticationProviderService.java:131`).
2. Hoặc để script `initdb.sh` của Docker sinh mật khẩu ngẫu nhiên và in ra log một lần.

### SEC-06 — Quickconnect mặc định cho mọi tham số và mọi host `[NGUYÊN TẮC]` · High · Đã xác nhận (rủi ro triển khai)

**Bằng chứng:**
- `X/guacamole-auth-quickconnect/.../utility/QCParser.java:314-326`: `allowedParams`/`deniedParams` rỗng (mặc định) thì mọi tham số đều hợp lệ.
- Không có whitelist cho protocol/hostname (`:131-156`).
- `getUserContext` không loại trừ user ẩn danh (`SharedAuthenticatedUser` là ANONYMOUS, `J/sharing/user/SharedAuthenticatedUser.java:80`). Việc người giữ share link có thực sự dùng được quickconnect thì chưa xác minh.
- Phía guacd không có allow-list nào cho tham số (`S/protocols/*/settings.c`).

**Ảnh hưởng:** user đã đăng nhập tự gõ URI kiểu `rdp://host?drive-path=/&enable-drive=true&recording-path=/etc/...`. Họ có thể:
1. **Đọc/ghi file trên máy chạy guacd** qua `drive-path` / `sftp-*`, hoặc ghi file vào thư mục tùy ý qua `recording-path` / `typescript-path` / `create-recording-path`.
2. Dùng guacd làm **pivot SSRF** tới mạng nội bộ (`hostname`, `port`, `gateway-*`, `wol-broadcast-addr`).
3. Bật `ignore-cert`.
4. Nếu dùng chung với vault: dùng token `${KEEPER_*}` để lấy secret của host khác (suy luận, chưa xác minh).

**Hướng xử lý:**
1. Đổi mặc định của quickconnect sang deny cho nhóm tham số nguy hiểm: `*-path`, `create-*-path`, `enable-drive`, `enable-sftp`, `sftp-*`, `recording-*`, `typescript-*`, `wol-*`, `gateway-*`, `static-channels`, `ignore-cert`, `security`, `printer-name`.
2. Thêm allow-list cho hostname/CIDR.
3. Chặn user ANONYMOUS.
4. Không thay `${…}` token trong connection do user tự định nghĩa.

### TEST-01 — Java gần như không có test `[NGUYÊN TẮC]` · High · Đã xác nhận

**Bằng chứng:**
- Toàn repo có 19 file test Java cho khoảng 111k dòng Java (`git ls-files '*.java' | grep /test/`).
- `guacamole-auth-jdbc` có 26,4k dòng mà **0 test**. `guacamole-auth-ldap` có 5,2k dòng, 0 test.
- Webapp có 27,6k dòng mà chỉ 1 test (`GuacamoleVersionTest`).
- Toàn bộ extensions có 12 `@Test`.
- Không có test nào cho `AuthenticationService`, `HashTokenSessionMap`, `TunnelRequestService`, `DirectoryResource` PATCH, mô hình quyền JDBC, `RestrictedGuacamoleTunnelService`.

**Ảnh hưởng:**
- Logic phân quyền (READ trong SQL, ADMINISTER khi cấp quyền) phải được sửa ở 3 DB (`ARCH-02`) mà không có lưới an toàn.
- Lỗi như `ERR-07` (sai ngày) tồn tại mà không bị phát hiện.
- Refactor `SEC-04` (đổi thuật toán hash) rất rủi ro nếu không có test trước.

**Hướng xử lý:**
1. Thêm test tích hợp JDBC bằng Testcontainers (Postgres, MySQL) cho: đăng nhập, `selectReadable`, cấp quyền, concurrency limit, history.
2. Unit test `AuthenticationService` (thứ tự provider, insufficient credentials).
3. Unit test `TokenFilter` và `DailyRestriction`.

### TEST-02 — Frontend 0 test, common-js chỉ 3 spec `[NGUYÊN TẮC]` · High · Đã xác nhận

**Bằng chứng:**
- `guacamole-common-js/src/test/javascript/` chỉ có `ParserSpec`, `EventSpec`, `SessionRecordingSpec` (393 dòng, khoảng 29 `it`).
- `guacamole/src/main/frontend` có khoảng 40,7k dòng mà **không có test nào**.

**Ảnh hưởng:** `Tunnel`/`ChainedTunnel` (fallback), `Display` (frame và flush), `Keyboard` (keysym), `authenticationService`, import CSV/JSON/YAML đều không có test. Các lỗi như `ERR-03`, `ERR-05`, `SEC-11` lọt qua.

**Hướng xử lý:**
1. Bắt đầu với Jasmine/Karma có sẵn cho common-js: `TunnelSpec` (fallback, timeout), `DisplaySpec` (decode lỗi), `ClientSpec` (dispatch).
2. Frontend: thêm Karma + angular-mocks cho `authenticationService` và `connectionParseService`.

---

## Chi tiết — Medium

### SEC-07 — Lỗi bộ nhớ RDPDR khi liệt kê thư mục và chuyển UTF-16 `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- **Đọc vượt stack khi liệt kê thư mục.** `S/protocols/rdp/channels/rdpdr/rdpdr-fs-messages-dir-info.c:40-45, 74`. Cùng mẫu này lặp ở `:88/123`, `:137/176`, `:190/216`.
  ```c
  int length = guac_utf8_strlen(entry_name);
  int utf16_length = length*2;
  unsigned char utf16_entry_name[256];
  guac_rdp_utf8_to_utf16(..., sizeof(utf16_entry_name));
  Stream_Write(output_stream, utf16_entry_name, utf16_length);
  ```
  Tên file dài hơn 128 ký tự (Linux cho phép tới 255) khiến `Stream_Write` đọc vượt mảng 256 byte, **gửi dữ liệu stack cho máy đích**. Dữ liệu này có thể chứa con trỏ, giúp vượt ASLR.
- **Off-by-one khi chuyển UTF-16 sang UTF-8.** `S/protocols/rdp/unicode.c:31-44`: vòng lặp có thể ghi đủ `size` byte, sau đó vẫn chạy `*(utf8++) = 0;`. Kết quả là ghi 1 byte NUL ra ngoài buffer `path[4096]` (`rdpdr-fs-messages.c:78, 509`; `rdpdr-fs-messages-file-info.c:161`).

**Ảnh hưởng:** máy đích RDP độc hại (khi drive được bật) đọc được stack của guacd, hoặc ghi một byte NUL ra ngoài buffer.

**Hướng xử lý:**
1. Mảng `utf16_entry_name` dùng `2*256+2` byte.
2. Lấy `utf16_length` từ số byte thực sự đã ghi.
3. Hàm `utf16_to_utf8` dành 1 byte cho NUL (truyền `size - 1` vào vòng lặp).

### SEC-08 — Ranh giới tin cậy web app ↔ guacd `[NGUYÊN TẮC]` · Medium · Đã xác nhận (rủi ro triển khai)

**Bằng chứng:**
- **guacd không có bước xác thực nào** trước `select`/`connect` (`S/guacd/connection.c:301-367`). TLS phía guacd không xác thực client (không có `SSL_CTX_set_verify`) và chỉ load cert/key, không đặt TLS tối thiểu (`S/guacd/daemon.c:559-586`).
- **Image Docker** chạy `exec /opt/guacamole/sbin/guacd -f -b 0.0.0.0 …` (`S/guacd-docker/bin/entrypoint.sh`) kèm `EXPOSE 4822`.
- **Web app** mặc định không dùng TLS khi nối guacd (`E/environment/LocalEnvironment.java:85`, `DEFAULT_GUACD_SSL = false`). Khi bật TLS thì không kiểm hostname, không gửi SNI (`C/net/SSLGuacamoleSocket.java:87-102`, không có `setEndpointIdentificationAlgorithm`).
- Cài native mặc định bind `localhost` (`S/guacd/conf.h:29`). Đây là điểm tốt.

**Ảnh hưởng:** ai tới được cổng 4822 (cổng publish nhầm, container khác cùng network, máy cùng VLAN) có thể:
- bắt guacd mở RDP/SSH tới host tùy ý (pivot);
- join phiên đang chạy nếu biết ID;
- ghi file qua `recording-path`.

Ngoài ra, mật khẩu máy đích đi plaintext giữa web app và guacd khi hai bên tách máy.

**Hướng xử lý:**
- **Vận hành, làm ngay:** không publish cổng 4822 (dùng docker network nội bộ), firewall, giới hạn egress của guacd.
- **Code:**
  1. Hỗ trợ mTLS (`ssl_client_ca` trong `guacd.conf`), đặt `SSL_CTX_set_min_proto_version(TLS1_2)`, dừng hẳn khi chỉ có một trong hai file cert/key.
  2. Phía Java: kiểm hostname bằng `SSLParameters`.

**Tham chiếu:** ADR-01.

### SEC-09 — Auth token nằm trong URL `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `FE/client/types/ManagedClient.js:341` (`"token=" + encodeURIComponent(...)`) kết hợp `CJ/Tunnel.js:991` (`new WebSocket(tunnelURL + "?" + data, …)`).
- `FE/rest/services/tunnelService.js:218, 304` dùng `?token=` cho download/upload.
- `W/rest/auth/TokenRESTService.java:209` có `DELETE /api/tokens/{token}`.
- `W/rest/auth/AuthenticationService.java:542` chấp nhận token từ query.
- Log DEBUG in nguyên token: `W/rest/auth/HashTokenSessionMap.java:143,158`, `W/tunnel/TunnelRequestService.java:300,368`, `W/rest/RESTExceptionMapper.java:78`.

**Ảnh hưởng:**
- Token 256 bit, tương đương session, nằm trong access log của Tomcat, reverse proxy, LB, WAF.
- Ai đọc được log là chiếm được phiên. Phiên có tunnel đang mở thì không bao giờ hết hạn.

**Hướng xử lý:**
1. Upload chuyển sang header `Guacamole-Token` (đã là XHR nên làm được ngay).
2. Download và WebSocket dùng ticket một lần, ngắn hạn, gắn với từng stream (ADR-03).
3. Trước mắt: cấu hình access log che `token=`.
4. Chỉ log hash hoặc tiền tố của token.

### SEC-10 — Không có header bảo mật; token trong localStorage `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `guacamole/src/main/webapp/WEB-INF/web.xml` chỉ có `CacheRevalidationFilter` và `GuiceFilter`.
- Grep `Content-Security-Policy|X-Frame-Options|frame-ancestors|X-Content-Type-Options` trên webapp và frontend không có kết quả.
- `FE/auth/service/authenticationService.js:83,146-148` lưu `GUAC_AUTH_TOKEN` vào `localStorage`.
- `FE/index.html` nạp `app.js` của extension ở phạm vi global.

**Ảnh hưởng:**
- UI quản trị và phiên remote có thể bị nhúng iframe để clickjacking, ví dụ lừa admin click cấp quyền.
- Không có lớp phòng thủ thứ hai cho XSS: một XSS bất kỳ (từ extension, hoặc từ CVE của AngularJS đã EOL) là đọc được token và tồn tại qua việc đóng tab.

**Hướng xử lý:**
1. Thêm servlet filter đặt các header, cho phép cấu hình bằng property:
   - `Content-Security-Policy: default-src 'self'; frame-ancestors 'self'; object-src 'none'`;
   - `X-Content-Type-Options: nosniff`;
   - `Referrer-Policy: same-origin`.
2. Hoặc ghi rõ trong tài liệu triển khai là bắt buộc thêm ở reverse proxy.
3. Cân nhắc `sessionStorage`.

### SEC-11 — Tham số URL ghi đè token đã lưu `[NGUYÊN TẮC]` · Medium · Rủi ro tiềm ẩn

**Bằng chứng:** `FE/auth/service/authenticationService.js:299-306`:
```js
var token = service.getCurrentToken();
if (token) httpParameters.token = service.getCurrentToken();
if (parameters) angular.extend(httpParameters, parameters);   // parameters = $location.search()
```
Khi server trả về token khác token hiện tại, frontend revoke token cũ rồi lưu token mới (`:217-230`).

**Ảnh hưởng:**
- Link `https://gw/#/?token=<token-của-kẻ-tấn-công>` đưa nạn nhân vào phiên của kẻ tấn công và đăng xuất phiên thật của họ.
- Nạn nhân có thể gõ mật khẩu vào desktop do kẻ tấn công điều khiển và ghi lại.
- Ngoài ra, `username`/`password` có thể nằm trên URL và bị lưu vào lịch sử trình duyệt.

**Hướng xử lý:**
1. Không cho tham số URL ghi đè `token` đã lưu: đảo thứ tự merge, hoặc chỉ cho phép một whitelist tham số SSO/share key.
2. Xoá query khỏi URL ngay sau khi xác thực xong.

### SEC-12 — guacd không giới hạn tài nguyên trước handshake `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- Vòng accept không có trần số kết nối (`S/guacd/daemon.c:647-715`). `listen(…, 5)` ở `:641`.
- Timeout 15 s áp cho **từng lần** `guac_socket_select` (`S/libguac/parser.c:221`), không áp cho cả handshake.
- Mỗi `select` gây fork + exec + dlopen. Process cha tốn 3 thread cho mỗi kết nối, mỗi thread stack 8 MiB (bộ nhớ ảo).

**Ảnh hưởng:** kiểu slowloris, gửi 1 byte mỗi 14 s là giữ được thread mãi. Gửi `select` hàng loạt làm cạn process/RAM của máy guacd.

**Hướng xử lý:**
1. Thêm `max_connections` vào `guacd.conf`.
2. Đặt deadline tổng cho handshake.
3. Tài liệu systemd: `TasksMax`, `LimitNPROC`.

### SEC-13 — TOTP fail-open và enroll kiểu TOFU `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `X/guacamole-auth-totp/.../user/UserVerificationService.java:363-364`: `if (key == null) return;`.
- `getKey` trả `null` khi không lưu được key ("TOTP will be disabled for this user", `:219`) hoặc khi secret không phải base32 hợp lệ (`:132`).
- Enroll (`:381-391`): ai có mật khẩu đều nhận được mã QR khi key chưa confirmed (TOFU, trust-on-first-use).
- Không có giới hạn số lần đoán mã ngoài `auth-ban`.

**Ảnh hưởng:**
- User chỉ có ở LDAP/SSO, chưa có bản ghi DB và không bật auto-create, hoặc khi DB lỗi: **đăng nhập không cần MFA** mà admin không được cảnh báo.
- Kẻ có mật khẩu bị lộ của một user chưa enroll thì enroll thay họ.

**Hướng xử lý:**
1. Thêm `totp-require=true` để fail-closed.
2. Đếm số lần nhập sai TOTP theo user.
3. Cho phép admin bắt buộc enroll qua kênh tin cậy.

### SEC-14 — Secret lưu plaintext trong DB `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `guacamole_connection_parameter.parameter_value varchar(4096)` (`schema/001-create-schema.sql:230`) chứa `password`, `private-key`, `passphrase` của máy đích.
- TOTP secret dạng base32 nằm trong `guacamole_user_attribute`.
- Cấu hình KSM (`X/guacamole-vault/.../KsmAttributeService.java:98, 194`).

**Ảnh hưởng:** lộ DB hoặc backup là lộ credential của **toàn bộ hạ tầng đích** và hạt giống MFA.

**Hướng xử lý:**
1. Mã hoá cấp ứng dụng (envelope encryption) với key nằm ngoài DB, đọc từ `guacamole.properties` hoặc biến môi trường `_FILE`.
2. Tài liệu khuyến nghị dùng vault cho credential.

### SEC-15 — JSON auth: thiết kế crypto `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `X/guacamole-auth-json/.../CryptoService.java:70-79, 142`: AES-CBC với IV toàn 0. Comment giải thích HMAC đứng đầu đóng vai trò IV.
- Cùng `json-secret-key` cho cả AES và HMAC, theo thứ tự MAC-then-encrypt.
- So sánh bằng `Arrays.equals` (`UserDataService.java:159`).
- `isExpired()` trả `false` khi không có `expires` (`UserData.java:374-383`).
- `json-trusted-networks` rỗng nghĩa là chấp nhận mọi IP (`RequestValidationService.java:87`).

**Ảnh hưởng:**
- Payload không có `expires` dùng lại được **mãi mãi** (replay).
- Thứ tự decrypt-rồi-verify mở khả năng padding oracle qua timing.
- Dùng chung một key cho hai mục đích là thiết kế crypto kém.

**Hướng xử lý:**
1. Phiên bản payload mới dùng AES-GCM với IV ngẫu nhiên.
2. Bắt buộc `expires`, có giới hạn tối đa.
3. So sánh bằng `MessageDigest.isEqual`.
4. Cảnh báo khi `json-trusted-networks` rỗng.

### SEC-16 — SSO và LDAP: các cấu hình dễ bị lợi dụng `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- CAS dùng `setAcceptAnyProxy(true)` (`X/guacamole-auth-sso/.../cas/ticket/TicketValidationService.java:141`): mọi service xin được proxy ticket đều đăng nhập được thay người dùng.
- OIDC dùng implicit flow `response_type=id_token` (`.../openid/AuthenticationProviderService.java:126`). Username mặc định lấy từ claim `email` (`.../openid/conf/ConfigurationService.java:43`) mà không kiểm `email_verified`.
- LDAP khi follow referral chọn mã hoá theo URL của referral (`X/guacamole-auth-ldap/.../LDAPConnectionService.java:249-253`): `ldap://` thì về `NONE`, gửi bind credential không mã hoá. Mặc định `ldap-follow-referrals=false`, `ldap-encryption-method=none`.
- Không thấy LDAP injection: filter và DN được dựng bằng `ExprNode`/`Rdn` của Apache Directory API.

**Hướng xử lý:**
1. CAS: cấu hình allowed proxy chains.
2. OIDC: chuyển sang Authorization Code + PKCE, dùng `sub` hoặc `preferred_username`, kiểm `email_verified`.
3. LDAP: giữ nguyên mức mã hoá ban đầu khi follow referral.

### SEC-17 — Mặc định yếu khi xác minh danh tính máy đích `[NGUYÊN TẮC]` · Medium · Đã xác nhận (rủi ro cấu hình)

**Bằng chứng:**
- **SSH:** `S/common-ssh/key.c:213-218` log "host identity will not be verified" rồi **tiếp tục** khi không có `host-key` và không có known_hosts. File known_hosts bị hard-code `/etc/guacamole/ssh_known_hosts` (`:194`).
- **RDP:** `security` mặc định là `any`, gồm cả RDP legacy (`S/protocols/rdp/settings.c:838-843, 1647-1656`). `ignore-cert` khiến callback trả 2 (`rdp.c:696-700`).
- **Kubernetes:** `use-ssl` mặc định false (`S/protocols/kubernetes/settings.c:338`).
- **VNC:** reverse-connect nhận kết nối đầu tiên tới (`S/protocols/vnc/vnc.c:327-336`).

**Ảnh hưởng:** MITM giữa guacd và máy đích, lộ mật khẩu nhập vào phiên (áp dụng cả SFTP của RDP/VNC).

**Hướng xử lý:**
1. Thêm tùy chọn strict cho SSH (cảnh báo rõ trong UI khi chưa có host key), đọc known_hosts theo `GUACAMOLE_HOME`.
2. Tài liệu hoá rủi ro của `security=any` và `ignore-cert`.

### SEC-18 — Header auth tin `REMOTE_USER` vô điều kiện `[NGUYÊN TẮC]` · Medium · Đã xác nhận (rủi ro triển khai)

**Bằng chứng:** `X/guacamole-auth-header/.../AuthenticationProviderService.java:67-75` đọc username từ header mà không kiểm IP nguồn, không có danh sách proxy tin cậy. Khi kết hợp JDBC, user này nhận luôn quyền DB của user trùng tên (`J/JDBCAuthenticationProviderService.java:105-108`).

**Ảnh hưởng:** Tomcat bị lộ ra ngoài mà không qua proxy, hoặc proxy không xoá header do client gửi lên, thì chỉ cần gửi `REMOTE_USER: guacadmin` là thành admin.

**Hướng xử lý:**
1. Thêm `http-auth-trusted-proxies` (CIDR) và chỉ đọc header khi request tới từ dải này.
2. Tài liệu bắt buộc proxy xoá header từ client.

### SEC-19 — Group từ IdP khớp group DB theo tên `[NGUYÊN TẮC]` · Medium · Đã xác nhận (có chủ đích)

**Bằng chứng:** `J/base/ModeledPermissions.java:231-233` và `MX/base/EntityMapper.xml:44-70`: effective groups gồm group DB **cộng group do provider khác khai báo, khớp theo tên**.

**Ảnh hưởng:** IdP cho người dùng tự đặt claim `groups`, hoặc có group trùng tên ngoài ý muốn (ví dụ `admins`), thì user nhận quyền của group DB đó, kể cả `ADMINISTER`.

**Hướng xử lý:**
1. Tùy chọn prefix/namespace cho group ngoài, hoặc whitelist group được ánh xạ.
2. Ghi cảnh báo rõ trong tài liệu.

### SEC-20 — Chống brute-force yếu `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `guacamole-auth-ban` chỉ đếm theo `credentials.getRemoteAddress()` và lưu trong RAM (`X/guacamole-auth-ban/.../InMemoryAuthenticationFailureTracker.java:140`). Mặc định 5 lần, khoá 300 s.
- JDBC không có lockout theo tài khoản.
- Đổi mật khẩu (`W/rest/user/UserResource.java:177-191`) gọi thẳng `authProvider.authenticateUser`, bỏ qua `AuthenticationService`, nên **không phát** `AuthenticationFailureEvent`. Vì vậy auth-ban không đếm các lần thử ở đây.

**Ảnh hưởng:**
- Brute-force phân tán qua nhiều IP không bị chặn.
- Sau reverse proxy mà không có RemoteIpValve: mọi người dùng chung một IP, nên 5 lần sai là khoá tất cả (DoS).
- User đã đăng nhập có thể thử mật khẩu qua endpoint đổi mật khẩu mà không bị đếm. Endpoint này lấy user từ path, nên khả năng thử mật khẩu của **user khác** phụ thuộc quyền mà provider cho phép (chưa xác minh với JDBC).

**Hướng xử lý:**
1. Kiểm mật khẩu cũ qua `AuthenticationService` (có phát event).
2. Chỉ cho đổi mật khẩu của chính mình.
3. Thêm đếm lỗi theo username.
4. Tài liệu cấu hình `RemoteIpValve`.

### SEC-21 — Ghostscript cho in ấn RDP `[NGUYÊN TẮC]` · Medium · Rủi ro tiềm ẩn

**Bằng chứng:** `S/protocols/rdp/print-job.c:330-366`:
- `fork()`, rồi `dup2` stdin/stdout, gọi `guac_client_log` (không async-signal-safe), rồi `execvp("gs", …)`, cuối cùng `exit(1)` (không phải `_exit`).
- Child **kế thừa mọi fd không có CLOEXEC**, bao gồm socket RDP/TLS, socket Guacamole, file drive/recording.
- `gs` được tìm qua `PATH`. Cờ `-dPARANOIDSAFER` đã lỗi thời.

**Ảnh hưởng:** Ghostscript chạy trên **PostScript do máy đích gửi**, mà Ghostscript có lịch sử bypass `-dSAFER`. Một lỗ hổng gs sẽ có sẵn fd của phiên để đọc/ghi.

**Hướng xử lý:**
1. Đóng mọi fd > 2 trước `execv`, hoặc dùng `close_range`.
2. Dùng `_exit`, đường dẫn tuyệt đối tới `gs`.
3. Sandbox (seccomp/AppArmor), hoặc khuyến nghị tắt `enable-printing` khi không cần.

### SEC-22 — `enable-drive` thiếu `drive-path` thì map vào `/` `[NGUYÊN TẮC]` · Medium · Đã xác nhận (rủi ro cấu hình)

**Bằng chứng:** `drive_path` mặc định là `""` (`S/protocols/rdp/settings.c:1077-1079`). `__guac_rdp_fs_translate_path` (`fs.c:137-170`) ghép `"" + "/etc/…"`. Không có kiểm tra nào ở `rdp.c:1098`.

**Ảnh hưởng:** máy đích RDP và user (qua trình duyệt file của Guacamole) đọc/ghi được toàn bộ filesystem mà user chạy guacd có quyền.

**Hướng xử lý:**
1. Từ chối bật drive khi path rỗng hoặc là đường dẫn tương đối.
2. Hardening thêm: `openat2(RESOLVE_BENEATH)`.

### SEC-30 — RADIUS và Blast-RADIUS `[NGUYÊN TẮC]` · Medium · Chưa đủ dữ liệu

**Bằng chứng:** `X/guacamole-auth-radius/.../RadiusConnectionService.java:216-217` gửi cả `User-Password` và `Cleartext-Password`, không thấy `Message-Authenticator`. Hành vi bên trong JRadius chưa đọc.

**Ảnh hưởng (nếu đúng):** chịu CVE-2024-3596 (Blast-RADIUS).

**Hướng xử lý:** xác minh với JRadius; nếu đúng thì thêm `Message-Authenticator`. Vấn đề này không bị trừ điểm.

### MAL-01 — Build Docker tải mã/JAR không kiểm checksum `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `guacamole-server/src/guacd-docker/bin/autobuild.sh`: `git clone` FreeRDP, libssh2, libtelnet, libvncserver, libwebsockets từ GitHub, rồi checkout **tag mới nhất khớp regex** (`git tag -l --sort=-v:refname | grep -Px -m1 "$PATTERN"`). Không ghim commit, không kiểm chữ ký tag.
- `guacamole-client/guacamole-docker/build.d/020-download-drivers.sh`: tải JDBC driver bằng `curl -L "$URL"` (không có `-f`, không `sha256sum`) rồi đặt vào classpath.

**Ảnh hưởng:**
- Build không tái lập được.
- Nếu upstream (tag mới) hoặc CDN bị chiếm, mã lạ vào thẳng image chính thức.
- `curl` không có `-f` thì trang lỗi HTML vẫn bị lưu thành `.jar`.

**Hướng xử lý:**
1. Ghim commit SHA và kiểm `sha256sum` cho từng dependency.
2. Thêm `curl -fsSL`.
3. Cân nhắc lấy JDBC driver qua Maven Central (đã có checksum).

### ARCH-01 — Lặp code giữa các protocol plugin `[NGUYÊN TẮC]` · Medium (hệ thống) · Đã xác nhận

**Bằng chứng:**
- Parse tham số recording (khoảng 47 dòng) được chép ở 5 file `S/protocols/*/settings.c`.
- Parse SFTP có 96/93 dòng gần trùng ở RDP và VNC. Parse WOL 26 dòng × 4.
- `argv.c` của SSH/Telnet/K8s gần như giống hệt nhau.
- `guac_rdp_push_settings` có hai nhánh song song (getter/setter và truy cập struct trực tiếp), khoảng 470 dòng (`S/protocols/rdp/settings.c:1546-2023`).
- Hai stack render: `guac_display` (RDP/VNC) và `guac_common_surface` (terminal).

**Ảnh hưởng:** sửa một lỗi (ví dụ chặn `recording-path` nguy hiểm) phải sửa 5 nơi, dễ sót.

**Hướng xử lý:**
1. Gom thành `S/common/settings-{recording,sftp,wol}.c` (cây thư mục đích ở `02-KIEN-TRUC-HE-THONG.md` §16).
2. Mỗi helper một commit, chạy test sau mỗi bước.

### ARCH-02 — Mapper và schema lặp ở 3 DB `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- Mỗi DB có 21 file mapper, khoảng 3,9k dòng. So với MySQL, Postgres giống khoảng 86%, SQL Server giống khoảng 74%.
- Schema 001 có cấu trúc bảng giống hệt nhau, chỉ khác cú pháp.
- Số upgrade script lệch nhau: MySQL 11, Postgres 9, SQL Server 2.

**Ảnh hưởng:** logic phân quyền (`selectReadable`, `isRelatedEntity`) phải sửa 3 lần mà không có test (`TEST-01`). Ba DB dễ lệch hành vi.

**Hướng xử lý:**
1. Tách các fragment SQL chung vào `<sql>`/`<include>` dùng chung.
2. Bộ test tích hợp chạy chung cho cả 3 DB.
3. Không đề xuất chuyển sang ORM khác (xem "Không nên làm").

### ARCH-03 — Trạng thái quan trọng nằm trong RAM của từng JVM `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- Các trạng thái sau đều là map trong bộ nhớ:
  - session token (`W/rest/auth/HashTokenSessionMap.java:55`);
  - concurrency limit (`J/tunnel/RestrictedGuacamoleTunnelService.java:63-87`);
  - ban (`InMemoryAuthenticationFailureTracker`);
  - mã TOTP đã dùng (`CodeUsageTrackingService`);
  - denylist JSON;
  - nonce SSO;
  - pending session (`E/.../AuthenticationSessionManager.java:145-159`, không được dọn khi `resume()` với id rác).
- Trong một node thì **không có race**, nhờ CAS loop (`RestrictedGuacamoleTunnelService.java:113-150`).

**Ảnh hưởng khi chạy nhiều node:**
- `max-connections` nhân lên theo số node.
- Replay TOTP/JSON dùng được qua node khác.
- Restart là mất toàn bộ phiên.
- `pendingSessions` có thể bị làm phình mà không cần xác thực.

**Hướng xử lý:** ADR-04. Chọn một trong hai:
- chính thức chỉ hỗ trợ single-node (hoặc sticky session) và ghi rõ trong tài liệu;
- dùng store chung.

Trước mắt: dọn `pendingSessions`.

### BE-01 — Tunnel bị veto không được đóng `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `W/tunnel/TunnelRequestService.java:352-364`: `createConnectedTunnel` xong mới `fireTunnelConnectEvent`. Khối `catch` chỉ huỷ session khi có `GuacamoleUnauthorizedException`, **không đóng tunnel**.
- Javadoc `E/net/event/TunnelConnectEvent.java:31-35` lại ghi tunnel "will be subsequently closed".
- `ConfiguredGuacamoleSocket` (`C/protocol/ConfiguredGuacamoleSocket.java:203-318`) không có try/finally, nên khi handshake lỗi thì socket bên trong không được đóng.

**Ảnh hưởng:** listener veto (ví dụ chính sách giờ làm việc) xong mà phiên RDP vẫn đã logon, recording đã bắt đầu. Socket/thread của guacd bị rò, bộ đếm active connection của provider sai.

**Hướng xử lý:**
```java
try { fireTunnelConnectEvent(...); }
catch (GuacamoleException | RuntimeException e) { tunnel.close(); throw e; }
```
Đồng thời thêm đóng socket trong constructor khi handshake lỗi.

### DB-01 — SSL tới DB mặc định opportunistic `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:** `guacamole-auth-jdbc-mysql/.../conf/MySQLEnvironment.java:106` (`DEFAULT_SSL_MODE = MySQLSSLMode.PREFERRED`) và `guacamole-auth-jdbc-postgresql/.../conf/PostgreSQLEnvironment.java:110` (`PostgreSQLSSLMode.PREFER`). Cả hai chế độ mã hoá nếu server hỗ trợ nhưng không verify certificate.

**Ảnh hưởng:** MITM giữa web app và DB khi tách máy. Dữ liệu đi qua gồm hash mật khẩu, mật khẩu máy đích (`SEC-14`).

**Hướng xử lý:** tài liệu và Docker env khuyến nghị `VERIFY_IDENTITY` / `verify-full`. Cân nhắc đổi mặc định ở major version.

### VAL-01 — DPI = 0 làm sập process `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `S/libguac/user-handlers.c:724-725`: `user->info.optimal_resolution = atoi(argv[2]);` không có kiểm tra.
- Giá trị này được dùng làm số chia ở `S/protocols/rdp/resolution.c:31`, `settings.c:872-874`, `input.c:110`.
- Phía Java không giới hạn width/height/dpi (`W/tunnel/TunnelRequestService.java:139-151`).

**Ảnh hưởng:** một người tham gia phiên chia sẻ (kể cả read-only) đặt `GUAC_DPI=0` trong URL tunnel là gây SIGFPE, làm sập phiên của owner và mọi người xem.

**Hướng xử lý:**
1. libguac: nếu `dpi ≤ 0` hoặc quá lớn thì dùng 96.
2. Java: validate khoảng giá trị (ví dụ width/height 1–8192, dpi 48–600) và trả 400 khi sai.

### ERR-01 — SFTP của SSH khởi tạo lỗi dẫn tới deref NULL `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `S/protocols/ssh/ssh.c:419-436` không kiểm `sftp_filesystem == NULL`. RDP và VNC thì có kiểm (`rdp.c:1208`, `vnc.c:662`).
- Handler vẫn được gán (`ssh/user.c:95-96`) rồi deref `filesystem` (`common-ssh/sftp.c:381, 569`).

**Ảnh hưởng:** server không có subsystem SFTP, hoặc `sftp-root-directory` là đường dẫn tương đối, thì user upload là crash phiên.

**Hướng xử lý:** kiểm NULL và hủy phiên giống RDP, hoặc không gán handler khi SFTP lỗi.

### ERR-02 — SSH ghi non-blocking bỏ qua giá trị trả về `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:** `S/protocols/ssh/ssh.c:496` đặt non-blocking, nhưng `:211-213` `libssh2_channel_write(...)` không kiểm `LIBSSH2_ERROR_EAGAIN` hay việc chỉ ghi được một phần.

**Ảnh hưởng:** paste lượng lớn (script, key) khi cửa sổ SSH đầy thì **mất dữ liệu âm thầm**, có thể chạy lệnh bị cắt dở.

**Hướng xử lý:** lặp tới khi ghi hết, xử lý `EAGAIN` bằng `libssh2_session_block_directions` + `select`.

### ERR-03 — Decode ảnh lỗi làm treo pipeline vẽ `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:** `CJ/Display.js:928-931` và `850-853`: promise của `imageDecoder.decode()` / `createImageBitmap()` không có `.catch`. Nhánh `Image` thì có `onerror = task.unblock` (`:879, 981`).

**Ảnh hưởng:**
- Một ảnh hỏng hoặc một mimetype không hỗ trợ làm frame bị chặn mãi (`syncFlush` dừng, `:207-209`).
- Client không bao giờ gửi `sync`, màn hình đứng hình, server timeout.

**Hướng xử lý:** `.catch(() => task.unblock())`, bọc `new ImageDecoder` trong try/catch, thêm test vào `DisplaySpec`.

### ERR-07 — Khung giờ qua nửa đêm kiểm sai ngày `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:** `X/guacamole-auth-restrict/src/main/java/org/apache/guacamole/calendar/DailyRestriction.java:130-133`:
```java
if (endTime.isBefore(startTime)) {
    if (weekDays.contains(currentDay) && currentTime.isAfter(startTime) ...) return true;
    return (weekDays.contains(currentDay.plus(1)) && ... currentTime.isBefore(endTime));
}
```
Phần sau nửa đêm thuộc về **ngày hôm trước**, nên phải là `currentDay.minus(1)`. Múi giờ cố định UTC.

**Ảnh hưởng:** rule "Thứ 2, 22:00–06:00" lại áp cho sáng **Chủ nhật** thay vì sáng Thứ 3. Kết quả là cho phép hoặc từ chối đăng nhập/kết nối sai giờ. Đây là control truy cập.

**Hướng xử lý:** đổi `plus(1)` thành `minus(1)`, thêm unit test cho các khung giờ qua nửa đêm, cân nhắc cấu hình múi giờ.

### CODE-01 — Data race và lock không nhất quán `[NGUYÊN TẮC]` · Medium (hệ thống) · Rủi ro tiềm ẩn

**Bằng chứng:**
- **Broadcast socket:** `S/libguac/socket-broadcast.c:259-312` lock bằng một lượt `foreach_user`, unlock bằng một lượt khác, không giữ users lock xuyên suốt. Ở giữa hai lượt, thread promote (`client.c:208`) hoặc `remove_user` có thể đổi danh sách. Hệ quả: unlock mutex không giữ (UB), hoặc user mới nhận nửa sau của một instruction.
- **`guac_rdp_fs`:** không có mutex. Thread channel của FreeRDP và thread của user cùng sửa `files[]` và `open_files`.
- **`client->state`:** là enum thường, đọc/ghi từ nhiều thread (`client.c:247, 397-399`). `__owner` được gán mà không giữ lock.
- **Java:** `GuacamoleSession.lastAccessedTime`, `userContexts`, `authenticatedUser` không `volatile` (`W/GuacamoleSession.java:51,57,72`). Cache của `FileAuthenticationProvider` không đồng bộ.

**Hướng xử lý:**
1. Giữ read-lock danh sách users từ lúc begin tới lúc end của broadcast.
2. Thêm mutex cho `guac_rdp_fs`.
3. Dùng `stdatomic` cho state.
4. Java: `volatile` hoặc `AtomicReference`.
5. Chạy test dưới ThreadSanitizer (`TEST-03`).

### TEST-03 — C: guacd không có test, không fuzz `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `S/guacd` không có test nào.
- `S/libguac/tests` có 88 hàm test (mem, parser, socket, string, unicode…). Đây là điểm tốt, nhưng test parser chỉ có ca đúng: không có ca vượt độ dài, vượt số phần tử, tràn chữ số hay UTF-8 lỗi.
- Plugin có 45 hàm test, chủ yếu về `normalize_path`.
- Không có test cho PDU RDPDR/rdpsnd/cliprdr, escape sequence của terminal.
- Không có cấu hình ASan/UBSan/TSan hay harness fuzz.

**Ảnh hưởng:** `SEC-01`, `SEC-02`, `SEC-03`, `SEC-07`, `VAL-01` đều là những lỗi mà fuzzing cơ bản sẽ tìm ra.

**Hướng xử lý:**
1. Thêm `--enable-sanitizers` trong `configure.ac`.
2. Viết harness libFuzzer cho `guac_parser_append`, `__guac_user_call_opcode_handler`, `guac_terminal_write`, parser PDU rdpdr.
3. Cân nhắc đăng ký OSS-Fuzz.

### TEST-04 — CI server không chạy test `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:**
- `guacamole-server/.github/workflows/pr-build.yml` chỉ chạy `docker build`. `autobuild.sh` chỉ chạy `make && make install`, không có `make check`.
- Client có `MAVEN_ARGUMENTS="-DskipTests=false"` trong Dockerfile, nên test Java/JS **có chạy** khi build Docker.
- Cả hai repo đều không đo coverage.

**Hướng xử lý:**
1. Thêm job `make check` (kèm ASan) cho server.
2. Thêm JaCoCo cho client để thấy độ phủ.

### OPS-01 — Runtime và dependency đã hết hỗ trợ (EOL) `[NGUYÊN TẮC]` · Medium (hệ thống) · Đã xác nhận phiên bản

**Bằng chứng:**

| Thành phần | Bằng chứng |
|---|---|
| `alpine:3.18` | `guacamole-server/Dockerfile:27` |
| `openssl1.1-compat-dev` | `:182` |
| Node `v16.19.1` cho test common-js | `guacamole-common-js/pom.xml:222` |
| AngularJS `^1.8.3` | `guacamole/src/main/frontend/package.json` |
| Java source/target `1.8`, servlet 2.5 | `pom.xml:260-261` |
| Dependency jetty 8.1.1 / jetty 9.0.7 / tomcat 7.0.37 cho module WebSocket legacy | `guacamole/pom.xml:263-300` |
| `actions/checkout@v3` | `pr-build.yml` (cả hai repo) |

**Ghi chú về EOL:** ngày EOL lấy theo hiểu biết chung của reviewer, không tra trực tuyến trong phiên này: AngularJS 31/12/2021, Node 16 9/2023, OpenSSL 1.1.1 9/2023, Alpine 3.18 5/2025. Cần xác nhận lại bằng `npm audit`, `osv-scanner` hoặc trang EOL chính thức.

**Ảnh hưởng:**
- Không còn bản vá bảo mật, CVE của AngularJS sẽ không được sửa.
- Image guacd chính thức link với OpenSSL 1.1.

**Hướng xử lý:**
1. Nâng Alpine và OpenSSL 3 cho image server.
2. Thống nhất Node 24 cho common-js.
3. Lên lộ trình thay AngularJS (ADR-05).
4. Gỡ module WebSocket legacy (jetty 8, tomcat 7) ở major version kế tiếp.

### OPS-02 — Không lint/SAST; CI chưa hardening `[CHUẨN]` · Medium · Đã xác nhận

**Bằng chứng:**
- Không có ESLint/JSHint, SpotBugs/PMD/Checkstyle, cũng không có clang-tidy/cppcheck trong build.
- OWASP dependency-check 8.3.1 chỉ có trong profile opt-in `check-dependencies` (`pom.xml:591-661`).
- Workflow dùng `actions/checkout@v3` không ghim SHA, không khai báo `permissions`. Điểm tốt: dùng `pull_request` (không phải `pull_request_target`) và `persist-credentials: false`.

**Hướng xử lý:**
1. Ghim action theo SHA, thêm `permissions: contents: read`.
2. Bật dependency-check trong CI.
3. Thêm ESLint (rule tối thiểu).
4. Cân nhắc CodeQL cho C/Java/JS.

### OPS-03 — Thay đổi quyền và membership không có audit `[NGUYÊN TẮC]` · Medium · Đã xác nhận

**Bằng chứng:** `W/rest/permission/PermissionSetResource.java:180-303` áp patch trực tiếp. Grep `fireDirectory|listenerService` trong `rest/permission/` và `rest/identifier/` không có kết quả. `EventLoggingListener` chỉ log thao tác trên directory object và auth.

**Ảnh hưởng:** việc cấp `ADMINISTER` hay thêm user vào group admin, tức sự kiện cần audit nhất, không có trong log (STRIDE: Repudiation).

**Hướng xử lý:** định nghĩa `PermissionChangeEvent` và `MembershipChangeEvent`, phát ở tầng REST, rồi để `EventLoggingListener` ghi lại.

---

## Chi tiết — Low (rút gọn)

| Mã | Bằng chứng chính | Ảnh hưởng | Hướng xử lý |
|---|---|---|---|
| SEC-23 | `SAXParserFactory.newInstance().newSAXParser()` không `setFeature` (`W/auth/file/FileAuthenticationProvider.java:106`); `Authorization.java:183-218` chỉ hỗ trợ PLAIN/MD5/SHA-256 không salt, so sánh bằng `String.equals` | Chỉ khai thác được khi ghi được `GUACAMOLE_HOME` (vốn là vùng tin cậy); hash yếu | Bật `disallow-doctype-decl`; hỗ trợ bcrypt/PBKDF2; tài liệu ghi "không dùng cho production" |
| SEC-24 | process title chứa host/port (`S/guacd/proc-title.c:243-277`); connection ID log ở mức INFO (`S/guacd/connection.c:386`); `APIError` trả nguyên `getMessage()` (`W/rest/APIError.java:184`); header `Guacamole-Error-Message` phản chiếu query (`C/servlet/GuacamoleHTTPTunnelServlet.java:239,324`) | User local trên máy guacd thấy đích kết nối; ai đọc được log và tới được guacd là join được phiên | Log ID ở mức DEBUG; message chung cho lỗi server |
| SEC-25 | không `setuid`/`chroot`/seccomp; init.d chạy root (`init.d/guacd.in:66`); pidfile mở bằng `fopen("w")` (`S/guacd/daemon.c:605`); tên protocol đi thẳng vào `dlopen` (`S/libguac/client.c:676-685`) | Hardening | Whitelist `[a-z0-9_-]`; hardening unit systemd (`NoNewPrivileges`, `ProtectSystem`) |
| SEC-26 | không override `checkOrigin` (`W/tunnel/websocket/RestrictedGuacamoleWebSocketTunnelEndpoint.java:80-91`); session không có hạn tuyệt đối khi có tunnel (`HashTokenSessionMap.java:150`); re-auth không đổi token (`AuthenticationService.java:395-399`) | Mức thấp, vì token không nằm trong cookie | Kiểm Origin theo cấu hình; đặt thời hạn phiên tuyệt đối |
| SEC-27 | `${GUAC_USERNAME}` (từ header/SSO, có thể chứa `../`) được thay vào `recording-path`/`drive-path` (`J/tunnel/AbstractGuacamoleTunnelService.java:549`); `.timing` mở không `O_EXCL` (`S/terminal/typescript.c:72-78`); không `O_NOFOLLOW` | Path traversal nếu admin dùng token trong path | Sanitize token khi dùng trong path; dùng `O_EXCL`/`O_NOFOLLOW` |
| SEC-28 | OSC riêng của Guacamole (`S/terminal/terminal-handlers.c:1317-1360`) → `sftp.c:525,566`, không đi qua `root_path` | `sftp-root-directory` không phải ranh giới bảo mật; `cat` một file là bị đẩy file về trình duyệt | Ghi rõ trong tài liệu; thêm tùy chọn tắt OSC |
| SEC-29 | focus/copy là đọc clipboard local rồi broadcast tới mọi `guacClient` (`FE/index/controllers/indexController.js:295-305`, `FE/client/directives/guacClient.js:623-625`) | Nội dung copy rò từ máy A sang máy B khi mở nhiều connection; tới cả người xem qua share link | Chỉ đồng bộ cho client đang focus, hoặc theo thao tác của người dùng |
| ARCH-04 | `DirectoryResource.patchObjects` ~240 dòng, 3 nhánh trùng (`:549-801`); `terminal.c` 3250 dòng; `rdp/settings.c` 2023 dòng | Khó review | Tách theo nhánh khi có test |
| ARCH-05 | `GUACD_TIMEOUT` 15 s hard-code (`S/guacd/proc.h:33`) áp cho mọi handler | SFTP chậm có thể làm dừng cả phiên | Cho cấu hình trong `guacd.conf` |
| DB-02 | MySQL `DEFAULT CHARSET=utf8` (utf8mb3, 23 bảng); SQL Server dùng `nvarchar(4000)` trong khi hai DB kia dùng `varchar(4096)` | Không lưu được emoji/ký tự 4 byte; 3 DB lệch nhau | `utf8mb4` trong upgrade script |
| DB-03 | không mapper nào dọn `end_date IS NULL`; không có retention | Bảng history phình mãi; phiên "đang chạy" giả sau crash | Job dọn dẹp lúc khởi động; tài liệu cách xoá dữ liệu cũ |
| DB-04 | `@Transactional` bọc cả handshake guacd (`J/tunnel/AbstractGuacamoleTunnelService.java:703`) | Giữ connection pool khi guacd chậm | Commit insert history trước khi gọi socket |
| DB-05 | subquery `SELECT user_id FROM guacamole_user WHERE POSITION(#{term} IN username)` dựa vào cột của query ngoài (`ConnectionRecordMapper.xml:119-129`); `002-create-admin-user.sql:52` JOIN dùng `guacamole_entity.type` thay vì `affected.type`; `password_salt` cho phép NULL | Đúng nhờ may mắn, dễ vỡ khi refactor | Viết lại rõ ràng, thêm test |
| VAL-02 | PATCH `replace` không gọi `filterExternalObject` (`W/rest/directory/DirectoryResource.java:657-664`) | Bỏ qua lọc attribute; JDBC vẫn chặn ở `beforeUpdate` | Gọi filter như PUT |
| VAL-03 | `ObjectPermission.Type.valueOf(...)` khi gặp input sai thì trả 500; 7 lệnh `apply` không atomic (`PermissionSetResource.java:196-301`) | Trạng thái quyền áp một nửa khi lỗi | Validate và trả 400; `tryAtomically` |
| ERR-04 | `ESC[2147483647@` làm overflow rồi `GUAC_ASSERT` abort (`S/terminal/terminal-handlers.c:758-770`); `pa_stream_peek` gặp hole thì deref NULL (`S/pulse/pulse.c:83-94`); `SSL_write` trả 0 thì vòng ghi lặp vô hạn (`S/libguac/socket.c:109-111`) | Máy đích làm sập phiên; ăn 100% CPU | Clamp giá trị; kiểm giá trị trả về |
| ERR-05 | `chained_tunnel.onerror(code, msg)` sai kiểu tham số (`CJ/Tunnel.js:1220-1221`); `catch (Throwable/Error)` (`W/rest/auth/AuthenticationService.java:153,199,285,316,372`); `releaseWriter()` không chạy khi có RuntimeException (`C/websocket/GuacamoleWebSocketTunnelEndpoint.java:345-356`) | Lỗi UI `undefined`; OOM bị nuốt; lock bị treo | Truyền `new Guacamole.Status(...)`; chỉ bắt `Exception`; dùng `finally` |
| ERR-06 | không kiểm NULL sau `guac_parser_alloc` (`S/guacd/connection.c:294`, `S/libguac/user-handshake.c:333`); gửi `ready` trước khi kiểm `argc` (`:340-353`); dòng config > 8192 byte bị bỏ qua im lặng (`S/guacd/conf-file.c:136-156`) | Crash khi OOM; cấu hình sai mà không ai biết | Kiểm NULL; báo lỗi rõ ràng |
| ERR-08 | user `disabled` trong DB rơi vào nhánh skeleton user khi xác thực bằng LDAP/SSO (`J/JDBCAuthenticationProviderService.java:108-145`) | Admin tưởng đã khoá user nhưng user vẫn vào được (không có quyền DB) | Từ chối rõ ràng khi bản ghi DB bị disable |
| ERR-09 | regex `(\\:(.*))?\\}` tham lam (`E/token/TokenFilter.java:49`), nên `${GUAC_USERNAME:LOWER}-${GUAC_DATE}` bị thay sai | Tham số kết nối sai; các user có thể dùng chung thư mục recording | Đổi thành `(\\:([^}]*))?`; thêm test |
| ERR-10 | so `format < GUAC_RDP_MAX_FORMATS` thay vì `< format_count` (`rdpsnd-messages.c:300`); `snprintf("%%%02X", (int) c)` với `char` âm (`kubernetes/url.c:72`) | Đọc format chưa khởi tạo; URL sai với ký tự có dấu | Sửa điều kiện; ép `(unsigned char)` |
| CODE-02 | `S/protocols/ssh/ssh_agent.c` include `ssh_buffer.h` không tồn tại; `GUAC_INSTRUCTION_MAX_DIGITS` định nghĩa mà không dùng | Dead code gây nhầm lẫn | Xoá `ssh_agent.c` và cờ `--enable-ssh-agent`, hoặc viết lại |
| CODE-03 | `parsed_length*10 + c-'0'` (`S/libguac/parser.c:80-81`); `__guac_parse_int` (`user-handlers.c:87-101`); `value << 6` (`protocol.c:1358`) | UB, thực tế bị chặn bởi buffer | Giới hạn số chữ số bằng hằng đã có sẵn |
| CODE-04 | `freerdp_settings_set_string(..., guac_strdup(...))` ở 13 chỗ, FreeRDP tự copy (`S/protocols/rdp/settings.c:1553-1555`); không dùng `explicit_bzero` | Mật khẩu nằm lại trong heap | Bỏ `guac_strdup`; xoá secret sau khi dùng |
| CODE-05 | `instructionHandlers[opcode]` trên object literal (`CJ/Client.js:1059,1837`); `managedClient.users[username]` (`FE/client/types/ManagedClient.js:568-571`); import CSV/JSON | Opcode `constructor` gọi nhầm hàm; username `__proto__` gây prototype pollution nhẹ | `Object.create(null)` hoặc `Map` |
| CODE-06 | `StandardTokens` (ext) và `StandardTokenMap` (webapp) cùng định nghĩa `GUAC_USERNAME`… | Hai nguồn sự thật | Webapp dùng lại hằng của ext |
| PERF-01 | `new Thread()` không tên cho mỗi WebSocket; tạo `GuacamoleParser` mới cho mỗi message (`C/websocket/GuacamoleWebSocketTunnelEndpoint.java:230-309`); parser JS không giới hạn độ dài (`CJ/Parser.js:150-239`); proc map cấp sẵn 131072 bucket kèm mutex (`S/guacd/proc-map.c:149`) | Tốn tài nguyên; guacd bị chiếm có thể làm trình duyệt hết bộ nhớ | Đặt tên thread; tái sử dụng parser; đặt trần độ dài cho parser JS |
| NAME-01 | process cha giữ `child_socket`, process con nhận `parent_socket` (`S/guacd/proc.c:563-583, 656`) | Dễ hiểu nhầm hướng truyền fd | Đổi thành `proc_socket_parent_end` / `proc_socket_child_end` |
| NAME-02 | `guac_client`, `close_tunnel`, `message_xmlhttprequest` lẫn với camelCase; chỉ `Keyboard.js` dùng `let`/`const` | Thiếu nhất quán | ESLint `camelcase`, `no-var` khi chạm vào file |
| NAME-03 | comment ghi "GUACAMOLE_HOME" nhưng code hard-code `/etc/guacamole` (`S/common-ssh/key.c:194`); Javadoc `TunnelConnectEvent` nói tunnel sẽ bị đóng (`E/net/event/TunnelConnectEvent.java:31-35`) | Comment nói sai hành vi thật | Sửa comment hoặc sửa code (xem `BE-01`) |
| OPS-04 | `devtool: 'source-map'` và `CopyPlugin { from: 'app/**/*' }` (`webpack.config.js`), không exclude trong `guacamole/pom.xml:150-157` | WAR nặng hơn | Exclude `*.map` và source khỏi WAR |
| OPS-05 | dừng ở địa chỉ đầu tiên bind được (`S/guacd/daemon.c:520-521`); FIXME proc map không free (`:746-750`); FreeRDP cần `HOME` ghi được (`S/protocols/rdp/client.c:178-197`); `gs` lấy theo PATH | `localhost` có thể chỉ bind `::1`; lỗi môi trường khó chẩn đoán | Bind mọi địa chỉ; ghi các yêu cầu môi trường vào tài liệu |
| DOC-01 | README hai repo không nêu các ranh giới tin cậy; manual nằm ở repo `guacamole-manual` (ngoài phạm vi) | Người triển khai dễ bỏ qua các yêu cầu bảo mật | Thêm `SECURITY-DEPLOYMENT.md` ngắn liệt kê: guacd chỉ trong mạng riêng, quyền GUACAMOLE_HOME, proxy xoá header, header bảo mật, đổi mật khẩu `guacadmin` |
| DOC-02 | README dạng text cũ; link `http://guac-dev.org/doc/...` trong `X/guacamole-auth-json/.../UserData.java:90` | Link chết hoặc domain có thể bị người khác đăng ký | Cập nhật link sang `guacamole.apache.org` |
