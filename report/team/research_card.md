# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R2 |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | MAST (Multi-Agent System Taxonomy), NeurIPS 2025, https://arxiv.org/abs/2410.02115, benchmark & empirical failure taxonomy |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | Hệ thống multi-agent gặp lỗi chủ yếu do ungrounded consensus (đồng thuận sai lệch), hallucination propagation giữa các peer và thiếu cơ chế kiểm chứng độc lập (verification failure) đối với input không tin cậy. |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Đánh giá trên các benchmark multi-agent tương tác (WebArena, GAIA, SWE-bench) với các LLM agent đối thoại tự do. |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Nếu chuyển từ Swarm (peer discussion) sang Workflow (code thuần có ground-truth verification từ DB), lỗi verification failure ở các claim có ticket giả mạo (C-002 seed 55) sẽ bị triệt tiêu, pass rate tăng từ 0.986 lên 1.000, seed variance giảm từ 0.000816 về 0.000000. |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Cố định: MockLLM, công cụ đọc (lookup_order, lookup_policy), 5 seeds (11,22,33,44,55), 14 claims public, evaluator trial_pass, bảng giá pricing.yaml. Biến thay đổi duy nhất: Topology/Architecture (Swarm peer discussion vs Workflow deterministic code). |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Lỗi kiểm chứng (verification failure) do tin ticket text thay vì DB: đo bằng pass rate (kỳ vọng 1.000 vs 0.986), pass^5 (1.000 vs 0.929), seed variance (0.000000 vs 0.000816). |

## Kết quả ablation

- Lệnh đã chạy (copy nguyên văn): `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json`
- Run id / file report: `src/reports/benchmark.json`, trial `R-swarm-C-002-s55` (fail: `status resolved != denied`) vs `R-workflow-C-002-s55` (pass: `denied` 0).
- Kết quả so với hypothesis (đúng / sai / không kết luận được), kèm số liệu: **Đúng**. Workflow đạt pass rate 1.000 (70/70), pass^5 1.000, seed variance 0.000000. Swarm bị fail đúng ở trial C-002 seed 55 do lỗi ungrounded consensus (tin ticket delay 5 ngày thay vì order delay 2 ngày), pass rate giảm còn 0.986 (69/70), pass^5 0.929, seed variance 0.000816. Chi phí của Workflow cũng giảm 41.7% (28.87 VND vs 49.57 VND/success).
- Giới hạn: MockLLM không phải model sản xuất, latency là `unavailable`.

Muốn đi xa hơn một ablation để lấy điểm cộng thì xem [docs/bonus_topics.md](../../docs/bonus_topics.md).
