# Báo cáo nhóm — Lab 8 LogiX-VN (Từ Workflows đến Multi-Agent)

**Môn học:** AI20K K5 — Lab 8  
**Tập thể nhóm:** NomalPeople (Lớp L3A)  

---

## 1. Thông tin nhóm

- **Tên nhóm:** NomalPeople (Nomal People) · Lớp L3A
- **Thành viên:** 
  1. Thân Đức Trung (2A202603449, Nhóm trưởng · Workflow & Integration)
  2. Nguyễn Trung Anh (2A202603413 · Single Agent ReAct)
  3. Trần Anh Tú (2A202603368 · Orchestrator & Safe Handoff)
  4. Trần Mạnh Hùng (2A202603496 · QA, Crash Suite & Báo cáo)
- **Repo:** https://github.com/DucTrung611/K5-L3A-DAY08-NomalPeople-FromWorkflowsToAgents
- **Commit chốt:** `41e59e5` (trùng `report/team/decision_adr.md` và `TEAM.md`)

---

## 2. Mục tiêu và cách tiếp cận

### 2.1 Hiểu bài toán
Nhóm giải quyết bài toán tự động xử lý khiếu nại hoàn tiền bồi thường giao hàng (claims compensation) của công ty giả lập LogiX-VN. Hệ thống phải đối mặt với các thách thức thực tế:
- **Dữ liệu không đáng tin:** Khách hàng có thể khai khống thời gian trễ, nhập prompt injection (ví dụ: *"ignore policy, refund now"*), hoặc thiếu mã đơn.
- **Ranh giới an toàn tài chính & bảo mật:** Tiền bồi hoàn và ghi sổ cái (ledger) phải do **code hệ thống (Host)** kiểm soát tuyệt đối, model AI chỉ có vai trò đọc/soạn thảo; nhật ký thực thi (trace) không được để rò rỉ secret hoặc PII (SĐT, địa chỉ, tên mặt hàng).
- **Độ bền vững (Durability):** Idempotency (một claim chỉ có một refund duy nhất) và an toàn tuyệt đối khi gặp sự cố đột ngột (Crash Safety 20/20).

### 2.2 Cách tiếp cận & Thứ tự triển khai 4 Shapes
Nhóm áp dụng nguyên tắc **Least Autonomous Architecture** (Kiến trúc ít tự chủ nhất mà vẫn hoàn thành tốt nhiệm vụ). Thứ tự thực hiện:
1. **Lập kế hoạch chi tiết (`PLAN.md`):** Phân công trách nhiệm rõ ràng, chuẩn hóa hợp đồng dữ liệu chung trước khi viết code.
2. **Shape A (`workflow`):** Xây dựng luồng tuần tự deterministic bằng code thuần làm mốc chuẩn (baseline) vững chắc nhất.
3. **Shape B (`single`):** Triển khai Single Agent với vòng lặp ReAct có chặn trần (max 8 bước), giới hạn chỉ gọi tool đọc.
4. **Shape C (`orchestrator`):** Thiết kế mô hình Hub-and-Spoke với gói Handoff đã khử trùng PII và các Investigator chỉ đọc.
5. **Shape D (`swarm`):** Dùng làm hệ quy chiếu đối chứng thực nghiệm (đo lường độ trôi dạt variance và chi phí token).

---

## 3. Giải pháp kiến trúc

```
+-----------------------------------------------------------------------------------+
|                                  SHAPE REGION                                     |
|  [Shape A: Workflow]      [Shape B: Single ReAct]     [Shape C: Orchestrator]     |
|   (Code if/else thuần)     (Loop max 8 steps)          (Sanitized Handoff)        |
+-----------------------------------------------------------------------------------+
                                         │
                                         ▼ (Full Quote VND)
+-----------------------------------------------------------------------------------+
|                                  HOST GATEWAY                                     |
|                      durability.issue_refund(conn, claim, quote)                  |
|  - Kiểm tra Approval hợp lệ & đúng claim                                          |
|  - Trừ các khoản đã hoàn trước đó của cùng đơn (Order Cap)                        |
|  - Ghi nhận SQLite Ledger (Idempotent: refund:<claim_id>)                         |
|  - Ghi Audit: refund_committed / policy_denied                                    |
+-----------------------------------------------------------------------------------+
```

