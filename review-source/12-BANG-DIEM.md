# 12 — Bảng điểm (cấp Chuyên sâu)

Cách tính theo `scoring-rubric.md`:
- Mỗi nhóm bắt đầu từ 10 điểm.
- Trừ theo vấn đề thuộc nhóm chính của nó: Critical −4, High −2, Medium −1 (Medium hệ thống −1,5), Low −0,25 (tối đa −1 cho cả nhóm). Vấn đề `[CHUẨN]` không bắt buộc trừ tối đa −1 cho cả nhóm. Vấn đề "Chưa đủ dữ liệu" không trừ.
- Điều chỉnh tối đa ±1, có ghi lý do.
- Trọng số theo cột "Chuyên sâu".

| # | Nhóm | Trọng số | Vấn đề tính điểm | Điểm A | Điều chỉnh | Điểm nhóm | Đóng góp | Ghi chú |
|---|---|---|---|---|---|---|---|---|
| 1 | Hoàn thành nghiệp vụ | 8 | ERR-07 (M), ERR-08/09/10 (3L) | 8,25 | 0 | 8,25 | 6,60 | Bộ tính năng rất đầy đủ: 5 protocol, 14 extension, chia sẻ phiên, recording, MFA, SSO |
| 2 | Kiến trúc & module | 15 | ARCH-01 (M hệ thống), ARCH-02 (M), ARCH-03 (M), BE-01 (M), ARCH-04/05 (2L) | 5,00 | **+1** | 6,00 | 9,00 | Nâng 1 điểm: kiến trúc lõi đáng làm mẫu (process isolation + fexecve, plugin API nhiều implementation thật, phụ thuộc một chiều, common-js độc lập framework). Các vấn đề chủ yếu là lặp code và trạng thái trong RAM, không phải lỗi tổ chức |
| 3 | Bảo mật & phân quyền | 17 | SEC-01…06 (6H), SEC-07…22 (16M), SEC-23…29 (7L); SEC-30 Chưa đủ dữ liệu (0) | 0 | **+1** | 1,00 | 1,70 | Trừ ra âm, giữ ở 0. Nâng 1 điểm (mức tối đa cho phép): nền tảng bảo mật có nhiều điểm mạnh (không SQLi, token 256 bit, lọc READ trong SQL, path traversal chặn kỹ, non-root). Mô tả "3–4" hợp hơn "0–2", nhưng rubric chỉ cho điều chỉnh ±1. Có 4/6 High là rủi ro triển khai hoặc mặc định không an toàn |
| 4 | Mã độc & chuỗi cung ứng | 8 | MAL-01 (M) | 9,00 | 0 | 9,00 | 7,20 | Không có mã độc; lockfile đầy đủ, 100% registry chính thức; Maven ghim phiên bản |
| 5 | Database & dữ liệu | 10 | DB-01 (M), DB-02…05 (4L) | 8,00 | 0 | 8,00 | 8,00 | Ràng buộc đầy đủ, SQL được tham số hoá, entity gộp user/group hợp lý |
| 6 | Validation & xử lý lỗi | 8 | VAL-01, ERR-01, ERR-02, ERR-03 (4M), VAL-02/03, ERR-04/05/06 (5L, trừ tối đa 1) | 5,00 | 0 | 5,00 | 4,00 | Nhiều lỗi crash do input biên; khớp mô tả "5–6" |
| 7 | Clean code | 10 | CODE-01 (M hệ thống), CODE-02…06, PERF-01 (6L, trừ tối đa 1) | 7,50 | 0 | 7,50 | 7,50 | Doc comment dày, ít magic, `-Werror`; điểm trừ chính là race |
| 8 | Đặt tên & nhất quán | 6 | NAME-01, NAME-03 (2L), NAME-02 ([CHUẨN] L) | 9,25 | 0 | 9,25 | 5,55 | Quy ước tiền tố `guac_` / `__guac_` / `Guacamole.*` nhất quán |
| 9 | Kiểm thử | 10 | TEST-01, TEST-02 (2H), TEST-03, TEST-04 (2M) | 4,00 | **+1** | 5,00 | 5,00 | Nâng 1 điểm: test hiện có là test thật (88 hàm libguac, parser Java/JS, normalize path), không có test giả. Mô tả "5–6: có thật nhưng thiếu luồng rủi ro cao" khớp |
| 10 | Tài liệu & vận hành | 8 | OPS-01 (M hệ thống), OPS-02 ([CHUẨN] M), OPS-03 (M), OPS-04 ([CHUẨN] L), OPS-05, DOC-01 (L), DOC-02 ([CHUẨN] L) | 6,00 | 0 | 6,00 | 4,80 | Trừ: OPS-01 −1,5; OPS-03 −1; OPS-05, DOC-01 −0,5; nhóm [CHUẨN] (OPS-02 + OPS-04 + DOC-02 = 1,5) áp trần −1. Docker README và doc comment tốt; runtime EOL, thiếu quality gate |
| | **Tổng** | **100** | | | | | **59,35 → 5,94/10** | Trần: không áp dụng (0 Critical, không có MAL) · Nỗ lực: không áp dụng ở chuyên sâu |

**Làm tròn đến 0,25:** 5,935 → **6,00/10**.

## Kết luận release (chuyên sâu)

**6,00/10 — "Cần cải thiện trước release"** (khung 5,0–6,25). Không có Critical, nên không bị cổng "Không đạt".

"Release" ở đây được hiểu là **đưa vào production tự host**, vì dự án là upstream. Sau đây là các điều kiện bắt buộc.

**Điều kiện bắt buộc trước khi đưa vào production (người vận hành, không cần sửa code):**
1. `SEC-05`: đã đổi hoặc vô hiệu hoá `guacadmin`.
2. `SEC-08`: guacd không lộ cổng 4822 ra ngoài network nội bộ.
3. `SEC-06`: không cài quickconnect, hoặc đã giới hạn `allowed-parameters`.
4. `SEC-10`, `SEC-09`, `SEC-18`: reverse proxy có header bảo mật, che token trong log, xoá header auth từ client.
5. `SEC-22`, `SEC-21`, `SEC-17`: connection bật drive có `drive-path`, tắt printing khi không dùng, SSH có `host-key`.

**Điều kiện để đánh giá lại lên mức "Sẵn sàng có điều kiện" (cần sửa code):** xử lý `SEC-01`…`SEC-04`, `SEC-07`, `VAL-01`, và có lưới test `TEST-01`/`TEST-03` cho các phần đó.

## Ghi chú về độ tin cậy của điểm

- Rubric trừ theo **số lượng** vấn đề. Guacamole có khoảng 280k dòng và 14 extension độc lập; mỗi extension góp vấn đề riêng vào nhóm Bảo mật, nên nhóm này chạm đáy dù nền tảng bảo mật lõi tốt. Nên đọc điểm nhóm 3 cùng với `04-BAO-MAT.md` §8 (điểm mạnh).
- 1 vấn đề "Chưa đủ dữ liệu" (`SEC-30`) không bị trừ. Các vấn đề "Rủi ro tiềm ẩn" (`SEC-02`, `SEC-11`, `SEC-21`, `CODE-01`…) vẫn bị trừ theo rubric. Nếu kiểm chứng PoC cho thấy không khai thác được thì cần chấm lại.
- Đây là lần review đầu (chế độ Đầy đủ), nên chưa có cột "Lần trước".
