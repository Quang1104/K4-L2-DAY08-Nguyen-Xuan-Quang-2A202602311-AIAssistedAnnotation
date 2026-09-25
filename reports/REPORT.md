# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Xuân Quang

Công cụ gán nhãn đã dùng: CVAT Docker local

## 1. Dữ liệu và cách chia tập

**Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?**

Video được quay bằng camera cố định và lấy mẫu ở 2.5 frame mỗi giây, nên hai frame liên tiếp chỉ cách nhau 0.4 giây và thường chứa cùng những chiếc xe ở vị trí rất gần nhau. Vì vậy, pool và test được chia theo trục thời gian, đồng thời có vùng đệm quanh các đoạn test. Nếu chia ngẫu nhiên, cùng một chiếc xe hoặc gần như cùng một khung cảnh có thể xuất hiện ở cả train và test. Đây là rò rỉ dữ liệu, làm số đo test bị lệch lên và tạo cảm giác model tổng quát tốt hơn thực tế. Trong lab, 268 ảnh pool dùng để chọn mẫu; 20 ảnh test chỉ dùng để đánh giá và không được đưa vào train hoặc sửa nhãn.

## 2. Mô hình khởi đầu lạnh (cold start)

**Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?**

| vòng | model | ảnh train | box train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Theo `metrics_round0.json`, cold start đạt AP50 0.7714, precision 0.9249, recall 0.4888 và F1 0.6396 trên 20 ảnh test. Model có precision cao nhưng recall thấp: nó tương đối ít báo nhầm, song bỏ sót nhiều xe. Recall của xe nhỏ chỉ là 0.1818, thấp hơn rõ rệt so với xe vừa 0.5473 và xe lớn 0.5610, cho thấy xe xa, tối hoặc chỉ hiện thành cụm đèn là nhóm khó nhất.

Trong `compare_round0.jpg`, riêng `frame_0050` có 11 true positive, 2 false positive và 7 false negative; `frame_0350` có 9 true positive, 2 false positive và 14 false negative. Nhiều box bị bỏ sót nằm ở vùng xe nhỏ phía xa. Tuy nhiên, với một cụm sáng rất xa hoặc phản chiếu trên mặt đường, cần người rà lại nhãn tham chiếu trước khi kết luận model sai, vì nhãn test cũng do một model khác tạo và chưa được kiểm thủ công từng box.

## 3. Chiến lược chọn mẫu

**Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?**

Điểm chọn mẫu được tính theo `score = 0.5·U + 0.3·A + 0.2·D`. `U` là trung bình độ bất định của năm box khó nhất, cao nhất khi confidence gần 0.5. `A` biểu diễn số box mơ hồ có confidence từ 0.15 đến dưới 0.5 sau khi chuẩn hóa theo pool. `D` là độ xa về thời gian so với ảnh đã gán gần nhất, nhằm tăng tính đa dạng. Ở vòng đầu chưa có ảnh nào đã gán nên `D = 1.0` cho mọi ảnh. `MIN_GAP_S = 2.0` ngăn hai ảnh cách nhau dưới hai giây cùng vào một lô, vì chúng thường gần trùng khi camera đứng yên.

Ba ví dụ được chọn là `frame_0182.jpg` (hạng 1, score 0.9591, 18 box mơ hồ), `frame_0369.jpg` (hạng 2, score 0.9324, 43 box dự đoán) và `frame_0312.jpg` (hạng 7, score 0.9100, 18 box mơ hồ). Ngược lại, `frame_0372.jpg` có score 0.9101 và đứng hạng 6 nhưng không được chọn vì chỉ cách `frame_0369.jpg` 1.2 giây. Nếu chỉ đủ công rà năm ảnh, tôi còn loại `frame_0331.jpg` khỏi nhóm ưu tiên vì nó chỉ cách `frame_0326.jpg` đúng 2 giây và cảnh rất giống nhau. Điểm bất định cao chỉ cho thấy model đang phân vân; nó không chứng minh ảnh sẽ được gán nhãn hoàn hảo hoặc chắc chắn làm model tốt hơn sau fine-tune.

## 4. Các vòng học chủ động (active learning)

**Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:**

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

**Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.**


| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 261 | 0.638 | -0.133 | 0.941 | 0.196 | 0.324 | 0.000 | 0.182 | 0.610 |

Ở vòng 1, model đề xuất 169 box trên 12 ảnh. Sau khi rà bằng CVAT, bộ nhãn cuối có 261 box: 152 box được giữ gần như nguyên, 4 box được chỉnh, 13 box bị xóa và 105 box được thêm. `REVIEW_LOG.csv` ghi các ca cụ thể như thêm xe bị cắt trong `frame_0107.jpg`, xóa box trên vệt đèn trong `frame_0099.jpg` và thêm xe tối trong `frame_0182.jpg`; các hành động này phù hợp với diff tương ứng là 6 box thêm ở frame 0107, 1 box xóa ở frame 0099 và 9 box thêm ở frame 0182.

