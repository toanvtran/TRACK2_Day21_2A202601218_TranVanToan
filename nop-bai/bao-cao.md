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
| Họ và tên | ___ |
| MSSV | ___ |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/___/___ |
| Ngày nộp | ___ |

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
| ___ | ___ | ___ |
| ___ | ___ | ___ |
| ___ | ___ | ___ |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

<!-- Lấy số liệu từ bảng ở mục 3.6 của tasks/buoc-3.md. -->

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | ___ | ___ |
| Bước 3 (thêm `train_batch2`) | ___ | ___ |

**Nhận xét:** ___

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
