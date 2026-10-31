# 01 — Sản phẩm và nghiệp vụ

Phân tích theo 13 mục của `product-analysis.md`. Dự án đã trưởng thành (phiên bản 1.6.1, khoảng 11.700 commit), nên phần "kế hoạch" được hiểu là kế hoạch **áp dụng hoặc đóng góp**, không phải xây mới.

## 1. Bài toán, người dùng, giá trị

**Bài toán:** truy cập máy tính từ xa (Windows qua RDP, Linux qua SSH/VNC, thiết bị qua Telnet, container qua Kubernetes) **chỉ bằng trình duyệt**, không cài client, qua một cổng tập trung có kiểm soát.

**Người dùng:**
- Bộ phận IT/DevOps quản trị server.
- Nhân viên làm việc từ xa.
- Đội hỗ trợ khách hàng.
- Phòng lab, trường học (máy ảo cho sinh viên).
- Nhà cung cấp dịch vụ cần cho bên thứ ba truy cập có kiểm soát.

**Giá trị:**
1. **Clientless:** chỉ cần HTML5.
2. **Tập trung credential:** người dùng không cần biết mật khẩu máy đích, nhờ token `${GUAC_*}` và vault.
3. **Kiểm soát và audit:** phân quyền theo connection, history, recording phiên.
4. **Tích hợp danh tính sẵn có:** LDAP/AD, SAML, OIDC, CAS, RADIUS, Duo, TOTP.
5. **Chia sẻ phiên** để hỗ trợ hoặc đào tạo.

## 2. Vấn đề thực tế cần giải quyết

- Mở thẳng RDP/SSH ra Internet là rủi ro lớn (brute-force, lỗ hổng RDP). Gateway gom về một điểm vào HTTPS có MFA.
- Cài và cập nhật client RDP/VNC trên mọi máy người dùng là tốn kém; không phải thiết bị nào cũng cài được (Chromebook, máy tính bảng).
- Cần bằng chứng về việc ai làm gì trên máy nào (compliance) → history và recording.

## 3. Chức năng theo nhóm

| Nhóm | Chức năng (đã có trong code) |
|---|---|
| **Lõi (MVP)** | Đăng nhập; danh sách connection theo quyền; mở phiên RDP/VNC/SSH/Telnet qua WebSocket/HTTP; bàn phím, chuột, cảm ứng; clipboard; resize; reconnect |
| **Quản trị** | CRUD user, group lồng nhau, connection, connection group (cân bằng tải, failover), sharing profile; quyền system và object; import connection CSV/JSON/YAML; giới hạn số phiên |
| **Bảo mật** | MFA (TOTP, Duo); SSO (SAML, OIDC, CAS, SSL client cert); LDAP; RADIUS; ban IP; khung giờ/IP (`auth-restrict`); password policy; hết hạn tài khoản; khung giờ truy cập |
| **Nâng cao** | Truyền file (RDP drive, SFTP), in RDP ra PDF, audio vào/ra, RemoteApp, WOL, chia sẻ phiên read-only/full, recording + phát lại, typescript, vault (Keeper), JSON auth (nhúng vào hệ thống khác), quickconnect, branding |
| **Tương lai / ngoài phạm vi** | HA thực sự (store chung), giao diện thay AngularJS, mTLS guacd, audit thay đổi quyền (xem `11-LO-TRINH-CAI-THIEN.md`) |

## 4. Vai trò, quyền, luồng chính

| Vai trò | Quyền tiêu biểu | Luồng chính |
|---|---|---|
| Admin hệ thống | system `ADMINISTER` | Tạo connection, cấp quyền, xem history/recording, cấu hình user |
| Admin uỷ quyền | `CREATE_CONNECTION`, `ADMINISTER` trên một nhánh connection group | Quản lý một phần cây connection |
| Auditor | `AUDIT` | Xem toàn bộ history |
| User | `READ` trên connection | Đăng nhập → home → mở connection → làm việc → (chia sẻ) → đóng |
| Người xem qua share link | user ẩn danh, chỉ thấy active connection được chia sẻ | Mở link → xem hoặc điều khiển |

Luồng kỹ thuật chi tiết (F1–F10) và sơ đồ Mermaid: `02-KIEN-TRUC-HE-THONG.md` §5–14.

## 5. Khả thi, độ phức tạp, rủi ro, điểm còn mơ hồ

- **Độ phức tạp cao:** hai ngôn ngữ (C, Java) + JS; 5 protocol nhị phân; 3 DB; 14 extension; khoảng 280k dòng.
- **Rủi ro chính:**
  - (1) Code C xử lý dữ liệu từ máy đích, có thể độc hại (`SEC-03`, `SEC-07`).
  - (2) Ranh giới tin cậy guacd ↔ web app chỉ dựa vào mạng (`SEC-08`).
  - (3) Cấu hình mặc định chưa an toàn cho production (`SEC-05`, `SEC-06`, `SEC-17`).
  - (4) Frontend phụ thuộc AngularJS đã EOL (`OPS-01`).
  - (5) Test mỏng ở luồng phân quyền (`TEST-01`).
- **Điểm repo không trả lời được:** mô hình triển khai cụ thể của người dùng (proxy, network, HA); manual chính thức (nằm ở repo khác); quy trình release và ký số image của ASF.

## 6. Kiến trúc và công nghệ (tóm tắt)

Gateway hai tầng: web app Java (Tomcat) + daemon C `guacd`, kèm giao thức text riêng. Đánh giá chung:
- **Lựa chọn hợp lý.** C dùng được trực tiếp FreeRDP, libssh2 và cho hiệu năng cao. Java có hệ sinh thái tốt cho xác thực doanh nghiệp.
- **Plugin API sạch.**
- **Không đề xuất đổi stack.**

