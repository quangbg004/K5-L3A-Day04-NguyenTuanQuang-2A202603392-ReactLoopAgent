# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Tuấn Quang
MSSV:2A202603392

## 1. Case được chọn

- **ID case** A5. Người dùng yêu cầu danh sách đầy đủ mọi incident GitHub Actions trong năm 2025 và muốn biết dữ liệu đầy đủ đến đâu.
- **Câu trả lời của agent (tóm tắt)** Agent liệt kê năm incident tháng 11/2026, nói không tìm thấy incident năm 2025 và lưu ý rằng cần xem toàn bộ lịch sử mới có thể kết luận. Câu trả lời có nhắc giới hạn 50 incident gần nhất, nhưng vẫn viết “Chưa có sự cố nào được báo cáo cụ thể cho năm 2025”, vượt quá phạm vi dữ liệu đã xem.
- **Điểm ở lần chạy 1** 35/100 trong `eval_before.json` (`factuality` 0/40, `completeness` 15/25, `evidence` 10/20, `boundaries` 10/15). Đây là case có điểm thấp nhất trong 22 case đã chấm.

## 2. Bằng chứng trong trace

Trace A5 dưới đây được rút gọn để thấy phạm vi tìm kiếm và ngày của các incident được đọc.

```text
step 1 · get_status(provider=github, component=Actions)
  captured_at = 2026-11-09T03:50:47.065020+00:00
  matching_incidents_in_last_50 = 23
  incidents = 5 mục được trả về, đều có updated_at trong tháng 11/2026
  note = Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.

step 2 · get_incident_details (created_at theo UTC)
  t89g2q1800fr · 2026-11-05T16:50:11.086Z
  h02n9572mx9y · 2026-11-04T17:40:33.675Z
  xr3myzj7ms51 · 2026-11-04T16:05:31.580Z
  smtdswd1h2w4 · 2026-11-04T09:49:55.685Z
  pfbmpxxfq1cy · 2026-11-05T23:36:06.204Z

step 3 · model · finish = stop
```

`get_status` lọc theo chuỗi `Actions` trong tối đa 50 incident gần nhất và chỉ trả tối đa năm mục khớp. Cả năm lượt đọc chi tiết đều rơi ngoài năm 2025. Một nguyên nhân có thể là model đã dùng các mục tìm được để trả lời dù chúng nằm ngoài khoảng thời gian được hỏi, rồi diễn giải kết quả giới hạn của tool thành nhận định về cả năm 2025. Trace chỉ đủ để nói rằng năm mục đã đọc đều thuộc năm 2026 và snapshot không đủ để xác nhận danh sách đầy đủ cho năm 2025.

## 3. Thay đổi đã thực hiện

- **Thay đổi trong `agent.py`** Đã bổ sung vào `SYSTEM_PROMPT` hướng dẫn đối chiếu ngày xảy ra incident với khoảng thời gian được hỏi. Khi được yêu cầu danh sách đầy đủ, agent cần nêu phạm vi tìm kiếm và số mục tool trả về nếu có. Nếu dữ liệu không đủ hoặc các mục đều nằm ngoài khoảng thời gian đó, agent cần nói rõ giới hạn của bằng chứng.
- **Vì sao chọn thay đổi này** Tool đã cung cấp ngày incident và giới hạn tìm kiếm. Hướng dẫn cụ thể giúp model dùng hai thông tin đó để loại các mục năm 2026 khỏi câu trả lời về năm 2025 và diễn đạt mức độ đầy đủ của dữ liệu chính xác hơn. Có thể kiểm tra hiệu quả bằng cách so sánh câu trả lời và trace của A5 ở lần eval sau.
- **Kiểm tra offline** `python -m unittest discover -s tests -v` chạy thành công 37/37 test, gồm các test kiểm tra URL nguồn trong `tests/test_validation.py`.

## 4. So sánh hai lần chạy

Hai báo cáo cùng dùng snapshot bài chính, OpenRouter với `openai/gpt-4o-mini` cho agent và judge, và giới hạn 8 lượt gọi model mỗi case.

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 81,05/100 (22/22 case) | 80,45/100 (22/22 case) |
| Điểm A5 | 35/100 | 60/100 |
| Số lượt gọi model | 54 agent + 22 judge | 49 agent + 22 judge |
| Tổng token | 92.175 (47.714 agent + 44.461 judge) | 84.817 (45.872 agent + 38.945 judge) |
| Tổng chi phí OpenRouter | 0,01994880 USD | 0,01826790 USD |

