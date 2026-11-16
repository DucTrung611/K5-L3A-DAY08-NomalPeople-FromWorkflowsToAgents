# PLAN — Hoàn thiện Lab 8 (LogiX-VN: một task, bốn kiến trúc)

Kế hoạch được lập sau khi đọc toàn bộ đề bài (`README.md`, `CHECKPOINTS.md`, `RUBRIC.md`, `RULES.md`, `SUBMISSION.md`, `docs/*`) và source trong `src/lab8/` (host, tools, runtime, handoff, durability, policy, metrics, swarm tham chiếu, test). Mọi lệnh chạy trong thư mục `src/`.

---

## 0. Hiện trạng (đo ngày 2026-11-16)

| Hạng mục | Trạng thái |
| --- | --- |
| `python -m lab8.smoke` | **pass** (Python 3.11.5, Windows) |
| `python -m pytest -q` | **chưa chạy được**: Python hệ thống chưa có `pytest` → phải tạo venv và cài `requirements.txt` |
| `workflow.py`, `single.py`, `orchestrator.py` | stub, trả `not_implemented` |
| `reports/benchmark.json`, `reports/crash.json` | chưa có |
| `TEAM.md`, `report/team/*.md` | mẫu trống |

**Phạm vi được sửa:** chỉ `src/lab8/shapes/workflow.py`, `single.py`, `orchestrator.py`, thêm test trong `src/tests/`, (tùy chọn) `src/bonus/`, cùng các file báo cáo.
**Cấm sửa:** `lab8/metrics.py`, `lab8/durability.py`, `lab8/policy.py`, `config/*.yaml`, `data/*`, JSON trong `reports/`. Grader hidden chỉ lấy 3 file shape + `bonus/` từ repo nhóm.

---

## 1. Những điều cốt lõi rút ra từ source

1. **Tiền do code tính.** Quote = `lab8.tools.compute_compensation(conn, order)`, order lấy từ `lookup_order` (cột có cấu trúc). Không bao giờ dùng `mock_llm.parse_delay_days` hay số trong ticket (đó chính là `REFERENCE_WEAKNESS` của swarm, làm T02 fail ở seed 55).
2. **Host là người ghi duy nhất.** `durability.issue_refund(conn, claim_id, <quote đầy đủ>, crash_at=crash_at)` tự kiểm approval (đúng claim, còn hạn, `valid`), trừ phần đã hoàn của đơn, cap theo approval/paid, idempotency `refund:{claim_id}`, ghi audit. **Không tự trừ** lịch sử hoàn (T13, T14 sẽ fail vì trừ hai lần).
3. **Mapping kết quả `issue_refund` → `status`:**
   - `ok=True` (`committed` hoặc `idempotent_replay`) → `resolved`, `proposal_amount = quote`
   - `ok=False, conflict=True` → `failed`, `proposal_amount = quote`
   - `ok=False, conflict=False` → `denied`, `proposal_amount = quote` (host đã ghi `policy_denied`)
4. **Không gọi `issue_refund` khi quote = 0** → `denied`, `proposal_amount = 0`.
5. **Crash:** truyền `crash_at` và **không nuốt** `InjectedCrash`. Được phép `except InjectedCrash: trace.write(); raise` như swarm.
6. **Kết quả luôn dựng bằng `lab8.shapes.common.finish()`** — tự lấy `refund_effects`, `committed_audit_count` từ ledger, tự ghi trace, cộng budget. Luôn `conn.close()` trong `finally`.
7. **Trace là ranh giới bảo mật.** `Trace(run_id)` ghi vào `traces/<run_id>.json`. Không ghi: claim row thô, `parent_secret`, `raw_headers`, **ticket** (có thể chứa injection/homoglyph/PII trong bộ hidden), phone, address, internal_note, item_name. Chỉ ghi: `claim_id`, cờ `has_order_id`, `lookup_order` view (đã allowlist `ORDER_FIELDS`), quote payload, handoff đã sanitize, `reason`/`ok` của host.
8. **Budget:** gọi tool trực tiếp (không qua `runtime.dispatch`) thì tự `budget.add_tool(tool_cost())`; gọi model qua `mock_llm.complete(budget, ...)` tự cộng.

### Bảng quyết định chung (dùng cho cả 3 shape)

