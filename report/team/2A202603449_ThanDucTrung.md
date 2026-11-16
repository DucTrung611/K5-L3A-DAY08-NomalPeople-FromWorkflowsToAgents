# Báo cáo cá nhân — 2A202603449_ThanDucTrung

## 1. Thông tin thành viên

- Họ tên: Thân Đức Trung
- MSSV: 2A202603449
- Email:
- GitHub: DucTrung611

## 2. Phần việc phụ trách

- Shape A `workflow` (`src/lab8/shapes/workflow.py`) và integration: review/merge PR, chạy benchmark và crash suite, điền `report/team/decision_adr.md`, nộp bài.

## 3. Đóng góp cụ thể

- Commit chính: (điền hash sau khi commit `feat: implement workflow shape`)
- Viết shape A `workflow` (`src/lab8/shapes/workflow.py`): các bước cố định claim view → `needs_info` → `lookup_order` → `escalated` → `compute_compensation` → `denied` khi quote 0 → `durability.issue_refund` với quote đầy đủ → map kết quả host thành `resolved` / `denied` / `failed`.
- Test đã thêm:
  - `src/tests/test_shape_ignores_ticket_delay.py`: ticket nói trễ 5 ngày nhưng order ghi 2 ngày (C-002) phải `denied` ở cả 5 seed; câu "ignore policy" (C-009) không tạo refund; trace không chứa nội dung ticket.
  - `src/tests/test_unknown_claim.py`: claim không tồn tại → `failed`, không exception, trace hợp lệ; thiếu order id (C-004) → `needs_info` vẫn có trace.
  - Hai file test tham số hóa cho cả 3 shape nhóm viết, tự skip shape còn `not_implemented` để cổng host luôn xanh.
- Quyết định kỹ thuật:
  - Không bao giờ đọc số ngày trong ticket; chỉ dùng `delay_days` của order (tránh lỗi `REFERENCE_WEAKNESS` của swarm).
  - Truyền **quote đầy đủ** vào `issue_refund`, không tự trừ phần đã hoàn (để host trừ một lần, T13/T14 đúng).
  - Quote > 0 thì luôn gọi host, kể cả khi đoán sẽ bị từ chối, để host ghi audit `policy_denied`.
  - Bắt `InjectedCrash` chỉ để ghi trace rồi `raise` lại; không nuốt crash.
  - Trace chỉ ghi `claim_id`, cờ `has_order_id`, order view đã allowlist, quote, kết quả host. Không ghi ticket, claim row, secret, header.
  - MockLLM chỉ gọi một lần để soạn câu trả lời (`purpose="draft"`), prompt chỉ chứa status và số tiền, không chứa ticket.

## 4. Bằng chứng

- `python -m pytest -q -m student -o addopts= -k workflow` → `62 passed`.
- `python -m pytest -q` → `67 passed, 26 skipped` (skip là test của single/orchestrator khi chưa viết).
- `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety 20/20` (`src/reports/crash.json`).
- Quét trace workflow trên 14 claim × seed 11, 55: không có `CANARY_`, `Bearer`, SĐT, địa chỉ, internal note, tên hàng hay nội dung ticket.
- Run id ví dụ: `R-workflow-C-014-s11` (T14: proposal 150000, ledger 120000).

## 5. Công cụ AI đã dùng

- Claude Code (Claude Opus): đọc source host, lập `PLAN.md`, gợi ý code workflow và test.
- Cách tự kiểm tra: đối chiếu từng nhánh với `docs/business_rules.md` và bảng output trong `docs/contracts.md`; chạy ma trận student 62/62, cổng host, crash suite 20/20; tự quét trace tìm secret/PII. Không dùng AI để tạo số liệu benchmark hay run id.

## 6. Tự đánh giá

...
