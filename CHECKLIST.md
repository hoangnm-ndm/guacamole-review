# CHECKLIST — Apache Guacamole (guacamole-client + guacamole-server)

## Trạng thái hiện tại

- Cập nhật lần cuối: 2026-10-31
- Giai đoạn đang thực hiện: P0, Khẩn cấp (chưa bắt đầu)
- Review gần nhất: chuyên sâu · **6,00/10** ("Cần cải thiện trước release"). Báo cáo ở `review-source/`
- Vấn đề còn mở: Critical 0 · High 8 · Medium 34 · Low 34
- Ghi chú cho người làm tiếp:
  - Đây là hai repo upstream của Apache, commit `client@31d2a6f6a` và `server@5f9bb3b4`.
  - Task đánh dấu **(V)** làm bằng cấu hình khi triển khai, không sửa code.
  - Task đánh dấu **(D)** cần sửa code. Lỗi bộ nhớ phải báo riêng cho Apache Security trước khi mở PR công khai (`review-source/04-BAO-MAT.md` §7).
  - Không có cặp file env mẫu/thật trong repo, nên quy tắc đồng bộ env không áp dụng.

## Công việc

### P0 — Khẩn cấp

- [ ] **(V)** [SEC-05] Đổi mật khẩu hoặc vô hiệu hoá `guacadmin` ngay sau khi init DB.
  - Hoàn thành khi: đăng nhập bằng thông tin mặc định thất bại.
  - Docs: quy trình ở `docs/06-van-hanh/02-trien-khai-docker-compose.md` bước 7 (chưa áp dụng — chưa có VPS).
- [ ] **(V)** [SEC-08][SEC-12] guacd chỉ nằm trong docker network hoặc VLAN nội bộ, không publish cổng 4822; firewall chỉ cho web app; giới hạn egress của guacd.
  - Hoàn thành khi: `nc -z <ip-ngoài> 4822` thất bại; guacd chỉ tới được dải máy đích.
  - Docs: `compose.yaml` chuẩn không publish 4822; kiểm tra ở bước 10 (chưa áp dụng). Giới hạn egress guacd chưa có trong docs.
- [ ] **(V)** [SEC-06] Không cài `guacamole-auth-quickconnect`, hoặc đặt `quickconnect-allowed-parameters=hostname,port,username`.
  - Hoàn thành khi: URI có `drive-path`/`recording-path` không còn tác dụng.
  - Docs: cảnh báo + giá trị ở `docs/06-van-hanh/05-bien-moi-truong.md` (nhóm extension khác); phương án chuẩn không bật quickconnect.
- [ ] **(V)** [SEC-10][SEC-09][SEC-18] Reverse proxy: thêm CSP `frame-ancestors 'self'`, `X-Content-Type-Options: nosniff`; che `token=` trong access log; xoá `REMOTE_USER` từ client; Tomcat không lộ thẳng ra ngoài.
  - Hoàn thành khi: `curl -I` có đủ header; log không chứa token; gửi header giả không đăng nhập được.
  - Docs: Caddyfile chuẩn có CSP `frame-ancestors 'self'` + `nosniff`, Tomcat không publish (bước 4, 10). Che `token=` trong log và xoá `REMOTE_USER` chưa có trong docs.
- [ ] **(V)** [SEC-22][SEC-21][SEC-17] Rà connection: bật drive thì phải có `drive-path`; tắt `enable-printing` khi không dùng; SSH có `host-key`; không dùng `ignore-cert` ở production.
  - Hoàn thành khi: query DB không còn connection vi phạm; log guacd không có "host identity will not be verified".
  - Docs: bảng quy tắc tham số connection ở bước 8 của `02-trien-khai-docker-compose.md`.
- [ ] **(D)** [SEC-01][SEC-02][SEC-03][SEC-07][VAL-01] Tái hiện trên môi trường thử nghiệm có build ASan, rồi gửi báo cáo riêng tới Apache Security.
  - Hoàn thành khi: có xác nhận tiếp nhận từ security@apache.org (hoặc đã kết luận là không khai thác được, ghi lại lý do).

### P1 — Lưới an toàn

- [ ] **(D)** [TEST-04][TEST-03] Thêm job CI `make check` cho guacamole-server, kèm cấu hình `--enable-sanitizers` (ASan/UBSan).
  - Hoàn thành khi: PR server chạy unit test dưới ASan và báo đỏ khi test fail.