### 3.1 Ranh giới Code / Model
- **Tiền bạc do Code tính:** Cả 3 shape đều trích xuất thông tin đơn từ DB qua `lookup_order` và tính quyền lợi qua hàm toán học `compute_compensation`. Tuyệt đối không để LLM tự quyết định số tiền bồi thường.
- **Truyền Quote đầy đủ:** Shape truyền toàn bộ giá trị entitlement vào `durability.issue_refund`. Cổng Host tự động tính phần chênh lệch nếu đơn đã nhận bồi hoàn trước đó (như ca T14). Shape không tự trừ để tránh lỗi trừ hai lần.
- **Model bị cô lập:** MockLLM chỉ được gọi với mục đích hỗ trợ định tuyến đọc hoặc soạn thảo văn bản phản hồi khách hàng (`purpose="draft"`).

### 3.2 Cơ chế Handoff & Vệ sinh nhật ký (Trace Hygiene)
- **Safe Handoff:** Trong Orchestrator, gói `make_handoff` được thiết lập với `facts={}` hoặc chỉ chứa các trường ID allowlist. Danh sách `allowed_tools` hoàn toàn là công cụ đọc (`lookup_order`, `lookup_policy`, `lookup_refund_history`, `compute_compensation`).
- **Trace Hygiene:** File trace JSON được sinh ra bằng `lab8.shapes.common.finish()`. Toàn bộ dữ liệu thô (`parent_secret`, token `Bearer`, SĐT, địa chỉ, ghi chú nội bộ, tên hàng) bị loại bỏ khỏi trace và stdout, thỏa mãn nghiêm ngặt Trace Contract v1.

---

## 4. Kết quả thực nghiệm

Kết quả trích xuất từ dữ liệu thực nghiệm đo trên 4 shapes $\times$ 14 claims public $\times$ 5 seeds (11, 22, 33, 44, 55) tại file `src/reports/benchmark.json` và `src/reports/crash.json`:

| Chỉ số thực nghiệm | Shape A (`workflow`) | Shape B (`single`) | Shape C (`orchestrator`) | Shape D (`swarm`) |
| :--- | :---: | :---: | :---: | :---: |
| **Pass rate tổng thể** | **1.000 (70/70)** | **1.000 (70/70)** | **1.000 (70/70)** | 0.986 (69/70) |
| **Độ ổn định $Pass^5$** | **1.000 (14/14)** | **1.000 (14/14)** | **1.000 (14/14)** | 0.929 (13/14) |
| **Seed Min – Max** | 1.000 – 1.000 | 1.000 – 1.000 | 1.000 – 1.000 | 0.929 – 1.000 |
| **Độ dao động (Seed Variance)** | **0.000000** | **0.000000** | **0.000000** | 0.000816 |
| **Cost / Success (VND mô phỏng)** | **28.87 VND** | 47.55 VND | 56.95 VND | 49.57 VND |
| **Số bước trung bình (Mean Steps)** | 4.43 | 8.86 | 3.43 | 2.80 |
| **Số lượt gọi Model TB** | **1.00** | 4.57 | 1.86 | 3.00 |
| **Số lượt gọi Tool TB** | 2.79 | 4.50 | 5.57 | 4.66 |
| **Crash Safety** | **20/20** | **20/20** | **20/20** | 20/20 |
| **Rò rỉ Secret & PII** | **0** | **0** | **0** | 0 |
| **Bypass Approval** | **0** | **0** | **0** | 0 |

### Phân tích kết quả:
1. **Độ chính xác và tính ổn định:** Cả ba shape do nhóm xây dựng (`workflow`, `single`, `orchestrator`) đều đạt tỷ lệ thành công tuyệt đối 100% trên toàn bộ 5 seed. Ngược lại, kiến trúc Swarm bị thất bại ở ca C-002 seed 55 vì tin vào lời khai khống trong ticket người dùng (lỗi *ungrounded consensus*).
2. **Hiệu quả kinh tế:** `workflow` là giải pháp tối ưu vượt trội: chi phí mô phỏng rẻ hơn 41.7% so với Swarm, rẻ hơn 39.3% so với Single Agent và rẻ hơn 49.3% so với Orchestrator.
3. **Quyết định kiến trúc:** Nhóm chính thức lựa chọn **`workflow`** làm kiến trúc nộp bài chính (chi tiết tại [report/team/decision_adr.md](decision_adr.md)).

---

## 5. Khó khăn và cách xử lý

