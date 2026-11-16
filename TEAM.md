# Team

## Thông tin nhóm

- Tên nhóm: NomalPeople (Nomal People)
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-NomalPeople-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: https://github.com/DucTrung611/K5-L3A-DAY08-NomalPeople-FromWorkflowsToAgents, nhánh `main`, commit chốt: (điền sau khi chạy `git rev-parse --short HEAD`, trùng `report/team/decision_adr.md`)

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Thân Đức Trung | 2A202603449 | thanductrung123@gmail.com | DucTrung611 | Nhóm trưởng · Shape A `workflow` + integration (merge PR, benchmark, crash suite, ADR) |
|   2 | Nguyễn Trung Anh | 2A202603413 | | | Shape B `single` (ReAct) |
|   3 | Trần Anh Tú | 2A202603368 | atus011205@gmail.com | | Shape C `orchestrator` + handoff |
|   4 | Trần Mạnh Hùng | 2A202603496 | | | QA (crash, secret, trace) + báo cáo nhóm, research card |

Gợi ý chia vai:

- Nhóm 6 người: integration; workflow; single agent; orchestrator; ledger/policy/crash; eval/report.
- Nhóm 4 người: workflow + integration; single; orchestrator + handoff; ledger + evaluation.

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Thân Đức Trung (2A202603449) | workflow + integration | `d7ee66a` | `src/tests/test_shape_ignores_ticket_delay.py`, `src/tests/test_unknown_claim.py` |
| Nguyễn Trung Anh (2A202603413) | single | b85df25 | `src/tests/test_single_step_cap.py` |
| Trần Anh Tú (2A202603368) | orchestrator + handoff | `3c1abd4` | `src/tests/test_orchestrator_handoff.py` |
| Trần Mạnh Hùng (2A202603496) | QA + báo cáo | | `src/tests/test_shape_trace_hygiene.py`, `src/tests/test_shape_full_quote.py` |

## Cá nhân

Mỗi thành viên có một báo cáo riêng trong `report/team/`:

- [report/team/2A202603449_ThanDucTrung.md](report/team/2A202603449_ThanDucTrung.md)
- [report/team/2A202603413_NguyenTrungAnh.md](report/team/2A202603413_NguyenTrungAnh.md)
- [report/team/2A202603368_TranAnhTu.md](report/team/2A202603368_TranAnhTu.md)
- [report/team/2A202603496_TranManhHung.md](report/team/2A202603496_TranManhHung.md)

## 2A202603449_ThanDucTrung

### Phần việc phụ trách

- Shape A `workflow` (`src/lab8/shapes/workflow.py`), integration: review và merge PR, chạy benchmark và crash suite, điền ADR, nộp bài.

### Đóng góp cụ thể

-

### Bằng chứng

-

## 2A202603413_NguyenTrungAnh

### Phần việc phụ trách

- Shape B `single` (`src/lab8/shapes/single.py`): vòng ReAct tối đa 8 bước, model chỉ chọn tool đọc.

### Đóng góp cụ thể

- Viết shape `single` (commit `b85df25`): agent chỉ chọn `lookup_order` → `lookup_refund_history` → `compute_compensation`; sau vòng lặp code tự tính lại quote từ order row và là nơi duy nhất gọi `durability.issue_refund` (quote đầy đủ, truyền `crash_at`).
- Thêm `src/tests/test_single_step_cap.py` (16 test, chạy trong cổng host).

### Bằng chứng

- `python -m pytest -q -m student -o addopts= -k "single and not workflow and not orchestrator and not swarm"` → 62 passed.
- `python -m pytest -q` → 70 passed. `python -m lab8.check_submission` → `[ OK ] shape single: format đúng, 12/12 claim public pass ở seed 11`.

## 2A202603368_TranAnhTu

### Phần việc phụ trách

- Shape C `orchestrator` (`src/lab8/shapes/orchestrator.py`): handoff allowlist, investigator chỉ đọc, trace chỉ ghi handoff đã sanitize.

### Đóng góp cụ thể

- Triển khai Shape C với investigator chỉ đọc, handoff allowlist, host ghi refund và trace sạch.
- Thêm test cho quyền tool, secret trong handoff/trace và nhánh return sớm.

### Bằng chứng

- Commit `3c1abd4`; 62/62 student test orchestrator và 2/2 test handoff mới pass.

## 2A202603496_TranManhHung

### Phần việc phụ trách

- QA: crash suite, quét secret/PII trong trace, test trần hoàn tiền theo đơn; viết `report/team/group_report.md` và `report/team/research_card.md`.

### Đóng góp cụ thể

-

### Bằng chứng

-
