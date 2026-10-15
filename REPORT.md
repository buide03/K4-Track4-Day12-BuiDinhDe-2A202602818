# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Bùi Đình Đề |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 202602818 |
| GitHub | buide03 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 6007 / 206 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (seed mô hình 206–210) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,249 | 19,5 | CHẶN (R2, R3, R4, R6) |
| IDM | 1,535 | 0,1 | (baseline) |
| EgoMLP | 0,757 ± 0,005 | 12,7–24,6 | CHẶN (R2, R3, R4, R5, R6) |
| PercMLP | 0,619 ± 0,003 | 0,4–0,9 | CHẶN (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0,823 ± 0,015 | 0,2–0,2 | PHÁT HÀNH (qua 7/7) |

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

Theo `evidence/scenarios.json`, lưới của em có tốc độ cruise **10 m/s**, khoảng cách lúc cắt vào **2,76–59,31 m** (p2–p98 của log), tốc độ tiến gần 0–10 m/s, tổng **60 × 45 = 2.700 ô**.
- **Bất khả kháng:** biên tính bằng `gap < dv·t_react + dv²/(2·9)`, tức quãng mất trong thời gian phản ứng cộng quãng phanh hết cỡ. Biên này chiếm **1,0%** số ô khi trễ 0 s và **5,1%** khi trễ 0,6 s. Kiểm chứng: bộ "phanh hết cỡ" đâm đúng 1,0% / 5,1% ô, với **0 ô lệch** khỏi biên ở cả hai độ trễ. Vậy bộ mô phỏng và hàm `unavoidable` khớp nhau.
- **Trễ 0 → 0,6 s:** vùng đâm của IDM tăng từ **1,0% lên 5,1%** (khoảng 5 lần). Nhưng vùng do lỗi của IDM (`attributable_area`) vẫn là **0** ở cả hai mức: IDM chỉ đâm ở những ô không ai tránh được (đường đỏ trùng đường cyan trong `scenarios_idm.png`). Nếu không có biên này, ta sẽ nhầm rằng IDM kém đi. Ngược lại, constant-velocity đâm **73,4%** ô, trong đó 72,4% (trễ 0 s) và 68,2% (trễ 0,6 s) là lỗi của chính nó.
- **Độ phủ của log:** đào được **614** sự kiện xe dẫn phanh (6,4% quỹ đạo), với đỉnh phanh trung vị −2,53 m/s² và p99 −4,21 m/s². Người lái trong log phanh mạnh nhất khoảng **3,62 m/s²** (p99,9), trong khi xe phanh được 9 m/s², và **5,2%** số ô của lưới đòi phanh mạnh hơn mức đó. Các chấm trắng (sự kiện khai thác) dồn ở vùng tốc độ tiến gần thấp, không chạm vùng sát biên bất khả kháng. Log **không phủ** vùng khó, nên model học từ log chưa từng thấy những tình huống này. Muốn kiểm thử vùng đó phải dùng kịch bản tổng hợp.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Số liệu từ `evidence/openloop.json` và `evidence/closedloop.json` (2.883 quỹ đạo giữ lại, 5 seed; model học lấy trung bình qua seed):

| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín | Hạng |
|---|---|---|---|---|
| constant-velocity | 1,249 | 3 | 19,5% | 4 |
| IDM | 1,535 | 4 | 0,1% | 1 |
| EgoMLP | 0,757 | 2 | 18,6% (dải 12,7–24,6%) | 3 |
| PercMLP | 0,619 | 1 | 0,7% (dải 0,4–0,9%) | 2 |

Có ba cặp đổi thứ tự (`order_flips`): (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM xếp **cuối** theo L2 nhưng **đầu** theo va chạm.

- **IDM:** L2 đo mức "giống người" chứ không đo mức "an toàn". IDM không bắt chước người lái: nó giữ khoảng thời gian 1,2 s và phanh theo khoảng cách cùng tốc độ tiến gần, nên lệch khỏi quỹ đạo người (L2 1,535 m). Nhưng trong vòng kín, nó luôn phản ứng với chính trạng thái nó tạo ra, nên chỉ đâm 0,1% (0,5% trên 197 quỹ đạo xe dẫn phanh) và có vùng lỗi 0% trên lưới ở cả hai độ trễ.
- **EgoMLP:** chỉ nhìn tốc độ của chính nó, **không biết khoảng cách hay tốc độ xe trước**. Vì người lái phần lớn thời gian giữ tốc độ đều, đoán theo quán tính đã cho L2 thấp (0,757 m). Trên lát cắt xe dẫn phanh, đầu vào của nó không có tín hiệu nào báo xe trước đang phanh, nên nó đâm 37,6–58,4% số quỹ đạo này (tùy seed), so với 54,8% của constant-velocity. Thêm vào đó, trong vòng hở mỗi cửa sổ bắt đầu lại từ trạng thái thật của người, còn trong vòng kín sai số tích lũy và đưa xe tới những trạng thái nằm ngoài phân bố huấn luyện.
- PercMLP có thông tin xe trước nên an toàn trên log (0,7%), nhưng trên lưới xe cắt vào vẫn đâm ở **8,5%** (trễ 0 s) và **12,3%** (trễ 0,6 s) số ô tránh được, trong khi IDM là 0%.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Theo `evidence/closedloop.json`, cùng dữ liệu và cùng code, chỉ khác seed khởi tạo, EgoMLP đâm từ **12,7% đến 24,6%** trên 2.883 quỹ đạo:

| Seed | 206 | 207 | 208 | 209 | 210 |
|---|---|---|---|---|---|
| Va chạm | 24,6% [23,1; 26,2] | 21,1% [19,6; 22,6] | 21,9% [20,4; 23,4] | 12,7% [11,5; 13,9] | 12,7% [11,5; 14,0] |

- **Khoảng tin cậy Wilson 95%** chỉ rộng khoảng ±1,5 điểm phần trăm cho mỗi seed. Khoảng của seed xấu nhất [23,1; 26,2] và seed tốt nhất [11,5; 13,9] **không chồng nhau**. Vậy chênh lệch gấp đôi không do may rủi khi lấy mẫu quỹ đạo, mà do chính quá trình huấn luyện. Wilson chỉ đo độ bất định do mẫu, không đo độ bất định do seed, nên phải báo cả CI lẫn dải qua seed. Trên lát cắt xe dẫn phanh, dải còn rộng hơn: 37,6–58,4%.
- **Nếu chỉ chạy một seed:** với seed 209 em sẽ báo "EgoMLP đâm 12,7%, tốt hơn hẳn constant-velocity (19,5%)". Với seed 206 em sẽ báo "EgoMLP đâm 24,6%, tệ hơn constant-velocity". Hai báo cáo dẫn tới hai kết luận trái ngược, và cả hai đều có CI hẹp trông rất chắc chắn. Nguy hiểm ở chỗ một con số đơn lẻ kèm CI hẹp tạo cảm giác tin cậy giả. Nếu seed đẹp được chọn để phát hành, model thực tế có thể đâm gần gấp đôi. PercMLP ổn định hơn (0,4–0,9%), nhưng dải đó vẫn rộng gấp đôi, nên quyết định phát hành phải dựa trên seed xấu nhất (0,9% [0,6; 1,3]), không dựa trên trung bình hay seed tốt nhất.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Theo `evidence/agents.json`, người lái trong log giữ khoảng thời gian trung vị **1,85 s**. Em thử IDM với T từ 0,6 đến 2,4 s và chọn **T = 1,4 s**, vì cho khoảng cách Wasserstein tới phân bố khoảng thời gian của người nhỏ nhất (**W1 = 0,34**). Một xe không bao giờ phản ứng cho **W1 = 0,84**, kém gần 2,5 lần, nên xe sau phản ứng đã hiệu chỉnh chân thực hơn hẳn xe chỉ phát lại. Xe sau đặt cách xe ta 16 m, phản ứng trễ 0,6 s. Tỷ lệ ô của lưới cắt vào có va chạm bất kỳ (phía sau trong ngoặc):

| Bộ điều khiển | Trễ 0 s: phát lại | Trễ 0 s: phản ứng | Trễ 0,6 s: phát lại | Trễ 0,6 s: phản ứng |
|---|---|---|---|---|
| const-vel | 73,4% (sau 0,0%) | 73,4% | 73,4% (sau 0,0%) | 73,4% |
| IDM | 69,6% (sau 68,6%) | 1,0% | 70,1% (sau 64,9%) | 5,1% |
| EgoMLP | 69,7% (sau 0,0%) | 69,7% | 70,6% (sau 0,0%) | 70,6% |
| PercMLP | 83,9% (sau 75,9%) | 9,4% | 82,5% (sau 65,5%) | 17,5% |

- **Va chạm giả:** khi có xe cắt vào, IDM phanh đúng. Nhưng xe sau phát lại cứ giữ tốc độ trong log, không biết xe ta đã phanh, nên tông vào đuôi. Kết quả là 68,6% ô bị tính va chạm phía sau. Người lái thật sẽ phanh theo: với xe sau phản ứng, va chạm phía sau về **0,0%**, và IDM chỉ còn 1,0% / 5,1%, đúng bằng vùng bất khả kháng ở câu 3.
- **Kết luận bị đổi:** trong thế giới phát lại, IDM (69,6%) trông ngang EgoMLP (69,7%) và chỉ hơn const-vel một chút. Hai bộ này không bao giờ phanh nên không bao giờ bị tông đuôi. Thế giới phát lại **phạt việc phanh đúng và thưởng việc không phản ứng**. Khi các tác nhân biết phản ứng, IDM tốt hơn rõ rệt, còn PercMLP (9,4% / 17,5%) vẫn kém IDM ở phía trước. Vì vậy cổng phát hành phải dùng số với xe sau phản ứng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Theo `evidence/gate.json`, PercMLP bị **chặn** bởi 4 quy tắc (baseline IDM, seed xấu nhất), với hai nguyên nhân gốc khác nhau:
- **Kịch bản mà log không chứa (R4, R6):** trên lưới cắt vào, vùng đâm tránh được là **12,3%**, so với 0,0% của IDM (ngưỡng +2 điểm). Với xe sau phản ứng là **17,5%**, so với 5,1%. Như câu 3: người lái trong log phanh mạnh nhất chỉ 3,62 m/s², và 614 ca phanh khai thác được không chạm vùng cắt vào gần, nhanh. Mạng chưa bao giờ thấy những trạng thái này.
- **Sai số tích luỹ khi tự lái (R2, R3):** ngay trên log thật, PercMLP đâm 0,9% với cận trên **1,3%**, vượt ngưỡng 0,1 + 1 = 1,1%. Trên lát cắt xe dẫn phanh là 2,5% với cận trên **5,8%**, vượt ngưỡng 5,5%. Mạng chỉ học ở trạng thái người đã đi qua; khi tự lái, lệch nhỏ cộng dồn và đưa xe tới trạng thái nằm ngoài phân bố huấn luyện.

**Cách sửa** (`fix.json`): (1) thêm cửa sổ từ **20.000** kịch bản cắt vào mô phỏng, có IDM làm mẫu, để chữa nguyên nhân thiếu kịch bản; (2) **DAgger 2 vòng**: model tự lái trên log train, ghi trạng thái nó đi vào, gắn nhãn bằng dự báo IDM ở đúng các trạng thái đó, để chữa sai số tích luỹ. Sau khi sửa, PercMLP+ qua **7/7** quy tắc: R2 0,2% (cận trên 0,4%), R3 0,5% (cận trên 2,8%), R4 **0,1%**, R5 0,2–0,2%, R6 **5,2%**. Mỗi cách sửa làm qua đúng các quy tắc ứng với nguyên nhân của nó.

**PercMLP+ không tốt hơn IDM**, chỉ **không kém hơn trong biên cho phép**: IDM vẫn đâm 0,1% so với 0,2% trên log, 0,0% so với 0,1% trên lưới, 5,1% so với 5,2% với xe sau phản ứng. Nó học nhãn từ chính IDM nên trên những tình huống này không thể vượt thầy. Ở **20 m/s** (ngoài vùng huấn luyện, cổng không kiểm tra), PercMLP+ còn 0,9–1,6% vùng đâm tránh được, trong khi IDM là 0,0%. L2 vòng hở lại **tệ hơn**: 0,619 → 0,823 m, thêm một bằng chứng rằng L2 không đo an toàn.

**Đề xuất:** chưa thay IDM. Qua cổng chỉ là điều kiện cần. PercMLP+ chưa mang lại lợi ích an toàn nào so với IDM, và còn kém ở tốc độ chưa thử. Trước khi quyết định, em sẽ: chạy shadow mode bên cạnh IDM để đo bất đồng trên dữ liệu thật; thêm vào cổng quy tắc kiểm tra ngoài vùng huấn luyện (ví dụ lưới 20 m/s); và chỉ cân nhắc phát hành nếu có lý do khác ngoài an toàn (ví dụ độ êm, thoải mái) được đo và chứng minh.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Thế giới mô phỏng là giả định của chính bài lab.** Bộ mô phỏng chỉ có một chiều dọc, xe cắt vào xuất hiện tức thời rồi giữ tốc độ, không có chuyển làn hay người đi bộ. Phép kiểm chứng "0 ô lệch" ở bước 2 chỉ chứng tỏ mô phỏng **khớp với công thức vật lý của chính nó**, không chứng tỏ nó khớp với đường thật. Xe sau phản ứng là IDM với T = 1,4 s (W1 = 0,34, so với 0,84 của xe không phản ứng): gần người hơn nhưng vẫn chỉ là một mô hình. Cổng cho PercMLP+ qua R6 với 5,2% so với 5,1% của IDM, mà con số này phụ thuộc trực tiếp vào giả định đó.
2. **Không có cảm biến thật.** Mọi bộ điều khiển nhận khoảng cách và tốc độ xe trước gần như hoàn hảo; phép thử nhiễu chỉ là ±1 m và ±0,5 m/s ngẫu nhiên. Thực tế có mất dấu, nhận nhầm vật thể và trễ không đều. Ngay bước 1, phát theo thời gian thật đã cho 582 khung trễ và 579 khung rớt trên 59.806 khung, làm đổi lệnh ở 377/600 đoạn, nhưng phần vòng kín không mô phỏng hiệu ứng này.
3. **Dữ liệu và vùng kiểm thử hẹp.** Log Argoverse 2 là giao thông đô thị chậm: người lái phanh mạnh nhất khoảng 3,62 m/s², và lưới chỉ chạy ở 10 m/s. Ở 20 m/s, tốc độ mà cổng không kiểm tra, PercMLP+ vẫn có 0,9–1,6% vùng đâm tránh được, trong khi IDM là 0. Qua cổng chỉ có nghĩa là "không kém hơn **trên những gì đã thử**".
4. **Thống kê sự kiện hiếm.** Với 2.883 quỹ đạo, IDM đâm 0,1% có CI [0,0; 0,3]. Với cỡ mẫu này, cổng không thể phân biệt hai bộ điều khiển có tỷ lệ va chạm nhỏ hơn nhiều so với độ rộng của khoảng tin cậy, trong khi một hệ thống thật phải đạt tỷ lệ cực thấp như vậy. Trên lát cắt xe dẫn phanh chỉ có 197 quỹ đạo nên biên phải nới tới 5 điểm.
5. **Baseline cũng là thầy.** PercMLP+ được gắn nhãn bằng chính IDM nên cổng đang so học trò với thầy. Cổng không trả lời được câu hỏi liệu IDM có đủ an toàn hay không.

**Việc cần làm tiếp:** chạy PercMLP+ ở chế độ shadow mode (chạy song song nhưng không điều khiển) trên xe thật với cảm biến thật và đo tỷ lệ bất đồng với IDM; mở rộng lưới sang tốc độ cao, nhiều làn, nhiều tác nhân phản ứng; thêm quy tắc ngoài phân bố huấn luyện (ví dụ 20 m/s) vào cổng; dùng lấy mẫu sự kiện hiếm hoặc nhiều dữ liệu hơn để thu hẹp khoảng tin cậy; và kiểm chứng bộ mô phỏng bằng log va chạm hoặc suýt va chạm thật.
