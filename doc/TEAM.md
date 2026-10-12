# Nhóm thực hiện K4 Track 3A

Tên nhóm: LCT

Repo chung: https://github.com/Tamle25/K4-Track-3A-Day24-LCT-Eval-Guard

Trưởng nhóm/người nộp: Lê Công Tâm — 2A202602406

Ngày lab/deadline áp dụng: 2026-10-12 — Asia/Ho_Chi_Minh (UTC+07:00)

Commit dùng để chấm: 1b95087

Mỗi nhóm nộp một repo. Điền đủ thành viên và một mục INDIVIDUAL cho từng người.
Thêm/xóa rows để khớp nhóm thật; mẫu ba rows không ép nhóm phải có ba người.

## Thành viên và phân công

| STT | Họ và tên | MSSV | Vai trò / công việc thật | Bằng chứng |
|---|---|---|---|---|
| 1 | Lê Công Tâm | 2A202602406 | Tích hợp và hoàn thiện Phase A/B/C; chạy live experiments; kiểm tra artifacts và reports | `artifacts/*.json`, `doc/reports/*.md`, `src/phase_a_ragas.py`, `src/phase_b_judge.py`, `src/phase_c_guard.py` |

Người tích hợp/chạy batch/nộp link: Lê Công Tâm — chạy các batch Phase A/B/C,
kiểm tra bằng `python scripts/check_lab.py` và chịu trách nhiệm nộp link repo.

## Môi trường và chuẩn bị trước lớp

OS/Python/compiler: Windows; Python 3.11.9; Microsoft Visual C++ Build Tools
14.44; `annoy==1.17.3` build thành công.
Virtualenv/dependencies/imports: Presidio, spaCy, NeMo Guardrails và các
dependency liên quan đã được cài; Phase C offline tests đạt 72 passed.
Models/Qdrant/50 answers đã chuẩn bị: `artifacts/answers_50q.json` có đủ 50
answers; Phase A/B/C artifacts đã được tạo. Guard dùng `gemini-3.1-flash-lite`;
latency input-layer P95 đo được vượt budget 500 ms và đã ghi trung thực trong
blueprint. API key chỉ lưu trong `.env`, không commit vào repo.
Nếu cài full NeMo chưa thành công, ghi lỗi thực tế; không dùng unit test pass
hoặc pip resolve để khẳng định runtime đã chạy.

## INDIVIDUAL

Sao chép mẫu cho mỗi thành viên; tiêu đề họ tên không dấu-MSSV.

### LeCongTam-2A202602406

- Phần được giao và phần thực tế đã làm: triển khai và tích hợp các task
  Phase A/B/C; sửa adapter/provider cho LLM Judge và NeMo; hoàn thiện latency
  measurement, reports và blueprint.
- Bằng chứng: `artifacts/ragas_50q.json`, `artifacts/judge_results.json`,
  `artifacts/guard_results.json`, `doc/reports/failure_clusters.md`,
  `doc/reports/bias_report.md`, `doc/reports/blueprint.md`.
- Cách kiểm tra: `python scripts/check_lab.py`; `python -m pytest
  tests/test_phase_c.py tests/test_support.py tests/test_report_contracts.py
  -q`; kết quả offline gần nhất 72 passed. Live Phase C đạt attacks 20/20,
  clean 8/8 và output 4/4.
- Điều đã học: phân biệt pairwise judge với binary calibration, quy đổi
  winner sau swap A/B, tính Cohen's kappa, giữ evidence raw cho guard latency
  và phân biệt input-layer latency với full-request latency.
- Giới hạn/cách xử lý: NeMo self-check gọi model từ xa nên input-layer P95
  vượt 500 ms; RAG, output rail và full request chưa đo trong Task 12, nên
  không suy diễn các phần này từ `input_guard_ms`.
- Nguồn/AI hỗ trợ nếu có: sử dụng Copilot SDK trong VS Code để hỗ trợ đọc
  contract, debug dependency và rà soát; mọi kết quả live, artifact và test
  đều được tự chạy và kiểm chứng trong repo.

## Cập nhật sau khi nộp

Phiên bản nộp: commit `1b95087`.
Chưa có cập nhật sau deadline.
