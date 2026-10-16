# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Đồng Mạnh Hùng |
| MSSV | 2A202602412 |
| GitHub | https://github.com/Hung23020370/K4-Track4-DAY12-DongManhHung-2A202602412 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8047 / 9125 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Số lấy từ `evidence/openloop.json`, `closedloop.json`, `gate.json`. Với mô hình học, cột va chạm là **dải qua 3 seed** (min..max); constant-velocity và IDM là bộ điều khiển tất định nên chỉ có một lần chạy.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.286 | 21.5 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.547 | 0.21 | (baseline) |
| EgoMLP | 0.770 | 14.5 .. 27.8 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.627 | 0.62 .. 1.21 | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.833 | 0.28 .. 0.31 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu

Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?
Bước 0 cho thấy ba vấn đề. (1) Cột `Acc_LV` không dùng được: tương quan với d(Speed_LV)/dt chỉ 0.045, trong khi cột gia tốc của xe theo sau là 0.929. Vì vậy mình dùng gia tốc xe dẫn suy ra từ `Speed_LV` đã làm mượt để khai thác ca phanh. (2) Có 383 quỹ đạo trùng hệt nhau (9993 -> 9610 quỹ đạo sạch), nằm ở các id xa nhau, nên chia ngẫu nhiên theo khung hình hay theo id liền kề sẽ để cùng một chuyến xuất hiện ở cả train và test (rò rỉ). (3) Phanh mạnh hiếm trong log: độ giảm tốc đã làm mượt của xe dẫn ở phân vị 1 chỉ là -1.95 m/s^2. Mình chia theo quỹ đạo, không theo khung hình, vì các khung trong một quỹ đạo gần như giống nhau; chia theo khung sẽ làm test "nhìn thấy" train.

## Câu 2 · Phát lại

Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?
Chạy lại cùng mã: 0/600 đoạn khác nhau. Bản refactor float32: không giống từng bit ở 600/600 đoạn nhưng vượt dung sai ở 0 đoạn (lệch lớn nhất 4.0e-4, dung sai atol = rtol = 1e-3), nên so bằng đẳng thức tuyệt đối là sai test; cộng float không có tính kết hợp (ví dụ 0.6000000000000001 so với 0.6). Thay đổi AEB làm lệch 11/600 đoạn (lệch lớn nhất 1.84 m/s^2): luật chỉ kích hoạt ở khung có TTC dưới 1.5 s và đang tiến gần, nên các khung còn lại của đoạn (và hầu hết các đoạn) không đổi. Ví dụ ở đoạn 58, các khung 98 đến 101 có TTC 1.47, 1.43, 1.43 và 1.49 s (khoảng cách 4.30 đến 3.62 m, tốc độ tiến gần 2.92 đến 2.43 m/s); lệnh cũ -5.44, -5.78, -5.91, -5.68 đều bị AEB chặn thành -6.0 m/s^2.. Phát nhanh nhất có 0 khung rớt và 0 khung trễ trên 59568 khung; phát theo thời gian thật có 577 khung rớt và 581 khung trễ, làm 370/600 đoạn vượt dung sai (lệch tới 215.7). Phát nhanh nhất không có hạn chót nên khung chậm (đột biến 140 ms > 100 ms) vô hình; theo thời gian thật khung đó trễ, khung kế tiếp bị rớt và lệnh cũ bị giữ lại.

## Câu 3 · Kịch bản

Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)
Lưới: tốc độ cruise 14 m/s, khoảng cách 2.76 .. 59.31 m, 2700 ô (60 x 45). Ô không ai tránh được: 3.6 % khi trễ 0 s và 10.2 % khi trễ 0.6 s (tăng gần 3 lần). Vùng đâm của IDM đúng bằng vùng bất khả thi ở cả hai độ trễ (collision_area = unavoidable_area, phần "đổ lỗi cho bộ điều khiển" = 0), còn constant-velocity đâm 80.6 % với 77.0 % / 70.4 % là lỗi của chính nó. Nhờ biên này mới phân biệt được "ai cũng đâm" với "bộ điều khiển đâm". Log chưa phủ vùng khó: người lái phanh mạnh nhất cỡ 3.62 m/s^2 (phân vị 99.9), mà 12.7 % lưới đòi phanh mạnh hơn; 614 ca phanh khai thác được chỉ chiếm 6.4 % quỹ đạo, với giảm tốc đỉnh trung vị -2.53 và p99 -4.21 m/s^2.

