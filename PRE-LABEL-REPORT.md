# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: 
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt). Nghiêm Trà My (MSSV: 02239)
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nghiêm Trà My; 02/10/2026 15:52; Windows 11 x86_64 / Docker Desktop Linux amd64 (4 CPUs, 4GB RAM limit)
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1` (working_tree_dirty: true)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - PCD / frame_id: `input/demo.pcd` / `demo` (17,238 điểm, chuyển đổi từ KITTI Vision Benchmark Suite 000008)
  - Nơi được phép chạy: Máy cá nhân/nhóm chạy offline bằng container qua runner `student-bundle.py`
  - Fingerprint SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp:
  - Đường dẫn checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`
  - Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (`[0, -39.68, -3, 69.12, 39.68, 1]`); score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - Giả định kênh thứ tư/intensity: PCD gốc được loại bỏ reflectance thật để chuẩn hóa theo adapter hiện tại; pipeline sử dụng kênh hằng số giả lập qua 2 lượt đọc (2-pass: reflectance = 0 cho `vehicles`, reflectance = 0.7 cho `cyclist`/`pedestrian`) nhằm kích hoạt đúng các prediction head tương ứng; RGB là placeholder uint32 = 0.
  - Nguồn z_ground: Được ước lượng tự động từ độ cao mặt đất cục bộ của scan PCD nguồn (`demo.pcd`), giá trị tính toán được là `z_ground = 0.075 m` (chính xác: `0.07500000000000001`).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Chỉ nhận diện được duy nhất 1 hộp `vehicles` ở vị trí x≈13.15m, y≈-0.45m, z=0.330m với score thấp (0.322). Khi delta=0, toàn bộ đám mây điểm bị lệch cao độ so với giả định sensor KITTI, model bỏ sót hầu hết xe và người đi bộ trong cảnh. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Nhận diện 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian). Các xe ở gần bám khít cụm điểm LiDAR (ví dụ xe tại x=8.09m score 0.933; xe tại x=14.77m score 0.928). Độ cao z của các hộp nằm trong khoảng 0.70m đến 1.43m, khớp với bề mặt thực của vật thể trên đường. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Nhận diện 6 hộp nhưng 100% đều là `pedestrian` (0 vehicles, 0 two-wheels). Voxel lớn gấp đôi (0.32m) làm giảm độ phân giải của pseudo-image (giảm 4 lần số cell), checkpoint pretrained ở 0.16m không nhận ra cấu trúc xe, sinh ra dự đoán false-positive cho pedestrian. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng `side-demo-*.png` và `boxes-*.json` khác ở số lượng và độ phủ của các hộp: ở A gần như toàn bộ vật thể bị bỏ sót (chỉ còn 1 hộp xe tại x≈13.15m, score 0.322); ở B có 10 xe, 1 xe hai bánh và 2 người đi bộ bám sát các cụm điểm thực tế từ x≈3.7m đến x≈55.6m. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là: sensor height delta = 1.73m là giả định chuẩn của xe thu thập KITTI, khi áp dụng trên các dòng xe khác (như Robotaxi thực tế) thì z_ground và delta cần được cân chỉnh chính xác theo vị trí đặt sensor thực tế; ngoài ra cụm điểm thưa ở xa x≈55.6m cần đối chiếu thêm camera để khẳng định có phải xe hay vật cản lề đường.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng `side-*.png` và JSON khác ở chỗ: ở C mất hoàn toàn 10 hộp vehicles và 1 hộp two-wheels, toàn bộ 6 hộp đều bị gán nhãn `pedestrian`, trong đó có những hộp nằm đè lên vị trí cụm điểm của xe ô tô ở lượt B (ví dụ vùng x≈13.2m, x≈10.5m). Số lượng/lớp/vị trí thay đổi như sau: số lượng giảm từ 13 xuống 6, class vehicles chuyển thành 0, two-wheels thành 0, pedestrian tăng từ 2 lên 6, kích thước hộp bị co nhỏ về cỡ pedestrian. Có đủ bằng chứng để kết luận tốt hơn không? Hoàn toàn KHÔNG tốt hơn; thực tế cấu hình C sai lệch hoàn toàn vì checkpoint được train tối ưu cho voxel size 0.16m; việc tăng pillar lên 0.32m mà không retrain mạng đã làm biến dạng biểu diễn không gian và phá vỡ cơ chế anchor matching.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
  - Giới hạn ROI: Checkpoint KITTI chỉ quan sát cửa sổ phía trước (front-window: `[0, -39.68, -3, 69.12, 39.68, 1]`), do đó các vật thể nằm phía sau (x < 0) hoặc nằm ngoài phạm vi ngang/dọc của ROI không được coi là model bỏ sót (miss) trong bài thử nghiệm này.
  - Góc nhìn Side (chiếu ngang x-z): Ảnh Side chiếu phẳng không gian 3D lên mặt phẳng x-z nên các vật thể có cùng khoảng cách dọc x nhưng khác làn y sẽ bị chồng chập lên nhau. Vì vậy, ta không thể đánh giá chính xác góc quay (yaw) hay phân biệt các xe chạy song song nếu chỉ nhìn riêng góc Side; cần phải kết hợp với góc nhìn chim bay từ trên xuống (Top/Bird's Eye View x-y), Front view và ảnh camera thực tế.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Cả 3 file JSON A, B, C đều KHÔNG được import vào CVAT vì đây là kết quả demo KITTI phục vụ thực hành offline, khác biệt về domain, cảm biến và frame so với nhiệm vụ Robotaxi trên CVAT.
  - Trong nội bộ bài thực hành: Lượt A và C hoàn toàn không đạt tiêu chuẩn. Lượt B có kết quả hợp lý nhất nhưng vẫn chỉ là pre-label sơ bộ, chưa đủ cơ sở để coi là nhãn đúng (ground-truth) vì còn một số hộp có score thấp (~0.31 - 0.38). Cần kiểm tra tiếp bằng cách chiếu kết hợp ảnh camera (camera projection) để xác nhận class, điều chỉnh hướng đầu xe (heading arrow) và kiểm tra đáy hộp (z_bottom) có khớp với mặt đường cục bộ hay không.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp bình thường | File `qc-cases/case-correct.json` và `qc-cases/side-correct.png` giữ nguyên prediction từ lượt B. Toàn bộ 13/13 hộp nằm khớp trên mặt đường (z từ 0.70m đến 1.43m). Không có dấu hiệu lỗi pipeline z. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi (chỉ lệch z) | DỪNG BATCH NGAY | File `qc-cases/case-batch-z.json` và `qc-cases/side-batch-z.png`. Toàn bộ 100% hộp (13/13) đều bị trừ z đúng 1.805m (= delta 1.73m + z_ground 0.075m), khiến tất cả các hộp bị chìm sâu xuống dưới lòng đất (z âm từ -0.88m đến -0.55m), trong khi x, y, yaw, size giữ nguyên. Đây là lỗi hệ thống của pipeline (quên bước biến đổi z ngược về hệ tọa độ sensor), tuyệt đối không sửa tay từng hộp. |
| case-one-box-z | 1 / 13 | -1.805 m (chỉ trên boxes[0]) | Không đổi | Kiểm từng hộp (không dừng batch) | File `qc-cases/case-one-box-z.json` và `qc-cases/side-one-box-z.png`. Chỉ duy nhất hộp đầu tiên (hộp xe tại x=8.09m, y=1.21m) bị chìm xuống đất với z = -0.884m (lệch -1.805m), trong khi 12 hộp còn lại vẫn giữ nguyên cao độ chuẩn. Đây là lỗi cục bộ của một đối tượng (object-local error), chỉ cần kiểm tra đa góc nhìn và điều chỉnh lại cao độ đáy của riêng hộp đó trên CVAT, không phải lỗi pipeline cả batch. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.
*Ghi nhận:* Thư mục `qc-cases` được sinh ra từ helper script `pipeline-qc-cases.py`, sử dụng prediction gốc của lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`) để mô phỏng có chủ đích hai lỗi quy trình thường gặp: (1) quên hoàn toàn bước nghịch đảo z cho toàn bộ batch (`case-batch-z`), và (2) lỗi cao độ cục bộ ở 1 đối tượng (`case-one-box-z`). Các file này chỉ phục vụ mục đích huấn luyện nhận diện lỗi (training only), không phải kết quả detector độc lập và không được import vào CVAT.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

- **Nghiêm Trà My (MSSV: 02239):**
  - *Vai trò đã làm:* Người trực tiếp vận hành chạy lệnh Docker runner (`student-bundle.py run --bundle . --out ..\output`) trên máy cá nhân; kiểm tra tính toàn vẹn và smoke test (`smoke.json` đạt status `passed`); đối chiếu số liệu và hình ảnh giữa các lượt A, B, C; phân tích các ca kiểm soát QC.
  - *Quan sát A/B/C:* So sánh lượt A và B: Trong `run-A/summary.csv`, chỉ nhận diện được 1 hộp với mean_z=0.330; sang `run-B/summary.csv`, số hộp tăng lên 13 với mean_z=1.034. Đối chiếu `run-B/side-demo-delta-1.73-voxel-0.16.png` và file JSON, thấy các xe ở cự ly gần (x=8.09m, x=14.77m) đều có confidence rất cao (>0.92) và bao trọn cụm điểm LiDAR. Khi so sánh B và C, việc tăng voxel_size từ 0.16 lên 0.32 khiến toàn bộ 10 xe và 1 xe hai bánh biến mất hoàn toàn, model chỉ phát hiện 6 pedestrian giả lập (false positives) trong `run-C/boxes-demo-delta-1.73-voxel-0.32.json`.
  - *Diễn giải phép z thuận/ngược:* Trong pipeline PointPillars KITTI:
    + Chiều thuận (chuẩn bị input vào model): `z_model = z_source - z_ground - delta`. Phép biến đổi này đưa gốc tọa độ z về đúng mặt phẳng quy chiếu mà model được train (sensor cách mặt đất một khoảng delta = 1.73m, mặt đất cục bộ ước lượng là z_ground = 0.075m).
    + Chiều ngược (sau khi model xuất hộp): `z_source = z_model + z_ground + delta`. Bước này cộng bù lại độ cao để trả hộp dự đoán về đúng hệ tọa độ ban đầu của đám mây điểm LiDAR nguồn (PCD). Nếu thiếu bước này, toàn bộ hộp sẽ bị chìm sâu xuống đất đúng một khoảng bằng `z_ground + delta = 1.805m`.
  - *Quyết định lỗi batch và hành động:* Khi gặp `case-batch-z` (toàn bộ 13/13 hộp đều bị chìm đều -1.805m so với mặt đất), quyết định dứt khoát là **DỪNG BATCH NGAY LẬP TỨC**. Báo cho LC/kỹ thuật phụ trách pipeline kiểm tra lại khâu post-processing chuyển đổi hệ tọa độ; tuyệt đối không mất thời gian điều chỉnh thủ công từng hộp vì đây là lỗi hệ thống. Ngược lại, nếu chỉ có 1 hộp bị lệch như `case-one-box-z`, ta giữ nguyên pipeline và tiến hành chỉnh tay cao độ hộp đó trên CVAT.
  - *Điều chưa chắc:* Do scan PCD hiện tại là KITTI demo đã bị loại bỏ kênh reflectance thật (thay bằng giá trị hằng số qua 2-pass đọc), em chưa chắc chắn về độ nhạy thực tế của model khi gặp các điều kiện phản xạ LiDAR biến thiên trên xe Robotaxi thật; đồng thời ở các cự ly xa ngoài 40m, độ phân giải điểm rất thưa nên việc xác định ranh giới sau của xe (rear boundary) vẫn cần đối chiếu thêm ảnh camera trực diện.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
