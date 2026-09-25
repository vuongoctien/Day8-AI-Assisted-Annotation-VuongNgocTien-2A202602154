# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét ảnh gần trùng hoặc trường hợp model không dự đoán được box:
Nếu chỉ được sửa 5 ảnh, tôi ưu tiên chọn 5 frame: `frame_0182.jpg` (hạng 1, t = 72.8s, score = 0.9591), `frame_0369.jpg` (hạng 2, t = 147.6s, score = 0.9324), `frame_0380.jpg` (hạng 3, t = 152.0s, score = 0.9170), `frame_0326.jpg` (hạng 4, t = 130.4s, score = 0.9155) và `frame_0331.jpg` (hạng 5, t = 132.4s, score = 0.9154). Đây là 5 ảnh có điểm tổng hợp cao nhất và phân bố ở các khoảng thời gian khác nhau. Đặc biệt, xét trường hợp ảnh gần trùng: `frame_0182.jpg` (t = 72.8s) và `frame_0187.jpg` (hạng 10, t = 74.8s) cách nhau đúng 2.0s, cảnh giao thông có sự tương đồng nhất định nên nếu ngân sách chỉ có 5 ảnh, tôi ưu tiên chọn `frame_0182.jpg` đứng đầu và dành các suất còn lại cho các cụm thời gian khác (`0326`, `0331`, `0369`, `0380`) để tối đa hóa tính đa dạng của dữ liệu.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
Ba frame tiêu biểu được chọn gồm:
1. `frame_0182.jpg` (hạng 1, score 0.9591, U = 0.9182, n_ambiguous = 18): Mô hình có độ bất định rất cao đối với các xe ở làn xa và vùng tối bên trái.
2. `frame_0099.jpg` (hạng 8, t = 39.6s, score 0.9063, U = 0.9460, n_ambiguous = 14): Độ bất định cao nhất trong nhóm đầu, có nhiều xe bị cắt mép ở góc dưới và xe mờ xa dưới chân cầu.
3. `frame_0107.jpg` (hạng 14, t = 42.8s, score 0.8876, U = 0.8752, n_ambiguous = 15): Chứa nhiều vệt sáng phản chiếu gây nhiễu và nhiều xe nhỏ kẹp giữa các làn xe chạy.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
Frame `frame_0372.jpg` có thứ hạng rất cao (hạng 6, score = 0.9101, t = 148.8s) cao hơn nhiều ảnh trong lô được chọn như `frame_0312` hay `frame_0099`, nhưng đã bị thuật toán loại bỏ (`selected = False`). Lý do là vì `frame_0372.jpg` chỉ cách `frame_0369.jpg` (hạng 2, t = 147.6s) một khoảng 1.2 giây (nhỏ hơn ngưỡng `MIN_GAP_S = 2.0s`). Do camera đặt tĩnh, hai khung hình cách nhau 1.2s gần như phản chiếu cùng một phân bố xe, việc gán nhãn cả hai sẽ gây lãng phí chi phí nhân công mà không mang lại thêm nhiều giá trị thông tin mới cho mô hình.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
Điểm số cao từ chiến lược chọn mẫu chỉ phản ánh mức độ phân vân, bất định (uncertainty) và số lượng khung hình lưỡng lự (ambiguity) của mô hình hiện tại trên các frame đó. Điểm cao hoàn toàn chưa chứng minh rằng việc gán nhãn và huấn luyện bổ sung các frame này chắc chắn sẽ giúp mô hình tăng điểm AP50 hay cải thiện khả năng tổng quát hóa trên tập kiểm thử độc lập (test set).
