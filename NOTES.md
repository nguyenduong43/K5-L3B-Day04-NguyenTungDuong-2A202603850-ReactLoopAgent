# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Tùng Dương
MSSV: 2A202603850

## 1. Case được chọn

- ID case: H6
- Câu trả lời của agent:
Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn.
(Agent đã đọc trạng thái GitHub Actions và incident h02n9572mx9y, tuy nhiên dừng lại ở Step 1 mà không truy cập incident liên kết xr3myzj7ms51 để tìm nguyên nhân gốc rễ, dẫn đến câu trả lời chưa làm rõ được nguyên nhân có trong bản tóm tắt gộp hay không).
- Điểm ở lần chạy 1: 60/100 (Bị áp trần điểm do vi phạm cơ học: "Cần tool calls ở ít nhất 2 model steps khác nhau.")

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
Step 1:
- get_status: provider='github', component='GitHub Actions'
  -> Trả về danh sách incident và 40 mục nhiễu (distractors) giả lập.
- get_incident_details: provider='github', incident_id='h02n9572mx9y'
  -> Result update: "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"

Sau Step 1, Agent dừng gọi tool và trả lời ngay mà không thực hiện gọi tiếp tool ở Step 2 để kiểm tra incident liên quan xr3myzj7ms51.
```

Nguyên nhân:
- Câu hỏi yêu cầu "theo liên kết của nó" để xem incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng.
- Trong cập nhật của `h02n9572mx9y`, hệ thống chỉ dẫn rõ liên kết đến `xr3myzj7ms51`. Tuy nhiên, Agent bị phân tâm bởi lượng dữ liệu nhiễu và chưa có cơ chế bắt buộc đào sâu khi gặp related incident, dẫn đến dừng quá sớm ở Step 1 và vi phạm điều kiện cơ học của rubric ("Cần tool calls ở ít nhất 2 model steps khác nhau").

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Cải tiến `SYSTEM_PROMPT` trong `agent.py`: bổ sung chỉ thị rõ ràng yêu cầu Agent khi điều tra incident mà nội dung cập nhật có dẫn link hoặc nhắc đến sự cố liên quan (related incident, combined summary) thì BẮT BUỘC phải tiếp tục gọi tool ở bước sau để kiểm tra sự cố liên quan đó nhằm tìm nguyên nhân và phạm vi ảnh hưởng đầy đủ.
  2. Tăng cường phòng vệ trước prompt injection và lỗi tool trong `SYSTEM_PROMPT`: Khi tool gặp lỗi/timeout, phải báo rõ không thể kiểm tra, tuyệt đối không được suy đoán hay khẳng định dịch vụ bình thường kể cả khi người dùng yêu cầu.
  3. Cơ chế Evidence Gate trong `agent.py`: theo dõi toàn bộ URL incident đã xem và yêu cầu sửa câu trả lời nếu thiếu nguồn.
- Vì sao chọn thay đổi này:
  - Khắc phục lỗi dừng sớm khi gặp liên kết sự cố mở rộng, kích hoạt khả năng suy luận và điều tra nhiều bước (multi-step investigation).
  - Giúp loại bỏ hoàn toàn vi phạm cơ học và tăng tính chính xác, đầy đủ của câu trả lời.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 80.91 | 79.55 |
| Điểm case đã chọn (H6) | 60.0 | 80.0 (+20) |
| Số model calls | 52 | 53 |
| Token, chi phí nếu có | 83,150 tokens ($0.0179) | 94,089 tokens ($0.0195) |

- Lỗi đã giải quyết:
  - Case được chọn `H6` tăng từ 60 lên 80 điểm (+20 điểm), xóa bỏ hoàn toàn vi phạm cơ học `Cần tool calls ở ít nhất 2 model steps khác nhau` (số step gọi tool tăng từ 1 lên 2).
  - Có tới 7 cases tăng điểm rõ rệt nhờ cải tiến prompt và kiểm tra nguồn: E1 (+10), E2 (+15), A4 (+25), A8 (+20), H3 (+15), H6 (+20), H7 (+10).
- Lỗi còn lại hoặc mới xuất hiện:
  - Xuất hiện regression ở case `H2` (từ 100 giảm xuống 50): ở lần 2, model tập trung kiểm tra `get_incident_details` mà bỏ sót lệnh gọi `get_status` để kiểm tra trạng thái tổng quan hiện tại của Actions, dẫn đến bị dính trần điểm cơ học vi phạm `Chưa gọi tool cần thiết: get_status`.
  - Case `E3` bị Judge chấm khắt khe hơn về mốc thời gian snapshot dẫn đến giảm điểm từ 100 xuống 75.

## 5. Giới hạn

- **Giới hạn của điểm judge:** LLM Judge (dùng model gpt-4o-mini) là phép đánh giá ước lượng xấp xỉ, có độ dao động (variance) nhất định giữa các lần chạy đối với cùng một câu trả lời. Judge có thể chấm khắt khe hơn hoặc thiên lệch về cách dùng từ ngữ.
- **Giới hạn của snapshot:** Dữ liệu chỉ phản ánh trạng thái đóng băng tại thời điểm chụp (09/11/2026) và chỉ lưu tối đa 50 incident gần nhất; không thể dùng để khẳng định tình trạng hoạt động ngoài đời thực hay lịch sử đầy đủ của cả năm.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** Mô hình ngôn ngữ có tính bất định (non-deterministic sampling). Việc một phiên bản có điểm tổng dao động nhẹ (80.91 -> 79.55) phản ánh tính đánh đổi (trade-off) và hiện tượng regression thường gặp trong phát triển Agent: cải tiến để giải quyết một lỗi phức tạp có thể vô tình làm thay đổi hành vi ở một case khác. Cần nhiều lần chạy trên tập dữ liệu lớn hơn để khẳng định độ ổn định thống kê.
