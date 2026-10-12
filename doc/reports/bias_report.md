# Đánh giá LLM Judge Phase B

Nguồn: `artifacts/judge_results.json`.
Run timestamp: `2026-10-12T05:50:06.106223+00:00`.
Judge model: `gemini-3.1-flash-lite`.

## Năm cặp A/B và hai lượt

Nguồn cố định: A từ `artifacts/answers_50q.json:answer`; B từ
`fixtures/human_labels_10q.json:model_answer`; reference từ
`fixtures/test_set_50q.json:ground_truth`.

| ID | Winner pass 1 | Winner pass 2 (đã quy đổi) | Final | Consistent | Scores pass 1 | Scores pass 2 |
|---:|:---:|:---:|:---:|:---:|:---|:---|
| 1 | B | B | B | Có | A 0.8, B 1.0 | A 0.8, B 1.0 |
| 5 | A | A | A | Có | A 0.6, B 0.0 | A 0.7, B 0.1 |
| 12 | A | A | A | Có | A 1.0, B 0.5 | A 1.0, B 0.8 |
| 21 | B | B | B | Có | A 0.5, B 1.0 | A 0.5, B 1.0 |
| 41 | tie | tie | tie | Có | A 0.1, B 0.1 | A 0.1, B 0.1 |

Judge chọn B ở câu 1 vì câu trả lời ngắn và trực tiếp hơn; chọn A ở câu 5 và
12 vì A giữ điều kiện hoặc thông tin chính sách thiết yếu; chọn B ở câu 21 vì
B trả lời đúng và ngắn gọn; câu 41 là tie vì cả hai câu trả lời đều bị đánh giá
không đạt theo chính sách hiện hành. Hai lượt đều đồng thuận sau khi quy đổi,
không có case bất nhất cần chuyển thành tie.

Position bias count/rate: `0/5 = 0.0`.
Verbosity bias: `2/4 = 0.5`; có 4 case decisive. Hai case A thắng và A dài
hơn B; không có case B thắng và B dài hơn A. Đây là dấu hiệu trên chỉ 5 cặp,
không đủ để kết luận độ dài là nguyên nhân của quyết định.

## Calibration đúng model_answer

Calibration dùng đúng `question` và `model_answer` gốc trong
`fixtures/human_labels_10q.json`. Prompt binary chỉ nhận question, answer và
reference; nhãn người được lấy sau để đối chiếu.

| ID | Model answer gốc | Judge | Human | Agree | Tóm tắt reasoning |
|---:|---|---:|---:|:---:|---|
| 1 | Nhân viên được nghỉ 3 ngày làm việc có lương khi kết hôn. | 1 | 1 | Có | Đúng số ngày và tính chất có lương. |
| 5 | Cần Giám đốc phòng ban phê duyệt. | 0 | 0 | Có | 55 triệu vượt ngưỡng 50 triệu, cần Tổng Giám đốc. |
| 12 | Nhân viên được thưởng Tết tối thiểu 1 tháng lương. | 1 | 1 | Có | Đúng mức thưởng và điều kiện từ 6 tháng. |
| 21 | Được nghỉ 18 ngày phép (15 cơ bản + 3 thâm niên). Lương Senior từ 20 đến 35 triệu. | 1 | 1 | Có | Tính đúng phép và khoảng lương theo v2024. |
| 23 | Hoàn trả 25 triệu vì chưa đủ 1 năm cam kết. | 1 | 1 | Có | Đúng số tiền hoàn trả theo thời gian cam kết. |
| 29 | Trưởng phòng phê duyệt. Phạt 2% tháng trên 8 triệu. | 0 | 0 | Có | Thiếu Kế toán trưởng và tính phí phạt sai. |
| 33 | Phép: 19 ngày. Phụ cấp: 1.500.000 VNĐ/tháng. | 1 | 1 | Có | Tính đúng cả phép và phụ cấp. |
| 41 | Nhân viên được nghỉ 12 ngày phép năm. | 0 | 0 | Có | Dùng chính sách v2023 đã hết hiệu lực. |
| 46 | Nhân viên thử việc không được nghỉ phép năm và phải xin nghỉ không lương nếu cần. | 1 | 1 | Có | Đúng điều kiện và phương án thay thế. |
| 50 | Được, miễn là đảm bảo kết nối an toàn. | 0 | 0 | Có | VPN cá nhân bị cấm; phải dùng VPN công ty. |

Cohen's κ: `1.0`.

Observed agreement là `10/10 = 1.0`. Có 6 nhãn 1 và 4 nhãn 0 ở cả hai bộ,
nên agreement kỳ vọng là `p_e = 0.6 * 0.6 + 0.4 * 0.4 = 0.52`. Vì vậy
`κ = (1.0 - 0.52) / (1 - 0.52) = 1.0`.

## Giới hạn và cách dùng judge

Kết quả dùng `gemini-3.1-flash-lite`, 5 cặp pairwise với hai lượt đổi vị trí
và 10 binary judgments. Position bias không quan sát thấy trong bộ nhỏ này;
verbosity bias là 0.5 và cần được đọc như một tín hiệu mô tả, không phải kết
luận nhân quả. Calibration khớp người hoàn toàn, nhưng 10 mẫu không đủ chứng
minh judge luôn đúng. Khi dùng thực tế, cần mở rộng bộ nhãn, kiểm tra lại các
case lệch nhãn và tiếp tục giữ pairwise winner tách biệt với binary label.