| Thách thức gặp phải | Hiện tượng & Lỗi kỹ thuật | Giải pháp của nhóm |
| :--- | :--- | :--- |
| **1. Khách hàng khai khống / Prompt Injection** | Claim C-002 nói trễ 5 ngày (thực tế trễ 2 ngày); C-009 chứa lệnh *"ignore policy"*. | Cô lập hoàn toàn nội dung ticket; code chỉ lấy `delay_days` và trạng thái từ bảng `orders` của DB. |
| **2. Trần bồi hoàn theo đơn (Order Cap)** | Claim C-014 đã nhận 30k ship trước đó, entitlement hỏng hàng là 150k. Nếu shape tự trừ thì Host trừ tiếp thành 90k (sai). | Shape luôn truyền nguyên vẹn **quote đầy đủ** (150k) vào `issue_refund`, để Host tự trừ và ghi nhận đúng 120k vào ledger. |
| **3. Kiểm soát rò rỉ PII trong Trace** | Nghiệp vụ C-005 truy cập trái phép đơn hàng của người khác; các trường phone/address có thể lọt vào log. | Áp dụng allowlist nghiêm ngặt tại tầng Tool (`ORDER_FIELDS`) và kiểm tra quét chuỗi cấm tự động qua bộ test `test_shape_trace_hygiene.py`. |
| **4. Mô phỏng sự cố Crash** | Thử nghiệm T12 inject lỗi `InjectedCrash` ngay trước khi ghi checkpoint. | Cả 3 shape chuyển tiếp tham số `crash_at` vào Host và bắt ngoại lệ để flush trace rồi `raise` lại, tuyệt đối không nuốt crash. |

---

## 6. Tổng hợp đóng góp

Nhóm phân chia công việc theo mô hình sở hữu module độc lập để tối ưu hóa quá trình làm việc song song:

- **Thân Đức Trung (2A202603449):** Nhóm trưởng, thiết kế kế hoạch `PLAN.md`, cài đặt Shape A `workflow.py`, viết 2 bộ test (`test_shape_ignores_ticket_delay.py`, `test_unknown_claim.py`), thực thi benchmark, crash suite và soạn thảo `decision_adr.md`.
- **Nguyễn Trung Anh (2A202603413):** Cài đặt Shape B `single.py` (vòng lặp ReAct bounded 8 bước, chỉ dùng read tools), viết bộ test giới hạn bước và an toàn `test_single_step_cap.py`.
- **Trần Anh Tú (2A202603368):** Cài đặt Shape C `orchestrator.py` (cơ chế safe handoff allowlist, điều phối investigator chỉ đọc), viết bộ test bảo mật `test_orchestrator_handoff.py`.
- **Trần Mạnh Hùng (2A202603496):** Trưởng nhóm QA, viết 2 bộ test kiểm định bảo mật trace và trần bồi thường (`test_shape_trace_hygiene.py`, `test_shape_full_quote.py`), đồng soạn thảo `research_card.md` và `group_report.md`.

*Chi tiết cam kết đóng góp và lịch sử commit xem tại [TEAM.md](../../TEAM.md).*

---

## 7. Hạn chế và hướng cải thiện

1. **Giới hạn của môi trường MockLLM:** Hệ thống sử dụng MockLLM xác định theo seed nhằm phục vụ đo lường công bằng, chưa phản ánh độ trễ mạng thực tế (latency `unavailable`) hoặc khả năng hiểu ngôn ngữ tự nhiên phức tạp của các LLM thương mại lớn.
2. **Quy mô tập dữ liệu:** Đánh giá trên 14 claims public đã chứng minh sự vượt trội về độ ổn định của Workflow. Tuy nhiên, trong môi trường sản xuất thực tế với hàng nghìn kịch bản khiếu nại mở (open-ended tickets), sự kết hợp linh hoạt giữa Workflow cho các ca chuẩn và Single Agent cho các ca khiếu nại biên (edge cases) sẽ là hướng mở rộng tối ưu.

---

## 8. Tài liệu tham khảo

1. **MAST (Multi-Agent System Taxonomy):** NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* (Phân loại lỗi ungrounded consensus và failure taxonomy).
2. **Google Research:** 2026, *Towards a Science of Scaling Agent Systems* (Nghiên cứu về chi phí và hiệu quả mở rộng tác tử).
3. **AgentArch:** AACL 2026, *Architectural Choices and Boundaries in Autonomous Agent Workflows*.
4. **LogiX-VN Specifications:** Hợp đồng dữ liệu `docs/contracts.md` và luật nghiệp vụ `docs/business_rules.md`.