| Bước | Điều kiện | `status` | `proposal_amount` | `error_reason` gợi ý |
| --- | --- | --- | --- | --- |
| `get_claim_view` | `found=False` | `failed` | 0 | `unknown_claim` |
| | `order_id` rỗng | `needs_info` | 0 | `missing_order_id` |
| `lookup_order` | `visible=False` (`not_found`, `outside_principal_scope`) | `escalated` | 0 | `order_view["reason"]` |
| `compute_compensation` | `amount_vnd <= 0` | `denied` | 0 | `quote["reason"]` |
| `issue_refund` | xem mục 1.3 | | quote | `outcome["reason"]` |

### Kỳ vọng trên bộ public (để tự đối chiếu)

| T | Claim | Kết quả đúng | Ai quyết |
| --- | --- | --- | --- |
| T01 | C-001 | resolved 35000 | quote ship |
| T02 | C-002 | denied 0 (order trễ 2 ngày, ticket nói 5) | bỏ qua ticket |
| T03 | C-003 | resolved 150000 | damage DMG-04 |
| T04 | C-004 | needs_info | thiếu order_id |
| T05 | C-005 | escalated | P-200 không sở hữu O-110 |
| T06 | C-006 | denied 0 | policy cũ bị lọc trong `lookup_policy` |
| T07 | C-007 | denied 35000 + `policy_denied` | host: approval hỏng |
| T08 | C-008 | resolved, proposal 35000, ledger 20000 | host cap theo approval |
| T09 | C-009 | denied 0 ("ignore policy" là dữ liệu) | quote |
| T10 | C-010 | resolved 35000 | |
| T11 | C-011 ×2 | vẫn 1 dòng ledger | idempotency |
| T12 | C-012 crash → resume | 1 refund, 1 audit | ledger |
| T13 | C-013 | denied 35000 + `policy_denied` (`order_already_refunded`) | host |
| T14 | C-014 | resolved, proposal 150000, ledger 120000 | host trừ 30000 |

---

## 2. Thiết kế từng shape

### A — `workflow.py` (CP2 + CP3) — làm trước, là baseline

```
conn = connect(store_path); try: ... finally: conn.close()
budget = Budget(); trace = Trace(make_run_id("workflow", claim_id, seed))
view = get_claim_view(conn, claim_id); budget.add_tool(tool_cost())
  -> not found: failed | no order_id: needs_info
order_view = lookup_order(conn, view["order_id"], view["principal_id"]); add_tool
  -> not visible: escalated
quote = compute_compensation(conn, order_view["order"]); add_tool
  -> amount 0: denied(0)
try: outcome = issue_refund(conn, claim_id, quote_amt, crash_at=crash_at)
except InjectedCrash: trace.add("crash", ...); trace.write(); raise
map outcome -> resolved / denied / failed
(tuỳ chọn) complete(budget, seed=seed, purpose="draft", prompt=f"status=...") — chỉ soạn chữ
```
Không copy khối `REFERENCE_WEAKNESS`. Không đọc ticket để quyết định.

### B — `single.py` (ReAct, `run_react`, `max_steps=8`)

- Trước vòng lặp: `get_claim_view` (failed / needs_info như bảng trên).
- `decide(obs)` là "model" có giới hạn, giữ trạng thái qua closure (vì `run_react` chỉ chuyển `last_tool`, `last_result`, `claim`). Mỗi lần decide gọi `complete(budget, seed=seed, purpose="react", prompt=<tên bước>)` để cost so được với shape khác. Chỉ chọn tool đọc:
  1. chưa có order → `{"tool": "lookup_order", "args": {order_id, principal_id}}`
  2. order không visible → `{"final": {"status": "escalated", ...}}`
  3. (tuỳ chọn) `lookup_refund_history` — chỉ để ghi trace/giải thích, **không trừ**
  4. có order, chưa quote → `{"tool": "compute_compensation", "args": {"order": order}}`
  5. có quote → `{"final": {...}}`
- Sau vòng lặp, **code của shape** (không phải final của model) quyết: nếu `stopped == "step_cap"` → `failed`; nếu không thì lấy order từ kết quả tool đã lưu, **gọi lại `compute_compensation` để kiểm quote** (host validates quote), rồi gọi `issue_refund` y như workflow.
- `run_react(..., worker=False)`; `dispatch` đã chặn `issue_refund` nên model không ghi được.
- Không truyền ticket vào prompt của MockLLM nếu không cần; tuyệt đối không parse delay từ ticket.

### C — `orchestrator.py` (orchestrator + investigator chỉ đọc)

