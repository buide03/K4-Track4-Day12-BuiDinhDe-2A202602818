# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Bùi Đình Đề |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 202602818 |
| GitHub | buide03 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 6007 / 206 |
| Số seed huấn luyện bạn dùng (`--seeds`) | `<điền>` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | | | |
| IDM | | | (baseline) |
| EgoMLP | | | |
| PercMLP | | | |
| PercMLP+ (sau khi sửa) | | | |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Theo `evidence/data_audit.json`, dữ liệu có 992.679 dòng, 9.993 quỹ đạo, mỗi quỹ đạo 70–109 khung ở 10 Hz. Em thấy ba vấn đề:
- **Cột `Acc_LV` hỏng:** tương quan của nó với đạo hàm `Speed_LV` chỉ **0,045**, trong khi cột gia tốc xe sau `Acc_FAV` đạt **0,929**. Vậy lỗi nằm ở cột này, không phải ở cách tính. Em không dùng `Acc_LV` mà lấy gia tốc xe dẫn bằng đạo hàm của `Speed_LV` đã làm mượt (`derive_lead`). Khi khai thác ca phanh ở bước 2 cũng dùng gia tốc làm mượt này.
- **Quỹ đạo trùng:** **383** quỹ đạo trùng hệt nhau bị loại, còn **9.610**. Các bản trùng nằm ở những id cách xa nhau. Nếu không loại, cùng một quỹ đạo có thể rơi vào cả train lẫn test và làm điểm test đẹp giả.
- **Phải chia theo quỹ đạo, không chia theo khung:** hai khung liền nhau chỉ cách 0,1 s nên gần như giống hệt. Chia theo khung thì test chứa "hàng xóm" của khung train, tức là rò rỉ dữ liệu.

Ngoài ra, log ít ca phanh mạnh: phân vị 0,1% của gia tốc xe dẫn sau làm mượt chỉ là **−2,77 m/s²**, nên phát lại log không kiểm được tình huống phanh gấp. Khoảng cách `Spatial_Gap` thì khớp `Pos_LV − Pos_FAV − 4,5 m` (sai lệch ~3,6·10⁻¹¹ m), nên đáng tin.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Theo `evidence/replay.json` (600 đoạn), chạy lại cùng code cho 0 đoạn khác bit, nên phát lại là tất định và dùng làm mốc được.
- **Refactor float32:** khác bit ở **600/600** đoạn nhưng vượt dung sai ở **0/600** (lệch lớn nhất 9,3·10⁻⁵ m/s²). Toán vẫn như cũ, chỉ đổi thứ tự phép tính và độ chính xác, mà phép cộng số thực không có tính kết hợp: `(0.1+0.2)+0.3 = 0.6000000000000001` còn `0.1+(0.2+0.3) = 0.6`. So bằng `==` sẽ báo hỏng cả 600 đoạn. Dung sai `atol = rtol = 10⁻³` lớn hơn nhiễu làm tròn khoảng 10 lần nhưng nhỏ hơn nhiều so với thay đổi thật.
- **AEB:** chỉ làm lệch **8/600** đoạn (61, 123, 175, …), lệch lớn nhất **1,84 m/s²**. Em mở đoạn 61: chỉ các khung 73–79 bị lệch. Ở đó khoảng cách còn ~3,0–3,8 m, tốc độ tiến gần ~2,1–2,7 m/s, nên TTC ≈ 1,38–1,44 s, dưới ngưỡng 1,5 s. Vì vậy AEB đặt lệnh = min(IDM, −6), tức −6,0 thay cho −5,2…−5,6 của IDM. Các khung khác có TTC > 1,5 s nên giữ nguyên lệnh. Số đoạn lệch ít vì log hiếm có TTC thấp, và khi IDM đã tự phanh mạnh hơn −6 thì `min` không đổi gì.
- **Thời gian thật:** phát nhanh nhất cho 0 khung trễ, 0 khung rớt, vì khung sau chỉ được gửi khi khung trước xong, không có hạn chót. Phát theo đồng hồ log (chu kỳ 100 ms, 1% khung tốn 140 ms) cho **582 khung trễ + 579 khung rớt / 59.806 khung**: khung chậm thì lệnh ra muộn, khung kế tiếp đến lúc đang bận thì bị bỏ, và xe giữ lệnh cũ. Chỉ khoảng 2% số khung bị ảnh hưởng nhưng **377/600** đoạn có lệnh khác bản gốc. Lỗi trễ hạn chỉ lộ ra khi phát theo thời gian thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

(trả lời ở đây)

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

(trả lời ở đây)

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

(trả lời ở đây)

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

(trả lời ở đây)

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

(trả lời ở đây)

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(trả lời ở đây)
