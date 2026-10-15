# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Văn Ước |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602445 |
| GitHub | nvuoc |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 4485 / 5505 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.26 m (lát cắt: 1.91 m) | 20.64% (CI: [19.20%, 22.15%]) | BLOCK (trượt R2, R3, R4, R6) |
| IDM | 1.52 m (lát cắt: 1.63 m) | 0.07% (CI: [0.02%, 0.25%]) | (baseline) |
| EgoMLP | 0.761 .. 0.767 m (TB: 0.76 m, lát cắt: 1.09 m) | 14.46% .. 22.79% (TB: 17.96%, xấu nhất CI: [21.29%, 24.36%]) | BLOCK (trượt R2, R3, R4, R5, R6) |
| PercMLP | 0.616 .. 0.620 m (TB: 0.62 m, lát cắt: 0.86 m) | 0.52% .. 1.28% (TB: 0.84%, xấu nhất CI: [0.93%, 1.76%]) | BLOCK (trượt R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.81 m (lát cắt: 1.00 m) | 0.17% .. 0.28% (TB: 0.22%, xấu nhất CI: [0.14%, 0.55%]) | RELEASE (vượt qua 7/7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong quá trình kiểm toán dữ liệu từ file `evidence/data_audit.json` trên tổng số 9993 quỹ đạo thô (992679 dòng ở tần số 10 Hz), tôi phát hiện ba vấn đề kỹ thuật trọng yếu:
1. **Dữ liệu trùng lặp nghiêm trọng**: Có đúng 383 quỹ đạo trùng lặp tuyệt đối (`exact_duplicates_removed = 383`), sau khi lọc làm sạch còn lại 9610 quỹ đạo chuẩn (`trajectories_clean = 9610`). Các quỹ đạo trùng nhau có thể nằm ở các chỉ số ID rất xa nhau.
2. **Cột gia tốc xe dẫn Acc_LV không thể sử dụng**: Hệ số tương quan tuyến tính giữa cột `Acc_LV` với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt 0.0454 (gần như bằng 0, không có giá trị vật lý), trong khi cột tương ứng của xe theo sau `Acc_FAV` có hệ số tương quan lên tới 0.9291 (92.9%). Điều này chứng tỏ tín hiệu gia tốc xe dẫn trong log bị lỗi phần cứng cảm biến hoặc vi phân trực tiếp từ tín hiệu đo thô đầy nhiễu. Do đó, hệ thống bắt buộc phải thay thế bằng gia tốc `a_lead` được tính bằng đạo hàm trên vận tốc xe dẫn đã làm mượt đa thức (`v_lead_s`).
3. **Phân bố dữ liệu mất cân bằng nặng**: Tình huống xe dẫn phanh gấp ($a < -2.0\text{ m/s}^2$) chỉ chiếm 6.39% số quỹ đạo (614/9610) và 4.84% tổng số cửa sổ trượt (slice_share = 0.0484). Đại đa số thời gian xe chạy êm dịu ở vận tốc trung vị 8.06 m/s và khoảng cách 13.33 m.
Lý do bắt buộc phải chia train/test theo **quỹ đạo** (trajectory-level split với seed 4485 ra 6727 train / 2883 test) mà không chia theo khung hình: Mỗi quỹ đạo là một chuỗi thời gian liên tục từ 70 đến 109 bước (trung vị 106.0 bước). Nếu chia ngẫu nhiên theo khung hình, các khung hình kế cận nhau ($t$ và $t+0.1\text{s}$) của cùng một xe sẽ đồng thời nằm ở cả tập train và test. Do tính tự tương quan thời gian cực lớn, mô hình học máy sẽ gặp hiện tượng rò rỉ dữ liệu (data leakage) và "học vẹt", cho ra điểm đánh giá cao giả tạo nhưng mất hoàn toàn khả năng khái quát hóa khi xe tự lái trong vòng kín.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ số liệu thực nghiệm trong `evidence/replay.json` trên 600 đoạn log (9.5 MB, gồm 60295 khung hình):
1. **Bản refactor float32**: Khác bit ở 600/600 đoạn (chiếm 100%, `not_bit_identical = 600`), nhưng vượt dung sai ở đúng 0/600 đoạn (`differ_beyond_tol = 0`), với sai số tuyệt đối lớn nhất cực nhỏ `max_abs_diff = 0.000403` m/s² (hoàn toàn nằm trong ngưỡng cho phép `atol = 0.001` và `rtol = 0.001`). Bản refactor không giống từng bit do tính chất không kết hợp của số thực trong máy tính: $(0.1 + 0.2) + 0.3 = 0.6000000000000001 \neq 0.1 + (0.2 + 0.3) = 0.6$. Việc tối ưu biểu thức và ép kiểu float32 làm xê dịch các bit thập phân cuối nhưng hành vi điều khiển vật lý không hề thay đổi, chứng minh kiểm thử có dung sai là bắt buộc để tránh báo động lỗi giả.
2. **Thay đổi thêm phanh khẩn cấp AEB (`aeb_ttc = 1.5` s)**: Làm lệch đúng 6/600 đoạn (`differ_beyond_tol = 6`, gồm các đoạn [56, 134, 179, 236, 359, 510]), với độ lệch gia tốc tối đa lên tới 1.9764 m/s². Khi mở chi tiết các đoạn bị lệch (như đoạn 56 hay 134), sự khác biệt diễn ra đúng tại các khung hình có tình huống khẩn cấp khi thời gian tới va chạm tức thời tụt xuống dưới 1.5 giây ($TTC = gap / closing < 1.5\text{ s}$). Ở các khung hình đó, thuật toán IDM thông thường chỉ phanh ở mức vừa phải (khoảng -1.5 đến -2.0 m/s²), trong khi bộ can thiệp AEB kích hoạt lệnh phanh kịch sàn -9.0 m/s², tạo ra bước nhảy hành vi lớn ở đúng kịch bản nguy hiểm.
3. **Phát theo thời gian thực (realtime) vs phát nhanh nhất (fastest)**: Chế độ `fastest` giả định năng lực xử lý vô hạn nên ghi nhận 0 khung hình trễ và 0 khung hình rớt (`late = 0`, `dropped = 0`). Ngược lại, chế độ `realtime` mô phỏng độ trễ thực tế của hệ điều hành và luồng tính toán đã bộc lộ 588 khung hình bị trễ hạn và 583 khung hình bị rớt trên tổng số 60295 khung, làm thay đổi lệnh điều khiển ở 382/600 đoạn với độ lệch gia tốc tối đa tới 16.6524 m/s². Phát thời gian thực cho thấy rủi ro chí mạng khi bộ điều khiển vi phạm giới hạn thời gian thực 100 ms (deadline overrun), buộc hệ thống phải giữ lệnh cũ hoặc trôi xe không kiểm soát.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Dựa trên dữ liệu sa hình cắt đầu trong `evidence/scenarios.json`:
1. **Thông số lưới**: Tốc độ hành trình $v_{ego} = 12.0$ m/s, dải khoảng cách từ 2.76 m đến 59.31 m (60 mốc), dải vận tốc tiến gần $dv$ từ 0.0 đến 12.0 m/s (45 mốc), tạo thành lưới 2700 ô kiểm thử.
2. **Tỷ lệ ô bất khả kháng (unavoidable frontier)**:
   - Khi không có độ trễ phản xạ ($\tau = 0.0$ s): Có 2.22% diện tích ô trên lưới (60/2700 ô, `unavoidable_area = 0.0222`) là hoàn toàn không thể tránh khỏi va chạm, do khoảng cách ban đầu nhỏ hơn quãng đường phanh vật lý cực đại $gap < dv^2 / (2 \cdot 9.0)$.
   - Khi độ trễ phản ứng tăng từ 0.0 lên 0.6 s: Vùng bất khả kháng tăng vọt lên 7.44% (201/2700 ô, `unavoidable_area = 0.0744`), tức tăng thêm 5.22 điểm phần trăm (tăng gấp hơn 3.3 lần). Trong 0.6 s trễ chấp hành, xe di chuyển thêm một quãng đường trôi $dv \times 0.6\text{ m}$ khiến biên an toàn bị thu hẹp đáng kể.
   - Vùng đâm của IDM: Tại $\tau = 0.0$ s, IDM đâm 2.22% diện tích lưới; tại $\tau = 0.6$ s, IDM đâm 7.44% diện tích lưới. Cả hai trường hợp này đều trùng khít 100% với biên bất khả kháng, nghĩa là diện tích va chạm quy kết lỗi cho thuật toán IDM (`attributable_area`) bằng đúng 0.00% (không lệch bất kỳ ô nào ngoài biên vật lý).
3. **Độ bao phủ của log**: Người lái con người trong log phanh gắt nhất (phân vị 99.9%) chỉ đạt mức gia tốc giảm tốc 3.6248 m/s² (`human_brake_limit_p99.9 = 3.62` m/s²), và trung vị phanh gấp khai phá chỉ là 2.53 m/s². Trong khi đó, lưới kiểm thử đòi hỏi gia tốc phanh kịch sàn lên tới $b_{max} = 9.0$ m/s², và có tới 8.56% số ô của lưới (`share_beyond_human_brake_limit = 0.0856`) đòi hỏi lực phanh vượt quá 3.62 m/s². Toàn bộ 614 sự kiện phanh gấp khai thác được từ log chỉ phân bố ở dải khoảng cách an toàn rộng ($> 10$ m) và $dv$ nhỏ, hoàn toàn không bao phủ được góc hiểm "khoảng cách hẹp, cắt đầu tốc độ cao".

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh đối chiếu giữa vòng hở và vòng kín (trích xuất từ `evidence/openloop.json` và `evidence/closedloop.json` trên 2883 quỹ đạo test giữ lại):

| Bộ điều khiển | L2 vòng hở (m) | Hạng vòng hở | Tỷ lệ va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| **PercMLP** | 0.6177 m (0.616 .. 0.620) | **1** | 0.84% (0.52% .. 1.28%) | **2** |
| **EgoMLP** | 0.7640 m (0.761 .. 0.767) | **2** | 17.96% (14.46% .. 22.79%) | **3** |
| **constant-velocity** | 1.2636 m | **3** | 20.64% (19.20% .. 22.15%) | **4** |
| **IDM** | 1.5249 m | **4** | 0.07% (0.02% .. 0.25%) | **1** |

Hiện tượng đảo ngược thứ tự (Ranking Inversion) xảy ra ở 3 cặp: `[const-vel, IDM]`, `[IDM, EgoMLP]`, và nổi bật nhất là cặp `[IDM, PercMLP]`.
- **Vì sao IDM có L2 tệ nhất (1.52 m) mà vòng kín lái an toàn nhất (0.07%, chỉ 2 ca va chạm)**: Vòng hở đo sai số L2 so với hành vi con người trong log. Con người lái xe có độ trễ sinh học, dao động chân ga thất thường và thói quen giữ cự ly không đồng đều. Luật IDM là một hàm toán học giải tích tối ưu khoảng cách an toàn theo hàm mũ bậc 4 ($s^*/s)^2$; nó chủ động phanh sớm hơn hoặc giữ khoảng cách rộng hơn con người nên sai số L2 lớn. Tuy nhiên, khi vào vòng kín (closed-loop), vòng lặp phản hồi cảm biến liên tục cập nhật khoảng cách thực tế: khi khoảng cách bị thu hẹp, IDM lập tức tăng lực phanh để triệt tiêu vận tốc tiến gần, giúp xe hoàn toàn tránh được va chạm.
- **Vì sao EgoMLP có L2 tốt hơn const-vel (0.76 m vs 1.26 m) mà vẫn đâm tới 17.96%**: EgoMLP chỉ được cung cấp đầu vào là lịch sử vận tốc của chính nó ($v_{ego}$), hoàn toàn không nhận diện được khoảng cách ($gap$) và vận tốc xe trước ($v_{lead}$). Trong vòng hở, do phần lớn dữ liệu là xe chạy đều, mô hình chỉ cần nội suy quán tính mượt mà là đạt L2 rất đẹp (0.76 m). Nhưng trong vòng kín, khi xe trước phanh gấp, EgoMLP bị "mù" hoàn toàn tình huống phía trước, tiếp tục duy trì tốc độ và gây tai nạn hàng loạt (đâm tới 48.17% trên lát cắt phanh gấp).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Số liệu thống kê đa hạt giống từ `evidence/closedloop.json` cho thấy sự bấp bênh lớn của các mô hình học máy:
1. **Dao động của EgoMLP qua 3 seed**:
   - Seed 5506: Tỷ lệ va chạm là 14.46% (417/2883 ca), khoảng tin cậy Wilson 95% là [13.23%, 15.80%].
   - Seed 5507: Tỷ lệ va chạm là 16.61% (479/2883 ca), khoảng tin cậy Wilson 95% là [15.30%, 18.02%].
   - Seed 5505: Tỷ lệ va chạm lên tới 22.79% (657/2883 ca), khoảng tin cậy Wilson 95% là [21.29%, 24.36%].
   - Độ chênh lệch giữa seed tốt nhất và seed xấu nhất lên tới 8.33 điểm phần trăm (tăng hơn 57% số vụ tai nạn giữa các lần khởi tạo ngẫu nhiên).
2. **Khoảng tin cậy Wilson 95% khẳng định điều gì**: Khoảng tin cậy của seed tốt nhất [13.23%, 15.80%] và seed xấu nhất [21.29%, 24.36%] hoàn toàn tách rời nhau (không hề chồng lấn, khoảng cách giữa 15.80% và 21.29% là 5.49 điểm phần trăm). Tương tự, PercMLP dao động từ 0.52% (seed 5505, CI: [0.32%, 0.86%]) lên tới 1.28% (seed 5506, CI: [0.93%, 1.76%]), tăng gấp 2.5 lần. Điều này chứng minh sự biến thiên này không phải do ngẫu nhiên của tập mẫu mà là do tính bất ổn định nội tại trong quá trình hội tụ trọng số của mạng nơ-ron.
3. **Nguy cơ chí mạng khi chỉ báo cáo một seed**: Nếu kỹ sư vô tình chỉ chạy một seed may mắn (như seed 5506 của EgoMLP hoặc seed 5505 của PercMLP với 0.52%), họ sẽ đưa ra nhận định sai lầm rằng hệ thống đã đủ an toàn để triển khai. Nhưng khi nạp mạng đó lên xe thực tế với một lần khởi tạo khác, tỷ lệ tai nạn thực tế có thể tăng gấp đôi hoặc gấp ba. Báo cáo một giá trị điểm duy nhất (point estimate) là hành vi che giấu rủi ro; đánh giá đa hạt giống kèm khoảng tin cậy Wilson là tiêu chuẩn an toàn bắt buộc để kiểm soát biên rủi ro tồi nhất (worst-case bound).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Từ kết quả thực nghiệm trong `evidence/agents.json` trên sa hình cắt đầu tại độ trễ phản xạ $\tau = 0.6$ s:
1. **Hiệu chỉnh thông số thời khoảng cách T**: Giá trị tối ưu là $T = 1.4$ giây, đạt khoảng cách Wasserstein-1 nhỏ nhất là $W_1 = 0.3464$ m so với phân bố headway của người lái trong log (headway trung vị của con người là 1.8640 s). Tác nhân phản ứng $T = 1.4$ s chân thực vượt bậc so với tác nhân không bao giờ phản ứng (`never_reacting_agent_W1 = 0.8732` m, giảm sai lệch phân bố hơn 2.5 lần).
2. **Bảng so sánh va chạm giữa hai thế giới**:

| Bộ điều khiển | Xe sau phát lại (Replay World) | Xe sau phản ứng (Reactive World) |
|---|---|---|
| **IDM** | Trước: 7.44% · Sau: **68.30%** · Tổng: **75.74%** | Trước: 7.44% · Sau: **0.00%** · Tổng: **7.44%** |
| **PercMLP** | Trước: 23.85% · Sau: **64.52%** · Tổng: **88.11%** | Trước: 24.30% · Sau: **0.00%** · Tổng: **24.30%** |
| **PercMLP+** | Trước: 7.70% · Sau: **67.96%** · Tổng: **75.67%** | Trước: 7.70% · Sau: **0.00%** · Tổng: **7.70%** |
| **const-vel** | Trước: 77.63% · Sau: **0.00%** · Tổng: **77.63%** | Trước: 77.63% · Sau: **0.00%** · Tổng: **77.63%** |
| **EgoMLP** | Trước: 73.89% · Sau: **0.00%** · Tổng: **73.89%** | Trước: 73.89% · Sau: **0.00%** · Tổng: **73.89%** |

3. **Cơ chế va chạm giả từ phía sau và sự thay đổi kết luận**:
   - Khi xe sau chỉ phát lại tĩnh (Replay), nó tua lại mù quáng chuỗi tốc độ người lái trong quá khứ mà không có cảm biến tương tác. Khi có xe cắt đầu, IDM xử lý thông minh bằng cách phanh gấp tối đa để không đâm xe phía trước. Nhưng chiếc xe sau không biết xe thử nghiệm vừa phanh nên giữ nguyên tốc độ và đâm dồn toa vào đuôi xe IDM, tạo ra tới 68.30% va chạm sau giả tạo! Trong khi đó, const-vel và EgoMLP không thèm phanh mà đâm thẳng vào xe trước nên xe sau không kịp đâm vào đuôi (rear = 0.00%).
   - Trong thế giới phát lại sai lầm này, IDM bị tính tổng tỷ lệ đâm lên tới 75.74% (gần tương đương mức 77.63% của const-vel), khiến người kiểm thử ngộ nhận IDM là bộ điều khiển thất bại. Khi chuyển sang thế giới phản ứng, toàn bộ 68.30% va chạm sau của IDM biến mất hoàn toàn (`rear = 0.00%`), khẳng định IDM đạt mức an toàn tối đa (tổng va chạm chỉ 7.44%, đúng bằng biên bất khả kháng vật lý).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Theo kết quả đánh giá cổng kiểm định trong `evidence/gate.json` và `evidence/fix.json`:
1. **Các quy tắc chặn PercMLP**: Mô hình bị CHẶN (BLOCK) do trượt đồng thời 4 quy tắc an toàn:
   - **R2 (Closed-loop logs)**: Trượt! Cận trên 95% ở seed xấu nhất là 1.76% (tỷ lệ 1.28%, 37 ca va chạm) vượt ngưỡng trần của baseline IDM là $0.07\% + 1.0\% = 1.07\%$.
   - **R3 (Braking slice)**: Trượt! Cận trên 95% trên lát cắt phanh là 6.0% (tỷ lệ 2.6%) vượt ngưỡng baseline $0.5\% + 5.0\% = 5.5\%$.
   - **R4 (Scenario grid cut-in)**: Trượt nặng! Diện tích va chạm quy kết lỗi là 16.85% vượt xa ngưỡng baseline $0.00\% + 2.0\% = 2.00\%$.
   - **R6 (Reactive world)**: Trượt! Tỷ lệ va chạm trong thế giới phản ứng là 24.30% vượt ngưỡng baseline $7.44\% + 2.0\% = 9.44\%$.
2. **Hai nguyên nhân gốc rễ**:
   - *Nguyên nhân A (Thiếu kịch bản trong log / Out-of-Distribution)*: Làm trượt R4 và R6. Dữ liệu Argoverse hầu như chỉ có xe bám đuôi thẳng hàng, hoàn toàn thiếu các ca xe khác tạt đầu ở khoảng cách ngắn.
   - *Nguyên nhân B (Trôi dạt phân phối / Covariate Shift)*: Làm trượt R2 và R3. Khi tự lái vòng kín, các sai số nhỏ tích tụ đưa xe vào các trạng thái lạ mà mạng chưa từng thấy trong tập huấn luyện tĩnh, khiến sai số bùng nổ dẫn tới va chạm.
3. **Giải pháp sửa lỗi**:
   - Sinh 20000 kịch bản cắt đầu mô phỏng (`cutin_windows`) dán nhãn bằng luật IDM $\rightarrow$ bổ sung miền dữ liệu thiếu, chữa dứt điểm nguyên nhân A.
   - Áp dụng thuật toán DAgger (`dagger_windows`) qua 2 vòng: cho mô hình tự lái vòng kín, ghi lại chính các trạng thái chệch hướng mà nó tự đi vào, mời chuyên gia IDM gán nhãn dự báo 3 giây để huấn luyện lại $\rightarrow$ dạy mạng khả năng tự sửa lỗi khi bị chệch quỹ đạo, chữa đúng nguyên nhân B.
4. **Kết quả PercMLP+ và câu hỏi "có tốt hơn IDM không"**:
   - PercMLP+ vượt qua trọn vẹn 7/7 quy tắc và được cấp quyền RELEASE (cận trên R2 giảm xuống 0.55% $\le 1.07\%$, diện tích va chạm R4 giảm từ 16.85% xuống 0.26% $\le 2.00\%$, R6 giảm về 7.70% $\le 9.44\%$).
   - Tuy nhiên, **PercMLP+ TUYỆT ĐỐI KHÔNG TỐT HƠN IDM!** Về mặt số liệu, tỷ lệ va chạm trên log của PercMLP+ là 0.28% (vẫn gấp 4 lần mức 0.07% của IDM), diện tích va chạm trên sa hình là 0.26% (trong khi IDM là 0.00%). Về mặt lý thuyết, PercMLP+ được huấn luyện từ nhãn của IDM nên IDM chính là "trần năng lực" của nó. Cổng phát hành cấp phép vì PercMLP+ thỏa mãn tiêu chuẩn **"không kém hơn đáng kể" (non-inferiority within margin)**, chứ không phải vì nó vượt trội hơn baseline.
5. **Quyết định phát hành và đề xuất**:
   - **Chưa nên đưa lên xe thật ngay lập tức**. Mạng MLP là hộp đen phi tuyến, thiếu sự bảo đảm chứng minh toán học chống va chạm.
   - Đề xuất trước khi phát hành: Thiết lập kiến trúc điều khiển lai an toàn (Safety Shield Architecture). Sử dụng PercMLP+ làm bộ điều khiển chính để tạo cảm giác lái tự nhiên, mượt mà giống con người; nhưng luôn duy trì một rào chắn an toàn chạy song song bằng luật vật lý IDM/AEB có quyền ưu tiên ngắt lệnh (override) ngay lập tức khi phát hiện khoảng cách an toàn bị vi phạm.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Mặc dù bài lab cung cấp một quy trình kiểm định khoa học chặt chẽ, kết luận an toàn của bài lab vẫn bị giới hạn bởi ba rào cản hiện thực cốt lõi:
1. **Mô hình động học một chiều (1-D Longitudinal) bỏ qua hoàn toàn động lực học xe thực tế**: Bài lab giả định xe là một chất điểm di chuyển dọc trên đường thẳng, bỏ qua hoàn toàn các yếu tố động học ngang (đánh lái chuyển làn tránh chướng ngại vật), góc trượt lốp, ma sát biến thiên theo thời tiết (đường ướt mưa, trơn trượt có hệ số ma sát $\mu < 0.7$) và độ trễ phản hồi cơ cấu chấp hành thủy lực của hệ thống phanh (actuator lag). Lực phanh tối đa $9.0\text{ m/s}^2$ trong mô phỏng là một hằng số tức thời lý tưởng; trên xe thật, một cú đạp phanh $9.0\text{ m/s}^2$ khi đường trơn có thể gây bó cứng bánh xe, kích hoạt ABS giật cục hoặc trượt bánh lật xe.
2. **Giả định cảm biến hoàn hảo không có nhiễu và độ trễ nhận diện (No Perception Degradation / Occlusion)**: Mô hình nhận trực tiếp giá trị vị trí, khoảng cách ($gap$) và vận tốc ($v_{lead}$) sạch từ bộ mô phỏng. Trên xe tự hành thực tế, các biến trạng thái này được ước lượng từ hệ thống cảm biến phức tạp (Camera, LiDAR, Radar). Các cảm biến này luôn tồn tại độ trễ xử lý (tracking latency), nhiễu đo đạc, hiện tượng mất dấu mục tiêu (track drop/ghost detection) khi đi qua gầm cầu, hoặc bị che khuất tầm nhìn (occlusion) bởi xe buýt/xe tải đi bên cạnh. Một mạng nơ-ron học máy rất nhạy cảm với nhiễu đầu vào; chỉ một sai số ước lượng khoảng cách vài mét có thể khiến mạng đưa ra quyết định sai lầm thảm khốc.
3. **Môi trường mô phỏng tĩnh và thiếu sự tương tác xã hội đa tác nhân (Synthetic Sim-to-Real Gap)**: Trong bài lab, xe dẫn đường chỉ phát lại cứng nhắc chuỗi log cũ mà không hề có phản xạ né tránh khi xe thử nghiệm tiếp cận gần, còn xe sau chỉ phản ứng theo một luật IDM đồng nhất ($T = 1.4$ s). Trong thế giới giao thông thực tế, con người liên tục giao tiếp phi ngôn ngữ, tương tác trò chơi (game-theoretic interaction), có các hành vi tạt đầu ác ý, xe máy chen lấn làn, hoặc người đi bộ bất ngờ băng qua đường. Hành vi của thế giới thực phức tạp và khó lường hơn sa hình mô phỏng hàng nghìn lần.

**Các bước hành động tiếp theo để thu hẹp khoảng cách Sim-to-Real**:
- Nâng cấp môi trường mô phỏng sang 3D hoàn chỉnh (như CARLA, BeamNG hoặc Simian) tích hợp đầy đủ mô hình vật lý lốp Pacejka, độ nghiêng mặt đường, điều kiện thời tiết khắc nghiệt và độ trễ cơ cấu chấp hành CAN-bus.
- Bổ sung mô hình nhiễu cảm biến thực tế (Sensor Noise Injection, jitter, dropout và trễ nhận diện của mô hình Perception).
- Đưa thuật toán ra kiểm nghiệm thực tế theo từng cấp độ: bắt đầu từ thử nghiệm phần cứng trong vòng lặp (Hardware-in-the-Loop - HIL), thử nghiệm trên sa hình thực tế khép kín (Proving Ground) với các xe bia mô hình mềm tự hành, trước khi triển khai thử nghiệm hạn chế trên đường phố với tài xế an toàn (Safety Driver) giám sát liên tục.

