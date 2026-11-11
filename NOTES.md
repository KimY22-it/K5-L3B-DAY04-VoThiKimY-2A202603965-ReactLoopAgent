# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

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
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
