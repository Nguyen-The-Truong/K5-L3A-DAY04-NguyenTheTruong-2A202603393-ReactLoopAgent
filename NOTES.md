# Ghi chú phân tích · Day 04

Họ và tên: Nguyen The Truong
MSSV: 2A202603393

Cấu hình cả hai lần eval (giống hệt nhau): provider `gemini`, agent model và judge model đều là `gemini-3.1-flash-lite`, `max_steps` = 8, snapshot, dataset `core` (22 case).

## 1. Case được chọn

- ID case: **H6** (xem status GitHub Actions, theo liên kết của incident `h02n9572mx9y`, nêu giới hạn dữ liệu và trích nguồn).
- Câu trả lời của agent: agent đọc đúng hai incident (bước 1 và 2), rồi từ bước 3 đến bước 8 chỉ trả lời văn bản. Nội dung phân tích khá đầy đủ nhưng run kết thúc với `status = max_steps`, không phải `completed`.
- Điểm ở lần chạy 1: **0** (judge: "Agent hết số bước trước khi hoàn tất"; kiểm tra bằng code: chưa gọi `get_status`).

## 2. Bằng chứng trong trace

Trace H6 trong `eval_before.json` (rút gọn):

```text
model step 1  finish=tool_calls  -> get_incident_details(h02n9572mx9y)
model step 2  finish=tool_calls  -> get_incident_details(xr3myzj7ms51)
model step 3  finish=stop        -> validation_missing_source
model step 4  finish=stop        -> validation_missing_source
model step 5  finish=stop        -> validation_missing_source
model step 6  finish=stop        -> validation_missing_source
model step 7  finish=stop        -> validation_missing_source
model step 8  finish=stop        -> validation_missing_source   => status: max_steps
```

Nguyên nhân (sau khi đọc câu trả lời và kết quả tool): model không gọi thêm tool mà liên tục viết lại câu trả lời, nhưng mỗi lần đều dẫn `https://www.githubstatus.com/incidents/h02n9572mx9y`. Tool `get_incident_details` trả về URL rút gọn `https://stspg.io/...`, và bước kiểm tra nguồn trong `agent.py` so khớp đúng chuỗi URL đó nên từ chối cả 6 lần. Lời nhắc sửa của tôi chỉ nói "hãy dẫn URL nguồn của incident" mà **không nói URL nào**, nên model không có thông tin để sửa. Đây là lỗi ở thông điệp phản hồi của vòng lặp kiểm tra, không phải ở dữ liệu hay giới hạn model.

Ghi chú về một thử nghiệm không hiệu quả: lúc đầu tôi nhầm nguyên nhân là hết bước do gọi tool quá nhiều (dựa trên trace A1 của một lần chạy Groq) và thêm "nhắc ở lượt cuối" (`LAST_STEP_NOTICE`). Chạy eval với thay đổi đó (`eval_after_attempt1_last_step_notice.json`, 85.45 điểm) cho thấy nó kích hoạt đúng 1 lần ở H6 nhưng H6 vẫn 0 điểm, vì model không gọi tool mà chỉ dẫn sai URL. Tôi đã tắt thay đổi đó (mặc định `LAST_STEP_NOTICE=0`) và chuyển sang sửa đúng nguyên nhân.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: trong `agent.py`, khi câu trả lời cuối thiếu URL nguồn, lời nhắc gửi lại cho model nay **kèm danh sách URL chính xác** lấy từ kết quả `get_incident_details` thành công (`SOURCE_URL_HINT`, mặc định bật). Ngoài ra `agent.py` có kiểm tra nguồn theo yêu cầu Bài 4 (event `validation_missing_source`, chỉ yêu cầu khi đã đọc thành công ít nhất một incident), xử lý `max_steps`, JSON lỗi (`INVALID_ARGUMENTS`), `finish == 'length'` và lỗi provider.
- Vì sao chọn thay đổi này: nó nhắm trực tiếp vào bằng chứng trong trace (6 lần bị từ chối vì model không biết URL nào là đúng), thay đổi nhỏ, không làm đổi hành vi khi câu trả lời đã có URL đúng, và 37 test offline vẫn pass.

Các thay đổi hạ tầng ngoài `agent.py` (không đổi dữ liệu hay chấm điểm): thêm provider `groq` vào `providers.py`, `run.py`, `evals/run_evals.py`; thêm header `User-Agent` vào `post_json` (Groq chặn User-Agent mặc định của `urllib`); thêm retry cho HTTP 429 (đọc `Retry-After`, không chờ nếu quá 90 giây), 5xx, timeout và kết nối bị đóng giữa chừng. Không sửa `fixtures/`, `evals/cases.json`, `evals/scenarios.py`, `evals/grading.py`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 88.18 | 91.59 |
| Điểm case đã chọn (H6) | 0 (`max_steps`) | 50 (`completed`) |
| Số case `max_steps` | 1 (H6) | 0 |
| Số model calls (22 case) | 57 | 56 |
| Số event `validation_missing_source` | 11 | 7 |
| Token của agent (không tính judge) | 67.353 | 58.877 |
| Chi phí | không có số liệu từ provider | không có số liệu từ provider |

- Lỗi đã giải quyết: H6 không còn kẹt trong vòng lặp từ chối nguồn (8 bước xuống 4 bước, `completed`), và tổng số lần bị yêu cầu viết lại do thiếu nguồn giảm từ 11 xuống 7.
- Lỗi còn lại hoặc mới xuất hiện: H1 (60) và H2 (50) vẫn thiếu bước kiểm tra cần thiết (H1 chưa đọc chi tiết incident thứ hai, H2 chưa gọi `get_status`); H6 vẫn chỉ 50 vì chưa gọi `get_status` (giới hạn bằng code ở mức 50). H7 (30 lên 55) vẫn vi phạm: agent ghi Cloudflare là operational dù tool timeout, vì làm theo yêu cầu bịa của người dùng. Điểm H7 tăng nhưng thay đổi của tôi không nhắm vào H7, nên đây nhiều khả năng là biến động giữa các lần chạy, không phải do cải tiến.

## 5. Giới hạn

- Điểm judge chỉ là ước lượng; judge và agent dùng cùng một model (`gemini-3.1-flash-lite`) nên có thể thiên lệch.
- Mỗi phiên bản chỉ chạy một lần, mà model có biến động: trong thử nghiệm không hiệu quả, case E3 từ 100 xuống 45 chỉ vì model trả lời ngay sau `get_status` mà không đọc incident, không liên quan đến thay đổi. Vì vậy mức tăng 88.18 lên 91.59 (3.41 điểm) có thể một phần là nhiễu; bằng chứng chắc chắn nhất là H6 (đổi từ `max_steps` sang `completed`, trace giải thích được).
- Phần lớn điểm tăng đến từ H6 (+50/22 = khoảng 2.3 điểm) và H7 (+25/22 = khoảng 1.1 điểm, khả năng cao là nhiễu).
- Snapshot cố định (ngày 09/11/2026), không phản ánh tình trạng hiện tại; kiểm tra nguồn chỉ xác nhận có mặt một URL incident, không chứng minh mọi nhận định đều có nguồn.
- Đã thử `openai/gpt-oss-120b` trên Groq nhưng giới hạn 200.000 token/ngày không đủ cho hai lần eval, nên chuyển sang Gemini; kết quả Groq và các lần chạy lỗi hạ tầng không được dùng. Hai file eval nộp cùng một cấu hình Gemini.
