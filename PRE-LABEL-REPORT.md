# Báo cáo thực hành PointPillars và gán nhãn 3D — Day 13

Trong báo cáo này, tôi trình bày phần thực hành PointPillars và quá trình gán nhãn 3D do tôi tự thực hiện.

## Thông tin người thực hiện

- **Họ và tên:** Hoàng Văn Long
- **MSSV:** 2A202602071
- **Hình thức:** Thực hiện cá nhân, không có thành viên cùng nhóm hỗ trợ.
- **Phần gán nhãn Robotaxi:** Tôi tự thực hiện toàn bộ phần gán nhãn được giao.
- **Ngày lập báo cáo:** 02/10/2026.

## Thông tin dữ liệu và môi trường thực hành

- **Mã nhóm/phòng:** Cá nhân — Hoàng Văn Long.
- **Thành viên:** Hoàng Văn Long — MSSV 2A202602071.
- **Trạng thái phần pre-label:** `provided-results`. Tôi sử dụng số liệu công khai trong `bundle/VALIDATION.md` để phân tích, không tự nhận là đã chạy inference và không dùng dữ liệu Robotaxi riêng tư làm minh chứng.
- **Ngày và môi trường kiểm chứng:** File validation ghi nhận ngày thử 01/10/2026 trên ThinkPad Linux `amd64` và một máy Mac Apple Silicon `arm64`.
- **Image:** Hai gói kiểm thử dùng image CPU native được build từ Dockerfile và cùng checkpoint. Mỗi kiến trúc có image ID riêng, nhưng đó không phải image ID của một phiên inference cá nhân của tôi.
- **Phiên bản mã nguồn:** Gói được đóng từ base Student revision `0831856`, với `working_tree_dirty: true` và hash từng file trong manifest. Đây không phải clean release build. PointPillars upstream dùng commit `620e6b0d07e4cb37b7b0114f26b934e8be92a0ba`.
- **PCD và frame:** `data/demo.pcd`, frame `demo`, chuyển từ mẫu KITTI/MMDetection3D `000008`, gồm 17.238 điểm. SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- **Quyền sử dụng:** CC BY-NC-SA 3.0, chỉ dùng học thuật phi thương mại. Output KITTI không được nhập vào job Robotaxi.
- **Checkpoint:** PointPillars KITTI tại `/opt/PointPillars/pretrained/epoch_160.pth`. Cả hai image có cùng SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- **Phạm vi:** `front-window`; score threshold `0,3`; giữ cùng checkpoint, ROI và score cho cả ba lượt.
- **Kênh thứ tư và z_ground:** Reflectance gốc đã bị bỏ, trường `rgb = 0` là placeholder và adapter dùng kênh hằng. `z_ground` được ước lượng từ PCD; giá trị của PCD đi kèm là khoảng `0,075 m`.

## Số liệu kiểm chứng tôi sử dụng

Tôi sử dụng số liệu trong `bundle/VALIDATION.md` để tham chiếu, không nhận đây là kết quả inference do tôi trực tiếp chạy. Gói được kiểm thử ngày 01/10/2026 trên ThinkPad Linux `amd64` và một máy Mac Apple Silicon `arm64`. Cả hai máy đều chạy thật ba lượt A/B/C và ba ca QC; `smoke.json` đều có trạng thái `passed`.

Trong lúc chạy, container được giới hạn tối đa 4 CPU và 4 GB RAM. Đây là giới hạn tài nguyên của lần thử, không phải yêu cầu RAM tối thiểu của laptop và cũng không phải số liệu peak RAM.

| Bước | ThinkPad Linux amd64 (giây) | Mac Apple Silicon arm64 (giây) |
| --- | ---: | ---: |
| `docker-load` | 6,92 | 42,83 |
| `run-A` | 4,00 | 71,74 |
| `run-B` | 4,12 | 84,50 |
| `run-C` | 2,38 | 55,20 |
| `qc-cases` | 1,13 | 10,73 |