## Câu 4 · Vòng hở và vòng kín

Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?
Xếp hạng L2 (thấp là tốt): PercMLP 0.627 (1), EgoMLP 0.770 (2), constant-velocity 1.286 (3), IDM 1.547 (4). Xếp hạng va chạm: IDM 0.21 % (1), PercMLP 0.86 % trung bình 3 seed (2), EgoMLP 21.4 % (3), constant-velocity 21.5 % (4). Ba cặp đổi thứ tự: (constant-velocity, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất vì nó là luật tự đặt (T = 1.2 s) chứ không bắt chước người lái (thời khoảng cách trung vị của người lái 1.86 s), nên "dự báo" lệch người lái; nhưng nó phản ứng với khoảng cách và tốc độ tiến gần nên không đâm. L2 chỉ đo giống người lái, không đo an toàn. EgoMLP chỉ thấy tốc độ của chính nó, không thấy xe dẫn, nên học được "người lái thường làm gì" (L2 tốt hơn constant-velocity) nhưng không thể biết xe dẫn đang phanh: va chạm 21.4 % trung bình, và 59.4 % trên lát cắt xe dẫn phanh (n = 192).

## Câu 5 · Một lần chạy chưa đủ

EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?
Va chạm của EgoMLP qua 3 seed là 14.5 %, 22.0 % và 27.8 %. Khoảng tin cậy Wilson 95 % của seed xấu nhất là [26.2 %, 29.4 %], cận dưới này vẫn cao hơn tỷ lệ của seed tốt nhất (14.5 %), nên hai seed không chồng nhau: sai khác là do huấn luyện chứ không do lấy mẫu. Nếu chỉ chạy một seed, mình có thể báo 14.5 % hoặc 27.8 % tuỳ may, một kết quả nghe "tốt hơn" gần gấp đôi cái kia, và không biết con số nào đại diện. Quy tắc R5 cũng bắt đúng điều này: EgoMLP rớt (14.5 .. 27.8 %). PercMLP ổn định hơn (0.62 .. 1.21 %) nhưng vẫn là một dải, không phải một số.

## Câu 6 · Tác nhân phản ứng

Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?
Xe sau phát lại và xe sau phản ứng, trễ 0.6 s (trên lưới cắt vào, tỷ lệ ô va chạm):

| Bộ điều khiển | Phát lại: trước / sau / bất kỳ | Phản ứng: trước / sau / bất kỳ |
|---|---|---|
| constant-velocity | 80.6 / 0.0 / 80.6 | 80.6 / 0.0 / 80.6 |
| IDM | 10.2 / 70.6 / 80.9 | 10.2 / 0.0 / 10.2 |
| PercMLP | 35.0 / 40.3 / 75.3 | 35.1 / 0.0 / 35.1 |
| PercMLP+ | 10.6 / 70.2 / 80.7 | 10.6 / 0.0 / 10.6 |

`T` hiệu chỉnh = 1.4 s (khoảng cách Wasserstein tới log 0.340, so với 0.881 của xe sau không bao giờ phản ứng), nên xe sau chân thực hơn rõ rệt. Xe sau phát lại là bản ghi người lái, không thể biết xe thử vừa phanh, nên IDM phanh gấp bị tính "va chạm giả" từ phía sau (70.6 % ô). Ở thế giới phát lại, IDM (80.9 %) trông tệ ngang constant-velocity (80.6 %); ở thế giới phản ứng là 10.2 % so với 80.6 %. Kết luận đảo ngược hoàn toàn chỉ vì loại tác nhân.

## Câu 7 · Cổng phát hành và sửa

Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?
PercMLP bị chặn bởi R2 (cận trên 1.68 % so với baseline 0.21 % + 1 điểm), R3 (lát cắt phanh 4.2 %, cận trên 8.0 % so với 1.6 % + 5 điểm), R4 (24.9 % lưới so với 0.0 % + 2 điểm) và R6 (35.1 % so với 10.2 %). Hai nguyên nhân gốc: (a) kịch bản log không chứa: R4/R6 cho thấy 24.9 % lưới cắt vào tránh được mà vẫn đâm, vì log không có ca nào phanh mạnh như vùng này (12.7 % lưới vượt giới hạn phanh của người); (b) sai số tích luỹ khi tự lái: R2 và R3 rớt với va chạm vòng kín cao nhất 1.21 % dù L2 vòng hở là tốt nhất (0.627). Cách sửa: thêm 20000 kịch bản cắt vào mô phỏng (nhãn do IDM) chữa (a), và DAgger 2 vòng (khoảng 895 nghìn cửa sổ, nhãn IDM tại chính trạng thái mô hình tự đi vào) chữa (b). Sau sửa PercMLP+ qua cả 7 quy tắc: R2 cận trên 0.59 %, R4 0.3 %, R6 10.6 % so với 10.2 %. Ở tốc độ cruise 20 m/s chưa từng thấy khi huấn luyện (<= 16), PercMLP đâm 26.4 .. 34.0 % lưới còn PercMLP+ 0.3 .. 1.5 %, IDM 0.0 %. Nhưng PercMLP+ **không được chứng minh là tốt hơn** IDM, chỉ không kém hơn: va chạm trên log 0.31 % so với 0.21 %, lưới 0.3 % so với 0.0 %, thế giới phản ứng 10.6 % so với 10.2 %. Nó học từ nhãn IDM nên nhiều nhất là bắt chước thầy. L2 vòng hở của nó còn xấu đi (0.833 so với 0.627 của PercMLP) trong khi vòng kín tốt lên, nên điểm vòng hở không phản ánh an toàn. Mình không đề xuất thay IDM ngay: cổng qua chưa có nghĩa là có lợi ích để đánh đổi rủi ro. Việc nên làm trước: chạy bóng (shadow mode) trên xe thật, thử thêm nhiễu cảm biến, độ trễ và tốc độ khác, và nêu rõ PercMLP+ mang lại điều gì mà IDM không có.

## Câu 8 · Bài lab này không chứng minh được gì

Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?
(1) Mô hình chỉ một chiều dọc: không có cắt làn, đánh lái, hay tránh sang bên, nên "không va chạm" chỉ đúng trong một thế giới đã bị đơn giản hoá. (2) Không có cảm biến thật: bộ điều khiển nhận khoảng cách và tốc độ xe dẫn gần như chính xác (chỉ thử nhiễu 1.0 m và 0.5 m/s ở một thí nghiệm nhỏ), trong khi lỗi nhận thức ngoài đời là nguồn rủi ro lớn. (3) Xe dẫn trong log không phản ứng với xe thử, và thế giới mô phỏng (lưới cắt vào, xe sau IDM với T = 1.4 s hiệu chỉnh chỉ từ phân bố thời khoảng cách) là giả định của chúng ta; "va chạm 0" trong mô phỏng không phải bằng chứng an toàn. Ngoài ra chỉ có 3 seed, một tốc độ cruise (14 m/s) cho lưới chính, và ngưỡng cổng là lựa chọn của đội. Cần làm tiếp: thử trong vòng kín có nhiễu và trễ thật, chạy bóng trên xe thật, mở rộng kịch bản và nhiều dữ liệu hơn, và chuyển sang mô hình hai chiều.
