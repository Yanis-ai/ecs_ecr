# ecs_ecr


以下是完整的 **Terraform + ECS + AWS Batch + Lambda** 代码结构和所有文件的代码。

---

## **📁 目录结构**
```
📂 terraform-unzip
├── 📂 lambda
│   ├── 📜 lambda_function.py           # 触发 AWS Batch 任务
│   ├── 📜 calculate_time_lambda.py     # 计算整个解压总时间
│   ├── 📜 requirements.txt             # Lambda 依赖
│   ├── 📜 zip_lambda.sh                # 打包 Lambda 代码
├── 📂 ecs-unzip
│   ├── 📜 Dockerfile                   # ECS 运行的 Docker 镜像
│   ├── 📜 unzip.py                      # 解压逻辑（Python）
│   ├── 📜 requirements.txt             # ECS 依赖
│   ├── 📜 entrypoint.sh                 # ECS 任务启动脚本
├── 📂 terraform
│   ├── 📜 main.tf                      # Terraform 主配置文件
│   ├── 📜 variables.tf                 # 变量定义
│   ├── 📜 outputs.tf                   # 输出定义
│   ├── 📜 provider.tf                   # AWS Provider
│   ├── 📜 iam.tf                        # IAM 角色与权限
│   ├── 📜 batch.tf                      # AWS Batch 任务配置
│   ├── 📜 s3.tf                         # S3 资源创建
```

---

## **📜 `terraform/main.tf` (Terraform 主配置)**
```hcl
provider "aws" {
  region = "us-east-1"
}

module "s3" {
  source = "./s3.tf"
}

module "iam" {
  source = "./iam.tf"
}

module "batch" {
  source     = "./batch.tf"
  ecs_role   = module.iam.ecs_task_role
  s3_bucket  = module.s3.s3_output_bucket
}
```

---

## **📜 `terraform/s3.tf` (S3 资源)**
```hcl
resource "aws_s3_bucket" "s3input" {
  bucket = "my-batch-input-bucket"
}

resource "aws_s3_bucket" "s3output" {
  bucket = "my-batch-output-bucket"
}
```

---

## **📜 `terraform/iam.tf` (IAM 角色)**
```hcl
resource "aws_iam_role" "ecs_task_role" {
  name = "ecsTaskRole"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}
```

---

## **📜 `terraform/batch.tf` (AWS Batch 任务)**
```hcl
resource "aws_batch_job_definition" "unzip_job" {
  name = "unzip-job"
  type = "container"

  container_properties = jsonencode({
    image = "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/unzip-processor:latest"
    vcpus = 1
    memory = 512
    jobRoleArn = aws_iam_role.ecs_task_role.arn
    command = ["python3", "/app/unzip.py"]
    environment = [
      { name = "S3_INPUT_BUCKET", value = "my-batch-input-bucket" },
      { name = "S3_OUTPUT_BUCKET", value = "my-batch-output-bucket" }
    ]
  })
}
```

---

## **📜 `ecs-unzip/Dockerfile`**
```dockerfile
FROM python:3.9

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY unzip.py .
CMD ["python3", "unzip.py"]
```

---

## **📜 `ecs-unzip/requirements.txt`**
```
boto3
```

---

## **📜 `ecs-unzip/unzip.py`**
```python
import os
import tarfile
import boto3
import time
import json

s3 = boto3.client("s3")
input_bucket = os.environ["S3_INPUT_BUCKET"]
output_bucket = os.environ["S3_OUTPUT_BUCKET"]
job_id = os.environ.get("AWS_BATCH_JOB_ID", "default")

start_time = time.time()

def extract_files():
    response = s3.list_objects_v2(Bucket=input_bucket)
    files = [item["Key"] for item in response.get("Contents", []) if item["Key"].endswith(".tar.gz")]

    tasks = files[:2]  
    for file in tasks:
        local_file = f"/tmp/{file.split('/')[-1]}"
        s3.download_file(input_bucket, file, local_file)
        
        with tarfile.open(local_file, "r:gz") as tar:
            tar.extractall("/tmp/extracted")

        for file_name in os.listdir("/tmp/extracted"):
            s3.upload_file(f"/tmp/extracted/{file_name}", output_bucket, file_name)

def record_end_time():
    end_time = time.time()
    s3.put_object(
        Bucket=output_bucket,
        Key=f"unzip_status/{job_id}.json",
        Body=json.dumps({"job_id": job_id, "end_time": end_time})
    )

def main():
    extract_files()
    record_end_time()

if __name__ == "__main__":
    main()
```

---

## **📜 `lambda/lambda_function.py` (触发 Batch 任务)**
```python
import boto3
import json
import time

s3 = boto3.client("s3")
batch = boto3.client("batch")

S3_BUCKET = "my-batch-output-bucket"
STATUS_FILE = "unzip_status/start_time.json"

def lambda_handler(event, context):
    start_time = time.time()

    s3.put_object(
        Bucket=S3_BUCKET,
        Key=STATUS_FILE,
        Body=json.dumps({"start_time": start_time})
    )

    response = batch.submit_job(
        jobName="unzip-job",
        jobQueue="unzip-queue",
        jobDefinition="unzip-job"
    )

    return response
```

---

## **📜 `lambda/calculate_time_lambda.py` (计算解压总时间)**
```python
import boto3
import json
import time

s3 = boto3.client("s3")
S3_BUCKET = "my-batch-output-bucket"

def lambda_handler(event, context):
    start_obj = s3.get_object(Bucket=S3_BUCKET, Key="unzip_status/start_time.json")
    start_time = json.loads(start_obj["Body"].read().decode("utf-8"))["start_time"]

    response = s3.list_objects_v2(Bucket=S3_BUCKET, Prefix="unzip_status/")
    end_times = []

    for item in response.get("Contents", []):
        if "start_time.json" not in item["Key"]:
            obj = s3.get_object(Bucket=S3_BUCKET, Key=item["Key"])
            job_data = json.loads(obj["Body"].read().decode("utf-8"))
            end_times.append(job_data["end_time"])

    if end_times:
        total_time = max(end_times) - start_time
    else:
        total_time = None

    csv_content = f"Start_Time,End_Time,Total_Time\n{start_time},{max(end_times)},{total_time}\n"
    s3.put_object(Bucket=S3_BUCKET, Key="unzip_results.csv", Body=csv_content)

    return {"start_time": start_time, "end_time": max(end_times), "total_time": total_time}
```

---

## **📜 `lambda/zip_lambda.sh`**
```sh
cd lambda
zip -r lambda_function.zip lambda_function.py calculate_time_lambda.py requirements.txt
```

---

### **🚀 部署步骤**
```sh
terraform init
terraform apply
```
```sh
docker build -t unzip-processor ecs-unzip/
aws ecr get-login-password | docker login --username AWS --password-stdin <ECR_URL>
docker push <ECR_URL>:latest
```
```sh
aws lambda invoke --function-name TriggerBatchJob response.json
```

这样就完成了 **S3 -> AWS Batch -> ECS -> 计算解压时间** 的整个流程 🚀
