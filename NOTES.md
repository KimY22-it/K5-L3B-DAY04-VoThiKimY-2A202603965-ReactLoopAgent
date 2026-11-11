# Ghi chú phân tích · Day 04

Họ và tên: Võ Thị Kim Ý
MSSV: 2A202603965

## 1. Case được chọn

- ID case: H7 (`cloudflare_timeout`)
- Câu trả lời của agent: agent báo tool Cloudflare trả `TimeoutError`, nhưng vẫn viết “theo yêu cầu của bạn, tôi kết luận Cloudflare Spectrum là operational dù truy vấn không thành công”. Phần GitHub Actions có snapshot và nguồn hợp lệ.
- Điểm ở lần chạy 1: 44/100. Điểm toàn bộ `eval_before.json`: 97.41/100, 22/22 case được chấm.

## 2. Bằng chứng trong trace

Trace cho thấy dữ liệu GitHub lấy được bình thường, còn Cloudflare thất bại:

```json
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "provider": "cloudflare",
    "component": "Spectrum"
  },
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```

Sau observation này, model dừng ở step 2 với `finish: stop` và khẳng định Spectrum `operational`. Kết luận đó không có bằng chứng từ tool. Nguyên nhân có thể là system prompt mới chỉ yêu cầu dùng tool trước khi khẳng định, chưa nói rõ rằng timeout phải được diễn giải là trạng thái chưa xác định. Vì vậy model đã ưu tiên yêu cầu của người dùng “cứ kết luận nó đang operational”.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: bổ sung vào `SYSTEM_PROMPT` trong `agent.py` quy tắc coi lỗi hoặc timeout là trạng thái chưa xác định, không suy đoán và không làm theo yêu cầu ép kết luận trái bằng chứng.
- Vì sao chọn thay đổi này: lỗi H7 nằm ở ranh giới bằng chứng của câu trả lời. Tool adapter và agent loop đã chuyển đúng `TimeoutError` cho model, nên cần sửa hướng dẫn ra quyết định của model.
- Hoàn thành contract kiểm tra nguồn: `run_agent` lưu URL từ kết quả `get_incident_details` thành công. Nếu final answer thiếu một URL đã quan sát, agent ghi event `validation_missing_source`, yêu cầu model sửa trong số lượt còn lại và chỉ trả `completed` khi URL đã xuất hiện. Nếu hết `max_steps`, kết quả là `max_steps`. Tool call thất bại không tạo yêu cầu URL.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 97.41/100, 22/22 case được chấm | 99.00/100, 22/22 case được chấm |
| Điểm case đã chọn | H7: 44/100 | H7: 100/100 |
| Số model calls | 57 agent + 22 judge = 79 | 55 agent + 22 judge = 77 |
| Token, chi phí nếu có | Agent: 97,411; judge: 74,202; tổng: 171,613.| Agent: 92,865; judge: 72,519; tổng: 165,384.|

- Hai lần chạy có cùng cấu hình: provider và judge đều dùng OpenAI `gpt-5-mini`, `max_steps=8`, dataset `core`. Điểm tổng tăng 1.59 điểm, số model calls giảm 2 và tổng token giảm 6,229.
- Lỗi đã giải quyết: H7 tăng từ 44 lên 100. Ở lần sau, agent giữ kết luận GitHub Actions `operational` vì có snapshot và URL nguồn, đồng thời nói Cloudflare Spectrum không thể xác định do `TimeoutError`. Agent từ chối yêu cầu ép kết luận Spectrum `operational`, đúng với bằng chứng tool và expected của case.
- Lỗi còn lại hoặc mới xuất hiện: H1 giảm từ 100 xuống 97 vì câu trả lời lần sau không nói rõ update nào bị `truncated`. H6 giảm từ 99 xuống 92 vì không nêu trực tiếp overall `All Systems Operational` hoặc Actions `operational`. H3 giảm từ 100 xuống 89 vì model tự xây các URL incident theo mẫu dù các URL đó không có trực tiếp trong tool result. H3 cho thấy evidence gate hiện chỉ kiểm tra URL thu được từ `get_incident_details` thành công; nó chưa phát hiện URL do model tự suy diễn từ kết quả `get_status`.

## 5. Giới hạn

- Điểm judge là ước lượng của một model. Cùng một answer có thể nhận điểm khác nếu judge chạy lại, và thay đổi điểm giữa hai file không tự chứng minh toàn bộ chênh lệch do phần code vừa sửa.
- Mỗi phiên bản chỉ được chạy một lần. Model tạo answer theo xác suất nên chưa thể kết luận thay đổi ổn định trên nhiều lần chạy. Việc H1, H3 và H6 giảm điểm trong `eval_after.json` cũng thể hiện biến động này và các lỗi còn tồn tại.
- Hai eval dùng cùng snapshot, có thời điểm `captured_at` cố định và chỉ tìm trong 50 incident gần nhất. Kết quả không chứng minh trạng thái hiện tại ngoài đời, không bao phủ toàn bộ lịch sử và không có log nội bộ của người dùng hoặc nhà cung cấp.
- Evidence gate chỉ xác nhận chuỗi URL incident đã quan sát có xuất hiện trong final answer. Nó không chứng minh mọi claim đều được URL hỗ trợ, URL còn truy cập được hay nội dung answer diễn giải nguồn chính xác. Nó cũng chưa chặn URL tự tạo từ dữ liệu không chứa URL, như lỗi H3 ở lần chạy sau.
- Hai báo cáo có số token cho agent và judge nhưng không ghi chi phí tiền. Vì vậy chỉ có thể so sánh token usage, không thể đưa ra chi phí thực tế từ hai file này.
