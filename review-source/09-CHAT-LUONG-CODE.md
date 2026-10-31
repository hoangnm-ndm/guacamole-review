# 09 — Chất lượng code, đặt tên, nhất quán

## 1. Số liệu

Đo bằng `git ls-files` và `wc -l`, chỉ đọc. Không có công cụ đo độ phức tạp nào được cài sẵn trên máy review.

| Phần | Ngôn ngữ | Dòng (không test) | File test | Ghi chú |
|---|---|---|---|---|
| guacamole-server | C | 103.976 (`.c` + `.h`) | 57 file test `.c` | Lõi 36,8k · plugin, common, terminal… 69,3k |
| guacamole-client Java | Java | 111.433 | 19 | Lõi 55,4k · extension khoảng 56k |
| guacamole-client JS | JS | 61.442 (kể cả cấu hình) | 3 spec | common-js 17,9k · SPA 40,7k |
| SQL | SQL | 5.240 | — | Schema và upgrade cho 3 DB |

**File lớn nhất** (dòng thô; JS và Java có khoảng 50% là doc comment):

| C | Java | JS |
|---|---|---|
| `terminal/terminal.c` 3250 | `DirectoryResource` 873 | `Display.js` 2143 (569 dòng code) |
| `common/surface.c` 2037 | `AbstractGuacamoleTunnelService` (jdbc) 827 | `Client.js` 2098 (851 dòng code) |
| `rdp/settings.c` 2023 | `KsmClient` 794 | `Keyboard.js` 1975 (706 dòng code) |
| `terminal/terminal-handlers.c` 1685 | `ModeledUser` 791 | `ManagedClient.js` 1147 (421 dòng code) |
| `libguac/protocol.c` 1440 | `ExtensionModule` 627 | `ColorScheme.js` 949 |

## 2. Điểm mạnh

- **Tài liệu inline rất dày và chất lượng:**
  - Mọi hàm public trong `libguac/guacamole/*.h` có doc comment đủ tham số và giá trị trả về.
  - JSDoc chiếm khoảng 54% dòng JS.
  - Javadoc đầy đủ cho API extension.
- **Quy ước đặt tên nhất quán theo ngôn ngữ:**
  - C dùng tiền tố `guac_<module>_<action>`, hàm nội bộ có tiền tố `__guac_`.
  - Java dùng camelCase và hậu tố có nghĩa (`*Service`, `*Resource`, `*Mapper`, `Modeled*`).
  - JS dùng namespace `Guacamole.*`.
- **Build C nghiêm:** `-Werror -Wall -pedantic` cho guacd và libguac.
- **Ít magic value:**
  - Hằng có tên: `GUAC_INSTRUCTION_MAX_LENGTH`, `GUACD_TIMEOUT`, `GUAC_RDP_FS_MAX_FILES`.
  - Status code tập trung trong `GuacamoleStatus` (Java) và `Guacamole.Status` (JS).
  - Status code được dùng lại xuyên ba ngôn ngữ với cùng giá trị.
- **Ít hàm nguy hiểm:** API cấp phát kiểm tràn `guac_mem_*` có test riêng; dùng `guac_strlcpy`.
- **Không có code bẩn đáng kể:** không thấy log debug sót, đoạn code bị comment-out hàng loạt hay file tạm. Có một FIXME đã ghi rõ (`S/guacd/daemon.c:746`).

## 3. Code smell và vấn đề (tham chiếu `03-VAN-DE-VA-RUI-RO.md`)

| Smell (`clean-code/code-smells.md`) | Biểu hiện | Mã |
|---|---|---|
| Copy-paste (DRY) | Parse settings recording/SFTP/WOL ở 5 plugin; mapper ở 3 DB; hằng token ở 2 module | `ARCH-01`, `ARCH-02`, `CODE-06` |
| God function / nhánh trùng | `DirectoryResource.patchObjects` ~240 dòng; `guac_rdp_push_settings` ~470 dòng với hai nhánh | `ARCH-04` |
| Shared mutable state không đồng bộ | broadcast lock, `guac_rdp_fs`, `client->state`, field trong `GuacamoleSession` | `CODE-01` |
| Static mutable state | Terminal parser dùng biến `static` (`terminal-handlers.c:238, 715-723, 1319, 1464`). Chỉ an toàn vì mỗi process có một terminal | (ghi nhận, nằm trong `CODE-01`) |
| Dead code | `ssh_agent.c` không biên dịch được; `GUAC_INSTRUCTION_MAX_DIGITS` không được dùng | `CODE-02` |
| Nuốt lỗi | Lỗi base64 bị bỏ qua (`S/libguac/socket.c:381-383`); bắt cả `Error` trong Java; `pa_stream_peek` bỏ qua kết quả | `ERR-04`, `ERR-05` |
| Undefined behavior | signed overflow khi parse số | `CODE-03` |
| Single-handler callback | `on*` chỉ nhận một handler nên ChainedTunnel phải ghi đè dây chuyền (`CJ/Tunnel.js:1122-1143`) | (thiết kế cũ, ghi nhận) |

## 4. Đặt tên và nhất quán

| Mã | Vấn đề |
|---|---|
| `NAME-01` | `child_socket`/`parent_socket` đặt ngược vai (`S/guacd/proc.c:563-583, 656`) |
| `NAME-02` | JS trộn `snake_case` (`guac_client`, `close_tunnel`) với camelCase; chỉ `Keyboard.js` dùng ES6 |
| `NAME-03` | Comment nói sai hành vi: known_hosts ghi "GUACAMOLE_HOME" nhưng code hard-code `/etc/guacamole`; Javadoc `TunnelConnectEvent` nói tunnel bị đóng nhưng không |

"Một việc, một cách" — các điểm chưa nhất quán:
- **Token đi ba đường:** header ở REST, query ở WebSocket/stream, POST body ở HTTP tunnel (`SEC-09`).
- **Lọc instruction từ client:** WebSocket chặn opcode rỗng, HTTP tunnel ghi raw.
- **Hai stack render C:** `guac_display` và `guac_common_surface`.
- **Hai kiểu đọc setting của FreeRDP:** getter/setter và truy cập struct trực tiếp.

## 5. Công cụ giữ chuẩn

| Công cụ | Có? |
|---|---|
| Formatter C/Java/JS (clang-format, google-java-format, prettier) | Không |
| Linter JS (ESLint) | Không |
| Phân tích tĩnh Java (SpotBugs, PMD, Checkstyle) | Không |
| Phân tích tĩnh C (clang-tidy, cppcheck); Coverity | Có chú thích Coverity trong code (`rwlock.c:28`), nhưng repo không có cấu hình chạy |
| `apache-rat` (kiểm license header) | Có |
| `.editorconfig` | Không |

Quy ước được giữ chủ yếu bằng review của maintainer. Việc này hiệu quả với lõi do vài người viết lâu năm. Nhưng code từ người đóng góp lẻ (extension, bản dịch) không được máy kiểm tra (`OPS-02`).
