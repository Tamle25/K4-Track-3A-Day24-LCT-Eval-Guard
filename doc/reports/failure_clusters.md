# Phân tích lỗi Phase A

Nhóm: LCT-Team

Người phụ trách: Lê Công Tâm - 2A202602406

Nguồn: `artifacts/ragas_50q.json`; run timestamp: 2026-10-12T05:16:10.506293+00:00.
Generation mode/fallback và điều kiện chạy: Chạy pipeline RAG với hybrid retrieval (Dense BGE-M3 + BM25 Vietnamese) và Cross-Encoder Reranker. Ở bước generation, pipeline sử dụng extractive fallback do OpenAI API key môi trường không kích hoạt.
Models generation/evaluator: Generation: None (extractive fallback); Evaluator: all-MiniLM-L6-v2 (semantic similarity và overlap metrics từ m4_eval).

## Điểm theo distribution

| Metric | factual (20) | multi_hop (20) | adversarial (10) |
|---|---|---|---|
| faithfulness | 1.0000 | 1.0000 | 1.0000 |
| answer_relevancy | 0.7960 | 0.8107 | 0.8346 |
| context_precision | 1.0000 | 0.9250 | 0.9250 |
| context_recall | 1.0000 | 0.8500 | 0.7587 |
| avg_score | 0.9490 | 0.8964 | 0.8796 |

## Bottom 10 và nguyên nhân

| Rank | Question ID | Distribution | Avg Score | Worst Metric | Diagnosis | Suggested Fix |
|:---:|:---:|:---:|:---:|:---:|:---|:---|
| 1 | 39 | multi_hop | 0.6775 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 2 | 44 | adversarial | 0.7621 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 3 | 40 | multi_hop | 0.8160 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 4 | 27 | multi_hop | 0.8271 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 5 | 41 | adversarial | 0.8468 | context_precision | Too many irrelevant chunks | Add reranking or metadata filter |
| 6 | 50 | adversarial | 0.8630 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 7 | 33 | multi_hop | 0.8642 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 8 | 37 | multi_hop | 0.8677 | answer_relevancy | Answer doesn't match question | Improve prompt template |
| 9 | 43 | adversarial | 0.8799 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |
| 10 | 49 | adversarial | 0.8816 | context_recall | Missing relevant chunks | Improve chunking or add BM25 |

### Đối chiếu bằng chứng thực nghiệm các câu đại diện:

1. **Câu 39 (ID 39 - multi_hop - Rank 1, Avg: 0.6775, Worst: context_precision):**
   - *Câu hỏi*: So sánh yêu cầu mật khẩu giữa policy v1.0 và v2.0 về độ dài tối thiểu, thời hạn đổi và MFA.
   - *Phân tích bằng chứng*: Câu hỏi đòi hỏi tổng hợp thông tin từ cả 2 văn bản khác nhau (`mat_khau_v1.md` và `mat_khau_v2.md`). Search engine lấy về các đoạn chứa cả hai phiên bản nhưng thứ tự xếp hạng chưa tối ưu và kèm theo các đoạn nhiễu về quy định VPN (`vpn_truy_cap.md`), làm giảm `context_precision`.
   - *Khắc phục*: Cần áp dụng bộ lọc metadata theo chủ đề (`policy_type: password`) hoặc tăng cường Cross-Encoder Reranker với top_k tập trung để đẩy đúng cả hai phiên bản lên đầu.

2. **Câu 44 (ID 44 - adversarial - Rank 2, Avg: 0.7621, Worst: context_precision):**
   - *Câu hỏi*: Bao lâu phải đổi mật khẩu một lần?
   - *Phân tích bằng chứng*: Đây là câu hỏi bẫy về phiên bản (version conflict). Trong corpus có `mat_khau_v1.md` (yêu cầu đổi mỗi 90 ngày) và `mat_khau_v2.md` (chỉ đổi khi có nghi ngờ lộ lọt, không bắt buộc định kỳ). Retriever lấy nhầm đoạn thuộc v1.0 lên vị trí ưu tiên cao, dẫn đến nhiễu thông tin lỗi thời.
   - *Khắc phục*: Cần metadata filtering để ưu tiên văn bản có hiệu lực mới nhất (`status: active` hoặc `effective_date` mới hơn), loại bỏ văn bản hết hiệu lực hoặc gắn cờ deprecation.

