# Báo Cáo Lab K3 — CI/CD cho AI Systems

**Sinh viên:** Do Thai Duong · **Repo:** github.com/dalex0512/Track2-Day21-2A202601331-DoThaiDuong · **Cloud:** AWS (S3 + EC2)

## 1. Siêu tham số đã chọn và lý do

10 lần chạy MLflow trên `train_phase1.csv` (2998 mẫu), quét `n_estimators` (50–800), `max_depth` (3–None), `min_samples_split` (2–5):

| n_estimators | max_depth | min_samples_split | accuracy | f1_score |
|---|---|---|---|---|
| **200** | **16** | **3** | **0.686** | **0.685** |
| 600 | 18 | 2 | 0.680 | 0.678 |
| 800 | 22 | 2 | 0.678 | 0.677 |
| 300 | 20 | 2 | 0.678 | 0.677 |
| 300 | 15 | 2 | 0.670 | 0.669 |
| 200 | 10 | 5 | 0.644 | 0.642 |
| 100 | 5 | 2 (mặc định) | 0.564 | 0.553 |
| 50 | 3 | 2 | 0.558 | 0.518 |

Chọn **`n_estimators=200, max_depth=16, min_samples_split=3`**: accuracy cao nhất trong toàn bộ lưới quét, đồng thời `max_depth` vừa phải (16) tránh overfit so với các cấu hình sâu hơn (20–25) vốn không cải thiện thêm — cho thấy mô hình đã bão hòa quanh mức ~0.68 với lượng dữ liệu này, không phải do thiếu cây hay thiếu độ sâu.

## 2. So sánh accuracy/f1 giữa 2.998 mẫu và 5.996 mẫu

Cùng một bộ siêu tham số (200/16/3), chỉ đổi lượng dữ liệu huấn luyện:

| Chỉ số | Bước 2 (2.998 mẫu) | Bước 3 (5.996 mẫu) |
|---|---|---|
| accuracy | 0.686 | 0.752 |
| f1_score | 0.685 | 0.751 |

Gấp đôi dữ liệu huấn luyện giúp accuracy tăng ~6.6 điểm phần trăm, vượt ngưỡng gate 0.70. Đây là bằng chứng trực tiếp rằng nút thắt của mô hình ở Bước 2 là **lượng dữ liệu**, không phải siêu tham số — quét toàn bộ lưới hyperparameter trên 2998 mẫu không vượt được 0.686.

## 3. Tại sao eval gate cần thiết (bằng chứng thực tế, không mô phỏng)

Lần chạy pipeline đầu tiên (2998 mẫu, hyperparameter tốt nhất tìm được) cho accuracy 0.686 < 0.70. Job **Eval tự nhiên thất bại** (`FAILED: accuracy 0.6860 < 0.70. Huy deploy.`), job **Deploy không chạy** — model chưa đủ tốt không hề chạm tới VM production. Nếu không có gate này, một model kém hơn baseline vẫn sẽ ghi đè `models/latest/model.pkl` trên S3 và service sẽ serve dự đoán sai cho người dùng thật, không có cơ chế nào ngăn lại. Sau khi bổ sung dữ liệu (Bước 3, accuracy 0.752), Eval qua, Deploy chạy, VM phục vụ model mới — đúng vòng đời mong muốn.

## 4. Khó khăn gặp phải và cách giải quyết

- **Trần accuracy dưới ngưỡng 0.70 chỉ với 2998 mẫu**: quét lưới hyperparameter đầy đủ không đưa được RandomForest qua 0.686. Giải quyết: chấp nhận đây là hành vi thật của gate (mục 3), không ép tham số giả để "qua ải" — cung cấp bằng chứng chân thực hơn yêu cầu đề bài.
- **`ModuleNotFoundError: pkg_resources`**: mlflow 2.13.0 cần `pkg_resources` nhưng setuptools ≥81 đã bỏ module này. Fix: ghim `setuptools<81` trong `requirements.txt`.
- **DVC S3 remote lộ secret nếu dùng `dvc remote modify` thường**: chuyển access key sang `dvc remote modify --local` (ghi vào `.dvc/config.local`, đã có sẵn trong `.dvc/.gitignore`) thay vì file `.dvc/config` được commit — tránh lộ credential vào git, đồng thời cho phép CI dùng biến môi trường riêng qua `secrets.CLOUD_CREDENTIALS`.
- **Deploy job fail dù service thực chạy tốt**: `sleep 5` trong health-check không đủ thời gian cho service tải model từ S3 lần đầu (~5–6s). Fix: thay bằng vòng lặp retry (10 lần, 3s/lần).
- **Push không tự kích hoạt pipeline trên repo fork**: GitHub mặc định khoá auto-trigger (`on: push`) trên repository fork cho tới khi chủ repo bấm "I understand my workflows, go ahead and enable them" trong tab Actions (không có API/CLI để bật) — `workflow_dispatch` không bị ảnh hưởng nên dùng tạm để xác minh pipeline trong lúc chờ bật thủ công một lần.
