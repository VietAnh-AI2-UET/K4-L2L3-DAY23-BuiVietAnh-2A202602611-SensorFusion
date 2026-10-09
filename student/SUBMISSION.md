# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Bùi Việt Anh
- MSSV: 2A202602611
- Email: 26ai.anhbv@vinuni.edu.vn
- Link repo (fork): `https://github.com/VietAnh-AI2-UET/K4-L2L3-DAY23-BuiVietAnh-2A202602611-SensorFusion`
- Commit hash nộp (`git rev-parse HEAD`): `7135d60aa5c398c4b7e5b3c6eed0d51e00fc64b0`

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: 
   - `compare`, 
   - `[0, 198]`,
   - `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`,
   - `0`.
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: 
   - `0.9700934579439252`, 
   - `0.7004048582995951`,
   - `519/16/222`.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 
   - `0.15032268781360134` m, 
   - `502`, 
   - `11.343649056695735` m², 
   - `0`, 
   - `239`, 
   - `2.522613065326633`.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 
   - `0.1358667883353908` m, 
   - `502`, 
   - `9.26681165463209` m², 
   - `0`, 
   - `239`, 
   - `2.522613065326633`.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: 
   - Kết hợp lidar và camera giảm RMSE từ 15.03 cm xuống 13.59 cm. 
   - Cả hai mode có 502 lần ghép track với xe thật
   - 239 lượt xe thật chưa được track theo dõi. 
   - Camera giúp định vị chính xác hơn nhưng chưa cải thiện số lượt theo dõi được hay bỏ sót. Các số đếm được cộng qua từng frame, không phải số xe riêng biệt; RMSE chỉ tính trên các cặp ghép được.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?
   - `z` là giá trị đo: lidar đo vị trí `(x, y, z)` theo mét, camera đo tâm hộp `(u, v)` theo pixel. 
   - `R` biểu diễn độ nhiễu của phép đo: lidar có ma trận 3×3, camera có ma trận 2×2. EKF dùng phép chiếu camera để so sánh vị trí 3D với ảnh 2D.
2. Vì sao cần gating Mahalanobis trước khi gán?
   - Đây là bước loại các cặp đo–track có sai lệch quá lớn, có xét độ bất định của dự đoán và phép đo. Nhờ vậy, giảm ghép nhầm vật thể và tránh cập nhật track bằng phép đo không phù hợp.
3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.
   -track-then-fuse: mỗi frame dự đoán track, ghép và cập nhật bằng lidar, rồi dùng camera cập nhật cùng các track đó. `grade_run_fused.log` có `mode: "fused"` và kết quả cuối mỗi frame; log không ghi thứ tự từng cảm biến. Thứ tự này được xác nhận trong `platform/fusion_lab/scripts/run_lab.py`.
4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?
   - Innovation/residual là chênh lệch giữa phép đo và giá trị dự đoán. Khi thông số căn chỉnh camera sai, chênh lệch thường lệch có hệ thống, không dao động quanh 0; nhiều phép đo có thể bị loại khi ghép hoặc kéo vị trí track sai nếu vẫn được nhận.
5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   - Khi danh sách phép đo rỗng, không thể suy ra cảm biến từ phép đo. `sensor` giúp xử lý đúng lượt lidar hay camera và vùng quan sát: lidar mất đo vẫn giảm score nếu track nằm trong vùng nhìn; camera mất đo không làm giảm score. Lidar đo vị trí 3D nên quyết định score, tạo và xóa track; camera chỉ bổ sung thông tin ảnh để cập nhật trạng thái qua EKF, tránh tính thêm một lần điểm tồn tại trong cùng frame.
6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.
   - Track mới có score `1/6`; mỗi lần ghép lidar tăng `1/6`, mất đo trong vùng nhìn giảm `1/6`. Xác nhận khi score `> 0.8`; đã confirmed thì giữ trạng thái này sau miss cho tới khi bị xóa. Xóa khi score của confirmed `< 0.6`, score của track chưa confirmed `<= 0`, hoặc phương sai vị trí `P[0,0]` hay `P[1,1]` `> 9 m²`. Chỉ lượt lidar thực hiện các quyết định này.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không

## Khai báo sử dụng AI (bắt buộc)

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Codex

| Công cụ | Dùng cho việc gì | Bạn đã kiểm chứng thế nào |
|---|---|---|
| codex | Thiết kế lập trình hàm | Kiểm tra type đầu vào / ra |
| codex | Đọc hiểu hàm | Hỏi chức năng của hàm, cách hàm xử lý luồng<br> Viết docstring cho hàm |
| codex | Hỏi ý nghĩa của command | Đọc command và chạy thử |
| codex | Tra cứu thông tin<br> | Hỏi bạn cùng bàn |
| codex | Xác định bước làm tiếp theo | Đọc lại các file hướng dẫn làm bài |
| codex | Phân tích kết quả | Tự đọc kết quả và đối chiếu |

## Checklist nộp

- [X] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [X] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [X] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [X] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [X] Đã điền đủ file này, gồm khai báo AI
- [X] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [X] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [X] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
