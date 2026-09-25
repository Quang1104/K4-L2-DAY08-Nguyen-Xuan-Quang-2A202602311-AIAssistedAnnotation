# Vì sao chọn lô này?

**Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:**

Nếu chỉ có ngân sách rà năm ảnh trong 50 ứng viên đầu, tôi ưu tiên `frame_0182.jpg` (hạng 1, 72.8 s, score 0.9591), `frame_0369.jpg` (hạng 2, 147.6 s, score 0.9324), `frame_0380.jpg` (hạng 3, 152.0 s, score 0.9170), `frame_0326.jpg` (hạng 4, 130.4 s, score 0.9155) và `frame_0312.jpg` (hạng 7, 124.8 s, score 0.9100). Bốn ảnh đầu có độ bất định `U` từ 0.9182 đến 0.9340 và nhiều box mơ hồ. `frame_0312.jpg` có 18 box mơ hồ (`A = 1.0`), đồng thời ở bước quét độc lập tôi thấy nhiều xe đứng sát hoặc che nhau nên ảnh này đáng dành công rà. Tôi không ưu tiên đồng thời `frame_0331.jpg` dù ảnh này đứng hạng 5, vì thời điểm 132.4 s chỉ cách `frame_0326.jpg` đúng 2 giây và hai ảnh có nền cảnh rất giống nhau; với ngân sách chỉ năm ảnh, một ảnh ở thời điểm khác có giá trị đa dạng cao hơn.

**Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:**

Ba frame trong lô 12 ảnh cho thấy các thành phần điểm hoạt động khác nhau. `frame_0182.jpg` có score cao nhất 0.9591, 28 box dự đoán và 18 box mơ hồ; cả `U = 0.9182` và `A = 1.0` đều cho thấy model phân vân. `frame_0369.jpg` có score 0.9324, 43 box dự đoán, 16 box mơ hồ và `U = 0.9315`, nên chi phí rà cao nhưng có nhiều quyết định khó để kiểm tra. `frame_0312.jpg` có score 0.9100, 37 box dự đoán và 18 box mơ hồ; quan sát trực tiếp còn cho thấy trường hợp xe sát nhau và xe bị che một phần, phù hợp để kiểm tra lỗi gộp box hoặc bỏ sót.

**Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:**

`frame_0372.jpg` đứng hạng 6 với score 0.9101 nhưng không được chọn. Ảnh ở 148.8 s, chỉ cách `frame_0369.jpg` ở 147.6 s là 1.2 giây, nhỏ hơn `MIN_GAP_S = 2.0`; vì camera cố định nên hai frame gần như cùng một cảnh. Bỏ một ảnh gần trùng giúp tránh tốn công gán nhãn mà thu thêm ít thông tin.

**Điều phép chọn này chưa chứng minh về chất lượng mô hình:**

Điểm cao chỉ cho biết model đang bất định, có nhiều box mơ hồ hoặc ảnh có tính đa dạng theo thời gian. Nó không chứng minh ảnh đó chắc chắn được gán đúng, cũng không bảo đảm fine-tune bằng ảnh đó sẽ làm AP50 tăng. Kết quả vòng 1 thực tế giảm AP50, vì vậy quyết định chọn mẫu vẫn phải được kiểm bằng chất lượng nhãn và phép đánh giá trên cùng tập test.
