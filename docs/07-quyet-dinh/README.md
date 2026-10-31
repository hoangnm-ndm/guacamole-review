# 07 · Quyết định (ADR)

> Ai đọc: người muốn biết **vì sao** docs / triển khai làm như vậy trước khi đổi.
> Không sửa ADR cũ — quyết định thay đổi thì thêm ADR mới ghi "thay ADR-0xx".
>
> Trước: [06-van-hanh/](../06-van-hanh/README.md) · Quay lại: [docs/README](../README.md)

| ADR | Quyết định | Ngày | Trạng thái |
| --- | --- | --- | --- |
| [ADR-001](ADR-001-dat-docs-o-thu-muc-goc.md) | Đặt docs ở thư mục gốc, ngoài 2 repo upstream; git riêng chỉ track docs | 2026-10-31 | Đang áp dụng |
| [ADR-002](ADR-002-dung-image-chinh-thuc-ghim-phien-ban.md) | Vận hành bằng image Docker chính thức, ghim tag `1.6.0`; không build từ clone chưa review | 2026-10-31 | Đang áp dụng |
| [ADR-003](ADR-003-phuong-an-trien-khai-compose-postgresql-caddy.md) | Phương án chuẩn: Docker Compose + PostgreSQL + Caddy trên một VPS | 2026-10-31 | Đang áp dụng |

Mẫu ADR mới (`ADR-00N-<mo-ta-khong-dau>.md`):

```markdown
# ADR-00N · <Quyết định ngắn gọn> (YYYY-MM-DD)

- **Bối cảnh**: vấn đề / ràng buộc dẫn tới quyết định.
- **Quyết định**: chọn gì (và bỏ phương án nào, vì sao).
- **Hệ quả**: được gì, mất gì, việc phải làm theo.
```
