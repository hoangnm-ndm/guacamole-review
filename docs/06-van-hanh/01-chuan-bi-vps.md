# 06/01 · Chuẩn bị VPS, DNS và Docker

> Ai đọc: người dựng hệ thống, làm **một lần** trước [02-trien-khai-docker-compose.md](02-trien-khai-docker-compose.md).
>
> Trước: [README](README.md) · Tiếp: [02-trien-khai-docker-compose.md](02-trien-khai-docker-compose.md)

> **Phương án chuẩn, chưa kiểm chứng trên hệ thống thật** (chưa có VPS, 2026-10-31).
> Lệnh hệ điều hành viết cho **Ubuntu 24.04 LTS**; distro khác tên gói / lệnh có thể khác.

## 0. Quyết định trước khi thuê VPS: guacd có tới được máy đích không?

Guacamole chỉ là cổng; **guacd (trên VPS) phải mở được TCP tới máy đích** (RDP 3389,
SSH 22, VNC 5900…). Xác định trước:

| Máy đích ở đâu | Cách để guacd tới được | Ghi chú |
| --- | --- | --- |
| Có IP public, mở cổng | Trực tiếp | **Chỉ cho phép IP của VPS** trên firewall máy đích — không mở RDP ra Internet |
| Trong mạng LAN nhà / công ty (sau NAT) | VPN từ VPS vào LAN (WireGuard, Tailscale…) hoặc chạy Guacamole **trong** LAN thay vì VPS | Ngoài phạm vi docs này; thêm phương án riêng khi chọn |
| Cùng VPS / cùng nhà cung cấp (mạng private) | IP private | |

Kiểm tra sau khi có VPS: `nc -zv -w 5 <IP-máy-đích> 3389` chạy **trên VPS** → `succeeded` / `open`.

## 1. Cấu hình VPS

| Mục | Tối thiểu | Khuyến nghị | Lý do |
| --- | --- | --- | --- |
| CPU | 1 vCPU | 2 vCPU | guacd nén ảnh (PNG/JPEG/WebP) cho mỗi phiên đồ họa — tốn CPU nhất |
| RAM | 2 GB | 4 GB | Tomcat/JVM (~0,5–1 GB) + PostgreSQL + mỗi phiên RDP một process guacd (ước lượng, chưa đo) |
| Ổ đĩa | 20 GB | 40 GB+ nếu ghi phiên | Recording lớn dần theo thời lượng phiên |
| OS | Ubuntu 22.04 / 24.04 LTS | 24.04 LTS | Docker hỗ trợ chính thức |
| Mạng | IPv4 public | | Let's Encrypt cần cổng 80 / 443 từ Internet |

