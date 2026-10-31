# Review source — Apache Guacamole (guacamole-client + guacamole-server)

| Mục | Giá trị |
|---|---|
| Ngày review | 2026-10-31 |
| Cấp độ | Chuyên sâu |
| Phạm vi | Toàn dự án: hai repo `guacamole-client` và `guacamole-server` |
| Chế độ | Đầy đủ (lần đầu, chưa có báo cáo cũ) |
| Commit được review | `guacamole-client@31d2a6f6a` (2026-10-14) · `guacamole-server@5f9bb3b4` (2026-10-16). Cả hai đều là bản 1.6.1 |
| Stack | **Server:** C99 · Autotools · FreeRDP 2/3 · libssh2 · libvncclient · libtelnet · libwebsockets · Cairo · OpenSSL · CUnit. **Client:** Java (target 1.8) · Maven · Guice 5.1 · Jersey 2.48 · Jackson 2.22 · MyBatis · Tomcat 9 (Docker) · JUnit 4. **Frontend:** AngularJS 1.8.3 · webpack 5 · Jasmine/Karma. **DB:** MySQL/MariaDB, PostgreSQL, SQL Server. **Hạ tầng:** Docker (Alpine 3.18, tomcat:9-jdk21), GitHub Actions (PR build) |
| Lockfile | `package-lock.json` (v3) × 2; Maven ghim phiên bản; C build từ tag git |
| Reference đã áp dụng | cap-do-review, report-format, scoring-rubric, architecture-modular, security, malware-supply-chain, testing-docs-ops, clean-code/*, backend/common (ánh xạ sang Java/C), monorepo, database, frontend/common, product-analysis, checklist-template |
| Cách thực hiện | Đọc tĩnh, không build hay chạy. 5 luồng đọc song song: guacd/libguac, protocol plugin, Java core, extension + DB, JS/frontend. Các phát hiện High và một số Medium đã được kiểm chứng lại trực tiếp trong code |

## Tóm tắt điều hành (cho người không đọc code)

**Guacamole là gì.** Cổng truy cập máy tính từ xa qua trình duyệt. Người dùng mở trang web, đăng nhập (có thể kèm MFA/SSO) rồi điều khiển máy Windows/Linux phía sau mà không cần cài phần mềm. Hệ thống gồm hai phần:
- **web app Java**: đăng nhập, phân quyền, giao diện;
- **guacd**: chương trình C dịch RDP/VNC/SSH sang một giao thức đơn giản mà trình duyệt hiểu.

**Chất lượng tổng thể: tốt về kiến trúc, cần chú ý về cấu hình và một số lỗi bộ nhớ.**
- **Thiết kế lõi đáng tin cậy:**
  - mỗi phiên chạy trong một process riêng, lỗi chỉ ảnh hưởng một phiên;
  - phân quyền được kiểm ngay trong câu truy vấn DB;
  - không thấy SQL injection;
  - token đăng nhập đủ mạnh.
- **Không phát hiện mã độc hay backdoor.**

**Rủi ro cần biết trước khi đưa vào dùng:**
1. **Cấu hình mặc định chưa an toàn cho production.**
   - Có tài khoản admin mặc định với mật khẩu công khai.
   - guacd tin mọi kết nối tới cổng 4822, và image Docker lắng nghe trên mọi địa chỉ.
   - Extension quickconnect cho người dùng tự đặt mọi tham số, kể cả đường dẫn file trên máy chủ.
   - **Cả ba xử lý được bằng cấu hình, không cần sửa code.**
2. **Một số lỗi bộ nhớ trong code C** (chưa kiểm chứng bằng PoC):
   - người dùng đã đăng nhập, kể cả người chỉ được xem, có thể làm sập phiên;
   - máy đích RDP độc hại có thể đọc hoặc ghi bộ nhớ ngoài phạm vi trên máy chạy guacd.
   - Cần báo riêng cho đội bảo mật Apache.
3. **Mật khẩu người dùng lưu bằng hash yếu** (SHA-256 một vòng). **Mật khẩu của các máy đích lưu dạng rõ** trong DB. Vì vậy backup DB phải được bảo vệ như két sắt.
4. **Giao diện dùng AngularJS đã hết hỗ trợ**, và thiếu header chống nhúng iframe hay chống XSS.
5. **Kiểm thử mỏng:** phần phân quyền DB và giao diện gần như không có test tự động.

**Kết luận: 6,00/10 — "Cần cải thiện trước release".** Có thể đưa vào production nếu đáp ứng 5 điều kiện cấu hình bắt buộc ở `12-BANG-DIEM.md`. Các lỗi code nên được theo dõi qua bản vá upstream.

## Kết luận nhanh

- **Điểm tổng:** 6,00/10. Chuyên sâu: "Cần cải thiện trước release". Không áp trần nào (0 Critical).
- **Mã độc / chuỗi cung ứng:** Không phát hiện (`malwareStatus: none`). Có 1 vấn đề chuỗi cung ứng Medium (`MAL-01`: build Docker tải mã và JAR không kiểm checksum).
- **Thống kê:**
  - theo mức độ: Critical 0 · High 8 · Medium 34 · Low 34, tổng 76;
  - theo nhãn: NGUYÊN TẮC 72 · CHUẨN 4;
  - 1 vấn đề "Chưa đủ dữ liệu".
- **Ba vấn đề cần xử lý trước nhất:**
  - `SEC-01`: handler lệnh không kiểm `argc`. User read-only cũng làm sập hoặc hỏng bộ nhớ process phiên được.
  - `SEC-02`: guacd và Java đếm độ dài UTF-8 khác nhau, nên dữ liệu từ máy đích có thể chèn instruction vào trình duyệt.
  - `SEC-03`: RDPDR không kiểm FileId đang mở, dẫn tới use-after-free / double-close khi máy đích RDP độc hại.
- **Ưu tiên cho người vận hành:** `SEC-05` (guacadmin), `SEC-08` (cổng guacd), `SEC-06` (quickconnect).

## Điểm mạnh

- **Process isolation cho mỗi connection**, có re-exec (`fexecve`), watchdog, process group, `PR_SET_PDEATHSIG`, CLOEXEC ở mọi nơi (`guacamole-server/src/guacd/proc.c:551-766`).
- **Plugin API sạch** (`guacamole-ext`):
  - nhiều implementation thật;
  - decorate chéo (MFA, vault);
  - classloader riêng cho mỗi extension;
  - phụ thuộc một chiều.
- **Giao thức text có tiền tố độ dài**, parser có giới hạn cứng ở cả ba ngôn ngữ.
- **Không có SQL injection.** Lọc READ ngay trong SQL. Cấp quyền cần ADMINISTER trên từng object.
- **Token 256 bit, xác thực qua header** (REST không bị CSRF). Share key 264 bit.
- **Path traversal được chặn kỹ, có unit test.** File recording được tạo an toàn (`guac_openat`).
- **`guacamole-common-js` độc lập framework.** Có backpressure qua `sync` và fallback từ WebSocket sang HTTP.
- **Tài liệu inline rất dày.** `-Werror -Wall -pedantic` cho lõi C. Image Docker chạy non-root.

## Hạn chế chính

- **Lỗi bộ nhớ và robustness trong C**, ở dữ liệu từ user và từ máy đích: `SEC-01`, `SEC-02`, `SEC-03`, `SEC-07`, `VAL-01`, `ERR-01`.
- **Mặc định chưa an toàn:** `SEC-05`, `SEC-06`, `SEC-08`, `SEC-17`, `SEC-22`, `DB-01`.
- **Crypto và lưu trữ secret:** `SEC-04`, `SEC-14`, `SEC-15`.
- **Token trong URL, thiếu header bảo mật:** `SEC-09`, `SEC-10`, `SEC-11`.
- **Test mỏng ở luồng rủi ro cao, CI server không chạy test, không fuzz:** `TEST-01`…`TEST-04`.
- **Runtime EOL:** AngularJS, Alpine 3.18, OpenSSL 1.1, Node 16 (`OPS-01`).
- **Lặp code** giữa plugin và giữa 3 DB (`ARCH-01`, `ARCH-02`). **Trạng thái trong RAM** cản HA (`ARCH-03`).

## Phần chưa thể xác minh

- **Chưa có PoC:** không lỗi nào được chạy thử hay viết PoC, vì skill chỉ đọc tĩnh. Khả năng khai thác `SEC-01`, `SEC-02`, `SEC-03`, `SEC-07`, `SEC-11`, `SEC-21` cần kiểm chứng trên môi trường thử nghiệm.
- **Chưa chạy công cụ quét:** `npm audit`, `osv-scanner`, OWASP dependency-check (máy review không cài sẵn). Danh sách CVE cụ thể của dependency là Chưa đủ dữ liệu. Ngày EOL lấy theo hiểu biết chung, chưa tra trực tuyến.
- **Chưa đọc chi tiết:** nội bộ thư viện ngoài (FreeRDP, libvncclient, libssh2, JRadius, java-saml, jose4j, mybatis-guice); `display-*.c`, `guacenc` video; mapper PostgreSQL/SQL Server (chỉ grep `${`); module WebSocket jetty/tomcat legacy.
- **Nằm ngoài repo:** cấu hình reverse proxy và Tomcat thực tế; manual chính thức (repo `guacamole-manual`); quy trình release và ký image của ASF.
- **Lịch sử git:** chưa đọc nội dung từng file code đã bị xoá (chỉ thống kê và kiểm commit của tác giả lạ trong 2 năm).

## Mục lục báo cáo

| File | Nội dung |
|---|---|
| [01-SAN-PHAM-VA-NGHIEP-VU.md](01-SAN-PHAM-VA-NGHIEP-VU.md) | 13 mục phân tích sản phẩm, vai trò, kế hoạch áp dụng |
| [02-KIEN-TRUC-HE-THONG.md](02-KIEN-TRUC-HE-THONG.md) | **C4 context/container, cấu trúc repo, giao thức Guacamole, 10 luồng kỹ thuật F1–F10 có Mermaid**, class diagram API extension, đánh giá kiến trúc, ADR đề xuất |
| [03-VAN-DE-VA-RUI-RO.md](03-VAN-DE-VA-RUI-RO.md) | Bảng 76 vấn đề + chi tiết bằng chứng, nguyên nhân, ảnh hưởng, hướng xử lý |
| [04-BAO-MAT.md](04-BAO-MAT.md) | Tài sản, actor, điểm vào, ranh giới tin cậy (Mermaid), STRIDE, OWASP, công bố có trách nhiệm |
| [05-MA-DOC-VA-CHUOI-CUNG-UNG.md](05-MA-DOC-VA-CHUOI-CUNG-UNG.md) | Kết quả quét mã độc theo từng mục, chuỗi build Docker |
| [06-BACKEND.md](06-BACKEND.md) | guacd/libguac, protocol plugin, web app Java, extension |
| [07-FRONTEND.md](07-FRONTEND.md) | guacamole-common-js và SPA AngularJS, luồng xử lý instruction |
| [08-DATABASE.md](08-DATABASE.md) | ER diagram, mô hình quyền (flowchart), đánh giá |
| [09-CHAT-LUONG-CODE.md](09-CHAT-LUONG-CODE.md) | Số liệu, smell, đặt tên, công cụ |
| [10-KIEM-THU-TAI-LIEU-VAN-HANH.md](10-KIEM-THU-TAI-LIEU-VAN-HANH.md) | Test, CI, Docker, env, logging, docs |
| [11-LO-TRINH-CAI-THIEN.md](11-LO-TRINH-CAI-THIEN.md) | Ngay / trước release / phiên bản sau / khi tăng trưởng / không nên làm |
| [12-BANG-DIEM.md](12-BANG-DIEM.md) | Bảng điểm 10 nhóm, kết luận release và điều kiện |
| `summary.json` | Kết quả máy đọc được |

**Đọc nhanh luồng hoạt động:** bắt đầu từ `02-KIEN-TRUC-HE-THONG.md`:
- §4: giao thức;
- §8 (F4): mở phiên end-to-end;
- §9 (F5): process model guacd;
- §10 (F6): vẽ màn hình và backpressure.
