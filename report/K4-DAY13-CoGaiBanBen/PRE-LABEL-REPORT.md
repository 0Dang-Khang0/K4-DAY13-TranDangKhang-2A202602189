# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng:
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group` / `executed-on-room-LC-machine` / `provided-results`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Ngày 1/10/2026, 21h; Windows/architecture: amd64
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` / `sha256:e039...c20da2fd2c82`; Repo: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo` / `input_sha256: 3b5ea3da...`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: Mặc định của model
- Giả định kênh thứ tư/intensity và nguồn z_ground: Bị bỏ (RGB=0 là placeholder), `z_ground` giả định = 0.075

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | Đủ (run-A) | Chỉ phát hiện 1 hộp do `delta z` sai so với nguồn. |
| B | 1.73 | 0.16 | 13 | 1.034 | Đủ (run-B) | Nhận diện tốt 13 hộp, `delta z` = 1.73 giúp model phân tích tốt. |
| C | 1.73 | 0.32 | 6 | 1.091 | Đủ (run-C) | Voxel 0.32 quá thô khiến nhiều hộp bị gộp hoặc bỏ qua (còn 6 hộp). |

- A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?
- B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không | Tiếp tục | Không có hộp nào bị lệch |
| case-batch-z | 13 / 13 | ~ -1.805 | Không | Dừng batch, kiểm pipeline | Toàn bộ 13 hộp đều bị dịch z (tất cả có z âm) |
| case-one-box-z | 1 / 13 | ~ -1.805 (hộp 1) | Không | Kiểm từng hộp | Chỉ có hộp đầu tiên bị lệch z, 12 hộp kia đúng |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

**1. Trần Đăng Khang:**
- **Vai trò đã làm:** Người vận hành (A), Người ghi log (B), Người xem hình học (C).
- **Tình trạng:** Sử dụng dữ liệu `provided-results` do lỗi môi trường chạy ban đầu.
- **Quan sát:** Ở lượt B, khi `delta=1.73`, model tìm được 13 hộp (so với 1 hộp ở lượt A). Việc cấu hình toạ độ Z đóng vai trò tối quan trọng đối với PointPillars.
- **Diễn giải phép Z:** Phép dịch Z thuận là dời điểm gốc LiDAR (đang ở trên trần xe, ví dụ cao 1.73m) xuống mặt đường (z=0) để phù hợp hệ toạ độ đào tạo của mô hình KITTI.
- **Quyết định lỗi batch:** Với `case-batch-z`, do toàn bộ 13 hộp đều có `z` âm (~ -0.88), đây chắc chắn là lỗi ở bước transform (quên dịch ngược Z). Cần dừng pipeline và không import lên CVAT.
- **Điều chưa chắc:** Việc thay đổi `voxel_size` từ 0.16 lên 0.32 ở lượt C làm giảm mạnh số hộp (từ 13 xuống 6) liệu có phải do model gộp nhầm các hộp gần nhau hay bỏ qua các vật thể nhỏ.

**2. Lê Thanh Tùng:**
- **Vai trò đã làm:** Người kiểm JSON (A), Người vận hành (B), Người ghi log (C).
- **Tình trạng:** Đọc và phân tích dựa trên `provided-results`.
- **Quan sát:** Khi đối chiếu `case-one-box-z.json` với bản gốc `correct`, chỉ có 1 hộp duy nhất bị lệch Z (hộp đầu tiên), 12 hộp còn lại hoàn toàn trùng khớp.
- **Diễn giải phép Z:** Nếu dịch z thuận để đưa về mặt đất khi chạy inference, thì phải có phép dịch z ngược để khôi phục tọa độ của các bounding box về hệ tọa độ gốc của xe trước khi xuất file kết quả.
- **Quyết định lỗi batch:** Với lỗi cục bộ như `case-one-box-z`, ta có thể tiếp tục và cho phép người gán nhãn kiểm tra, điều chỉnh riêng hộp đó trên 3D view.
- **Điều chưa chắc:** Điểm ngưỡng (score threshold) hiện tại của model cắt ở mức nào mà giữ lại được các object có score thấp như 0.31 (pedestrian).

**3. Nguyễn Công Thành:**
- **Vai trò đã làm:** Người xem hình học (A), Người kiểm JSON (B), Người vận hành (C).
- **Tình trạng:** Đọc báo cáo thông qua `provided-results`.
- **Quan sát:** Ở cấu hình A (không dịch Z), model chỉ dự đoán được duy nhất 1 hộp có score thấp, chứng tỏ điểm point cloud bị lơ lửng và không khớp với phân bố hình học mà model đã học.
- **Diễn giải phép Z:** PointPillars cực kỳ nhạy cảm với trục Z. Bất kì sự sai lệch mặt đường nào cũng khiến cho cấu trúc các "pillar" bị trống rỗng hoặc chồng chéo sai lệch.
- **Quyết định lỗi batch:** Lỗi lệch toàn batch tốn chi phí sửa tay rất lớn, do đó luôn phải cấu hình script từ chối đẩy data lên hệ thống gán nhãn nếu phát hiện lệch đồng loạt.
- **Điều chưa chắc:** Không rõ `intensity` thật của LiDAR khi đưa vào model (nếu không giả định RGB=0) sẽ giúp tăng hay giảm độ chính xác của score.

**4. Đoàn Vĩnh Nguyên:**
- **Vai trò đã làm:** Người ghi log (A), Người xem hình học (B), Người kiểm JSON (C).
- **Tình trạng:** Chỉ phân tích kết quả đã chạy.
- **Quan sát:** Trong ảnh `side-demo-delta-1.73-voxel-0.16.png`, các bounding box ôm khá sát các phương tiện trên 2D view, cho thấy việc config kích thước voxel 0.16m có độ phân giải đủ tốt.
- **Diễn giải phép Z:** Hệ toạ độ đào tạo KITTI coi tâm camera/Lidar tương đối ngang bằng, vì vậy hệ toạ độ gốc (đặt LiDAR trên nóc cao) phải được chiếu dời xuống.
- **Quyết định lỗi batch:** Dừng quy trình ngay lập tức, báo cho dev phụ trách pipeline.
- **Điều chưa chắc:** Nếu inference chay không dùng GPU như trong bài này thì bottleneck hiệu suất nằm ở bước chuyển đổi điểm thành voxel hay bước mạng nơ-ron?

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