- [ ] **(D)** [TEST-01] Test tích hợp JDBC bằng Testcontainers (PostgreSQL trước), gồm login, `selectReadable`, cấp/thu quyền cần ADMINISTER, group lồng nhau, history theo quyền.
  - Hoàn thành khi: `mvn -pl extensions/guacamole-auth-jdbc test` chạy ≥ 15 test pass.
- [ ] **(D)** [TEST-01] Unit test `AuthenticationService` (thứ tự provider, insufficient credentials, re-auth) và `HashTokenSessionMap` (hết hạn).
  - Hoàn thành khi: test pass trong `mvn -pl guacamole test`.
- [ ] **(D)** [TEST-03] Viết harness libFuzzer cho `guac_parser_append` và `__guac_user_call_opcode_handler`.
  - Hoàn thành khi: chạy fuzz 1 giờ không crash, sau khi đã sửa SEC-01.
- [ ] **(D)** [TEST-02] Thêm `TunnelSpec` (fallback, timeout) và `DisplaySpec` (decode lỗi) cho common-js.
  - Hoàn thành khi: Karma chạy ≥ 10 spec mới.
- [ ] **(D)** [OPS-02] Ghim `actions/checkout` theo SHA, thêm `permissions: contents: read`, bật dependency-check trong CI.
  - Hoàn thành khi: workflow có `permissions` và job dependency-check.
- [ ] **(D)** [OPS-02] Thêm ESLint (rule tối thiểu: `no-undef`, `no-unused-vars`, `eqeqeq`) cho common-js và frontend.
  - Hoàn thành khi: `npx eslint` (cài qua devDependencies) chạy trong build, 0 lỗi mức error.

### P2 — Nền tảng (sửa lỗi bảo mật và robustness)

- [ ] **(D)** [SEC-01] Thêm `min_argc` vào bảng handler và kiểm tra trong `__guac_user_call_opcode_handler`; dùng `guac_mem_zalloc` cho parser; reset `__elementv`.
  - Hoàn thành khi: test CUnit gửi mọi opcode với `argc = 0` không crash dưới ASan.
- [ ] **(D)** [VAL-01] Validate DPI/width/height ở libguac (≤ 0 thì dùng 96) và ở `TunnelRequestService` (trả 400 khi sai khoảng).
  - Hoàn thành khi: test `size 1024 768 0` không gây SIGFPE.
- [ ] **(D)** [SEC-02] Validate và làm sạch UTF-8 trong `__guac_socket_write_length_string`; Java dùng `CodingErrorAction.REPORT`.
  - Hoàn thành khi: test với chuỗi `\xE0;X` cho cùng độ dài ở C và Java (hoặc tunnel đóng có kiểm soát).
- [ ] **(D)** [SEC-03][SEC-07][CODE-01] RDPDR: cờ `in_use` cho slot file, reset `bytes_written`, buffer UTF-16 `2*256+2`, sửa off-by-one ở `utf16_to_utf8`, mutex cho `guac_rdp_fs`.
  - Hoàn thành khi: unit test open → close → close và tên file 255 ký tự pass dưới ASan.
- [ ] **(D)** [SEC-04] Thêm `PBKDF2PasswordEncryptionService` (≥ 600k vòng), cột `password_hash_algorithm` trong upgrade script cho 3 DB, rehash khi đăng nhập, so sánh bằng `MessageDigest.isEqual`.
  - Hoàn thành khi: user cũ vẫn đăng nhập được và hash được nâng cấp; test tích hợp pass.
- [ ] **(D)** [SEC-05] Seed `002-create-admin-user.sql` đặt `expired = 1`, đồng thời sửa alias JOIN (DB-05).
  - Hoàn thành khi: lần đăng nhập đầu bằng `guacadmin` bị bắt đổi mật khẩu.
- [ ] **(D)** [SEC-06] Quickconnect mặc định deny nhóm tham số nguy hiểm (`*-path`, `create-*-path`, `enable-drive`, `enable-sftp`, `sftp-*`, `recording-*`, `typescript-*`, `wol-*`, `gateway-*`, `ignore-cert`, `security`, `static-channels`); chặn user ANONYMOUS.
  - Hoàn thành khi: `QCParserTest` có ca deny mặc định và pass.
- [ ] **(D)** [SEC-11] `updateCurrentToken` không cho tham số URL ghi đè token đã lưu; xoá query sau khi xác thực.
  - Hoàn thành khi: mở `#/?token=abc` khi đang đăng nhập không đổi phiên.
- [ ] **(D)** [SEC-09] Upload dùng header `Guacamole-Token`; log chỉ in tiền tố hoặc hash của token.
  - Hoàn thành khi: grep log DEBUG không còn token đầy đủ; upload không có `?token=`.
