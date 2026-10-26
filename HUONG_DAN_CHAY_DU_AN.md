# Hướng dẫn cài đặt, huấn luyện và sử dụng

Tài liệu này mô tả luồng chạy của dự án phát hiện hành vi bạo lực trong video CCTV, từ chuẩn bị môi trường và dữ liệu đến huấn luyện, đánh giá, demo video/webcam và cảnh báo email.

> Đây là hệ thống thử nghiệm học thuật, không thay thế người giám sát. Không dùng kết quả tự động làm căn cứ duy nhất cho quyết định khẩn cấp hoặc cáo buộc một người.

## 1. Dự án làm gì?

Hệ thống nhận một video hoặc hình ảnh từ webcam, lấy mẫu các khung hình theo thời gian, rồi dự đoán video thuộc một trong hai lớp:

- **Violence**: có hành vi đánh nhau/bạo lực.
- **NonViolence**: không có hành vi bạo lực.

Dự án có hai luồng sử dụng:

1. **Đánh giá offline**: chạy mô hình trên tập kiểm thử RWF-2000 và tính accuracy, precision, recall, F1, AUC.
2. **Demo suy luận**: chạy trên video hoặc webcam, hiển thị xác suất và trạng thái cảnh báo theo thời gian. Với file video, chương trình còn in dự đoán tổng hợp cho toàn clip.

Mô hình được huấn luyện là **R(2+1)D-18**, một mạng tích chập không gian-thời gian cho video. Mô hình khởi tạo bằng trọng số học trước trên **Kinetics-400**, sau đó fine-tune cho hai lớp của RWF-2000.

## 2. Luồng xử lý tổng quát

    RWF-2000
      └─ src/build_manifest.py
           ├─ train: lấy từ official train
           ├─ val: tách có phân tầng từ official train
           └─ test: dùng official val/test
                ↓
         src/dataset.py: lấy 16 frame, resize/crop, augment khi train
                ↓
         R(2+1)D-18: dự đoán NonViolence / Violence
                ↓
         scripts/train.py: huấn luyện, chọn checkpoint theo F1 validation
                ↓
         scripts/evaluate.py: đánh giá checkpoint trên test, trung bình 5 đoạn/clip

Luồng demo:

    Video/Webcam
      → OpenCV đọc frame và đổi BGR sang RGB
      → bộ lọc chất lượng: bỏ cảnh quá tối, quá phẳng hoặc đứng hình
      → lấy mẫu 16 frame trong cửa sổ video
      → R(2+1)D-18 dự đoán P(Violence)
      → Temporal confidence gate làm mượt điểm và xác nhận sự kiện kéo dài
      → hiện overlay, ghi log; gửi email nếu được bật và có thông tin xác thực

## 3. Cấu trúc thư mục chính

    configs/config.yaml             cấu hình dữ liệu, model, train, đánh giá, gate, alert
    src/config.py                   đọc YAML và bí mật từ .env
    src/check_env.py                kiểm tra Python, PyTorch, CUDA, GPU và thư viện
    src/build_manifest.py           quét dữ liệu và tạo train/val/test manifest
    src/dataset.py                  đọc video, lấy frame, tiền xử lý và augmentation
    src/model.py                    tạo mô hình R(2+1)D-18
    src/gate.py                     làm mượt điểm và xác nhận sự kiện theo thời gian
    src/alerter.py                  gửi email, lưu ảnh chụp và audit log
    scripts/train.py                huấn luyện mô hình
    scripts/evaluate.py             đánh giá trên validation và test
    scripts/infer.py                demo video/webcam
    scripts/cross_dataset_eval.py   đánh giá tùy chọn trên bộ dữ liệu khác
    outputs/                        artifacts sinh khi chạy; chỉ metrics và log train đã rà soát được công khai
    RWF-2000/                       video dữ liệu (không đưa lên Git)

## 4. Công nghệ và công cụ

