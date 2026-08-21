# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

<!--
HƯỚNG DẪN - đọc rồi XÓA TOÀN BỘ các khối chú thích này sau khi điền xong:

  - Giới hạn: KHÔNG QUÁ 1 TRANG A4, tương đương khoảng 450 - 550 từ nội dung.
  - Chỉ điền vào các chỗ ___ và các ô trong bảng. Không thêm mục mới.
  - Viết bằng câu hoàn chỉnh, không gạch đầu dòng cụt lủn.
  - Kiểm tra độ dài sau khi đã xóa hết chú thích:
        wc -w nop-bai/bao-cao.md
    và xem trước bản in bằng cách mở file trên GitHub rồi Ctrl+P / Cmd+P.
-->

| | |
|---|---|
| Họ và tên | Trần Văn Toàn |
| MSSV | 2A202601218 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/toanvtran/TRACK2_Day21_2A202601218_TranVanToan |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.878 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.846 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.874 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Lần chạy 3 đạt f1_score cao nhất (0.7149) trên tập holdout, vượt xa lần chạy 2 (0.6051) và nhỉnh hơn lần chạy 1 (0.7109), nên đây là bộ được chọn vì lab đánh giá theo F1 của lớp dương chứ không theo accuracy. Đáng chú ý, lần có accuracy cao nhất lại là lần chạy 1 (0.878) chứ không phải lần có f1_score cao nhất; điều này cho thấy accuracy bị lớp đa số "thu nhập thấp" kéo lên và không phản ánh đúng khả năng bắt lớp dương của mô hình. Về đánh đổi giữa n_estimators và learning_rate: lần chạy 2 giảm cả learning_rate lẫn số cây nên mô hình học chưa đủ (f1 thấp), trong khi tăng n_estimators lên 200 kèm max_depth lớn hơn giúp bù lại và cải thiện F1, đúng với đặc tính cộng dồn của GradientBoosting.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult mất cân bằng: chỉ khoảng 24,8% số mẫu thuộc lớp thu nhập > 50K, còn lại 75,2% là thu nhập thấp. Hệ quả là một mô hình "luôn đoán thu nhập thấp" cho mọi mẫu vẫn đạt accuracy khoảng 0,752 dù không bắt được một trường hợp thu nhập cao nào. Con số accuracy cao đó gây hiểu nhầm vì nó chủ yếu phản ánh việc đoán đúng lớp đa số chứ không đo được năng lực thực sự của mô hình. F1 của lớp dương lại đo đồng thời precision và recall trên đúng lớp thu nhập cao mà ta quan tâm, nên phạt nặng mô hình bỏ sót hoặc báo nhầm lớp này — điều accuracy không làm được. Vì vậy khi gọi f1_score ta để mặc định tính cho lớp dương, KHÔNG dùng average="weighted" hay "macro", vì các giá trị đó bị lớp đa số kéo lên cao và làm mất ý nghĩa của ngưỡng 0,65.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

<!-- Nêu 2 - 3 khó khăn thật, mỗi ô một câu ngắn. -->

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Unit Test trên GitHub Actions lỗi `PermissionError: /C:` | Thư mục `mlruns/` bị commit kèm `meta.yaml` chứa đường dẫn Windows `file:///C:/...`, runner Ubuntu không ghi được vào `/C:` | Thêm `mlruns/` vào `.gitignore`, `git rm -r --cached mlruns/` rồi push lại để runner tự tạo store mới |
| Service trên EC2 crash-loop, `curl :8080/healthz` không kết nối được | Model huấn luyện bằng scikit-learn 1.4.2 nhưng EC2 cài sẵn 1.7.2, `joblib.load()` lỗi unpickle | Cài đúng phiên bản `scikit-learn==1.4.2`, `joblib==1.4.2`, `numpy<2` trên EC2 rồi restart service |
| Lệnh `curl -d '{...}'` báo `JSON decode error` trên PowerShell | PowerShell hiểu nháy đơn khác bash nên JSON bị hỏng | Dùng Git Bash với nháy đơn, hoặc `curl --data-binary "@file.json"` |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

<!-- Lấy số liệu từ bảng ở mục 3.6 của tasks/buoc-3.md. -->

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`, 22.361 mẫu) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`, 44.722 mẫu) | 0.7354 | 0.8820 |

**Nhận xét:** Sau khi gấp đôi dữ liệu, f1_score tăng nhẹ khoảng 0,02 (0,7149 → 0,7354) và accuracy tăng khoảng 0,008 (0,874 → 0,882). Vì hai nửa dữ liệu được chia ngẫu nhiên từ cùng một nguồn nên cùng phân phối, phần dữ liệu mới gần như không mang thêm thông tin mới; mức tăng nhỏ này chủ yếu do mô hình ước lượng ổn định hơn trên tập lớn hơn (giảm phương sai) chứ không phải học được mẫu mới. Điều quan trọng được kiểm chứng ở Bước 3 là quy trình tự động chạy đúng: dữ liệu mới đi trọn vòng từ commit → DVC → huấn luyện lại → quality gate → triển khai mà không cần thao tác thủ công.

<!--
Một câu trả lời trung thực kiểu "f1 giảm 0,01 vì dữ liệu mới cùng phân phối, không mang
thêm thông tin mới" được đánh giá cao hơn kết luận sai rằng thêm dữ liệu luôn tốt hơn.
-->

---

## 5. Phần Bonus Đã Thực Hiện (nếu có)

<!-- Xóa cả mục 5 nếu không làm bonus. Mỗi bonus tối đa 1 dòng. -->

- [ ] Bonus 1 - Tracking MLflow từ xa với DagsHub: ___
- [ ] Bonus 2 - Điều chỉnh ngưỡng quyết định: ___
- [ ] Bonus 3 - Báo cáo precision / recall tự động: ___
- [ ] Bonus 4 - Hoàn trả về phiên bản trước: ___
- [ ] Bonus 5 - Cảnh báo lệch lạc dữ liệu: ___