- [ ] **(D)** [ERR-03][ERR-05] `Display.js` thêm `.catch(task.unblock)` cho `ImageDecoder`/`createImageBitmap`; `ChainedTunnel.onerror` truyền `Guacamole.Status`.
  - Hoàn thành khi: `DisplaySpec` với ảnh hỏng vẫn flush frame sau.
- [ ] **(D)** [BE-01] Đóng tunnel khi `TunnelConnectEvent` bị veto; đóng socket khi handshake trong `ConfiguredGuacamoleSocket` lỗi.
  - Hoàn thành khi: test với listener veto không để lại tunnel mở.
- [ ] **(D)** [ERR-07] `DailyRestriction`: đổi `plus(1)` thành `minus(1)`, thêm test qua nửa đêm.
  - Hoàn thành khi: rule "Thứ 2 22:00–06:00" áp cho sáng Thứ 3, không áp cho sáng Chủ nhật.
- [ ] **(D)** [ERR-01][ERR-02] SSH: kiểm `sftp_filesystem == NULL` (hủy phiên giống RDP); `libssh2_channel_write` lặp tới khi ghi hết và xử lý EAGAIN.
  - Hoàn thành khi: paste 1 MB vào SSH không mất dữ liệu; SSH tới server không có SFTP rồi upload không crash.
- [ ] **(V/D)** [SEC-13] Thêm `totp-require` (fail-closed) và đếm số lần nhập sai TOTP theo user.
  - Hoàn thành khi: user không lưu được key thì không đăng nhập được khi bật tùy chọn.
- [ ] **(D)** [SEC-20] Đổi mật khẩu đi qua `AuthenticationService` (có phát event) và chỉ cho self; auth-ban đếm thêm theo username.
  - Hoàn thành khi: 5 lần sai mật khẩu cũ khiến IP bị ban.
- [ ] **(D)** [MAL-01][OPS-01] Image: nâng Alpine và OpenSSL 3; ghim commit SHA và checksum cho dependency trong `autobuild.sh`; `curl -fsSL` + `sha256sum` cho JDBC driver; Node 24 cho common-js.
  - Hoàn thành khi: build lại hai lần cho ra cùng phiên bản dependency; checksum sai thì build fail.
- [ ] **(D)** [DB-01] Tài liệu và env Docker khuyến nghị `VERIFY_IDENTITY` / `verify-full`.
  - Hoàn thành khi: `guacamole-docker/README.md` có mục SSL tới DB.

### P3 — Kiến trúc backend

- [ ] **(D)** [ARCH-01] Gom parse recording thành `src/common/settings-recording.c`, dùng ở 5 plugin (mỗi plugin một commit).
  - Hoàn thành khi: `grep -c "recording-path" src/protocols/*/settings.c` chỉ còn phần khai báo tên arg; test pass.
- [ ] **(D)** [ARCH-01] Gom parse SFTP và WOL tương tự.
  - Hoàn thành khi: RDP/VNC dùng chung helper SFTP.
- [ ] **(D)** [ARCH-02] Tách fragment SQL quyền dùng chung (`<sql>`/`<include>`) giữa 3 DB, sau khi đã có test tích hợp cho cả 3.
  - Hoàn thành khi: test JDBC chạy cho MySQL, PostgreSQL, SQL Server.
- [ ] **(D)** [ARCH-03] Viết ADR-04 (single-node/sticky hay store chung); dọn `pendingSessions` khi `resume()` gặp id lạ.
  - Hoàn thành khi: có ADR trong docs; map không tăng khi gửi state rác.
- [ ] **(D)** [SEC-08][SEC-12] guacd: `ssl_client_ca` (mTLS), `SSL_CTX_set_min_proto_version(TLS1_2)`, `max_connections`, deadline tổng cho handshake; Java kiểm hostname TLS.
  - Hoàn thành khi: client không có cert bị từ chối; vượt `max_connections` thì từ chối.
- [ ] **(D)** [SEC-14] Mã hoá cấp ứng dụng cho tham số connection nhạy cảm và TOTP secret.
  - Hoàn thành khi: dump DB không thấy mật khẩu máy đích ở dạng rõ.
- [ ] **(D)** [SEC-15] JSON auth v2: AES-GCM, IV ngẫu nhiên, bắt buộc `expires`, `MessageDigest.isEqual`; cảnh báo khi `json-trusted-networks` rỗng.
  - Hoàn thành khi: payload không có `expires` bị từ chối ở v2.
