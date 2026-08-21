Thu muc nop bai K3 - CI/CD cho AI Systems

Keo 5 anh chup man hinh vao day, dat ten theo thu tu sau (danh dung thu tu khi nop):

01_mlflow.png       - MLflow UI, 10 runs, cot accuracy/f1_score
02_actions.png      - GitHub Actions, 5 job runs, run #5 (push-triggered) xanh
03_health.png       - curl http://44.200.156.93:8000/health -> {"status":"ok"}
04_predict.png      - curl.exe POST /predict -> {"prediction":0,"label":"thap"}
05_s3.png           - S3 console: dvc/files/md5/ (4 thu muc hash) va models/latest/model.pkl

REPORT.md da co san trong thu muc nay (copy tu repo goc).

Nop: repo URL (github.com/dalex0512/Track2-Day21-2A202601331-DoThaiDuong)
   + 5 anh theo thu tu tren
   + REPORT.md
