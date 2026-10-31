# 03/04 · Plugin giao thức (RDP, VNC, SSH, Telnet, Kubernetes)

> Ai đọc: dev sửa hành vi một giao thức, thêm tham số, hoặc thêm giao thức mới.
> Đường dẫn tính từ `guacamole-server/src/protocols/` trừ khi ghi khác.
>
> Trước: [03-libguac-va-giao-thuc.md](03-libguac-va-giao-thuc.md) · Tiếp: [05-build-va-test.md](05-build-va-test.md)

## Khung chung của một plugin

Mỗi plugin là một thư viện `libguac-client-<p>.la` (cài ra `libguac-client-<p>.so`)
export **một** hàm `guac_client_init`. guacd nạp bằng
`dlopen("libguac-client-<p>.so")` + `dlsym("guac_client_init")` (`src/libguac/client.c`).

| File (có ở hầu hết plugin) | Vai trò |
| --- | --- |
| `client.c` | `guac_client_init()`: gán `client->args = GUAC_<P>_CLIENT_ARGS`, cấp phát `client->data`, gán `join_handler`, `join_pending_handler`, `leave_handler`, `free_handler`. |
| `settings.c` / `settings.h` | Mảng tên tham số `GUAC_<P>_CLIENT_ARGS[]` + `enum` chỉ số `IDX_*` cùng thứ tự + hàm `guac_<p>_parse_args()` đọc giá trị bằng `guac_user_parse_args_string/int/boolean…`. |
| `user.c` | `join_handler`: parse tham số; nếu là **owner** thì khởi động thread kết nối (vd. `guac_vnc_client_thread`, tên thread `"vnc-client"`); gán handler nhập liệu nếu không `read-only`. |
| `<p>.c` (`rdp.c`, `vnc.c`, `ssh.c`…) | Vòng lặp chính: kết nối tới máy đích, nhận dữ liệu, vẽ lên `guac_display`. |
| `input.c`, `clipboard.c` | Chuột / phím / clipboard từ user → máy đích. |
| `argv.c` | Đổi tham số lúc đang chạy (lệnh `argv`), ví dụ nhập mật khẩu khi được hỏi (`required`). |
| `pipe.c`, `sftp.c` | Named pipe và truyền file qua SFTP. |

Ví dụ rút gọn `vnc/client.c` + `vnc/user.c`:

```c
int guac_client_init(guac_client* client) {
    client->args = GUAC_VNC_CLIENT_ARGS;            /* gửi trong lệnh "args" */
    client->data = guac_mem_zalloc(sizeof(guac_vnc_client));
    client->join_handler  = guac_vnc_user_join_handler;
    client->leave_handler = guac_vnc_user_leave_handler;
    client->free_handler  = guac_vnc_client_free_handler;
    return 0;
}

int guac_vnc_user_join_handler(guac_user* user, int argc, char** argv) {
    guac_vnc_settings* settings = guac_vnc_parse_args(user, argc, (const char**) argv);
    if (settings == NULL) return 1;                  /* "Badly formatted client arguments." */
    if (user->owner)                                 /* chỉ owner mở kết nối thật */
        guac_thread_create(&vnc_client->client_thread, guac_vnc_client_thread,
                user->client, "vnc-client");
    if (!settings->read_only) {
        user->mouse_handler = guac_vnc_user_mouse_handler;
        user->key_handler   = guac_vnc_user_key_handler;
    }
    return 0;
}
```

## Từng plugin

| Plugin | Thư viện ngoài | Số tham số (`*_CLIENT_ARGS`) | Điểm riêng |
| --- | --- | --- | --- |
| `rdp/` | FreeRDP **2** (ưu tiên) hoặc **3** (configure in cảnh báo "experimental") | 84 | `channels/` (cliprdr, disp, rail, rdpei, rdpgfx, rdpdr, rdpsnd, audio-input), `plugins/` (thư viện phụ `libguac-common-svc-client`, `libguacai-client` nạp như plugin FreeRDP), `keymaps/*.keymap` (bố cục bàn phím), `fs.c` (ổ đĩa ảo), `print-job.c` (in ra PDF bằng ghostscript), `tests/`. `guac_client_init` khai báo `(guac_client*, int, char**)` khác chữ ký chuẩn `(guac_client*)` — quan sát khi đọc code, chưa kiểm chứng ảnh hưởng. |
| `vnc/` | libvncclient (LibVNCServer) | 51 | `auth.c`, `cursor.c`, `display.c`; âm thanh qua `src/pulse` nếu có PulseAudio; reverse connect (`reverse-connect`, `listen-timeout`). |
| `ssh/` | libssh2 + OpenSSL, terminal (Pango) | 43 | `ssh_agent.c` (chỉ khi `--enable-ssh-agent`), `ttymode.c`, `sftp.c`. Dùng `src/terminal/`. |
| `telnet/` | libtelnet, terminal | 34 | Tự điền đăng nhập theo regex: `username-regex`, `password-regex`, `login-success-regex`, `login-failure-regex`. |
| `kubernetes/` | libwebsockets + OpenSSL, terminal | 31 | Exec / attach vào container qua API Kubernetes (`io.c`, `ssl.c`, `url.c`, `tests/`). |

Tham số hay dùng (tên chính xác lấy từ `settings.c`; ý nghĩa đầy đủ xem manual
chương "Configuring Guacamole → Connection configuration"):

