# Báo Cáo Ngắn Môn MLOps (Day 21 - Track 2)

## 1) Bộ siêu tham số đã chọn và lý do

Mô hình sử dụng là `RandomForestClassifier` (xem `src/train.py`), với cấu hình hiện tại trong `params.yaml`:

- `n_estimators: 200`
- `max_depth: 10`
- `min_samples_split: 5`
- `min_samples_leaf: 1`
- `max_features: 0.5`
- `class_weight: balanced_subsample`
- `criterion: log_loss`
- `bootstrap: false`
- `random_state: 42` (gắn trực tiếp trong code để tái lập kết quả)

Lý do chọn:

- Tăng `n_estimators` lên 200 để giảm phương sai và ổn định dự đoán hơn so với cấu hình nhỏ.
- Giới hạn `max_depth=10` và `min_samples_split=5` để kiểm soát overfitting.
- Dùng `max_features=0.5` để tăng tính đa dạng giữa các cây, giúp tổng quát hóa tốt hơn.
- Dùng `class_weight=balanced_subsample` để xử lý mất cân bằng lớp theo từng bootstrap sample.
- Dùng `criterion=log_loss` để tối ưu tiêu chí tách node theo hướng xác suất.
- `bootstrap=false` được thử nghiệm để tận dụng toàn bộ dữ liệu mỗi cây trong cấu hình hiện tại.

Kết quả gần nhất đọc từ `outputs/metrics.json`:

- `accuracy = 0.66`
- `f1_score = 0.6529`

## 2) Khó khăn gặp phải và cách giải quyết

1. Thiết lập MLflow local với SQLite và artifact path ổn định
- Khó khăn: khi dùng tracking URI dạng SQLite, artifact có thể bị ghi không nhất quán nếu không đặt experiment/artifact location rõ ràng.
- Cách giải quyết: thêm hàm `_configure_local_mlflow_experiment()` trong `src/train.py` để tự tạo/chọn experiment `local-default` và cố định artifact location về thư mục `mlartifacts/`.

2. Kết nối DVC với Cloud Storage trong CI
- Khó khăn: pipeline CI cần vừa xác thực cloud, vừa đảm bảo `dvc pull` chạy được với credential tạm thời của GitHub Actions.
- Cách giải quyết: trong `.github/workflows/mlops.yml`, dùng `google-github-actions/auth@v2`, export `GOOGLE_APPLICATION_CREDENTIALS`, cấu hình `dvc remote modify --local ... credentialpath`, và thêm bước “Verify bucket access” trước khi pull.

3. Đảm bảo chất lượng trước deploy (evaluation gate)
- Khó khăn: cần ngăn deploy nếu mô hình chưa đạt chất lượng tối thiểu.
- Cách giải quyết: job `eval` đọc `accuracy` từ output của job train và chặn pipeline nếu thấp hơn ngưỡng (`0.69` trong workflow hiện tại).
- Ghi chú: mục tiêu bài lab mô tả ngưỡng `>= 0.70`, vì vậy cần đồng bộ lại ngưỡng giữa mô tả và workflow để tránh lệch tiêu chí.

4. Triển khai suy luận trên VM và tải model từ cloud
- Khó khăn: service phải tự tải model mới nhất khi khởi động và có health check đáng tin cậy sau deploy.
- Cách giải quyết: `src/serve.py` tải `models/latest/model.pkl` từ GCS khi start app, expose `/health` và `/predict`; job `deploy` restart `systemd` service và retry health check nhiều lần trước khi fail.

## 3) Tài nguyên minh chứng trong thư mục submission

- MLflow tracking: `submission/screenshots/mlflow.png`
- GitHub Actions pipeline: `submission/screenshots/github-action.png`
- DVC / Cloud storage: `submission/screenshots/dvc.png`
- Serving endpoint: `submission/screenshots/serving.png`
- Continuous training: `submission/screenshots/continuous-training.png`
- Kết quả tổng hợp: `submission/screenshots/result.png`

## 4) Ghi chú hiện trạng

- Repository hiện có workflow chính: `.github/workflows/mlops.yml`.
- Nếu cần bám đầy đủ yêu cầu Bước 3 theo README, cần bổ sung thêm workflow huấn luyện liên tục theo lịch hoặc trigger dữ liệu mới.