- **Python 3.12**: ngôn ngữ chạy dự án.
- **PyTorch + torchvision**: tensor, huấn luyện GPU và kiến trúc R(2+1)D-18.
- **CUDA**: tăng tốc trên GPU NVIDIA nếu bản PyTorch/CUDA tương thích.
- **Decord**: đọc và giải mã video khi tạo dữ liệu huấn luyện/đánh giá.
- **OpenCV**: resize frame và hiển thị/lưu video demo.
- **NumPy**: xử lý mảng frame và xác suất.
- **scikit-learn**: accuracy, precision, recall, F1, AUC, confusion matrix.
- **PyYAML / python-dotenv**: cấu hình YAML và bí mật trong .env.
- **Git/GitHub**: lưu mã nguồn. Bộ video, môi trường ảo, file .env và checkpoint đã được loại khỏi Git bằng .gitignore.

Máy đã dùng để tái hiện dự án: Windows, Python 3.12.10, PyTorch 2.11.0+cu128, CUDA khả dụng, NVIDIA GeForce RTX 3050 Ti Laptop GPU với VRAM 4 GB. Với card 4 GB, cấu hình hiện tại dùng batch size 4 và AMP; nếu hết VRAM, giảm batch size xuống 2.

## 5. Cài đặt trên Windows

Mở PowerShell trong thư mục gốc dự án, ví dụ:

    cd C:\Users\ADMIN\Desktop\HTTM

Tạo môi trường ảo Python 3.12:

    py -3.12 -m venv .venv

Nếu PowerShell chặn script kích hoạt, chỉ cho phép trong cửa sổ hiện tại rồi kích hoạt:

    Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
    .\.venv\Scripts\Activate.ps1

Cài các thư viện theo cấu hình CUDA 12.8 của môi trường đã chạy dự án:

    python -m pip install --upgrade pip
    pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cu128

Kiểm tra môi trường:

    python -m src.check_env

Cần thấy CUDA available: True và tên GPU. Nếu hiện False, chương trình vẫn có thể chuyển sang CPU nhưng huấn luyện sẽ chậm đáng kể.

> Lệnh cài đặt trên phù hợp với môi trường đã kiểm tra trong dự án này. Với máy khác, hãy chọn đúng gói PyTorch/CUDA cho driver và GPU của máy đó.

## 6. Chuẩn bị RWF-2000

Dữ liệu không nằm trong Git repo. Dự án cần cấu trúc thư mục như sau:

    HTTM/
    └─ RWF-2000/
       ├─ train/
       │  ├─ Fight/       video bạo lực dùng để huấn luyện
       │  └─ NonFight/    video không bạo lực dùng để huấn luyện
       └─ val/
          ├─ Fight/       video bạo lực thuộc official val/test
          └─ NonFight/    video không bạo lực thuộc official val/test

Tên lớp Violence/NonViolence cũng được bộ tạo manifest nhận diện. Các định dạng video được quét gồm AVI, MP4, MOV, MKV và WEBM.

Trong lần chạy đã ghi nhận, kho dữ liệu có 1.600 clip official train và 400 clip official val/test. Script dùng 10% từ official train để tạo validation, chia đều theo lớp; 90% còn lại dùng train. Toàn bộ official val được giữ làm test. Manifest ghi đường dẫn video trên máy, vì vậy không nên commit manifest vào repo công khai.

