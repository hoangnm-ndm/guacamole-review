# 02 — Kiến trúc hệ thống và luồng kỹ thuật

> Phạm vi: `guacamole-client` @ `31d2a6f6a` (1.6.1) và `guacamole-server` @ `5f9bb3b4` (1.6.1).
> Ký hiệu đường dẫn dùng trong file:
>
> | Viết tắt | Đường dẫn thật |
> |---|---|
> | `S/` | `guacamole-server/src/` |
> | `C/` | `guacamole-client/guacamole-common/src/main/java/org/apache/guacamole/` |
> | `E/` | `guacamole-client/guacamole-ext/src/main/java/org/apache/guacamole/` |
> | `W/` | `guacamole-client/guacamole/src/main/java/org/apache/guacamole/` |
> | `CJ/` | `guacamole-client/guacamole-common-js/src/main/webapp/modules/` |
> | `FE/` | `guacamole-client/guacamole/src/main/frontend/src/app/` |
> | `J/` | `guacamole-client/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-base/src/main/java/org/apache/guacamole/auth/jdbc/` |

Mục lục:
1. [Bức tranh tổng thể (C4 context)](#1-bức-tranh-tổng-thể-c4--context)
2. [Container và hợp đồng giữa các thành phần](#2-container-và-hợp-đồng-giữa-các-thành-phần)
3. [Cấu trúc repo và phụ thuộc module](#3-cấu-trúc-repo-và-phụ-thuộc-module)
4. [Giao thức Guacamole — "ngôn ngữ chung" của cả hệ thống](#4-giao-thức-guacamole--ngôn-ngữ-chung-của-cả-hệ-thống)
5. [Luồng F1 — Khởi động web app và nạp extension](#5-luồng-f1--khởi-động-web-app-và-nạp-extension)
6. [Luồng F2 — Đăng nhập, MFA và vòng đời auth token](#6-luồng-f2--đăng-nhập-mfa-và-vòng-đời-auth-token)
7. [Luồng F3 — Phân quyền REST](#7-luồng-f3--phân-quyền-rest)
8. [Luồng F4 — Mở một phiên remote desktop end-to-end](#8-luồng-f4--mở-một-phiên-remote-desktop-end-to-end)
9. [Luồng F5 — Mô hình process của guacd](#9-luồng-f5--mô-hình-process-của-guacd)
10. [Luồng F6 — Trong phiên: vẽ màn hình, sync/backpressure, input](#10-luồng-f6--trong-phiên-vẽ-màn-hình-syncbackpressure-input)
11. [Luồng F7 — Chia sẻ phiên và join](#11-luồng-f7--chia-sẻ-phiên-và-join)
12. [Luồng F8 — Truyền file, clipboard, in ấn](#12-luồng-f8--truyền-file-clipboard-in-ấn)
13. [Luồng F9 — Ghi phiên và phát lại](#13-luồng-f9--ghi-phiên-và-phát-lại)
14. [Luồng F10 — Kết thúc phiên và dọn dẹp](#14-luồng-f10--kết-thúc-phiên-và-dọn-dẹp)
15. [API extension (class diagram)](#15-api-extension-class-diagram)
16. [Đánh giá kiến trúc](#16-đánh-giá-kiến-trúc)
17. [ADR đề xuất](#17-adr-đề-xuất)

---

## 1. Bức tranh tổng thể (C4 — context)

Apache Guacamole là **remote desktop gateway "clientless"**: người dùng chỉ cần trình duyệt HTML5. Họ điều khiển máy RDP/VNC/SSH/Telnet/Kubernetes nằm sau gateway mà không cần cài client.

Ý tưởng cốt lõi là **dịch giao thức**:
- JavaScript không xử lý hiệu quả các giao thức nhị phân như RDP/VNC. Vì vậy dự án tự định nghĩa một giao thức text đơn giản là *Guacamole protocol*.
- `guacd` là proxy dịch từ RDP/VNC/SSH… sang Guacamole protocol. Theo chiều ngược lại, nó dịch lệnh vẽ thành instruction gửi về trình duyệt.

```mermaid
flowchart LR
    U["Người dùng<br/>(trình duyệt HTML5)"]
    A["Quản trị viên"]
    subgraph GW["Apache Guacamole gateway"]
        WEB["guacamole web app<br/>(Java, Tomcat)"]
        GUACD["guacd<br/>(C daemon)"]
    end
    IDP["Nguồn danh tính<br/>DB / LDAP / SAML / OIDC / CAS / RADIUS / Duo"]
    DB[("Database<br/>MySQL / PostgreSQL / SQL Server")]
    VAULT["Vault (Keeper KSM)"]
    T1["Máy đích RDP"]
    T2["Máy đích VNC"]
    T3["Máy đích SSH / Telnet"]
    T4["Kubernetes API (attach pod)"]

    U -- "HTTPS: REST + WebSocket/HTTP tunnel" --> WEB
    A -- "HTTPS: quản trị user, connection, quyền" --> WEB
    WEB -- "xác thực" --> IDP
    WEB -- "JDBC (MyBatis)" --> DB
    WEB -- "lấy secret" --> VAULT
    WEB -- "Guacamole protocol qua TCP 4822 (TLS tùy chọn)" --> GUACD
    GUACD -- "RDP (FreeRDP)" --> T1
    GUACD -- "RFB (libvncclient)" --> T2
    GUACD -- "SSH (libssh2) / Telnet (libtelnet)" --> T3
    GUACD -- "WebSocket (libwebsockets)" --> T4
```

| Actor | Vai trò | Ghi chú |
|---|---|---|
| Người dùng cuối | Mở connection được cấp quyền, upload/download file, clipboard, chia sẻ phiên | Có thể chỉ là người giữ share link (user ẩn danh) |
| Quản trị viên | Quản lý user, group, connection, quyền; xem history, recording | Quyền hệ thống `ADMINISTER` |
| Auditor | Xem toàn bộ history | Quyền hệ thống `AUDIT` |
| Máy đích | Server RDP/VNC/SSH… | **Có thể độc hại.** Dữ liệu từ máy đích đi vào parser C của guacd (xem `04-BAO-MAT.md`) |
| IdP và vault | Xác thực, cấp secret | Cấu hình qua extension |

---

## 2. Container và hợp đồng giữa các thành phần

```mermaid
flowchart TB
    subgraph Browser["Trình duyệt"]
        NG["AngularJS SPA<br/>FE/ (index, auth, client, manage, settings…)"]
        GJS["guacamole-common-js<br/>Client, Tunnel, Parser, Display, Keyboard, Mouse…"]
        NG --> GJS
    end

    subgraph Tomcat["Servlet container (Tomcat 9)"]
        REST["REST API /api/*<br/>Jersey + Jackson (W/rest)"]
        TUN["Tunnel endpoint<br/>/websocket-tunnel (JSR-356)<br/>/tunnel (HTTP long-poll)"]
        CORE["Core: Guice, AuthenticationService,<br/>HashTokenSessionMap, TunnelRequestService"]
        EXT["Extensions .jar trong GUACAMOLE_HOME/extensions<br/>(classloader riêng)"]
        COMMON["guacamole-common<br/>ConfiguredGuacamoleSocket, Reader/Writer, Parser"]
        API["guacamole-ext<br/>AuthenticationProvider, UserContext, Directory, Permission"]
        REST --> CORE
        TUN --> CORE
        CORE --> API
        EXT -. "implements" .-> API
        CORE --> COMMON
    end

    subgraph GuacdHost["Máy chạy guacd"]
        PARENT["guacd process cha<br/>S/guacd/daemon.c, connection.c"]
        CHILD["Process con mỗi connection<br/>(fork + fexecve, argv0 = guacd(proto))"]
        LIB["libguac (S/libguac)<br/>socket, parser, user, client, display"]
        PLUG["Plugin .so<br/>libguac-client-rdp/vnc/ssh/telnet/kubernetes"]
        PARENT -- "fd qua SCM_RIGHTS" --> CHILD
        CHILD --> LIB
        CHILD -- "dlopen + guac_client_init" --> PLUG
    end

    GJS -- "REST: header Guacamole-Token" --> REST
    GJS -- "WebSocket ?token=…&GUAC_ID=… (subprotocol guacamole)" --> TUN
    COMMON -- "TCP 4822: select / args / size / connect / ready" --> PARENT
    EXT -- "JDBC / LDAP / HTTP" --> EXTSYS[("DB / LDAP / IdP")]
```

**Hợp đồng giữa các container:**

| Ranh giới | Giao thức / định dạng | Xác thực | Nơi định nghĩa |
|---|---|---|---|
| Browser ↔ REST | JSON qua HTTPS | Header `Guacamole-Token` (fallback query `?token=`) | `W/rest/auth/AuthenticationService.java:533-548` |
| Browser ↔ Tunnel | Guacamole protocol qua WebSocket hoặc HTTP | `token` trong query string (WebSocket không đặt được header) | `FE/client/types/ManagedClient.js:329-370`, `W/tunnel/TunnelRequest.java:36-100` |
| Web app ↔ guacd | Guacamole protocol qua TCP, TLS tùy chọn (`guacd-ssl`) | **Không có xác thực.** guacd tin bất kỳ ai kết nối tới | `C/protocol/ConfiguredGuacamoleSocket.java:203-318`, `S/guacd/connection.c:291-439` |
| guacd cha ↔ con | Unix socketpair, truyền fd bằng `SCM_RIGHTS`, config qua pipe | Cùng máy, cùng UID | `S/guacd/move-fd.c:35-139`, `S/guacd/proc.c:551-766` |
| guacd ↔ máy đích | RDP/RFB/SSH/Telnet/WebSocket | Theo tham số connection (mật khẩu, key, cert) | `S/protocols/*/settings.c` |
| Web app ↔ extension | Java interface `guacamole-ext` | Cùng JVM, cùng toàn quyền | `E/net/auth/*` |

---

## 3. Cấu trúc repo và phụ thuộc module

Thư mục review chứa **hai repo git độc lập**, đều thuộc Apache. Chúng giao tiếp qua mạng, không import mã của nhau. Theo `monorepo.md`, đây là cấu trúc **cấp C (nhóm app)** tách repo, phù hợp với quy mô: hai ngôn ngữ, hai chu kỳ build, có thể triển khai độc lập.

```text
guacamole/                      ← thư mục chứa (không phải git repo)
├── guacamole-client/           ← Maven multi-module, Java 8 target + JS
│   ├── guacamole-common/       ← thư viện Java: protocol, tunnel servlet/websocket
│   ├── guacamole-common-js/    ← thư viện JS: Client, Tunnel, Display… (đóng gói .war/.js)
│   ├── guacamole-ext/          ← API cho extension (AuthenticationProvider…)
│   ├── guacamole/              ← web app (.war): Java REST/tunnel + frontend AngularJS (webpack)
│   ├── extensions/             ← 14 extension chính thức (jdbc, ldap, sso, totp, duo, header, json, ban,
│   │                              restrict, quickconnect, radius, vault, history-recording-storage, display-statistics)
│   ├── guacamole-docker/       ← script build/entrypoint cho image guacamole/guacamole
│   ├── doc/                    ← ví dụ (example, branding, playback) + license bundle
│   └── Dockerfile, pom.xml
└── guacamole-server/           ← Autotools, C99
    ├── src/libguac/            ← thư viện lõi: socket, parser, protocol, user, client, display, recording
    ├── src/guacd/              ← daemon: accept, route, process model, watchdog
    ├── src/protocols/{rdp,vnc,ssh,telnet,kubernetes}/  ← plugin .so
    ├── src/common/, common-ssh/, terminal/, pulse/     ← code dùng chung giữa plugin
    ├── src/guacenc/, guaclog/  ← công cụ chuyển recording sang video / log phím
    ├── src/guacd-docker/       ← autobuild.sh, entrypoint.sh
    └── Dockerfile, configure.ac
```

**Hướng phụ thuộc** (mũi tên = "phụ thuộc vào"):

```mermaid
flowchart BT
    subgraph client["guacamole-client (Java/JS)"]
        common["guacamole-common"]
        ext["guacamole-ext"]
        webapp["guacamole (webapp)"]
        cjs["guacamole-common-js"]
        jdbc["auth-jdbc"]
        others["các extension khác"]
        ext --> common
        webapp --> ext
        webapp --> common
        webapp --> cjs
        jdbc --> ext
        others --> ext
    end
    subgraph server["guacamole-server (C)"]
        libguac["libguac"]
        guacd["guacd"]
        commonc["common / common-ssh / terminal / pulse"]
        rdp["rdp"]
        vnc["vnc"]
        ssh["ssh"]
        telnet["telnet"]
        k8s["kubernetes"]
        guacd --> libguac
        commonc --> libguac
        rdp --> commonc
        vnc --> commonc
        ssh --> commonc
        telnet --> commonc
        k8s --> commonc
        guacenc["guacenc / guaclog"] --> libguac
    end
```

Kết luận về hướng phụ thuộc: **một chiều, không có vòng.** Extension chỉ phụ thuộc `guacamole-ext`, không phụ thuộc webapp. Plugin C chỉ phụ thuộc `libguac` và `common*`. guacd không biết tới plugin cụ thể nào, chỉ `dlopen` theo tên (`S/libguac/client.c:656-714`).

---

## 4. Giao thức Guacamole — "ngôn ngữ chung" của cả hệ thống

Mọi thành phần (JS, Java, C) đều nói cùng một giao thức text. Hiểu giao thức này là hiểu được toàn bộ luồng dữ liệu.

**Định dạng instruction:** `LEN.VALUE,LEN.VALUE,...;`
- Phần tử đầu tiên là **opcode**.
- `LEN` là số **ký tự Unicode** (code point) của `VALUE`, không phải số byte.
- `,` ngăn cách các phần tử, `;` kết thúc instruction.

Ví dụ: `4.size,4.1024,3.768,2.96;` nghĩa là `size(1024, 768, 96)`.

Vì độ dài đứng trước nội dung, giá trị có chứa `,` hay `;` vẫn không phá được cấu trúc. Đây là cơ chế chống "injection" của giao thức. Điều kiện tiên quyết: **mọi bên phải đếm độ dài giống nhau.** Xem `SEC-02` để thấy chuyện gì xảy ra khi guacd và Java đếm khác nhau với chuỗi UTF-8 lỗi.

| Bộ parse | File | Giới hạn |
|---|---|---|
| C (guacd) | `S/libguac/parser.c:57-163`, `S/libguac/guacamole/parser-constants.h:32-42` | phần tử ≤ 8192 ký tự, ≤ 128 phần tử, buffer 32 KiB |
| Java | `C/protocol/GuacamoleParser.java:38-48, 271-306` | 8192 ký tự, 5 chữ số, 128 phần tử |
| JS | `CJ/Parser.js:127-252` | không giới hạn độ dài (xem `PERF-01`) |

**Nhóm opcode chính:**

| Nhóm | Hướng | Opcode tiêu biểu |
|---|---|---|
| Handshake | web app → guacd | `select`, `size`, `audio`, `video`, `image`, `timezone`, `name`, `connect` |
| Handshake | guacd → web app | `args` (danh sách tham số + version), `ready <connection-id>` |
| Vẽ | guacd → browser | `img`, `png`/`jpeg`, `blob`, `end`, `copy`, `rect`, `cfill`, `cursor`, `size`, `move`, `shade`, `dispose`, `transfer` |
| Đồng bộ | hai chiều | `sync <timestamp>` (dùng cho backpressure), `nop` (keep-alive) |
| Input | browser → guacd | `key <keysym> <pressed>`, `mouse <x> <y> <mask>`, `touch`, `size` (resize) |
| Stream | hai chiều | `clipboard`, `file`, `pipe`, `audio`, `video`, `blob`, `ack`, `end`, `filesystem`, `get`, `put`, `body` |
| Điều khiển | hai chiều | `error <msg> <status>`, `disconnect`, `argv`, `required`, `nest` |
| Nội bộ tunnel | tunnel ↔ browser | opcode rỗng `""` (UUID tunnel, `ping`). Web app chặn opcode này từ browser ở WebSocket (`C/websocket/GuacamoleWebSocketTunnelEndpoint.java:316`) |

---

## 5. Luồng F1 — Khởi động web app và nạp extension

```mermaid
flowchart TD
    A["Tomcat nạp guacamole.war"] --> B["GuacamoleServletContainerInitializer<br/>ép Logback, log INFO sớm"]
    B --> C["GuacamoleServletContextListener.contextInitialized<br/>W/GuacamoleServletContextListener.java"]
    C --> D{"Chọn GUACAMOLE_HOME<br/>E/environment/LocalEnvironment.java:167-185"}
    D -->|"1. system property guacamole.home<br/>2. env GUACAMOLE_HOME<br/>3. ~/.guacamole<br/>4. /etc/guacamole"| E["Đọc guacamole.properties<br/>+ env vars (nếu enable-environment-properties)<br/>+ *_FILE (nếu enable-file-environment-properties)"]
    E --> F["Tạo HashTokenSessionMap<br/>api-session-timeout mặc định 60 phút"]
    F --> G["Guice injector cha:<br/>EnvironmentModule, LogModule"]
    G --> H["Guice injector con:<br/>ExtensionModule, RESTServiceModule, TunnelModule"]
    H --> I["ExtensionModule.getExtensions<br/>W/extension/ExtensionModule.java:443-500"]
    I --> J["Liệt kê *.jar trong GUACAMOLE_HOME/extensions"]
    J --> K["Mỗi jar: đọc guac-manifest.json<br/>tạo ExtensionClassLoader (parent = lib/)"]
    K --> L{"guacamoleVersion<br/>tương thích?"}
    L -->|Không| M["Bỏ qua extension, log cảnh báo"]
    L -->|Có| N["Sắp xếp theo extension-priority<br/>(mặc định theo tên file)"]
    N --> O["loadExtensions: bind AuthenticationProvider (facade),<br/>Listener, JS/CSS gộp vào app.js/app.css,<br/>translations, HTML patch, static /app/ext/ns/"]
    O --> P["FileAuthenticationProvider (user-mapping.xml)<br/>luôn bind CUỐI CÙNG"]
    P --> Q["RESTServiceModule: Jersey /api/*<br/>TunnelModule: /tunnel + /websocket-tunnel"]
    Q --> R["Phát ApplicationStartedEvent"]
```

Điểm cần nhớ:
- **Thứ tự extension chính là thứ tự xác thực.** Provider đứng trước có quyền trả kết quả trước. Đổi tên file `.jar` là đổi thứ tự (`ARCH`/`DOC-01`).
- **GUACAMOLE_HOME là ranh giới tin cậy.** Mọi `.jar` trong đó chạy với toàn quyền JVM và có thể chèn JS vào UI (`W/extension/ExtensionModule.java:451-476, 550-575`).

---

## 6. Luồng F2 — Đăng nhập, MFA và vòng đời auth token

### 6.1 Đăng nhập thường (ví dụ JDBC + TOTP)

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser (authenticationService.js)
    participant R as TokenRESTService
    participant AS as AuthenticationService
    participant P1 as Provider 1 (JDBC)
    participant P2 as Provider 2 (TOTP)
    participant SM as HashTokenSessionMap
    participant L as Listener (auth-ban, …)

    B->>B: Route resolve gọi updateCurrentToken
    Note over B: token cũ lấy từ localStorage GUAC_AUTH_TOKEN<br/>rồi merge thêm $location.search() (SEC-11)
    B->>R: POST /api/tokens (username, password, token?)
    R->>AS: authenticate(credentials)
    AS->>AS: updateCredentials() trên mọi provider
    AS->>L: AuthenticationRequestReceivedEvent (có thể veto, ví dụ IP bị ban)
    alt Có token và session còn sống
        AS->>AS: updateAuthenticatedUser (re-auth, không đổi token)
    else Đăng nhập mới
        AS->>P1: authenticateUser(credentials)
        P1->>P1: selectOne(username), so SHA-256(pass + hex(salt)) (SEC-04)
        P1-->>AS: AuthenticatedUser
    end
    AS->>P1: getUserContext(user)
    AS->>P2: getUserContext / decorate(context)
    P2->>P2: verifyIdentity: đã có guac-totp chưa?
    P2-->>AS: throw GuacamoleInsufficientCredentialsException (expected: guac-totp)
    AS-->>R: lỗi INSUFFICIENT_CREDENTIALS
    R-->>B: 403 + APIError (expected = field TOTP)
    B->>B: Hiện form nhập mã TOTP
    B->>R: POST /api/tokens (username, password, guac-totp)
    R->>AS: authenticate (lặp lại các bước trên)
    P2->>P2: code == generate() hoặc previous(), codeService.useCode() chống replay
    AS->>AS: SecureRandomAuthTokenGenerator: 32 byte, hex 64 ký tự
    AS->>SM: put(token, GuacamoleSession)
    AS->>L: AuthenticationSuccessEvent (có thể veto)
    AS-->>R: token
    R-->>B: authToken, username, dataSource, availableDataSources
    B->>B: localStorage.setItem GUAC_AUTH_TOKEN, revoke token cũ nếu khác
```

Ghi chú theo bước:
- **Bước 1–2:** `FE/auth/service/authenticationService.js:294-310`.
- **Bước 4–7:** `W/rest/auth/AuthenticationService.java:357-449`.
- Nhánh re-auth: `:183-203`. Lựa chọn provider: `:121-163`. Ưu tiên `InsufficientCredentials`: `:138-151`.
- `getUserContexts` và decorate chéo: `:264-333`. Sinh token: `W/rest/auth/SecureRandomAuthTokenGenerator.java:33-40`.
- **TOTP:** `extensions/guacamole-auth-totp/.../user/UserVerificationService.java:363-406`. Chống replay trong 2 chu kỳ: `CodeUsageTrackingService.java:220-251`.

### 6.2 SSO (OpenID Connect, SAML)

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser
    participant W as Web app (/api/tokens)
    participant O as SSO provider (openid / saml)
    participant I as Identity Provider

    B->>W: POST /api/tokens (chưa có id_token / state)
    W->>O: authenticateUser
    O-->>W: throw Insufficient + RedirectField(loginURI)
    W-->>B: 403 expected = redirect
    alt OpenID (implicit flow, response_type=id_token)
        B->>I: redirect authorize?nonce=…&response_type=id_token
        I-->>B: redirect về, id_token nằm trong URL fragment
        B->>W: POST /api/tokens id_token=…
        W->>O: jose4j verify chữ ký JWKS, iss, aud, exp, nonce (dùng 1 lần)
    else SAML (SP-initiated)
        B->>I: AuthnRequest + RelayState (lưu 5 phút)
        I->>W: POST /api/ext/saml/callback (SAMLResponse)
        W->>O: java-saml validate chữ ký, InResponseTo, saml-strict
        W-->>B: 303 về ?state=id
        B->>W: POST /api/tokens state=id
    end
    O-->>W: AuthenticatedUser (username = claim email mặc định, SEC-16)
    W-->>B: authToken
```

### 6.3 Vòng đời token

```mermaid
stateDiagram-v2
    [*] --> Active: POST /api/tokens thành công
    Active --> Active: mỗi request REST hoặc tunnel cập nhật lastAccessedTime
    Active --> Active: re-auth với cùng token (không rotate, SEC-26)
    Active --> Invalidated: DELETE /api/session hoặc DELETE /api/tokens/:token
    Active --> Invalidated: GuacamoleUnauthorizedException bất kỳ (RESTExceptionMapper:75-79)
    Active --> Expired: không có tunnel và idle ≥ api-session-timeout (task chạy mỗi phút)
    Active --> Expired: extension báo UserContext.isValid() = false
    Note right of Active: Có tunnel đang mở thì không bao giờ hết hạn
    Invalidated --> [*]: đóng mọi tunnel, invalidate UserContext, phát UserSessionInvalidatedEvent
    Expired --> [*]
```

Token chỉ nằm trong `ConcurrentHashMap` của từng JVM (`W/rest/auth/HashTokenSessionMap.java:55, 124-178`): restart là mất phiên, và nhiều node không chia sẻ được (`ARCH-03`).

---

## 7. Luồng F3 — Phân quyền REST

Core REST **không tự quyết định quyền**. Nó ủy cho `Directory` của từng provider. Riêng provider JDBC lọc quyền READ ngay trong SQL.

```mermaid
flowchart TD
    REQ["Request /api/session/data/{ds}/connections/{id}<br/>header Guacamole-Token"] --> S1{"Token có session?<br/>SessionRESTService"}
    S1 -->|Không| D403["403 Permission Denied"]
    S1 -->|Có| S2["GuacamoleSession.getUserContext(ds)<br/>W/GuacamoleSession.java:176-196"]
    S2 --> S3["DirectoryResource / DirectoryObjectResource"]
    S3 --> M{"HTTP method"}
    M -->|GET list| G1["directory.getAll()<br/>JDBC: selectReadable (lọc READ trong SQL)"]
    M -->|GET id| G2["directory.get(id)<br/>không có READ → null → 404"]
    M -->|GET id/parameters| G3{"ADMINISTER hoặc UPDATE?<br/>W/rest/connection/ConnectionResource.java:132-134"}
    M -->|PUT / DELETE| G4["directory.update / remove<br/>JDBC: beforeUpdate/beforeDelete kiểm UPDATE/DELETE"]
    M -->|PATCH permissions| G5["PermissionSet.add/remove<br/>JDBC: cần UPDATE trên entity + ADMINISTER trên mọi object"]
    G3 -->|Không| D403
    G1 --> OK["200 JSON"]
    G2 --> OK
    G3 -->|Có| OK
    G4 --> OK
    G5 --> OK

    subgraph JDBC["Quyết định quyền trong JDBC (J/base, J/permission)"]
        P0{"Có system ADMINISTER?<br/>ModeledPermissions.isPrivileged"}
        P0 -->|Có| ALLOW["Bỏ qua kiểm tra object"]
        P0 -->|Không| P1["Effective groups = group DB (đệ quy)<br/>+ group do provider khác khai báo, khớp theo tên (SEC-19)"]
        P1 --> P2{"Có quyền READ/UPDATE/DELETE/ADMINISTER<br/>cho entity của user hoặc group?"}
    end
```

Chi tiết mô hình quyền và ER diagram ở `08-DATABASE.md`.

---

## 8. Luồng F4 — Mở một phiên remote desktop end-to-end

Đây là luồng quan trọng nhất. Một click "mở connection" đi qua 6 thành phần.

```mermaid
sequenceDiagram
    autonumber
    participant UI as AngularJS (ManagedClient)
    participant GJ as guacamole-common-js (Client + ChainedTunnel)
    participant WS as Web app WebSocket endpoint
    participant TR as TunnelRequestService
    participant PR as Provider (JDBC)
    participant GP as guacd cha (conn-route thread)
    participant GC as guacd con (process mới)
    participant PL as Plugin RDP + FreeRDP
    participant RS as Máy đích RDP

    UI->>UI: route /client/:id → ManagedClient.getInstance
    UI->>GJ: new Client(new ChainedTunnel(WebSocketTunnel, HTTPTunnel))
    GJ->>WS: WebSocket /websocket-tunnel?token=…&GUAC_DATA_SOURCE&GUAC_ID&GUAC_TYPE=c&GUAC_WIDTH/HEIGHT/DPI…
    WS->>TR: createTunnel(request)
    TR->>TR: lấy session theo token, UserContext theo data source
    TR->>PR: connectionDirectory.get(id) (null nếu không có READ)
    TR->>PR: connection.connect(info, StandardTokenMap)
    PR->>PR: acquire(): kiểm max-connections, max-connections-per-user (CAS)
    PR->>PR: insert guacamole_connection_history, thay token ${GUAC_USERNAME}…
    PR->>GP: TCP 4822: select rdp
    GP->>GP: guac_parser_expect "select" (timeout 15s)
    GP->>GC: socketpair + pipe + fork + fexecve(guacd(rdp))
    GC->>GC: đọc config pipe, PR_SET_PDEATHSIG, alarm, đóng pipe báo sẵn sàng
    GC->>PL: dlopen libguac-client-rdp.so, guac_client_init
    GP->>GC: sendmsg SCM_RIGHTS (fd của user)
    GC-->>PR: args VERSION_1_5_0, hostname, port, username, password, …
    PR->>GC: size 1920 1080 96 / audio / video / image / timezone / name
    PR->>GC: connect (giá trị tương ứng từng arg)
    GC->>PL: join_handler(owner): parse settings, tạo thread rdp-client
    GC-->>PR: ready $uuid-connection-id
    PR-->>TR: GuacamoleTunnel (ConfiguredGuacamoleSocket)
    TR->>TR: TunnelConnectEvent (listener có thể veto, BE-01) → session.addTunnel
    WS-->>GJ: instruction nội bộ "0.,36.uuid-tunnel"
    PL->>RS: freerdp_connect (TLS/NLA, VerifyCertificate, Authenticate)
    RS-->>PL: bitmap updates
    PL-->>GJ: img / blob / end / sync (đi qua guacd con → cha → web app → WebSocket)
    GJ->>GJ: Display vẽ canvas, lần sync đầu: state CONNECTED
```

Các giai đoạn và vị trí code:

| Giai đoạn | Vị trí |
|---|---|
| Dựng connect string | `FE/client/types/ManagedClient.js:329-370` |
| ChainedTunnel và fallback | `CJ/Tunnel.js:1108-1222` |
| WebSocket handshake lưu request | `W/tunnel/websocket/RestrictedGuacamoleWebSocketTunnelEndpoint.java:80-91` |
| Tạo tunnel | `W/tunnel/TunnelRequestService.java:330-375` |
| Kiểm concurrency | `J/tunnel/RestrictedGuacamoleTunnelService.java:113-240` |
| Ghi history và thay token | `J/tunnel/AbstractGuacamoleTunnelService.java:474-569` |
| Handshake phía Java | `C/protocol/ConfiguredGuacamoleSocket.java:203-318` |
| Route phía guacd cha | `S/guacd/connection.c:291-439` |
| Tạo process | `S/guacd/proc.c:551-766` |
| Process con khởi động | `S/guacd/proc-child.c:157-323` |
| Handshake phía libguac | `S/libguac/user-handshake.c:257-402` |
| Plugin RDP | `S/protocols/rdp/client.c:145`, `user.c:46`, `rdp.c:804-958` |

**Ba lớp kiểm soát quyền trong luồng này:**
1. **Web app:** token phải hợp lệ, provider phải trả về connection (READ).
2. **Provider:** giới hạn số phiên, cửa sổ thời gian (`auth-restrict`), veto qua listener.
3. **guacd:** không kiểm quyền gì. Nó thực thi đúng tham số được gửi tới. Vì vậy mọi tham số nguy hiểm (`drive-path`, `recording-path`, `hostname`…) phải bị chặn từ web app (`SEC-06`).

---

## 9. Luồng F5 — Mô hình process của guacd

Từ bản 1.6.x, guacd không chỉ `fork` mà còn **`fexecve` lại chính binary** (mở sẵn qua `/proc/self/exe` lúc khởi động). Nhờ vậy mỗi connection có một address space sạch.

```mermaid
flowchart TD
    subgraph Parent["guacd process cha (S/guacd/daemon.c)"]
        P0["main: đọc guacd.conf + CLI<br/>(mặc định bind localhost:4822)"] --> P1["guacd_open_exe: mở /proc/self/exe"]
        P1 --> P2["bind + listen(5), TLS context nếu có cert/key"]
        P2 --> P3["accept4 → set SO_RCVTIMEO/SNDTIMEO 15s"]
        P3 --> P4["thread conn-route: guacd_connection_thread"]
        P4 --> P5{"select <id>"}
        P5 -->|"id bắt đầu bằng $"| J1["proc_map_open: tìm process đang chạy<br/>(join phiên có sẵn)"]
        P5 -->|"tên protocol"| N1["guacd_create_proc"]
        N1 --> N2["socketpair SOCK_DGRAM (kênh truyền fd)<br/>pipe2 (kênh config)"]
        N2 --> N3["fork()"]
        N3 --> N4["con: setpgid, dup2 vào fd 3 và 4,<br/>fexecve(exe_fd, argv0 = guacd(rdp))"]
        N3 --> N5["cha: ghi config (log level, connection id, deadline)<br/>poll chờ con đóng pipe ≤ 15s"]
        N5 --> N6["proc_map_add, thread conn-route block<br/>trong waitid(WNOWAIT) suốt phiên"]
        J1 --> U1["guacd_add_user: socketpair STREAM user_fd/proc_fd,<br/>sendmsg SCM_RIGHTS proc_fd sang con"]
        N6 -.-> U1
        U1 --> U2["thread conn-read + conn-write:<br/>bơm byte giữa socket web app và user_fd"]
    end

    subgraph Child["Process con (S/guacd/proc-child.c, proc.c)"]
        C0["guacd_connection_process: alarm, PR_SET_PDEATHSIG,<br/>đọc config fd 4, đóng fd 4 báo sẵn sàng"] --> C1["guacd_exec_proc: watchdog (op 15s, cleanup 5s)"]
        C1 --> C2["dlopen libguac-client-PROTO.so → guac_client_init"]
        C2 --> C3["vòng recvmsg fd 3: mỗi fd nhận được là một user<br/>(fd đầu tiên = owner)"]
        C3 --> C4["thread user-conn: guac_user_handle_connection<br/>(handshake args…connect, ready)"]
        C4 --> C5["thread user-input: guac_parser_read →<br/>__guac_instruction_handler_map"]
    end

    N4 --> C0
    U1 -- "fd qua SCM_RIGHTS" --> C3
```

**Các thread trong một connection RDP:**

| Process | Thread | Vai trò |
|---|---|---|
| cha | `conn-route` | Parse `select`, tạo hoặc join process, chờ process con thoát |
| cha | `conn-read`, `conn-write` | Bơm dữ liệu hai chiều giữa web app và process con |
| con | main | Vòng nhận fd user mới |
| con | `teardown-watch`, watchdog | Theo dõi timeout của từng thao tác và của giai đoạn dọn dẹp |
| con | `user-conn`, `user-input` (mỗi user) | Handshake, đọc lệnh từ user |
| con | `user-promote` | Chuyển user pending sang active sau `join_pending_handler` |
| con | `rdp-client` | Vòng sự kiện FreeRDP |
| con | render thread | Flush `guac_display` thành instruction vẽ |

**Vì sao thiết kế này tốt:**
- **Cô lập lỗi:** plugin crash, kể cả do máy đích độc hại, chỉ chết một phiên.
- **Bề mặt tấn công nhỏ ở process cha:** process cha chỉ parse đúng một lệnh `select`.
- **Không cần cơ chế tái sử dụng PID:** `waitid(WNOWAIT)` cùng process group riêng giúp kill sạch toàn bộ cây process.
- Mọi fd đều `CLOEXEC`.

**Điểm yếu:** process cha không giới hạn số kết nối đồng thời, và timeout 15 giây chỉ áp cho từng lần read (`SEC-12`).

---

## 10. Luồng F6 — Trong phiên: vẽ màn hình, sync/backpressure, input

```mermaid
sequenceDiagram
    autonumber
    participant RS as Máy đích RDP
    participant PL as Plugin (FreeRDP GDI)
    participant DSP as guac_display + render thread
    participant BS as broadcast socket (mọi user)
    participant JV as Web app (đọc, buffer 8192 ký tự)
    participant JS as Guacamole.Client + Display (canvas)
    participant KB as Keyboard / Mouse

    RS->>PL: Bitmap update (RDP PDU)
    PL->>PL: BeginPaint/EndPaint, đánh dấu vùng dirty (gdi.c:73-135)
    PL->>DSP: guac_display_render_thread_notify_modified
    DSP->>DSP: gom vùng dirty theo frame, chọn encoder (PNG/JPEG/WebP)
    DSP->>BS: img + blob… + end, rồi sync ts
    BS->>JV: (qua process cha)
    JV->>JS: WebSocket text frame
    JS->>JS: Parser → instructionHandlers[opcode]
    JS->>JS: Display xếp task vào Frame, frame chỉ flush khi mọi ảnh đã decode xong
    JS->>JS: sync: display.flush(cb)
    JS-->>DSP: sync ts (sau khi vẽ xong, đây là ACK backpressure)
    DSP->>DSP: __guac_handle_sync: tính processing_lag, estimated_rtt
    Note over DSP: Plugin đọc lag để giảm tốc độ gửi frame.<br/>Client chậm thì nhận ít frame hơn thay vì bị dồn hàng đợi.
    KB->>JS: keydown (keysym X11)
    JS->>JV: key 65307 1
    JV->>PL: (qua guacd) __guac_handle_key → user->key_handler
    PL->>PL: guac_fifo_enqueue, rdp-client gửi scancode/unicode tới RS
```

Ghi chú:
- **Backpressure:**
  - Phía JS: `CJ/Client.js:1672-1697`, `CJ/Display.js:195-224, 404-413`.
  - Phía C: `S/libguac/user-handlers.c:107-164`, `S/libguac/client.c:781`.
- **Keep-alive:**
  - WebSocket gửi ping mỗi 500 ms (`CJ/Tunnel.js:868-911`).
  - Client gửi `nop` mỗi 5 giây (`CJ/Client.js:49, 1795-1832`).
  - guacd gửi `nop` khi im lặng quá 5 giây (`S/libguac/socket.c:46-83`).
  - Sau 15 giây không nhận gì thì báo `UPSTREAM_TIMEOUT`. Mất tín hiệu quá 1,5 giây thì vào trạng thái `UNSTABLE` (`CJ/Tunnel.js:116-127`).
- **Rủi ro:** nếu decode ảnh lỗi, frame bị kẹt mãi, client không gửi `sync`, phiên đứng hình (`ERR-03`).

---

## 11. Luồng F7 — Chia sẻ phiên và join

```mermaid
sequenceDiagram
    autonumber
    participant O as Owner (đang trong phiên)
    participant W as Web app (JDBC sharing)
    participant V as Người xem (share link)
    participant GP as guacd cha
    participant GC as guacd con (phiên của owner)

    O->>W: Tạo share link theo sharing profile (read-only hoặc full)
    W-->>O: URL ?key=… (share key 44 ký tự base64url, 264 bit)
    V->>W: POST /api/tokens key=…
    W-->>V: token của SharedAuthenticatedUser (user ẩn danh)
    V->>W: WebSocket tunnel (GUAC_TYPE = a, active connection)
    W->>GP: select $connection-id
    GP->>GP: proc_map_open: tìm process theo id, dup fd
    GP->>GC: sendmsg SCM_RIGHTS fd mới
    GC->>GC: handshake, connect kèm read-only nếu profile yêu cầu
    GC->>GC: join_handler(non-owner): read-only thì không gán mouse/key/clipboard handler
    GC-->>V: hình ảnh hiện tại (guac_display_dup) + stream tiếp theo qua broadcast socket
```

Lưu ý bảo mật:
- **Ai tới được guacd trực tiếp và biết `$connection-id` là join được phiên đó,** với cờ `read-only` do chính họ tự khai báo (`SEC-08`). Connection ID được log ở mức INFO (`SEC-24`).
- **Người xem read-only vẫn gửi được lệnh `size` (kể cả DPI=0) và các lệnh không kiểm `argc`** (`VAL-01`, `SEC-01`). Kết quả có thể làm sập process, tức đánh sập phiên của owner.

---

## 12. Luồng F8 — Truyền file, clipboard, in ấn

### 12.1 RDP drive redirection và SFTP

```mermaid
flowchart TD
    subgraph Upload["Upload từ trình duyệt"]
        U1["ManagedClient.uploadFile → instruction file"] --> U2["guacd: user->file_handler"]
        U2 --> U3{"Protocol?"}
        U3 -->|RDP drive| U4["upload.c:73 thay / và \ bằng _<br/>→ guac_rdp_fs_open FILE_OVERWRITE_IF"]
        U3 -->|SFTP| U5["common-ssh/sftp.c:373 kiểm disable_upload<br/>append_filename chặn . .. /<br/>libssh2_sftp_open 0600"]
        U1 --> U6["Dữ liệu: XHR POST chunk 4 MB tới<br/>/api/session/tunnels/uuid/streams/idx/name?token=… (SEC-09)"]
    end

    subgraph RDPDR["Máy đích RDP truy cập ổ ảo"]
        R1["IRP_MJ_CREATE từ server"] --> R2["rdpdr-fs-messages.c:49-85<br/>kiểm path_length, UTF-16 → UTF-8 (off-by-one, SEC-07)"]
        R2 --> R3["guac_rdp_fs_open (fs.c:214)"]
        R3 --> R4["normalize_path: xử lý .., chặn :, độ sâu ≤ 64"]
        R4 --> R5["translate_path: drive_path + đường dẫn ảo<br/>(drive_path rỗng → '/', SEC-22)"]
        R5 --> R6["open(…, 0600), cấp file_id từ pool"]
        R7["IRP READ/WRITE/CLOSE với FileId"] --> R8["guac_rdp_fs_get_file: chỉ kiểm khoảng giá trị,<br/>không kiểm slot đang mở (SEC-03)"]
        R9["Liệt kê thư mục"] --> R10["dir-info: buffer 256 byte nhưng ghi length*2 (SEC-07)"]
    end

    subgraph Download["Download về trình duyệt"]
        D1["RDP: server ghi file vào \\Download\\ → guac_rdp_download_to_user"] --> D3
        D2["SSH: remote in OSC escape → terminal-handlers.c:1342 → sftp.c:525<br/>(không đi qua root_path, SEC-28)"] --> D3
        D3["instruction file + blob…"] --> D4["Web app StreamInterceptingTunnel<br/>REST GET …/streams/idx/name?token=…"]
        D4 --> D5["Content-Type octet-stream + Content-Disposition attachment<br/>(W/rest/tunnel/StreamResource.java:107-111)"]
    end
```

### 12.2 Clipboard

```mermaid
sequenceDiagram
    participant LC as Clipboard máy local
    participant IDX as indexController / clipboardService
    participant MC as Mọi ManagedClient đang mở
    participant G as guacd plugin
    participant R as Máy đích

    LC->>IDX: focus / copy / cut (window event)
    IDX->>IDX: navigator.clipboard.readText
    IDX->>MC: broadcast guacClipboard (tới MỌI connection, SEC-29)
    MC->>G: clipboard + blob + end (trừ khi disable-paste)
    G->>R: cliprdr / RFB cut text / gõ vào terminal
    R->>G: clipboard thay đổi
    G->>G: guac_iconv CP1252/UTF-16 → UTF-8, cắt theo giới hạn
    G->>MC: clipboard stream (trừ khi disable-copy)
    MC->>LC: clipboardService.setClipboard (ghi thẳng clipboard local)
```

### 12.3 In ấn RDP

Luồng: máy đích in tới máy in ảo → `rdpdr-printer.c` → `print-job.c:330-366`: `fork` rồi `execvp("gs", "-dSAFER", "-sDEVICE=pdfwrite", …)` → PDF được stream về trình duyệt của owner. Ghostscript chạy trên **PostScript do máy đích gửi** và kế thừa toàn bộ fd (`SEC-21`).

---

## 13. Luồng F9 — Ghi phiên và phát lại

```mermaid
flowchart LR
    A["Tham số recording-path / recording-name<br/>(do admin hoặc quickconnect đặt)"] --> B["guac_recording_create<br/>S/libguac/recording.c:42"]
    B --> C["guac_openat (S/libguac/file.c:109)<br/>guac_is_filename chặn / \\ . ..<br/>mkdir 0750 nếu create-recording-path"]
    C --> D["open O_CREAT + O_EXCL (hậu tố .1…255), 0640, khóa F_SETLK"]
    D --> E["socket tee: mọi instruction gửi user<br/>được ghi đồng thời vào file"]
    E --> F["File .guac (chính là Guacamole protocol)"]
    F --> G["guacenc → video .m4v<br/>guaclog → log phím"]
    F --> H["history-recording-storage extension:<br/>tìm file theo UUID history, quyền theo history"]
    H --> I["Browser: Guacamole.SessionRecording phát lại<br/>(player dùng header Guacamole-Token)"]
```

Typescript (SSH/Telnet) dùng cùng cơ chế, kèm thêm file `.timing`. File `.timing` được mở **không có `O_EXCL`** (`SEC-27`).

---

## 14. Luồng F10 — Kết thúc phiên và dọn dẹp

```mermaid
stateDiagram-v2
    [*] --> Running
    Running --> UserLeaving: user gửi disconnect / EOF / đóng WebSocket
    UserLeaving --> Running: còn user khác trong phiên
    UserLeaving --> UsersGone: user cuối cùng rời (cờ USERS_GONE)
    Running --> Stopping: plugin lỗi / máy đích ngắt / SIGTERM / watchdog timeout 15s
    UsersGone --> Stopping
    Stopping --> Cleanup: teardown-watch shutdown fd 3, recvmsg trả về
    Cleanup --> Freed: guac_client_stop, chờ user, free_handler, dlclose, reap con
    Cleanup --> Killed: quá 5s → watchdog kill(0, SIGKILL)
    Freed --> [*]: _exit
    Killed --> [*]
    note right of Freed
        Process cha: waitid trả về, kill(-pgid), proc_map_remove, waitpid
        Web app: UserTunnel.close, TunnelCloseEvent, JDBC ghi end_date vào history
    end note
```

Phía Java, `ConnectionCleanupTask` chạy đúng một lần (`AtomicBoolean`): nó trả lại "ghế" concurrency rồi cập nhật `end_date` (`J/tunnel/AbstractGuacamoleTunnelService.java:397-438`). Nếu JVM crash, record history nằm lại với `end_date IS NULL` (`DB-03`).

---

## 15. API extension (class diagram)

```mermaid
classDiagram
    class AuthenticationProvider {
        <<interface>>
        +getIdentifier() String
        +updateCredentials(Credentials) Credentials
        +authenticateUser(Credentials) AuthenticatedUser
        +updateAuthenticatedUser(AuthenticatedUser, Credentials) AuthenticatedUser
        +getUserContext(AuthenticatedUser) UserContext
        +updateUserContext(UserContext, AuthenticatedUser, Credentials) UserContext
        +decorate(UserContext, AuthenticatedUser, Credentials) UserContext
        +redecorate(...) UserContext
        +shutdown()
    }
    class AuthenticatedUser {
        <<interface>>
        +getIdentifier() String
        +getAuthenticationProvider() AuthenticationProvider
        +getCredentials() Credentials
        +getEffectiveUserGroups() Set
        +invalidate()
    }
    class UserContext {
        <<interface>>
        +self() User
        +getUserDirectory() Directory
        +getUserGroupDirectory() Directory
        +getConnectionDirectory() Directory
        +getConnectionGroupDirectory() Directory
        +getActiveConnectionDirectory() Directory
        +getSharingProfileDirectory() Directory
        +getConnectionHistory() ActivityRecordSet
        +getPrivileged() UserContext
        +isValid() boolean
        +invalidate()
    }
    class Directory~T~ {
        <<interface>>
        +get(String) T
        +getAll(Collection) Collection
        +getIdentifiers() Set
        +add(T)
        +update(T)
        +remove(String)
        +tryAtomically(AtomicDirectoryOperation)
    }
    class Connectable {
        <<interface>>
        +connect(GuacamoleClientInformation, Map tokens) GuacamoleTunnel
        +getActiveConnections() int
    }
    class Connection
    class ConnectionGroup
    class ActiveConnection
    class Permissions {
        <<interface>>
        +getSystemPermissions() SystemPermissionSet
        +getConnectionPermissions() ObjectPermissionSet
        +getUserPermissions() ObjectPermissionSet
    }
    class User
    class UserGroup
    class Credentials {
        +getUsername()
        +getPassword()
        +getRequestDetails() RequestDetails
    }
    class Listener {
        <<interface>>
        +handleEvent(Object)
    }
    AuthenticationProvider --> AuthenticatedUser : tạo
    AuthenticationProvider --> UserContext : tạo / decorate
    AuthenticatedUser --> Credentials
    UserContext --> Directory : cung cấp
    Connectable <|.. Connection
    Connectable <|.. ConnectionGroup
    Connectable <|.. ActiveConnection
    Permissions <|.. User
    Permissions <|.. UserGroup
    Directory o-- Connection
    Directory o-- User
```

Đây là một **plugin API đúng nghĩa**:
- Web app không biết DB, LDAP hay SAML là gì. Mọi thứ đi qua `AuthenticationProvider` và `UserContext`.
- Extension có thể *decorate* context của nhau. Ví dụ TOTP bọc JDBC, vault bọc mọi provider để chèn secret.

Các lớp hỗ trợ: `AbstractAuthenticationProvider`, `DelegatingUserContext`, `TokenInjectingUserContext`, `Simple*` (`E/net/auth/`).

---

## 16. Đánh giá kiến trúc

| Tiêu chí (`architecture-modular.md`) | Hiện trạng | Đánh giá |
|---|---|---|
| Module theo nghiệp vụ | Java: `rest/{auth,connection,user,permission,tunnel…}`, `tunnel/`, `extension/`. C: plugin theo protocol, `common*` dùng chung | Tốt |
| Hướng phụ thuộc | Một chiều, không vòng (mục 3) | Tốt |
| Ngân sách trừu tượng | Interface `Directory`/`UserContext` có **nhiều implementation thật** (JDBC, LDAP, JSON, file, quickconnect…), nên DI bằng Guice là hợp lý, không thừa | Tốt |
| Thiếu tổ chức | Một số file lớn: `DirectoryResource.patchObjects` có ba nhánh gần như trùng nhau; `rdp/settings.c` 2023 dòng; `terminal.c` 3250 dòng (`ARCH-04`) | Chấp nhận được |
| Lặp code | Parse settings recording/SFTP/WOL chép tay ở 5 plugin; mapper SQL lặp ở 3 DB, giống nhau 74–86% (`ARCH-01`, `ARCH-02`) | Cần cải thiện |
| Trạng thái phân tán | Session, concurrency limit, ban, TOTP replay, nonce SSO, denylist JSON đều nằm trong RAM của từng JVM (`ARCH-03`) | Giới hạn mở rộng ngang |
| Cô lập lỗi | Process cho mỗi connection, classloader cho mỗi extension, `skip-if-unavailable` | Rất tốt |
| Điểm lỗi đơn (SPOF) | guacd cha (crash là mất mọi phiên), web app (token trong RAM), DB | Phải chạy HA bằng sticky session. Chưa có HA thật |

**Cây thư mục đích:** không đề xuất tái cấu trúc lớn, vì cấu trúc hiện tại đã theo module thật và hợp với quy mô. Chỉ đề xuất gom phần lặp lại.

```text
guacamole-server/src/common/
├── settings-recording.c/.h     ← MỚI: parse recording-path/name/exclude-*/include-keys (thay 5 bản chép)
├── settings-sftp.c/.h          ← MỚI: parse sftp-* dùng chung RDP/VNC
├── settings-wol.c/.h           ← MỚI: parse wol-* dùng chung
└── (giữ nguyên các file hiện có)

guacamole-client/extensions/guacamole-auth-jdbc/modules/
├── guacamole-auth-jdbc-base/src/main/resources/…/common/   ← MỚI (tùy chọn): fragment SQL dùng chung
│                                                             qua <include>/<sql>, chỉ giữ phần khác cú pháp ở từng DB
└── guacamole-auth-jdbc-{mysql,postgresql,sqlserver}/       ← giữ nguyên
```

---

## 17. ADR đề xuất

Các quyết định kiến trúc còn bỏ ngỏ, nên chốt thành ADR (Architecture Decision Record):

| ADR | Câu hỏi | Lựa chọn khuyến nghị | Liên quan |
|---|---|---|---|
| ADR-01 | guacd có cần xác thực web app không? | Hỗ trợ mTLS tùy chọn (CA của client), giữ mặc định bind localhost; Docker docs không publish cổng 4822 | `SEC-08` |
| ADR-02 | Thuật toán hash mật khẩu JDBC | PBKDF2-HMAC-SHA256 (có sẵn trong JDK, không thêm dependency) hoặc Argon2id; thêm cột `password_hash_algorithm`; rehash khi đăng nhập | `SEC-04` |
| ADR-03 | Token trong URL | Ticket một lần cho WebSocket và stream, đổi bằng REST có header | `SEC-09` |
| ADR-04 | Chiến lược chạy nhiều node | (a) chính thức chỉ hỗ trợ single-node, hoặc (b) store chung cho session/limit/ban | `ARCH-03` |
| ADR-05 | Thay thế AngularJS 1.x (đã EOL) | Lộ trình chuyển dần sang Angular 2+ hoặc Web Components, giữ `guacamole-common-js` độc lập framework; trước mắt dùng CSP | `OPS-01`, `SEC-10` |
| ADR-06 | Allow-list tham số connection ở tầng web | Danh sách tham số "nguy hiểm" (path, host, cert) mặc định deny cho mọi nguồn do user định nghĩa (quickconnect, JSON) | `SEC-06` |