- [ ] **(D)** [SEC-16] OIDC Authorization Code + PKCE, kiểm `email_verified`; CAS allowed proxy chains; LDAP referral giữ nguyên mức mã hoá.
  - Hoàn thành khi: có test hoặc cấu hình mẫu cho từng mục.
- [ ] **(D)** [SEC-19] Tùy chọn prefix hoặc whitelist cho group từ IdP khi khớp với group DB.
  - Hoàn thành khi: group IdP `admins` không tự nhận quyền của group DB `admins` khi bật tùy chọn.
- [ ] **(D)** [SEC-10] `SecurityHeadersFilter` trong webapp, cấu hình được qua property.
  - Hoàn thành khi: response `/` có CSP và XFO mà không cần proxy.
- [ ] **(D)** [OPS-03] `PermissionChangeEvent`/`MembershipChangeEvent`, phát ở REST và log qua `EventLoggingListener`.
  - Hoàn thành khi: cấp ADMINISTER sinh một dòng log audit.
- [ ] **(D)** [SEC-21] Print job: đóng fd > 2 trước `execv`, dùng `_exit`, đường dẫn tuyệt đối tới `gs`.
  - Hoàn thành khi: `ls /proc/<gs-pid>/fd` chỉ còn 0, 1, 2.
- [ ] **(D)** [SEC-30] Xác minh JRadius có gửi Message-Authenticator không; thêm nếu thiếu.
  - Hoàn thành khi: có kết luận ghi vào `review-source/`.

### P4 — Frontend

- [ ] **(D)** [SEC-09] Ticket một lần cho WebSocket tunnel và download (ADR-03).
  - Hoàn thành khi: URL WebSocket và download không chứa auth token.
- [ ] **(D)** [OPS-01] Viết ADR-05: lộ trình rời AngularJS theo từng màn hình, giữ `guacamole-common-js`.
  - Hoàn thành khi: ADR được duyệt.
- [ ] **(D)** [TEST-02] Thêm Karma + angular-mocks cho `authenticationService` và `connectionParseService`.
  - Hoàn thành khi: ≥ 10 spec frontend pass.

### P5 — Hoàn thiện

- [ ] **(D)** [DOC-01] Thêm `SECURITY-DEPLOYMENT.md` (ranh giới tin cậy guacd, quyền `GUACAMOLE_HOME`, proxy xoá header, header bảo mật, guacadmin, sftp-root không phải ranh giới).
  - Hoàn thành khi: README hai repo có link tới file này.
- [ ] **(D)** [TEST-03] Đăng ký OSS-Fuzz hoặc chạy fuzz định kỳ cho parser PDU RDPDR, cliprdr, terminal.
  - Hoàn thành khi: có job fuzz định kỳ.

### Docs

- [x] Bộ docs dev + vận hành ở `docs/` (27 file, mục lục `docs/README.md`), viết từ việc đọc code, không thực thi. 2026-10-31.
  - Còn lại: phần vận hành chưa kiểm chứng trên hệ thống thật — khi dựng VPS lần đầu, điền bảng hiện trạng `docs/06-van-hanh/README.md` và sửa chỗ sai.
- [ ] Docs phương án riêng (CDN trước VPS, VPN tới LAN, panel) — chỉ viết khi chọn.

### Không nên làm lúc này

- Viết lại guacd bằng Rust/Go. Lý do: 104k dòng C ổn định, process isolation đã giới hạn thiệt hại; sửa lỗi cụ thể kèm fuzz rẻ hơn nhiều.
- Tách web app thành microservice. Lý do: plugin API trong một JVM là đủ; tách ra thêm ranh giới mạng và bề mặt tấn công.
- Đổi MyBatis sang JPA/Hibernate. Lý do: rủi ro hồi quy phân quyền lớn; `<sql>`/`<include>` là đủ để giảm lặp.
- Viết lại `guacamole-common-js` theo framework UI. Lý do: thư viện độc lập framework là tài sản, chỉ cần thay lớp SPA.

## Quyết định đã chốt

- 2026-10-31 — Docs đặt ở `docs/` thư mục gốc, git gốc chỉ track docs (ADR-001).
- 2026-10-31 — Vận hành bằng image chính thức `guacamole/guacamole:1.6.0`, `guacamole/guacd:1.6.0`, ghim tag (ADR-002).
- 2026-10-31 — Phương án chuẩn: Docker Compose + PostgreSQL 16 + Caddy trên một VPS (ADR-003).

## Lịch sử review

- 2026-10-31 — review-source (chuyên sâu, đầy đủ): 6,00/10, 76 vấn đề (H 8 · M 34 · L 34), không phát hiện mã độc.
