# 11 — Lộ trình cải thiện

Guacamole là dự án **upstream của Apache**. Lộ trình vì vậy tách hai nhóm người đọc:
- **(V) Người vận hành**: tự host Guacamole, không sửa code. Hầu hết các rủi ro triển khai xử lý được ngay bằng cấu hình.
- **(D) Người đóng góp code**: fork nội bộ hoặc gửi patch upstream. Lỗ hổng bộ nhớ phải đi qua quy trình bảo mật riêng của Apache (`04-BAO-MAT.md` §7).

## 1. Ngay (trước khi đưa vào production, hoặc tuần này)

| # | Nhóm | Việc | Mã | Tiêu chí hoàn thành |
|---|---|---|---|---|
| 1 | V | Đổi mật khẩu `guacadmin` hoặc tạo admin mới rồi vô hiệu hoá `guacadmin` | `SEC-05` | Đăng nhập bằng thông tin mặc định thất bại |
| 2 | V | guacd chỉ nằm trong network nội bộ, không publish 4822; firewall chỉ cho web app; giới hạn egress của guacd tới dải máy đích | `SEC-08`, `SEC-12` | `nc -z <ip-public> 4822` thất bại từ ngoài; guacd không tới được dải ngoài danh sách |
| 3 | V | Không cài `guacamole-auth-quickconnect`. Nếu cần thì đặt `quickconnect-allowed-parameters` chỉ gồm `hostname,port,username` | `SEC-06` | URI có `drive-path`/`recording-path` bị bỏ qua |
| 4 | V | Reverse proxy: TLS, thêm CSP `frame-ancestors 'self'`, `X-Content-Type-Options`, che `token=` trong access log; nếu dùng header auth thì xoá `REMOTE_USER` từ client và chặn truy cập thẳng Tomcat | `SEC-09`, `SEC-10`, `SEC-18` | `curl -I` thấy đủ header; access log không có token; gửi `REMOTE_USER` thẳng tới Tomcat không đăng nhập được |
| 5 | V | Tắt `enable-printing` khi không cần; mọi connection bật drive đều có `drive-path` riêng | `SEC-21`, `SEC-22` | Rà connection trong DB: không connection nào có `enable-drive=true` mà `drive-path` rỗng |
| 6 | V | Cấu hình `host-key` cho mọi connection SSH; không dùng `ignore-cert` cho RDP ở production | `SEC-17` | Log guacd không còn "host identity will not be verified" |
| 7 | D | Báo cáo riêng `SEC-01`, `SEC-02`, `SEC-03`, `SEC-07`, `VAL-01` cho Apache Security, sau khi tái hiện trên môi trường thử nghiệm có ASan | `SEC-01`…`VAL-01` | Có ticket bảo mật riêng tư được ghi nhận |

## 2. Trước release / trước khi mở rộng cho nhiều người dùng

| # | Nhóm | Việc | Mã |
|---|---|---|---|
| 8 | D | libguac: kiểm `argc` tập trung; zero hoá parser; validate DPI/size | `SEC-01`, `VAL-01` |
| 9 | D | Validate UTF-8 trước khi đếm độ dài (C), decoder REPORT (Java) | `SEC-02` |
| 10 | D | RDPDR: cờ `in_use`, buffer UTF-16 đủ lớn, off-by-one, mutex cho `guac_rdp_fs` | `SEC-03`, `SEC-07`, `CODE-01` |
| 11 | D | Hash mật khẩu PBKDF2/Argon2 kèm rehash khi đăng nhập; `expired=1` cho seed | `SEC-04`, `SEC-05` |
| 12 | D | Quickconnect mặc định deny tham số nguy hiểm; chặn user ẩn danh | `SEC-06` |
| 13 | D | Frontend: không cho URL ghi đè token; upload dùng header; `.catch` cho decode ảnh | `SEC-11`, `SEC-09`, `ERR-03` |
| 14 | D | Đóng tunnel khi bị veto; đóng socket khi handshake lỗi | `BE-01` |
| 15 | D | Sửa `DailyRestriction` (`minus(1)`) kèm test | `ERR-07` |
| 16 | D | SSH: kiểm NULL cho SFTP, xử lý EAGAIN | `ERR-01`, `ERR-02` |
| 17 | V+D | TOTP `totp-require` (fail-closed); auth-ban kết hợp `RemoteIpValve`; đổi mật khẩu qua `AuthenticationService` | `SEC-13`, `SEC-20` |
| 18 | D | Lưới an toàn: test tích hợp JDBC (Testcontainers), test `AuthenticationService`, `make check` + ASan trong CI server | `TEST-01`, `TEST-03`, `TEST-04` |
| 19 | D | Image: Alpine/OpenSSL mới; ghim commit và checksum dependency; `curl -fsSL` + sha256 cho JDBC driver | `OPS-01`, `MAL-01` |