Có bản RWF-2000 do người dùng khác đăng trên [Hugging Face](https://huggingface.co/datasets/DanJoshua/RWF-2000). Đây là bản mirror của bên thứ ba, không phải trang tải chính thức của nhóm tạo dữ liệu. Hãy tự xem thông tin nguồn và điều khoản sử dụng trước khi dùng. Không tải hoặc đưa video dữ liệu lên GitHub.

## 7. Tạo danh sách dữ liệu (manifest)

Từ thư mục gốc dự án, tạo thư mục output nếu chưa có:

    New-Item -ItemType Directory -Force outputs
    python -m src.build_manifest

Kết quả thành công có dạng:

    Wrote ...\outputs\split_manifest.json
    train: 1440 clips
    val:    160 clips
    test:   400 clips

Manifest nằm trong outputs/split_manifest.json. Có thể dùng thêm --validate để thử giải mã từng clip trước khi train; thao tác này chậm hơn:

    python -m src.build_manifest --validate

## 8. Huấn luyện

### Chạy thử ngắn

    python scripts/train.py --epochs 2 --max-batches 10

Đây chỉ là kiểm tra luồng dữ liệu/model/GPU, **không phải huấn luyện hoàn chỉnh**. Vì cấu hình đóng băng backbone trong 3 epoch đầu, lượt thử 2 epoch chưa fine-tune backbone.

### Huấn luyện đầy đủ

    python scripts/train.py

Cấu hình hiện tại đặt tối đa 20 epoch, batch size 4, mixed precision (AMP), đóng băng backbone 3 epoch đầu, sau đó mở backbone để fine-tune. Mỗi epoch đánh giá trên validation. Nếu F1 không cải thiện trong 5 epoch liên tiếp, early stopping kết thúc sớm. Checkpoint tốt nhất được lưu tại:

    outputs/checkpoints/best.pt

Log huấn luyện được ghi vào outputs/logs/train_log.csv. Không đóng terminal hoặc để máy sleep khi train còn chạy. Trên GPU 4 GB, nếu gặp lỗi hết VRAM thì sửa train.batch_size trong configs/config.yaml từ 4 xuống 2 rồi chạy lại.

### Kết quả huấn luyện đã tái hiện

Trong lần chạy trên RTX 3050 Ti Laptop 4 GB:

- Dừng sớm tại epoch 11/20.
- Checkpoint tốt nhất ở epoch 6.
- F1 validation tại epoch tốt nhất: 0,8831.

Đây là số liệu của lần chạy cụ thể này; kết quả có thể thay đổi theo dữ liệu, phiên bản thư viện và môi trường.

## 9. Đánh giá trên test

Sau khi có outputs/checkpoints/best.pt, chạy:

    python scripts/evaluate.py

Script dùng checkpoint tốt nhất, tính điểm validation để chọn threshold rồi báo cáo hiệu năng trên official test. Mỗi clip được đánh giá qua 5 đoạn lấy mẫu; xác suất các đoạn được trung bình. Kết quả được ghi vào outputs/eval_metrics.json.

### Kết quả đã ghi nhận

| Ngưỡng | Accuracy | Precision | Recall | F1 | AUC | False-positive rate |
|---:|---:|---:|---:|---:|---:|---:|
| 0,50 | 0,8450 | 0,8966 | 0,7800 | 0,8342 | 0,9079 | 0,0900 |
| 0,45 (chọn trên validation) | 0,8450 | 0,8833 | 0,7950 | 0,8368 | 0,9079 | 0,1050 |

Ở ngưỡng 0,50, confusion matrix là [[182, 18], [44, 156]]: 18 báo nhầm video không bạo lực và bỏ sót 44 video bạo lực. Ngưỡng 0,45 nhận ra thêm 3 video bạo lực nhưng báo nhầm thêm 3 video không bạo lực. Ngưỡng thấp hơn thường nhạy hơn nhưng có thể tạo thêm báo động giả.

- **Accuracy**: tỷ lệ dự đoán đúng trên toàn bộ clip.
- **Precision**: trong các clip bị dự đoán bạo lực, tỷ lệ thực sự bạo lực.
- **Recall**: trong các clip bạo lực, tỷ lệ được mô hình phát hiện.
- **F1**: trung bình điều hòa giữa precision và recall.
- **AUC**: khả năng xếp hạng clip bạo lực cao hơn clip không bạo lực trên nhiều ngưỡng.
- **FPR**: tỷ lệ clip không bạo lực bị báo nhầm là bạo lực.

Các số trên là kết quả thực tế được sao lại từ lần chạy của dự án, không phải bảo đảm hiệu năng cho mọi camera hoặc môi trường triển khai.

## 10. Chạy demo trên file AVI/MP4

Chạy từ thư mục gốc dự án, truyền đường dẫn video trong dấu ngoặc kép:

    python scripts/infer.py --source "C:\duong-dan\video-cua-ban.avi"

MP4 cũng chạy cùng lệnh:

    python scripts/infer.py --source "C:\duong-dan\video-cua-ban.mp4"

Cửa sổ demo hiển thị điểm bạo lực, điểm đã làm mượt và trạng thái gate. Nhấn q khi cửa sổ video đang được chọn để thoát. Sau khi file chạy hết, chương trình in CLIP VERDICT với xác suất tổng hợp trên nhiều đoạn clip; video dài chỉ lấy tối đa khoảng 40 giây đầu cho verdict này.

Muốn lưu video có overlay:

    python scripts/infer.py --source "C:\duong-dan\video-cua-ban.mp4" --save outputs\demo_annotated.mp4

Muốn chạy không mở cửa sổ:

    python scripts/infer.py --source "C:\duong-dan\video-cua-ban.mp4" --no-display

## 11. Chạy demo webcam

    python scripts/infer.py --source webcam

Nếu máy có nhiều camera, có thể thử chỉ số thiết bị, ví dụ:

    python scripts/infer.py --source 0

Nhấn q trong cửa sổ demo để dừng. Webcam là luồng theo thời gian thực: mỗi lần dự đoán chỉ nhìn một cửa sổ video, nên điểm dự đoán có thể dao động.

## 12. Model và temporal confidence gate

Model phân loại đưa ra xác suất P(Violence) cho từng cửa sổ. Một cửa sổ đơn lẻ có thể chấm điểm cao do chuyển động mạnh nhưng không phải đánh nhau. Temporal gate xử lý tín hiệu theo chuỗi:

1. Làm mượt điểm bằng EMA hoặc trung bình.
2. Chỉ xác nhận khi điểm đã làm mượt vượt gate.confirm_threshold đủ số cửa sổ liên tiếp.
3. Sau khi xác nhận, áp dụng gate.cooldown_seconds để không phát liên tiếp nhiều cảnh báo cho cùng một sự kiện.

Các giá trị này nằm trong configs/config.yaml. Ngưỡng dùng để tô nhãn/video-level verdict (--threshold) và ngưỡng xác nhận sự kiện của gate (gate.confirm_threshold) là **hai ngưỡng riêng**. Thay một ngưỡng không tự động thay đổi ngưỡng còn lại.

Input-quality gate của demo bỏ qua cửa sổ quá tối, quá đồng nhất hoặc gần như đứng hình. Bộ lọc này nhằm tránh đưa tín hiệu không giống footage thật vào gate; nó không thay thế việc kiểm tra độ chính xác trên dữ liệu phù hợp.

Nếu clip bạo lực không kích hoạt gate, chương trình vẫn in verdict tổng hợp cuối clip. Cửa sổ streaming mặc định là 3 giây; có thể thử 4 giây:

    python scripts/infer.py --source "C:\duong-dan\video.mp4" --window-seconds 4

## 13. Cảnh báo email

Cấu hình hiện có alert.enabled: true. Email chỉ có thể gửi nếu có đủ thông tin xác thực trong file .env ở thư mục gốc:

    EMAIL_USER=dia_chi_gmail
    EMAIL_APP_PASSWORD=mat_khau_ung_dung_google
    ALERT_RECIPIENT=dia_chi_nhan

Gmail yêu cầu App Password; không dùng mật khẩu tài khoản Gmail thông thường. Không commit hoặc chia sẻ .env. Nếu chỉ muốn demo mà không gửi email, đặt alert.enabled: false trong configs/config.yaml.

Khi gate xác nhận sự kiện, chương trình có thể lưu ảnh chụp tại outputs/snapshots/, ghi log tại outputs/logs/inference_events.csv và outputs/logs/alerts_audit.csv. Gửi email thử bằng python -m src.alerter --test sẽ gửi một email thật nếu thông tin xác thực hợp lệ; chỉ chạy khi bạn thực sự muốn kiểm tra gửi email.

Cảnh báo tự động ở đây chỉ là proof-of-concept. Hãy có người xác minh trước mọi hành động ngoài đời thực.

## 14. Các lệnh hữu ích

    # Kiểm tra môi trường/GPU
    python -m src.check_env

    # Tạo lại manifest sau khi đổi vị trí dữ liệu
    New-Item -ItemType Directory -Force outputs
    python -m src.build_manifest

    # Chạy thử pipeline train ngắn
    python scripts/train.py --epochs 2 --max-batches 10

    # Huấn luyện đầy đủ
    python scripts/train.py

    # Đánh giá held-out test
    python scripts/evaluate.py

    # Demo video (AVI hoặc MP4)
    python scripts/infer.py --source "C:\duong-dan\video.mp4"

    # Demo webcam
    python scripts/infer.py --source webcam

    # Lưu video demo có overlay
    python scripts/infer.py --source "C:\duong-dan\video.mp4" --save outputs\demo_annotated.mp4

## 15. Lỗi thường gặp

### Không tìm thấy outputs\split_manifest.json

Tạo thư mục output rồi tạo manifest lại:

    New-Item -ItemType Directory -Force outputs
    python -m src.build_manifest

### CUDA available: False

Kiểm tra PyTorch CUDA và driver bằng python -m src.check_env. Máy có thể chạy trên CPU, nhưng train sẽ chậm; cần cài phiên bản PyTorch phù hợp với GPU/driver.

### Hết bộ nhớ GPU

Đổi train.batch_size: 4 thành train.batch_size: 2 trong configs/config.yaml. Giữ train.amp: true.

### Không tìm thấy checkpoint

Kiểm tra file outputs/checkpoints/best.pt. Nếu chưa có, train trước bằng python scripts/train.py.

### Không mở được video

Kiểm tra đường dẫn trong dấu ngoặc kép, thử file AVI/MP4 khác, và chắc chắn bạn đang chạy lệnh từ thư mục gốc dự án.

## 16. Trước khi đẩy dự án lên GitHub

Repo loại .venv/, .env, RWF-2000/, checkpoint .pt/.pth và video .avi/.mp4. outputs/ bị ignore mặc định; riêng outputs/eval_metrics.json và outputs/logs/train_log.csv được đưa lên sau khi rà soát vì chỉ chứa chỉ số và log epoch. Manifest split_manifest.json, log có đường dẫn video/cảnh báo, snapshots và checkpoint vẫn để local.

- Không ép add dữ liệu, video, checkpoint hoặc bí mật vào Git.
- Chỉ đăng mã nguồn, cấu hình không chứa bí mật và tài liệu.
- Ghi rõ cách lấy dữ liệu và điều khoản của nguồn dữ liệu; bản Hugging Face nói trên là mirror của bên thứ ba.
- Các số liệu trong tài liệu này là lần chạy của người viết; người khác cần chạy lại để xác nhận trên môi trường của họ.
- README gốc có thể chứa kết quả/cấu hình tham khảo khác với lần chạy này. Trước khi công khai, hãy đối chiếu và thống nhất README với kết quả thực tế của bạn.

---

**Checkpoint và dữ liệu không đi kèm file này.** Người dùng cần tự chuẩn bị RWF-2000 và tự huấn luyện hoặc có checkpoint tương thích trước khi chạy demo.

