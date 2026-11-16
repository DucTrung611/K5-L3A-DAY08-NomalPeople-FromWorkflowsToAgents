# Báo cáo cá nhân — 2A202603413_NguyenTrungAnh

## 1. Thông tin thành viên

- Họ tên: Nguyễn Trung Anh
- MSSV: 2A202603413
- Email:
- GitHub:

## 2. Phần việc phụ trách

- Shape B `single` (`src/lab8/shapes/single.py`): vòng ReAct tối đa 8 bước, model chỉ chọn tool đọc, code của shape gọi `durability.issue_refund`.

## 3. Đóng góp cụ thể

Commit chính (hash), test đã thêm hoặc sửa, quyết định kỹ thuật bạn đưa ra.

- Commit chính: `b85df25` — `feat(single): implement shape B ReAct agent with read-only tools` (nhánh `feat/single`).
- File sửa: `src/lab8/shapes/single.py`. Test thêm: `src/tests/test_single_step_cap.py` (16 test, không gắn marker `student` nên chạy trong cổng host).
- Quyết định kỹ thuật:
  1. **Ranh giới code / model.** Observation đưa cho agent chỉ có `claim_id`, `principal_id`, `order_id`. Ticket là dữ liệu người dùng không tin cậy nên không bao giờ vào observation, prompt MockLLM hay trace. Nhờ vậy câu "ignore policy" (T09) hay "trễ 5 ngày" (T02) không ảnh hưởng quyết định.
  2. **Agent chỉ chọn tool đọc**: `lookup_order` → `lookup_refund_history` → `compute_compensation` → final. `run_react(max_steps=8, worker=False)`; `dispatch` của host chặn write tool.
  3. **Final của agent chỉ là tín hiệu dừng.** Sau vòng lặp, code của shape đọc lại kết quả tool, **tính lại quote** từ order row do `lookup_order` trả về (không tin tham số agent truyền vào `compute_compensation`), rồi là nơi duy nhất gọi `durability.issue_refund`. Nếu quote của agent khác quote của code thì ghi event `quote_mismatch` và dùng quote của code.
  4. **Truyền quote đầy đủ** vào `issue_refund` (T13, T14); `lookup_refund_history` chỉ để ghi trace, không tự trừ.
  5. **Crash:** truyền `crash_at`, khi gặp `InjectedCrash` thì ghi trace rồi `raise` lại (T12).
  6. Chạm trần 8 bước hoặc tool bị từ chối → `failed`, không ghi ledger, không ném exception.
  7. **Trace** chỉ ghi: `claim` (claim_id, has_order_id), `react_step` (tên tool), `order_view` (các cột allowlist của `lookup_order`), `refund_history` (số lần, tổng tiền), `quote`, `host` (ok, reason, conflict), `decision`. Cố ý không ghi ticket, `parent_secret`, `raw_headers`, SĐT, địa chỉ, internal note, tên hàng, claim row hay order row thô.

## 4. Bằng chứng

Link commit, tên test, run id hoặc trích đoạn report liên quan.

- Ma trận student cho shape single: `python -m pytest -q -m student -o addopts= -k "single and not workflow and not orchestrator and not swarm"` → `62 passed` (12 claim × 5 seed, T11, T12).
- Cổng host: `python -m pytest -q` → `70 passed, 248 deselected`.
- `python -m lab8.check_submission` → `[ OK ] shape single: format đúng, 12/12 claim public pass ở seed 11`.
- Run mẫu `R-single-C-014-s55`: `resolved`, `proposal_amount` 150000, ledger 120000, 5 model call, 5 tool call. `R-single-C-002-s55`: `denied` 0 (đúng ở seed mà swarm fail).
- Test trong `test_single_step_cap.py`: `test_single_stays_under_step_cap_and_uses_model`, `test_single_never_dispatches_a_write_tool`, `test_single_step_cap_fails_safely`, `test_single_ignores_model_tampered_quote`, `test_single_trace_has_no_ticket_text`.

## 5. Công cụ AI đã dùng

Công cụ, dùng vào việc gì, và bạn đã tự kiểm tra kết quả thế nào (xem [RULES.md](../../RULES.md)).

- Claude Code: đọc đề và source host, gợi ý và viết code `single.py`, viết test `test_single_step_cap.py`.
- Cách tự kiểm tra: chạy ma trận student của shape single (62 passed), cổng host (70 passed), `check_submission`; chạy tay 60 trial single (12 claim × 5 seed) rồi grep trace tìm `CANARY`, `Bearer`, SĐT, địa chỉ, tên hàng, `ticket` — không có kết quả.

## 6. Tự đánh giá

...
