# 06/04 · Sao lưu, khôi phục và xử lý sự cố

> Ai đọc: người vận hành — khi đặt lịch backup, khi cần khôi phục, khi hệ thống lỗi.
> Đang có sự cố → nhảy thẳng tới [Bảng sự cố](#bảng-sự-cố).
>
> Trước: [03-van-hanh-hang-ngay.md](03-van-hanh-hang-ngay.md) · Tiếp: [05-bien-moi-truong.md](05-bien-moi-truong.md)

## Cần sao lưu gì

| Dữ liệu | Ở đâu | Quan trọng | Mất thì |
| --- | --- | --- | --- |
| DB (user, connection + **mật khẩu máy đích**, quyền, lịch sử) | container `postgres` (`data/postgres`) | **Cao nhất** | Mất toàn bộ cấu hình |
| `.env`, `compose.yaml`, `Caddyfile`, `guacamole-home/` (nếu có) | `/opt/guacamole` | Cao | Dựng lại được theo docs, nhưng mất mật khẩu DB |
| Recording | `data/recordings` | Tùy chính sách | Mất bản ghi phiên |
| Chứng chỉ Caddy | volume `caddy_data` | Thấp | Caddy tự xin lại (giới hạn tần suất Let's Encrypt nếu xin lại nhiều lần) |

Không sao lưu `data/postgres` bằng cách copy thư mục khi DB đang chạy — dùng `pg_dump`.

## Sao lưu

Script `/opt/guacamole/backup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
cd /opt/guacamole
mkdir -p backups
STAMP=$(date +%Y%m%d-%H%M%S)
docker compose exec -T postgres pg_dump -U guacamole_user -d guacamole_db -Fc > "backups/guacamole_db-$STAMP.dump"
tar -czf "backups/config-$STAMP.tar.gz" .env compose.yaml Caddyfile
# Giữ 14 bản gần nhất
ls -1t backups/guacamole_db-*.dump | tail -n +15 | xargs -r rm --
ls -1t backups/config-*.tar.gz      | tail -n +15 | xargs -r rm --
```

```bash
chmod 700 backup.sh
( crontab -l 2>/dev/null; echo '30 2 * * * /opt/guacamole/backup.sh >> /opt/guacamole/backups/backup.log 2>&1' ) | crontab -
```

Kiểm tra ngay: `./backup.sh && ls -lh backups/` → có file `.dump` vài trăm KB trở lên
(DB mới gần như rỗng thì nhỏ). File 0 byte → `pg_dump` lỗi, xem `backups/backup.log`.

**Đưa backup ra khỏi VPS** (backup cùng VPS mất cùng VPS): đồng bộ `backups/` sang
object storage / máy khác (rclone, restic, `scp` theo cron). Chọn công cụ nào chưa chốt —
ghi vào [README § Hiện trạng](README.md#hiện-trạng) khi làm. File backup chứa mật khẩu
máy đích (xem [03](03-van-hanh-hang-ngay.md#quản-lý-người-dùng)) → mã hóa khi lưu ngoài.

## Khôi phục

> **Ghi đè dữ liệu hiện tại.** Chỉ làm khi đã có bản `pg_dump` mới nhất của trạng thái hiện
> tại (chạy `./backup.sh` trước) hoặc DB hiện tại đã hỏng hẳn.

```bash
cd /opt/guacamole
docker compose stop guacamole
docker compose exec -T postgres pg_restore -U guacamole_user -d guacamole_db --clean --if-exists --no-owner < backups/guacamole_db-<STAMP>.dump
docker compose start guacamole
```

Kiểm tra: đăng nhập bằng user có trong bản backup; số connection khớp. Lỗi `pg_restore`
dạng `role … does not exist` → bỏ qua được nhờ `--no-owner`; lỗi khác → dừng, giữ nguyên
`guacamole` tắt, kiểm tra file dump (`pg_restore -l <file> | head`).

**Chuyển sang VPS mới**: làm [01](01-chuan-bi-vps.md) → [02](02-trien-khai-docker-compose.md)
bước 1–5 (dùng lại `.env` cũ) → khôi phục như trên → bước 6 trở đi → đổi DNS sang IP mới.

### Khôi phục thử định kỳ

Backup chưa khôi phục thử = chưa có backup. Mỗi quý, trên VPS (không đụng DB thật):

```bash
docker run -d --name pg-restore-test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=guacamole_db -e POSTGRES_USER=guacamole_user postgres:16
sleep 10
docker exec -i pg-restore-test pg_restore -U guacamole_user -d guacamole_db --no-owner < backups/<file-mới-nhất>.dump
docker exec pg-restore-test psql -U guacamole_user -d guacamole_db -c 'SELECT count(*) FROM guacamole_connection;'
docker rm -f pg-restore-test
```

Mong đợi: số connection khớp với giao diện. Ghi ngày khôi phục thử vào [README § Hiện trạng](README.md#hiện-trạng).

## Bảng sự cố

| Hiện tượng | Nguyên nhân / kiểm tra | Cách xử lý |
| --- | --- | --- |
| Trang web không mở, trình duyệt báo lỗi chứng chỉ / không kết nối | `docker compose logs caddy` có lỗi ACME; `dig +short A <domain>` sai IP; cổng 80/443 bị chặn ở firewall nhà cung cấp | Sửa DNS ([01 § DNS](01-chuan-bi-vps.md#4-dns)), mở 80/443, `docker compose restart caddy` |
| `502 Bad Gateway` từ Caddy | Container `guacamole` chưa lên / đang khởi động Tomcat (vài chục giây) / crash | `docker compose ps`, `docker compose logs --tail 100 guacamole` |
| Trang `/` báo 404, nhưng `/guacamole/` vào được | Thiếu `WEBAPP_CONTEXT: ROOT` | Thêm biến, `docker compose up -d guacamole` |
| Đăng nhập `guacadmin` lần đầu báo sai mật khẩu; log web app có `relation "guacamole_user" does not exist` hoặc lỗi SQL | **DB chưa có schema** — script init không chạy vì `data/postgres` không rỗng lúc postgres khởi động lần đầu | DB còn trống (mới dựng): `docker compose down`, **xóa** `data/postgres/*` (chỉ khi chắc chắn không có dữ liệu cần giữ), kiểm tra `init/initdb.sql`, làm lại [02 bước 5](02-trien-khai-docker-compose.md#bước-5--khởi-động-postgresql-trước). Hoặc chạy tay: `docker compose exec -T postgres psql -U guacamole_user -d guacamole_db < init/initdb.sql` |
| Log web app `password authentication failed for user "guacamole_user"` | Mật khẩu trong `.env` khác mật khẩu đã khởi tạo DB (đã đổi `.env` sau lần đầu) | [03 § Đổi mật khẩu DB](03-van-hanh-hang-ngay.md#đổi-mật-khẩu-db) |
| Sau vài lần ai đó gõ sai, **mọi người** bị chặn đăng nhập ~5 phút | Extension ban thấy mọi request đến từ IP của Caddy — thiếu `REMOTE_IP_VALVE_ENABLED=true` hoặc IP Caddy không nằm trong `internalProxies` | Thêm biến; nếu đã có: xem IP thật trong Settings → History; Docker network dùng dải lạ thì đặt `REMOTE_IP_VALVE_INTERNAL_PROXIES` |
| Đổi biến trong `.env` / `compose.yaml` mà không có tác dụng | Đã dùng `docker compose restart`; hoặc key nằm trong `guacamole.properties` template | `docker compose up -d`; xem [05 § Cơ chế](05-bien-moi-truong.md#cơ-chế--đọc-trước-khi-sửa-biến) |
| Extension LDAP / TOTP… tự bật dù không muốn | Còn biến tiền tố đó trong `environment` (kể cả giá trị rỗng) | Xóa biến, hoặc `<TIỀN_TỐ>ENABLED: "false"` |
| Mở connection: "**Connection error** / The remote desktop server is currently unreachable" | guacd không tới được máy đích | `docker compose exec guacd nc -zv <host> <port>`; mở firewall máy đích cho IP VPS; kiểm tra hostname phân giải được từ container |
| Mở connection lỗi ngay, log web app `Connection refused` tới `guacd:4822` | Sai `GUACD_HOSTNAME` hoặc guacd không chạy | `docker compose ps guacd`; `docker compose exec guacamole getent hosts guacd` (lệnh có thể không có trong image — chưa kiểm chứng) |
| RDP: lỗi xác thực / "security negotiation" ngay khi kết nối | Chế độ bảo mật RDP không khớp, chứng chỉ tự ký | Trong connection: *Security mode* `any` hoặc `nla`, tick *Ignore server certificate*; log guacd ở `debug` cho chi tiết |
| RDP màn hình đen / giật | Băng thông, độ sâu màu, CPU VPS | Giảm *Color depth*, tắt wallpaper / theme, `docker stats` xem CPU guacd |
| SSH: "Host key verification" / không vào | `host-key` đặt sai | Để trống `host-key` hoặc đặt đúng khóa (`ssh-keyscan <host>`) |
| Không thấy nút xem lại recording | Thiếu `RECORDING_SEARCH_PATH`; recording path của connection không nằm dưới thư mục đó; web app không đọc được file (quyền) | `ls -ln data/recordings/*` → owner `1000`, group `1000`, quyền `rw-r-----`; container guacamole có `group_add: ["1000"]`; [02 bước 3](02-trien-khai-docker-compose.md#bước-3--thư-mục-dữ-liệu-và-quyền) |
| guacd log `Unable to create recording` / `Permission denied` | `data/recordings` không thuộc uid 1000 | `sudo chown -R 1000:1000 data/recordings` |
| Phiên chạy chậm, log trình duyệt thấy request `/tunnel` liên tục | WebSocket bị chặn, rơi về HTTP tunnel | Kiểm tra proxy trung gian / CDN có hỗ trợ WebSocket; với Caddy chuẩn thì WebSocket tự chạy |
| Bị đăng xuất hết sau khi cập nhật / restart | Bình thường — session lưu trong RAM web app | Báo trước người dùng khi bảo trì |
| Link chia sẻ phiên báo `No such connection` | Phiên gốc đã kết thúc | Bình thường |
| **Báo nhầm**: `docker compose ps` hiện `guacd … (health: starting)` vài phút | Healthcheck image chạy mỗi 5 phút | Không cần xử lý; kiểm tra thật: `docker compose exec guacd nc -z 127.0.0.1 4822 && echo OK` |
| **Báo nhầm**: log web app `Using default RemoteIpValve value for "…" attribute.` | Thông báo của entrypoint khi không đặt thuộc tính tùy chọn | Không cần xử lý |
| **Báo nhầm**: log `WARNING: The "…" environment variable has been deprecated` | Dùng tên biến cũ — vẫn chạy | Đổi sang tên mới theo [05 § Tên cũ](05-bien-moi-truong.md#tên-cũ-deprecated--entrypoint-tự-đổi-và-in-cảnh-báo) |

Sự cố thật đã gặp: thêm một dòng vào bảng trên (hiện tượng → nguyên nhân đã xác nhận →
cách sửa đã làm), ghi ngày ở cột nguyên nhân.

## Bật log chi tiết khi gỡ lỗi

```bash
sed -i 's/^GUACD_LOG_LEVEL_VALUE=.*/GUACD_LOG_LEVEL_VALUE=debug/; s/^GUACAMOLE_LOG_LEVEL=.*/GUACAMOLE_LOG_LEVEL=debug/' .env
docker compose up -d guacd guacamole
```

Gỡ lỗi xong đặt lại `info` (log `debug` có thể chứa tham số kết nối).
