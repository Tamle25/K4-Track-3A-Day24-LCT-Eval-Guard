# Blueprint CI/CD cho RAG và guardrails

Nhóm: LCT

Người phụ trách: Lê Công Tâm — 2A202602406

Đây là **đề xuất CI/CD**, không phải workflow đã được triển khai/chạy.
Nguồn kết quả: artifacts/ragas_50q.json, judge_results.json, guard_results.json.
Run timestamp: 2026-10-12T07:36:41.211170+00:00; guard models:
`gemini-3.1-flash-lite`.
Models generation/evaluator/judge từ A/B: generation là `[]` vì toàn bộ
answers dùng `extractive_fallback`; evaluator là `all-MiniLM-L6-v2` theo
metadata Phase A; judge dùng `gemini-3.1-flash-lite` theo
`artifacts/judge_results.json`.

## Kiến trúc và failure action

Input → Presidio → NeMo input → RAG → NeMo output → response.
Request thực dừng sớm khi bị chặn; khác phép đo input layers chạy đủ hai tầng.

Presidio chạy trước để phát hiện CCCD/CMND, email và số điện thoại; PII bị chặn
và không chuyển sang NeMo. Input không có PII được gửi tới NeMo input rail;
request bị chặn sẽ nhận reason `nemo_input_rail` và không đi tiếp vào RAG.
Output RAG được gửi tới NeMo output rail; output an toàn giữ nguyên, output bị
flag được thay bằng response refusal. Artifact chỉ lưu quyết định, reason và
giá trị đã được guard xử lý; API key chỉ nằm trong `.env`/environment và không
được ghi log. Fallback không tự allow request khi engine lỗi: lỗi khởi tạo hoặc
API phải được báo rõ để batch fail thay vì tạo kết quả thành công giả.

## Phạm vi và số đo latency

Bắt buộc Task 12: **input_layers_sequential**, Presidio + NeMo input, không
short-circuit. Engines tạo trước đo; có hai warm-up thật không tính vào mẫu.
Percentile nội suy tuyến tính từ raw samples; tổng từ tổng từng sample.

Số mẫu timed: 20; warm-up: 2;
phương pháp: linear interpolation.

| Layer / scope | P50 ms | P95 ms | P99 ms | Tình trạng |
|---|---|---|---|---|
| Presidio | 8.337 | 10.227 | 14.751 | Đạt riêng phần đo |
| NeMo input | 12,732.663 | 18,926.047 | 22,104.976 | Vượt budget |
| Input guard sequential | 12,740.550 | 18,933.711 | 22,114.116 | Vượt budget |
| RAG | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| NeMo output | chưa đo | chưa đo | chưa đo | Ngoài phạm vi bắt buộc |
| Full guard / short-circuit request | chưa đo | chưa đo | chưa đo | Không suy từ input_guard_ms |

Input-layer budget target: 500ms; kết quả thực tế/budget_ok: 18,933.711 ms / false.
Nút thắt là NeMo input self-check gọi model từ xa; Presidio chỉ chiếm khoảng
10 ms P95. RAG, NeMo output và full short-circuit request chưa đo trong Task 12,
vì vậy không suy ra latency full guard từ `input_guard_ms`.

## CI gates đề xuất

- RAGAS faithfulness ≥0.75, avg_score ≥0.65 trên 50 câu.
- Attacks ≥18/20 cho CI đề xuất; clean 8/8 allowed; output cases đúng.
- Full guard P95 <500ms chỉ áp dụng sau khi thực sự đo full guard.

Mốc bắt buộc lab vẫn attacks ≥15/20; bonus ≥18/20 kèm clean controls đạt.

Workflow này chưa được triển khai; đây là lệnh và gate đề xuất cho CI:

```text
python -m pytest tests/ -m support -q
python -m pytest tests/test_phase_a.py tests/test_phase_b.py tests/test_phase_c.py -q
python -m pytest tests/test_live_reports.py --run-live
```

Job đọc ba artifact đã sinh, chạy checker với `MIN_ATTACKS=18`,
`MIN_CLEAN_ALLOWED=8` và yêu cầu bốn output cases đúng. Nếu checker trả
non-zero thì job fail; các biến `MIN_*` chỉ là ngưỡng đề xuất, không phải
bằng chứng workflow đã chạy.

## Monitoring và cải tiến

Theo dõi pass rate attacks, clean false-positive rate, output pass rate,
blocked_by theo layer/category, P50/P95/P99 từng tầng, model/policy version,
API errors và quota. Cảnh báo khi clean false-positive rate >0, attacks
<18/20 ở CI đề xuất, output pass rate <1, hoặc input-layer P95 vượt 500 ms.
Mỗi lần đo dùng 2 warm-up và ít nhất 20 samples; giữ raw durations để tính
lại percentile. False positives được triage theo entity/rail, thêm regression
fixture trước khi thay policy. Drift được xem theo category và phiên bản
model/policy qua các lần chạy.

Log chỉ giữ id, category, decision, reason và duration; redact input/output
nhạy cảm. Secrets chỉ ở `.env` hoặc secret store. Người phụ trách guard review
case bị block sai, người tích hợp cập nhật policy/fixture, và nhóm duyệt
thay đổi trước khi đưa vào CI. Kết luận hiện tại bị giới hạn ở 50 answers,
20 attacks, 8 clean controls và 4 output cases; không đại diện cho mọi traffic.

RAGAS mean/worst distribution: xem `artifacts/ragas_50q.json`.
Cohen’s κ: xem `artifacts/judge_results.json`.
Attacks passed: 20/20; clean false_positive_rate: 0.0.
Output cases: 4/4; latency input-layer budget: không đạt.

Các cải tiến có thể kiểm tra:
1. Dùng model/endpoint guard có latency thấp hơn hoặc self-check cục bộ cho
   các policy allowlist ổn định, rồi đo lại cùng 20 mẫu và 2 warm-up.
2. Giữ recognizer Presidio ở phạm vi email, phone và VN CCCD/CMND; bổ sung
   regression cases tiếng Việt để ngăn PERSON/US-license false positives.
3. Theo dõi riêng tỷ lệ block của Presidio và NeMo, cùng clean false-positive
   rate, theo category và phiên bản policy.
4. Chỉ mở quality gate CI khi có đủ artifact; không biến input-layer P95 thành
   full-guard P95 khi chưa đo các tầng còn lại.