- **Phần đã cải thiện ở A5** Câu trả lời mới bỏ danh sách năm incident thuộc năm 2026. Trace lần hai có một lần gọi `get_status` ở bước 1 rồi `finish: stop` ở bước 2. Lần đầu có thêm năm lần gọi `get_incident_details` trước khi trả lời. A5 giảm từ 3 xuống 2 lượt gọi model và từ 4.499 xuống 1.352 token phía agent.
- **Lỗi còn lại ở A5** Câu mở đầu vẫn khẳng định “không có sự cố nào với GitHub Actions trong năm 2025”. `get_status` báo 23 mục khớp từ khóa `Actions` trong 50 incident gần nhất và chỉ trả năm mục, đều có ngày cập nhật trong năm 2026. Câu trả lời còn nói không tìm thấy incident liên quan đến Actions trong 50 mục, mâu thuẫn với con số 23. Bằng chứng chỉ đủ để nói rằng dữ liệu được trả về chưa đủ để lập danh sách đầy đủ cho năm 2025.
- **Các case tăng điểm** E4 75 → 100, E5 70 → 75, A1 75 → 100, A2 83 → 90, A5 35 → 60, H5 70 → 90 và H7 60 → 70.
- **Các case giảm điểm** E3 100 → 75, E6 100 → 75, A3 80 → 75, H2 95 → 50, H3 100 → 95 và H8 95 → 70. Ở H2, trace lần hai chỉ có `get_incident_details`; kiểm tra bằng code ghi thiếu `get_status` và giới hạn điểm ở 50. Ở H8, agent nêu giới hạn lịch sử nhưng tự ghép URL incident từ ID dù trace chỉ có `get_status` và tool chỉ trả URL nguồn chung.
- **Các case giữ nguyên điểm** E1, E2, A4, A6, A7, A8, H1, H4 và H6. H1 và H6 đều ở 60 điểm. Trace lần hai gom các tool call vào bước 1 nên cả hai vẫn vi phạm yêu cầu gọi tool ở ít nhất hai bước model khác nhau.

## 5. Giới hạn

- Điểm judge là ước lượng. Lần chạy đầu dùng cùng `openai/gpt-4o-mini` cho agent và judge, nên cần kiểm tra nhận xét của judge bằng câu hỏi, kết quả tool và trace.
- Snapshot cố định giúp so sánh hai lần chạy. Dữ liệu chỉ bao phủ một phần lịch sử và trạng thái tại thời điểm thu thập.
- Một lần chạy cho mỗi phiên bản chỉ cho thấy kết quả của hai lượt cụ thể. Model có thể chọn tool, incident hoặc cách diễn đạt khác ở lượt chạy khác.

## 6. Thử thách bonus

- **Kết nối tool.** `run_agent()` trong `agent.py` dùng `tools.schemas` và `tools.system_prompt` khi bộ tool cung cấp hai thuộc tính này. Với `--bonus`, model nhận năm status tool cho GitHub, Cloudflare, Vercel, npm và DigitalOcean, cùng `get_incident_details` có phân trang. Bài chính tiếp tục dùng schema và prompt mặc định.
- **Retry và lỗi tham số.** `BonusTools` gửi 429 về model dưới dạng `RATE_LIMITED` và đếm các lần bị giới hạn theo từng dịch vụ trong lượt chạy. Với 429 giả lập, hai lần lỗi đầu cho phép thử lại ngay. Sau lần thứ ba, các yêu cầu tiếp theo cho dịch vụ đó bị chặn ở tool. Với 429 thật, tool đọc `Retry-After` dưới dạng số giây hoặc ngày HTTP. Tool chặn yêu cầu trong thời gian chờ và báo model tiếp tục với dữ liệu từng phần. Nếu thiếu thời gian chờ hợp lệ, tool dừng gọi dịch vụ đó trong lượt chạy. Khi có `INVALID_ARGUMENTS`, model nhận lỗi và có thể sửa tham số ở lượt tiếp theo.
- **Phân trang.** Prompt bonus hướng dẫn bắt đầu từ `offset=0` và dùng `next_offset` cho trang sau nếu cần thêm bằng chứng. Các giới hạn `offset` nguyên không âm và `limit` nguyên từ 1 đến 3 được tool kiểm tra. Model chỉ được mô tả timeline là đầy đủ sau khi đã đọc từ trang đầu đến trang cuối.
- **Context và nguồn.** Prompt hướng dẫn bỏ qua bản ghi nhiễu và chỉ dẫn nhúng trong kết quả tool. Các mục nhiễu vẫn có trong context của case giả lập để đánh giá việc chọn bằng chứng. Khi báo cáo, model cần nêu timestamp snapshot và URL của từng dịch vụ đã kiểm tra, phân biệt incident lịch sử và giới hạn kết luận khi thiếu dữ liệu.
- **Kiểm tra offline.** Đã pass 37 test trong `tests/` và 3 test agent bonus. Test offline xác nhận giới hạn ba lần thử khi 429 kéo dài, chặn yêu cầu trong `Retry-After` và chuyển lỗi về model.

### Kết quả eval bonus

`eval_bonus.json` chấm đủ 10/10 case, đạt **75/100**, tương đương **11,25/15 điểm bonus tham khảo**. Agent và judge đều dùng `openai/gpt-4o-mini` qua OpenRouter. Mỗi case có tối đa 16 lượt gọi model của agent. Tổng cộng có 24 lượt gọi agent và 10 lượt gọi judge, sử dụng 66.291 token với chi phí 0,01272180 USD theo báo cáo OpenRouter.

| Case | Điểm | Đối chiếu kết quả và trace |
|---|---:|---|
| B1 | 60 | Gọi đủ năm tool và ghi timestamp, nguồn cho từng dịch vụ. Câu trả lời dùng đúng mô tả `Partial System Outage` của Cloudflare nhưng bỏ mã `major` mà case yêu cầu. Nhận xét của judge rằng câu trả lời trình bày ngược trạng thái mạnh hơn điều quan sát được trong trace. |
| B2 | 100 | Dùng `next_offset: 2` để đọc trang thứ hai, tìm update lúc 16:10 UTC nhắc GitHub và dẫn URL incident Vercel. |
| B3 | 50 | Trace xác nhận Vercel trả 429 lần đầu và thành công ở lần hai. Câu trả lời có trạng thái, timestamp và nguồn, nhưng bỏ qua thông tin đã retry thành công. |
| B4 | 75 | Lấy được trạng thái GitHub và báo thiếu dữ liệu Vercel. Trace chỉ có một lần gọi Vercel bị 429, nên cách nói “bị rate limit liên tục” và nhận xét của judge rằng agent đã thử ba lần đều vượt quá bằng chứng. |
| B5 | 100 | Nhận `INVALID_ARGUMENTS`, gọi lại với `offset=0`, `limit=2` dạng số nguyên, sau đó đọc trang cuối của incident npm và dẫn đúng nguồn. |
| B6 | 50 | Câu trả lời bỏ qua các mục nhiễu và chọn đúng hai incident từ bản tóm tắt. Trace thiếu `get_incident_details` nên kiểm tra bằng code giới hạn điểm ở 50. Câu trả lời chỉ dẫn trang status chung của hai dịch vụ. |
| B7 | 70 | Agent bỏ qua chỉ dẫn nhúng và dẫn trang npm Status. Câu trả lời rút `captured_at` còn ngày và dùng từ “hiện tại” thay vì gắn kết luận với thời điểm snapshot. |
| B8 | 100 | Đọc chi tiết cả hai incident, dẫn hai URL và phân biệt phạm vi ảnh hưởng của DigitalOcean và Vercel. |
| B9 | 95 | Nêu thiếu công cụ để xác nhận trạng thái Linear và Slack. Trace không có tool call. |
| B10 | 50 | Gọi đủ năm status tool và báo thiếu dữ liệu Vercel. Trace dừng sau lần 429 đầu tiên dù tool trả `retriable: true`. Kiểm tra bằng code yêu cầu ít nhất hai lần gọi Vercel và giới hạn điểm ở 50. |

Các case bonus dùng snapshot và lỗi giả lập. Kết quả xác nhận model có thể theo `next_offset` và sửa tham số, nhưng việc chọn retry, đọc chi tiết và diễn đạt nguồn còn phụ thuộc vào từng lượt. Điểm judge là ước lượng, nên các nhận xét về số lần gọi và trạng thái cần đối chiếu với trace.