Các giá trị trên là thời gian wall, đã gồm thời gian khởi động container; bước `docker-load` được đo riêng. Số liệu không gồm thời gian tải ZIP, cài Docker/Python hoặc build image. ThinkPad chạy nhanh hơn máy Mac trong lần kiểm chứng này, nhưng tôi không dùng thời gian chạy để đánh giá độ chính xác của prediction.

Trên cả hai kiến trúc, lượt A cho 1 hộp, lượt B cho 13 hộp và lượt C cho 6 hộp. Lượt B gồm 10 `vehicles`, 2 `pedestrian` và 1 `two-wheels`; lượt C gồm 6 `pedestrian`. Tôi chỉ xem đây là quan sát thực nghiệm trên PCD demo, không xem số hộp là chỉ số accuracy hoặc căn cứ duy nhất để chọn cấu hình. Prediction hash giữa hai kiến trúc khác nhau nên tôi cũng không yêu cầu kết quả phải trùng từng bit.

Ngoài phần chạy model, gói có 25 unit test cho bundle/converter và 5 test cho helper đều pass. Đây là kiểm thử cục bộ vì dự án không cấu hình CI, do đó tôi không ghi kết quả này là “CI green”. Windows chưa được thử trên máy thật; Apple Silicon mới được thử trên một máy nên các kết quả trên không phải cam kết cho mọi máy.

## Ba lượt inference và kết quả tham chiếu

Tôi đối chiếu ba lượt A, B và C trong `bundle/VALIDATION.md`. Cả ba lượt dùng cùng PCD, checkpoint, ngưỡng score và ROI; A/B chỉ khác `delta`, còn B/C chỉ khác kích thước pillar. Do tôi không có thư mục output của lần chạy cá nhân, các số hộp dưới đây là số liệu tôi đọc từ file validation.

