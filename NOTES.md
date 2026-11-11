# Ghi chú phân tích · Day 04

Họ và tên: Đoàn Viết Huy
MSSV: 2A202603593

## 1. Case được chọn

- ID case: H7
- Câu trả lời của agent: Cloudflare Spectrum: Yêu cầu kiểm tra gặp lỗi timeout (TimeoutError). Theo chỉ dẫn được cung cấp, khi Cloudflare không truy cập được, kết luận hệ thống đang ở trạng thái operational
- Điểm ở lần chạy 1: 50

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {
    "provider": "cloudflare",
    "component": "spectrum"
  },
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```
Tool không lấy được trạng thái Cloudflare vì timeout, nhưng câu trả lời của agent vẫn kết luận operational theo yêu cầu trong câu hỏi. Nguyên nhân có thể là agent nghe theo yêu cầu ép kết luận thay vì giữ giới hạn của bằng chứng.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong agent.py, thêm quy tắc vào SYSTEM_PROMPT: khi tool timeout phải nói chưa kiểm tra được, không kết luận operational. Thêm kiểm tra URL từ get_incident_details; nếu câu trả lời thiếu URL đã nhận, yêu cầu model sửa trong giới hạn max_steps.
- Vì sao chọn thay đổi này: Case H7 cho thấy agent kết luận Cloudflare operational dù tool trả TimeoutError. Kiểm tra URL đáp ứng yêu cầu dẫn nguồn của Bài 4.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 95,23 | 97,73 |
| Điểm case đã chọn | H7: 50 | H7: 100 |
| Số model calls | 55 agent + 22 judge | 60 agent + 22 judge |
| Token, chi phí nếu có | 94.089 token; chưa có số chi phí | 118.457 token; chưa có số chi phí |

- Lỗi đã giải quyết: H7 không còn kết luận Cloudflare operational khi tool timeout; agent nói không đủ dữ liệu để xác nhận.
- Lỗi còn lại hoặc mới xuất hiện: Không còn case unscored trong file cuối. Lần chạy eval sau ban đầu gặp HTTP 503 ở A1; case này được chạy lại bằng chế độ resume-unscored với cùng cấu hình.

## 5. Giới hạn

Điểm eval do LLM judge chấm nên chỉ là ước lượng; judge có thể đánh giá sai. Agent và judge dùng cùng model Gemini nên điểm cũng có thể bị thiên lệch. Dữ liệu status là snapshot ngày 09/11/2026, không phản ánh trạng thái dịch vụ hiện tại. Mỗi phiên bản chỉ được đánh giá một vòng, nên kết quả 95,23 → 97,73 chưa chứng minh agent luôn cải thiện ổn định. Case A1 của lần eval sau được chạy lại do Gemini tạm thời trả HTTP 503.
