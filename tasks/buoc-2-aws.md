# Bước 2 - Pipeline CI/CD Tự Động (Phiên Bản AWS)

Đây là hướng dẫn thực hiện **Bước 2** dùng **AWS** thay cho GCP.
Ánh xạ khái niệm:

| Khái niệm | GCP (mặc định) | AWS (dùng ở đây) |
|---|---|---|
| Object Storage | Google Cloud Storage | Amazon S3 |
| VM | Compute Engine | Amazon EC2 |
| CLI | `gcloud` / `gsutil` | `aws` |
| DVC extra | `dvc[gs]` | `dvc[s3]` |
| Cloud SDK Python | `google-cloud-storage` | `boto3` |
| Credentials | Service Account JSON | IAM User Access Key |

Các file trong repo đã được chỉnh sẵn cho AWS:
- `requirements.txt` → `dvc[s3]`, `boto3`
- `src/serve.py` → tải model từ S3 bằng `boto3`
- `.github/workflows/cicd.yml` → xác thực AWS, `dvc pull`, upload S3, deploy EC2

---

## Chuẩn Bị: Cài Và Cấu Hình AWS CLI

```bash
aws --version          # kiểm tra đã cài chưa
aws configure          # nhập Access Key, Secret Key, region (vd: ap-southeast-1)
```

Đặt biến dùng lại cho các lệnh bên dưới. **Chọn đúng shell bạn đang dùng.**

> QUAN TRỌNG (Windows): Nếu bạn dùng **PowerShell** hay **cmd.exe**, cú pháp bash
> `export BUCKET=...` và `$BUCKET` KHÔNG hoạt động — kết quả là `$BUCKET` bị truyền
> nguyên văn và `aws s3 mb` báo lỗi "Invalid bucket name". Hãy dùng phần PowerShell
> dưới đây, hoặc đơn giản là thay thẳng tên bucket thật vào lệnh.

PowerShell (Windows):

```powershell
$BUCKET  = "income-lab-toan-2026"     # tên bucket duy nhất (chỉ chữ thường, số, dấu -)
$REGION  = "ap-southeast-1"           # Singapore, gần VN
$KEYPAIR = "income-key"               # tên EC2 key pair
```

Bash (Linux / macOS / Git Bash / WSL):

```bash
export BUCKET=income-lab-toan-2026    # tên bucket duy nhất
export REGION=ap-southeast-1          # Singapore, gần VN
export KEYPAIR=income-key             # tên EC2 key pair
```

Ghi chú: các lệnh `aws` bên dưới viết theo cú pháp bash (`$BUCKET`). Trên PowerShell,
biến `$BUCKET` vẫn được thay đúng, nhưng với chuỗi có `://` nên bọc trong dấu nháy kép,
ví dụ: `aws s3 mb "s3://$BUCKET" --region $REGION`. Nếu vẫn gặp lỗi biến, hãy thay
thẳng tên bucket thật vào lệnh.


---

## 2.1 Tạo S3 Bucket

Tên bucket phải **duy nhất toàn cầu**.

```bash
aws s3 mb s3://$BUCKET --region $REGION
```

Chặn truy cập public (khuyến nghị, bảo mật):

```bash
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Kiểm tra:

```bash
aws s3 ls | grep $BUCKET
```

---

## 2.2 Tạo IAM User + Access Key (Quyền Tối Thiểu)

Tạo IAM user riêng cho lab, chỉ cấp quyền đọc/ghi trên đúng bucket này (nguyên tắc quyền tối thiểu).

```bash
# 1. Tạo user
aws iam create-user --user-name income-lab-user

