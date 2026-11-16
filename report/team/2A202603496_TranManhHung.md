# Báo cáo cá nhân — 2A202603496_TranManhHung

## 1. Thông tin thành viên

- Họ tên: Trần Mạnh Hùng
- MSSV: 2A202603496
- Email:
- GitHub:

## 2. Phần việc phụ trách

- QA: crash suite, quét secret/PII trong trace, test trần hoàn tiền theo đơn; viết `report/team/group_report.md` và `report/team/research_card.md`.

## 3. Đóng góp cụ thể

- Thêm 2 file test QA cốt lõi (33 test cases chạy trong cổng host):
  - `src/tests/test_shape_trace_hygiene.py`: Quét trace cả 3 shape trên các claim nhạy cảm (C-001, C-004, C-005, C-007, C-010, C-013, C-014); khẳng định không lộ `CANARY_SECRET_*`, `Bearer`, SĐT, địa chỉ, internal notes, tên hàng hay nội dung ticket. Đặc biệt kiểm tra ca C-005 truy cập chéo principal không lộ PII.
  - `src/tests/test_shape_full_quote.py`: Kiểm tra bảo toàn quote đầy đủ và trần theo đơn: T14 (đề xuất 150.000đ, ledger ghi 120.000đ), T13 (đã hoàn đủ 35.000đ từ trước -> từ chối có audit `policy_denied`), T08 (approval trần 20.000đ thấp hơn quote 35.000đ).
- Đồng tác giả báo cáo nghiên cứu `report/team/research_card.md` (chủ đề R2: Failure taxonomy theo MAST NeurIPS 2025, phân tích lỗi ungrounded consensus của Swarm ở seed 55) và báo cáo nhóm `report/team/group_report.md`.
- Quyết định kỹ thuật:
  - Ranh giới bảo mật: Trace phải được kiểm tra fail-closed, không để bất kỳ trường thô nào từ DB hay ticket lọt vào trace JSON.
  - Trần hạn mức theo đơn (Order Cap): Shape bắt buộc phải truyền quote đầy đủ vào `durability.issue_refund` để Host tự tính phần chênh, tránh lỗi trừ hai lần.

## 4. Bằng chứng

- `python -m pytest -q tests/test_shape_trace_hygiene.py tests/test_shape_full_quote.py` → `33 passed`.
- Cổng host: `python -m pytest -q` → `144 passed, 248 deselected`.
- `src/reports/crash.json`: `crash_safety 20/20` (không có ca nào tạo duplicate refund).
- Quét thực tế trên toàn bộ trace sinh ra: `0` trường hợp leak secret hoặc PII.

## 5. Công cụ AI đã dùng

- Gemini CLI / Claude Code: hỗ trợ thiết kế các ca kiểm thử bảo mật và đối chiếu rule order cap với hợp đồng hệ thống.
- Cách tự kiểm tra:
  - Chạy thực nghiệm toàn bộ test suite trên local.
  - Quét regex chuỗi cấm trực tiếp trên thư mục `src/traces/*.json`.
  - Đối chiếu số liệu bảng bồi thường và audit events với SQLite DB.

## 6. Tự đánh giá

- Mức độ hoàn thành: 100% phần việc QA, kiểm thử bảo mật trace và trần bồi hoàn theo đơn.
- Đóng góp: Giúp nhóm phát hiện sớm các nguy cơ rò rỉ PII trong trace, đảm bảo vượt qua 3 điều kiện must-pass (không bypass approval, không leak secret, crash an toàn 20/20).
- Điểm tự đánh giá: 10/10.