Chi tiết ở `02-KIEN-TRUC-HE-THONG.md`, `06-BACKEND.md`, `07-FRONTEND.md`.

## 7. Cảnh báo bảo mật (tóm tắt theo mức độ)

- **High:**
  - `SEC-01`: thiếu kiểm `argc`.
  - `SEC-02`: lệch đếm UTF-8, có thể chèn instruction.
  - `SEC-03`: use-after-free trong RDPDR.
  - `SEC-04`: hash SHA-256 một vòng.
  - `SEC-05`: `guacadmin` mặc định.
  - `SEC-06`: quickconnect không giới hạn tham số.
- **Medium (16 mục):** token trong URL, thiếu CSP, guacd không xác thực, TOTP fail-open, secret plaintext trong DB…
- **Chi tiết:** `03-VAN-DE-VA-RUI-RO.md`. Threat model: `04-BAO-MAT.md`.

## 8. Thực thể dữ liệu

Các thực thể chính:
- `entity`: gộp user và group.
- `user`, `user_group` và `user_group_member`: hỗ trợ group lồng nhau.
- `connection_group`: dạng cây.
- `connection` và `connection_parameter`.
- `sharing_profile`.
- 5 bảng object permission và 1 bảng system permission.
- 3 bảng history: kết nối, đăng nhập, mật khẩu.

ER diagram: `08-DATABASE.md` §2.

## 9. Chia module và thứ tự xây dựng (đối chiếu hiện trạng)

Thứ tự phụ thuộc thực tế, cũng là thứ tự nên đọc hiểu cho người mới:
1. `libguac`: giao thức và socket.
2. `guacd`: process model.
3. Một plugin (SSH là đơn giản nhất), sau đó RDP.
4. `guacamole-common`: Java protocol và tunnel.
5. `guacamole-common-js`: Client, Tunnel, Display.
6. `guacamole-ext`: API auth.
7. Webapp: REST, tunnel, extension loader.
8. `auth-jdbc`: mô hình quyền.
9. Các extension khác.

## 10. Kế hoạch áp dụng theo giai đoạn (cho tổ chức muốn dùng)

| Giai đoạn | Nội dung | Mã liên quan |
|---|---|---|
| G0 — Thử nghiệm (1–2 tuần) | Docker compose: guacd (network nội bộ) + guacamole + PostgreSQL; đổi `guacadmin`; 2–3 connection thử | `SEC-05`, `SEC-08` |
| G1 — Pilot nội bộ | Reverse proxy TLS + header bảo mật; LDAP/SSO + TOTP; không cài quickconnect; SSH host key; tắt printing | `SEC-06`, `SEC-10`, `SEC-13`, `SEC-17`, `SEC-21` |
| G2 — Production | Backup DB (được mã hoá, vì DB chứa credential plaintext); recording lưu trên volume riêng; giám sát; quy trình cập nhật image | `SEC-14`, `DB-03`, `OPS-01` |
| G3 — Mở rộng | Vault cho credential; nhiều node với sticky session (ADR-04) | `ARCH-03` |

## 11. Ước tính sơ bộ (giả định)

Giả định: tổ chức 100–500 người dùng, khoảng 50 phiên đồng thời, một site.

**Hạ tầng:**
- 1 VM cho web app: 2 vCPU, 4 GB RAM.
- 1 VM cho guacd: 4 vCPU, 8 GB RAM. Khoảng 50–150 MB RAM cho mỗi phiên RDP tùy độ phân giải; đây là ước tính, chưa đo.
- 1 DB nhỏ.
- Storage cho recording: phụ thuộc thời lượng, có thể vài GB/ngày.

**Nhân lực:**
- Dựng G0–G1: 1 kỹ sư DevOps, 2–4 tuần.
- Vận hành: khoảng 0,2 FTE.

**Sửa các High ở `11-LO-TRINH` nếu tự fork:** 1 kỹ sư C + 1 kỹ sư Java, khoảng 3–5 tuần, kèm viết test. Ước tính này chưa gồm thời gian phối hợp với upstream.

## 12. Tiêu chí nghiệm thu và đo hiệu quả

- **Nghiệm thu bảo mật:**
  - Không còn cổng RDP/SSH mở ra Internet.
  - 100% user có MFA.
  - Tất cả điều kiện bắt buộc ở `12-BANG-DIEM.md` đều đạt.
- **Vận hành:**
  - Thời gian mở phiên dưới 5 giây.
  - Tỉ lệ phiên rớt dưới 1%.
  - `processing_lag` trung bình thấp.
- **Audit:** mọi phiên đều có history; recording bật cho connection nhạy cảm; có quy trình xem lại recording.

## 13. Tính năng dễ gây phình phạm vi / không nên bật vội

| Tính năng | Vì sao |
|---|---|
| Quickconnect | Biến gateway thành proxy mở cho mọi host và mọi tham số (`SEC-06`) |
| JSON auth không có `expires` / trusted-networks | Payload dùng lại mãi mãi (`SEC-15`) |
| Header auth | Chỉ an toàn khi proxy được cấu hình đúng tuyệt đối (`SEC-18`) |
| Printing RDP | Ghostscript xử lý dữ liệu từ máy đích (`SEC-21`) |
| Chạy nhiều node không sticky session | Session và limit nằm trong RAM (`ARCH-03`) |
| Recording mọi phiên | Tốn storage, và recording chứa dữ liệu nhạy cảm cần chính sách lưu giữ riêng |
