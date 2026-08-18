# PTXLA v2.0 — Skin Lesion Classification trên ISIC 2019

## Ghi chú về "v2.0"

Repo này chỉ chứa bản rebuild cuối cùng (`ptxla.ipynb`) và bản đối chứng 
không tiền xử lý dùng cho ablation (`ptxla_raw.ipynb`). Bản gốc từng report 
accuracy ~73% là notebook cá nhân cũ (Google Colab/Kaggle), không được lưu 
lại trong repo này nên không có gì để link ra làm bằng chứng — mọi thứ dưới 
đây về "bản gốc" là mô tả lại từ quá trình debug, không phải thứ bạn có thể 
tự kiểm tra bằng cách đọc code cũ. Cái verify được là: kết quả và số liệu 
trong `ptxla.ipynb`/`ptxla_raw.ipynb` hiện tại, chạy được và tái lập lại được.

## Vì sao con số 73% ban đầu không dùng nữa

Khi quay lại để đưa vào portfolio, con số 73% không tái lập được. Lần theo 
lại logic cũ (không phải diff code, vì bản cũ không còn), nhận ra pipeline 
gốc có vấn đề ở việc tiền xử lý ảnh, cách implement attention, và một vài 
lỗi biến/thứ tự cell khiến kết quả không đáng tin. Quyết định rebuild lại 
từ đầu thay vì cố sửa vá, và lần này viết theo hướng mọi bước đều kiểm 
chứng được ngay trong chính notebook đang có ở đây.

## Những gì áp dụng trong bản hiện tại (và lý do)

| Kỹ thuật | Lý do chọn |
|---|---|
| `GroupShuffleSplit` theo `lesion_id` | Tránh leak ảnh cùng 1 lesion giữa train/val |
| Focal Loss | NV chiếm gần 50x lớp hiếm nhất, `class_weight='balanced'` không đủ |
| `val_macro_f1` làm metric theo dõi (MacroMetricsCallback) | Accuracy bị NV chi phối, che mất việc model học lớp hiếm tệ |
| `vertical_flip=True` trong augmentation | Tăng đa dạng dữ liệu ảnh dermoscopy |
| Ngưỡng an toàn trong Dull Razor + CLAHE | Tránh xóa nhầm vùng tổn thương tối màu khi khử lông |
| Set seed cho phần test | Đảm bảo kết quả test tái lập được giữa các lần chạy |

*(Bảng này mô tả bản hiện tại, không phải diff so với code cũ — vì code cũ không còn để so sánh trực tiếp.)*

## Ablation A/B: tiền xử lý ảnh có thật sự giúp ích?

Đây là phần verify được 100% từ 2 notebook trong repo: giữ nguyên toàn bộ 
pipeline, chỉ đổi biến tiền xử lý ảnh.

| | Processed (`ptxla.ipynb`) | Raw (`ptxla_raw.ipynb`) |
|---|---|---|
| Macro-F1 (closed-set) | **0.42** | 0.36 |
| AUROC (biết vs UNK) | 0.64 | 0.67* |

\* AUROC raw cao hơn nhưng là ảo giác — model raw kém tự tin đều trên mọi 
lớp nên dễ bị threshold quét nhầm thành UNK, không phải vì nó phân biệt UNK 
tốt hơn thật sự. Tiền xử lý giúp rõ nhất ở các lớp hiếm (DF, VASC, SCC, BKL).

## Model có học ổn không?

Train/val accuracy và loss bám sát nhau suốt training, val macro-F1 tăng đều 
0.30 → 0.44 (đỉnh epoch 35). Val (0.44) vs Test (0.42) chênh nhỏ — không có 
domain shift nghiêm trọng. Vấn đề không nằm ở cách train mà là trần giới hạn 
từ mất cân bằng dữ liệu gốc.

## Threshold UNK — 3 chính sách cost-sensitive

Sweep cost ratio trên tập calibration, áp 1 lần duy nhất lên tập eval tách 
biệt (50/50), tránh leak thông tin giữa bước chọn threshold và bước đánh giá.

| Chính sách | Threshold | UNK recall | Recall TB nhóm ưu tiên (MEL/BCC/AK/SCC) |
|---|---|---|---|
| F1-max | 0.353 | 0.38 | 0.30 |
| Recall-max | 0.170 | 0.00 (suy biến) | 0.42 |
| **Cân bằng (khuyến nghị)** | 0.235 | 0.05 | 0.41 |

Chính sách Cân bằng đạt recall nhóm ưu tiên gần bằng Recall-max (0.41 vs 
0.42) nhưng vẫn giữ được khả năng nhận UNK — đánh đổi có chủ đích, ưu tiên 
phát hiện bệnh ác tính.

> Accuracy không dùng làm metric chính vì bị chi phối bởi lớp lớn (NV, và 
> UNK chiếm ~25% eval set), che khuất việc các policy mới cải thiện đúng chỗ 
> quan trọng dù accuracy tổng không tăng.

## Cấu trúc repo

- `ptxla.ipynb` — notebook chính, dữ liệu qua tiền xử lý (Dull Razor + CLAHE)
- `ptxla_raw.ipynb` — bản đối chứng không tiền xử lý, dùng riêng cho ablation study

## Hạn chế

- Seed mới áp cho lúc test, chưa fix seed lúc train — số liệu có thể xê dịch 
  nhẹ giữa các lần chạy lại
- Chưa nâng cấp attention (SE/CBAM) và metadata fusion