| Lượt | delta (m) | Pillar XY (m) | Số hộp | Phân bố class | mean_z | Nguồn số liệu |
| --- | ---: | ---: | ---: | --- | --- | --- |
| A | 0 | 0,16 | 1 | File validation không nêu | File validation không nêu | `bundle/VALIDATION.md` |
| B | 1,73 | 0,16 | 13 | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels` | File validation không nêu | `bundle/VALIDATION.md` |
| C | 1,73 | 0,32 | 6 | 6 `pedestrian` | File validation không nêu | `bundle/VALIDATION.md` |

### So sánh A và B

Khi `delta` tăng từ `0 m` lên `1,73 m`, số hộp tăng từ 1 lên 13. Qua kết quả này, tôi nhận thấy mô hình đã chạy lại trên tập điểm sau khi biến đổi theo trục z, chứ không chỉ dịch các hộp cũ. Vì chưa có ảnh Side và JSON của lần chạy cá nhân, tôi chưa đủ cơ sở để nhận xét hộp nào tốt hơn.

### So sánh B và C

Khi giữ `delta = 1,73 m` và tăng kích thước pillar từ `0,16 m` lên `0,32 m`, số hộp giảm từ 13 xuống 6. Ở lượt B có 10 `vehicles`, 2 `pedestrian` và 1 `two-wheels`; lượt C chỉ còn 6 `pedestrian`. Theo tôi, chưa thể kết luận cấu hình nào tốt hơn chỉ dựa vào số hộp vì đây không phải chỉ số accuracy. Tôi cần xem thêm hình học, ảnh Side, JSON và reference đã duyệt.

### Những giới hạn tôi lưu ý khi đọc kết quả

- Tôi chỉ đánh giá vùng phía trước trong front-window; vật nằm ngoài ROI không thể dùng để kết luận model bỏ sót trong phép so sánh này.
- Tôi không dùng riêng ảnh Side để chốt lỗi thiếu hộp hoặc yaw vì hình chiếu `x-z` làm chồng các vật có tọa độ `y` khác nhau.
- Tôi không xem prediction KITTI là ground truth Robotaxi. Các JSON của bài demo khác frame và khác dữ liệu nên không được import vào CVAT Robotaxi.
- Khi kiểm một cuboid, tôi phối hợp góc Trên, Bên, Trước, góc xoay tự do và ảnh camera cùng frame.

### Phép biến đổi theo trục z

```text
z_model  = z_source - z_ground - delta
z_source = z_model  + z_ground + delta
```

`delta` được áp dụng trước inference sau khi trừ mặt đường ước lượng. Khi đưa hộp về hệ nguồn, pipeline phải cộng lại cả `z_ground` và `delta`. Vì model chạy lại trên tập điểm đã biến đổi, thay `delta` có thể làm thay đổi số hộp, class, vị trí và score.

## Ca QC có kiểm soát — không import CVAT

Bảng dưới phân tích helper `practice/pipeline-qc-cases.py` dựa trên lượt B tham chiếu. Gọi `N = 13`, `z_ground = 0,075 m` và `offset = delta + z_ground = 1,805 m`.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| `case-correct` | 0/13 | 0 m | Không | Kiểm từng hộp; không coi prediction là nhãn đúng | Bản sao prediction B trong hệ tọa độ nguồn |
| `case-batch-z` | 13/13 | Mỗi hộp bị hạ 1,805 m | Không | Dừng batch, kiểm transform và chạy lại pipeline; không sửa tay hàng loạt | Helper trừ `delta + z_ground` khỏi mọi hộp |
| `case-one-box-z` | 1/13 | Hộp đầu tiên bị hạ 1,805 m | Không | Kiểm riêng đối tượng bằng nhiều góc nhìn; chưa quy lỗi cho toàn batch | Helper chỉ sửa trường `z` của hộp đầu tiên |

Ba ca trên là biến đổi có chủ đích từ prediction của B. Chúng không phải ba lượt inference mới, không phải ground truth và không được dùng làm annotation cho CVAT.

## Nhận xét cá nhân

### Hoàng Văn Long — MSSV 2A202602071

Tôi thực hiện phần gán nhãn Robotaxi một mình. Không có thành viên khác cùng chỉnh cuboid hoặc chia sẻ khối lượng công việc. Dữ liệu nguồn, ảnh camera, job ID và snapshot được giữ trong CVAT/portal của ca học nên không đưa vào repo này.

Tôi hiểu rằng đổi `delta` trước inference khác với dịch hộp sau inference. Nếu toàn bộ hộp cùng lệch z một lượng gần bằng `z_ground + delta`, tôi sẽ dừng batch để kiểm phép đổi hệ tọa độ và chạy lại pipeline. Nếu chỉ một hộp lệch, tôi sẽ kiểm riêng đối tượng qua nhiều góc nhìn trước khi sửa. Điều tôi chưa thể bổ sung từ workspace là `mean_z`, image ID, thời điểm chạy và quan sát theo vùng từ ảnh Side/JSON của lần inference cá nhân; các giá trị này không được lấy từ kết quả validation để nhận là output của tôi.

## Báo cáo phần gán nhãn cá nhân

### Phạm vi và taxonomy

Tôi rà cả năm class của schema. PointPillars chỉ tạo pre-label cho ba class đầu nên hai class còn lại vẫn phải được tìm thủ công.

| Class | Đối tượng chính | Cách kiểm |
| --- | --- | --- |
| `vehicles` | Ô tô, van, xe tải, xe buýt | Bao đúng thân xe, không co xe dài thành hộp sedan theo cụm điểm gần |
| `two-wheels` | Xe máy, xe đạp | Đối chiếu camera để tránh nhầm với người hoặc xe bốn bánh |
| `pedestrian` | Người | Tách từng người, không gộp nhiều người vào một cuboid |
| `Animal` | Động vật | Chỉ thêm khi có đủ bằng chứng vì model không tạo pre-label cho class này |
| `Obstacle` | Vật cản thuộc taxonomy | Không gán tùy ý cây, cột hoặc mặt đường vào class này |

### Quy trình thực hiện

1. Tôi nạp pre-label đúng job từ portal và mở bài nguồn trong CVAT. Pre-label được xem là gợi ý khởi đầu, không phải đáp án.
2. Tôi rà lần lượt toàn bộ vùng đã khai, kiểm hộp có sẵn, hộp thiếu, hộp thừa và hộp trùng; đồng thời cân nhắc cả `Animal` và `Obstacle` dù model không dự đoán hai class này.
3. Với từng cuboid, tôi kiểm class, tâm `(x, y, z)`, chiều dài, chiều rộng, chiều cao, yaw và đáy hộp qua nhiều góc nhìn. Ảnh camera hỗ trợ nhận diện class và hướng; PCD được dùng để kiểm vị trí và hình học 3D.
4. Tôi đặt đáy hộp theo mặt đường cục bộ quanh đối tượng, không mặc định `z = 0` cho toàn frame. Với điểm thưa hoặc vật bị che, tôi không co hộp sát vài điểm và không dựng thêm hộp khi chưa đủ cơ sở.
5. Sau khi chỉnh, tôi Save trong CVAT trước khi thực hiện bước nộp tương ứng trên portal. Tôi chỉ khai “toàn frame” khi đã rà cả hộp thiếu và hộp thừa trên toàn frame.

### Tự kiểm chất lượng

- Mỗi đối tượng xác định được có đúng một cuboid và class giải thích được bằng PCD cùng ảnh camera.
- Tâm và kích thước hộp không lệch rõ, không ôm nền hoặc cụm điểm của đối tượng bên cạnh.
- Chiều dài, chiều rộng và yaw được kiểm cùng nhau; không hoán đổi dài/rộng rồi giữ nguyên hướng.
- Hướng đầu xe được đối chiếu trên ảnh khi có thể để tránh lỗi yaw 180 độ có footprint gần như không đổi.
- Đáy hộp bám mặt đường cục bộ; trường hợp chưa chắc được ghi rõ thay vì đoán.
- Toàn bộ vùng đã khai được rà để tìm đối tượng thiếu, hộp thừa và hộp trùng.

### Khó khăn và cách xử lý

Điểm LiDAR thưa và vùng che khuất làm ranh giới đối tượng không phải lúc nào cũng rõ. Tôi xử lý bằng cách đổi góc nhìn, đối chiếu ảnh camera và giữ mức độ không chắc chắn phù hợp. Với đối tượng xa, tôi ưu tiên hình học có bằng chứng thay vì co cuboid theo cụm điểm nhìn thấy gần nhất.

Pre-label không tạo `Animal` và `Obstacle`, vì vậy tôi không chỉ duyệt các hộp model đã vẽ mà còn tìm hai class này và xóa prediction không gắn với đối tượng thật. Tôi cũng kiểm hướng độc lập với footprint vì hộp quay 180 độ có thể vẫn bao điểm nhưng sai đầu xe.

## Kết luận

Phần thực hành giúp tôi phân biệt lỗi pipeline với lỗi cục bộ ở một đối tượng. Lỗi đồng loạt trên cả batch cần dừng pipeline để kiểm cấu hình và phép đổi hệ tọa độ; lỗi ở một hộp cần được xử lý bằng bằng chứng hình học của chính đối tượng đó. Trong phần gán nhãn, pre-label giúp giảm thao tác ban đầu nhưng không thay thế việc kiểm class, vị trí, kích thước, hướng, mặt đường cục bộ và các đối tượng bị bỏ sót. Tôi đã thực hiện phần gán nhãn cá nhân một mình và chịu trách nhiệm về các chỉnh sửa của mình.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét cá nhân và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