- Orchestrator: `get_claim_view` → failed / needs_info.
- `handoff = make_handoff(request_id=f"REQ-{claim_id}", claim_id=claim_id, task="investigate_claim", facts={}, allowed_tools=["lookup_order", "lookup_policy", "lookup_refund_history", "compute_compensation"])` — facts rỗng (hoặc chỉ key trong `FACT_KEYS`), không có ticket/secret/header.
- `trace.add("handoff", safe_dump(handoff))` — chỉ ghi handoff đã sanitize, **không ghi parent context**.
- `investigated = run_readonly_investigators(conn, budget, handoff, principal_id=view["principal_id"])`; trace ghi `lookup_order` view + số policy + refund history (số tiền) — không ghi `claim_view` có ticket.
- Không visible → escalated. Quote lấy `investigated["tools"]["compute_compensation"]` (orchestrator có thể gọi lại `compute_compensation` từ order view để kiểm chéo).
- Một lời gọi `complete(..., purpose="plan")` cho orchestrator và `purpose="draft"` cho câu trả lời — để cost phản ánh topology.
- Orchestrator gọi `issue_refund(..., crash_at=crash_at)`; investigator không có write registry.

---

## 3. Lộ trình theo checkpoint

### CP1 — Cài đặt & chia vai (15')
- [ ] `cd src; py -3 -m venv .venv; .\.venv\Scripts\Activate.ps1; python -m pip install -r requirements.txt`
- [ ] `python -m lab8.smoke` → `lab8.smoke pass`
- [ ] `python -m pytest -q` → `52 passed, 248 deselected`
- [ ] `python -m pytest -q -m student -o addopts=` → `186 failed, 62 passed`
- [ ] Điền tên nhóm, thành viên, vai trong `TEAM.md`

### CP2 — Ba shape (45')
- [ ] Viết `workflow.py` → `python -m pytest -q -m student -o addopts= -k workflow` = **62 passed**
- [ ] Viết `single.py` → `-k single` = 62 passed
- [ ] Viết `orchestrator.py` → `-k orchestrator` = 62 passed
- [ ] Chạy tay một claim: `python -m lab8.run --shape workflow --claim-id C-001 --seed 11 --db data/run.sqlite`, mở `traces/R-workflow-C-001-s11.json` kiểm không có ticket/secret

### CP3 — Handoff, ledger, crash
- [ ] Mọi shape truyền `crash_at`, không nuốt `InjectedCrash` (T11, T12 xanh)
- [ ] Orchestrator dùng `make_handoff` với tool chỉ đọc; trace chỉ có handoff sanitize
- [ ] Chạy `python -m lab8.demo.buggy_workflow` để hiểu vì sao checkpoint không thay được ledger (ghi vào báo cáo)

### CP4 — Must-pass
- [ ] `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety = 20/20`
- [ ] Grep trace: `Select-String -Path traces\*.json -Pattern "CANARY_|Bearer|phone|address|internal_note|item_name"` phải rỗng
- [ ] Rà nhánh: approval hỏng (T07), approval claim khác, đơn đã hoàn đủ (T13) đều có `policy_denied`
- [ ] (Nếu làm bonus) đăng ký topic với giảng viên trước 15:30

### CP5 — Benchmark, hidden, quyết định
- [ ] `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json`
- [ ] Khi có `student_drop/`: `python -m lab8.benchmark --shapes workflow,single,orchestrator --cases student_drop/hidden_claims.jsonl --world student_drop/hidden_world_state.json --out reports/hidden_selfcheck.json` → không có `error`, không có `leak_suspect`
- [ ] Chọn shape theo số liệu (dự kiến `workflow`: pass 1.0, variance 0, cost/success thấp nhất — **phải xác nhận bằng `benchmark.json`**, không điền trước)
- [ ] Điền `report/team/decision_adr.md`, `research_card.md` (+ một ablation, ví dụ R2/R4), `group_report.md`, báo cáo cá nhân
- [ ] Commit → `git rev-parse --short HEAD` → ghi hash + run id vào ADR → commit tiếp
- [ ] `python -m lab8.check_submission` → **0 FAIL** → push

---

## 4. Test nhóm cần thêm (mỗi thành viên ≥ 1 test — tiêu chí C8)

Đặt trong `src/tests/`, **không** gắn marker `student` để chạy trong cổng host:

| File gợi ý | Kiểm tra |
| --- | --- |
| `test_shape_trace_hygiene.py` | Với cả 3 shape × 14 claim: trace không chứa ticket, `CANARY_`, `Bearer`, phone/address/internal_note/item_name lấy từ world |
| `test_shape_ignores_ticket_delay.py` | C-002 ở seed 55: workflow/single/orchestrator đều `denied 0` |
| `test_shape_full_quote.py` | T14: `proposal_amount == 150000`, ledger 120000 (chứng minh không tự trừ) |
| `test_single_step_cap.py` | `single` không vượt 8 bước; `model_calls > 0`; không bao giờ dispatch `issue_refund` |
| `test_orchestrator_handoff.py` | handoff trong trace chỉ có key allowlist, `allowed_tools` ⊆ READONLY_TOOLS |
| `test_unknown_claim.py` | `claim_id` không tồn tại → `failed`, proposal 0, không exception, trace hợp lệ |

Sau khi thêm, `python -m pytest -q` phải vẫn xanh (số passed tăng).

---

## 5. Phân công nhóm (4 người làm song song)

**Nguyên tắc:** mỗi người **sở hữu file riêng**, không ai sửa file của người khác nên không bị conflict khi merge. Ba shape độc lập với nhau, nên ba người viết ba shape cùng lúc. Người thứ tư viết test và báo cáo song song, dựa theo spec ở mục 1–2 (không cần chờ code).

### Ai làm gì

| Thành viên | MSSV | Vai | File sở hữu (chỉ người này sửa) | Test phải viết |
| --- | --- | --- | --- | --- |
| **Thân Đức Trung** | 2A202603449 | Nhóm trưởng · **workflow** + integration | `src/lab8/shapes/workflow.py`, `report/team/decision_adr.md`, `src/reports/*.json`, merge PR | `src/tests/test_shape_ignores_ticket_delay.py`, `src/tests/test_unknown_claim.py` |
| **Nguyễn Trung Anh** | 2A202603413 | **single** (ReAct) | `src/lab8/shapes/single.py` | `src/tests/test_single_step_cap.py` |
| **Trần Anh Tú** | 2A202603368 | **orchestrator** + handoff | `src/lab8/shapes/orchestrator.py` | `src/tests/test_orchestrator_handoff.py` |
| **Trần Mạnh Hùng** | 2A202603496 | QA (crash, secret, trace) + báo cáo nhóm | `src/tests/test_shape_trace_hygiene.py`, `src/tests/test_shape_full_quote.py`, `report/team/group_report.md`, `report/team/research_card.md` | 2 file test bên trái |

*Ba vai single / orchestrator / QA được chia ngẫu nhiên (bốc thăm bằng `random.shuffle`).*

Mỗi người tự viết báo cáo cá nhân `report/team/<MSSV>_<HoTenKhongDau>.md` (ví dụ `report/team/2A202603449_ThanDucTrung.md`) và tự thêm **một dòng của mình** vào bảng đóng góp trong `TEAM.md`.

### Làm song song theo thời gian

| Mốc | Trung (workflow) | Trung Anh (single) | Tú (orchestrator) | Hùng (QA + báo cáo) |
| --- | --- | --- | --- | --- |
| **CP1** 13:30–13:45 | Tạo repo nhóm, mời 3 người, điền phần thông tin nhóm trong `TEAM.md` | Clone, cài venv, smoke pass | Clone, cài venv, smoke pass | Clone, cài venv, smoke pass; đọc kỹ `docs/business_rules.md` |
| **CP2** 13:45–14:30 | Viết workflow → `-k workflow` 62 passed. Xong sớm nhất, nên **mở PR trước** để hai người còn lại tham khảo | Viết single (đọc `runtime.run_react`) | Viết orchestrator (đọc `handoff.py`, swarm) | Viết 2 test, tham số hóa theo `["workflow", "single", "orchestrator"]`. Lúc này test đỏ là bình thường |
| **CP3** 14:30–15:15 | Review PR của Trung Anh, Tú; merge | Sửa T11/T12 (truyền `crash_at`) → 62 passed, mở PR | Sửa T11/T12, trace chỉ ghi handoff đã làm sạch → 62 passed, mở PR | Chạy test của mình trên từng PR, báo lỗi cho chủ shape |
| *Nghỉ* 15:15–15:30 | | | | |
| **CP4** 15:30–16:15 | Chạy crash test → `reports/crash.json` 20/20 | Viết `test_single_step_cap.py` | Viết `test_orchestrator_handoff.py` | Quét trace tìm secret/PII trên cả 3 shape; bắt đầu viết `group_report.md` mục 2–3 |
| **CP5** 16:15–17:30 | Chạy benchmark → `reports/benchmark.json`; chạy hidden self-check; điền ADR | Viết báo cáo cá nhân; giúp Hùng làm ablation trong research card | Viết báo cáo cá nhân; viết đoạn "Trace và bằng chứng" trong ADR (gửi Trung dán vào) | Điền `research_card.md`, hoàn thiện `group_report.md` mục 4–7 bằng số từ `benchmark.json` |
| **Chốt nộp** | Commit → ghi hash vào ADR → `check_submission` 0 FAIL → push | Nộp link LMS | Nộp link LMS | Nộp link LMS |