| Plugin | Ví dụ tham số |
| --- | --- |
| RDP | `hostname`, `port`, `username`, `password`, `domain`, `security`, `ignore-cert`, `width`, `height`, `dpi`, `color-depth`, `resize-method`, `enable-drive`, `drive-path`, `enable-printing`, `remote-app`, `gateway-hostname`, `enable-sftp`, `recording-path`, `recording-name`, `wol-send-packet`, `force-lossless` |
| VNC | `hostname`, `port`, `username`, `password`, `read-only`, `encodings`, `color-depth`, `cursor`, `autoretry`, `dest-host`, `dest-port`, `reverse-connect`, `enable-audio`, `enable-sftp`, `compress-level`, `quality-level`, `disable-server-input` |
| SSH | `hostname`, `host-key`, `port`, `username`, `password`, `private-key`, `passphrase`, `public-key`, `enable-agent`, `command`, `enable-sftp`, `typescript-path`, `recording-path`, `terminal-type`, `scrollback`, `locale`, `server-alive-interval` |
| Kubernetes | `hostname`, `port`, `namespace`, `pod`, `container`, `exec-command`, `use-ssl`, `client-cert`, `client-key`, `ca-cert`, `ignore-cert` |

## Thêm một tham số cho giao thức có sẵn

Tên tham số là hợp đồng giữa 2 repo; phải sửa **cả hai phía** trong cùng thay đổi:

1. **Server** — `src/protocols/<p>/settings.c`:
   - thêm chuỗi tên vào `GUAC_<P>_CLIENT_ARGS[]` (trước `NULL`),
   - thêm `IDX_<TÊN>` vào `enum` **đúng cùng vị trí** (lệch vị trí → mọi tham số sau đọc sai giá trị),
   - thêm trường vào struct trong `settings.h`, đọc trong `guac_<p>_parse_args()`, giải phóng nếu là chuỗi (`guac_<p>_settings_free`).
2. **Client** — `guacamole-client/guacamole-ext/src/main/resources/org/apache/guacamole/protocols/<p>.json`:
   thêm field (`name` trùng tên ở bước 1, `type`: `TEXT`, `NUMERIC`, `BOOLEAN`, `ENUM`, `PASSWORD`, `USERNAME`, `MULTILINE`, `TIMEZONE`…) vào form phù hợp.
3. **Bản dịch** — `guacamole-client/guacamole/src/main/frontend/src/translations/en.json`, mục `"PROTOCOL_<P>"`:
   thêm `FIELD_HEADER_<TÊN>` (tên viết hoa, `-` → `_`). Ngôn ngữ khác có thể thêm sau.
4. Kiểm tra: build cả hai, tạo connection có tham số mới, bật `LOG_LEVEL=debug`
   cho guacd → log của plugin phản ánh giá trị. Nếu field không hiện trên form → sai
   bước 2; hiện nhưng không tác dụng → sai tên hoặc thứ tự ở bước 1.

Không cần sửa extension JDBC: tham số connection lưu dạng key–value
(`guacamole_connection_parameter`), không có cột cố định.

## Thêm một giao thức mới

1. Tạo `src/protocols/<p>/` theo khung ở trên (copy `telnet/` nếu là giao thức dạng text, `vnc/` nếu đồ họa).
2. `Makefile.am` của plugin: `lib_LTLIBRARIES = libguac-client-<p>.la`, `*_LIBADD = @COMMON_LTLIB@ @LIBGUAC_LTLIB@ …`.
3. `configure.ac`: thêm `--with-<p>`, kiểm tra thư viện, `AM_CONDITIONAL([ENABLE_<P>], …)`, thêm `src/protocols/<p>/Makefile` vào `AC_CONFIG_FILES`, thêm dòng vào bảng tóm tắt cuối file.
4. `Makefile.am` gốc: `DIST_SUBDIRS` + khối `if ENABLE_<P> SUBDIRS += src/protocols/<p> endif`.
5. Phía client (`guacamole-ext/.../environment/LocalEnvironment.java`, `readProtocols()`): web app đọc mọi file `GUACAMOLE_HOME/protocols/*.json`, sau đó bổ sung các giao thức trong mảng `KNOWN_PROTOCOLS` từ classpath `/org/apache/guacamole/protocols/<p>.json`. Hai cách:
   - **không build lại web app**: đặt `<p>.json` vào `GUACAMOLE_HOME/protocols/`;
   - **đưa vào upstream**: thêm `<p>.json` vào `guacamole-ext/src/main/resources/org/apache/guacamole/protocols/` và tên `<p>` vào `KNOWN_PROTOCOLS`.

   Kèm mục `PROTOCOL_<P>` trong bản dịch (hoặc trong `translations/` của một extension).
6. Kiểm tra: `ls $(prefix)/lib/libguac-client-<p>.so`; tạo connection giao thức `<p>` → guacd log `Creating new client for protocol "<p>"` và **không** có `is not installed`.

## Công cụ trong phiên terminal: `guacctl` / `guacget`

`bin/guacctl` là shell script chạy **bên trong phiên SSH** (trên máy đích), in mã
escape OSC mà terminal Guacamole hiểu:

| Lệnh | Mã OSC | Tác dụng |
| --- | --- | --- |
| `guacctl -d FILE…` / `guacget FILE…` | 482200 | Tải file về trình duyệt |
| `guacctl -s DIR` | 482201 | Đặt thư mục nhận file upload |
| `guacctl -o NAME` / `-c` | 482202 / 482203 | Mở / đóng named pipe |
| `guacctl -S ROWS` | 482204 | Đổi số dòng scrollback |

`guacget` là symlink tới `guacctl`; script chọn chế độ theo tên gọi (suy từ tên,
chưa đọc kỹ nhánh xử lý).