Con số RAM / CPU là ước lượng — sau khi chạy thật, đo bằng `docker stats` và ghi lại ở
[README § Hiện trạng](README.md#hiện-trạng).

## 2. Bảo mật cơ bản (trước khi cài gì khác)

| # | Làm gì | Kiểm tra | Nếu sai |
| --- | --- | --- | --- |
| 1 | Tạo user sudo: `adduser <user> && usermod -aG sudo <user>` | `sudo -l -U <user>` liệt kê `(ALL : ALL) ALL` | |
| 2 | Từ máy bạn: `ssh-copy-id <user>@<IP-VPS>` | `ssh <user>@<IP-VPS>` vào không hỏi mật khẩu | Kiểm tra `~/.ssh/authorized_keys` quyền `600` |
| 3 | **Chỉ khi bước 2 chạy**: sửa `/etc/ssh/sshd_config` → `PasswordAuthentication no`, `PermitRootLogin no`; `sudo systemctl reload ssh` | Mở **terminal mới**: `ssh -o PubkeyAuthentication=no <user>@<IP-VPS>` → `Permission denied (publickey)` | Giữ phiên SSH cũ mở đến khi kiểm tra xong — sai cấu hình mà đóng phiên là mất quyền truy cập (phải vào console nhà cung cấp) |
| 4 | Firewall: `sudo ufw allow OpenSSH && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp && sudo ufw enable` | `sudo ufw status` → chỉ 22, 80, 443 | |
| 5 | Cập nhật tự động: `sudo apt install unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades` | `systemctl status unattended-upgrades` → active | |

**Docker vượt qua ufw**: cổng container publish (`ports:`) mở ra Internet **dù ufw không
cho phép**. Vì vậy phương án chuẩn chỉ publish 80/443 của Caddy; guacd / Tomcat / Postgres
không publish. Kiểm tra cổng thật sự mở từ **máy ngoài**, không tin `ufw status`
(bảng ở [02 § Bước 10](02-trien-khai-docker-compose.md#bước-10--kiểm-tra-sau-triển-khai)).

## 3. Cài Docker Engine + Compose plugin

Làm theo hướng dẫn chính thức (repo APT của Docker, không dùng gói `docker.io` cũ của Ubuntu):
<https://docs.docker.com/engine/install/ubuntu/>. Sau đó:

```bash
sudo usermod -aG docker "$USER"   # đăng xuất / đăng nhập lại để có hiệu lực
docker version --format '{{.Server.Version}}'
docker compose version
```

Mong đợi: in phiên bản Docker Engine và `Docker Compose version v2.x`. Lỗi
`permission denied … docker.sock` → chưa đăng nhập lại sau `usermod`.

(Thêm user vào group `docker` ≈ cho quyền root; chấp nhận được trên VPS một người quản trị.)

## 4. DNS

Tạo **một bản ghi** ở nhà cung cấp DNS của tên miền:

| Loại | Tên | Giá trị | TTL | Proxy / CDN |
| --- | --- | --- | --- | --- |
| `A` | `remote` (→ `remote.example.com`) | `<IP-VPS>` | 300 | **Tắt** (DNS only) khi dựng lần đầu |

Kiểm tra (từ máy bạn): `dig +short A remote.example.com` → đúng `<IP-VPS>`.
Ra IP khác → đang bật proxy / CDN (Cloudflare "Proxied"…), hoặc bản ghi chưa lan truyền
(chờ hết TTL). Nhà cung cấp DNS nào đang giữ tên miền: `dig +short NS example.com`.

Vì sao tắt proxy lần đầu: Caddy xin chứng chỉ Let's Encrypt qua HTTP-01 / TLS-ALPN-01, cần
Let's Encrypt kết nối thẳng tới VPS. Bật proxy CDN sau đó là **phương án riêng** (cần cấu
hình SSL mode, WebSocket qua CDN, `REMOTE_IP_VALVE_INTERNAL_PROXIES`/header IP thật của CDN) —
chưa viết; thêm file `06-…` khi chọn.

Giá trị phụ thuộc tên miền (đổi tên miền thì phải sửa cùng lúc):

| Đổi tên miền → sửa | Ở đâu |
| --- | --- |
| `GUAC_DOMAIN` | `/opt/guacamole/.env`, rồi `docker compose up -d caddy` |
| Redirect URI của SSO (nếu dùng OpenID / SAML / Duo) | Biến `OPENID_REDIRECT_URI`, `SAML_CALLBACK_URL`, `DUO_REDIRECT_URI` **và** cấu hình phía IdP |
| Link đã gửi cho người dùng, bookmark | |

## 5. Xong bước chuẩn bị khi

- [ ] SSH bằng key, mật khẩu bị tắt
- [ ] `ufw status` chỉ 22 / 80 / 443
- [ ] `docker compose version` chạy không cần `sudo`
- [ ] `dig +short A <GUAC_DOMAIN>` = IP VPS
- [ ] Từ VPS `nc -zv <máy-đích> <cổng>` thông

→ sang [02-trien-khai-docker-compose.md](02-trien-khai-docker-compose.md).
