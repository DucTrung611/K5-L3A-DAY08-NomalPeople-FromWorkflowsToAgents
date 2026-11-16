# Báo cáo cá nhân — 2A202603449_ThanDucTrung

## 1. Thông tin thành viên

- Họ tên: Thân Đức Trung
- MSSV: 2A202603449
- Email: thanductrung123@gmail.com
- GitHub: DucTrung611

## 2. Phần việc phụ trách

- Shape A `workflow` (`src/lab8/shapes/workflow.py`) và integration: review/merge PR, chạy benchmark và crash suite, điền `report/team/decision_adr.md`, nộp bài.

## 3. Đóng góp cụ thể

- Commit chính: `d7ee66a` (feat: implement workflow shape, add tests and crash suite report)
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

- `python -m pytest -q -m student -o addopts= -k workflow` → `62 passed`; toàn bộ ma trận student 4 shape → `248 passed`.
- `python -m pytest -q` → `111 passed, 248 deselected` (toàn bộ host tests và unit tests của nhóm đều xanh).
- `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety 20/20` (`src/reports/crash.json`).
- `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json` → `src/reports/benchmark.json` (workflow đạt pass rate 1.000, pass^5 1.000, variance 0, cost/success 28.87 VND - thấp nhất trong 4 shape).
- Quét trace workflow trên 14 claim × seed 11, 55: không có `CANARY_`, `Bearer`, SĐT, địa chỉ, internal note, tên hàng hay nội dung ticket.
- Run id ví dụ: `R-workflow-C-014-s11` (T14: proposal 150000, ledger 120000), `R-workflow-C-002-s55` (T02: denied 0 dù ticket khai khống trễ 5 ngày).

## 5. Công cụ AI đã dùng

- Claude Code / Gemini CLI: hỗ trợ đọc và phân tích source code host, lập `PLAN.md`, gợi ý cấu trúc code workflow và các bộ test biên.
- Cách tự kiểm tra:
  - Tự đối chiếu từng nhánh rẽ nghiệp vụ với `docs/business_rules.md` và hợp đồng dữ liệu trong `docs/contracts.md`.
  - Chạy toàn bộ ma trận kiểm thử tự động: student matrix 248/248, cổng host 111/111, crash suite 20/20.
  - Tự viết script quét trace tìm secret và PII (`CANARY_`, `Bearer`, phone, address).
  - Tuyệt đối không dùng AI để sinh số liệu giả cho benchmark hay sửa tay các file JSON kết quả.

## 6. Tự đánh giá

- Mức độ hoàn thành: 100% công việc được phân công theo `PLAN.md`.
- Về mặt kỹ thuật: Xây dựng thành công Shape A (Workflow) theo đúng triết lý Least Autonomous Architecture: tốc độ thực thi nhanh nhất, chi phí mô phỏng thấp nhất (28.87 VND/success so với 49.57 VND của Swarm), độ tin cậy tuyệt đối (pass rate 1.000, variance = 0).
- Về mặt quản lý & integration: Đã xây dựng kế hoạch phân công rõ ràng, hỗ trợ các thành viên merge code không xung đột, hoàn thiện bộ benchmark và tài liệu quyết định kiến trúc (ADR) với đầy đủ dẫn chứng khoa học thực nghiệm.
- Điểm tự đánh giá: 10/10.
