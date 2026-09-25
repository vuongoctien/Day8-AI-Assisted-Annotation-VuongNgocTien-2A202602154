# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Vương Ngọc Tiến

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian kèm vùng đệm (buffer) ở giữa thay vì chia ngẫu nhiên xuất phát từ đặc tính vật lý của dữ liệu video giám sát giao thông. Trong video được quay bởi camera đặt cố định trên cao tốc, một chiếc xe di chuyển trên đường sẽ xuất hiện liên tục qua nhiều khung hình kế tiếp trong khoảng vài giây đến vài chục giây. 

Nếu áp dụng phương pháp chia ngẫu nhiên (random split), các khung hình liền kề của cùng một chiếc xe sẽ bị phân tán vào cả tập học (train/pool) và tập kiểm thử (test). Khi đó, mô hình chỉ cần ghi nhớ chiếc xe và bối cảnh ở các frame liền kề là đã đạt kết quả cao trên tập test (hiện tượng rò rỉ dữ liệu - data leakage). Kết quả là số đo đánh giá (như AP50, F1) trên tập kiểm thử sẽ bị **lệch theo hướng quá lạc quan (overestimated)** so với hiệu năng thực tế. Việc chia theo trục thời gian và tạo vùng đệm thời gian cách ly đảm bảo tập test kiểm tra khả năng tổng quát hóa thực sự của mô hình đối với các luồng xe và thời điểm hoàn toàn mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `outputs/compare_round0.jpg` và các số đo độ phủ theo kích thước:
- Mô hình khởi đầu lạnh (YOLOv8n pretrained trên COCO) đạt độ chính xác Precision cao (0.925) nhưng Recall khá thấp (0.489). Mô hình gặp khó khăn rõ rệt đối với các loại xe ở xa có kích thước nhỏ (R small chỉ đạt 0.182, trong khi R medium là 0.547 và R large là 0.561). Điều này cho thấy mô hình bỏ sót rất nhiều xe ở xa dưới chân cầu hoặc xe mờ tối trong đêm. Ngoài ra, mô hình dễ nhầm các vệt sáng đèn phản chiếu trên mặt đường ướt thành phương tiện.
- Trường hợp cần người rà lại nhãn tham chiếu (ground truth): Nhãn kiểm thử hiện tại được sinh tự động bởi mô hình tham chiếu lớn hơn, chưa qua kiểm duyệt thủ công từng khung hình. Ví dụ, ở các vị trí xe rất xa chỉ có 2 đốm sáng mờ hoặc xe bị cắt sát mép ảnh, nhãn tham chiếu có thể đã bỏ qua hoặc định vị lệch; do đó khi mô hình dự đoán đúng một xe nhỏ ở xa nhưng nhãn test không có, mô hình lại bị tính là False Positive. Cần có chuyên gia rà lại các trường hợp này trước khi vội kết luận mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Chiến lược chọn mẫu sử dụng công thức tính điểm tổng hợp:
`score = W_U * U + W_A * A + W_D * D`
với trọng số tương ứng: 50% cho độ bất định (Uncertainty $U$), 30% cho số lượng bounding box lưỡng lự (Ambiguity $A$) và 20% cho tính đa dạng thời gian (Diversity $D$).
- **Vai trò của `MIN_GAP_S = 2.0s`**: Do camera cố định, các frame chụp cách nhau dưới 2 giây có bối cảnh gần như trùng khít. Tham số `MIN_GAP_S` đóng vai trò lọc khử trùng lặp (non-maximum suppression theo thời gian), ngăn việc đưa các ảnh gần giống nhau vào cùng một lô gán nhãn, từ đó tối ưu hóa ngân sách và công sức của người gán nhãn.
- **Minh chứng từ các frame**:
  1. `frame_0182.jpg` (hạng 1, score 0.9591) và `frame_0099.jpg` (hạng 8, score 0.9063, U = 0.9460): Được chọn vào lô 12 ảnh vì có độ bất định cao và chứa nhiều đối tượng gây phân vân.
  2. `frame_0107.jpg` (hạng 14, score 0.8876): Được chọn vào lô vì đại diện cho phân đoạn thời gian có nhiều vệt sáng phức tạp.
  3. `frame_0372.jpg` (hạng 6, score 0.9101, t = 148.8s): Dù điểm rất cao nhưng bị loại bỏ vì chỉ cách `frame_0369.jpg` (hạng 2, t = 147.6s) 1.2 giây (< 2.0s).
- **Điểm bất định và chất lượng mô hình**: Điểm bất định cao chỉ phản ánh việc mô hình hiện tại đang phân vân hoặc chưa tự tin về dự đoán của nó trên khung hình đó. Nó **không chứng minh** rằng việc gán nhãn frame đó chắc chắn sẽ nâng cao chất lượng mô hình trên tập test, bởi điểm bất định cao có thể do ảnh bị nhiễu ngoại lai (outliers), vệt lóa ánh sáng hoặc bối cảnh không mang tính đại diện cho tập kiểm thử.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp kết quả các vòng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 350 | 0.484 | -0.288 | 1.000 | 0.072 | 0.134 | 0.000 | 0.047 | 0.366 |