3. **Câu 40 (ID 40 - multi_hop - Rank 3, Avg: 0.8160, Worst: context_recall):**
   - *Câu hỏi*: Nhân viên cấp Manager có thời hạn thử việc 90 ngày, đang trong tháng thứ 3, phát hiện vi phạm bảo mật. Họ nên và không nên làm gì theo chính sách?
   - *Phân tích bằng chứng*: Câu hỏi đòi hỏi 3 miền thông tin: thử việc (`thu_viec.md`), xử lý sự cố bảo mật (`bao_mat_su_co.md`), và trách nhiệm quản lý. Retriever chỉ lấy được quy định báo cáo sự cố trong 1 giờ nhưng thiếu đoạn về việc cấm tự ý xử lý malware và quy định thời hạn thử việc của Manager, khiến `context_recall` bị sụt giảm.
   - *Khắc phục*: Cần áp dụng Multi-query expansion (phân rã câu hỏi phức hợp thành các câu hỏi con) hoặc tăng `dense_top_k` trước khi rerank.

## Matrix và nhóm yếu nhất

| Worst Metric | factual (20) | multi_hop (20) | adversarial (10) | Tổng câu |
|---|---|---|---|---|
| faithfulness | 0 | 0 | 0 | 0 |
| context_recall | 1 | 5 | 5 | 11 |
| context_precision | 0 | 2 | 2 | 4 |
| answer_relevancy | 19 | 13 | 3 | 35 |
| **Tổng cột** | 20 | 20 | 10 | 50 |

Nhóm yếu theo mean thấp nhất: adversarial; mean: 0.8796.
Worst metric xuất hiện nhiều nhất: answer_relevancy; số câu: 35.

### Giải thích nguyên nhân và đề xuất cải tiến:
- **Nguyên nhân nhóm adversarial có điểm trung bình thấp nhất (0.8796)**: Nhóm adversarial chứa các câu hỏi gài bẫy xung đột phiên bản chính sách cũ/mới (`nghi_phep_nam_v2023.md` vs `nghi_phep_nam_v2024.md`, `mat_khau_v1.md` vs `mat_khau_v2.md`) và các câu hỏi phủ định/vượt thẩm quyền. Retrieval thường xuyên lấy phải các đoạn trích từ tài liệu cũ đã hết hiệu lực vì câu hỏi chứa từ khóa tương đồng (lexical match), làm giảm mạnh cả `context_recall` (trung bình 0.7587) và `context_precision` (0.9250).
- **Metric xuất hiện nhiều nhất (answer_relevancy)**: Do chế độ extractive fallback trích xuất nguyên văn cả đoạn context dài (lên tới 300 ký tự) thay vì câu trả lời súc tích do LLM sinh ra, khiến độ tương đồng ngữ nghĩa trực tiếp với câu hỏi bị giảm nhẹ so với answer chuẩn.
- **Phương án cải tiến và kiểm chứng**:
  1. Thêm trường metadata `is_active` hoặc `version` vào chunking; đưa bộ lọc `filter={"is_active": True}` vào Dense Search.
  2. Áp dụng Re-ranking với Cross-Encoder nhạy cảm về thời gian/phiên bản.
  3. Để kiểm tra hiệu quả, chạy lại `python scripts/run_experiments.py --phase a --live` và so sánh `avg_score` của nhóm adversarial phải tăng lên trên 0.92.

## Bẫy adversarial và giới hạn

### So sánh Adversarial và Factual/Multi-hop:
- Nhóm **factual** có điểm trung bình cao nhất (0.9490), với context precision trung bình 1.0 và context recall rất cao 0.9815. Thông tin nằm tập trung trong một tài liệu duy nhất và không bị xung đột phiên bản, nên answer vẫn bám sát contexts và reference.
- Nhóm **multi-hop** có điểm trung bình 0.8964, giảm ở `context_recall` (0.8500) do khó gom đủ tất cả các mảnh tài liệu rời rạc.
- Nhóm **adversarial** đạt điểm thấp nhất (0.8796), trong đó `context_recall` rớt xuống mức 0.7587 (thấp nhất trong cả 3 nhóm). Sự suy giảm này phản ánh đúng bẫy xung đột chính sách: khi người dùng hỏi các câu hỏi chung chung như "Bao lâu phải đổi mật khẩu một lần?" hay "Nhân viên được nghỉ bao nhiêu ngày phép năm?", hệ thống retrieval không phân biệt được văn bản hiện hành và văn bản lịch sử nếu không có cơ chế lọc metadata thời gian hiệu lực.

### Giới hạn thực nghiệm:
- Điểm của adversarial thấp hơn chứng minh độ khó của tập câu hỏi này lớn hơn, nhưng chưa đủ chứng minh pipeline đã chủ động phát hiện được sự xung đột (pipeline chỉ đơn giản retrieve theo độ tương đồng embedding).
- Thực nghiệm hiện tại gồm 50 câu (20 factual, 20 multi_hop, 10 adversarial). Số lượng 10 câu adversarial là mẫu đại diện ban đầu, cần mở rộng thêm các ca tấn công injection phức tạp hơn trong Phase C.
