# 04 · guacamole-client (Java + JavaScript)

> Mục lục phần client. Ai đọc: dev muốn hiểu / sửa web app, REST API, giao diện,
> hoặc viết extension. Cần đọc trước [02-kien-truc.md](../02-kien-truc.md).
>
> Trước: [03-guacamole-server/](../03-guacamole-server/README.md) · Tiếp: [05-quy-uoc-va-dong-gop.md](../05-quy-uoc-va-dong-gop.md)

Repo: `guacamole-client/` (upstream `github.com/apache/guacamole-client`, nhánh
`main`, phiên bản `1.6.1` chưa phát hành).

| #   | File                                                             | Nội dung                                                              |
| --- | ---------------------------------------------------------------- | --------------------------------------------------------------------- |
| 01  | [01-cau-truc-maven.md](01-cau-truc-maven.md)                     | Các module Maven, thư viện `guacamole-common` / `-common-js` / `-ext`, ví dụ trong `doc/` |
| 02  | [02-webapp-rest-va-tunnel.md](02-webapp-rest-va-tunnel.md)       | Khởi động webapp, Guice, GUACAMOLE_HOME, nạp extension, REST API, tunnel |
| 03  | [03-frontend.md](03-frontend.md)                                 | AngularJS, webpack, cấu trúc module, bản dịch                         |
| 04  | [04-extension.md](04-extension.md)                               | API extension, 14 extension có sẵn, cách viết extension               |
| 05  | [05-build-va-test.md](05-build-va-test.md)                       | Maven build, profile, test Java / JS, Docker image                    |

Thứ tự đọc gợi ý: 01 → 02 → 04 → 03 → 05. Chỉ muốn đổi giao diện / branding: 04
(mục branding) là đủ, không cần sửa webapp.
