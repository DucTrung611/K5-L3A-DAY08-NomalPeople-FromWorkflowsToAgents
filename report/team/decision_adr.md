# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: workflow
- Commit hash: 41e59e5 (code 4 shape + `src/reports/benchmark.json`, `src/reports/crash.json` sinh trên commit này)
- Run id dẫn chứng: R-workflow-C-002-s55 (T02 ticket nói trễ 5 ngày, workflow vẫn denied), R-workflow-C-014-s11 (T14 proposal 150000, ledger 120000), R-swarm-C-002-s55 (swarm fail cùng trial); file `src/reports/benchmark.json`
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000–1.000 / 0.000000 (70 trial = 14 claim public × seed 11,22,33,44,55)
- Cost per success (mô phỏng, VND): 28.87 (workflow) · single 47.55 · swarm 49.57 · orchestrator 56.95
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 / 0 trên bộ public (70/70 trial pass; tiêu chí pass gồm quét chuỗi cấm trong trace + kết quả và khớp ledger)

## Vì sao shape này

Trên cùng corpus (14 claim public T01–T14), cùng 5 seed, cùng MockLLM, bảng giá và evaluator (`src/reports/benchmark.json`), ba shape nhóm viết đều đạt pass rate 1.000, pass^5 1.000, variance 0; swarm đạt 0.986 / pass^5 0.929 vì tin số ngày trong ticket ở seed 55 (T02). Khi độ đúng ngang nhau, workflow là shape **ít tự chủ nhất** và **rẻ nhất**: 28.87 VND/success, bằng 61% single (47.55), 58% swarm (49.57) và 51% orchestrator (56.95); mỗi trial chỉ 1 lời gọi model (soạn câu trả lời) và trung bình 2.79 tool call. Toàn bộ quyết định tiền, approval và idempotency nằm trong code, nên kết quả không phụ thuộc seed theo thiết kế, không chỉ theo số đo.

## Có đáng đổi lấy multi-agent không

Không. Trên bộ public, single và orchestrator không thắng thêm trial nào so với workflow nhưng tốn hơn 65–97% cost/success; swarm vừa đắt hơn vừa có variance > 0. Nhóm giữ workflow. Quyết định sẽ đảo lại nếu: (1) ticket thật cần hiểu ngôn ngữ mơ hồ mà luật cố định không bắt được (ví dụ khách mô tả hỏng hàng nhưng chưa có cờ `damage_verified`, cần model quyết định có hỏi thêm không), khi đó single có thể pass cao hơn; (2) bộ hidden cho thấy workflow fail ở nhóm claim mà agent xử lý đúng, với chênh pass rate đủ lớn để bù cost; (3) tool đọc chậm và độc lập nhau, khi đó investigator song song của orchestrator có thể giảm critical path (chưa đo được vì MockLLM không có latency).

## Trace và bằng chứng

Workflow ghi các event: `claim_view` (claim_id, found, has_order_id), `lookup_order` (visible, reason, order view đã qua allowlist `ORDER_FIELDS`: order_id, status, delay_days, shipping_fee_vnd, paid_vnd, damage_verified, carrier), `quote` (amount_vnd, reason, clause_ids, parts), `issue_refund` (ok, reason, conflict), `decision` (status, proposal_amount, reason), và `crash` khi có crash injection. Cố ý **không** ghi: nội dung ticket (dữ liệu người dùng, có thể chứa injection/PII), claim row thô, `parent_secret`, `raw_headers`, SĐT, địa chỉ, internal note, tên hàng, và prompt gửi MockLLM. Trace chỉ ghi vào `src/traces/<run_id>.json`, không có file phụ hay stdout nào khác, nên scanner của grader quét được toàn bộ. Nhóm tự quét thêm trace workflow trên 14 claim × seed 11, 55 với danh sách SĐT, địa chỉ, note, tên hàng và ticket lấy từ world state: 0 trùng. Test `src/tests/test_shape_ignores_ticket_delay.py` khẳng định ticket không xuất hiện trong trace của cả 3 shape.

## Giới hạn

MockLLM không phải model sản xuất: nó deterministic theo seed và không hiểu ngôn ngữ, nên chênh lệch giữa các shape ở đây phản ánh topology và số lời gọi, không phản ánh chất lượng suy luận. Cost là giá token mô phỏng theo `config/pricing.yaml`, không phải hóa đơn API. Latency để `unavailable`. Bộ public chỉ có 14 claim; pass rate 1.000 chưa chứng minh được ngưỡng 0.75 trên 40 claim hidden, và chưa có `student_drop/` để tự kiểm. Không sửa JSON trong `src/reports/` cho đủ ngưỡng.