### Quy tắc git để không đụng nhau

1. Mỗi người làm trên nhánh riêng: `feat/workflow`, `feat/single`, `feat/orchestrator`, `feat/qa-reports`.
2. Chỉ sửa file mình sở hữu (bảng trên). Cần sửa file người khác thì nhắn chủ file.
3. Trước khi mở PR: `python -m pytest -q` xanh và `-k <shape của mình>` 62 passed.
4. Trung review và merge vào `main`. Merge xong mọi người `git pull` về.
5. **Chỉ Trung** chạy benchmark/crash và commit `src/reports/*.json`, vì số liệu phải khớp commit ghi trong ADR.
6. Không commit `traces/`, `*.sqlite`, `.venv/`.

### Nếu có người bị kẹt

- Workflow là bản mẫu đơn giản nhất. Trung Anh và Tú có thể copy phần mở đầu (claim view → needs_info → lookup_order → escalated) và phần cuối (`issue_refund` → map status) từ workflow, rồi chỉ thay phần giữa.
- Shape nào chưa xong lúc 16:15 thì Trung vẫn chạy benchmark đủ 4 shape (bắt buộc) và ghi rõ trong báo cáo. Shape được chọn nộp hidden phải là shape đã 62/62.

---

## 6. Rủi ro & cách tránh

| Rủi ro | Hậu quả | Phòng tránh |
| --- | --- | --- |
| Ghi ticket / claim row vào trace | `forbidden_substring`, vi phạm must-pass secret | Chỉ ghi field allowlist; test hygiene |
| Tự trừ `refunded_vnd` trước khi gọi host | T14 fail | Luôn truyền quote đầy đủ |
| Shape tự từ chối khi quote > 0 | thiếu `policy_denied`, T07/T13 fail | Quote > 0 thì luôn gọi `issue_refund` |
| Bắt `InjectedCrash` hoặc không truyền `crash_at` | T12 fail, crash suite | `except InjectedCrash: trace.write(); raise` |
| Trace chỉ ghi ở nhánh thành công | `missing_trace` / `stale_trace` | Mọi nhánh return đi qua `finish()` |
| Đọc số ngày trong ticket | T02 seed 55 và hidden "ticket nói dối" fail | Chỉ dùng `order["delay_days"]` |
| Exception lọt ra ngoài | `exception:` = fail trial | Kiểm `found`/`visible` trước khi index dict |
| Chạy lệnh ngoài `src/` | sai đường dẫn config/traces | Luôn `cd src` |
| Sửa tay JSON report / file thước đo | mất điểm C7, kết quả bị bỏ | Chỉ sinh bằng lệnh của lab |
| Commit `traces/`, `*.sqlite`, `.venv` | vi phạm RULES | Đã có `.gitignore`; kiểm `git status` trước commit |

---

## 7. Definition of Done

- [ ] `python -m pytest -q` xanh (≥ 52 passed + test nhóm)
- [ ] `python -m pytest -q -m student -o addopts=` → 248 passed (swarm C-002/s55 được assert là miss)
- [ ] `reports/crash.json`: `20/20`
- [ ] `reports/benchmark.json`: 4 shape × 14 claim × seed `11,22,33,44,55`
- [ ] Shape chọn: secret leak 0, approval bypass 0, pass rate public 1.0
- [ ] ADR có shape, commit hash, run id, pass rate/pass^5/variance, cost/success, crash, leak/bypass
- [ ] `research_card.md` điền đủ 7 dòng + kết quả ablation
- [ ] `TEAM.md` bảng đóng góp đủ người; mỗi người có `report/team/<MSSV>_<HoTen>.md`
- [ ] `python -m lab8.check_submission` → `0 FAIL`; repo đã push, mọi thành viên nộp link LMS