# 2. Tạo policy chỉ cho phép thao tác trên bucket của bạn
cat > income-lab-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::$BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::$BUCKET/*"
    }
  ]
}
EOF

# 3. Gắn policy vào user (inline policy)
aws iam put-user-policy \
  --user-name income-lab-user \
  --policy-name income-lab-s3 \
  --policy-document file://income-lab-policy.json

# 4. Tạo access key - LƯU LẠI output (chỉ hiển thị 1 lần)
aws iam create-access-key --user-name income-lab-user
```

Output có `AccessKeyId` và `SecretAccessKey`. Ghi lại — sẽ dùng cho DVC (local) và GitHub Secrets.

Cấu hình local để DVC dùng được (một trong hai cách):

```bash
# Cách A: dùng profile mặc định
aws configure   # nhập AccessKeyId + SecretAccessKey của income-lab-user

# Cách B: export biến môi trường trong session hiện tại
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_DEFAULT_REGION=$REGION
```

Lưu ý: KHÔNG commit file access key / policy JSON chứa secret vào git.

---

## 2.3 Cài Đặt DVC Với S3 Remote

```bash
dvc init

# Trỏ DVC vào S3
dvc remote add -d labstore s3://$BUCKET/dvc

# (tuỳ chọn) chỉ định region cho remote
dvc remote modify labstore region $REGION

# DVC tự đọc credentials từ ~/.aws/credentials hoặc biến môi trường
# AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY. KHÔNG cần credentialpath như GCP.

# Theo dõi các file dữ liệu
dvc add data/train_batch1.csv
dvc add data/holdout.csv
dvc add data/train_batch2.csv

# Commit các file con trỏ .dvc (KHÔNG commit file CSV)
git add data/train_batch1.csv.dvc data/holdout.csv.dvc data/train_batch2.csv.dvc \
        .gitignore .dvc/config
git commit -m "feat: track datasets with DVC (S3 remote)"

# Đẩy dữ liệu lên S3
dvc push
```

Xác nhận trên S3:

```bash
aws s3 ls s3://$BUCKET/dvc/ --recursive
```

---

## 2.4 Tạo EC2 Instance

Tạo key pair để SSH vào EC2:

```bash
aws ec2 create-key-pair --key-name $KEYPAIR \
  --query 'KeyMaterial' --output text > ~/.ssh/$KEYPAIR.pem
chmod 400 ~/.ssh/$KEYPAIR.pem
```

Tạo security group và mở cổng 22 (SSH) + 8080 (API):

```bash
# Tạo security group
SG_ID=$(aws ec2 create-security-group \
  --group-name income-api-sg \
  --description "Income API SG" \
  --query 'GroupId' --output text)
echo $SG_ID

# Mở cổng 22 (SSH) va 8080 (inference API)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

Lưu ý bảo mật: mở cổng 22/8080 cho `0.0.0.0/0` chỉ nên dùng cho lab. Trong thực tế nên giới hạn CIDR về IP của bạn.

Lấy AMI Ubuntu 22.04 mới nhất và tạo instance:

```bash
# Lay AMI Ubuntu 22.04 LTS (x86_64) moi nhat
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' --output text)
echo $AMI_ID

# Tao instance t3.small
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.small \
  --key-name $KEYPAIR \
  --security-group-ids $SG_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=income-api}]' \
  --query 'Instances[0].InstanceId' --output text)
echo $INSTANCE_ID

# Cho instance chay
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Lay IP cong khai (LUU LAI - dung cho GitHub Secrets SERVER_HOST)
VM_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "VM_IP=$VM_IP"
```

---

## 2.5 Cấu Hình EC2 (Thực Hiện Một Lần)

SSH vào EC2 (user mặc định của Ubuntu AMI là `ubuntu`):

```bash
ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP
```

Bên trong EC2, cài thư viện:

```bash
sudo apt update && sudo apt install -y python3-pip
pip3 install fastapi uvicorn scikit-learn joblib boto3

mkdir -p ~/models ~/src
echo $USER          # ghi lai -> dung cho GitHub Secret SERVER_USER (thuong la "ubuntu")
exit
```

Xác thực S3 trên EC2 — chọn MỘT trong hai cách:

Cách A (khuyến nghị): gán **IAM Role** cho EC2 để không phải lưu key trên máy.

> LƯU Ý (nguyên nhân 2 lỗi thường gặp):
> 1. `Unable to load paramfile file://income-lab-policy.json` → file policy không nằm
>    trong thư mục hiện tại. Vì vậy phần dưới **tạo lại** cả hai file JSON ngay tại đây,
>    không phụ thuộc bước 2.2.
> 2. `Invalid IAM Instance Profile name` khi `associate` → instance profile vừa tạo cần
>    vài giây để lan truyền (propagation). Thêm `sleep`/`Start-Sleep` trước khi gắn.
> Ngoài ra kiểm tra `$INSTANCE_ID` đã có giá trị (`echo $INSTANCE_ID`); trên PowerShell
> phải set bằng `$INSTANCE_ID = "..."` chứ không phải `export`.

Bash (Linux / macOS / Git Bash / WSL):

```bash
# Tao lai policy S3 (dung chung cho user va role)
cat > income-lab-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::$BUCKET"},
    {"Effect": "Allow", "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"], "Resource": "arn:aws:s3:::$BUCKET/*"}
  ]
}
EOF

# Trust policy cho EC2
cat > ec2-trust.json <<EOF
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

aws iam create-role --role-name income-ec2-role \
  --assume-role-policy-document file://ec2-trust.json

aws iam put-role-policy --role-name income-ec2-role \
  --policy-name income-ec2-s3 \
  --policy-document file://income-lab-policy.json

aws iam create-instance-profile --instance-profile-name income-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name income-ec2-profile --role-name income-ec2-role

# Cho instance profile lan truyen truoc khi gan (tranh loi "Invalid IAM Instance Profile name")
sleep 15

aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=income-ec2-profile
```

PowerShell (Windows) — nếu `create-instance-profile`/`add-role-to-instance-profile` ở trên đã chạy xong, chỉ cần chờ rồi gắn:

```powershell
# Tao lai 2 file JSON tai thu muc hien tai
@'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}
'@ | Set-Content -Encoding ascii ec2-trust.json

@"
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::$BUCKET"},
    {"Effect": "Allow", "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"], "Resource": "arn:aws:s3:::$BUCKET/*"}
  ]
}
"@ | Set-Content -Encoding ascii income-lab-policy.json

# Neu role chua co policy (do lenh truoc bao loi thieu file), chay lai:
aws iam put-role-policy --role-name income-ec2-role `
  --policy-name income-ec2-s3 `
  --policy-document file://income-lab-policy.json

# Cho lan truyen roi gan (INSTANCE_ID phai da set: $INSTANCE_ID = "i-...")
Start-Sleep -Seconds 15
aws ec2 associate-iam-instance-profile `
  --instance-id $INSTANCE_ID `
  --iam-instance-profile Name=income-ec2-profile
```

Nếu vẫn báo `Invalid IAM Instance Profile name`, xác nhận profile đã có role rồi thử lại:

```bash
aws iam get-instance-profile --instance-profile-name income-ec2-profile
# Phai thay "Roles" chua income-ec2-role. Neu rong, chay lai add-role-to-instance-profile.
```

Cách B (đơn giản hơn nhưng kém an toàn): copy access key lên EC2 qua `aws configure` trong session SSH. Nếu dùng cách này, thêm hai dòng `Environment=` cho AWS key vào systemd service ở bước 2.7.


Hướng dẫn này dùng **Cách A (IAM Role)** nên `serve.py` chỉ cần gọi `boto3.client("s3")` là tự xác thực.

---

## 2.6 `src/serve.py` (Đã Viết Sẵn Cho AWS)

File `src/serve.py` trong repo đã dùng `boto3` để tải model từ S3. Không cần sửa. Upload lên EC2:

```bash
scp -i ~/.ssh/$KEYPAIR.pem src/serve.py ubuntu@$VM_IP:~/src/serve.py
```

Điểm khác so với GCP: dùng `boto3.client("s3")` + `s3.download_file(bucket, key, path)` thay cho `storage.Client()`.

---

## 2.7 Cấu Hình Systemd Service Trên EC2

SSH lại vào EC2:

```bash
ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP
```

Tạo service (dùng IAM Role nên KHÔNG cần khai báo AWS key):

```bash
sudo tee /etc/systemd/system/income-api.service > /dev/null <<EOF
[Unit]
Description=Income Model Inference Server
After=network.target

[Service]
User=$USER
WorkingDirectory=/home/$USER
Environment="ARTIFACT_BUCKET=<TEN_BUCKET_CUA_BAN>"
Environment="AWS_DEFAULT_REGION=ap-southeast-1"
ExecStart=/usr/bin/python3 /home/$USER/src/serve.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable income-api
```

Thay `<TEN_BUCKET_CUA_BAN>` bằng tên bucket thật. Nếu dùng **Cách B** (access key), thêm:

```
Environment="AWS_ACCESS_KEY_ID=..."
Environment="AWS_SECRET_ACCESS_KEY=..."
```

Chưa cần start service — model chưa có trên S3 cho tới khi pipeline chạy lần đầu. Thoát EC2 (`exit`).

---

## 2.8 Tạo SSH Key Cho GitHub Actions Deploy

Chạy trên máy cá nhân:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/income_deploy -N "" -C "github-actions-deploy"
```

Thêm public key vào EC2:

```bash
ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP \
  "echo '$(cat ~/.ssh/income_deploy.pub)' >> ~/.ssh/authorized_keys"
```

---

## 2.9 Thêm GitHub Secrets

Repo GitHub: Settings > Secrets and variables > Actions > New repository secret.

| Tên secret | Giá trị (AWS) |
|---|---|
| `STORAGE_CREDENTIALS` | JSON: `{"aws_access_key_id":"AKIA...","aws_secret_access_key":"...","region":"ap-southeast-1"}` |
| `ARTIFACT_BUCKET` | Tên bucket (vd: `income-lab-toan-2026`) |
| `SERVER_HOST` | IP công khai của EC2 (`$VM_IP` ở bước 2.4) |
| `SERVER_USER` | `ubuntu` |
| `SERVER_SSH_KEY` | Toàn bộ nội dung `~/.ssh/income_deploy` (private key) |

Kiểm tra: không có khoảng trắng thừa đầu/cuối mỗi secret. `STORAGE_CREDENTIALS` phải là JSON hợp lệ (workflow parse bằng `json.loads`).

---

## 2.10 `tests/test_train.py`

File này đã có sẵn và không phụ thuộc cloud, giữ nguyên. Chạy thử local:

```bash
pytest tests/ -v
```

Cả 3 test phải qua trước khi push.

---

## 2.11 `.github/workflows/cicd.yml` (Đã Điền Cho AWS)

Workflow trong repo đã hoàn thiện cho AWS. Các điểm chính:

- **Authenticate**: đọc `STORAGE_CREDENTIALS` (JSON), set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION` vào `$GITHUB_ENV`.
- **Pull data**: `dvc pull data/train_batch1.csv.dvc data/holdout.csv.dvc`.
- **Upload model**: `boto3` upload `models/model.joblib` lên `s3://<bucket>/artifacts/current/model.joblib`.
- **Quality gate**: `float()` f1 rồi so sánh `>= 0.65`.
- **Release**: SSH vào EC2, `systemctl restart income-api`, kiểm tra `/healthz`.

---

## 2.12 Lần Chạy Pipeline Đầu Tiên

```bash
touch src/__init__.py tests/__init__.py

git add .
git commit -m "feat: add CI/CD pipeline, tests, and serving API (AWS)"
git push origin main
```

Theo dõi tab **Actions**. Sau khi pipeline xanh và model đã lên S3, start service trên EC2:

```bash
ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP "sudo systemctl start income-api"
```

Thử endpoint:

```bash
# Health
curl http://$VM_IP:8080/healthz

# Du doan mau 1
curl -X POST http://$VM_IP:8080/score \
  -H "Content-Type: application/json" \
  -d '{"features": [60, 2, 5, 2, 4, 0, 1, 0, 0, 45]}'
# Mong doi: {"prediction": 0, "label": "thu_nhap_thap"}

# Du doan mau 2 (hoc van cao hon)
curl -X POST http://$VM_IP:8080/score \
  -H "Content-Type: application/json" \
  -d '{"features": [28, 2, 14, 2, 11, 0, 1, 0, 0, 45]}'
# Mong doi: {"prediction": 1, "label": "thu_nhap_cao"}
```

---

## Xử Lý Sự Cố (AWS)

**`dvc push` lỗi xác thực (AccessDenied)**
- Kiểm tra `aws s3 ls s3://$BUCKET` chạy được không.
- Xác nhận IAM policy đã gắn đúng bucket ARN.
- Kiểm tra `~/.aws/credentials` hoặc biến `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.

**GitHub Actions không parse được `STORAGE_CREDENTIALS`**
- Phải là JSON hợp lệ với đúng 3 khóa: `aws_access_key_id`, `aws_secret_access_key`, `region`.

**Release fail dù f1 đủ cao**
- Output của GitHub Actions là chuỗi; workflow đã `float()` trước khi so sánh. Kiểm tra log job Train xem giá trị f1 in ra.

**Service EC2 không khởi động**
```bash
ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP "sudo journalctl -u income-api -n 50"
```
Nguyên nhân thường gặp: `ARTIFACT_BUCKET` sai; EC2 chưa có IAM Role/AWS key; model chưa có trên S3 (chạy pipeline trước).

**Không SSH được vào EC2**
- Kiểm tra security group đã mở cổng 22.
- Đúng key `.pem` và `chmod 400`.
- Đúng user `ubuntu`.

---

## Kết Quả Cần Đạt

- Bốn job Actions (Unit Test, Train, Quality Gate, Release) đều xanh.
- `curl http://$VM_IP:8080/healthz` trả `{"status": "ok"}`.
- `curl .../score` trả kết quả hợp lệ.
- S3 có `dvc/` và `artifacts/current/model.joblib`:
  ```bash
  aws s3 ls s3://$BUCKET/artifacts/current/
  ```

Ba ảnh nộp bài (lưu vào `nop-bai/anh-chup-man-hinh/`):

| Tên file | Nội dung |
|---|---|
| `02-actions-buoc-2.png` | Tab Actions với 4 job xanh |
| `04-curl-api.png` | Terminal chứa 2 lệnh `curl` + kết quả, thấy rõ IP EC2 |
| `05-cloud-storage.png` | S3 console hiện `dvc/` và `artifacts/current/model.joblib` |

---

Tiếp theo: [Bước 3 - Huấn luyện liên tục](buoc-3.md) (thay `gcloud compute ssh` bằng `ssh -i ~/.ssh/$KEYPAIR.pem ubuntu@$VM_IP`, còn lại giữ nguyên).
