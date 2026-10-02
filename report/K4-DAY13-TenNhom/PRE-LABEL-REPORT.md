# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: **labeler**
- Thành viên: xem `TEAMMATES.md` (Nguyễn Tiến Sỹ - 2A202602308, Dư Văn Sang - 2A202602330, Nguyễn Hữu Sơn - 2A202602253).
- Trạng thái: `executed-by-group` (môi trường Docker CPU).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Tiến Sỹ; 02/10/2026; Windows x86_64 / Docker Desktop Linux containers.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lab` / Image ID: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`; repo revision: `0831856`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: KITTI demo `000008` (17.238 điểm, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`); chạy local trong phiên thực hành.
- Checkpoint: PointPillars KITTI có sẵn trong image (`/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: `front-window`; score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Dữ liệu PCD gốc đã lược bỏ reflectance thật, script sử dụng kênh cường độ hằng số theo class adapter (0.0 cho vehicles, 0.7 cho pedestrian/two-wheels); `z_ground` được ước lượng cục bộ từ phân bố độ cao đám mây điểm.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | -1.25m | `boxes-demo-delta-0-voxel-0.16.json` | Chỉ phát hiện 1 hộp (`vehicles`). Khi `delta = 0`, mô hình không bù trừ giả định chiều cao sensor 1.73m của KITTI, làm tọa độ z của đám mây điểm bị lệch khỏi khoảng neo (anchor range) huấn luyện $\rightarrow$ bỏ sót gần như toàn bộ xe và người. |
| B | 1.73 | 0.16 | 13 | 0.42m | `boxes-demo-delta-1.73-voxel-0.16.json` | Phát hiện 13 hộp (10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`). Đây là cấu hình chuẩn (baseline): các hộp bao khít cụm điểm, đáy hộp bám sát mặt đường cục bộ trên ảnh Side view, phân bố class đa dạng. |
| C | 1.73 | 0.32 | 6 | 0.38m | `boxes-demo-delta-1.73-voxel-0.32.json` | Phát hiện 6 hộp (toàn bộ là `pedestrian`, mất hết `vehicles` và `two-wheels`). Việc tăng kích thước pillar gấp đôi (0.16m $\rightarrow$ 0.32m) làm giảm độ phân giải không gian XY, gộp thô các điểm khiến đặc trưng hình học của các vật thể lớn bị méo mó. |

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**  
  *Khác hoàn toàn về mặt bản chất.* Phép dịch input trước model ($z_{model} = z_{source} - z_{ground} - \delta$) làm thay đổi vị trí tương đối của từng điểm laser rơi vào các pillar/voxel trong không gian 3D. Điều này làm thay đổi toàn bộ phân bố voxel và đặc trưng trích xuất qua mạng nơ-ron tích chập (PointPillars backbone), dẫn đến mô hình thay đổi cả số lượng hộp phát hiện (A chỉ ra 1 hộp, B ra 13 hộp), độ tin cậy (confidence score) và phân bố nhãn. Ngược lại, nếu chỉ dịch một hằng số cho các hộp sau inference thì đó chỉ là phép tịnh tiến tọa độ thuần túy hình học, số lượng hộp và score của các hộp hoàn toàn giữ nguyên.

- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**  
  Khi tăng kích thước pillar XY từ 0.16m lên 0.32m (giữ nguyên $\delta = 1.73m$), số lượng hộp giảm mạnh từ 13 xuống còn 6 hộp, đồng thời nhãn bị chuyển dịch hoàn toàn sang `pedestrian` (mất hết xe 4 bánh và xe 2 bánh). Không đủ bằng chứng để khẳng định cấu hình nào "tốt hơn" chỉ dựa vào số lượng hộp hoặc score; đây là sự đánh đổi kinh điển giữa độ phân giải không gian (spatial resolution) và chi phí tính toán trong thiết kế mạng LiDAR 3D.

- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**  
  Mô hình chỉ inference trên cửa sổ phía trước (`front-window`), do đó các vật thể nằm ngoài vùng ROI (như phía sau xe hoặc quá xa hai bên) không được coi là lỗi bỏ sót (miss). Ảnh `Side view` là hình chiếu trực giao toàn cảnh lên mặt phẳng X-Z, làm nén trục Y $\rightarrow$ các vật thể ở các làn đường khác nhau bị chồng lấn lên nhau trên cùng một hình chiếu. Do đó, ảnh Side chỉ dùng để kiểm tra độ cao và mặt đường, không thể dùng một mình nó để xác định hướng đầu xe (yaw) hoặc hình học chi tiết của từng đối tượng.

- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**  
  Cả 3 file JSON đều chưa đủ cơ sở để coi là nhãn chuẩn (ground truth) để import trực tiếp mà không qua kiểm tra. Đặc biệt JSON của A và C có sai lệch nghiêm trọng về số lượng và phân loại. Với JSON lượt B (baseline), cần tiếp tục đối chiếu ảnh camera để xác định lại hướng mũi xe (tránh quay ngược 180°), rà soát các vùng che khuất để không bị co cụm kích thước hộp, và tự bổ sung các class mà mô hình không được huấn luyện (như `Animal` và `Obstacle`).

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| `case-correct` | 0 / 13 | 0 m | Không đổi | Không có lỗi batch; tiếp tục kiểm tra từng đối tượng | 13/13 hộp giữ nguyên độ cao và hình học của lượt B; đáy hộp bám khít mặt đường trên ảnh Side view. |
| `case-batch-z` | 13 / 13 | Lệch đồng loạt ~ -1.73m | Không đổi class/x/y/yaw, chỉ lệch trục z | **DỪNG BATCH**, báo LC kiểm tra pipeline/hệ tọa độ | Toàn bộ 13/13 hộp đều bị chìm/lơ lửng cùng một độ lệch z giống hệt nhau. Đây là lỗi chuyển đổi hệ tọa độ có hệ thống (systematic pipeline error), không được sửa tay từng hộp mà phải sửa code pipeline để tạo lại prediction. |
| `case-one-box-z` | 1 / 13 | Chỉ 1 hộp lệch ~ -1.73m | 12 hộp còn lại hoàn toàn bình thường | **Kiểm tra từng hộp**, sửa lỗi cá thể | 12 hộp còn lại vẫn bám mặt đường chuẩn xác, chỉ có 1 hộp đơn lẻ bị sai đáy. Đây là lỗi cá thể của detector, người gán nhãn có thể dùng góc nhìn Bên và Top để chỉnh lại vị trí và đáy hộp. |

*Ghi rõ:* Helper tạo biến đổi có chủ đích từ prediction của lượt B nhằm mục đích huấn luyện nhận biết lỗi, không phải là kết quả inference riêng biệt hay nhãn ground truth.

## Nhận xét cá nhân

### 1. Thành viên: Nguyễn Tiến Sỹ (MSSV: 2A202602308)
- **Vai trò đã làm:** Vận hành lệnh Docker (Lượt A), Kiểm tra hình học & Side view (Lượt B), Kiểm tra JSON/CSV (Lượt C).
- **Quan sát thực tế:** Khi chạy lượt A với $\delta = 0$, em nhận thấy trong file `summary.csv` chỉ ghi nhận đúng 1 hộp xe, trong khi mở ảnh `side-run-A.png` các cụm điểm thực tế của xe và người đều không có bounding box bao quanh. Sang lượt B với $\delta = 1.73$, các hộp xuất hiện đầy đủ 13 hộp và bám khít mặt đất.
- **Diễn giải phép biến đổi z:** Phép đổi z tuân theo công thức $z_{model} = z_{source} - z_{ground} - \delta$ và $z_{source} = z_{model} + z_{ground} + \delta$. Việc thiết lập đúng $\delta = 1.73m$ giúp đưa gốc tọa độ của điểm về đúng vị trí sensor mà mô hình KITTI đã học, đảm bảo điểm rơi đúng vào các anchor box 3D.
- **Quyết định lỗi batch và hành động:** Khi gặp trường hợp `case-batch-z` (tất cả các hộp đều bị lệch z cùng một giá trị), em quyết định **dừng sửa tay ngay lập tức** và báo cáo kỹ sư pipeline để kiểm tra lại ma trận biến đổi tọa độ; không tốn công chỉnh sửa thủ công từng hộp khi lỗi nằm ở cả hệ thống.
- **Điều chưa chắc chắn:** Khi quan sát các đối tượng ở xa trên ảnh Side view, các điểm LiDAR phản xạ rất thưa (sparse) khiến em khó xác định chính xác ranh giới đuôi xe nếu không có thêm thông tin từ ảnh camera.

### 2. Thành viên: Dư Văn Sang (MSSV: 2A202602330)
- **Vai trò đã làm:** Kiểm tra JSON/CSV (Lượt A), Vận hành lệnh Docker (Lượt B), Ghi log & Biên soạn báo cáo (Lượt C).
- **Quan sát thực tế:** Khi đối chiếu dữ liệu giữa Lượt B và Lượt C, em quan sát thấy việc tăng kích thước pillar từ 0.16m lên 0.32m làm file `boxes-run-C.json` mất hoàn toàn các nhãn `vehicles` (10 hộp) và `two-wheels` (1 hộp), chỉ còn lại 6 nhãn `pedestrian`. Điều này chứng minh kích thước voxel ảnh hưởng trực tiếp đến khả năng phân giải đặc trưng của mạng nơ-ron.
- **Diễn giải phép biến đổi z:** Trục z của PointPillars KITTI trả về tọa độ đáy hộp ($z_{bottom}$), trong khi nhãn của hệ thống nguồn Robotaxi yêu cầu tọa độ tâm hộp ($z_{center} = z_{bottom} + \frac{h}{2}$). Script đã tự động thực hiện phép chuyển đổi này, do đó khi đọc file JSON học viên không được tự ý dịch thêm lần nữa.
- **Quyết định lỗi batch và hành động:** Đối với `case-one-box-z`, vì 12 hộp còn lại đều đúng độ cao mặt đất nên em xác định đây không phải lỗi pipeline mà là lỗi nhận diện cục bộ của mô hình trên vật thể bị che khuất; hành động đúng là giữ nguyên batch và dùng công cụ CVAT để chỉnh lại đáy hộp đơn lẻ đó.
- **Điều chưa chắc chắn:** Đối với class `two-wheels` (xe hai bánh kèm người lái), trong biểu diễn đám mây điểm thưa rất dễ nhầm lẫn với một người đi bộ `pedestrian`, cần phải xoay góc 3D tự do và kiểm tra kỹ ảnh camera.

### 3. Thành viên: Nguyễn Hữu Sơn (MSSV: 2A202602253)
- **Vai trò đã làm:** Kiểm tra hình học & Ghi log (Lượt A), Ghi log & Hỗ trợ báo cáo (Lượt B), Vận hành lệnh Docker (Lượt C).
- **Quan sát thực tế:** Trên ảnh `side-run-B.png`, đường tham chiếu $z = 0$ không phải lúc nào cũng trùng khớp với mặt đường thực tế của từng vật thể do địa hình có độ dốc cục bộ. Tuy nhiên, các hộp của Lượt B đều có đáy bám sát các điểm mặt đường lân cận của từng xe.
- **Diễn giải phép biến đổi z:** Dịch input trước khi đưa vào mô hình làm thay đổi tập điểm đầu vào của các trụ (pillars), khiến mô hình trích xuất ra tensor đặc trưng hoàn toàn khác; nó khác hoàn toàn với việc cộng/trừ một lượng z cố định vào kết quả bounding box sau khi mô hình đã tính toán xong.
- **Quyết định lỗi batch và hành động:** Khi thấy toàn bộ các hộp đều bay lơ lửng trên không trung hoặc lún xuống lòng đường (lỗi cả batch), hành động duy nhất có ý nghĩa kỹ thuật là yêu cầu kiểm tra lại thông số `sensor_height` ($\delta$) và phép ước lượng `z_ground`, tuyệt đối không cố gắng kéo từng hộp xuống bằng tay.
- **Điều chưa chắc chắn:** Khả năng nhận diện hướng đầu xe (yaw): trong đám mây điểm LiDAR nhìn từ trên xuống, hình hộp chữ nhật có tính đối xứng cao, rất khó khẳng định đầu xe quay về hướng nào nếu chỉ nhìn vào đám mây điểm mà không đối chiếu với ảnh camera phía trước.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: