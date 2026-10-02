# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

> Đường dẫn file trong báo cáo tính từ thư mục `output/` của gói nộp này (bản sao nguyên vẹn của `ket-qua-nhom-01`).

## Nhóm và provenance

- Mã nhóm: Zero 3
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: **[ĐIỀN tên]**; 2026-10-01, docker-load bắt đầu 08:06:54 UTC (15:06 giờ VN), run-A/B/C/qc-cases xong lúc ~08:08 UTC; laptop Windows 11 Home, Docker Desktop (Linux containers, WSL2), engine `linux/x86_64` = **amd64**, 4 CPU, ~6 GB RAM cấp cho Docker VM. Container giới hạn 4 CPU / 4 GB, `--network none`. Thời gian theo `smoke.json`: load 50.4 s, A 6.5 s, B 4.8 s, C 3.1 s, QC 1.6 s (gồm khởi động container, không phải số đo hiệu năng).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`, ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`. Code trong image: repo revision `0831856` (manifest ghi `working_tree_dirty: true`), `preannotate.py` sha256 `65edf6ac…c5ca`, helper `pipeline-qc-cases.py` sha256 `c177fc00…aa7`. Repo Student trên máy: `f5f1de0`. Runner `student-bundle.py` đã kiểm hash mọi file theo `manifest.json` trước khi chạy; `smoke.json` → `status: passed`.
- PCD được cấp / frame_id: `input/demo.pcd` (gói Student amd64), `frame_id=demo`, 17238 điểm, sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`. Nguồn: KITTI / MMDetection3D demo `000008.bin`, giấy phép CC BY-NC-SA 3.0, dùng học thuật phi thương mại trên laptop nhóm. Fingerprint LC: **[ĐIỀN nếu LC cấp]**
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window của checkpoint (không `--full-scene`, `rear=0` ở cả ba lượt); score threshold 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance thật của KITTI đã bị bỏ khi tạo PCD; script dùng **kênh hằng** theo adapter hiện tại, RGB=0 chỉ là placeholder — không phải intensity phục hồi. `z_ground = 0.075 m` do script **ước lượng từ PCD** (giống nhau ở cả ba lượt), `delta = 1.73 m` là giả định cao độ sensor của checkpoint KITTI. PCD đã được dịch z +1.73 m (x/y giữ nguyên) để thực hành hệ nguồn.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV                                                                                                    | Quan sát có bằng chứng                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------ | ----- | --------- | -------- | ------ | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A      | 0     | 0.16      | 1        | 0.330  | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv`       | Chỉ 1`vehicles` tại x=13.15, y=−0.45, score **0.32** (sát ngưỡng 0.3). Đáy hộp z = 0.33 − 1.46/2 = **−0.40 m**, tức chìm dưới đường tham chiếu z=0 trên Side. Các cụm điểm dày ở x≈3–25 m (thấy rõ trên Side) không được phát hiện.                                                                                                                                                                                                                                                     |
| B      | 1.73  | 0.16      | 13       | 1.034  | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 10`vehicles` (score 0.50–0.93), 1 `two-wheels` (0.38), 2 `pedestrian` (0.34, 0.32). Hộp gần (x<25 m) có đáy ≈ −0.05…0.15 m, bám mặt đất trên Side; hộp xa (x=33.7/41.0/55.6) đáy 0.32–0.46 m, cùng chiều với các vệt điểm mặt đất dâng lên ở x≈30–50 m. Kích thước xe l≈3.1–4.2, w≈1.5–1.7, h≈1.5–1.8 m — hợp lý với ô tô. Điểm nghi: `vehicles` x=9.38, y=4.23 đáy **0.65 m** (lơ lửng) và chồng BEV một phần với `two-wheels` x=10.32, y=5.25 (đáy 0.47 m). |
| C      | 1.73  | 0.32      | 6        | 1.091  | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | **Cả 6 hộp đều `pedestrian`**, không còn `vehicles` nào. Một số hộp nằm đúng chỗ B có xe score cao: C ped (9.11, 0.40) ↔ B xe (8.09, 1.21, score 0.93); C ped (13.24, −0.95) ↔ B xe (14.77, −1.08, 0.93); C ped (13.15, 4.20) & (10.46, 4.93) ↔ vùng B xe nghi lơ lửng + two-wheels. Score C cao nhất 0.81 nhưng class mâu thuẫn với kích thước cụm điểm.                                                                                                                                      |

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  Khác. Đổi `delta` là đổi **input trước inference** (`z_model = z_source − z_ground − delta`), nên mạng nhìn thấy đám mây điểm ở một độ cao khác. Checkpoint KITTI học với LiDAR cao ~1.73 m (mặt đường ở z ≈ −1.73 trong hệ model); ở A (delta=0) mặt đường nằm quanh z≈0 trong hệ model, sai phân bố huấn luyện, nên mạng gần như không nhận ra vật: 1 hộp so với 13 hộp. Không phải mọi hộp lệch đúng 1.73 m: hộp A (13.15, −0.45, z=0.33) gần nhất với hộp B (14.77, −1.08, z=0.90) chỉ chênh z 0.57 m và chênh x 1.6 m, và 12 hộp còn lại của B không có cặp ở A. Còn **dịch hộp sau inference** (quên/nhầm phép ngược) thì giữ nguyên số hộp, class, x/y/yaw và chỉ cộng/trừ một hằng số z — đó là đúng cái ca `batch-z` mô phỏng.
- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  Giữ delta=1.73, đổi pillar 0.16 → 0.32 m: số hộp 13 → 6 và class đổi hoàn toàn sang `pedestrian`, kể cả ở vị trí B có xe score 0.93. Checkpoint được huấn luyện với pillar 0.16; pillar 0.32 làm lưới BEV thô gấp đôi, đặc trưng/anchor không còn khớp với những gì mạng đã học → đầu ra lệch domain. mean_z C (1.091) cao hơn B (1.034) nhưng so sánh vô nghĩa vì tập hộp/class khác hẳn. **Không có nhãn chuẩn nên không đủ bằng chứng xếp hạng tuyệt đối**; chỉ có thể nói B nhất quán hơn với hình học điểm (kích thước xe hợp lý, đáy bám đất) và với cấu hình huấn luyện của checkpoint. Không chọn theo số hộp hay score.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  Chỉ chạy front-window (`rear=0`), nên vật phía sau/ngoài ROI không được tính là miss. Side là hình chiếu x–z toàn scene: các vật khác y bị chồng lên nhau (vd. ở x≈7–11 m có 4 hộp B chồng nhau trên Side dù y từ −3.9 đến 5.3), nên không dùng Side để kết luận trùng lặp hay hộp nào bám cụm nào. Side **không thấy yaw** (yaw quay quanh trục z) và không thấy chiều rộng; cần Top (BEV) để kiểm yaw/vị trí xy, Front để kiểm w/h. Đường z=0 chỉ là tham chiếu plot, không phải mặt đường cục bộ (mặt đất dâng ở x≈30–50 m).
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - `run-A` và `run-C`: không dùng làm pre-annotation (cấu hình không khớp checkpoint, kết quả trái hình học).
  - `run-B`: tốt nhất trong ba, nhưng vẫn **chưa đủ cơ sở import** — đây là KITTI demo, khác frame Robotaxi (không bao giờ import vào job Robotaxi). Nếu là frame được cấp, trước khi dùng cần kiểm Top/Front/camera cho: xe x=9.38/y=4.23 (đáy 0.65 m, nghi lơ lửng hoặc FP), cặp xe–two-wheels chồng nhau ở y≈4–5, các hộp score sát ngưỡng (two-wheels 0.38, ped 0.34/0.32), và xe xa x=55.6 (score 0.50, ít điểm). Không có intensity thật nên ped/two-wheels càng cần nhìn camera.
  - `qc-cases/*.json`: training-only, **không import**.

## Ca QC có kiểm soát — không import CVAT

Helper `pipeline-qc-cases.py` tạo ba biến đổi **có chủ đích** từ prediction B (`boxes-demo-delta-1.73-voxel-0.16.json`, sha256 `16f30b08…cf61`), không chạy lại model, không phải nhãn đúng. `height_offset = delta + z_ground = 1.73 + 0.075 = 1.805 m`. Nhóm đã so từng trường của 13 hộp với B bằng script.

| Ca             | Số hộp lệch z / tổng hộp                               | Lượng lệch         | Class/x/y/yaw có đổi?                         | Dừng batch, kiểm từng hộp hay chưa rõ?                                                                                                                                                                                                                                                                                                   | Bằng chứng                                                                                                                                                                                                                    |
| -------------- | ----------------------------------------------------------- | --------------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| case-correct   | 0 / 13                                                      | 0                     | Không (class, x, y, l/w/h, yaw, score giống B) | Không có lỗi pipeline z; vẫn**chưa phải** cuboid đúng — chỉ là prediction B giữ nguyên chuyển đổi, các nghi vấn của B ở trên vẫn cần kiểm                                                                                                                                                                        | `case-correct.json`, `side-correct.png` trùng `run-B`                                                                                                                                                                    |
| case-batch-z   | 13 / 13                                                     | −1.805 m (toàn bộ) | Không — chỉ z đổi                           | **Dừng batch**, không sửa tay; báo LC kiểm phép chuyển frame (thiếu cộng ngược `z_ground + delta`), yêu cầu tạo lại prediction từ pipeline đúng                                                                                                                                                                      | `side-batch-z.png`: mọi hộp tụt xuống dưới z=0, phần lớn nằm hẳn dưới đất (đáy −1.1…−1.9 m) trong khi điểm không đổi; lượng lệch đúng bằng `delta+z_ground` và giống hệt nhau ở mọi hộp |
| case-one-box-z | 1 / 13 (hộp đầu:`vehicles` x=8.09, y=1.21, score 0.93) | −1.805 m             | Không — chỉ z của 1 hộp                     | **Kiểm từng hộp**: 12 hộp còn lại bám đất như B → không phải lỗi batch; mở Top/Side/Front (+camera) cho hộp này và sửa z hộp đó. Lưu ý lượng lệch trùng `delta+z_ground` là dấu hiệu nên báo LC xem có bước xử lý riêng cho hộp này không, nhưng không chốt lỗi pipeline chỉ từ 1 hộp | `side-one-box-z.png`: duy nhất một hộp ở x≈6.2–10 m nằm dưới z=0 (z −1.65…−0.12)                                                                                                                                  |

## Nhận xét cá nhân

### Nguyễn Công Bình —02278

- **Vai trò đã làm:** Người vận hành: chuẩn bị máy (Windows 11, Docker Desktop Linux containers, engine amd64 khớp gói `student-prelabel-amd64`), chạy `student-bundle.py run --bundle . --out ..\ket-qua-nhom-01` một lần, kiểm `smoke.json` có `status: passed` và đủ file `run-A/B/C` + `qc-cases`; sau đó đọc lại JSON/Side của cả ba lượt.
- **Một quan sát A/B/C có dẫn file hoặc hộp/vùng:** Ở `run-A/boxes-demo-delta-0-voxel-0.16.json` chỉ có 1 hộp `vehicles` tại x=13.15, y=−0.45, score 0.32 — chỉ vừa qua ngưỡng 0.3, và đáy hộp ở z = 0.33 − 1.46/2 ≈ −0.40 m, nghĩa là chìm dưới đường z=0 trên `side-demo-delta-0-voxel-0.16.png`. Cùng PCD đó, `run-B` (delta=1.73) ra 13 hộp, trong đó 10 xe có score 0.50–0.93 và các xe gần (x<25 m) có đáy ≈ −0.05…0.15 m, bám mặt đất. Hộp gần nhất với hộp A ở B là xe (14.77, −1.08) z=0.90: lệch z chỉ 0.57 m chứ không phải 1.73 m, và lệch cả x 1.6 m.
- **Diễn giải phép z thuận/ngược:** Phép thuận `z_model = z_source − z_ground − delta` đưa điểm về hệ mà checkpoint KITTI đã học (LiDAR cao ~1.73 m, đường ở z ≈ −1.73). Ở A, delta=0 nên đường nằm quanh z≈0 trong hệ model — sai phân bố huấn luyện — model gần như không nhận ra vật nào. Vì vậy đổi delta là đổi **input**, kết quả thay đổi cả số hộp/vị trí/score chứ không chỉ dịch một hằng số. Phép ngược `z_source = z_model + z_ground + delta` chỉ đưa hộp về lại hệ nguồn; nếu quên phép ngược thì mọi hộp cùng tụt đúng `z_ground + delta`, còn class/x/y/yaw giữ nguyên.
- **Một quyết định lỗi batch và hành động:** Với `qc-cases/case-batch-z.json`, cả 13/13 hộp lệch đúng −1.805 m (= 1.73 + 0.075) và mọi trường khác giống hệt B; trên `side-batch-z.png` toàn bộ hộp nằm dưới đất trong khi điểm không đổi. Quyết định: **dừng batch**, không sửa tay từng hộp, báo LC kiểm bước chuyển frame ngược và yêu cầu tạo lại prediction từ pipeline đúng. Ngược lại `case-one-box-z` chỉ lệch 1/13 hộp (xe x=8.09) nên tôi sẽ kiểm riêng hộp đó bằng Top/Side/Front chứ không kết luận lỗi pipeline.
- **Điều chưa chắc:** Không có nhãn chuẩn hay ảnh camera nên tôi không biết hộp score thấp ở B (two-wheels 0.38, pedestrian 0.34/0.32) và xe x=9.38/y=4.23 có đáy cao 0.65 m là vật thật hay false positive. Tôi cũng chưa chắc `z_ground = 0.075` (ước lượng một giá trị cho cả scene) còn đúng ở xa, vì trên Side các vệt điểm mặt đất dâng lên ở x≈30–50 m và hộp xa có đáy 0.3–0.46 m. Ngoài ra PCD đã bỏ reflectance thật, tôi chưa đánh giá được việc dùng kênh hằng ảnh hưởng thế nào tới pedestrian/two-wheels.

### LÊ ĐỨC HUY-02245

- Vai trò đã làm:
- Một quan sát A/B/C có dẫn file hoặc hộp/vùng:
- Diễn giải phép z thuận/ngược:
- Một quyết định lỗi batch và hành động:
- Điều chưa chắc:

### NGUYỄN NGỌC NGUYÊN-02286

- Vai trò đã làm:
- Một quan sát A/B/C có dẫn file hoặc hộp/vùng:
- Diễn giải phép z thuận/ngược:
- Một quyết định lỗi batch và hành động:
- Điều chưa chắc:

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

---