## 3. Phiên bản sau

| # | Việc | Mã |
|---|---|---|
| 20 | Servlet filter header bảo mật, cấu hình được | `SEC-10` |
| 21 | Ticket một lần cho WebSocket/stream thay token trong URL (ADR-03) | `SEC-09` |
| 22 | mTLS giữa web app và guacd; kiểm hostname ở Java; giới hạn số kết nối của guacd (ADR-01) | `SEC-08`, `SEC-12` |
| 23 | Mã hoá cấp ứng dụng cho tham số connection nhạy cảm và TOTP secret | `SEC-14` |
| 24 | JSON auth v2 (AES-GCM, bắt buộc `expires`) | `SEC-15` |
| 25 | OIDC Authorization Code + PKCE; CAS allowed proxy chains; LDAP referral giữ nguyên mức mã hoá | `SEC-16` |
| 26 | Audit event cho thay đổi quyền và membership | `OPS-03` |
| 27 | Gom helper settings recording/SFTP/WOL; fragment SQL chung cho 3 DB | `ARCH-01`, `ARCH-02` |
| 28 | ESLint, SpotBugs, CodeQL; action ghim SHA kèm `permissions` | `OPS-02` |
| 29 | Fuzzing (libFuzzer/OSS-Fuzz) cho parser và PDU | `TEST-03` |
| 30 | Test cho common-js (Tunnel, Display) và frontend (auth, import) | `TEST-02` |

## 4. Khi tăng trưởng (nhiều node, nhiều tenant)

| # | Việc | Mã |
|---|---|---|
| 31 | Quyết định ADR-04: hỗ trợ HA bằng store chung (session, limit, ban, TOTP replay), hoặc ghi rõ chỉ hỗ trợ sticky session / single-node | `ARCH-03` |
| 32 | Lộ trình rời AngularJS (ADR-05), chuyển từng màn hình, giữ `guacamole-common-js` | `OPS-01` |
| 33 | Retention và job dọn history; metrics (số phiên, lag, lỗi) cho giám sát | `DB-03` |
| 34 | Namespace group của IdP | `SEC-19` |

## 5. KHÔNG nên làm (YAGNI hoặc rủi ro lớn hơn lợi ích)

| Đề xuất bị loại | Lý do |
|---|---|
| Viết lại guacd bằng Rust/Go | 104k dòng C ổn định, kiến trúc process isolation đã giới hạn thiệt hại. Sửa các lỗi cụ thể và fuzz rẻ hơn nhiều. Có thể cân nhắc viết lại riêng parser PDU sau khi đã có fuzz |
| Tách web app thành microservice | Một JVM với plugin API là đủ. Tách ra sẽ thêm ranh giới mạng và token, tức nhiều bề mặt tấn công hơn |
| Chuyển MyBatis sang JPA/Hibernate để bớt lặp mapper | Chi phí migrate và rủi ro hồi quy phân quyền rất lớn; `<sql>`/`<include>` đã đủ |
| Viết lại `guacamole-common-js` theo framework UI mới | Thư viện độc lập framework là tài sản; chỉ cần thay lớp SPA |
| Thêm message queue / event bus cho audit | `Listener` + DB là đủ cho quy mô gateway |
| Bật mã hoá mọi bảng DB | Chỉ cần mã hoá cột nhạy cảm (`SEC-14`); phần còn lại nên dùng mã hoá ở tầng đĩa/DB |