- **Thống kê sửa nhãn vòng 1** (từ `outputs/round1_diff.md`): Trên 12 ảnh gán nhãn, mô hình đề xuất ban đầu 169 box, sau khi rà soát và chỉnh sửa trên CVAT thu được 350 box chuẩn. Cụ thể: 131 box giữ nguyên (accepted, tỷ lệ 78%), 24 box được kéo chỉnh biên (edited), 14 box bị xóa do nhận diện nhầm vệt đèn (deleted - False Positive của pre-label), và 195 box được vẽ thêm cho các xe bị sót (added - False Negative của pre-label).
- **Biến động số đo**: Sau vòng 1 fine-tune với 12 ảnh (350 box), AP50 đạt 0.484 (giảm 0.288 so với cold start). Precision đạt tuyệt đối 1.000 nhưng Recall giảm mạnh xuống 0.072 (R small về 0.000, R medium là 0.047, R large là 0.366).
- **Phân tích kết quả sau fine-tune**: Việc fine-tune trên tập dữ liệu nhỏ (12 ảnh) với số lượng box được bổ sung rất dày đặc (350 box) khiến mô hình trở nên quá thận trọng, độ tự tin dự đoán bị kéo xuống thấp, dẫn đến chỉ phát hiện được các xe lớn ở cự ly gần và bỏ qua hầu hết các xe nhỏ.
- **Phân biệt ba góc nhìn dữ liệu**:
  1. *Quan sát độc lập (`BLIND_SCAN.md`)*: Khi nhìn ảnh `frame_0107.jpg` nguyên bản chưa có pre-label, mắt người đếm được 25 xe, chỉ ra các vùng khó ở giữa dưới (xe cắt mép) và trên trái (xe xa kẹp giữa làn).
  2. *Lỗi pre-label đã sửa (`REVIEW_LOG.csv` & `round1_diff.md`)*: Thực tế pre-label của AI ở `frame_0107.jpg` chỉ đưa ra 13 box; người gắn nhãn đã xóa 3 box vệt đèn phản chiếu, chỉnh sửa 2 box và thêm 17 box xe còn sót.
  3. *Kết quả sau train*: Mô hình sau fine-tune loại bỏ hoàn toàn các lỗi cảnh báo nhầm (FP = 0, Precision = 1.0) nhưng bị suy giảm độ phủ trên tập test.
- **Ca khó theo guideline**: Xe bị cắt mép ở góc dưới ảnh hoặc xe ở rất xa chỉ nhìn thấy 2 chấm sáng đèn hậu; theo quy tắc chỉ khoanh phần thân xe thực sự nhìn thấy trong khung hình và không khoanh vệt sáng chiếu trên mặt đường.

## 5. Kết luận và giới hạn

- **Đánh giá kết quả & Quyết định**: Điểm AP50 ở vòng 1 giảm so với cold start (0.484 so với 0.771). Dù Precision tăng lên 1.000, sự sụt giảm của Recall cho thấy mô hình đang bị thiên lệch dự đoán sau khi học trên 12 ảnh đầu tiên. Tuy nhiên, không nên dừng lại mà cần **tiếp tục vòng học chủ động tiếp theo (vòng 2)** với các điều chỉnh: mở rộng tập train, cân bằng trọng số loss và điều chỉnh lại ngưỡng confidence threshold để khôi phục Recall.
- **Đề xuất hai ca còn yếu cho vòng sau**:
  1. *Xe nhỏ ở xa*: Cần chọn các frame có mật độ xe dày đặc ở hậu cảnh để mô hình học lại đặc trưng xe nhỏ.
  2. *Xe trong vùng tối mép đường hoặc xe bị che khuất một phần*: Bổ sung mẫu gán nhãn đa dạng để cải thiện khả năng phân tách xe với nền tối.
  - Chi phí rà nhãn: Các ảnh có mật độ cao đòi hỏi thêm nhiều thời gian (vẽ 25-40 box/ảnh). Cần duy trì ràng buộc `MIN_GAP_S >= 2.0s` để tránh chọn các frame gần trùng gây lãng phí.
- **Giới hạn của tập kiểm thử**: Tập test chỉ gồm 20 ảnh, quy tắc đánh giá lọc bỏ các box có chiều cao dưới 16 px, và nhãn tham chiếu chưa được kiểm duyệt thủ công 100%. Những giới hạn này có thể làm phóng đại mức độ biến động của các chỉ số (AP50, Recall) khi kích thước mẫu đánh giá tương đối nhỏ.
- **Hành động kiểm tra khi AP50 giảm**: Trước khi cho mô hình học thêm, cần:
  1. Kiểm tra lại toàn bộ file nhãn trong `labels/round1/` xem tọa độ box có bị lệch hay chuẩn hóa sai không.
  2. So sánh phân phối kích thước box giữa tập train và test.
  3. Rà soát lại siêu tham số huấn luyện (learning rate, số epoch fine-tune, augmentations) để tránh overfitting vào 12 ảnh ban đầu.