Sau fine-tune, AP50 giảm từ 0.7714 xuống 0.6382, tức giảm 0.1332. Recall giảm từ 0.4888 xuống 0.1960 và F1 giảm từ 0.6396 xuống 0.3244, dù precision tăng nhẹ từ 0.9249 lên 0.9405. Theo kích thước, recall của xe nhỏ giảm từ 0.1818 xuống 0.0000 và xe vừa giảm từ 0.5473 xuống 0.1824; chỉ xe lớn tăng từ 0.5610 lên 0.6098. Điều này cho thấy model sau fine-tune trở nên bảo thủ hơn: ít báo nhầm hơn nhưng bỏ sót rất nhiều xe nhỏ và vừa.

Ảnh so sánh cho thấy thay đổi rõ ở `frame_0050`: cold start có 11 true positive, 2 false positive và 7 false negative, còn vòng 1 chỉ có 3 true positive, 0 false positive và 16 false negative. Việc hết false positive không bù được số xe bị bỏ sót tăng mạnh. Tương tự, ở `frame_0350`, true positive giảm từ 9 xuống 5 và false negative tăng từ 14 lên 18. Đây là bằng chứng trực tiếp rằng vòng 1 không cải thiện độ phủ trên tập test.

Ba nguồn bằng chứng có vai trò khác nhau. Trong `BLIND_SCAN.md`, trước khi xem pre-label tôi đếm khoảng 23 xe ở `frame_0312` và chú ý các xe đứng sát nhau hoặc bị che một phần. Khi rà nhãn, diff của frame này cho thấy 13 pre-label được sửa thành 17 box cuối, gồm 10 accepted, 1 edited, 2 deleted và 6 added. Đây là lỗi nhãn gợi ý đã được sửa, còn kết quả sau train phải được đánh giá riêng bằng metrics và ảnh compare. Trường hợp xe bị che là ca khó: theo guideline chỉ vẽ phần nhìn thấy, không suy diễn toàn bộ thân xe; còn xe quá xa chỉ còn hai chấm đèn có thể gán hoặc bỏ nếu box cao dưới khoảng 16 pixel.

Một giả thuyết hợp lý cho kết quả giảm là 12 ảnh vẫn là một tập rất nhỏ và đều thuộc cùng một cảnh camera, trong khi có tới 105 box được thêm so với pre-label. Fine-tune 50 epoch trên tập nhỏ có thể làm model thích nghi quá mức với cách gán nhãn của lô này hoặc thay đổi ngưỡng phát hiện, dẫn đến giảm khả năng phát hiện xe nhỏ và vừa trên các đoạn test khác. Đây chỉ là giả thuyết cần kiểm bằng rà nhãn và thử nghiệm tiếp, không phải kết luận chắc chắn từ một vòng.

## 5. Kết luận và giới hạn

**Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?**

Tôi dừng sau vòng 1 để kiểm tra dữ liệu trước khi train thêm, vì AP50 giảm 0.1332 và recall giảm 0.2928. Hai nhóm cần ưu tiên nếu làm vòng tiếp theo là: (1) xe nhỏ hoặc tối ở xa, vì recall small hiện bằng 0; và (2) xe vừa bị che, đứng sát nhau hoặc cắt mép, vì recall medium giảm còn 0.1824 và đây là các ca dễ gộp box hoặc bỏ sót. Tôi sẽ chọn ảnh ở những đoạn thời gian khác nhau, tránh lấy nhiều frame gần trùng, vì rà một ảnh đông xe có thể cần thêm hoặc kiểm hàng chục box nhưng ảnh sát thời gian thường cung cấp ít thông tin mới.

Trước khi train tiếp, tôi sẽ rà lại toàn bộ 105 box được thêm, đặc biệt các box quanh cụm đèn xa, xe bị che và vệt phản chiếu; kiểm sự nhất quán của kích thước box; và đối chiếu những ảnh có số box thay đổi nhiều như `frame_0369.jpg`, nơi có 14 box được thêm. Tôi cũng sẽ xem lại một số test frame để phân biệt lỗi model với lỗi của nhãn tham chiếu, nhưng không sửa nhãn test.

Kết luận bị giới hạn bởi tập test chỉ có 20 ảnh, 14 box tham chiếu cao dưới 16 pixel bị bỏ qua, và toàn bộ nhãn tham chiếu do model tạo chứ chưa được người rà thủ công. Do đó, thay đổi của một số ít xe có thể ảnh hưởng đáng kể đến số đo, còn một box tham chiếu sai có thể làm dự đoán đúng bị tính thành lỗi. Kết quả vòng 1 là bằng chứng rằng cấu hình hiện tại giảm độ phủ trên bộ tham chiếu này; nó chưa đủ để kết luận model kém hơn trong mọi tình huống thực tế.
