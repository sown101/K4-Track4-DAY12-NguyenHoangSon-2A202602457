# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Nguyễn Hoàng Sơn |
| MSSV (đúng bằng `--student-id` đã chạy) | `2A202602457` |
| GitHub | [sown101](https://github.com/sown101) |
| Seed chia dữ liệu / seed mô hình | `9903` / `4124, 4125, 4126, 4127, 4128` |
| Số seed huấn luyện (`--seeds`) | `5` |

MSSV được giữ nguyên cả chữ `A`: `config.student_params` băm chuỗi bằng SHA-256, không chuyển MSSV trực tiếp thành số nguyên. Vì vậy không cần rút còn 5 số cuối. MSSV xác định seed chia dữ liệu, seed mô hình và tốc độ cruise của lưới là 12 m/s.

## Kết quả cuối

Kết quả được chạy đầy đủ, không dùng `--quick`. Dải trong bảng là min–max qua 5 seed; baseline xác định chỉ có một lần chạy. Tỷ lệ trong bảng là phần trăm.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.2662 | 19.9445 | BLOCK |
| IDM | 1.5233 | 0.0347 (1/2883) | baseline |
| EgoMLP | 0.7600–0.7720 | 13.0420–22.8928 | BLOCK |
| PercMLP | 0.6251–0.6280 | 0.6937–2.0118 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.7945–0.8205 | 0.2775–0.3122 | RELEASE (7/7 quy tắc) |

Nguồn: `evidence/openloop.json`, `closedloop.json`, `gate.json`; dải L2 của PercMLP+ được tính lại từ các model đã huấn luyện và lưu trong `evidence/report_details.json`.

## Câu 1 · Dữ liệu

Dữ liệu có 992679 dòng, 9993 quỹ đạo ban đầu; loại 383 bản trùng còn 9610 quỹ đạo (`data_audit.json`).
Tương quan của `Acc_LV` với đạo hàm `Speed_LV` chỉ 0.04535, trong khi cột gia tốc xe theo sau đạt 0.92909; vì vậy không dùng trực tiếp `Acc_LV` để tìm ca phanh.
Gia tốc xe dẫn được suy ra bằng đạo hàm tốc độ đã làm mượt, lưu ở `logs.a_lead`; bước thời gian là 0.1 s.
Các quỹ đạo trùng cần loại trước khi chia tập, vì cùng một chuyển động xuất hiện ở train và test sẽ làm kết quả đánh giá quá lạc quan.
Chia theo khung hình cũng gây rò rỉ vì các khung lân cận của một quỹ đạo rất giống nhau; seed 9903 chia theo quỹ đạo thành 6727 train và 2883 test.
Lưu ý khác là `Spatial_Gap` là khoảng cách giữa hai cản xe: chênh lệch vị trí trừ 4.5 m, không phải khoảng cách giữa hai điểm tham chiếu xe.

## Câu 2 · Phát lại

Trên 600 đoạn, chạy cùng mã hai lần cho 0 đoạn khác; bản float32 khác chính xác ở 600 đoạn nhưng 0 đoạn vượt dung sai (`replay.json`).
Sai khác tuyệt đối lớn nhất của refactor là 1.81055e-5 m/s²; so sánh dùng `abs(new-golden) <= 0.001 + 0.001*abs(golden)` và coi hai NaN phần đệm cùng vị trí là bằng nhau.
AEB làm lệch 17/600 đoạn: đây là thay đổi hành vi có ý nghĩa, khác với sai số làm tròn số thực.
Mở đoạn 65 thấy đúng các khung 75, 76, 77 đổi lệnh; tại khung 75, gap=4.9981 m và closing=3.3883 m/s nên TTC=1.47511 s, dưới ngưỡng AEB 1.5 s.
Lệnh phanh ở khung này đổi từ -4.38453 sang -6.0 m/s²; hai khung tiếp có TTC 1.46219 và 1.43567 s (`report_details.json`, mục `replay_case`).
Phát nhanh nhất ghi 0 khung trễ/rớt; phát theo lịch thời gian có 584 khung trễ, 580 khung rớt trên 59917 khung và 390 đoạn vượt dung sai. Cách phát nhanh che khuất vi phạm hạn xử lý; thời gian xử lý trong lab được mô phỏng, không phải đo hiệu năng phần cứng thực.

## Câu 3 · Kịch bản

Lưới có 60 khoảng cách × 45 tốc độ tiến gần = 2700 ô, cruise 12 m/s; gap từ 2.76005 đến 59.31233 m, closing từ 0 đến 12 m/s (`scenarios.json`).
Với xe dẫn giữ tốc độ, khoảng cách cần thiết để phanh là `dv*t_react + dv²/(2*b_max)`; khi gap nhỏ hơn biên này thì không bộ điều khiển nào tránh được tiếp xúc trong giả định vật lý của bài.
Độ trễ tăng từ 0 lên 0.6 s làm vùng bất khả thi tăng từ 2.2222% lên 7.4444%; vùng va chạm IDM tăng tương ứng 5.2222 điểm phần trăm, còn vùng tránh được nhưng IDM vẫn đâm bằng 0%.
Kiểm chứng bằng bộ phanh tối đa cho 0 ô lệch ngoài vùng lân cận biên ở cả hai độ trễ; các ô cách biên dưới 5% được bỏ qua vì bước tích phân rời rạc.
Khai thác được 614 sự kiện phanh trong 9610 quỹ đạo (6.3892%); các chấm log tập trung ở tốc độ tiến gần thấp, ít phủ vùng khoảng cách nhỏ và closing lớn trên `scenarios_idm.png`.
Mức phanh người lái tại phân vị 99.9% là 3.62484 m/s², thấp hơn giới hạn cơ cấu 9 m/s²; 8.5556% lưới đòi mức phanh lớn hơn phân vị này. Đây là thống kê phân vị, không phải khẳng định mọi khung người lái đều phanh nhẹ hơn 3.62484 m/s².

## Câu 4 · Vòng hở và vòng kín

Bảng dùng L2 trung bình và tỷ lệ va chạm trung bình qua seed để xếp hạng, đúng với mục `ranking` trong `closedloop.json`; bảng đầu báo cáo trình bày thêm dải qua seed.

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1.2662 | 3 | 19.9445 | 4 |
| IDM | 1.5233 | 4 | 0.0347 | 1 |
| EgoMLP | 0.7640 | 2 | 18.5848 | 3 |
| PercMLP | 0.6270 | 1 | 1.2556 | 2 |

Ba cặp đảo thứ hạng là constant-velocity/IDM, IDM/EgoMLP và IDM/PercMLP, đúng với `order_flips`.
IDM có L2 cao vì hành vi theo luật và dự báo xe dẫn giữ tốc độ không trùng quỹ đạo người lái; khi tự lái, luật vẫn chủ động giữ khoảng cách và phanh theo tốc độ tiến gần.
EgoMLP học xu hướng tốc độ người lái nên L2 tốt hơn constant-velocity, nhưng không quan sát gap và tốc độ xe dẫn, nên không đủ thông tin để ứng phó khi xe trước phanh hoặc cắt vào.
Trong vòng kín, trạng thái sau chịu ảnh hưởng của quyết định trước, khiến sai số tích lũy; điểm bắt chước trên trạng thái log không kiểm tra được cơ chế này.
Nhóm xe dẫn sắp phanh chỉ chiếm 4.3704% của 72098 cửa sổ test; L2 PercMLP trên nhóm này là 0.8813 m so với 0.6270 m toàn bộ (`openloop.json`), cho thấy trung bình che lỗi ở tình huống khó.

## Câu 5 · Một lần chạy chưa đủ

Trên cùng 2883 quỹ đạo test, EgoMLP thay đổi từ 13.0420% ở seed 4126 lên 22.8928% ở seed 4128: dải rộng 9.8508 điểm phần trăm (`closedloop.json`).
Seed 4126 có 376 va chạm, Wilson 95% là [11.8617%, 14.3206%]; seed 4128 có 660 va chạm, khoảng là [21.3958%, 24.4620%].
Hai khoảng không chồng nhau, cho thấy lựa chọn seed làm kết luận thay đổi rõ rệt trên tập thử này; không nên chỉ công bố một seed tốt nhất.
Chỉ chạy seed 4126 có thể báo 13.04%, tốt hơn constant-velocity 19.94%; chỉ chạy seed 4128 lại có thể báo 22.89%, kém hơn baseline đó.
Wilson mô tả bất định lấy mẫu cho từng tỷ lệ; dải giữa các seed mô tả biến thiên do huấn luyện, nên cần báo cả hai thay vì coi một khoảng là bao trùm mọi bất định.
Ngay cả IDM chỉ có 1/2883 va chạm (0.0347%), cận trên Wilson vẫn là 0.1962%; một tỷ lệ nhỏ trên mẫu hữu hạn không chứng minh hệ thống an toàn trong mọi hoàn cảnh.

## Câu 6 · Tác nhân phản ứng

Hiệu chỉnh trên 3000 quỹ đạo train chọn `T=1.4 s`, với Wasserstein-1 giữa phân bố headway mô phỏng và người lái là 0.34887, tốt hơn xe không phản ứng 0.85302 (`agents.json`).
Headway trung vị của người lái là 1.88049 s; `T` là tham số của IDM, không phải bản sao trực tiếp của trung vị đó. Xe sau có độ trễ 0.6 s và gap khởi tạo 18.8 m.
Bảng sau lấy tại độ trễ điều khiển ego 0.6 s; các tỷ lệ là phần trăm trên toàn bộ 2700 ô, với mô hình học lấy hàng được chọn theo trường hợp reactive xấu nhất qua seed.

| Bộ điều khiển | Xe sau phát lại: mọi va chạm (%) | Xe sau phản ứng: mọi va chạm (%) | Va chạm sau: phát lại / phản ứng (%) |
|---|---:|---:|---:|
| constant-velocity | 77.6296 | 77.6296 | 0 / 0 |
| IDM | 75.7407 | 7.4444 | 68.2963 / 0 |
| EgoMLP | 77.9259 | 77.9259 | 0 / 0 |
| PercMLP | 84.5556 | 27.2222 | 58.0370 / 0 |
| PercMLP+ | 75.5926 | 7.7778 | 67.8148 / 0 |

Khi IDM phanh để tránh xe phía trước, xe sau phát lại vẫn chạy tiếp mà không biết ego đã phanh, tạo 68.2963% ô va chạm phía sau; đây là giả định thế giới làm sai lệch đánh giá trách nhiệm của bộ điều khiển.
Ở thế giới phát lại, IDM 75.7407% chỉ nhỉnh hơn constant-velocity 77.6296%; với xe sau phản ứng, khoảng cách mở rộng thành 7.4444% so với 77.6296%. Thứ tự của cặp này không đảo ở lần chạy này, nhưng mức khác biệt thay đổi rất lớn.
Xe sau phản ứng hợp lý hơn theo phân bố headway đã hiệu chỉnh; việc khớp một thống kê không chứng minh nó tái hiện đầy đủ hành vi người lái hay mọi tình huống tương tác.

## Câu 7 · Cổng phát hành và sửa

PercMLP bị chặn bởi R2, R3, R4, R5, R6; R1 và R7 đạt (`gate.json`).
R2 có cận trên va chạm 2.5918%, vượt baseline 0.0347% + 1 điểm phần trăm; R3 có cận trên nhóm phanh khoảng 13.1%, vượt ngưỡng 5%.
R4 ghi 19.7778% ô tránh được nhưng vẫn đâm, vượt 2%; R5 có dải va chạm 0.6937–2.0118%, rộng 1.3181 điểm phần trăm, vượt 1; R6 ghi 27.2222% va chạm trong thế giới phản ứng, vượt 7.4444% + 2 điểm phần trăm.
Hai nguyên nhân phù hợp với bằng chứng là thiếu các kịch bản cắt vào khó ngoài phân bố log (R4/R6), và sai số bắt chước tích lũy khi tự lái (R2/R3/R5); chưa có thí nghiệm ablation để tách định lượng đóng góp từng nguyên nhân.
Bước sửa thêm 197040 cửa sổ từ 20000 kịch bản cắt vào vào 167035 cửa sổ log train, rồi chạy 2 vòng DAgger: lấy trạng thái mô hình tự đi vào trên train và gắn nhãn dự báo 3 s của IDM tại đúng trạng thái đó.
Sau sửa, R2–R6 đều chuyển sang đạt: cận trên log còn 0.5923%, cận trên nhóm phanh 3.0300%, vùng đâm tránh được 0.3333%, dải qua seed chỉ 0.0347 điểm phần trăm và thế giới phản ứng còn 7.7778%; PercMLP+ đạt 7/7 quy tắc và được RELEASE trong phạm vi lab (`fix.json`, `gate.json`).
L2 trung bình lại tăng từ 0.6270 lên 0.8093 m, trong khi va chạm giảm còn 0.2775–0.3122%; mô hình chấp nhận dự báo ít giống người lái hơn để xử lý trạng thái vòng kín tốt hơn. Nhãn IDM cho các tình huống khó và trạng thái tự đi vào phù hợp với hai nguyên nhân trên, nhưng chưa có ablation nên không quy toàn bộ cải thiện cho riêng DAgger.
Qua cổng là đạt tiêu chí không kém hơn với dung sai, không chứng minh vượt IDM: PercMLP+ có 8–9 va chạm/2883 so với IDM 1/2883 và học từ chính chuyên gia IDM. Có thể vượt chuyên gia về nguyên tắc nếu có mục tiêu hoặc dữ liệu tốt hơn, nhưng thí nghiệm này chưa chứng minh điều đó; chưa nên thay IDM trên xe thực, cần kiểm thử độc lập, mở rộng miền và chạy shadow trước.

## Câu 8 · Bài lab này không chứng minh được gì

Thứ nhất, mô phỏng chỉ một chiều dọc với bước 0.1 s và giới hạn phanh 9 m/s², chưa có chuyển làn thực, đường cong, bám đường thay đổi hay động học đầy đủ; cần kiểm thử thêm trong mô phỏng có các yếu tố này và trên bãi thử có kiểm soát.
Thứ hai, kiểm thử nhiễu chỉ thêm sai số gap 1 m và tốc độ xe dẫn 0.5 m/s; chưa có pipeline cảm biến, che khuất, mất phát hiện và độ trễ biến thiên, nên cần đánh giá perception và điều khiển cùng nhau với lỗi có cấu trúc.
Thứ ba, xe dẫn vẫn phát lại log, xe sau phản ứng chỉ dùng IDM hiệu chỉnh headway; chênh lệch 75.7407% so với 7.4444% của IDM giữa hai thế giới cho thấy giả định tác nhân ảnh hưởng mạnh đến kết luận, cần nhiều kiểu người lái và tương tác hai chiều được kiểm chứng bằng dữ liệu thật.
Thứ tư, dữ liệu bổ sung chỉ phủ cruise 4–16 m/s: tại kiểm tra ngoài miền 20 m/s, PercMLP+ vẫn có 0.2593–2.0370% ô tránh được nhưng đâm, trong khi IDM là 0% (`fix.json`). Có seed vượt mốc 2% dùng trong R4, nên không suy rộng quyết định RELEASE tại 12 m/s sang tốc độ khác; cần thêm test ngoài miền được giữ riêng trước khi mở rộng phạm vi.
Cuối cùng, 2883 quỹ đạo test ngắn và 2700 ô có tham số là bằng chứng trong phạm vi hữu hạn; Wilson không bao hàm sai lệch mô phỏng, tương quan giữa tình huống hoặc biến cố cực hiếm. Cần thêm dữ liệu độc lập và các họ kịch bản chưa dùng để sửa mô hình trước quyết định triển khai thực tế.

## Bằng chứng và tái lập

- Tám file chính: `data_audit.json`, `replay.json`, `scenarios.json`, `openloop.json`, `closedloop.json`, `agents.json`, `gate.json`, `fix.json` trong `evidence/`.
- `report_details.json`: môi trường chạy, các khung AEB đã mở và L2 từng seed của PercMLP+; mọi số được tính từ log/model, không sửa bằng tay.
- Hình: `evidence/scenarios_idm.png` và `evidence/openloop_vs_closedloop.png`.
- Môi trường: Python 3.13.1, numpy 2.4.2, pandas 3.0.0, matplotlib 3.10.8; giới hạn BLAS ở 1 luồng CPU.

Lệnh tái lập trên PowerShell, chạy từ thư mục gốc repo:

```powershell
$env:PYTHONUTF8 = "1"
$env:OPENBLAS_NUM_THREADS = "1"
$env:OMP_NUM_THREADS = "1"
$env:MKL_NUM_THREADS = "1"
python -m unittest discover -s tests -v
python run_all.py --student-id 2A202602457 --seeds 5
python scripts/check_submission.py --save-verify
```

Không đưa dữ liệu thô hoặc `work/` vào Git. Quyết định RELEASE/BLOCK của lab chỉ có ý nghĩa trong phạm vi kiểm thử và ngưỡng đã định, không phải chứng nhận an toàn xe thực.
